<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue'

const token = ref('')
const connectionState = ref('disconnected')
const connectionError = ref('')
const userProfile = ref(null)
const repos = ref([])
const repoFilter = ref('')
const selectedRepo = ref(null)
const files = ref([])
const fileSearch = ref('')
const statusMessage = ref('Conecte-se ao GitHub para começar.')
const isLoadingRepos = ref(false)
const isLoadingFiles = ref(false)
const isLoadingFileContent = ref(false)
const fileError = ref('')
const openFilePath = ref('')
const openFileContent = ref('')
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

const resizing = ref(null)
const minPanelWidth = 220

const filteredRepos = computed(() => {
  if (!repoFilter.value.trim()) {
    return repos.value
  }
  const term = repoFilter.value.trim().toLowerCase()
  return repos.value.filter((repo) => repo.name.toLowerCase().includes(term) || repo.full_name.toLowerCase().includes(term))
})

const filteredFiles = computed(() => {
  if (!fileSearch.value.trim()) {
    return files.value
  }
  const term = fileSearch.value.trim().toLowerCase()
  return files.value.filter((entry) => entry.path.toLowerCase().includes(term))
})

const minimapContent = computed(() => {
  if (!openFileContent.value) {
    return ''
  }
  return openFileContent.value
    .split('\n')
    .map((line) => line.slice(0, 80))
    .join('\n')
})

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
}

async function loadRepositoryFiles() {
  if (!selectedRepo.value) {
    return
  }

  isLoadingFiles.value = true
  files.value = []
  fileSearch.value = ''
  openFilePath.value = ''
  openFileContent.value = ''
  fileError.value = ''

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
    files.value = blobs
    statusMessage.value = `${blobs.length} arquivos carregados de ${selectedRepo.value.name}.`
    if (!blobs.length) {
      statusMessage.value = 'Nenhum arquivo encontrado neste repositório.'
    }
  } catch (error) {
    files.value = []
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

  isLoadingFileContent.value = true
  fileError.value = ''
  openFilePath.value = entry.path
  openFileContent.value = ''

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
    if (fileData.encoding === 'base64') {
      const decoded = atob((fileData.content || '').replace(/\n/g, ''))
      openFileContent.value = decoded
    } else if (typeof fileData.content === 'string') {
      openFileContent.value = fileData.content
    } else {
      openFileContent.value = ''
    }
    statusMessage.value = `${entry.path} carregado com sucesso.`
  } catch (error) {
    fileError.value = error instanceof Error ? error.message : 'Erro desconhecido ao carregar o arquivo.'
    statusMessage.value = fileError.value
  } finally {
    isLoadingFileContent.value = false
  }
}

function formatFileSize(size) {
  if (size == null) return ''
  if (size < 1024) return `${size} B`
  if (size < 1024 * 1024) return `${(size / 1024).toFixed(1)} KB`
  return `${(size / (1024 * 1024)).toFixed(1)} MB`
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
      <section class="panel repositories" :style="{ width: `${panelSizes.repositories}px` }">
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
          <div class="file-list" role="list">
            <button
              v-for="entry in filteredFiles"
              :key="entry.sha"
              class="file-item"
              role="listitem"
              type="button"
              :class="{ 'file-item--active': entry.path === openFilePath }"
              @click="openFile(entry)"
            >
              <span class="file-path">{{ entry.path }}</span>
              <span class="file-size">{{ formatFileSize(entry.size) }}</span>
            </button>
            <p v-if="!filteredFiles.length && selectedRepo && !isLoadingFiles" class="empty">Nenhum arquivo encontrado.</p>
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
          <span class="editor-breadcrumb">{{ openFilePath || 'Nenhum arquivo aberto' }}</span>
          <span v-if="isLoadingFileContent" class="tag">carregando…</span>
        </div>
        <div class="editor-surface">
          <div v-if="!openFilePath" class="editor-placeholder">
            <p>Selecione um arquivo para exibir o conteúdo aqui.</p>
          </div>
          <textarea
            v-else
            v-model="openFileContent"
            class="code-editor"
            spellcheck="false"
          ></textarea>
          <div v-if="openFilePath" class="editor-minimap">
            <pre>{{ minimapContent }}</pre>
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

.repo-list,
.file-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
  overflow-y: auto;
  padding-right: 6px;
}

.repo-item,
.file-item {
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

.repo-item strong,
.file-item strong {
  color: var(--text-strong);
}

.repo-item p,
.file-item span {
  font-size: 12px;
  color: var(--text-muted);
}

.repo-item:hover,
.repo-item:focus-visible,
.file-item:hover,
.file-item:focus-visible {
  border-color: var(--accent-primary);
  background: rgba(79, 156, 255, 0.08);
}

.repo-item--active,
.file-item--active {
  border-color: var(--accent-secondary);
  background: rgba(124, 92, 255, 0.15);
}

.repo-meta,
.file-size {
  font-size: 11px;
  color: var(--text-muted);
  white-space: nowrap;
}

.file-path {
  color: var(--text-soft);
  font-size: 13px;
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
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

.code-editor {
  flex: 1;
  background: var(--surface-3);
  color: var(--text-strong);
  border: none;
  padding: 24px;
  font-family: inherit;
  font-size: 13px;
  line-height: 1.5;
  resize: none;
  white-space: pre;
  overflow: auto;
  padding-right: 140px;
}

.editor-minimap {
  position: absolute;
  top: 16px;
  right: 16px;
  bottom: 16px;
  width: 112px;
  background: rgba(5, 8, 15, 0.8);
  border: 1px solid rgba(79, 156, 255, 0.2);
  border-radius: 10px;
  overflow: hidden;
  pointer-events: none;
}

.editor-minimap pre {
  font-family: inherit;
  font-size: 8px;
  line-height: 0.7;
  padding: 12px 8px;
  color: rgba(231, 236, 255, 0.4);
  white-space: pre;
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
