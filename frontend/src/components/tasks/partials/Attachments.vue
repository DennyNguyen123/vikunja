<template>
	<div class="attachments">
		<h3>
			<span class="icon is-grey">
				<Icon icon="paperclip" />
			</span>
			{{ $t('task.attachment.title') }}
		</h3>

		<input
			v-if="editEnabled"
			id="files"
			ref="filesRef"
			:disabled="loading || undefined"
			multiple
			type="file"
			@change="uploadNewAttachment()"
		>

		<ProgressBar
			v-if="attachmentService.uploadProgress > 0"
			:value="attachmentService.uploadProgress * 100"
			is-primary
		/>

		<div
			v-if="attachments.length > 0"
			class="files"
		>
			<button
				v-for="a in attachments"
				:key="a.id"
				class="attachment"
				@click="viewOrDownload(a)"
			>
				<div class="preview-column">
					<FilePreview
						class="attachment-preview"
						:model-value="a"
					/>
				</div>
				<div class="attachment-info-column">
					<div class="filename">
						{{ a.file.name }}
						<span
							v-if="task.coverImageAttachmentId === a.id"
							class="is-task-cover"
						>
							{{ $t('task.attachment.usedAsCover') }}
						</span>
					</div>
					<div class="info">
						<p class="attachment-info-meta">
							<i18n-t
								keypath="task.attachment.createdBy"
								scope="global"
							>
								<span v-tooltip="formatDateLong(a.created)">
									{{ formatDisplayDate(a.created) }}
								</span>
								<User
									:avatar-size="24"
									:user="a.createdBy"
									:is-inline="true"
								/>
							</i18n-t>
							<span>
								{{ getHumanSize(a.file.size) }}
							</span>
							<span v-if="a.file.mime">
								{{ a.file.mime }}
							</span>
						</p>
						<p>
							<BaseButton
								v-tooltip="$t('task.attachment.downloadTooltip')"
								class="attachment-info-meta-button"
								@click.prevent.stop="downloadAttachment(a)"
							>
								<Icon icon="download" />
							</BaseButton>
							<BaseButton
								v-tooltip="$t('task.attachment.copyUrlTooltip')"
								class="attachment-info-meta-button"
								@click.stop="copyUrl(a)"
							>
								<Icon icon="copy" />
							</BaseButton>
							<BaseButton
								v-if="editEnabled"
								v-tooltip="$t('task.attachment.deleteTooltip')"
								class="attachment-info-meta-button"
								@click.prevent.stop="setAttachmentToDelete(a)"
							>
								<Icon icon="trash-alt" />
							</BaseButton>
							<BaseButton
								v-if="editEnabled && canPreview(a)"
								v-tooltip="task.coverImageAttachmentId === a.id
									? $t('task.attachment.unsetAsCover')
									: $t('task.attachment.setAsCover')"
								class="attachment-info-meta-button"
								@click.prevent.stop="setCoverImage(task.coverImageAttachmentId === a.id ? null : a)"
							>
								<Icon :icon="task.coverImageAttachmentId === a.id ? 'eye-slash' : 'eye'" />
							</BaseButton>
						</p>
					</div>
				</div>
			</button>
		</div>

		<XButton
			v-if="editEnabled"
			:disabled="loading"
			class="mbe-4"
			icon="cloud-upload-alt"
			variant="secondary"
			:shadow="false"
			@click="filesRef?.click()"
		>
			{{ $t('task.attachment.upload') }}
		</XButton>

		<!-- Dropzone -->
		<Teleport to="body">
			<div
				v-if="editEnabled"
				:class="{hidden: !showDropzone}"
				class="dropzone"
			>
				<div class="drop-hint">
					<div class="icon">
						<Icon icon="cloud-upload-alt" />
					</div>
					<div class="hint">
						{{ $t('task.attachment.drop') }}
					</div>
				</div>
			</div>
		</Teleport>

		<!-- Delete modal -->
		<Modal
			:enabled="attachmentToDelete !== null"
			@close="setAttachmentToDelete(null)"
			@submit="deleteAttachment()"
		>
			<template #header>
				<span>{{ $t('task.attachment.delete') }}</span>
			</template>

			<template #text>
				<p>
					{{ $t('task.attachment.deleteText1', {filename: attachmentToDelete.file.name}) }}<br>
					<strong class="has-text-white">{{ $t('misc.cannotBeUndone') }}</strong>
				</p>
			</template>
		</Modal>

		<!-- Attachment image modal -->
		<Modal
			:enabled="attachmentImageBlobUrl !== null"
			class="attachment-modal"
			@close="attachmentImageBlobUrl = null"
		>
			<div
				ref="imageWrapperRef"
				class="image-wrapper"
				@wheel.prevent="onWheel"
				@mousedown="startDrag"
				@mousemove="onDrag"
				@mouseup="stopDrag"
				@mouseleave="stopDrag"
			>
				<img
					:src="attachmentImageBlobUrl"
					:style="{
						transform: `translate(${transform.x}px, ${transform.y}px) scale(${transform.scale})`,
						cursor: isDragging ? 'grabbing' : 'grab',
					}"
					draggable="false"
					alt=""
				>
			</div>
			<div class="zoom-controls">
				<BaseButton
					class="zoom-button"
					:title="t('task.attachment.zoomIn')"
					@click.stop="zoomIn"
				>
					<Icon icon="search-plus" />
				</BaseButton>
				<BaseButton
					class="zoom-button"
					:title="t('task.attachment.zoomReset')"
					@click.stop="resetZoom"
				>
					<Icon icon="compress-arrows-alt" />
				</BaseButton>
				<BaseButton
					class="zoom-button"
					:title="t('task.attachment.zoomOut')"
					@click.stop="zoomOut"
				>
					<Icon icon="search-minus" />
				</BaseButton>
			</div>
		</Modal>
	</div>
</template>

<script setup lang="ts">
import {ref, shallowReactive, computed, watch, reactive} from 'vue'
import {useDropZone} from '@vueuse/core'

import User from '@/components/misc/User.vue'
import ProgressBar from '@/components/misc/ProgressBar.vue'
import BaseButton from '@/components/base/BaseButton.vue'

import AttachmentService from '@/services/attachment'
import {canPreview} from '@/models/attachment'
import type {IAttachment} from '@/modelTypes/IAttachment'
import type {ITask} from '@/modelTypes/ITask'

import {useAttachmentStore} from '@/stores/attachments'
import {formatDisplayDate, formatDateLong} from '@/helpers/time/formatDate'
import {uploadFiles, generateAttachmentUrl} from '@/helpers/attachments'
import {getHumanSize} from '@/helpers/getHumanSize'
import {useCopyToClipboard} from '@/composables/useCopyToClipboard'
import {error, success} from '@/message'
import {useTaskStore} from '@/stores/tasks'
import {useI18n} from 'vue-i18n'
import FilePreview from '@/components/tasks/partials/FilePreview.vue'

const props = withDefaults(defineProps<{
	task: ITask,
	editEnabled?: boolean,
}>(), {
	editEnabled: true,
})

// FIXME: this should go through the store
const emit = defineEmits<{
	'taskChanged': [ITask],
}>()

const EDITOR_SELECTOR = '.tiptap, .tiptap__editor, [contenteditable]'

function eventTargetsEditor(event: Event | null | undefined): boolean {
	if (!event) {
		return false
	}

	const target = event.target
	if (target instanceof HTMLElement && target.closest(EDITOR_SELECTOR)) {
		return true
	}

	if (typeof event.composedPath === 'function') {
		return event.composedPath().some(element =>
			element instanceof HTMLElement && element.matches(EDITOR_SELECTOR),
		)
	}

	return false
}

const taskStore = useTaskStore()
const {t} = useI18n({useScope: 'global'})

const attachmentService = shallowReactive(new AttachmentService())

const attachmentStore = useAttachmentStore()
const attachments = computed(() => attachmentStore.attachments)

const loading = computed(() => attachmentService.loading || taskStore.isLoading)

const isDraggingFiles = ref(false)
const isDragOverEditor = ref(false)

function resetDragState() {
	isDraggingFiles.value = false
	isDragOverEditor.value = false
}

const {isOverDropZone} = useDropZone(document, {
	onEnter(files, event) {
		if (!props.editEnabled) {
			return
		}
		
		isDraggingFiles.value = true
		isDragOverEditor.value = eventTargetsEditor(event)
	},
	onOver(files, event) {
		if (!props.editEnabled) {
			return
		}

		isDragOverEditor.value = eventTargetsEditor(event)
	},
	onLeave(files, event) {
		if (!props.editEnabled) {
			return
		}

		if (!isOverDropZone.value) {
			resetDragState()
			return
		}

		isDragOverEditor.value = eventTargetsEditor(event)
	},
	onDrop(files, event) {
		if (!props.editEnabled) {
			return
		}

		const dropOverEditor = eventTargetsEditor(event)
		resetDragState()

		// Ignore drops over editor - let TipTap handle them
		if (dropOverEditor || !files || files.length === 0) {
			return
		}

		uploadFilesToTask(files)
	},
})

const showDropzone = computed(() =>
	props.editEnabled && isDraggingFiles.value && !isDragOverEditor.value,
)

watch(() => props.editEnabled, enabled => {
	if (!enabled) {
		resetDragState()
	}
})

function downloadAttachment(attachment: IAttachment) {
	attachmentService.download(attachment)
}

const filesRef = ref<HTMLInputElement | null>(null)

function uploadNewAttachment() {
	const files = filesRef.value?.files

	if (!files || files.length === 0) {
		return
	}

	uploadFilesToTask(files)
}

function uploadFilesToTask(files: File[] | FileList) {
	uploadFiles(attachmentService, props.task.id, files)
}

const attachmentToDelete = ref<IAttachment | null>(null)

function setAttachmentToDelete(attachment: IAttachment | null) {
	attachmentToDelete.value = attachment
}

async function deleteAttachment() {
	if (attachmentToDelete.value === null) {
		return
	}

	try {
		const r = await attachmentService.delete(attachmentToDelete.value)
		attachmentStore.removeById(attachmentToDelete.value.id)
		success(r)
		setAttachmentToDelete(null)
	} catch (e) {
		error(e)
	}
}

const attachmentImageBlobUrl = ref<string | null>(null)

async function viewOrDownload(attachment: IAttachment) {
	if (canPreview(attachment)) {
		attachmentImageBlobUrl.value = await attachmentService.getBlobUrl(attachment)
	} else {
		downloadAttachment(attachment)
	}
}

const copy = useCopyToClipboard()

function copyUrl(attachment: IAttachment) {
	copy(generateAttachmentUrl(props.task.id, attachment.id))
}

async function setCoverImage(attachment: IAttachment | null) {
	const updatedTask = await taskStore.setCoverImage(props.task, attachment)
	emit('taskChanged', updatedTask)
	success({message: t('task.attachment.successfullyChangedCoverImage')})
}

// ZOOM & PAN LOGIC
const imageWrapperRef = ref<HTMLElement | null>(null)
const transform = reactive({
	scale: 1,
	x: 0,
	y: 0,
})
const isDragging = ref(false)
const startPan = { x: 0, y: 0 }

function zoomIn() {
	applyZoom(0.25)
}

function zoomOut() {
	applyZoom(-0.25)
}

function applyZoom(delta: number) {
	const newScale = Math.max(0.1, transform.scale + delta)
	transform.scale = newScale
}

function resetZoom() {
	transform.scale = 1
	transform.x = 0
	transform.y = 0
}

function onWheel(e: WheelEvent) {
	const zoomIntensity = 0.1
	const direction = e.deltaY < 0 ? 1 : -1
	const factor = direction * zoomIntensity
	const newScale = Math.max(0.1, transform.scale + (transform.scale * factor))

	// Zoom towards cursor logic
	// 1. Get cursor position relative to the image wrapper center
	// Note: We assume the image wrapper is centered in the viewport
	if (imageWrapperRef.value) {
		const rect = imageWrapperRef.value.getBoundingClientRect()
		
		// Mouse position relative to the element
		const mouseX = e.clientX - rect.left
		const mouseY = e.clientY - rect.top

		// Center of the element
		const centerX = rect.width / 2
		const centerY = rect.height / 2

		// Calculate offset from center
		const offsetX = mouseX - centerX
		const offsetY = mouseY - centerY

		// Adjust translation to keep the point under cursor stable
		// Formula: newTranslate = oldTranslate - (offset / oldScale) * (newScale - oldScale) 
		// Simpler approximation for "zoom to point":
		// We want to shift the image so that the point under the mouse remains in the same screen position.
		// The shift needed is proportional to the distance from center and the change in scale.
		
		transform.x -= (offsetX - transform.x) * (newScale / transform.scale - 1)
		transform.y -= (offsetY - transform.y) * (newScale / transform.scale - 1)
	}

	transform.scale = newScale
}

function startDrag(e: MouseEvent) {
	isDragging.value = true
	startPan.x = e.clientX - transform.x
	startPan.y = e.clientY - transform.y
}

function onDrag(e: MouseEvent) {
	if (!isDragging.value) return
	transform.x = e.clientX - startPan.x
	transform.y = e.clientY - startPan.y
}

function stopDrag() {
	isDragging.value = false
}

watch(attachmentImageBlobUrl, () => {
	resetZoom()
})
</script>

<style lang="scss" scoped>
.attachments {
	input[type="file"] {
		display: none;
	}

	@media screen and (max-width: $tablet) {
		.button {
			inline-size: 100%;
		}
	}
}

.files {
	margin-block-end: 1rem;
}

.attachment {
	display: grid;
	grid-template-columns: 9rem 1fr;
	align-items: center;
	inline-size: 100%;
	
	padding: .5rem;
	
	transition: background-color $transition;
	background-color: transparent;
	
	border: transparent;
	border-radius: $radius;

	&:hover {
		background-color: var(--grey-200);
	}
}

.filename {
	display: flex;
	align-items: center;
	font-weight: bold;
	min-block-size: 2rem;
	color: var(--text);
	text-align: start;
	word-break: break-all;
	min-inline-size: 0;
}

.info {
	color: var(--grey-500);
	font-size: .9rem;
	display: flex;
	flex-direction: column;

	p {
		margin-block-end: 0;
		display: flex;

		> span,
		> button:not(:last-child):after {
			padding: 0 .25rem;
		}
	}
}

.dropzone {
	position: fixed;
	background: hsla(var(--grey-100-hsl), 0.8);
	inset-block-start: 0;
	inset-inline-start: 0;
	inset-block-end: 0;
	inset-inline-end: 0;
	z-index: 4001; // modal z-index is 4000
	text-align: center;

	&.hidden {
		display: none;
	}
}

.drop-hint {
	position: absolute;
	inset-block-end: 0;
	inset-inline-start: 0;
	inset-inline-end: 0;

	.icon {
		inline-size: 100%;
		font-size: 5rem;
		block-size: auto;
		text-shadow: var(--shadow-md);
		animation: bounce 2s infinite;

		@media (prefers-reduced-motion: reduce) {
			animation: none;
		}
	}

	.hint {
		margin: .5rem auto 2rem;
		border-radius: $radius;
		box-shadow: var(--shadow-md);
		background: var(--primary);
		padding: 1rem;
		color: $white; // Should always be white because of the background, regardless of the theme
		inline-size: 100%;
		max-inline-size: 300px;
	}
}

.attachment-info-column {
	display: flex;
	flex-flow: column wrap;
	align-self: start;
	min-inline-size: 0;
}

.attachment-info-meta {
	display: flex;
	align-items: center;

	:deep(.user) {
		display: flex !important;
		align-items: center;
		margin: 0 .5rem;
	}

	@media screen and (max-width: $mobile) {
		flex-direction: column;
		align-items: flex-start;

		:deep(.user) {
			margin: .5rem 0;
		}

		> span:not(:last-child):after,
		> button:not(:last-child):after {
			display: none;
		}

		.user .username {
			display: none;
		}
	}
}

.attachment-info-meta-button {
	color: var(--link);
	padding: 0 .25rem;
}

@keyframes bounce {
	0%,
	20%,
	53%,
	80%,
	100% {
		animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1);
		transform: translate3d(0, 0, 0);
	}

	40%,
	43% {
		animation-timing-function: cubic-bezier(0.755, 0.05, 0.855, 0.06);
		transform: translate3d(0, -30px, 0);
	}

	70% {
		animation-timing-function: cubic-bezier(0.755, 0.05, 0.855, 0.06);
		transform: translate3d(0, -15px, 0);
	}

	90% {
		transform: translate3d(0, -4px, 0);
	}
}

.preview-column {
	max-inline-size: 8rem;
	block-size: 5.2rem;
}

.attachment-preview {
	block-size: 100%;
}

.is-task-cover {
	background: var(--primary);
	color: var(--white);
	margin-inline-start: .25rem;
	padding: .25rem .35rem;
	border-radius: 4px;
	font-size: .75rem;
}

.attachment-modal {
	:deep(.modal-content) {
		position: static;
		transform: none;
		margin: auto;
		display: flex;
		justify-content: center;
		align-items: center;
		min-block-size: 100%;
		max-inline-size: none;
		inline-size: 100%; /* Changed from auto to 100% */
		padding: 0; /* Remove padding to maximize space */
		background: transparent;
		box-shadow: none;
		overflow: hidden; /* Hide overflow to handle zoomed image */
	}
	
	:deep(.modal-container) {
		background: rgba(0, 0, 0, 0.9);
	}
}

.image-wrapper {
	width: 100%;
	height: 100vh;
	display: flex;
	justify-content: center;
	align-items: center;
	overflow: hidden;
	
	img {
		max-inline-size: 70vw; /* Initial size 70% width */
		max-block-size: 70vh; /* Initial size 70% height */
		object-fit: contain;
		display: block;
		box-shadow: 0 0 20px rgba(0,0,0,0.5);
		transition: transform 0.1s ease-out; /* Smooth zoom transition */
		will-change: transform;
	}
}

.zoom-controls {
	position: fixed;
	inset-block-end: 2rem;
	inset-inline-start: 50%;
	transform: translateX(-50%);
	background: rgba(40, 40, 40, 0.8);
	backdrop-filter: blur(10px);
	padding: 0.5rem 1rem;
	border-radius: 2rem;
	display: flex;
	gap: 1rem;
	z-index: 100;
	border: 1px solid rgba(255,255,255,0.1);

	.button {
		color: var(--white);
		background: transparent;
		border: none;
		width: 2.5rem;
		height: 2.5rem;
		padding: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		border-radius: 50%;
		font-size: 1.2rem;
		transition: background 0.2s;

		&:hover {
			background: rgba(255, 255, 255, 0.2);
		}
	}
}
</style>
