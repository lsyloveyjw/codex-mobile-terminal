<script setup>
import { computed, ref } from "vue";

const props = defineProps({
  loading: Boolean,
  statusText: { type: String, default: "" },
  modelValue: { type: String, default: "" },
  rememberToken: { type: Boolean, default: true }
});

const emit = defineEmits(["update:modelValue", "update:rememberToken", "submit"]);

const accessTokenModel = computed({
  get: () => props.modelValue,
  set: (value) => emit("update:modelValue", value)
});

const rememberTokenModel = computed({
  get: () => props.rememberToken,
  set: (value) => emit("update:rememberToken", value)
});

const showUsage = ref(false);
</script>

<template>
  <section class="login-view">
    <div class="login-card">
      <p class="login-kicker">Codex</p>
      <h1>登录</h1>
      <form @submit.prevent="emit('submit')">
      <label class="field">
        <span>Access Token</span>
        <input
          v-model="accessTokenModel"
          type="password"
          placeholder="输入 token"
          autocomplete="current-password"
        />
      </label>
      <label class="remember-field">
        <input
          v-model="rememberTokenModel"
          type="checkbox"
        />
        <span>在当前设备记住 token</span>
      </label>
      <button class="primary-button" type="submit" :disabled="loading">
        {{ loading ? "进入中..." : "进入" }}
      </button>
      <p v-if="statusText" class="status-copy">{{ statusText }}</p>
      </form>

      <div class="usage-section">
        <button type="button" class="usage-toggle" @click="showUsage = !showUsage">
          <span class="usage-toggle-icon">{{ showUsage ? '▾' : '▸' }}</span>
          <span>使用说明</span>
        </button>
        <div v-if="showUsage" class="usage-body">
          <div class="usage-block">
            <p class="usage-heading">这是什么？</p>
            <p class="usage-text">基于浏览器的 Codex 终端，可以在手机或任何设备的浏览器中远程使用电脑上的 Codex CLI。</p>
          </div>
          <div class="usage-block">
            <p class="usage-heading">如何获取 Access Token？</p>
            <p class="usage-text">在运行此服务的电脑上，查看项目目录下 <code>.env</code> 文件中 <code>ACCESS_TOKEN</code> 的值。服务启动时终端日志也会显示自动生成的 token。</p>
          </div>
          <div class="usage-block">
            <p class="usage-heading">手机如何访问？</p>
            <p class="usage-text">确保手机和电脑在同一 WiFi 网络下，使用电脑的局域网 IP 地址访问（如 <code>http://192.168.x.x:3210</code>）。若使用 Tailscale，则用 Tailscale IP 访问即可。</p>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.usage-section {
  margin-top: 18px;
  border-top: 1px solid var(--line);
  padding-top: 14px;
}

.usage-toggle {
  display: flex;
  align-items: center;
  gap: 6px;
  width: 100%;
  padding: 6px 0;
  font-size: 13px;
  color: var(--muted);
}

.usage-toggle-icon {
  font-size: 11px;
}

.usage-body {
  margin-top: 8px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.usage-block {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.usage-heading {
  margin: 0;
  font-size: 13px;
  font-weight: 600;
  color: var(--text);
}

.usage-text {
  margin: 0;
  font-size: 12px;
  line-height: 1.5;
  color: var(--muted);
}

.usage-text code {
  padding: 1px 5px;
  border-radius: 6px;
  background: rgba(0, 0, 0, 0.05);
  font-size: 11px;
  font-family: 'SF Mono', 'Menlo', 'PingFang SC', monospace;
}
</style>
