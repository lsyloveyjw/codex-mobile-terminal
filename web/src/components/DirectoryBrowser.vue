<script setup>
import { ref, onMounted } from "vue";
import { request } from "../lib/api.js";

const props = defineProps({
  initialPath: { type: String, default: "" }
});

const emit = defineEmits(["select", "close"]);

const currentPath = ref(props.initialPath || "/");
const entries = ref([]);
const parentPath = ref(null);
const loading = ref(false);
const errorText = ref("");

async function loadDirectory(path) {
  loading.value = true;
  errorText.value = "";
  try {
    const payload = await request(`/api/fs?path=${encodeURIComponent(path)}`);
    currentPath.value = payload.path;
    parentPath.value = payload.parentPath;
    entries.value = payload.entries || [];
  } catch (err) {
    errorText.value = err?.message || String(err);
  } finally {
    loading.value = false;
  }
}

function navigateTo(path) {
  if (path) {
    loadDirectory(path);
  }
}

function selectCurrent() {
  emit("select", currentPath.value);
}

onMounted(() => {
  if (currentPath.value) {
    loadDirectory(currentPath.value);
  }
});
</script>

<template>
  <div class="dir-browser">
    <div class="dir-browser-header">
      <button type="button" class="dir-browser-back" @click="emit('close')">取消</button>
      <h3 class="dir-browser-title">选择目录</h3>
      <button type="button" class="dir-browser-confirm" @click="selectCurrent">选择此处</button>
    </div>

    <div class="dir-browser-path">
      <span class="dir-browser-path-text">{{ currentPath }}</span>
    </div>

    <div v-if="loading" class="dir-browser-loading">加载中…</div>
    <div v-else-if="errorText" class="dir-browser-error">{{ errorText }}</div>
    <div v-else class="dir-browser-list">
      <button
        v-if="parentPath"
        type="button"
        class="dir-entry dir-entry-parent"
        @click="navigateTo(parentPath)"
      >
        <span class="dir-entry-icon">↑</span>
        <span class="dir-entry-name">..</span>
      </button>
      <button
        v-for="entry in entries.filter(e => e.type === 'directory')"
        :key="entry.path"
        type="button"
        class="dir-entry"
        @click="navigateTo(entry.path)"
      >
        <span class="dir-entry-icon">📁</span>
        <span class="dir-entry-name">{{ entry.name }}</span>
      </button>
      <p v-if="!entries.filter(e => e.type === 'directory').length && !parentPath" class="dir-browser-empty">此目录没有子目录</p>
    </div>
  </div>
</template>

<style scoped>
.dir-browser {
  display: flex;
  flex-direction: column;
  height: 70dvh;
  overflow: hidden;
}

.dir-browser-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid var(--line);
  flex-shrink: 0;
}

.dir-browser-title {
  margin: 0;
  font-size: 15px;
  font-weight: 600;
}

.dir-browser-back,
.dir-browser-confirm {
  min-height: 36px;
  padding: 0 12px;
  border-radius: 999px;
  font-size: 13px;
  font-weight: 600;
}

.dir-browser-back {
  background: transparent;
  color: var(--muted);
}

.dir-browser-confirm {
  background: linear-gradient(180deg, #b8aa9b 0%, #a49586 100%);
  color: #fffdf9;
  box-shadow: 0 4px 10px rgba(133, 114, 96, 0.14);
}

.dir-browser-path {
  padding: 8px 16px;
  border-bottom: 1px solid rgba(226, 218, 210, 0.5);
  background: rgba(242, 236, 229, 0.5);
  flex-shrink: 0;
}

.dir-browser-path-text {
  font-size: 12px;
  font-family: 'SF Mono', 'Menlo', monospace;
  color: var(--muted);
  word-break: break-all;
}

.dir-browser-list {
  flex: 1;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
  padding: 4px 0;
}

.dir-entry {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  padding: 11px 16px;
  border: 0;
  background: transparent;
  text-align: left;
  font-size: 14px;
  color: var(--text);
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.dir-entry:active {
  background: rgba(161, 145, 129, 0.1);
}

.dir-entry-icon {
  flex: 0 0 24px;
  font-size: 16px;
  text-align: center;
}

.dir-entry-name {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.dir-entry-parent {
  color: var(--muted);
}

.dir-browser-loading,
.dir-browser-error,
.dir-browser-empty {
  padding: 24px 16px;
  text-align: center;
  color: var(--muted);
  font-size: 13px;
}

.dir-browser-error {
  color: #b42318;
}
</style>
