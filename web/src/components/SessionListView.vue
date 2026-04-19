<script setup>
import { computed } from "vue";
import { isLowSignalTitle } from "../lib/session-helpers.js";

const props = defineProps({
  groups: { type: Array, default: () => [] },
  activeSessionId: { type: String, default: "" },
  pendingSessionId: { type: String, default: "" },
  activeRunning: { type: Boolean, default: false },
  formatRelativeTime: { type: Function, required: true }
});

const emit = defineEmits(["open", "create-group-session"]);

const STATUS_GROUPS = [
  { key: "needs-you", label: "需要你", icon: "!" },
  { key: "running", label: "运行中", icon: "▶" },
  { key: "quiet", label: "安静", icon: "—" }
];

function classifySession(session) {
  const isActive = session.id === props.activeSessionId;
  if (isActive) {
    return props.activeRunning ? "running" : "needs-you";
  }
  return "quiet";
}

function sessionStatus(session) {
  const cls = classifySession(session);
  if (cls === "running") return "running";
  if (cls === "needs-you") return "needs-you";
  return "quiet";
}

function statusColor(status) {
  if (status === "running") return "var(--st-running)";
  if (status === "needs-you") return "var(--st-success)";
  return "var(--st-idle)";
}

function repoMark(name) {
  return String(name || "?")
    .replace(/[^A-Za-z0-9]/g, "")
    .toUpperCase()
    .slice(0, 2) || "?";
}

const REPO_COLORS = [
  "#2E4B5F", "#7A3F2E", "#3F5A3A", "#5D4A70",
  "#7E6A2E", "#4A3B5F", "#2E5F4B", "#5F3E3E"
];

function repoColor(name) {
  let hash = 0;
  for (const ch of String(name || "")) {
    hash = ((hash << 5) - hash + ch.charCodeAt(0)) | 0;
  }
  return REPO_COLORS[Math.abs(hash) % REPO_COLORS.length];
}

function eventBadgeText(session) {
  const status = classifySession(session);
  if (status === "needs-you") return "待审批";
  if (status === "running") return "运行";
  const preview = String(session.displayPreview || "").trim();
  if (/error|fail|失败/i.test(preview)) return "出错";
  if (/done|complete|完成|已推送/i.test(preview)) return "完成";
  return "对话";
}

function eventBadgeStyle(session) {
  const text = eventBadgeText(session);
  const map = {
    "待审批": { fg: "var(--ev-approval)", bg: "rgba(201,122,20,0.12)" },
    "运行":   { fg: "var(--ev-action)",   bg: "var(--accent-soft)" },
    "出错":   { fg: "var(--ev-error)",    bg: "rgba(194,77,77,0.1)" },
    "完成":   { fg: "var(--st-success)",   bg: "var(--accent-soft)" },
    "对话":   { fg: "var(--ev-thought)",   bg: "rgba(139,149,163,0.12)" },
  };
  return map[text] || map["对话"];
}

const allSessions = computed(() => {
  const sessions = [];
  for (const group of props.groups) {
    for (const session of group.sessions) {
      // Keep active/pending sessions regardless
      const isActive = session.id === props.activeSessionId || session.id === props.pendingSessionId;
      if (isActive) {
        sessions.push({ ...session, _groupName: group.name });
        continue;
      }
      // Filter out low-signal sessions (noise titles, generic names)
      const titleLow = isLowSignalTitle(session.displayTitle, session);
      const previewLow = isLowSignalTitle(session.displayPreview, session);
      if (titleLow && previewLow) {
        continue;
      }
      sessions.push({ ...session, _groupName: group.name });
    }
  }
  return sessions;
});

const statusGroups = computed(() => {
  const buckets = { "needs-you": [], running: [], quiet: [] };
  for (const session of allSessions.value) {
    const status = classifySession(session);
    buckets[status].push(session);
  }
  return STATUS_GROUPS.map((g) => ({
    ...g,
    sessions: buckets[g.key]
  })).filter((g) => g.sessions.length > 0);
});
</script>

<template>
  <main class="session-screen">
    <div class="global-status-bar">
      <div class="status-dot"></div>
      <div class="status-bar-info">
        <span class="status-bar-host">已连接</span>
        <span class="status-bar-dot">·</span>
        <span class="status-bar-meta">{{ allSessions.length }} 个会话</span>
      </div>
      <span class="status-bar-counts">
        <span class="count-waiting">{{ statusGroups.find(g => g.key === 'needs-you')?.sessions.length || 0 }}</span> 待审 ·
        <span class="count-running">{{ statusGroups.find(g => g.key === 'running')?.sessions.length || 0 }}</span> 运行
      </span>
    </div>

    <section v-if="statusGroups.length" class="session-groups">
      <section v-for="group in statusGroups" :key="group.key" class="status-group">
        <div class="status-group-head">
          <h2>{{ group.label }}</h2>
        </div>
        <div class="status-group-body">
          <button
            v-for="session in group.sessions"
            :key="session.id"
            class="session-row"
            :class="{
              active: session.id === activeSessionId,
              pending: session.id === pendingSessionId,
              ['status-' + classifySession(session)]: true,
              emphasis: classifySession(session) === 'needs-you'
            }"
            :aria-current="session.id === activeSessionId ? 'true' : undefined"
            @click="emit('open', session)"
          >
            <div class="session-spine" :style="{ backgroundColor: statusColor(classifySession(session)) }"></div>
            <div class="session-row-inner">
              <div class="session-row-top">
                <div class="session-repo-mark" :style="{ background: repoColor(session._groupName) }">{{ repoMark(session._groupName) }}</div>
                <div class="session-row-title-area">
                  <div class="session-row-name-row">
                    <span class="session-row-title">{{ session.displayTitle }}</span>
                  </div>
                  <div class="session-row-branch">{{ session._groupName }}</div>
                </div>
                <time class="session-row-time">{{ formatRelativeTime(session.updatedAt) }}</time>
              </div>
              <div class="session-row-bottom">
                <span class="event-badge" :style="{ color: eventBadgeStyle(session).fg, background: eventBadgeStyle(session).bg }">
                  {{ eventBadgeText(session) }}
                </span>
                <p class="session-row-preview">{{ session.displayPreview }}</p>
              </div>
              <div v-if="classifySession(session) === 'running'" class="session-progress">
                <div class="session-progress-bar"></div>
              </div>
            </div>
          </button>
        </div>
      </section>
    </section>

    <div v-else class="empty-state">还没有可展示的会话。</div>
  </main>
</template>

<style scoped>
.session-screen {
  display: flex;
  flex-direction: column;
  gap: 0;
  min-width: 0;
  padding: 0 0 calc(16px + env(safe-area-inset-bottom));
}

/* ===== Global Status Bar ===== */
.global-status-bar {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 0 16px 12px;
  padding: 10px 12px;
  background: var(--panel);
  border-radius: 8px;
  border: 0.5px solid var(--line-strong);
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 4px;
  background: var(--accent);
  box-shadow: 0 0 0 3px var(--accent-soft);
  flex-shrink: 0;
}

.status-bar-info {
  flex: 1;
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--text-sec);
  letter-spacing: -0.2px;
}

.status-bar-host {
  color: var(--text);
  font-weight: 600;
}

.status-bar-dot {
  margin: 0 2px;
}

.status-bar-meta {
  color: var(--text-ter);
}

.status-bar-counts {
  font-family: var(--font-mono);
  font-size: 10px;
  color: var(--text-ter);
}

.count-waiting {
  color: var(--ev-approval);
  font-weight: 600;
}

.count-running {
  color: var(--ev-action);
  font-weight: 600;
}

/* ===== Session Groups ===== */
.session-groups {
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-width: 0;
}

.status-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.status-group-head {
  display: flex;
  align-items: center;
  gap: 0;
  padding: 6px 16px 4px;
}

.status-group-head h2 {
  margin: 0;
  font-family: var(--font-mono);
  font-size: 10px;
  font-weight: 600;
  color: var(--text-ter);
  letter-spacing: 1px;
  text-transform: uppercase;
}

.status-group-count {
  padding: 2px 7px;
  border-radius: 999px;
  background: var(--accent-soft);
  color: var(--accent);
  font-size: 11px;
  font-weight: 600;
  line-height: 1.2;
}

.status-group-body {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

/* ===== Session Row ===== */
.session-row {
  position: relative;
  width: 100%;
  margin: 4px 12px;
  padding: 12px 12px 12px 14px;
  background: var(--panel);
  border-radius: 8px;
  border: 0.5px solid var(--line);
  color: inherit;
  text-align: left;
  cursor: pointer;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
  transition: transform 0.14s ease, background-color 0.14s ease, border-color 0.14s ease;
  box-sizing: border-box;
  max-width: calc(100% - 24px);
}

.session-row.emphasis {
  border: 1px solid var(--accent-edge);
  box-shadow: 0 2px 10px var(--accent-soft);
}

.session-row:active {
  transform: translateY(1px) scale(0.998);
}

.session-row.active {
  border-color: var(--accent-edge);
  background: linear-gradient(180deg, rgba(13, 143, 124, 0.04), rgba(13, 143, 124, 0.01));
}

.session-row.pending {
  opacity: 0.6;
}

/* Left spine — absolute positioned */
.session-spine {
  position: absolute;
  top: 10px;
  bottom: 10px;
  left: 0;
  width: 2px;
  border-radius: 2px;
}

.session-row-inner {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.session-row-top {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}

/* Repo mark — unique color per repo, white text */
.session-repo-mark {
  width: 32px;
  height: 32px;
  border-radius: 6px;
  color: #fff;
  font-size: 12px;
  font-weight: 600;
  font-family: var(--font-mono);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  letter-spacing: -0.5px;
  box-shadow: inset 0 0 0 0.5px rgba(0,0,0,0.1);
}

.session-row-title-area {
  flex: 1;
  min-width: 0;
}

.session-row-name-row {
  display: flex;
  align-items: center;
  gap: 6px;
}

.session-row-title {
  margin: 0;
  overflow: hidden;
  color: var(--text);
  font-size: 15px;
  font-weight: 600;
  line-height: 1.3;
  text-overflow: ellipsis;
  white-space: nowrap;
  letter-spacing: -0.2px;
}

.session-row-branch {
  margin: 1px 0 0;
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--text-ter);
}

.session-row-time {
  flex: 0 0 auto;
  font-family: var(--font-mono);
  color: var(--text-ter);
  font-size: 10px;
  white-space: nowrap;
}

.session-row-bottom {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: 42px;
  min-width: 0;
}

.event-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 1px 6px;
  border-radius: 3px;
  font-family: var(--font-mono);
  font-size: 10px;
  font-weight: 600;
  letter-spacing: 0.4px;
  text-transform: uppercase;
  white-space: nowrap;
  flex-shrink: 0;
}

.session-row-preview {
  margin: 0;
  flex: 1;
  min-width: 0;
  font-size: 12px;
  line-height: 1.35;
  color: var(--text-sec);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Progress bar for running sessions */
.session-progress {
  margin-left: 42px;
  margin-top: 2px;
  height: 2px;
  background: var(--line);
  border-radius: 2px;
  overflow: hidden;
}

.session-progress-bar {
  height: 100%;
  width: 42%;
  border-radius: 2px;
  background: var(--st-running);
  transition: width 0.4s;
  animation: progress-slide 1.8s ease-in-out infinite;
}

@keyframes progress-slide {
  0% { transform: translateX(-100%); }
  100% { transform: translateX(350%); }
}

/* Empty state */
.empty-state {
  padding: 24px 18px;
  border: 1px dashed var(--line-strong);
  border-radius: 14px;
  background: var(--panel);
  color: var(--text-ter);
  font-size: 13px;
  text-align: center;
}

@media (min-width: 700px) {
  .session-screen {
    padding: 14px 16px 20px;
  }
}
</style>
