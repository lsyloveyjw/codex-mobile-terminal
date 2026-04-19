<script setup>
import { ref } from "vue";
import { request } from "../lib/api.js";
import DirectoryBrowser from "./DirectoryBrowser.vue";

const props = defineProps({
  defaultCwd: { type: String, default: "" }
});

const emit = defineEmits(["close", "created"]);

const cwd = ref(props.defaultCwd);
const creating = ref(false);
const errorText = ref("");
const showDirBrowser = ref(false);

async function handleCreate() {
  if (creating.value) {
    return;
  }
  if (!cwd.value.trim()) {
    errorText.value = "请选择或输入工作目录";
    return;
  }
  try {
    creating.value = true;
    errorText.value = "";
    const payload = await request("/api/sessions", {
      method: "POST",
      body: JSON.stringify({
        provider: "codex",
        cwd: cwd.value.trim()
      })
    });
    emit("created", payload.session);
  } catch (err) {
    errorText.value = err?.message || String(err);
  } finally {
    creating.value = false;
  }
}

function handleDirSelect(path) {
  cwd.value = path;
  showDirBrowser.value = false;
}

function handleBackdropClick(event) {
  if (event.target === event.currentTarget) {
    emit("close");
  }
}
</script>

<template>
  <div class="modal-backdrop" @click="handleBackdropClick">
    <div class="modal-sheet">
      <template v-if="!showDirBrowser">
        <div class="modal-header">
          <h2>新建会话</h2>
          <button type="button" class="modal-close" @click="emit('close')">✕</button>
        </div>

        <form class="modal-body" @submit.prevent="handleCreate">
          <label class="field">
            <span>工作目录</span>
            <div class="cwd-row">
              <input
                v-model="cwd"
                type="text"
                placeholder="选择或输入目录路径"
                class="cwd-input"
              />
              <button type="button" class="cwd-browse-btn" @click="showDirBrowser = true">
                浏览
              </button>
            </div>
          </label>

          <button class="primary-button" type="submit" :disabled="creating">
            {{ creating ? "创建中…" : "创建会话" }}
          </button>
          <p v-if="errorText" class="status-copy">{{ errorText }}</p>
        </form>
      </template>

      <DirectoryBrowser
        v-else
        :initial-path="cwd"
        @select="handleDirSelect"
        @close="showDirBrowser = false"
      />
    </div>
  </div>
</template>

<style scoped>
.modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 2100;
  display: grid;
  place-items: end center;
  background: rgba(60, 50, 40, 0.32);
  backdrop-filter: blur(4px);
}

.modal-sheet {
  width: min(100%, 480px);
  max-height: 85dvh;
  overflow-y: auto;
  border-radius: 22px 22px 0 0;
  background: rgba(255, 251, 247, 0.97);
  border: 1px solid rgba(228, 219, 210, 0.9);
  box-shadow: 0 -14px 40px rgba(89, 73, 58, 0.1);
  padding: 20px 20px calc(20px + env(safe-area-inset-bottom));
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.modal-header h2 {
  margin: 0;
  font-size: 17px;
  font-weight: 600;
}

.modal-close {
  width: 32px;
  height: 32px;
  border-radius: 999px;
  font-size: 14px;
  color: var(--muted);
  display: grid;
  place-items: center;
}

.modal-close:active {
  background: rgba(0, 0, 0, 0.06);
}

.modal-body {
  display: flex;
  flex-direction: column;
}

.cwd-row {
  display: flex;
  gap: 8px;
  align-items: stretch;
}

.cwd-input {
  flex: 1;
  min-width: 0;
  padding: 12px 14px;
  border: 1px solid var(--line);
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.9);
  font-size: 14px;
  font-family: 'SF Mono', 'Menlo', monospace;
}

.cwd-input:focus {
  outline: none;
  border-color: var(--accent);
}

.cwd-browse-btn {
  flex: 0 0 auto;
  padding: 0 14px;
  min-height: 44px;
  border: 1px solid var(--line-strong);
  border-radius: 14px;
  background: var(--soft);
  color: var(--text);
  font-size: 13px;
  font-weight: 600;
}

.cwd-browse-btn:active {
  background: var(--line);
}
</style>
