<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from "vue";
import MarkdownIt from "markdown-it";
import DOMPurify from "dompurify";
import hljs from "highlight.js/lib/core";
import bash from "highlight.js/lib/languages/bash";
import javascript from "highlight.js/lib/languages/javascript";
import typescript from "highlight.js/lib/languages/typescript";
import python from "highlight.js/lib/languages/python";
import json from "highlight.js/lib/languages/json";
import css from "highlight.js/lib/languages/css";
import yaml from "highlight.js/lib/languages/yaml";
import xml from "highlight.js/lib/languages/xml";
import sql from "highlight.js/lib/languages/sql";
import markdown from "highlight.js/lib/languages/markdown";
import plaintext from "highlight.js/lib/languages/plaintext";

hljs.registerLanguage("bash", bash);
hljs.registerLanguage("shell", bash);
hljs.registerLanguage("sh", bash);
hljs.registerLanguage("zsh", bash);
hljs.registerLanguage("javascript", javascript);
hljs.registerLanguage("js", javascript);
hljs.registerLanguage("typescript", typescript);
hljs.registerLanguage("ts", typescript);
hljs.registerLanguage("tsx", typescript);
hljs.registerLanguage("jsx", javascript);
hljs.registerLanguage("python", python);
hljs.registerLanguage("py", python);
hljs.registerLanguage("json", json);
hljs.registerLanguage("css", css);
hljs.registerLanguage("scss", css);
hljs.registerLanguage("yaml", yaml);
hljs.registerLanguage("yml", yaml);
hljs.registerLanguage("xml", xml);
hljs.registerLanguage("html", xml);
hljs.registerLanguage("sql", sql);
hljs.registerLanguage("markdown", markdown);
hljs.registerLanguage("md", markdown);
hljs.registerLanguage("plaintext", plaintext);
hljs.registerLanguage("text", plaintext);

const BOTTOM_THRESHOLD = 84;
const MAX_COMPOSER_HEIGHT = 160;
const VISIBLE_BUFFER = 8;
const PAGE_SIZE = 25;

const props = defineProps({
  sessionKey: { type: String, default: "" },
  openToken: { type: [String, Number], default: 0 },
  title: { type: String, default: "会话" },
  threadId: { type: String, default: "" },
  expectedThreadId: { type: String, default: "" },
  threadMismatch: { type: Boolean, default: false },
  workspaceName: { type: String, default: "" },
  assistantName: { type: String, default: "Codex" },
  messages: { type: Array, default: () => [] },
  draft: { type: String, default: "" },
  canSend: Boolean,
  canInterrupt: Boolean,
  loading: Boolean,
  statusText: { type: String, default: "" }
});

const emit = defineEmits(["back", "update:draft", "submit", "interrupt"]);
const messageListEl = ref(null);
const composerEl = ref(null);
const viewportHeight = ref(0);
const keyboardInset = ref(0);
const isPinnedToBottom = ref(true);
const isTouchDevice = ref(false);
const showProcessDetails = ref(false);
const expandedEventGroups = ref(new Set());
const lightboxImage = ref(null);
const firstVisibleIdx = ref(0);
const lastVisibleIdx = ref(-1);
const renderedHtmlCache = new Map();
// 分页加载：displayStartIdx 表示当前展示的起始下标（0 = 全部展示）
const displayStartIdx = ref(0);
const isLoadingMore = ref(false);
let loadMoreCooldownUntil = 0; // 加载历史后的冷却截止时间戳，防止抖动循环
const lightboxScale = ref(1);
const lightboxTranslateX = ref(0);
const lightboxTranslateY = ref(0);
const lightboxGesture = {
  mode: "",
  startScale: 1,
  startDistance: 0,
  startTranslateX: 0,
  startTranslateY: 0,
  startTouchX: 0,
  startTouchY: 0,
  startMidpointX: 0,
  startMidpointY: 0
};

const chatShellStyle = computed(() => ({
  "--chat-vh": viewportHeight.value ? `${viewportHeight.value}px` : undefined,
  "--chat-keyboard-inset": `${keyboardInset.value}px`
}));
const isRunning = computed(() => Boolean(props.canInterrupt));
const hasDraft = computed(() => String(props.draft || "").trim().length > 0);
const primaryActionLabel = computed(() => (isRunning.value ? "中断" : "发送"));
const canPrimaryAction = computed(() => (isRunning.value ? !props.loading : props.canSend && !props.loading));

const statusIndicator = computed(() => {
  const text = String(props.statusText || "").trim();
  const running = Boolean(props.canInterrupt);

  if (!text && running) {
    return { state: "streaming", label: "正在回复…", icon: "⏳" };
  }
  if (/正在连接/.test(text)) {
    return { state: "connecting", label: text, icon: "🔗" };
  }
  if (/正在发送/.test(text)) {
    return { state: "sending", label: text, icon: "📤" };
  }
  if (/等待.*回复/.test(text)) {
    return { state: "waiting", label: text, icon: "⏳" };
  }
  if (/中断/.test(text)) {
    return { state: "interrupted", label: text, icon: "⏹" };
  }
  if (/已结束/.test(text)) {
    return { state: "completed", label: text, icon: "✓" };
  }
  if (text && (/失败|异常退出|已关闭|错误|不可用|缺失|权限/.test(text))) {
    return { state: "error", label: text, icon: "⚠" };
  }
  if (text) {
    return { state: "info", label: text, icon: "ℹ" };
  }
  return { state: "idle", label: "", icon: "" };
});
const lightboxImageStyle = computed(() => ({
  transform: `translate3d(${lightboxTranslateX.value}px, ${lightboxTranslateY.value}px, 0) scale(${lightboxScale.value})`
}));

const PROCESS_PATTERNS = [
  /^›/,
  /^>/,
  /^Working\(/i,
  /^\d+% left/i,
  /^tokens?\b/i,
  /^subagent/i,
  /^thinking\b/i,
  /^•\s+/,
  /^tool\b/i,
  /^observation\b/i,
  /^bash\b/i,
  /^zsh\b/i,
  /^pwd\b/i,
  /^cd\b/i,
  /^\/Users\//,
  /^node_modules\//,
  /^<subagent_notification>/i,
  /^<\/subagent_notification>/i,
  /^\{".*agent_path".*\}$/,
  /esc to interrupt/i,
  /dangerously-bypass-approvals-and-sandbox/i,
  /codex resume/i,
  /'codex'.*'resume'/i,
  /current changes/i,
  /\bworkdir\b/i
];

const LANGUAGE_ALIAS = {
  sh: "bash",
  shell: "bash",
  zsh: "bash",
  console: "bash",
  env: "bash",
  yml: "yaml",
  conf: "ini",
  config: "ini",
  text: "plaintext",
  plain: "plaintext",
  md: "markdown"
};

function normalizeLanguageTag(lang) {
  const key = String(lang || "")
    .trim()
    .toLowerCase();
  if (!key) {
    return "";
  }
  return LANGUAGE_ALIAS[key] || key;
}

function classifyLanguageFamily(language) {
  const key = String(language || "").toLowerCase();
  if (!key) {
    return "plain";
  }
  if (["sql", "postgresql", "mysql", "plsql"].includes(key)) {
    return "sql";
  }
  if (["bash", "shell", "zsh", "powershell", "pwsh", "fish", "sh"].includes(key)) {
    return "shell";
  }
  if (
    [
      "javascript",
      "typescript",
      "tsx",
      "jsx",
      "json",
      "yaml",
      "toml",
      "ini",
      "xml",
      "html",
      "css",
      "scss",
      "less",
      "dockerfile",
      "nginx",
      "markdown"
    ].includes(key)
  ) {
    return "config";
  }
  return "code";
}

function renderCodeFence(sourceText, languageTag) {
  const normalizedLanguage = normalizeLanguageTag(languageTag);
  let highlighted = "";
  let resolvedLanguage = "";

  if (normalizedLanguage && hljs.getLanguage(normalizedLanguage)) {
    highlighted = hljs.highlight(sourceText, {
      language: normalizedLanguage,
      ignoreIllegals: true
    }).value;
    resolvedLanguage = normalizedLanguage;
  } else {
    highlighted = sourceText.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
    resolvedLanguage = "plaintext";
  }

  const family = classifyLanguageFamily(resolvedLanguage);
  const languageLabel = resolvedLanguage === "plaintext" ? "text" : resolvedLanguage;
  return [
    `<pre class="code-block language-${resolvedLanguage} family-${family}">`,
    `<span class="code-lang">${languageLabel}</span>`,
    `<code class="hljs language-${resolvedLanguage}">${highlighted}</code>`,
    "</pre>"
  ].join("");
}

function isLocalFilePathHref(href) {
  const value = String(href || "").trim();
  if (!value) {
    return false;
  }
  return (
    /^\/(Users|Volumes|private|tmp|var|opt|Applications)\//.test(value) ||
    /^[A-Za-z]:[\\/]/.test(value)
  );
}

function isProcessLine(line) {
  const compact = String(line || "").trim();
  if (!compact) {
    return true;
  }
  return PROCESS_PATTERNS.some((pattern) => pattern.test(compact));
}

function splitMessageParts(message) {
  const text = String(message?.text || "").trim();
  if (!text || message?.role === "user") {
    return { primary: text, process: "" };
  }

  const lines = text
    .split("\n")
    .map((line) => line.trimEnd())
    .filter((line) => String(line || "").trim().length > 0);

  const processLines = lines.filter((line) => isProcessLine(line));
  return {
    // Keep full assistant text visible by default to avoid hiding real content.
    primary: lines.join("\n").trim(),
    process: processLines.join("\n").trim()
  };
}

const md = new MarkdownIt({
  html: false,
  breaks: true,
  linkify: true,
  typographer: true,
  highlight(code, language) {
    return renderCodeFence(String(code || ""), language);
  }
});

const sanitizerConfig = {
  ALLOWED_TAGS: [
    "p",
    "br",
    "pre",
    "code",
    "blockquote",
    "ul",
    "ol",
    "li",
    "a",
    "strong",
    "em",
    "del",
    "h1",
    "h2",
    "h3",
    "h4",
    "h5",
    "h6",
    "hr",
    "img",
    "span"
  ],
  ALLOWED_ATTR: ["href", "target", "rel", "class", "src", "alt", "title", "loading", "decoding", "referrerpolicy"],
  FORBID_ATTR: ["style", "onerror", "onclick", "onload"]
};

function renderMarkdownToHtml(text, cacheKey) {
  if (cacheKey && renderedHtmlCache.has(cacheKey)) {
    return renderedHtmlCache.get(cacheKey);
  }
  const source = preprocessDisplayMarkdown(String(text || ""));
  if (!source.trim()) {
    return "";
  }
  const rendered = md.render(source);
  const sanitized = DOMPurify.sanitize(rendered, sanitizerConfig).trim();
  if (typeof window === "undefined" || !sanitized) {
    return sanitized;
  }

  const container = window.document.createElement("div");
  container.innerHTML = sanitized;
  for (const link of container.querySelectorAll("a[href]")) {
    const href = String(link.getAttribute("href") || "").trim();
    if (!isLocalFilePathHref(href)) {
      continue;
    }
    const replacement = window.document.createElement("span");
    replacement.className = "local-file-ref";
    replacement.textContent = link.textContent || href;
    link.replaceWith(replacement);
  }
  const result = container.innerHTML.trim();
  if (cacheKey) {
    renderedHtmlCache.set(cacheKey, result);
  }
  return result;
}

function prettifyDirectiveLine(line) {
  const match = String(line || "").trim().match(/^::([a-z0-9-]+)\{([\s\S]*)\}$/i);
  if (!match) {
    return null;
  }

  const action = match[1];
  const payload = match[2].trim();
  return [
    `> 操作：\`${action}\``,
    payload ? `> 参数：\`${payload}\`` : ""
  ]
    .filter(Boolean)
    .join("\n");
}

function preprocessDisplayMarkdown(value) {
  const lines = String(value || "").split("\n");
  const output = [];
  for (const raw of lines) {
    const line = raw.trimEnd();
    const directive = prettifyDirectiveLine(line);
    if (directive) {
      output.push("", directive, "");
      continue;
    }
    output.push(line);
  }
  return output.join("\n").replace(/\n{3,}/g, "\n\n");
}

function extractFileMentions(value) {
  const source = String(value || "");
  if (!source) {
    return [];
  }
  const matches = [...source.matchAll(/\[[^\]]+\]\((\/[^)\s]+)\)/g)];
  const seen = new Set();
  const files = [];
  for (const match of matches) {
    const rawPath = String(match[1] || "").trim();
    if (!rawPath || seen.has(rawPath)) {
      continue;
    }
    seen.add(rawPath);
    const normalized = rawPath.replace(/#L\d+(C\d+)?$/i, "").replace(/:\d+(?::\d+)?$/i, "");
    const label = normalized.split("/").filter(Boolean).slice(-3).join("/");
    files.push({
      path: rawPath,
      label: label || rawPath
    });
  }
  return files;
}

function compactText(value) {
  return String(value || "").replace(/\s+/g, " ").trim();
}

function mapEventTypeToZh(rawType, fallbackLabel = "事件") {
  const key = String(rawType || "")
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9_/-]/g, "");
  if (!key) {
    return fallbackLabel;
  }
  if (key.includes("commandexecution")) {
    return "命令执行";
  }
  if (key.includes("filechange")) {
    return "文件变更";
  }
  if (key.includes("reasoning")) {
    return "思考过程";
  }
  if (key.includes("agentmessage")) {
    return "执行进度";
  }
  if (key.includes("subagent") || key.includes("collab")) {
    return "子任务";
  }
  if (key.includes("reference") || key.includes("citation")) {
    return "引用信息";
  }
  if (key.includes("toolcall") || key.includes("mcptool")) {
    return "工具调用";
  }
  return fallbackLabel;
}

function mapStatusToZh(value) {
  const key = String(value || "").trim().toLowerCase();
  if (!key) {
    return "";
  }
  if (["success", "succeeded", "done", "completed", "ok"].includes(key)) {
    return "完成";
  }
  if (["running", "inprogress", "processing"].includes(key)) {
    return "执行中";
  }
  if (["failed", "error", "aborted", "cancelled", "canceled"].includes(key)) {
    return "失败";
  }
  return String(value || "").trim();
}

function isNoisySummary(value, rawType) {
  const text = compactText(value).toLowerCase();
  const type = compactText(rawType).toLowerCase();
  if (!text) {
    return true;
  }
  if (text === type) {
    return true;
  }
  if (["commentary", "reasoning", "agentmessage", "commandexecution", "event", "tool"].includes(text)) {
    return true;
  }
  return false;
}

function buildEventDetails(payload, rawType) {
  const source = payload && typeof payload === "object" ? payload : {};
  const nested = source.payload && typeof source.payload === "object" ? source.payload : {};
  const fields = [
    { label: "命令", value: nested.command || source.command },
    { label: "工具", value: nested.toolName || nested.tool_name || source.toolName || source.tool_name },
    { label: "路径", value: nested.path || source.path },
    { label: "状态", value: mapStatusToZh(nested.status || source.status) }
  ];
  const details = fields
    .map((item) => ({ ...item, value: compactText(item.value) }))
    .filter((item) => Boolean(item.value));
  if (details.length) {
    return details;
  }
  const fallback = mapEventTypeToZh(rawType, "");
  return fallback ? [{ label: "类型", value: fallback }] : [];
}

function buildEventCompactText(summary, details, fallbackType) {
  const cleanSummary = compactText(summary);
  if (cleanSummary) {
    return cleanSummary;
  }
  if (Array.isArray(details) && details.length > 0) {
    const first = details[0];
    if (first?.label && first?.value) {
      return `${first.label}：${first.value}`;
    }
  }
  return fallbackType || "事件";
}

function classifyAssistantDisplay(displayText) {
  const text = String(displayText || "").trim();
  if (!text) return "empty";
  if (text.length <= 80 && !/[.。!！?？]$/.test(text)) return "thought";
  return "reply";
}

function getEventBadgeInfo(item) {
  if (item.role === "user") return { label: "你", fg: "#0D8F7C", bg: "rgba(13,143,124,0.12)" };
  if (item.renderKind === "event") {
    const type = String(item.eventLabel || "").trim();
    if (/文件变更|改动|diff/i.test(type)) return { label: "DIFF", fg: "var(--ev-diff)", bg: "rgba(91,108,255,0.1)" };
    if (/命令执行|执行/i.test(type)) return { label: "执行", fg: "var(--ev-action)", bg: "var(--accent-soft)" };
    if (/思考|推理/i.test(type)) return { label: "思考", fg: "var(--ev-thought)", bg: "rgba(139,149,163,0.12)" };
    if (/子任务/i.test(type)) return { label: "子代理", fg: "var(--ev-action)", bg: "var(--accent-soft)" };
    return { label: type || "事件", fg: "var(--ev-action)", bg: "var(--accent-soft)" };
  }
  if (item.role === "assistant") {
    const cls = classifyAssistantDisplay(item.displayText || "");
    if (cls === "thought") return { label: "思考", fg: "var(--ev-thought)", bg: "rgba(139,149,163,0.12)" };
    return { label: "回复", fg: "var(--ev-action)", bg: "var(--accent-soft)" };
  }
  return { label: "消息", fg: "var(--ev-action)", bg: "var(--accent-soft)" };
}

function formatEventTime(ts) {
  const date = new Date(ts);
  if (!Number.isFinite(date.getTime())) return "";
  return date.toLocaleTimeString("zh-CN", { hour: "2-digit", minute: "2-digit", hour12: false });
}

const slashCommands = [
  { label: "/继续", text: "/continue" },
  { label: "/回滚", text: "/rollback" },
  { label: "/解释这段", text: "/explain" },
  { label: "/跑测试", text: "/test" }
];

function insertSlashCommand(text) {
  emit("update:draft", text);
  nextTick(() => {
    if (composerEl.value) {
      composerEl.value.focus();
    }
  });
}

function formatDurationLabel(ms) {
  const totalSeconds = Math.max(0, Math.floor(Number(ms || 0) / 1000));
  const minutes = Math.floor(totalSeconds / 60);
  const seconds = totalSeconds % 60;
  if (minutes <= 0) {
    return `${seconds}s`;
  }
  return `${minutes}m ${seconds}s`;
}

// 当前分页展示的消息切片（从 displayStartIdx 开始到末尾）
const pagedMessages = computed(() => {
  const start = Math.max(0, displayStartIdx.value);
  return props.messages.slice(start);
});

const hasMoreHistory = computed(() => displayStartIdx.value > 0);

const renderedMessages = computed(() => {
  const msgs = pagedMessages.value;
  const total = msgs.length;
  const lo = Math.max(0, firstVisibleIdx.value - VISIBLE_BUFFER);
  const hi = Math.min(total - 1, lastVisibleIdx.value + VISIBLE_BUFFER);

  return msgs.map((message, idx) => {
    const parts = splitMessageParts(message);
    const partType = String(message?.partType || "").trim();
    const payload = message?.payload || {};
    const imageUrl = String(payload?.url || "").trim();
    const imageAlt = String(payload?.alt || "").trim() || "image";
    const structuredPart = new Set(["tool", "subagent", "reference", "event"]);
    const renderKind = partType === "image" && imageUrl ? "image" : structuredPart.has(partType) ? "event" : "markdown";
    const eventLabelMap = {
      tool: "工具调用",
      subagent: "子任务",
      reference: "引用",
      event: "事件"
    };
    const eventLabel = eventLabelMap[partType] || "事件";
    const eventRawType = String(payload?.rawType || message?.rawType || "").trim();
    const eventTypeText = mapEventTypeToZh(eventRawType, eventLabel);
    const baseSummary = compactText(payload?.summary || parts.primary || message.text || "");
    const eventSummary = isNoisySummary(baseSummary, eventRawType) ? "" : baseSummary;
    const eventDetails = buildEventDetails(payload, eventRawType);
    const eventCompactText = buildEventCompactText(eventSummary, eventDetails, eventTypeText);
    const displayText = parts.primary || "";

    const inRange = idx >= lo && idx <= hi;
    const cacheKey = String(message.id || idx);

    return {
      ...message,
      renderKind,
      imageUrl,
      imageAlt,
      eventLabel: eventTypeText,
      eventSummary,
      eventDetails,
      eventCompactText,
      displayText,
      renderedHtml: inRange ? renderMarkdownToHtml(displayText || message.text || "", cacheKey) : "",
      fileMentions: inRange && message.role === "assistant" && renderKind === "markdown" ? extractFileMentions(message.text) : [],
      processText: parts.process || "",
      hasProcessDetails: Boolean(parts.process),
      processSummary: parts.primary ? "查看过程详情" : "查看运行过程",
      assistantDisplay: message.role === "assistant" && renderKind === "markdown"
        ? classifyAssistantDisplay(parts.primary || "")
        : "",
      _idx: idx
    };
  });
});

const hasAnyProcessDetails = computed(() => renderedMessages.value.some((message) => message.hasProcessDetails));
const hasAssistantBody = computed(() =>
  renderedMessages.value.some(
    (message) => message.role === "assistant" && message.renderKind === "markdown" && compactText(message.displayText)
  )
);
const shouldCollapseCompletedGroups = computed(() => hasAssistantBody.value && !props.canInterrupt && !props.loading);
function isEventGroupExpanded(toggleId) {
  return expandedEventGroups.value.has(toggleId);
}

function toggleEventGroup(toggleId) {
  const next = new Set(expandedEventGroups.value);
  if (next.has(toggleId)) {
    next.delete(toggleId);
  } else {
    next.add(toggleId);
  }
  expandedEventGroups.value = next;
}

const visibleMessages = computed(() => {
  if (!shouldCollapseCompletedGroups.value) {
    return renderedMessages.value;
  }

  const output = [];
  let pendingEventGroup = [];
  const flushPending = () => {
    if (!pendingEventGroup.length) {
      return;
    }
    const toggleId = `event-toggle-${pendingEventGroup[0].id}`;
    if (pendingEventGroup.length === 1) {
      output.push(...pendingEventGroup);
      pendingEventGroup = [];
      return;
    }
    const timestamps = pendingEventGroup
      .map((message) => Date.parse(String(message.ts || message.timestamp || "")))
      .filter((value) => Number.isFinite(value));
    const durationLabel =
      timestamps.length >= 2 ? formatDurationLabel(Math.max(...timestamps) - Math.min(...timestamps)) : "";
    output.push({
      id: toggleId,
      toggleId,
      renderKind: "event-toggle",
      count: pendingEventGroup.length,
      durationLabel
    });
    if (isEventGroupExpanded(toggleId)) {
      output.push(...pendingEventGroup);
    } else {
      output.push(pendingEventGroup[pendingEventGroup.length - 1]);
    }
    pendingEventGroup = [];
  };

  for (const message of renderedMessages.value) {
    if (message.role === "assistant" && message.renderKind === "event") {
      pendingEventGroup.push(message);
      continue;
    }
    flushPending();
    output.push(message);
  }
  flushPending();
  return output;
});
const visibleThreadId = computed(() => String(props.threadId || props.expectedThreadId || "").trim());

function formatTimeSeparator(ts) {
  const date = new Date(ts);
  if (!Number.isFinite(date.getTime())) return "";
  const now = new Date();
  const diffMs = now.getTime() - date.getTime();
  const diffMin = Math.floor(diffMs / 60000);
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  const yesterday = new Date(today.getTime() - 86400000);
  const msgDay = new Date(date.getFullYear(), date.getMonth(), date.getDate());
  const timeStr = date.toLocaleTimeString("zh-CN", { hour: "2-digit", minute: "2-digit", hour12: false });
  if (diffMin < 5) return "";
  if (diffMin < 60) return `${diffMin} 分钟前`;
  if (msgDay.getTime() === today.getTime()) return timeStr;
  if (msgDay.getTime() === yesterday.getTime()) return `昨天 ${timeStr}`;
  const month = date.getMonth() + 1;
  const day = date.getDate();
  if (date.getFullYear() === now.getFullYear()) return `${month}月${day}日 ${timeStr}`;
  return `${date.getFullYear()}年${month}月${day}日 ${timeStr}`;
}

const messagesWithSeparators = computed(() => {
  const result = [];
  const msgs = visibleMessages.value;
  for (let i = 0; i < msgs.length; i++) {
    const msg = msgs[i];
    const ts = Date.parse(String(msg.ts || msg.timestamp || ""));
    if (i === 0) {
      if (Number.isFinite(ts)) {
        const label = formatTimeSeparator(ts);
        if (label) result.push({ _separator: true, id: `sep-${msg.id}`, label });
      }
    } else {
      const prev = msgs[i - 1];
      const prevTs = Date.parse(String(prev.ts || prev.timestamp || ""));
      if (Number.isFinite(ts) && Number.isFinite(prevTs) && ts - prevTs >= 5 * 60 * 1000) {
        const label = formatTimeSeparator(ts);
        if (label) result.push({ _separator: true, id: `sep-${msg.id}`, label });
      }
    }
    result.push(msg);
  }
  return result;
});

const threadHint = computed(() => {
  if (!visibleThreadId.value) {
    return "thread_id: 暂未获取";
  }
  if (props.threadMismatch) {
    return `thread_id 不一致：当前 ${props.threadId} / 目标 ${props.expectedThreadId}`;
  }
  return `thread_id: ${visibleThreadId.value}`;
});

function isNearBottom(element, threshold = BOTTOM_THRESHOLD) {
  if (!element) {
    return true;
  }
  return element.scrollHeight - element.clientHeight - element.scrollTop <= threshold;
}

function resizeComposer(target, { keepBottom = false } = {}) {
  const el = target?.target || target;
  if (!el) {
    return;
  }
  const wasNearBottom = keepBottom && isNearBottom(messageListEl.value);
  el.style.height = "auto";
  el.style.height = `${Math.min(el.scrollHeight, MAX_COMPOSER_HEIGHT)}px`;
  el.style.overflowY = el.scrollHeight > MAX_COMPOSER_HEIGHT ? "auto" : "hidden";
  if (wasNearBottom) {
    scrollToBottom(true);
  }
}

function scrollToBottom(force = false) {
  nextTick(() => {
    const el = messageListEl.value;
    if (!el) {
      return;
    }
    if (!force && !isPinnedToBottom.value) {
      return;
    }

    const applyScroll = () => {
      el.scrollTop = el.scrollHeight;
      isPinnedToBottom.value = true;
    };

    applyScroll();
    if (typeof window !== "undefined" && typeof window.requestAnimationFrame === "function") {
      window.requestAnimationFrame(() => {
        window.requestAnimationFrame(applyScroll);
      });
    }
  });
}

function handleInput(event) {
  emit("update:draft", event.target.value);
  resizeComposer(event, { keepBottom: true });
}

function handleComposerKeydown(event) {
  if (event.isComposing || event.keyCode === 229) {
    return;
  }

  const wantsSubmitShortcut =
    event.key === "Enter" &&
    !event.shiftKey &&
    !event.altKey &&
    !event.ctrlKey &&
    !event.metaKey;

  if (wantsSubmitShortcut) {
    event.preventDefault();
    emit("submit");
  }
}

function handlePrimaryAction() {
  if (isRunning.value) {
    emit("interrupt");
    return;
  }
  emit("submit");
}

async function loadMoreHistory() {
  if (isLoadingMore.value || displayStartIdx.value <= 0) return;
  if (Date.now() < loadMoreCooldownUntil) return;
  const container = messageListEl.value;
  if (!container) return;

  isLoadingMore.value = true;
  const prevScrollHeight = container.scrollHeight;
  const newStart = Math.max(0, displayStartIdx.value - PAGE_SIZE);
  displayStartIdx.value = newStart;

  await nextTick();
  // 保持滚动位置不跳变：新内容插在顶部，补偿 scrollTop
  const delta = container.scrollHeight - prevScrollHeight;
  if (delta > 0) {
    container.scrollTop += delta;
  }
  isLoadingMore.value = false;
  // 冷却 600ms，防止因 DOM 重排引起 scrollTop 短暂归零再次触发
  loadMoreCooldownUntil = Date.now() + 600;
}

function handleStreamScroll(event) {
  const el = event.target;
  isPinnedToBottom.value = isNearBottom(el);
  updateVisibleRange();

  // 触顶加载历史：只在完全到顶（scrollTop === 0）且不在底部锁定、不在冷却期时触发
  if (
    el.scrollTop === 0 &&
    hasMoreHistory.value &&
    !isLoadingMore.value &&
    !isPinnedToBottom.value &&
    Date.now() >= loadMoreCooldownUntil
  ) {
    loadMoreHistory();
  }
}

function updateVisibleRange() {
  const container = messageListEl.value;
  if (!container || !pagedMessages.value.length) return;
  const items = container.querySelectorAll("[data-idx]");
  if (!items.length) {
    firstVisibleIdx.value = 0;
    lastVisibleIdx.value = pagedMessages.value.length - 1;
    return;
  }
  const cTop = container.scrollTop;
  const cBottom = cTop + container.clientHeight;
  let first = -1;
  let last = -1;
  for (const item of items) {
    const rect = item.getBoundingClientRect();
    const cRect = container.getBoundingClientRect();
    const top = rect.top - cRect.top + container.scrollTop;
    const bottom = top + rect.height;
    if (bottom >= cTop && top <= cBottom) {
      const idx = Number(item.getAttribute("data-idx"));
      if (first === -1 || idx < first) first = idx;
      if (idx > last) last = idx;
    }
  }
  if (first !== -1) {
    firstVisibleIdx.value = first;
    lastVisibleIdx.value = last;
  }
}

function openLightboxImage(url, alt = "image") {
  const imageUrl = String(url || "").trim();
  if (!imageUrl) {
    return;
  }
  lightboxImage.value = {
    url: imageUrl,
    alt: String(alt || "image").trim() || "image"
  };
  lightboxScale.value = 1;
  lightboxTranslateX.value = 0;
  lightboxTranslateY.value = 0;
}

function closeLightboxImage() {
  lightboxImage.value = null;
  lightboxScale.value = 1;
  lightboxTranslateX.value = 0;
  lightboxTranslateY.value = 0;
  lightboxGesture.mode = "";
}

function handleMarkdownBodyClick(event) {
  const target = event?.target;
  if (!(target instanceof window.HTMLImageElement)) {
    return;
  }
  openLightboxImage(target.currentSrc || target.src || "", target.alt || "image");
}

function getTouchDistance(touches) {
  if (!touches || touches.length < 2) {
    return 0;
  }
  const [a, b] = touches;
  return Math.hypot(b.clientX - a.clientX, b.clientY - a.clientY);
}

function getTouchMidpoint(touches) {
  if (!touches || touches.length < 2) {
    return { x: 0, y: 0 };
  }
  const [a, b] = touches;
  return {
    x: (a.clientX + b.clientX) / 2,
    y: (a.clientY + b.clientY) / 2
  };
}

function clampLightboxScale(value) {
  return Math.min(4, Math.max(1, value));
}

function handleLightboxTouchStart(event) {
  if (!lightboxImage.value) {
    return;
  }
  if (event.touches.length >= 2) {
    const midpoint = getTouchMidpoint(event.touches);
    lightboxGesture.mode = "pinch";
    lightboxGesture.startScale = lightboxScale.value;
    lightboxGesture.startDistance = getTouchDistance(event.touches);
    lightboxGesture.startTranslateX = lightboxTranslateX.value;
    lightboxGesture.startTranslateY = lightboxTranslateY.value;
    lightboxGesture.startMidpointX = midpoint.x;
    lightboxGesture.startMidpointY = midpoint.y;
    event.preventDefault();
    return;
  }
  if (event.touches.length === 1 && lightboxScale.value > 1) {
    const touch = event.touches[0];
    lightboxGesture.mode = "pan";
    lightboxGesture.startTouchX = touch.clientX;
    lightboxGesture.startTouchY = touch.clientY;
    lightboxGesture.startTranslateX = lightboxTranslateX.value;
    lightboxGesture.startTranslateY = lightboxTranslateY.value;
    event.preventDefault();
  }
}

function handleLightboxTouchMove(event) {
  if (!lightboxImage.value) {
    return;
  }
  if (event.touches.length >= 2) {
    const midpoint = getTouchMidpoint(event.touches);
    const distance = getTouchDistance(event.touches);
    const ratio = lightboxGesture.startDistance > 0 ? distance / lightboxGesture.startDistance : 1;
    lightboxScale.value = clampLightboxScale(lightboxGesture.startScale * ratio);
    lightboxTranslateX.value = lightboxGesture.startTranslateX + (midpoint.x - lightboxGesture.startMidpointX);
    lightboxTranslateY.value = lightboxGesture.startTranslateY + (midpoint.y - lightboxGesture.startMidpointY);
    event.preventDefault();
    return;
  }
  if (event.touches.length === 1 && lightboxScale.value > 1) {
    const touch = event.touches[0];
    lightboxTranslateX.value = lightboxGesture.startTranslateX + (touch.clientX - lightboxGesture.startTouchX);
    lightboxTranslateY.value = lightboxGesture.startTranslateY + (touch.clientY - lightboxGesture.startTouchY);
    event.preventDefault();
  }
}

function handleLightboxTouchEnd() {
  if (lightboxScale.value <= 1) {
    lightboxScale.value = 1;
    lightboxTranslateX.value = 0;
    lightboxTranslateY.value = 0;
  }
  lightboxGesture.mode = "";
}

function handleComposerFocus() {
  if (!isNearBottom(messageListEl.value)) {
    return;
  }
  scrollToBottom(true);
}

function syncViewportMetrics() {
  if (typeof window === "undefined") {
    return;
  }

  const viewport = window.visualViewport;
  const height = viewport?.height ? Math.round(viewport.height) : window.innerHeight;
  const inset = viewport
    ? Math.max(0, Math.round(window.innerHeight - viewport.height - viewport.offsetTop))
    : 0;

  viewportHeight.value = height;
  keyboardInset.value = inset;
}

function handleViewportChange() {
  syncViewportMetrics();
  resizeComposer(composerEl.value, { keepBottom: true });
  if (isPinnedToBottom.value) {
    scrollToBottom(true);
  }
}

function handleWindowResize() {
  syncViewportMetrics();
  resizeComposer(composerEl.value, { keepBottom: true });
}

function handleWindowKeydown(event) {
  if (event.key === "Escape" && lightboxImage.value) {
    closeLightboxImage();
  }
}

watch(
  () => props.messages.map((message) => `${message.id}:${message.text?.length || 0}`).join("|"),
  () => {
    nextTick(updateVisibleRange);
    scrollToBottom(false);
  },
  { flush: "post" }
);

watch(
  () => `${props.sessionKey}::${props.openToken}`,
  () => {
    isPinnedToBottom.value = true;
    expandedEventGroups.value = new Set();
    renderedHtmlCache.clear();
    firstVisibleIdx.value = 0;
    lastVisibleIdx.value = -1;
    // 切换会话时：初始只展示最后 PAGE_SIZE 条，其余通过上滑加载
    const total = props.messages.length;
    displayStartIdx.value = Math.max(0, total - PAGE_SIZE);
    scrollToBottom(true);
  },
  { flush: "post", immediate: true }
);

// 新消息流入时：pagedMessages 已通过 displayStartIdx 截断，只要新消息在截断范围之后
// 就已经自然显示，无需主动修改 displayStartIdx（会引发不必要的 DOM 重排和滚动抖动）。
// 唯一需要处理的情况：初始加载后 props.messages 从 0 增长到有内容时，确保 displayStartIdx 不越界。
watch(
  () => props.messages.length,
  (newLen) => {
    // displayStartIdx 不能超过总长度
    if (displayStartIdx.value > 0 && displayStartIdx.value >= newLen) {
      displayStartIdx.value = Math.max(0, newLen - PAGE_SIZE);
    }
  }
);

watch(
  () => props.draft,
  () => {
    nextTick(() => resizeComposer(composerEl.value, { keepBottom: true }));
  },
  { flush: "post", immediate: true }
);

onMounted(() => {
  if (typeof window !== "undefined") {
    isTouchDevice.value =
      window.matchMedia?.("(pointer: coarse)").matches ||
      navigator.maxTouchPoints > 0;
    window.addEventListener("resize", handleWindowResize, { passive: true });
    window.addEventListener("keydown", handleWindowKeydown);
    window.visualViewport?.addEventListener("resize", handleViewportChange, { passive: true });
    window.visualViewport?.addEventListener("scroll", handleViewportChange, { passive: true });
  }
  syncViewportMetrics();
  scrollToBottom(true);
  nextTick(updateVisibleRange);
  resizeComposer(composerEl.value, { keepBottom: true });
});

onBeforeUnmount(() => {
  if (typeof window !== "undefined") {
    window.removeEventListener("resize", handleWindowResize);
    window.removeEventListener("keydown", handleWindowKeydown);
    window.visualViewport?.removeEventListener("resize", handleViewportChange);
    window.visualViewport?.removeEventListener("scroll", handleViewportChange);
  }
});
</script>

<template>
  <section class="chat-shell" :style="chatShellStyle">
    <!-- V2 Header (AppBarV2 style) -->
    <header class="v2-header">
      <button class="v2-header-back" aria-label="返回会话列表" @click="emit('back')">
        <span aria-hidden="true">&#8592;</span>
      </button>
      <div class="v2-header-center">
        <p class="v2-header-meta">{{ workspaceName.toUpperCase() }}</p>
        <h1 class="v2-header-title">{{ title }}</h1>
      </div>
      <div class="v2-header-status-pill" :class="'v2-status-' + statusIndicator.state">
        <span class="v2-status-dot"></span>
        <span>{{ isRunning ? "RUNNING" : statusIndicator.state === 'idle' ? '' : statusIndicator.state.toUpperCase().slice(0, 7) }}</span>
      </div>
      <button class="v2-header-more" aria-label="更多">
        <span class="v2-header-dot"></span>
        <span class="v2-header-dot"></span>
        <span class="v2-header-dot"></span>
      </button>
    </header>

    <!-- V2 Timeline -->
    <main class="chat-main">
      <section ref="messageListEl" class="timeline-stream" @scroll="handleStreamScroll">
        <!-- 历史消息加载指示器 -->
        <div v-if="isLoadingMore" class="tl-load-more tl-load-more--loading">加载中…</div>
        <div v-else-if="hasMoreHistory" class="tl-load-more">上滑加载更多历史</div>

        <!-- Single absolute spine line -->
        <div class="tl-spine"></div>

        <template v-for="item in messagesWithSeparators" :key="item.id">
          <div v-if="item._separator" class="time-separator">{{ item.label }}</div>
          <article v-else class="tl-item" :class="[item.role, 'tl-' + item.renderKind]" :data-idx="item._idx">
            <!-- Timeline node -->
            <div class="tl-node" :style="{ '--node-color': getEventBadgeInfo(item).fg }">
              <span class="tl-node-dot"></span>
            </div>

            <!-- Content column -->
            <div class="tl-body">
              <!-- Meta row: badge + time -->
              <div class="tl-meta">
                <span class="tl-badge" :style="{ color: getEventBadgeInfo(item).fg, background: getEventBadgeInfo(item).bg }">{{ getEventBadgeInfo(item).label }}</span>
                <span class="tl-time">{{ formatEventTime(item.ts || item.timestamp) }}</span>
              </div>

              <!-- Event toggle -->
              <button
                v-if="item.renderKind === 'event-toggle'"
                type="button"
                class="tl-event-toggle"
                @click="toggleEventGroup(item.toggleId)"
              >
                {{ isEventGroupExpanded(item.toggleId) ? "收起执行过程" : "查看执行过程" }}
                <span class="tl-event-toggle-meta">
                  ({{ item.count }} 条<span v-if="item.durationLabel">, {{ item.durationLabel }}</span>)
                </span>
              </button>

              <!-- Image -->
              <div v-else-if="item.renderKind === 'image'" class="tl-card-image">
                <div class="tl-image-wrap">
                  <img
                    class="tl-image"
                    :src="item.imageUrl"
                    :alt="item.imageAlt"
                    loading="lazy"
                    decoding="async"
                    @click="openLightboxImage(item.imageUrl, item.imageAlt)"
                  />
                </div>
              </div>

              <!-- Event card (ActionCard style) -->
              <div v-else-if="item.renderKind === 'event'" class="tl-card-event">
                <div class="action-header" v-if="item.eventDetails?.find(d => d.label === '命令')">
                  <span class="action-dollar">$</span>
                  <span class="action-cmd">{{ item.eventDetails.find(d => d.label === '命令').value }}</span>
                </div>
                <div class="action-output">
                  <p class="action-output-text">{{ item.eventCompactText }}</p>
                  <dl v-if="item.eventDetails.length && !item.eventDetails.find(d => d.label === '命令')" class="action-details">
                    <template v-for="detail in item.eventDetails" :key="`${item.id}-${detail.label}`">
                      <dt>{{ detail.label }}</dt>
                      <dd>{{ detail.value }}</dd>
                    </template>
                  </dl>
                  <p v-if="item.eventSummary && item.eventSummary !== item.eventCompactText" class="action-result">→ {{ item.eventSummary }}</p>
                </div>
              </div>

              <!-- User message (dark block) -->
              <div v-else-if="item.role === 'user'" class="tl-card-user">
                <div class="tl-user-text">{{ item.displayText }}</div>
              </div>

              <!-- Assistant: thought (plain gray, no card) -->
              <div v-else-if="item.assistantDisplay === 'thought'" class="tl-card-thought">
                <p class="tl-thought-text">{{ item.displayText }}</p>
              </div>

              <!-- Assistant: reply / markdown (white card) -->
              <div v-else-if="item.displayText && item.renderedHtml" class="tl-card-reply">
                <div class="tl-reply-text markdown-body" v-html="item.renderedHtml" @click="handleMarkdownBodyClick"></div>
                <section v-if="item.fileMentions?.length" class="tl-files">
                  <p class="tl-files-title">{{ item.fileMentions.length }} 个文件已修改</p>
                  <div class="tl-files-list">
                    <div v-for="file in item.fileMentions" :key="`${item.id}-${file.path}`" class="tl-files-item">
                      {{ file.label }}
                    </div>
                  </div>
                </section>
              </div>

              <!-- Placeholder for out-of-range -->
              <div v-else-if="item.displayText && !item.renderedHtml" class="tl-card-placeholder">
                <p class="tl-placeholder-text">{{ item.displayText.slice(0, 120) }}</p>
              </div>
            </div>

            <details v-if="showProcessDetails && item.hasProcessDetails" class="tl-process">
              <summary>{{ item.processSummary }}</summary>
              <pre class="tl-process-text">{{ item.processText }}</pre>
            </details>
          </article>
        </template>

        <div v-if="visibleMessages.length === 0" class="tl-empty">
          <div class="tl-empty-icon">&#10024;</div>
          <p>向 Codex 发送指令开始</p>
        </div>
      </section>

      <!-- Status indicator -->
      <div v-if="statusIndicator.state !== 'idle'" class="v2-status-bar" :class="'status-' + statusIndicator.state">
        <span class="v2-status-icon" aria-hidden="true">{{ statusIndicator.icon }}</span>
        <span class="v2-status-label">{{ statusIndicator.label }}</span>
      </div>

      <!-- Slash command bar + Composer -->
      <div class="v2-composer">
        <div v-if="!isRunning && !loading" class="slash-bar">
          <button
            v-for="cmd in slashCommands"
            :key="cmd.label"
            type="button"
            class="slash-pill"
            @click="insertSlashCommand(cmd.text)"
          >{{ cmd.label }}</button>
        </div>
        <div class="v2-composer-row">
          <textarea
            ref="composerEl"
            :value="draft"
            class="v2-composer-input"
            rows="1"
            placeholder="继续对话或输入 / 调出指令…"
            :disabled="loading"
            enterkeyhint="send"
            @input="handleInput"
            @focus="handleComposerFocus"
            @keydown="handleComposerKeydown"
          ></textarea>
          <button
            v-if="isRunning"
            class="v2-send-btn v2-send-interrupt"
            type="button"
            aria-label="中断当前流程"
            :disabled="!canPrimaryAction"
            @click="handlePrimaryAction"
          >
            &#9632;
          </button>
          <button
            v-else
            class="v2-send-btn"
            type="button"
            aria-label="发送消息"
            :disabled="!canPrimaryAction"
            @click="handlePrimaryAction"
          >
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M22 2 11 13"/><path d="M22 2 15 22 11 13 2 9z"/></svg>
          </button>
        </div>
      </div>
    </main>

    <!-- Lightbox -->
    <div
      v-if="lightboxImage"
      class="image-lightbox"
      @click="closeLightboxImage"
      @touchstart="handleLightboxTouchStart"
      @touchmove="handleLightboxTouchMove"
      @touchend="handleLightboxTouchEnd"
      @touchcancel="handleLightboxTouchEnd"
    >
      <button class="image-lightbox-close" type="button" aria-label="关闭预览" @click.stop="closeLightboxImage">&times;</button>
      <img
        class="image-lightbox-media"
        :src="lightboxImage.url"
        :alt="lightboxImage.alt"
        :style="lightboxImageStyle"
        loading="eager"
        decoding="async"
        @click.stop
      >
    </div>
  </section>
</template>

<style scoped>
/* ===== V2 Shell ===== */
.chat-shell {
  height: var(--chat-vh, 100dvh);
  min-height: var(--chat-vh, 100dvh);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: var(--chat-bg);
  color: var(--text);
}

/* ===== V2 Header (AppBarV2) ===== */
.v2-header {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: calc(env(safe-area-inset-top) + 4px) 16px 8px;
  background: var(--panel);
  position: relative;
}

.v2-header-back {
  width: 32px;
  height: 32px;
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
  color: var(--text);
  font-size: 22px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  cursor: pointer;
}

.v2-header-center {
  flex: 1;
  min-width: 0;
}

.v2-header-meta {
  margin: 0;
  font-family: var(--font-mono);
  font-size: 10px;
  color: var(--text-ter);
  letter-spacing: 0.4px;
  text-transform: uppercase;
  margin-bottom: 1px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.v2-header-title {
  margin: 0;
  font-size: 17px;
  font-weight: 600;
  color: var(--text);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  letter-spacing: -0.3px;
  line-height: 1.25;
}

.v2-header-status-pill {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 4px 8px;
  background: var(--accent-soft);
  border-radius: 4px;
  font-family: var(--font-mono);
  font-size: 10px;
  font-weight: 600;
  color: var(--ev-approval);
  letter-spacing: 0.3px;
}

.v2-status-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--ev-approval);
}

.v2-header-status-pill.v2-status-streaming,
.v2-header-status-pill.v2-status-running {
  color: var(--st-running);
}

.v2-header-status-pill.v2-status-streaming .v2-status-dot,
.v2-header-status-pill.v2-status-running .v2-status-dot {
  background: var(--st-running);
  animation: dot-pulse 1.6s ease-out infinite;
}

.v2-header-status-pill.v2-status-completed {
  color: var(--st-success);
}

.v2-header-status-pill.v2-status-completed .v2-status-dot {
  background: var(--st-success);
}

.v2-header-status-pill.v2-status-error,
.v2-header-status-pill.v2-status-interrupted {
  color: var(--st-error);
}

.v2-header-status-pill.v2-status-error .v2-status-dot,
.v2-header-status-pill.v2-status-interrupted .v2-status-dot {
  background: var(--st-error);
}

@keyframes dot-pulse {
  0%   { box-shadow: 0 0 0 0 currentColor; }
  70%  { box-shadow: 0 0 0 5px rgba(0,0,0,0); }
  100% { box-shadow: 0 0 0 0 rgba(0,0,0,0); }
}

.v2-header-more {
  width: 32px;
  height: 32px;
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 3px;
  flex-shrink: 0;
  cursor: pointer;
}

.v2-header-dot {
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: var(--text);
  opacity: 0.5;
}

/* ===== Chat Main ===== */
.chat-main {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* ===== Timeline Stream ===== */
.timeline-stream {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  gap: 0;
  padding: 10px 12px calc(16px + env(safe-area-inset-bottom));
  overflow-y: auto;
  overflow-x: hidden;
  overscroll-behavior: contain;
  -webkit-overflow-scrolling: touch;
  position: relative;
  background: var(--chat-bg);
}

/* ===== Time Separator ===== */
.time-separator {
  text-align: center;
  padding: 10px 0 6px;
  font-size: 12px;
  color: var(--text-ter);
  user-select: none;
}

/* ===== Timeline Item ===== */
.tl-item {
  display: flex;
  gap: 12px;
  padding: 6px 14px;
  position: relative;
  align-items: flex-start;
}

/* Spine line — single absolute line on the stream */
.tl-spine {
  position: absolute;
  left: 27px;
  top: 16px;
  bottom: 20px;
  width: 1px;
  background: var(--line-strong);
  z-index: 0;
}

/* Timeline node — hollow circle with colored inner dot */
.tl-node {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  margin-top: 6px;
  background: var(--chat-bg);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  z-index: 1;
}

.tl-node-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--node-color, var(--ev-action));
}

/* ===== Timeline Body (content column) ===== */
.tl-body {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

/* Meta row: badge + time */
.tl-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}

.tl-badge {
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

.tl-time {
  font-family: var(--font-mono);
  font-size: 10px;
  color: var(--text-ter);
}

/* ===== User Card (right-side bubble) ===== */
.tl-item.user {
  flex-direction: row-reverse;
}

.tl-item.user .tl-body {
  align-items: flex-end;
}

.tl-item.user .tl-meta {
  flex-direction: row-reverse;
}

.tl-card-user {
  background: var(--user-bg, #95ec69);
  color: var(--user-text, #181818);
  padding: 10px 14px;
  border-radius: 8px;
  max-width: 85%;
  align-self: flex-end;
}

.tl-user-text {
  font-size: 15px;
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
  color: var(--user-text, #181818);
}

/* ===== Thought (plain gray text, no card) ===== */
.tl-card-thought {
  padding-left: 2px;
}

.tl-thought-text {
  margin: 0;
  font-size: 13px;
  line-height: 1.5;
  color: var(--text-soft);
  white-space: pre-wrap;
  word-break: break-word;
}

/* ===== Reply Card (white card) ===== */
.tl-card-reply {
  background: var(--panel);
  border: 1px solid var(--line);
  border-radius: 8px;
  padding: 10px 14px;
}

.tl-reply-text {
  width: 100%;
  margin: 0;
  white-space: normal;
  overflow-wrap: break-word;
  word-break: break-word;
  font-size: 14px;
  line-height: 1.55;
  color: var(--text);
}

/* ===== Event Card (ActionCard — dark code block) ===== */
.tl-card-event {
  background: var(--code-bg);
  border-radius: 6px;
  overflow: hidden;
  color: var(--code-text);
}

.action-header {
  padding: 7px 10px;
  border-bottom: 1px solid rgba(255,255,255,0.06);
  font-family: var(--font-mono);
  font-size: 11px;
  color: var(--code-text);
  display: flex;
  align-items: center;
  gap: 6px;
}

.action-dollar {
  color: var(--accent);
}

.action-cmd {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.action-output {
  padding: 8px 10px;
  font-family: var(--font-mono);
  font-size: 10.5px;
  color: var(--code-dim);
  line-height: 1.55;
}

.action-output-text {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
}

.action-details {
  margin: 4px 0 0;
  display: grid;
  grid-template-columns: auto 1fr;
  column-gap: 8px;
  row-gap: 4px;
}

.action-details dt {
  margin: 0;
  font-size: 11px;
  color: var(--code-dim);
  white-space: nowrap;
}

.action-details dd {
  margin: 0;
  font-size: 11px;
  color: var(--code-text);
  overflow-wrap: anywhere;
}

.action-result {
  margin: 4px 0 0;
  color: var(--accent);
}

/* ===== Image Card ===== */
.tl-card-image {
}

/* ===== Placeholder Card ===== */
.tl-card-placeholder {
}

.tl-placeholder-text {
  margin: 0;
  font-size: 13px;
  line-height: 1.4;
  color: var(--text-ter);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}

.tl-image-wrap {
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid var(--line);
}

.tl-image {
  display: block;
  width: 156px;
  max-width: 156px;
  max-height: 156px;
  height: auto;
  object-fit: cover;
  cursor: zoom-in;
}

/* ===== File Mentions ===== */
.tl-files {
  width: 100%;
  margin-top: 4px;
  padding: 8px 10px;
  border: 1px solid var(--line);
  border-radius: 6px;
  background: var(--chat-bg);
}

.tl-files-title {
  margin: 0 0 6px;
  font-size: 12px;
  color: var(--text-sec);
}

.tl-files-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.tl-files-item {
  padding: 6px 8px;
  border-radius: 4px;
  background: var(--accent-soft);
  color: var(--accent);
  font-family: var(--font-mono);
  font-size: 12px;
  overflow-wrap: anywhere;
  pointer-events: none;
  user-select: text;
}

/* ===== Event Toggle ===== */
.tl-event-toggle {
  margin: 0;
  padding: 4px 10px;
  font-size: 11px;
  line-height: 1.4;
  color: var(--accent);
  background: var(--accent-soft);
  border: 0;
  border-radius: 6px;
  cursor: pointer;
  text-align: left;
}

.tl-event-toggle-meta {
  color: var(--text-ter);
  margin-left: 2px;
}

/* ===== Process Details ===== */
.tl-process {
  width: 100%;
  border: 1px solid var(--line);
  border-radius: 6px;
  background: var(--chat-bg);
  overflow: hidden;
  margin-top: 4px;
}

.tl-process summary {
  list-style: none;
  padding: 8px 12px;
  color: var(--text-ter);
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.tl-process summary::-webkit-details-marker {
  display: none;
}

.tl-process-text {
  margin: 0;
  padding: 0 12px 12px;
  color: var(--text-ter);
  font-size: 12px;
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
}

/* ===== Load More Indicator ===== */
.tl-load-more {
  text-align: center;
  padding: 8px 0;
  font-size: 12px;
  color: var(--text-ter);
  user-select: none;
}

.tl-load-more--loading {
  opacity: 0.6;
}

/* ===== Empty State ===== */
.tl-empty {
  margin: auto 0;
  padding: 48px 20px;
  text-align: center;
}

.tl-empty-icon {
  font-size: 36px;
  margin-bottom: 12px;
  opacity: 0.4;
}

.tl-empty p {
  margin: 0;
  font-size: 14px;
  color: var(--text-ter);
}

/* ===== V2 Status Bar ===== */
.v2-status-bar {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  margin: 0 14px 4px;
  padding: 4px 0;
  font-size: 12px;
  line-height: 1.4;
}

.v2-status-icon {
  flex: 0 0 auto;
  font-size: 13px;
}

.v2-status-label {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.v2-status-bar.status-streaming {
  color: var(--text-ter);
}

.v2-status-bar.status-connecting,
.v2-status-bar.status-sending,
.v2-status-bar.status-waiting {
  color: var(--st-running);
  animation: v2-pulse 1.6s ease-in-out infinite;
}

.v2-status-bar.status-interrupted {
  color: var(--st-error);
}

.v2-status-bar.status-error {
  color: var(--st-error);
}

.v2-status-bar.status-completed {
  color: var(--st-success);
}

.v2-status-bar.status-info {
  color: var(--text-ter);
}

@keyframes v2-pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* ===== V2 Composer ===== */
.v2-composer {
  flex-shrink: 0;
  background: var(--panel);
  border-top: 0.5px solid var(--line);
  padding: 8px 12px calc(10px + env(safe-area-inset-bottom) + clamp(0px, var(--chat-keyboard-inset, 0px), 24px));
}

.slash-bar {
  display: flex;
  gap: 6px;
  margin-bottom: 8px;
  overflow-x: auto;
  scrollbar-width: none;
  -ms-overflow-style: none;
}

.slash-bar::-webkit-scrollbar {
  display: none;
}

.slash-pill {
  flex-shrink: 0;
  padding: 5px 10px;
  border: 0.5px solid var(--line);
  border-radius: 4px;
  background: var(--chat-bg);
  color: var(--text-sec);
  font-family: var(--font-mono);
  font-size: 11px;
  cursor: pointer;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

.slash-pill:active {
  background: var(--accent-soft);
}

.v2-composer-row {
  display: flex;
  align-items: flex-end;
  gap: 8px;
}

.v2-composer-input {
  flex: 1;
  min-height: 38px;
  max-height: 160px;
  padding: 9px 12px;
  border: none;
  border-radius: 8px;
  background: var(--chat-bg);
  color: var(--text);
  font-size: 16px; /* ≥16px 防止 iOS Safari 自动缩放页面 */
  line-height: 1.5;
  resize: none;
  box-shadow: none;
  outline: none;
  -webkit-appearance: none;
}

.v2-composer-input::placeholder {
  color: var(--text-placeholder);
}

.v2-composer-input:focus {
  outline: none;
}

.v2-composer-input:disabled {
  color: var(--text-ter);
}

/* V2 send button */
.v2-send-btn {
  width: 38px;
  height: 38px;
  min-width: 38px;
  min-height: 38px;
  padding: 0;
  border: 0;
  border-radius: 8px;
  background: var(--accent);
  color: #ffffff;
  font-size: 16px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
  transition: opacity 0.14s ease;
}

.v2-send-btn:disabled {
  opacity: 0.35;
}

.v2-send-interrupt {
  background: var(--st-running);
}

/* ===== Lightbox ===== */
.image-lightbox {
  position: fixed;
  inset: 0;
  z-index: 80;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  background: rgba(0, 0, 0, 0.85);
  touch-action: none;
  overscroll-behavior: contain;
}

.image-lightbox-media {
  max-width: min(100vw - 32px, 960px);
  max-height: min(100vh - 48px, 88vh);
  border-radius: 8px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
  background: var(--panel);
  transform-origin: center center;
  user-select: none;
  -webkit-user-drag: none;
  touch-action: none;
  will-change: transform;
}

.image-lightbox-close {
  position: absolute;
  top: calc(env(safe-area-inset-top) + 12px);
  right: 14px;
  width: 36px;
  height: 36px;
  border: 0;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.9);
  color: var(--text);
  font-size: 24px;
  line-height: 1;
  cursor: pointer;
}

/* ===== Markdown Body (V2) ===== */
.markdown-body :deep(h1),
.markdown-body :deep(h2),
.markdown-body :deep(h3),
.markdown-body :deep(h4),
.markdown-body :deep(h5),
.markdown-body :deep(h6) {
  margin: 0 0 6px;
  line-height: 1.35;
  font-weight: 700;
}

.markdown-body :deep(h1) { font-size: 20px; }
.markdown-body :deep(h2) { font-size: 18px; }
.markdown-body :deep(h3) { font-size: 16px; }
.markdown-body :deep(h4),
.markdown-body :deep(h5),
.markdown-body :deep(h6) { font-size: 15px; }

.markdown-body :deep(p) {
  margin: 0 0 4px;
}

.markdown-body :deep(p:last-child) {
  margin-bottom: 0;
}

.markdown-body :deep(code) {
  padding: 1px 6px;
  border-radius: 4px;
  background: rgba(0, 0, 0, 0.06);
  font-family: var(--font-mono);
  font-size: 0.9em;
}

.markdown-body :deep(pre) {
  margin: 0 0 6px;
  padding: 10px 11px;
  border-radius: 6px;
  border: 1px solid var(--line-strong);
  background: var(--code-bg);
  color: var(--code-text);
  overflow-x: auto;
  max-width: 100%;
  box-sizing: border-box;
}

.markdown-body :deep(pre code) {
  padding: 0;
  border-radius: 0;
  background: transparent;
  font-family: var(--font-mono);
  font-size: 13px;
  line-height: 1.6;
  white-space: pre;
  max-width: 100%;
  color: var(--code-text);
}

.markdown-body :deep(pre.code-block) {
  position: relative;
  padding-top: 28px;
}

.markdown-body :deep(pre.code-block .code-lang) {
  position: absolute;
  top: 8px;
  right: 10px;
  padding: 1px 7px;
  border-radius: 4px;
  font-size: 10px;
  line-height: 1.3;
  text-transform: lowercase;
  letter-spacing: 0.01em;
  border: 1px solid rgba(255,255,255,0.1);
  background: rgba(255,255,255,0.08);
  color: var(--code-dim);
}

.markdown-body :deep(pre.code-block.family-sql) {
  border-color: #3d3825;
}

.markdown-body :deep(pre.code-block.family-shell) {
  border-color: #2a3828;
}

.markdown-body :deep(pre.code-block.family-config) {
  border-color: #283040;
}

.markdown-body :deep(pre.code-block .hljs) {
  color: var(--code-text);
  background: transparent;
}

.markdown-body :deep(.hljs-comment),
.markdown-body :deep(.hljs-quote) {
  color: var(--code-dim);
  font-style: italic;
}

.markdown-body :deep(.hljs-keyword),
.markdown-body :deep(.hljs-selector-tag),
.markdown-body :deep(.hljs-literal),
.markdown-body :deep(.hljs-built_in),
.markdown-body :deep(.hljs-type) {
  color: #e06c75;
  font-weight: 600;
}

.markdown-body :deep(.hljs-string),
.markdown-body :deep(.hljs-title),
.markdown-body :deep(.hljs-name),
.markdown-body :deep(.hljs-section),
.markdown-body :deep(.hljs-attribute),
.markdown-body :deep(.hljs-symbol),
.markdown-body :deep(.hljs-bullet) {
  color: #98c379;
}

.markdown-body :deep(.hljs-number),
.markdown-body :deep(.hljs-meta),
.markdown-body :deep(.hljs-variable),
.markdown-body :deep(.hljs-template-variable),
.markdown-body :deep(.hljs-params) {
  color: #61afef;
}

.markdown-body :deep(blockquote) {
  margin: 0 0 6px;
  padding: 6px 10px;
  border-left: 3px solid var(--accent);
  background: var(--accent-soft);
  border-radius: 0 6px 6px 0;
  color: var(--text-sec);
}

.markdown-body :deep(blockquote p) {
  margin: 0;
  line-height: 1.4;
}

.markdown-body :deep(ul),
.markdown-body :deep(ol) {
  margin: 0 0 6px;
  padding-left: 18px;
}

.markdown-body :deep(li) {
  margin-bottom: 4px;
}

.markdown-body :deep(a) {
  color: var(--accent);
  text-decoration: none;
  text-underline-offset: 2px;
  overflow-wrap: anywhere;
  word-break: break-word;
}

.markdown-body :deep(.local-file-ref) {
  display: inline-block;
  padding: 1px 7px;
  border-radius: 4px;
  border: 1px solid var(--accent-edge);
  background: var(--accent-soft);
  color: var(--accent);
  text-decoration: none;
  overflow-wrap: anywhere;
  word-break: break-word;
  pointer-events: none;
  cursor: default;
  user-select: text;
}

.markdown-body :deep(img) {
  display: block;
  width: 156px;
  max-width: 156px;
  max-height: 156px;
  height: auto;
  object-fit: cover;
  margin: 6px 0;
  border-radius: 6px;
  border: 1px solid var(--line);
  background: var(--chat-bg);
  cursor: zoom-in;
}

/* ===== 手机端优化 ===== */
@media (max-width: 640px) {
  /* 隐藏时间线节点和脊线，让消息全宽展示 */
  .tl-spine { display: none; }
  .tl-node  { display: none; }

  /* 减少左边距，内容区宽度最大化 */
  .tl-item {
    gap: 0;
    padding: 4px 10px;
  }

  /* 用户消息气泡不超过屏幕宽度 90% */
  .tl-card-user {
    max-width: 90%;
  }

  /* 缩小 header，节省垂直空间 */
  .v2-header {
    padding-top: calc(env(safe-area-inset-top) + 2px);
    padding-bottom: 6px;
  }
  .v2-header-title {
    font-size: 15px;
  }

  /* 消息时间 badge 在小屏缩小字号 */
  .tl-badge { font-size: 9px; }
  .tl-time  { font-size: 9px; }

  /* 代码块横向可滚动，不强制换行 */
  .markdown-body :deep(pre) {
    font-size: 12px;
  }

  /* slash 指令栏在小屏隐藏，减少干扰 */
  .slash-bar { display: none; }
}
</style>
