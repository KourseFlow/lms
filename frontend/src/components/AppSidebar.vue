<template>
	<div
		class="flex h-full flex-col justify-between border-r bg-surface-menu-bar transition-all duration-300 ease-in-out"
		:class="sidebarStore.isSidebarCollapsed ? 'w-14' : 'w-56'"
	>
		<div
			class="flex flex-col overflow-hidden"
			:class="sidebarStore.isSidebarCollapsed ? 'items-center' : ''"
		>
			<UserDropdown :isCollapsed="sidebarStore.isSidebarCollapsed" />
			<div class="flex flex-col" v-if="sidebarSettings.data">
				<template v-for="(link, index) in sidebarLinks" :key="index">
					<!-- Add divider before highlighted links -->
					<div
						v-if="link.highlight && !sidebarStore.isSidebarCollapsed"
						class="mx-2 my-2 border-t border-gray-200"
					></div>
					<SidebarLink
						:link="link"
						:isCollapsed="sidebarStore.isSidebarCollapsed"
						class="mx-2 my-0.5"
					/>
				</template>
			</div>
			<div
				v-if="sidebarSettings.data?.web_pages?.length || isModerator"
				class="mt-4"
			>
				<div
					v-if="sidebarSettings.data?.web_pages?.length"
					class="flex flex-col transition-all duration-300 ease-in-out"
					:class="!sidebarStore.isWebpagesCollapsed ? 'block' : 'hidden'"
				>
					<SidebarLink
						v-for="link in sidebarSettings.data.web_pages"
						:link="link"
						:isCollapsed="sidebarStore.isSidebarCollapsed"
						class="mx-2 my-0.5"
						:showControls="isModerator ? true : false"
						@openModal="openPageModal"
						@deletePage="deletePage"
					/>
				</div>
			</div>
		</div>
		<div class="m-2 flex flex-col gap-1">
			<div
				v-if="readOnlyMode && !sidebarStore.isSidebarCollapsed"
				class="z-10 m-2 rounded-md bg-surface-modal px-3 py-2.5 text-xs leading-5 text-ink-gray-7"
			>
				{{
					__(
						'This site is being updated. You will not be able to make any changes. Full access will be restored shortly.'
					)
				}}
			</div>

			<div
				class="mt-4 flex items-center"
				:class="
					sidebarStore.isSidebarCollapsed ? 'flex-col space-y-3' : 'flex-row'
				"
			>
				<div
					class="flex flex-1 items-center"
					:class="
						sidebarStore.isSidebarCollapsed
							? 'flex-col space-y-3'
							: 'flex-row space-x-3'
					"
				></div>
				<Tooltip
					:text="
						sidebarStore.isSidebarCollapsed ? __('Expand') : __('Collapse')
					"
				>
					<CollapseSidebar
						class="size-4 cursor-pointer stroke-1.5 text-ink-gray-7 duration-300 ease-in-out"
						:class="{
							'[transform:rotateY(180deg)]': sidebarStore.isSidebarCollapsed,
						}"
						@click="toggleSidebar()"
					/>
				</Tooltip>
			</div>
		</div>

		<IntermediateStepModal
			v-model="showIntermediateModal"
			:currentStep="currentStep"
		/>
	</div>
	<PageModal
		v-model="showPageModal"
		v-model:reloadSidebar="sidebarSettings"
		:page="pageToEdit"
	/>
</template>

<script setup>
import UserDropdown from '@/components/UserDropdown.vue'
import CollapseSidebar from '@/components/Icons/CollapseSidebar.vue'
import SidebarLink from '@/components/SidebarLink.vue'
import {
	ref,
	onMounted,
	inject,
	watch,
	reactive,
	markRaw,
	h,
	onUnmounted,
	computed,
} from 'vue'
import { getSidebarLinks } from '@/utils'
import { usersStore } from '@/stores/user'
import { sessionStore } from '@/stores/session'
import { useSidebar } from '@/stores/sidebar'
import { useSettings } from '@/stores/settings'
import { createResource, Tooltip } from 'frappe-ui'
import PageModal from '@/components/Modals/PageModal.vue'
import { useRouter } from 'vue-router'
import InviteIcon from './Icons/InviteIcon.vue'
import {
	BookOpen,
	CircleAlert,
	CircleHelp,
	FolderTree,
	FileText,
	UserPlus,
	Users,
	BookText,
	Presentation,
} from 'lucide-vue-next'
import {
	useOnboarding,
	minimize,
	IntermediateStepModal,
} from 'frappe-ui/frappe'

const { user } = sessionStore()
const { userResource } = usersStore()
let sidebarStore = useSidebar()
const socket = inject('$socket')
const unreadCount = ref(0)
const sidebarLinks = ref(getSidebarLinks())
const showPageModal = ref(false)
const isModerator = ref(false)
const isInstructor = ref(false)
const pageToEdit = ref(null)
const settingsStore = useSettings()
const { sidebarSettings } = settingsStore
const showOnboarding = ref(false)
const showIntermediateModal = ref(false)
const currentStep = ref({})
const router = useRouter()
let onboardingDetails
let isOnboardingStepsCompleted = false
const readOnlyMode = window.read_only_mode
const iconProps = {
	strokeWidth: 1.5,
	width: 16,
	height: 16,
}

onMounted(() => {
	addNotifications()
	setSidebarLinks()
	setUpOnboarding()
	socket.on('publish_lms_notifications', (data) => {
		unreadNotifications.reload()
	})
})

const setSidebarLinks = () => {
	sidebarSettings.reload(
		{},
		{
			onSuccess(data) {
				Object.keys(data).forEach((key) => {
					if (!parseInt(data[key])) {
						sidebarLinks.value = sidebarLinks.value.filter(
							(link) => link.label.toLowerCase().split(' ').join('_') !== key
						)
					}
				})

				filterAdminLinks()
			},
		}
	)
}

const filterAdminLinks = () => {
	if (!userResource.data?.is_system_manager) {
		const adminOnlyLinks = ['Statistics']
		sidebarLinks.value = sidebarLinks.value.filter(
			(link) => !adminOnlyLinks.includes(link.label)
		)
	}
}

const unreadNotifications = createResource({
	cache: 'Unread Notifications Count',
	url: 'frappe.client.get_count',
	makeParams(values) {
		return {
			doctype: 'Notification Log',
			filters: {
				for_user: user,
				read: 0,
			},
		}
	},
	onSuccess(data) {
		unreadCount.value = data
		sidebarLinks.value = sidebarLinks.value.map((link) => {
			if (link.label === 'Notifications') {
				link.count = data
			}
			return link
		})
	},
	auto: user ? true : false,
})

const addNotifications = () => {
	if (user) {
		sidebarLinks.value.push({
			label: 'Notifications',
			icon: 'Bell',
			to: 'Notifications',
			activeFor: ['Notifications'],
			count: unreadCount.value,
		})
	}
}

const addQuizzes = () => {
	if (isInstructor.value || isModerator.value) {
		sidebarLinks.value.splice(4, 0, {
			label: 'Quizzes',
			icon: 'CircleHelp',
			to: 'Quizzes',
			activeFor: [
				'Quizzes',
				'QuizForm',
				'QuizSubmissionList',
				'QuizSubmission',
			],
		})
	}
}

const addAssignments = () => {
	if (isInstructor.value || isModerator.value) {
		sidebarLinks.value.splice(5, 0, {
			label: 'Assignments',
			icon: 'Pencil',
			to: 'Assignments',
			activeFor: [
				'Assignments',
				'AssignmentForm',
				'AssignmentSubmissionList',
				'AssignmentSubmission',
			],
		})
	}
}

const addPageBuilder = () => {
	if (isInstructor.value || isModerator.value) {
		sidebarLinks.value.push({
			label: 'Page Builder',
			icon: 'Presentation',
			to: 'builder/',
			highlight: true,
		})
	}
}

const addPrograms = () => {
	let activeFor = ['Programs', 'ProgramForm']
	let index = 1
	let canAddProgram = false

	if (
		!isInstructor.value &&
		!isModerator.value &&
		settingsStore.learningPaths.data
	) {
		sidebarLinks.value = sidebarLinks.value.filter(
			(link) => link.label !== 'Courses'
		)
		activeFor.push('CourseDetail')
		activeFor.push('Lesson')
		index = 0
		canAddProgram = true
	} else if (isInstructor.value || isModerator.value) {
		canAddProgram = true
	}

	if (canAddProgram) {
		sidebarLinks.value.splice(index, 0, {
			label: 'Programs',
			icon: 'Route',
			to: 'Programs',
			activeFor: activeFor,
		})
	}
}

const openPageModal = (link) => {
	showPageModal.value = true
	pageToEdit.value = link
}

const deletePage = (link) => {
	createResource({
		url: 'lms.lms.api.delete_sidebar_item',
		makeParams(values) {
			return {
				webpage: link.web_page,
			}
		},
	}).submit(
		{},
		{
			onSuccess() {
				sidebarSettings.reload()
			},
		}
	)
}

const toggleSidebar = () => {
	sidebarStore.isSidebarCollapsed = !sidebarStore.isSidebarCollapsed
	localStorage.setItem(
		'isSidebarCollapsed',
		JSON.stringify(sidebarStore.isSidebarCollapsed)
	)
}

const toggleWebPages = () => {
	sidebarStore.isWebpagesCollapsed = !sidebarStore.isWebpagesCollapsed
	localStorage.setItem(
		'isWebpagesCollapsed',
		JSON.stringify(sidebarStore.isWebpagesCollapsed)
	)
}

const getFirstCourse = async () => {
	let firstCourse = localStorage.getItem('firstCourse')
	if (firstCourse) return firstCourse
	return await call('lms.lms.onboarding.get_first_course')
}

const getFirstBatch = async () => {
	let firstBatch = localStorage.getItem('firstBatch')
	if (firstBatch) return firstBatch
	return await call('lms.lms.onboarding.get_first_batch')
}

const steps = reactive([
	{
		name: 'create_first_course',
		title: __('Create your first course'),
		icon: markRaw(h(BookOpen, iconProps)),
		completed: false,
		onClick: () => {
			minimize.value = true
			router.push({
				name: 'Courses',
			})
		},
	},
	{
		name: 'create_first_chapter',
		title: __('Add your first chapter'),
		icon: markRaw(h(FolderTree, iconProps)),
		completed: false,
		dependsOn: 'create_first_course',
		onClick: async () => {
			minimize.value = true
			let course = await getFirstCourse()
			if (course) {
				router.push({ name: 'CourseForm', params: { courseName: course } })
			} else {
				router.push({ name: 'CourseForm' })
			}
		},
	},
	{
		name: 'create_first_lesson',
		title: __('Add your first lesson'),
		icon: markRaw(h(FileText, iconProps)),
		completed: false,
		dependsOn: 'create_first_chapter',
		onClick: async () => {
			minimize.value = true
			let course = await getFirstCourse()
			if (course) {
				router.push({
					name: 'CourseForm',
					params: { courseName: course },
				})
			} else {
				router.push({ name: 'Courses' })
			}
		},
	},
	{
		name: 'create_first_quiz',
		title: __('Create your first quiz'),
		icon: markRaw(h(CircleHelp, iconProps)),
		completed: false,
		dependsOn: 'create_first_course',
		onClick: () => {
			minimize.value = true
			router.push({ name: 'Quizzes' })
		},
	},
	{
		name: 'invite_students',
		title: __('Invite your team and students'),
		icon: markRaw(h(InviteIcon, iconProps)),
		completed: false,
		onClick: () => {
			minimize.value = true
			settingsStore.activeTab = 'Members'
			settingsStore.isSettingsOpen = true
		},
	},
	{
		name: 'create_first_batch',
		title: __('Create your first batch'),
		icon: markRaw(h(Users, iconProps)),
		completed: false,
		onClick: () => {
			minimize.value = true
			router.push({ name: 'Batches' })
		},
	},
	{
		name: 'add_batch_student',
		title: __('Add students to your batch'),
		icon: markRaw(h(UserPlus, iconProps)),
		completed: false,
		dependsOn: 'create_first_batch',
		onClick: async () => {
			minimize.value = true
			let batch = await getFirstBatch()
			if (batch) {
				router.push({
					name: 'Batch',
					params: {
						batchName: batch,
					},
				})
			} else {
				router.push({ name: 'Batch' })
			}
		},
	},
	{
		name: 'add_batch_course',
		title: __('Add courses to your batch'),
		icon: markRaw(h(BookText, iconProps)),
		completed: false,
		dependsOn: 'create_first_batch',
		onClick: async () => {
			minimize.value = true
			let batch = await getFirstBatch()
			if (batch) {
				router.push({
					name: 'Batch',
					params: {
						batchName: batch,
					},
					hash: '#courses',
				})
			} else {
				router.push({ name: 'Batch' })
			}
		},
	},
])

const articles = ref([
	{
		title: __('Introduction'),
		opened: false,
		subArticles: [
			{ name: 'introduction', title: __('Introduction') },
			{ name: 'setting-up', title: __('Setting up') },
		],
	},
	{
		title: __('Creating a course'),
		opened: false,
		subArticles: [
			{ name: 'create-a-course', title: __('Create a course') },
			{ name: 'add-a-chapter', title: __('Add a chapter') },
			{ name: 'add-a-lesson', title: __('Add a lesson') },
		],
	},
	{
		title: __('Creating a batch'),
		opened: false,
		subArticles: [
			{ name: 'create-a-batch', title: __('Create a batch') },
			{ name: 'create-a-live-class', title: __('Create a live class') },
		],
	},
	{
		title: __('Assessments'),
		opened: false,
		subArticles: [
			{ name: 'quizzes', title: __('Quizzes') },
			{ name: 'assignments', title: __('Assignments') },
		],
	},
	{
		title: __('Certification'),
		opened: false,
		subArticles: [
			{ name: 'issue-a-certificate', title: __('Issue a Certificate') },
			{
				name: 'custom-certificate-templates',
				title: __('Custom Certificate Templates'),
			},
		],
	},
	{
		title: __('Monetization'),
		opened: false,
		subArticles: [
			{
				name: 'setting-up-payment-gateway',
				title: __('Setting up payment gateway'),
			},
		],
	},
	{
		title: __('Settings'),
		opened: false,
		subArticles: [{ name: 'roles', title: __('Roles') }],
	},
])

const setUpOnboarding = () => {
	if (userResource.data?.is_system_manager) {
		onboardingDetails = useOnboarding('learning')
		onboardingDetails.setUp(steps)
		isOnboardingStepsCompleted = onboardingDetails.isOnboardingStepsCompleted
		showOnboarding.value = true
	}
}

watch(userResource, () => {
	if (userResource.data) {
		isModerator.value = userResource.data.is_moderator
		isInstructor.value = userResource.data.is_instructor
		addPrograms()
		addQuizzes()
		addAssignments()
		setUpOnboarding()
		addPageBuilder()
	}
})

onUnmounted(() => {
	socket.off('publish_lms_notifications')
})
</script>
