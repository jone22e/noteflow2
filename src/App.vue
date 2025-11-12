<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, shallowRef } from 'vue'
import CodeEditor from './components/CodeEditor.vue'
import FileTreeNode from './components/FileTreeNode.vue'

const token = ref('')
const connectionState = ref('disconnected')
const connectionError = ref('')
const userProfile = ref(null)
const repos = ref([])
const repoFilter = ref('')
const selectedRepo = ref(null)
const fileTree = ref([])
const fileSearch = ref('')
const statusMessage = ref('Conecte-se ao GitHub para começar.')
const isLoadingRepos = ref(false)
const isLoadingFiles = ref(false)
const isLoadingFileContent = ref(false)
const fileError = ref('')
const showRepoPanel = ref(true)
const expandedPaths = shallowRef(new Set())
const openFiles = reactive([])
const activeFilePath = ref('')
const savedRepoName = ref('')

const actionButtons = [
  { label: 'Explorer', icon: 'codicon-files' },
  { label: 'Search', icon: 'codicon-search' },
  { label: 'Source Control', icon: 'codicon-source-control' },
  { label: 'Deploy', icon: 'codicon-cloud-upload' },
  { label: 'Settings', icon: 'codicon-gear' },
]

const panelSizes = reactive({
  repositories: 320,
  files: 340,
  editor: 640,
  context: 420,
})

const defaultRepositoryWidth = panelSizes.repositories

const resizing = ref(null)
const minPanelWidth = 220

const filteredRepos = computed(() => {
  if (!repoFilter.value.trim()) {
    return repos.value
  }
  const term = repoFilter.value.trim().toLowerCase()
  return repos.value.filter((repo) => repo.name.toLowerCase().includes(term) || repo.full_name.toLowerCase().includes(term))
})

const isSearchActive = computed(() => !!fileSearch.value.trim())

const filteredFileTree = computed(() => {
  const term = fileSearch.value.trim().toLowerCase()
  if (!term) {
    return fileTree.value
  }

  function filterNodes(nodes) {
    const result = []
    for (const node of nodes) {
      if (node.type === 'directory') {
        const filteredChildren = filterNodes(node.children)
        if (filteredChildren.length) {
          result.push({ ...node, children: filteredChildren })
          continue
        }
        if (node.name.toLowerCase().includes(term)) {
          result.push({ ...node, children: [] })
        }
      } else if (node.path.toLowerCase().includes(term)) {
        result.push({ ...node })
      }
    }
    return result
  }

  return filterNodes(fileTree.value)
})

const activeFile = computed(() => openFiles.find((entry) => entry.path === activeFilePath.value) || null)

const activeFileLanguage = computed(() => activeFile.value?.language || 'plaintext')

function detectLanguage(path) {
  if (!path) {
    return 'plaintext'
  }

  const normalized = path.toLowerCase()
  if (normalized === 'dockerfile') {
    return 'dockerfile'
  }
  if (normalized.endsWith('.d.ts')) {
    return 'typescript'
  }

  const extensionMatch = normalized.match(/\.([^.]+)$/)
  const extension = extensionMatch ? extensionMatch[1] : ''

  const map = {
    js: 'javascript',
    cjs: 'javascript',
    mjs: 'javascript',
    jsx: 'javascript',
    ts: 'typescript',
    tsx: 'typescript',
    json: 'json',
    jsonc: 'json',
    vue: 'html',
    html: 'html',
    css: 'css',
    scss: 'scss',
    less: 'less',
    sass: 'scss',
    md: 'markdown',
    markdown: 'markdown',
    yaml: 'yaml',
    yml: 'yaml',
    py: 'python',
    rb: 'ruby',
    php: 'php',
    go: 'go',
    java: 'java',
    kt: 'kotlin',
    swift: 'swift',
    rs: 'rust',
    cs: 'csharp',
    cpp: 'cpp',
    cxx: 'cpp',
    c: 'c',
    h: 'cpp',
    hpp: 'cpp',
    sql: 'sql',
    sh: 'shell',
    bash: 'shell',
    zsh: 'shell',
    ps1: 'powershell',
    xml: 'xml',
    txt: 'plaintext',
    ini: 'ini',
    env: 'ini',
    cfg: 'ini',
    toml: 'toml',
    dockerfile: 'dockerfile',
  }

  return map[extension] || 'plaintext'
}

function buildFileTree(blobs) {
  const rootNodes = []
  const directoryMap = new Map()
  directoryMap.set('', { children: rootNodes })

  for (const blob of blobs) {
    const segments = blob.path.split('/')
    let parent = directoryMap.get('')
    let currentPath = ''

    segments.forEach((segment, index) => {
      const isLast = index === segments.length - 1
      currentPath = currentPath ? `${currentPath}/${segment}` : segment

      if (isLast) {
        parent.children.push({
          type: 'file',
          name: segment,
          path: blob.path,
          sha: blob.sha,
          size: blob.size,
        })
        return
      }

      let directory = directoryMap.get(currentPath)
      if (!directory) {
        directory = {
          type: 'directory',
          name: segment,
          path: currentPath,
          children: [],
        }
        parent.children.push(directory)
        directoryMap.set(currentPath, directory)
      }
      parent = directory
    })
  }

  function sortChildren(children) {
    children.sort((a, b) => {
      if (a.type === b.type) {
        return a.name.localeCompare(b.name)
      }
      return a.type === 'directory' ? -1 : 1
    })

    children.forEach((child) => {
      if (child.type === 'directory') {
        sortChildren(child.children)
      }
    })
  }

  sortChildren(rootNodes)
  return rootNodes
}

function toggleDirectory(path) {
  const next = new Set(expandedPaths.value)
  if (next.has(path)) {
    next.delete(path)
  } else {
    next.add(path)
  }
  expandedPaths.value = next
}

function ensureExpandedForPath(path) {
  if (!path) {
    return
  }
  const segments = path.split('/')
  if (segments.length <= 1) {
    return
  }
  const next = new Set(expandedPaths.value)
  let currentPath = ''
  for (let index = 0; index < segments.length - 1; index += 1) {
    currentPath = currentPath ? `${currentPath}/${segments[index]}` : segments[index]
    next.add(currentPath)
  }
  expandedPaths.value = next
}

function setActiveTab(path) {
  activeFilePath.value = path
  ensureExpandedForPath(path)
}

function closeTab(path) {
  const index = openFiles.findIndex((entry) => entry.path === path)
  if (index === -1) {
    return
  }
  openFiles.splice(index, 1)

  if (activeFilePath.value === path) {
    if (openFiles.length) {
      const nextIndex = index > 0 ? index - 1 : 0
      const nextPath = openFiles[nextIndex]?.path || ''
      activeFilePath.value = nextPath
      ensureExpandedForPath(nextPath)
    } else {
      activeFilePath.value = ''
    }
  }
}

function updateActiveFileContent(value) {
  const file = activeFile.value
  if (!file) {
    return
  }
  file.content = value
  file.isDirty = value !== file.originalContent
}

function collapseRepositoryPanel() {
  if (!showRepoPanel.value) {
    return
  }
  const width = panelSizes.repositories
  panelSizes.repositories = 0
  panelSizes.files += width
  showRepoPanel.value = false
}

function revealRepositorySelector() {
  if (showRepoPanel.value) {
    return
  }

  let desiredWidth = Math.max(defaultRepositoryWidth, minPanelWidth)
  let remaining = desiredWidth

  const adjustPanel = (key) => {
    if (remaining <= 0) {
      return
    }
    const available = panelSizes[key] - minPanelWidth
    if (available <= 0) {
      return
    }
    const take = Math.min(available, remaining)
    panelSizes[key] -= take
    remaining -= take
  }

  adjustPanel('files')
  adjustPanel('editor')
  adjustPanel('context')

  panelSizes.repositories = Math.max(minPanelWidth, desiredWidth - remaining)
  showRepoPanel.value = true
}

function toggleRepositorySelector() {
  if (showRepoPanel.value) {
    collapseRepositoryPanel()
  } else {
    revealRepositorySelector()
  }
}

function startResize(currentKey, nextKey, event) {
  event.preventDefault()
  const total = panelSizes[currentKey] + panelSizes[nextKey]
  resizing.value = {
    currentKey,
    nextKey,
    startX: event.clientX,
    total,
    startCurrent: panelSizes[currentKey],
  }
  window.addEventListener('pointermove', handlePointerMove)
  window.addEventListener('pointerup', stopResize)
}

function handlePointerMove(event) {
  const state = resizing.value
  if (!state) {
    return
  }
  const delta = event.clientX - state.startX
  let newCurrent = state.startCurrent + delta
  const maxCurrent = state.total - minPanelWidth
  if (newCurrent < minPanelWidth) {
    newCurrent = minPanelWidth
  } else if (newCurrent > maxCurrent) {
    newCurrent = maxCurrent
  }
  const newNext = state.total - newCurrent
  panelSizes[state.currentKey] = newCurrent
  panelSizes[state.nextKey] = newNext
}

function stopResize() {
  if (!resizing.value) {
    return
  }
  window.removeEventListener('pointermove', handlePointerMove)
  window.removeEventListener('pointerup', stopResize)
  resizing.value = null
}

function buildHeaders() {
  const trimmedToken = token.value.trim()
  if (!trimmedToken) {
    throw new Error('Informe um token do GitHub para continuar.')
  }
  return {
    Accept: 'application/vnd.github+json',
    Authorization: `Bearer ${trimmedToken}`,
  }
}

async function fetchAllRepositories(headers) {
  const collected = []
  let page = 1
  while (page <= 5) {
    const response = await fetch(`https://api.github.com/user/repos?per_page=100&page=${page}&sort=updated`, {
      headers,
    })
    if (!response.ok) {
      throw new Error('Não foi possível carregar os repositórios. Verifique o token e tente novamente.')
    }
    const data = await response.json()
    collected.push(...data)
    if (data.length < 100) {
      break
    }
    page += 1
  }
  collected.sort((a, b) => new Date(b.updated_at).getTime() - new Date(a.updated_at).getTime())
  return collected
}

async function connectGitHub(auto = false) {
  if (!token.value.trim()) {
    connectionError.value = 'Informe um token pessoal do GitHub (PAT) com permissão de leitura.'
    connectionState.value = 'disconnected'
    return
  }

  connectionState.value = 'connecting'
  connectionError.value = ''
  isLoadingRepos.value = true

  try {
    const headers = buildHeaders()
    const userResponse = await fetch('https://api.github.com/user', { headers })
    if (!userResponse.ok) {
      throw new Error('Token inválido ou sem permissões de leitura.')
    }
    const userData = await userResponse.json()
    userProfile.value = userData

    const reposData = await fetchAllRepositories(headers)
    repos.value = reposData
    statusMessage.value = 'Selecione um repositório para visualizar os arquivos.'
    connectionState.value = 'connected'
    localStorage.setItem('codex:github-token', token.value.trim())

    if (!auto) {
      savedRepoName.value = localStorage.getItem('codex:last-repo') || ''
    }

    if (savedRepoName.value) {
      const repo = reposData.find((item) => item.full_name === savedRepoName.value)
      if (repo) {
        await selectRepository(repo)
      }
    }
  } catch (error) {
    connectionState.value = 'disconnected'
    connectionError.value = error instanceof Error ? error.message : 'Não foi possível conectar ao GitHub.'
    userProfile.value = null
    repos.value = []
  } finally {
    isLoadingRepos.value = false
  }
}

async function selectRepository(repo) {
  selectedRepo.value = repo
  savedRepoName.value = repo.full_name
  localStorage.setItem('codex:last-repo', repo.full_name)
  await loadRepositoryFiles()
  collapseRepositoryPanel()
}

async function loadRepositoryFiles() {
  if (!selectedRepo.value) {
    return
  }

  isLoadingFiles.value = true
  fileTree.value = []
  fileSearch.value = ''
  openFiles.splice(0, openFiles.length)
  activeFilePath.value = ''
  fileError.value = ''
  expandedPaths.value = new Set()

  try {
    const headers = buildHeaders()
    const branch = selectedRepo.value.default_branch || 'main'
    const treeResponse = await fetch(`https://api.github.com/repos/${selectedRepo.value.full_name}/git/trees/${branch}?recursive=1`, {
      headers,
    })
    if (!treeResponse.ok) {
      throw new Error('Falha ao carregar a árvore de arquivos do repositório.')
    }
    const treeData = await treeResponse.json()
    if (!Array.isArray(treeData.tree)) {
      throw new Error('Resposta inesperada da API do GitHub.')
    }
    const blobs = treeData.tree.filter((node) => node.type === 'blob')
    blobs.sort((a, b) => a.path.localeCompare(b.path))
    fileTree.value = buildFileTree(blobs)
    statusMessage.value = `${blobs.length} arquivos carregados de ${selectedRepo.value.name}.`
    if (!blobs.length) {
      statusMessage.value = 'Nenhum arquivo encontrado neste repositório.'
    }
  } catch (error) {
    fileTree.value = []
    statusMessage.value = error instanceof Error ? error.message : 'Não foi possível carregar os arquivos.'
  } finally {
    isLoadingFiles.value = false
  }
}

async function openFile(entry) {
  if (!selectedRepo.value) {
    return
  }

  if (entry.size && entry.size > 1024 * 1024) {
    fileError.value = 'Arquivos maiores que 1MB não são carregados para proteger o desempenho.'
    statusMessage.value = fileError.value
    return
  }

  ensureExpandedForPath(entry.path)

  let tab = openFiles.find((item) => item.path === entry.path)
  if (!tab) {
    tab = {
      path: entry.path,
      name: entry.name || entry.path.split('/').pop(),
      content: '',
      originalContent: '',
      language: detectLanguage(entry.path),
      isDirty: false,
      isLoading: true,
      hasLoaded: false,
    }
    openFiles.push(tab)
  } else {
    tab.name = entry.name || entry.path.split('/').pop()
    tab.language = detectLanguage(entry.path)
  }

  activeFilePath.value = entry.path
  fileError.value = ''

  if (tab.hasLoaded) {
    tab.isLoading = false
    statusMessage.value = `${entry.path} aberto.`
    return
  }

  tab.isLoading = true
  isLoadingFileContent.value = true

  try {
    const headers = buildHeaders()
    const encodedPath = entry.path
      .split('/')
      .map((segment) => encodeURIComponent(segment))
      .join('/')
    const fileResponse = await fetch(`https://api.github.com/repos/${selectedRepo.value.full_name}/contents/${encodedPath}`, {
      headers,
    })
    if (!fileResponse.ok) {
      throw new Error('Não foi possível abrir o arquivo selecionado.')
    }
    const fileData = await fileResponse.json()
    let content = ''
    if (fileData.encoding === 'base64') {
      content = atob((fileData.content || '').replace(/\n/g, ''))
    } else if (typeof fileData.content === 'string') {
      content = fileData.content
    }
    tab.content = content
    tab.originalContent = content
    tab.isDirty = false
    tab.isLoading = false
    tab.hasLoaded = true
    statusMessage.value = `${entry.path} carregado com sucesso.`
  } catch (error) {
    tab.isLoading = false
    fileError.value = error instanceof Error ? error.message : 'Erro desconhecido ao carregar o arquivo.'
    statusMessage.value = fileError.value
  } finally {
    isLoadingFileContent.value = false
  }
}

onMounted(() => {
  const storedToken = localStorage.getItem('codex:github-token')
  const storedRepo = localStorage.getItem('codex:last-repo')
  if (storedToken) {
    token.value = storedToken
    savedRepoName.value = storedRepo || ''
    connectGitHub(true)
  }
})

onBeforeUnmount(() => {
  stopResize()
})
</script>

<template>
  <div class="codex-shell">
    <aside class="toolbar">
      <div class="toolbar__brand" aria-label="Codex web">⌘</div>
      <div class="toolbar__buttons">
        <button
          v-for="button in actionButtons"
          :key="button.label"
          class="toolbar__button"
          type="button"
          :title="button.label"
        >
          <i class="codicon" :class="button.icon" aria-hidden="true"></i>
          <span class="sr-only">{{ button.label }}</span>
        </button>
      </div>
    </aside>

    <div class="workspace">
      <div v-if="selectedRepo" class="workspace-actions">
        <button class="repo-toggle" type="button" @click="toggleRepositorySelector">
          <i class="codicon codicon-repo"></i>
          <span>{{ showRepoPanel ? 'Ocultar repositórios' : 'Selecionar repositório' }}</span>
        </button>
      </div>

      <section
        v-if="showRepoPanel || !selectedRepo"
        class="panel repositories"
        :style="{ width: `${panelSizes.repositories}px` }"
      >
        <div class="panel-block connection-card">
          <label for="github-token">Token pessoal do GitHub</label>
          <div class="token-row">
            <input
              id="github-token"
              v-model="token"
              :disabled="connectionState === 'connecting'"
              autocomplete="off"
              autocapitalize="off"
              spellcheck="false"
              type="password"
              placeholder="ghp_xxxxxxxxxxxxxxxxxxxx"
            />
            <button
              class="primary"
              type="button"
              :disabled="connectionState === 'connecting' || !token"
              @click="connectGitHub()"
            >
              {{ connectionState === 'connected' ? 'Reconectar' : 'Conectar' }}
            </button>
          </div>
          <p class="hint">Use um token com permissão <strong>repo:read</strong>. Ele é mantido apenas localmente neste navegador.</p>
          <p v-if="connectionError" class="error">{{ connectionError }}</p>
          <p v-if="userProfile" class="success">
            Conectado como <strong>{{ userProfile.login }}</strong>
          </p>
        </div>

        <div class="panel-block list-section" :class="{ 'list-section--disabled': connectionState !== 'connected' }">
          <input
            v-model="repoFilter"
            :disabled="connectionState !== 'connected'"
            class="search-input"
            placeholder="Filtrar repositórios"
            type="search"
          />
          <div class="repo-list" role="list">
            <button
              v-for="repo in filteredRepos"
              :key="repo.id"
              class="repo-item"
              role="listitem"
              type="button"
              :class="{ 'repo-item--active': selectedRepo && repo.id === selectedRepo.id }"
              @click="selectRepository(repo)"
            >
              <div>
                <strong>{{ repo.name }}</strong>
                <p>{{ repo.full_name }}</p>
              </div>
              <span class="repo-meta">{{ new Date(repo.updated_at).toLocaleDateString() }}</span>
            </button>
            <p v-if="!filteredRepos.length && connectionState === 'connected'" class="empty">Nenhum repositório encontrado.</p>
          </div>
        </div>
      </section>

      <div
        v-if="showRepoPanel || !selectedRepo"
        class="resize-handle"
        role="separator"
        aria-orientation="vertical"
        @pointerdown="(event) => startResize('repositories', 'files', event)"
      ></div>

      <section class="panel files" :style="{ width: `${panelSizes.files}px` }">
        <div class="panel-block status-banner">
          <span>{{ statusMessage }}</span>
          <div class="status-flags">
            <span v-if="isLoadingRepos" class="tag">conectando…</span>
            <span v-if="isLoadingFiles" class="tag">atualizando…</span>
          </div>
        </div>
        <div class="panel-block list-section">
          <input
            v-model="fileSearch"
            :disabled="!selectedRepo || isLoadingFiles"
            class="search-input"
            placeholder="Filtrar arquivos"
            type="search"
          />
          <div class="file-tree" role="tree">
            <template v-if="filteredFileTree.length">
              <FileTreeNode
                v-for="node in filteredFileTree"
                :key="node.path"
                :node="node"
                :level="0"
                :active-path="activeFilePath"
                :expanded-paths="expandedPaths.value"
                :is-search-active="isSearchActive"
                @toggle="toggleDirectory"
                @select="openFile"
              />
            </template>
            <p
              v-if="selectedRepo && !filteredFileTree.length && !isLoadingFiles"
              class="empty"
            >
              Nenhum arquivo encontrado.
            </p>
            <p v-if="!selectedRepo" class="empty">Selecione um repositório para visualizar a árvore.</p>
          </div>
        </div>
      </section>

      <div
        class="resize-handle"
        role="separator"
        aria-orientation="vertical"
        @pointerdown="(event) => startResize('files', 'editor', event)"
      ></div>

      <section class="panel editor" :style="{ width: `${panelSizes.editor}px` }">
        <div class="editor-topbar">
          <div class="tab-bar" role="tablist">
            <button
              v-for="file in openFiles"
              :key="file.path"
              class="tab"
              type="button"
              role="tab"
              :aria-selected="file.path === activeFilePath"
              :class="{ 'tab--active': file.path === activeFilePath, 'tab--dirty': file.isDirty }"
              @click="setActiveTab(file.path)"
            >
              <span class="tab__title">{{ file.name }}</span>
              <span v-if="file.isDirty" class="tab__dirty" aria-hidden="true">•</span>
              <span v-if="file.isLoading" class="tab__status codicon codicon-sync"></span>
              <span
                class="tab__close"
                role="button"
                tabindex="0"
                @click.stop="closeTab(file.path)"
                @keydown.enter.stop.prevent="closeTab(file.path)"
                @keydown.space.stop.prevent="closeTab(file.path)"
              >
                <i class="codicon codicon-close"></i>
                <span class="sr-only">Fechar {{ file.name }}</span>
              </span>
            </button>
            <p v-if="!openFiles.length" class="tab-empty">Nenhum arquivo aberto</p>
          </div>
          <div class="editor-topbar__meta">
            <span class="editor-breadcrumb">{{ activeFile ? activeFile.path : 'Nenhum arquivo aberto' }}</span>
            <span v-if="isLoadingFileContent" class="tag">carregando…</span>
          </div>
        </div>
        <div class="editor-surface">
          <div v-if="!activeFile" class="editor-placeholder">
            <p>Selecione um arquivo para exibir o conteúdo aqui.</p>
          </div>
          <div v-else class="editor-canvas">
            <CodeEditor
              :model-value="activeFile.content"
              :language="activeFileLanguage"
              :path="activeFile.path"
              @update:modelValue="updateActiveFileContent"
            />
            <div v-if="activeFile.isLoading" class="editor-loading" role="status">
              <span class="codicon codicon-sync"></span>
              <span>Carregando conteúdo…</span>
            </div>
          </div>
        </div>
        <p v-if="fileError" class="error error--inline">{{ fileError }}</p>
      </section>

      <div
        class="resize-handle"
        role="separator"
        aria-orientation="vertical"
        @pointerdown="(event) => startResize('editor', 'context', event)"
      ></div>

      <section class="panel webview" :style="{ width: `${panelSizes.context}px` }">
        <iframe
          src="https://chatgpt.com/codex"
          title="Codex web preview"
          loading="lazy"
          referrerpolicy="no-referrer"
        ></iframe>
      </section>
    </div>
  </div>
</template>

<style scoped>
.codex-shell {
  flex: 1;
  display: flex;
  height: 100vh;
  background: linear-gradient(135deg, var(--surface-0), var(--surface-2));
  color: var(--text-soft);
}

.toolbar {
  background: rgba(6, 10, 18, 0.9);
  border-right: 1px solid var(--color-border);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 28px 14px;
  gap: 20px;
}

.toolbar__brand {
  width: 40px;
  height: 40px;
  border-radius: 12px;
  background: linear-gradient(145deg, var(--accent-primary), var(--accent-secondary));
  display: grid;
  place-items: center;
  color: var(--surface-0);
  font-weight: 700;
  font-size: 18px;
  box-shadow: var(--shadow-elevated);
}

.toolbar__buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.toolbar__button {
  width: 40px;
  height: 40px;
  border-radius: 10px;
  display: grid;
  place-items: center;
  color: var(--text-muted);
  background: transparent;
  transition: background 0.2s ease, color 0.2s ease;
}

.toolbar__button .codicon {
  font-size: 20px;
}

.toolbar__button:hover,
.toolbar__button:focus-visible {
  background: rgba(79, 156, 255, 0.12);
  color: var(--text-strong);
}

.workspace {
  flex: 1;
  display: flex;
  overflow: hidden;
  position: relative;
}

.workspace-actions {
  position: absolute;
  top: 18px;
  right: 24px;
  z-index: 3;
}

.repo-toggle {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  background: rgba(9, 14, 23, 0.85);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 8px 14px;
  color: var(--text-soft);
  font-size: 12px;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  transition: background 0.2s ease, border-color 0.2s ease, color 0.2s ease;
}

.repo-toggle:hover,
.repo-toggle:focus-visible {
  background: rgba(79, 156, 255, 0.15);
  border-color: var(--accent-primary);
  color: var(--text-strong);
}

.panel {
  display: flex;
  flex-direction: column;
  gap: 24px;
  padding: 24px 20px;
  background: rgba(14, 21, 34, 0.7);
  backdrop-filter: blur(18px);
  border-right: 1px solid var(--color-border);
  overflow: hidden;
}

.panel:last-of-type {
  border-right: none;
}

.panel-block {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.connection-card {
  background: var(--surface-2);
  border: 1px solid var(--color-border);
  border-radius: 16px;
  padding: 18px;
  box-shadow: var(--shadow-elevated);
}

.connection-card label {
  font-size: 12px;
  color: var(--text-muted);
}

.token-row {
  display: flex;
  align-items: center;
  gap: 12px;
}

.token-row input {
  flex: 1;
  background: var(--surface-3);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 10px 14px;
  color: var(--text-strong);
}

.token-row input::placeholder {
  color: rgba(231, 236, 255, 0.3);
}

.primary {
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  color: var(--surface-0);
  padding: 10px 18px;
  border-radius: 12px;
  font-weight: 600;
  transition: transform 0.2s ease;
}

.primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.primary:not(:disabled):hover {
  transform: translateY(-1px);
}

.hint {
  font-size: 12px;
  color: var(--text-muted);
}

.error {
  color: #ff7676;
  font-size: 13px;
}

.error--inline {
  margin-top: -8px;
}

.success {
  color: #5de8b0;
  font-size: 13px;
}

.list-section {
  flex: 1;
}

.list-section--disabled {
  opacity: 0.5;
  pointer-events: none;
}

.search-input {
  background: var(--surface-3);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 10px 14px;
  color: var(--text-strong);
}

.search-input::placeholder {
  color: rgba(231, 236, 255, 0.3);
}

.repo-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
  overflow-y: auto;
  padding-right: 6px;
}
.file-tree {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
  overflow-y: auto;
  padding-right: 6px;
}

.repo-item {
  background: var(--surface-3);
  border: 1px solid transparent;
  border-radius: 14px;
  padding: 12px 14px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  text-align: left;
  transition: border-color 0.2s ease, background 0.2s ease;
}

.repo-item strong {
  color: var(--text-strong);
}

.repo-item p {
  font-size: 12px;
  color: var(--text-muted);
}

.repo-item:hover,
.repo-item:focus-visible {
  border-color: var(--accent-primary);
  background: rgba(79, 156, 255, 0.08);
}

.repo-item--active {
  border-color: var(--accent-secondary);
  background: rgba(124, 92, 255, 0.15);
}

.repo-meta {
  font-size: 11px;
  color: var(--text-muted);
  white-space: nowrap;
}

.empty {
  color: var(--text-muted);
  font-size: 12px;
  text-align: center;
  padding: 16px 0;
}

.status-banner {
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  background: rgba(9, 14, 23, 0.6);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 10px 14px;
  color: var(--text-soft);
}

.status-flags {
  display: flex;
  gap: 8px;
}

.tag {
  background: rgba(79, 156, 255, 0.15);
  color: var(--accent-primary);
  border-radius: 999px;
  padding: 4px 10px;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.editor {
  background: rgba(9, 14, 23, 0.82);
}

.editor-topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  background: rgba(9, 14, 23, 0.6);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 10px 14px;
  color: var(--text-soft);
}

.tab-bar {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 8px;
  overflow-x: auto;
  padding-bottom: 2px;
}

.tab {
  appearance: none;
  border: 1px solid transparent;
  background: rgba(13, 20, 32, 0.65);
  color: var(--text-soft);
  display: inline-flex;
  align-items: center;
  gap: 8px;
  border-radius: 10px;
  padding: 6px 12px;
  font-size: 12px;
  transition: background 0.2s ease, border-color 0.2s ease, color 0.2s ease;
  white-space: nowrap;
  flex-shrink: 0;
}

.tab:hover,
.tab:focus-visible {
  border-color: var(--accent-primary);
  color: var(--text-strong);
}

.tab--active {
  background: rgba(79, 156, 255, 0.18);
  border-color: var(--accent-secondary);
  color: var(--text-strong);
  box-shadow: inset 0 0 0 1px rgba(124, 92, 255, 0.2);
}

.tab__title {
  max-width: 160px;
  overflow: hidden;
  text-overflow: ellipsis;
}

.tab__dirty {
  color: var(--accent-secondary);
  font-size: 16px;
  line-height: 1;
  margin-top: -2px;
}

.tab__status {
  color: var(--accent-primary);
  font-size: 14px;
}

.tab__close {
  appearance: none;
  border: none;
  background: transparent;
  color: var(--text-muted);
  display: flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  border-radius: 6px;
  transition: background 0.2s ease, color 0.2s ease;
}

.tab__close:hover,
.tab__close:focus-visible {
  background: rgba(79, 156, 255, 0.15);
  color: var(--accent-primary);
}

.tab-empty {
  font-size: 12px;
  color: var(--text-muted);
  margin-left: 8px;
}

.editor-topbar__meta {
  display: flex;
  align-items: center;
  gap: 12px;
  justify-content: flex-end;
  min-width: 220px;
}

.editor-breadcrumb {
  font-size: 12px;
  max-width: 70%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.editor-surface {
  position: relative;
  flex: 1;
  background: var(--surface-2);
  border: 1px solid var(--color-border);
  border-radius: 18px;
  margin-top: 12px;
  overflow: hidden;
  display: flex;
}

.editor-placeholder {
  width: 100%;
  display: grid;
  place-items: center;
  color: var(--text-muted);
  border: 2px dashed rgba(126, 142, 178, 0.18);
  border-radius: 14px;
  margin: 18px;
}

.editor-canvas {
  flex: 1;
  position: relative;
  display: flex;
}

.editor-canvas :deep(.code-editor) {
  flex: 1;
}

.editor-loading {
  position: absolute;
  top: 16px;
  right: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 12px;
  border-radius: 999px;
  background: rgba(6, 10, 18, 0.85);
  border: 1px solid rgba(79, 156, 255, 0.25);
  color: var(--text-muted);
  font-size: 12px;
}

.tab__status,
.editor-loading .codicon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.webview {
  padding: 0;
}

.webview iframe {
  width: 100%;
  height: 100%;
  border: none;
  background: var(--surface-3);
}

.resize-handle {
  width: 6px;
  cursor: col-resize;
  background: rgba(10, 16, 26, 0.6);
  border-right: 1px solid var(--color-border);
  border-left: 1px solid var(--color-border);
  transition: background 0.2s ease;
}

.resize-handle:hover,
.resize-handle:active {
  background: rgba(79, 156, 255, 0.25);
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

@media (max-width: 1440px) {
  .workspace {
    overflow-x: auto;
  }

  .panel {
    min-width: 260px;
  }
}

@media (max-width: 1200px) {
  .toolbar {
    padding: 20px 10px;
  }

  .toolbar__button {
    width: 36px;
    height: 36px;
  }
}

@media (max-width: 960px) {
  .webview {
    display: none;
  }
}
</style>
