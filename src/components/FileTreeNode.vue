<template>
  <div
    class="tree-node"
    :class="[
      `tree-node--${node.type}`,
      { 'is-expanded': isExpanded, 'is-active': isActive }
    ]"
  >
    <button
      class="tree-node__label"
      type="button"
      :style="{ paddingLeft: `${12 + level * 14}px` }"
      role="treeitem"
      :aria-level="level + 1"
      :aria-expanded="isDirectory ? isExpanded : undefined"
      @click="handleClick"
    >
      <span
        v-if="isDirectory"
        class="tree-node__caret"
        :class="{ 'is-expanded': isExpanded }"
        aria-hidden="true"
      ></span>
      <i class="codicon" :class="iconClass" aria-hidden="true"></i>
      <span class="tree-node__text">{{ node.name }}</span>
    </button>

    <transition name="tree-reveal">
      <div v-if="isDirectory && isExpanded" class="tree-node__children" role="group">
        <FileTreeNode
          v-for="child in node.children"
          :key="child.path"
          :node="child"
          :level="level + 1"
          :active-path="activePath"
          :expanded-paths="expandedPaths"
          :is-search-active="isSearchActive"
          @toggle="emit('toggle', $event)"
          @select="emit('select', $event)"
        />
      </div>
    </transition>
  </div>
</template>

<script setup>
import { computed } from 'vue'

defineOptions({
  name: 'FileTreeNode',
})

const props = defineProps({
  node: {
    type: Object,
    required: true,
  },
  level: {
    type: Number,
    default: 0,
  },
  activePath: {
    type: String,
    default: '',
  },
  expandedPaths: {
    type: Object,
    required: true,
  },
  isSearchActive: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits(['toggle', 'select'])

const isDirectory = computed(() => props.node.type === 'directory')
const isExpanded = computed(() => {
  if (!isDirectory.value) {
    return false
  }
  if (props.isSearchActive) {
    return true
  }
  return props.expandedPaths.has(props.node.path)
})
const isActive = computed(() => props.activePath === props.node.path)

const iconClass = computed(() => {
  if (isDirectory.value) {
    return isExpanded.value ? 'codicon-folder-opened' : 'codicon-folder'
  }
  return 'codicon-file-code'
})

function handleClick() {
  if (isDirectory.value) {
    emit('toggle', props.node.path)
    return
  }
  emit('select', props.node)
}
</script>

<style scoped>
.tree-node {
  display: flex;
  flex-direction: column;
  color: var(--text-soft);
}

.tree-node__label {
  appearance: none;
  border: none;
  background: transparent;
  color: inherit;
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  padding: 6px 10px;
  width: 100%;
  border-radius: 10px;
  transition: background 0.2s ease, color 0.2s ease;
  text-align: left;
}

.tree-node__label:hover,
.tree-node.is-active > .tree-node__label,
.tree-node__label:focus-visible {
  background: rgba(79, 156, 255, 0.12);
  color: var(--text-strong);
}

.tree-node__caret {
  width: 10px;
  height: 10px;
  border-right: 1px solid currentColor;
  border-bottom: 1px solid currentColor;
  transform: rotate(-45deg);
  transition: transform 0.2s ease;
}

.tree-node__caret.is-expanded {
  transform: rotate(45deg);
}

.tree-node__text {
  flex: 1;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.tree-node__children {
  display: flex;
  flex-direction: column;
}

.tree-reveal-enter-active,
.tree-reveal-leave-active {
  transition: all 0.14s ease;
}

.tree-reveal-enter-from,
.tree-reveal-leave-to {
  opacity: 0;
  max-height: 0;
}

.tree-reveal-enter-to,
.tree-reveal-leave-from {
  opacity: 1;
  max-height: 600px;
}
</style>
