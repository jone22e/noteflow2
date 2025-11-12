<script setup>
import { computed, onMounted, ref } from 'vue'

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
  { label: 'Explorer', icon: '☰' },
  { label: 'Buscar', icon: '🔍' },
  { label: 'Commit', icon: '⎘' },
  { label: 'Deploy', icon: '⏫' },
  { label: 'Config', icon: '⚙️' },
]

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
</script>

<template>
  <div class="codex-shell">
    <aside class="toolbar">
      <div class="toolbar__brand">⌘</div>
      <div class="toolbar__buttons">
        <button
          v-for="button in actionButtons"
          :key="button.label"
          class="toolbar__button"
          type="button"
        >
          <span aria-hidden="true">{{ button.icon }}</span>
          <span>{{ button.label }}</span>
        </button>
      </div>
    </aside>

    <section class="repositories">
      <header class="panel-header">
        <div>
          <h1>codex web</h1>
          <p class="panel-subtitle">Conecte, selecione e navegue pelos arquivos do GitHub.</p>
        </div>
      </header>

      <div class="connection-card">
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

      <div class="list-section" :class="{ 'list-section--disabled': connectionState !== 'connected' }">
        <div class="list-header">
          <h2>Repositórios</h2>
          <span v-if="isLoadingRepos" class="tag">carregando…</span>
        </div>
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

    <section class="files">
      <header class="panel-header">
        <div>
          <h2>Arquivos</h2>
          <p class="panel-subtitle">{{ statusMessage }}</p>
        </div>
        <div class="file-info" v-if="openFilePath">
          <span class="file-info__name">{{ openFilePath }}</span>
          <span v-if="isLoadingFileContent" class="tag">carregando…</span>
        </div>
      </header>

      <div class="list-section">
        <div class="list-header">
          <h3>Estrutura</h3>
          <span v-if="isLoadingFiles" class="tag">atualizando…</span>
        </div>
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

    <section class="editor">
      <header class="panel-header">
        <div>
          <h2>Editor</h2>
          <p class="panel-subtitle">Visualize o conteúdo dos arquivos selecionados.</p>
        </div>
      </header>
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
      </div>
      <p v-if="fileError" class="error error--inline">{{ fileError }}</p>
    </section>

    <aside class="minimap">
      <header class="panel-header">
        <h2>scroll editor</h2>
      </header>
      <div class="minimap-surface">
        <pre>{{ minimapContent }}</pre>
      </div>
    </aside>

    <aside class="context-panel">
      <header class="panel-header">
        <h2>codex web</h2>
        <p class="panel-subtitle">Insights rápidos sobre o seu projeto.</p>
      </header>
      <div class="context-cards">
        <article class="context-card">
          <h3>Estado</h3>
          <p>{{ selectedRepo ? `Repositório ${selectedRepo.full_name}` : 'Nenhum repositório selecionado' }}</p>
        </article>
        <article class="context-card">
          <h3>Arquivo ativo</h3>
          <p>{{ openFilePath || 'Selecione um arquivo para visualizar detalhes.' }}</p>
        </article>
        <article class="context-card">
          <h3>Atividade</h3>
          <p v-if="selectedRepo">Última atualização: {{ new Date(selectedRepo.updated_at).toLocaleString() }}</p>
          <p v-else>Conecte-se para acompanhar sua atividade.</p>
        </article>
        <article class="context-card">
          <h3>Dicas</h3>
          <ul>
            <li>Use a busca para localizar arquivos rapidamente.</li>
            <li>Tokens são armazenados somente no navegador.</li>
            <li>Arquivos binários podem não ser exibidos corretamente.</li>
          </ul>
        </article>
      </div>
    </aside>
  </div>
</template>

<style scoped>
.codex-shell {
  flex: 1;
  display: grid;
  grid-template-columns: 92px 320px 340px 1fr 180px 300px;
  grid-template-rows: 100%;
  background: linear-gradient(135deg, var(--surface-0), var(--surface-2));
  color: var(--text-soft);
}

.toolbar {
  background: rgba(6, 10, 18, 0.9);
  border-right: 1px solid var(--color-border);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 32px 18px;
  gap: 24px;
}

.toolbar__brand {
  width: 42px;
  height: 42px;
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
  gap: 16px;
}

.toolbar__button {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  padding: 6px 0;
  width: 100%;
  border-radius: 12px;
  background: transparent;
  transition: background 0.2s ease, color 0.2s ease;
}

.toolbar__button span:first-child {
  font-size: 18px;
}

.toolbar__button span:last-child {
  font-size: 12px;
  letter-spacing: 0.04em;
}

.toolbar__button:hover,
.toolbar__button:focus-visible {
  background: rgba(79, 156, 255, 0.12);
  color: var(--text-strong);
}

.repositories,
.files,
.editor,
.minimap,
.context-panel {
  padding: 28px 24px;
  border-right: 1px solid var(--color-border);
  background: rgba(14, 21, 34, 0.65);
  backdrop-filter: blur(22px);
}

.context-panel {
  border-right: none;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  margin-bottom: 20px;
}

.panel-header h1,
.panel-header h2,
.panel-header h3 {
  color: var(--text-strong);
  font-weight: 600;
  letter-spacing: 0.02em;
}

.panel-header h1 {
  font-size: 24px;
}

.panel-header h2 {
  font-size: 18px;
}

.panel-header h3 {
  font-size: 14px;
  text-transform: uppercase;
  letter-spacing: 0.16em;
}

.panel-subtitle {
  font-size: 13px;
  color: var(--text-muted);
  margin-top: 4px;
}

.connection-card {
  background: var(--surface-2);
  border: 1px solid var(--color-border);
  border-radius: 16px;
  padding: 20px;
  box-shadow: var(--shadow-elevated);
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.connection-card label {
  font-size: 13px;
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
  margin-top: 8px;
}

.success {
  color: #5de8b0;
  font-size: 13px;
}

.list-section {
  margin-top: 28px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.list-section--disabled {
  opacity: 0.4;
  pointer-events: none;
}

.list-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
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
  max-height: calc(100vh - 320px);
  overflow-y: auto;
  padding-right: 4px;
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

.file-info {
  display: flex;
  align-items: center;
  gap: 10px;
}

.file-info__name {
  font-size: 12px;
  color: var(--text-muted);
  max-width: 260px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
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

.editor-surface {
  flex: 1;
  background: var(--surface-2);
  border: 1px solid var(--color-border);
  border-radius: 18px;
  padding: 16px;
  min-height: calc(100vh - 200px);
  display: flex;
  flex-direction: column;
}

.editor-placeholder {
  flex: 1;
  display: grid;
  place-items: center;
  color: var(--text-muted);
  border: 2px dashed rgba(126, 142, 178, 0.18);
  border-radius: 14px;
  font-size: 14px;
}

.code-editor {
  flex: 1;
  background: var(--surface-3);
  color: var(--text-strong);
  border: 1px solid rgba(126, 142, 178, 0.18);
  border-radius: 14px;
  padding: 16px;
  font-family: inherit;
  font-size: 13px;
  line-height: 1.5;
  resize: none;
  min-height: 100%;
  white-space: pre;
  overflow: auto;
}

.minimap {
  background: rgba(10, 16, 26, 0.82);
}

.minimap-surface {
  background: var(--surface-3);
  border: 1px solid var(--color-border);
  border-radius: 16px;
  padding: 12px;
  height: calc(100vh - 160px);
  overflow: hidden;
}

.minimap pre {
  font-family: inherit;
  font-size: 8px;
  line-height: 0.7;
  white-space: pre;
  color: rgba(231, 236, 255, 0.38);
}

.context-panel {
  background: rgba(14, 20, 32, 0.74);
}

.context-cards {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.context-card {
  background: var(--surface-2);
  border: 1px solid var(--color-border);
  border-radius: 16px;
  padding: 18px;
  box-shadow: var(--shadow-elevated);
}

.context-card h3 {
  color: var(--text-strong);
  font-size: 14px;
  margin-bottom: 8px;
}

.context-card p,
.context-card li {
  color: var(--text-soft);
  font-size: 13px;
  line-height: 1.4;
}

.context-card ul {
  list-style: disc;
  margin-left: 18px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

@media (max-width: 1680px) {
  .codex-shell {
    grid-template-columns: 80px 280px 320px 1fr 160px 260px;
  }
}

@media (max-width: 1440px) {
  .codex-shell {
    grid-template-columns: 72px 260px 300px minmax(360px, 1fr);
    grid-template-areas:
      'toolbar repos files editor'
      'toolbar repos files editor';
  }

  .minimap,
  .context-panel {
    display: none;
  }
}

@media (max-width: 1200px) {
  .codex-shell {
    grid-template-columns: 64px 260px 1fr;
  }

  .files {
    display: none;
  }
}

@media (max-width: 900px) {
  .codex-shell {
    grid-template-columns: 56px 1fr;
  }

  .repositories {
    display: none;
  }
}
</style>
