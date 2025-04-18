<script setup lang="ts">
import { throttledWatch, useEventListener } from '@vueuse/core'
import { computed, ref, watch } from 'vue'
import { useNav } from '../composables/useNav'
import { useDynamicSlideInfo } from '../composables/useSlideInfo'
import { activeElement, editorHeight, editorWidth, isInputting, showEditor, isEditorVertical as vertical } from '../state'
import IconButton from './IconButton.vue'
import ShikiEditor from './ShikiEditor.vue'

const props = defineProps<{
  resize?: boolean
}>()

const { currentSlideNo, openInEditor } = useNav()

const tab = ref<'content' | 'note'>('content')
const content = ref('')
const note = ref('')
const dirty = ref(false)

const { info, update } = useDynamicSlideInfo(currentSlideNo)

watch(
  info,
  (v) => {
    if (!isInputting.value) {
      note.value = (v?.note || '').trim()
      const frontmatterPart = v?.frontmatterRaw?.trim() ? `---\n${v.frontmatterRaw.trim()}\n---\n\n` : ''
      content.value = frontmatterPart + (v?.source.contentRaw || '').trim()
      dirty.value = false
    }
  },
  { immediate: true },
)

async function save() {
  dirty.value = false

  let frontmatterRaw: string | undefined
  const contentOnly = content.value.trim().replace(/^---\n([\s\S]*?)\n---\n/, (_, f) => {
    frontmatterRaw = f
    return ''
  })

  await update({
    note: note.value || undefined,
    content: contentOnly,
    frontmatterRaw,
  })
}

function close() {
  showEditor.value = false
}

useEventListener('keydown', (e) => {
  if (activeElement.value?.tagName === 'TEXTAREA' && e.code === 'KeyS' && (e.ctrlKey || e.metaKey)) {
    save()
    e.preventDefault()
  }
})

const contentRef = computed({
  get() { return content.value },
  set(v) {
    if (content.value.trim() !== v.trim())
      dirty.value = true
    content.value = v
  },
})

const noteRef = computed({
  get() { return note.value },
  set(v) {
    note.value = v
    dirty.value = true
  },
})

const handlerDown = ref(false)
function onHandlerDown() {
  handlerDown.value = true
}
function updateSize(v?: number) {
  if (vertical.value)
    editorHeight.value = Math.min(Math.max(300, v ?? editorHeight.value), window.innerHeight - 200)
  else
    editorWidth.value = Math.min(Math.max(318, v ?? editorWidth.value), window.innerWidth - 200)
}
function switchTab(newTab: typeof tab.value) {
  tab.value = newTab
  // @ts-expect-error force cast
  document.activeElement?.blur?.()
}

if (props.resize) {
  useEventListener('pointermove', (e) => {
    if (handlerDown.value) {
      updateSize(vertical.value
        ? window.innerHeight - e.pageY
        : window.innerWidth - e.pageX)
    }
  }, { passive: true })
  useEventListener('pointerup', () => {
    handlerDown.value = false
  })
  useEventListener('resize', () => {
    updateSize()
  })
}

throttledWatch(
  [content, note],
  () => {
    if (dirty.value)
      save()
  },
  { throttle: 500 },
)
</script>

<template>
  <div
    v-if="resize" class="fixed bg-gray-400 select-none opacity-0 hover:opacity-10 z-dragging"
    :class="vertical ? 'left-0 right-0 w-full h-10px' : 'top-0 bottom-0 w-10px h-full'" :style="{
      opacity: handlerDown ? '0.3' : undefined,
      bottom: vertical ? `${editorHeight - 5}px` : undefined,
      right: !vertical ? `${editorWidth - 5}px` : undefined,
      cursor: vertical ? 'row-resize' : 'col-resize',
    }" @pointerdown="onHandlerDown"
  />
  <div
    class="shadow bg-main p-2 pt-4 grid grid-rows-[max-content_1fr] h-full overflow-hidden"
    :class="resize ? 'border-l border-gray-400 border-opacity-20' : ''"
    :style="resize ? {
      height: vertical ? `${editorHeight}px` : undefined,
      width: !vertical ? `${editorWidth}px` : undefined,
    } : {}"
  >
    <div class="flex pb-2 text-xl -mt-1">
      <div class="mr-4 rounded flex">
        <IconButton
          title="Switch to content tab" :class="tab === 'content' ? 'text-primary' : ''"
          @click="switchTab('content')"
        >
          <div class="i-carbon:account" />
        </IconButton>
        <IconButton
          title="Switch to notes tab" :class="tab === 'note' ? 'text-primary' : ''"
          @click="switchTab('note')"
        >
          <div class="i-carbon:align-box-bottom-right" />
        </IconButton>
      </div>
      <span class="text-2xl pt-1">
        {{ tab === 'content' ? 'Slide' : 'Notes' }}
      </span>
      <div class="flex-auto" />
      <template v-if="resize">
        <IconButton v-if="vertical" title="Dock to right" @click="vertical = false">
          <div class="i-carbon:open-panel-right" />
        </IconButton>
        <IconButton v-else title="Dock to bottom" @click="vertical = true">
          <div class="i-carbon:open-panel-bottom" />
        </IconButton>
      </template>
      <IconButton title="Open in editor" @click="openInEditor()">
        <div class="i-carbon:launch" />
      </IconButton>
      <IconButton title="Close" @click="close">
        <div class="i-carbon:close" />
      </IconButton>
    </div>
    <div class="relative overflow-hidden rounded" style="background-color: var(--slidev-code-background)">
      <ShikiEditor v-show="tab === 'content'" v-model="contentRef" placeholder="Create slide content..." />
      <ShikiEditor v-show="tab === 'note'" v-model="noteRef" placeholder="Write some notes..." />
    </div>
  </div>
</template>
