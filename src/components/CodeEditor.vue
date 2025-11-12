<template>
  <div class="code-editor" ref="container"></div>
</template>

<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: String,
    default: '',
  },
  language: {
    type: String,
    default: 'plaintext',
  },
  path: {
    type: String,
    default: '',
  },
  readOnly: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits(['update:modelValue'])

const container = ref(null)
let editorInstance = null
let monacoInstance = null
let resizeObserver = null
let isApplyingExternalChange = false
let isEmittingChange = false

const MONACO_VERSION = '0.52.0'
const MONACO_BASE_URL = `https://cdnjs.cloudflare.com/ajax/libs/monaco-editor/${MONACO_VERSION}/min`
let loaderPromise = null

function loadMonacoFromCdn() {
  if (typeof window === 'undefined') {
    return Promise.reject(new Error('Monaco só pode ser carregado no navegador.'))
  }

  if (window.monaco) {
    return Promise.resolve(window.monaco)
  }

  if (loaderPromise) {
    return loaderPromise
  }

  loaderPromise = new Promise((resolve, reject) => {
    const existingScript = document.getElementById('monaco-loader-script')

    function onLoaderReady() {
      if (!window.require) {
        reject(new Error('AMD loader do Monaco não foi encontrado.'))
        return
      }

      window.require.config({
        paths: {
          vs: `${MONACO_BASE_URL}/vs`,
        },
      })

      window.MonacoEnvironment = {
        getWorkerUrl() {
          const workerSource = `self.MonacoEnvironment={baseUrl:'${MONACO_BASE_URL}/'};importScripts('${MONACO_BASE_URL}/vs/base/worker/workerMain.js');`
          return `data:text/javascript;charset=utf-8,${encodeURIComponent(workerSource)}`
        },
      }

      window.require(['vs/editor/editor.main'], () => {
        resolve(window.monaco)
      }, reject)
    }

    if (existingScript) {
      if (existingScript.dataset.loaded === 'true') {
        onLoaderReady()
      } else {
        existingScript.addEventListener('load', onLoaderReady, { once: true })
        existingScript.addEventListener('error', reject, { once: true })
      }
      return
    }

    const script = document.createElement('script')
    script.id = 'monaco-loader-script'
    script.src = `${MONACO_BASE_URL}/vs/loader.min.js`
    script.async = true
    script.dataset.loaded = 'false'
    script.addEventListener('load', () => {
      script.dataset.loaded = 'true'
      onLoaderReady()
    }, { once: true })
    script.addEventListener('error', reject, { once: true })
    document.body.appendChild(script)
  })

  return loaderPromise
}

function createModel(path, language, value) {
  if (!monacoInstance) {
    return null
  }

  const safePath = path ? path.replace(/^\//, '') : 'untitled'
  const uri = monacoInstance.Uri.parse(`file:///${safePath}`)
  let model = monacoInstance.editor.getModel(uri)

  if (!model) {
    model = monacoInstance.editor.createModel(value ?? '', language || 'plaintext', uri)
  } else if (typeof value === 'string' && model.getValue() !== value) {
    model.setValue(value)
  }

  monacoInstance.editor.setModelLanguage(model, language || 'plaintext')

  return model
}

function setModel() {
  if (!editorInstance || !monacoInstance) {
    return
  }

  const model = createModel(props.path, props.language, props.modelValue)
  if (model) {
    editorInstance.setModel(model)
  }
}

function applyExternalValue(value) {
  if (!editorInstance) {
    return
  }
  const model = editorInstance.getModel()
  if (model && model.getValue() !== value) {
    isApplyingExternalChange = true
    model.setValue(value ?? '')
    isApplyingExternalChange = false
  }
}

onMounted(async () => {
  try {
    monacoInstance = await loadMonacoFromCdn()
  } catch (error) {
    console.error('Não foi possível carregar o Monaco Editor.', error)
    return
  }

  if (!container.value) {
    return
  }

  editorInstance = monacoInstance.editor.create(container.value, {
    value: props.modelValue ?? '',
    language: props.language || 'plaintext',
    theme: 'vs-dark',
    readOnly: props.readOnly,
    automaticLayout: false,
    minimap: {
      enabled: true,
    },
    fontSize: 13,
    scrollBeyondLastLine: false,
    smoothScrolling: true,
    tabSize: 2,
  })

  setModel()

  editorInstance.onDidChangeModelContent(() => {
    if (isApplyingExternalChange) {
      return
    }
    const value = editorInstance.getValue()
    if (value !== props.modelValue) {
      isEmittingChange = true
      emit('update:modelValue', value)
      isEmittingChange = false
    }
  })

  resizeObserver = new ResizeObserver(() => {
    editorInstance?.layout()
  })

  resizeObserver.observe(container.value)
})

watch(() => props.path, () => {
  if (editorInstance) {
    setModel()
  }
})

watch(() => props.language, (language) => {
  if (!editorInstance || !monacoInstance) {
    return
  }
  const model = editorInstance.getModel()
  if (model) {
    monacoInstance.editor.setModelLanguage(model, language || 'plaintext')
  }
})

watch(() => props.modelValue, (value) => {
  if (!editorInstance || isEmittingChange) {
    return
  }
  applyExternalValue(value ?? '')
})

watch(() => props.readOnly, (readOnly) => {
  editorInstance?.updateOptions({ readOnly })
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
  editorInstance?.dispose()
})
</script>

<style scoped>
.code-editor {
  width: 100%;
  height: 100%;
  position: relative;
}
</style>
