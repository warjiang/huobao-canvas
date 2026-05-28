<template>
  <div class="audio-node-wrapper relative" @mouseenter="showHandleMenu = true" @mouseleave="showHandleMenu = false">
    <div
      class="audio-node bg-[var(--bg-secondary)] rounded-xl border min-w-[240px] max-w-[320px] transition-all duration-200"
      :class="data.selected ? 'border-1 border-blue-500 shadow-lg shadow-blue-500/20' : 'border border-[var(--border-color)]'"
    >
      <div class="px-3 py-2 border-b border-[var(--border-color)] flex items-center justify-between">
        <span
          v-if="!isEditingLabel"
          @dblclick="startEditLabel"
          class="text-sm font-medium text-[var(--text-primary)] cursor-text hover:bg-[var(--bg-tertiary)] px-1 rounded transition-colors"
          title="双击编辑名称"
        >{{ data.label || '音频节点' }}</span>
        <input
          v-else
          ref="labelInputRef"
          v-model="editingLabelValue"
          @blur="finishEditLabel"
          @keydown.enter="finishEditLabel"
          @keydown.escape="cancelEditLabel"
          class="text-sm font-medium bg-[var(--bg-tertiary)] text-[var(--text-primary)] px-1 rounded outline-none border border-blue-500"
        />
        <div class="flex items-center gap-1">
          <button @click="handleDuplicate" class="p-1 hover:bg-[var(--bg-tertiary)] rounded transition-colors" title="复制节点">
            <n-icon :size="14"><CopyOutline /></n-icon>
          </button>
          <button @click="handleDelete" class="p-1 hover:bg-[var(--bg-tertiary)] rounded transition-colors" title="删除节点">
            <n-icon :size="14"><TrashOutline /></n-icon>
          </button>
        </div>
      </div>

      <div class="p-3 space-y-3">
        <div v-if="data.url" class="space-y-2">
          <audio :src="data.url" controls class="w-full h-10" />
          <div class="text-xs text-[var(--text-secondary)] truncate">{{ data.fileName || '参考音频' }}</div>
        </div>
        <div v-else class="rounded-lg bg-[var(--bg-tertiary)] border-2 border-dashed border-[var(--border-color)] p-3">
          <div class="flex flex-col items-center justify-center gap-2 relative cursor-pointer hover:bg-[var(--bg-secondary)] rounded-lg transition-colors py-4">
            <n-icon :size="28" class="text-[var(--text-secondary)]"><MusicalNotesOutline /></n-icon>
            <span class="text-sm text-[var(--text-secondary)]">上传参考音频</span>
            <input type="file" accept="audio/*" class="absolute inset-0 opacity-0 cursor-pointer" @change="handleFileUpload" />
          </div>
          <div class="flex items-center gap-2 my-3">
            <div class="flex-1 h-px bg-[var(--border-color)]"></div>
            <span class="text-xs text-[var(--text-secondary)]">或</span>
            <div class="flex-1 h-px bg-[var(--border-color)]"></div>
          </div>
          <div class="flex gap-2">
            <input
              v-model="urlInput"
              type="text"
              placeholder="输入音频地址..."
              class="flex-1 px-2 py-1 text-sm bg-[var(--bg-secondary)] border border-[var(--border-color)] rounded-lg outline-none focus:border-[var(--accent-color)] text-[var(--text-primary)] placeholder:text-[var(--text-secondary)]"
              @keydown.enter="handleUrlSubmit"
            />
            <button
              @click="handleUrlSubmit"
              :disabled="!urlInput.trim()"
              class="px-3 py-2 text-xs bg-[var(--accent-color)] hover:bg-[var(--accent-hover)] text-white rounded-lg transition-colors disabled:opacity-50 disabled:cursor-not-allowed whitespace-nowrap"
            >
              预览
            </button>
          </div>
        </div>
      </div>

      <NodeHandleMenu :nodeId="id" nodeType="audio" :visible="showHandleMenu" :operations="operations" @select="handleSelect" />
      <Handle type="target" :position="Position.Left" id="left" class="!bg-[var(--accent-color)]" />
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { Handle, Position, useVueFlow } from '@vue-flow/core'
import { NIcon } from 'naive-ui'
import { MusicalNotesOutline, TrashOutline, CopyOutline, VideocamOutline } from '@vicons/ionicons5'
import { updateNode, removeNode, duplicateNode, addNode, addEdge, nodes } from '../../stores/canvas'
import NodeHandleMenu from './NodeHandleMenu.vue'

const props = defineProps({
  id: String,
  data: Object
})

const { updateNodeInternals } = useVueFlow()
const showHandleMenu = ref(false)
const isEditingLabel = ref(false)
const editingLabelValue = ref('')
const labelInputRef = ref(null)
const urlInput = ref('')

const operations = [
  { type: 'videoConfig', label: '生视频', icon: VideocamOutline, action: 'audio_videoConfig' }
]

const fileToBase64 = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(reader.result)
    reader.onerror = reject
    reader.readAsDataURL(file)
  })
}

const handleFileUpload = async (event) => {
  const file = event.target.files?.[0]
  if (!file) return

  try {
    const base64 = await fileToBase64(file)
    updateNode(props.id, {
      url: base64,
      base64,
      fileName: file.name,
      fileType: file.type,
      label: '参考音频',
      updatedAt: Date.now()
    })
  } catch (err) {
    window.$message?.error(err.message || '音频上传失败')
  }
}

const handleUrlSubmit = () => {
  const url = urlInput.value.trim()
  if (!url) return
  if (!url.startsWith('http://') && !url.startsWith('https://')) {
    window.$message?.warning('请输入有效的音频地址 (http:// 或 https://)')
    return
  }

  updateNode(props.id, {
    url,
    label: '参考音频',
    updatedAt: Date.now()
  })
  urlInput.value = ''
}

const handleSelect = (item) => {
  if (item.action !== 'audio_videoConfig') return

  const currentNode = nodes.value.find(n => n.id === props.id)
  const nodeX = currentNode?.position?.x || 0
  const nodeY = currentNode?.position?.y || 0

  const textNodeId = addNode('text', { x: nodeX + 300, y: nodeY - 100 }, {
    content: '',
    label: '提示词'
  })

  const configNodeId = addNode('videoConfig', { x: nodeX + 600, y: nodeY }, {
    label: '视频生成'
  })

  addEdge({
    source: props.id,
    target: configNodeId,
    sourceHandle: 'right',
    targetHandle: 'left'
  })

  addEdge({
    source: textNodeId,
    target: configNodeId,
    sourceHandle: 'right',
    targetHandle: 'left'
  })

  setTimeout(() => updateNodeInternals([textNodeId, configNodeId]), 50)
  window.$message?.success('已创建音频参考视频工作流')
}

const startEditLabel = () => {
  editingLabelValue.value = props.data?.label || '音频节点'
  isEditingLabel.value = true
  nextTick(() => {
    labelInputRef.value?.focus()
    labelInputRef.value?.select()
  })
}

const finishEditLabel = () => {
  const newLabel = editingLabelValue.value.trim()
  if (newLabel && newLabel !== props.data?.label) {
    updateNode(props.id, { label: newLabel })
  }
  isEditingLabel.value = false
}

const cancelEditLabel = () => {
  isEditingLabel.value = false
}

const handleDelete = () => {
  removeNode(props.id)
}

const handleDuplicate = () => {
  const newId = duplicateNode(props.id)
  if (newId) {
    updateNode(props.id, { selected: false })
    updateNode(newId, { selected: true })
    window.$message?.success('节点已复制')
  }
}
</script>

<style scoped>
.audio-node-wrapper {
  padding-top: 20px;
}
</style>
