<template>
	<button
		v-if="link && !link.onlyMobile"
		class="flex h-7 cursor-pointer items-center rounded text-ink-gray-8 duration-300 ease-in-out focus:outline-none focus:transition-none focus-visible:rounded focus-visible:ring-2 focus-visible:ring-outline-gray-3"
		:class="
			link.highlight
				? 'border border-blue-200 bg-blue-100 shadow-sm'
				: isActive
					? 'bg-surface-selected shadow-sm'
					: 'hover:bg-surface-gray-2'
		"
		@click="handleClick"
	>
		<div
			class="group flex w-full items-center duration-300 ease-in-out"
			:class="isCollapsed ? 'relative p-1' : 'px-2 py-1'"
		>
			<Tooltip :text="link.label" placement="right">
				<slot name="icon">
					<span class="grid h-5 w-6 flex-shrink-0 place-items-center">
						<component
							:is="icons[link.icon]"
							class="h-4 w-4 stroke-1.5"
							:class="link.highlight ? 'text-blue-700' : 'text-ink-gray-8'"
						/>
					</span>
				</slot>
			</Tooltip>
			<span
				class="flex-shrink-0 text-sm duration-300 ease-in-out"
				:class="
					isCollapsed
						? 'ml-0 w-0 overflow-hidden opacity-0'
						: 'ml-2 w-auto opacity-100'
				"
				:style="link.highlight ? 'color: #1e40af; font-weight: 600;' : ''"
			>
				{{ __(link.label) }}
			</span>
			<span
				v-if="link.count"
				class="!ml-auto block text-xs text-ink-gray-5"
				:class="
					isCollapsed && link.count > 9
						? 'absolute right-0 top-[2px] bg-surface-white'
						: ''
				"
			>
				{{ link.count }}
			</span>
			<div
				v-if="showControls && !isCollapsed"
				class="invisible !ml-auto block flex items-center space-x-2 text-xs text-ink-gray-5 group-hover:visible"
			>
				<component
					:is="icons['Edit']"
					class="h-3 w-3 stroke-1.5 text-ink-gray-7"
					@click.stop="openModal(link)"
				/>
				<component
					:is="icons['X']"
					class="h-3 w-3 stroke-1.5 text-ink-gray-7"
					@click.stop="deletePage(link)"
				/>
			</div>
		</div>
	</button>
</template>
<script setup>
import { Tooltip } from 'frappe-ui'
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import * as icons from 'lucide-vue-next'

const router = useRouter()
const emit = defineEmits(['openModal', 'deletePage'])

const props = defineProps({
	link: {
		type: Object,
		required: true,
	},
	isCollapsed: {
		type: Boolean,
		default: false,
	},
	showControls: {
		type: Boolean,
		default: false,
	},
})

function handleClick() {
	if (props.link.to === 'builder/') {
		window.open(window.location.origin + '/builder', '_blank')
		return
	}

	if (router.hasRoute(props.link.to)) {
		router.push({ name: props.link.to })
	} else if (props.link.to) {
		window.location.href = `/${props.link.to}`
	}
}

const isActive = computed(() => {
	return props.link?.activeFor?.includes(router.currentRoute.value.name)
})

const openModal = (link) => {
	emit('openModal', link)
}

const deletePage = (link) => {
	emit('deletePage', link)
}
</script>
