<script setup>
import { reactive, ref, computed, onMounted } from 'vue'
import { AutoDetect, SetExePath, GetStatus, PatchFile, BackupFile, RestoreFile, CharaAttach, CharaDetach, CharaGetAll, CharaSetOne, CharaSetAll } from '../../wailsjs/go/main/App'
import { WindowMinimise, Quit } from '../../wailsjs/runtime/runtime'
import SigilGenerator from './SigilGenerator.vue'

const state = reactive({
  exePath: '',
  fileExists: false,
  fileSize: 0,
  backupExists: false,
  backupSize: 0,
  patches: [],
})

const activeTab = ref('patch')
const manualPath = ref('')
const patchValues = reactive({})
const isLoaded = ref(false)
const isDetecting = ref(false)
const patchingID = ref('')
const forceBackup = ref(false)
const saveStatus = ref('')
const statusType = ref('')

onMounted(() => {
  isDetecting.value = true
  AutoDetect()
    .then((path) => {
      isDetecting.value = false
      if (path) {
        state.exePath = path
        manualPath.value = path
        return loadFile(path)
      }
    })
    .catch(() => { isDetecting.value = false })
})

function loadFile(path) {
  return GetStatus(path).then((info) => {
    Object.assign(state, info)
    ;(info.patches || []).forEach(p => {
      if (p.state === 'patched') patchValues[p.id] = String(p.currentValue)
      else if (!patchValues[p.id]) patchValues[p.id] = ''
    })
    isLoaded.value = true
    showStatus('File loaded successfully', 'success')
  })
}

function applyManualPath() {
  const p = manualPath.value.trim()
  if (!p) { showStatus('Please enter a file path', 'error'); return }
  SetExePath(p)
    .then((info) => {
      Object.assign(state, info)
      ;(info.patches || []).forEach(p => {
        if (p.state === 'patched') patchValues[p.id] = String(p.currentValue)
        else if (!patchValues[p.id]) patchValues[p.id] = ''
      })
      isLoaded.value = true
      showStatus('File loaded successfully', 'success')
    })
    .catch((err) => showStatus(String(err), 'error'))
}

function refreshStatus() {
  return GetStatus(state.exePath).then((info) => {
    Object.assign(state, info)
    ;(info.patches || []).forEach(p => {
      if (p.state === 'patched') patchValues[p.id] = String(p.currentValue)
    })
  })
}

function applyPatch(patchID) {
  const v = parseInt(patchValues[patchID])
  if (isNaN(v) || v < 0) { showStatus('Please enter a valid value', 'error'); return }
  patchingID.value = patchID
  PatchFile(patchID, v)
    .then(() => refreshStatus())
    .then(() => { patchingID.value = ''; showStatus('Patch applied successfully', 'success') })
    .catch((err) => { patchingID.value = ''; showStatus('Patch failed: ' + (err || 'Unknown error'), 'error') })
}

function backup() {
  BackupFile(forceBackup.value)
    .then(() => refreshStatus())
    .then(() => showStatus('Backup created successfully', 'success'))
    .catch((err) => showStatus('Backup failed: ' + (err || 'Unknown error'), 'error'))
}

function restore() {
  RestoreFile()
    .then(() => refreshStatus())
    .then(() => showStatus('File restored successfully', 'success'))
    .catch((err) => showStatus('Restore failed: ' + (err || 'Unknown error'), 'error'))
}

const CARD_COLORS = {
  mission: { bg: 'linear-gradient(135deg, rgba(124,58,237,0.25) 0%, rgba(249,212,35,0.1) 100%)', shadow: 'rgba(124,58,237,0.18)' },
  likes:   { bg: 'linear-gradient(135deg, rgba(245,158,11,0.25) 0%, rgba(249,212,35,0.1) 100%)', shadow: 'rgba(245,158,11,0.18)' },
}

const CARD_HINTS = {
  mission: 'Does not affect save file',
  likes: 'Takes effect after receiving a like (affects save)',
}

function showStatus(msg, type) {
  saveStatus.value = msg; statusType.value = type
  setTimeout(() => { saveStatus.value = '' }, 3000)
}

// ── Character Usage Count ──
const charaEditEnabled = true // ENABLED: show edit controls
const charaConnected = ref(false)
const charaInfo = reactive({ pid: 0, moduleBase: 0, manager: 0 })
const charaList = ref([])
const charaEditValues = reactive({})
const charaBatchValue = ref('')
const charaLoading = ref(false)
const charaSortDesc = ref(false)
const charaSorted = computed(() => {
  if (!charaSortDesc.value) return charaList.value
  return [...charaList.value].sort((a, b) => b.count - a.count)
})

function charaConnect() {
  charaLoading.value = true
  CharaAttach()
    .then((info) => {
      charaConnected.value = true
      Object.assign(charaInfo, info)
      return charaRefresh()
    })
    .catch((err) => showStatus(String(err), 'error'))
    .finally(() => { charaLoading.value = false })
}

function charaDisconnect() {
  CharaDetach()
    .then(() => {
      charaConnected.value = false
      charaList.value = []
      Object.assign(charaInfo, { pid: 0, moduleBase: 0, manager: 0 })
    })
    .catch((err) => showStatus(String(err), 'error'))
}

function charaRefresh() {
  return CharaGetAll()
    .then((list) => {
      charaList.value = list || []
      list.forEach(c => {
        if (charaEditValues[c.index] === undefined) charaEditValues[c.index] = String(c.count)
      })
    })
    .catch((err) => showStatus(String(err), 'error'))
}

function charaSetSingle(index) {
  const v = parseInt(charaEditValues[index])
  if (isNaN(v) || v < 0) { showStatus('Please enter a valid value', 'error'); return }
  CharaSetOne(index, v)
    .then(() => charaRefresh())
    .then(() => showStatus('Set successfully', 'success'))
    .catch((err) => showStatus(String(err), 'error'))
}

function charaSetBatch() {
  const v = parseInt(charaBatchValue.value)
  if (isNaN(v) || v < 0) { showStatus('Please enter a valid value', 'error'); return }
  CharaSetAll(v)
    .then((n) => charaRefresh().then(() => showStatus(`Set ${n} characters`, 'success')))
    .catch((err) => showStatus(String(err), 'error'))
}
</script>

<template>
  <div class="app-window">
    <div class="titlebar" style="--wails-draggable:drag">
      <div class="titlebar-left">
        <span class="titlebar-title">GBFR Save Editor</span>
        <transition name="fade">
          <span v-if="saveStatus" class="titlebar-status" :class="statusType">
            {{ statusType === 'success' ? '●' : '✕' }} {{ saveStatus }}
          </span>
        </transition>
      </div>
      <div class="titlebar-controls" style="--wails-draggable:no-drag">
        <button class="win-btn minimize" @click="WindowMinimise" title="Minimize">
          <svg width="10" height="1" viewBox="0 0 10 1"><rect width="10" height="1.5" rx="0.75" fill="currentColor"/></svg>
        </button>
        <button class="win-btn close" @click="Quit" title="Close">
          <svg width="10" height="10" viewBox="0 0 10 10"><line x1="1" y1="1" x2="9" y2="9" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/><line x1="9" y1="1" x2="1" y2="9" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>
        </button>
      </div>
    </div>

    <div class="tab-bar" style="--wails-draggable:no-drag">
      <button class="tab-btn" :class="{ active: activeTab === 'patch' }" @click="activeTab = 'patch'">
        Patch Tool
      </button>
      <button class="tab-btn" :class="{ active: activeTab === 'sigil' }" @click="activeTab = 'sigil'">
        Sigil Generator
      </button>
    </div>

    <main v-if="activeTab === 'patch'" class="container" style="--wails-draggable:no-drag">
      <div class="path-section">
        <div class="path-label">
          <span v-if="isDetecting">Scanning Steam installation path...</span>
          <span v-else-if="isLoaded" class="path-found">Game file located</span>
          <span v-else>Enter granblue_fantasy_relink.exe path</span>
        </div>
        <div class="path-input-row">
          <input v-model="manualPath" type="text" class="path-input"
            placeholder="Paste full path to exe file..." @keyup.enter="applyManualPath" />
          <button class="btn-path-confirm" @click="applyManualPath" :disabled="!manualPath.trim()">Load</button>
        </div>
      </div>

      <transition name="slide-up">
        <div v-if="isLoaded" class="data-panel">
          <div v-if="state.exePath" class="path-bar">
            <span class="path-text" :title="state.exePath">{{ state.exePath }}</span>
            <span class="file-size">{{ (state.fileSize / 1024 / 1024).toFixed(1) }} MB</span>
          </div>

          <div v-for="p in state.patches" :key="p.id" class="data-card"
            :style="{ background: (CARD_COLORS[p.id]||CARD_COLORS.mission).bg, boxShadow: '0 4px 20px '+(CARD_COLORS[p.id]||CARD_COLORS.mission).shadow }">
            <div class="card-header">
              <span class="card-label">{{ p.name }}</span>
              <span v-if="CARD_HINTS[p.id]" class="card-hint">{{ CARD_HINTS[p.id] }}</span>
              <span v-if="p.state==='original'" class="state-badge original">Not Patched</span>
              <span v-else-if="p.state==='patched'" class="state-badge patched">Patched</span>
              <span v-else class="state-badge unknown">Unknown</span>
            </div>
            <div v-if="p.state==='patched'" class="card-detail">
              Current value: {{ p.currentValue }} (0x{{ p.currentValue.toString(16).toUpperCase() }})
            </div>
            <div class="card-edit-row">
              <input v-model="patchValues[p.id]" type="number" min="0" class="edit-input" placeholder="Enter value" />
              <button class="btn-patch" @click="applyPatch(p.id)"
                :disabled="patchingID === p.id || !patchValues[p.id] || isNaN(parseInt(patchValues[p.id]))">
                {{ patchingID === p.id ? 'Writing...' : 'Apply' }}
              </button>
            </div>
          </div>

          <div class="backup-section">
            <div class="backup-row">
              <button class="btn-secondary" @click="backup">Backup</button>
              <button class="btn-secondary restore" @click="restore" :disabled="!state.backupExists">Restore</button>
            </div>
            <label class="force-label">
              <input type="checkbox" v-model="forceBackup" />
              <span>Force overwrite existing backup</span>
            </label>
            <div v-if="state.backupExists" class="backup-info">Backup: {{ (state.backupSize / 1024 / 1024).toFixed(1) }} MB (exe only)</div>
          </div>
        </div>
      </transition>

      <!-- Character Usage Count -->
      <div class="chara-section">
        <div class="chara-header">
          <span class="chara-title">Character Usage Count</span>
          <span class="chara-hint">Game must be running · Play one quest with the character to save changes<br/>
            (Affects save file)</span>
        </div>
        <div class="chara-connect-row">
          <button v-if="!charaConnected" class="btn-chara-connect" @click="charaConnect" :disabled="charaLoading">
            {{ charaLoading ? 'Connecting...' : 'Connect to Game' }}
          </button>
          <button v-else class="btn-chara-disconnect" @click="charaDisconnect">Disconnect</button>
          <span v-if="charaConnected" class="chara-pid">PID: {{ charaInfo.pid }}</span>
        </div>

        <template v-if="charaConnected && charaList.length">
          <div v-if="charaEditEnabled" class="chara-batch-row">
            <input v-model="charaBatchValue" type="number" min="0" class="chara-batch-input" placeholder="Value" />
            <button class="btn-chara-batch" @click="charaSetBatch" :disabled="!charaBatchValue || isNaN(parseInt(charaBatchValue))">Set All To</button>
            <button class="btn-chara-refresh" @click="charaRefresh">Refresh</button>
          </div>
          <div v-else class="chara-batch-row">
            <button class="btn-chara-refresh" @click="charaRefresh">Refresh</button>
            <button class="btn-chara-sort" @click="charaSortDesc = !charaSortDesc">
              {{ charaSortDesc ? 'Default Order' : 'Sort by Count' }}
            </button>
          </div>
          <div class="chara-table">
            <div class="chara-row chara-row-header">
              <span class="chara-col-idx">#</span>
              <span class="chara-col-name">Character</span>
              <span class="chara-col-count">Count</span>
              <span v-if="charaEditEnabled" class="chara-col-edit">Edit</span>
            </div>
            <div v-for="c in charaSorted" :key="c.index" class="chara-row">
              <span class="chara-col-idx">{{ c.index }}</span>
              <span class="chara-col-name">{{ c.name }}</span>
              <span class="chara-col-count">{{ c.count }}</span>
              <div v-if="charaEditEnabled" class="chara-col-edit">
                <input v-model="charaEditValues[c.index]" type="number" min="0" class="chara-edit-input" @keyup.enter="charaSetSingle(c.index)" />
                <button class="btn-chara-set" @click="charaSetSingle(c.index)">Set</button>
              </div>
            </div>
          </div>
        </template>
        <div v-else-if="charaConnected" class="chara-empty">No character data found. Make sure you are in-game with a save loaded.</div>
      </div>

      <transition name="fade">
        <div v-if="!isLoaded && !isDetecting" class="placeholder">
          <p>Game not detected automatically. Please enter the exe path manually.</p>
          <p class="placeholder-tip">It is recommended to backup the original file before patching.</p>
        </div>
      </transition>
      <div class="footer-hint"><a href="https://github.com/BitterG/GBFR-PE-Patch-Tool" target="_blank" class="footer-link">github.com/BitterG/GBFR-PE-Patch-Tool</a></div>
    </main>

    <main v-else class="container" style="--wails-draggable:no-drag">
      <SigilGenerator @status="showStatus" />
    </main>
  </div>
</template>

<style scoped>
.app-window { display:flex; flex-direction:column; height:100vh; overflow:hidden; background-color:rgba(27,38,54,1); border-radius:10px; box-shadow:0 8px 40px rgba(0,0,0,0.6); }
.tab-bar { display:flex; gap:2px; padding:4px 14px; background:rgba(18,26,38,0.95); border-bottom:1px solid rgba(255,255,255,0.06); flex-shrink:0; }
.tab-btn { padding:4px 14px; border-radius:6px; border:none; background:transparent; color:rgba(255,255,255,0.35); font-size:0.76rem; font-weight:600; cursor:pointer; transition:background 0.15s, color 0.15s; }
.tab-btn:hover { color:rgba(255,255,255,0.6); background:rgba(255,255,255,0.05); }
.tab-btn.active { color:#67e8f9; background:rgba(103,232,249,0.12); }
.titlebar { display:flex; align-items:center; justify-content:space-between; height:38px; padding:0 6px 0 14px; background:rgba(18,26,38,0.95); border-bottom:1px solid rgba(255,255,255,0.06); flex-shrink:0; user-select:none; }
.titlebar-left { display:flex; align-items:center; gap:8px; }
.titlebar-title { font-size:0.8rem; font-weight:600; color:rgba(255,255,255,0.55); letter-spacing:0.5px; }
.titlebar-controls { display:flex; align-items:center; gap:2px; }
.win-btn { width:32px; height:28px; border:none; border-radius:6px; background:transparent; color:rgba(255,255,255,0.45); cursor:pointer; display:flex; align-items:center; justify-content:center; transition:background 0.15s,color 0.15s; }
.win-btn.minimize:hover { background:rgba(255,255,255,0.1); color:rgba(255,255,255,0.9); }
.win-btn.close:hover { background:rgba(239,68,68,0.8); color:#fff; }
.titlebar-status { font-size:0.68rem; font-weight:600; padding:2px 8px; border-radius:20px; white-space:nowrap; }
.titlebar-status.success { color:#4ade80; background:rgba(34,197,94,0.15); }
.titlebar-status.error { color:#f87171; background:rgba(239,68,68,0.15); }
.container { flex:1; overflow-y:auto; max-width:720px; width:100%; margin:0 auto; padding:20px 20px 40px; box-sizing:border-box; display:flex; flex-direction:column; align-items:center; gap:14px; scrollbar-width:none; }
.container::-webkit-scrollbar { display:none; }
.path-section { width:100%; }
.path-label { font-size:0.78rem; color:rgba(255,255,255,0.4); margin-bottom:8px; }
.path-found { color:#4ade80; }
.path-input-row { display:flex; gap:8px; }
.path-input { flex:1; padding:10px 14px; border-radius:10px; border:1px solid rgba(255,255,255,0.15); background:rgba(255,255,255,0.07); color:#fff; font-size:0.85rem; font-family:'Courier New',monospace; outline:none; transition:border-color 0.2s; }
.path-input::placeholder { color:rgba(255,255,255,0.22); }
.path-input:focus { border-color:rgba(103,232,249,0.5); background:rgba(255,255,255,0.1); }
.btn-path-confirm { padding:10px 18px; border-radius:10px; border:1px solid rgba(103,232,249,0.3); background:rgba(103,232,249,0.1); color:#67e8f9; font-size:0.85rem; font-weight:600; cursor:pointer; transition:background 0.2s,transform 0.15s; }
.btn-path-confirm:not(:disabled):hover { background:rgba(103,232,249,0.2); transform:scale(1.02); }
.btn-path-confirm:disabled { opacity:0.4; cursor:not-allowed; }
.path-bar { width:100%; box-sizing:border-box; padding:8px 14px; border-radius:10px; background:rgba(255,255,255,0.05); border:1px solid rgba(255,255,255,0.08); display:flex; align-items:center; justify-content:space-between; gap:8px; }
.path-text { font-size:0.72rem; color:rgba(255,255,255,0.4); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; font-family:'Courier New',monospace; flex:1; }
.file-size { font-size:0.68rem; color:rgba(255,255,255,0.3); flex-shrink:0; }
.data-panel { width:100%; display:flex; flex-direction:column; gap:12px; }
.data-card { border-radius:16px; padding:16px 18px; border:1px solid rgba(255,255,255,0.08); transition:transform 0.2s ease; display:flex; flex-direction:column; gap:8px; }
.data-card:hover { transform:translateY(-2px); }
.card-header { display:flex; align-items:center; justify-content:space-between; }
.card-label { font-size:0.88rem; font-weight:600; color:rgba(255,255,255,0.65); letter-spacing:1px; }
.card-hint { font-size:0.68rem; color:rgba(255,255,255,0.25); margin-left:4px; }
.card-detail { font-size:0.75rem; color:rgba(255,255,255,0.45); font-family:'Courier New',monospace; }
.state-badge { font-size:0.72rem; font-weight:700; padding:3px 10px; border-radius:20px; letter-spacing:0.5px; }
.state-badge.original { color:#67e8f9; background:rgba(103,232,249,0.15); }
.state-badge.patched { color:#4ade80; background:rgba(34,197,94,0.15); }
.state-badge.unknown { color:#fbbf24; background:rgba(251,191,36,0.15); }
.card-edit-row { display:flex; gap:8px; align-items:center; }
.edit-input { flex:1; padding:8px 14px; border-radius:8px; border:1px solid rgba(255,255,255,0.15); background:rgba(255,255,255,0.07); color:#fff; font-size:0.95rem; font-family:inherit; outline:none; transition:border-color 0.2s; }
.edit-input::placeholder { color:rgba(255,255,255,0.22); }
.edit-input:focus { border-color:rgba(255,255,255,0.4); background:rgba(255,255,255,0.12); }
.edit-input::-webkit-outer-spin-button, .edit-input::-webkit-inner-spin-button { -webkit-appearance:none; margin:0; }
.btn-patch { padding:8px 20px; border-radius:8px; border:1px solid rgba(165,180,252,0.3); background:rgba(165,180,252,0.1); color:#a5b4fc; font-size:0.85rem; font-weight:600; cursor:pointer; transition:background 0.2s,transform 0.15s; white-space:nowrap; }
.btn-patch:not(:disabled):hover { background:rgba(165,180,252,0.2); transform:scale(1.02); }
.btn-patch:disabled { opacity:0.4; cursor:not-allowed; }
.backup-section { padding:14px 18px; border-radius:16px; background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.06); display:flex; flex-direction:column; gap:10px; }
.backup-row { display:flex; gap:10px; }
.btn-secondary { flex:1; padding:10px 0; border-radius:10px; border:1px solid rgba(255,255,255,0.12); background:rgba(40,48,64,0.8); color:rgba(255,255,255,0.6); font-size:0.85rem; font-weight:600; cursor:pointer; display:flex; align-items:center; justify-content:center; gap:6px; transition:color 0.2s,border-color 0.2s,transform 0.15s; }
.btn-secondary:not(:disabled):hover { color:#67e8f9; border-color:rgba(103,232,249,0.35); transform:scale(1.02); }
.btn-secondary.restore:not(:disabled):hover { color:#fbbf24; border-color:rgba(251,191,36,0.35); }
.btn-secondary:disabled { opacity:0.4; cursor:not-allowed; }
.force-label { display:flex; align-items:center; gap:6px; font-size:0.75rem; color:rgba(255,255,255,0.35); cursor:pointer; }
.force-label input[type="checkbox"] { accent-color:#67e8f9; }
.backup-info { font-size:0.72rem; color:rgba(255,255,255,0.3); font-family:'Courier New',monospace; }
.placeholder { margin-top:40px; color:rgba(255,255,255,0.25); text-align:center; font-size:0.88rem; line-height:1.8; }
.placeholder-tip { font-size:0.78rem; color:rgba(255,255,255,0.18); margin-top:8px; }
.footer-hint { width:100%; text-align:center; font-size:0.72rem; color:rgba(255,255,255,0.2); margin-top:auto; padding-top:16px; }
.footer-link { color:rgba(255,255,255,0.25); text-decoration:none; transition:color 0.2s; }
.footer-link:hover { color:rgba(103,232,249,0.6); }
.fade-enter-active, .fade-leave-active { transition:opacity 0.3s ease; }
.fade-enter-from, .fade-leave-to { opacity:0; }
.slide-up-enter-active { transition:all 0.4s cubic-bezier(0.25,0.46,0.45,0.94); }
.slide-up-enter-from { opacity:0; transform:translateY(24px); }
.chara-section { width:100%; border-radius:16px; padding:16px 18px; background:linear-gradient(135deg, rgba(56,189,248,0.12) 0%, rgba(103,232,249,0.06) 100%); border:1px solid rgba(103,232,249,0.15); display:flex; flex-direction:column; gap:10px; }
.chara-header { display:flex; align-items:center; justify-content:space-between; }
.chara-title { font-size:0.88rem; font-weight:600; color:rgba(255,255,255,0.65); letter-spacing:1px; }
.chara-hint { font-size:0.68rem; color:rgba(255,255,255,0.25); }
.chara-connect-row { display:flex; align-items:center; gap:10px; }
.btn-chara-connect { padding:8px 18px; border-radius:8px; border:1px solid rgba(34,197,94,0.4); background:rgba(34,197,94,0.12); color:#4ade80; font-size:0.82rem; font-weight:600; cursor:pointer; transition:background 0.2s,transform 0.15s; }
.btn-chara-connect:not(:disabled):hover { background:rgba(34,197,94,0.22); transform:scale(1.02); }
.btn-chara-connect:disabled { opacity:0.5; cursor:not-allowed; }
.btn-chara-disconnect { padding:8px 18px; border-radius:8px; border:1px solid rgba(239,68,68,0.4); background:rgba(239,68,68,0.12); color:#f87171; font-size:0.82rem; font-weight:600; cursor:pointer; transition:background 0.2s; }
.btn-chara-disconnect:hover { background:rgba(239,68,68,0.22); }
.chara-pid { font-size:0.72rem; color:rgba(255,255,255,0.35); font-family:'Courier New',monospace; }
.chara-batch-row { display:flex; gap:8px; align-items:center; }
.chara-batch-input { width:80px; padding:6px 10px; border-radius:6px; border:1px solid rgba(255,255,255,0.15); background:rgba(255,255,255,0.07); color:#fff; font-size:0.82rem; outline:none; }
.chara-batch-input:focus { border-color:rgba(103,232,249,0.5); }
.chara-batch-input::-webkit-outer-spin-button, .chara-batch-input::-webkit-inner-spin-button { -webkit-appearance:none; margin:0; }
.btn-chara-batch { padding:6px 14px; border-radius:6px; border:1px solid rgba(165,180,252,0.3); background:rgba(165,180,252,0.1); color:#a5b4fc; font-size:0.78rem; font-weight:600; cursor:pointer; transition:background 0.2s; white-space:nowrap; }
.btn-chara-batch:not(:disabled):hover { background:rgba(165,180,252,0.2); }
.btn-chara-batch:disabled { opacity:0.4; cursor:not-allowed; }
.btn-chara-refresh { padding:6px 14px; border-radius:6px; border:1px solid rgba(255,255,255,0.12); background:rgba(255,255,255,0.05); color:rgba(255,255,255,0.5); font-size:0.78rem; font-weight:600; cursor:pointer; transition:background 0.2s; }
.btn-chara-refresh:hover { background:rgba(255,255,255,0.1); color:rgba(255,255,255,0.7); }
.btn-chara-sort { padding:6px 14px; border-radius:6px; border:1px solid rgba(255,255,255,0.12); background:rgba(255,255,255,0.05); color:rgba(255,255,255,0.5); font-size:0.78rem; font-weight:600; cursor:pointer; transition:background 0.2s; }
.btn-chara-sort:hover { background:rgba(255,255,255,0.1); color:rgba(255,255,255,0.7); }
.chara-table { display:flex; flex-direction:column; gap:1px; background:rgba(255,255,255,0.04); border-radius:10px; overflow:hidden; }
.chara-row { display:flex; align-items:center; padding:6px 10px; gap:6px; background:rgba(27,38,54,0.6); }
.chara-row-header { background:rgba(255,255,255,0.06); font-size:0.7rem; color:rgba(255,255,255,0.3); font-weight:600; padding:5px 10px; }
.chara-col-idx { width:24px; text-align:center; font-size:0.72rem; color:rgba(255,255,255,0.3); font-family:'Courier New',monospace; flex-shrink:0; }
.chara-col-name { flex:1; font-size:0.8rem; color:rgba(255,255,255,0.6); min-width:0; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
.chara-col-count { width:48px; text-align:right; font-size:0.8rem; color:#67e8f9; font-family:'Courier New',monospace; flex-shrink:0; }
.chara-col-edit { width:120px; display:flex; gap:4px; align-items:center; flex-shrink:0; }
.chara-edit-input { width:56px; padding:4px 6px; border-radius:4px; border:1px solid rgba(255,255,255,0.12); background:rgba(255,255,255,0.06); color:#fff; font-size:0.78rem; outline:none; text-align:center; }
.chara-edit-input:focus { border-color:rgba(103,232,249,0.4); }
.chara-edit-input::-webkit-outer-spin-button, .chara-edit-input::-webkit-inner-spin-button { -webkit-appearance:none; margin:0; }
.btn-chara-set { padding:4px 10px; border-radius:4px; border:1px solid rgba(165,180,252,0.25); background:rgba(165,180,252,0.08); color:#a5b4fc; font-size:0.72rem; font-weight:600; cursor:pointer; transition:background 0.15s; white-space:nowrap; }
.btn-chara-set:hover { background:rgba(165,180,252,0.18); }
.chara-empty { font-size:0.78rem; color:rgba(255,255,255,0.3); text-align:center; padding:12px 0; }
</style>
