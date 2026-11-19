<script setup>
import { computed, nextTick, ref } from 'vue'
import {
  BookOpen, Braces, Brackets, Check, ChevronDown, CircleAlert, CodeXml,
  Copy, CornerDownRight, FileText, GitBranch, Layers3, Plus, RotateCcw,
  SlidersHorizontal, Terminal, UserRound, X,
} from 'lucide-vue-next'

const example = [
  '你好，{USER.a}！',
  '',
  '{#if USER.a == USER.b}',
  '  欢迎回来，{USER.a}。',
  '{#elseif SYS.a == SYS.b}',
  '  当前系统状态为 {SYS.a}。',
  '{#else}',
  '  请检查你的设置，当前值为 {a}。',
  '{#endif}',
  '',
  '谢谢你的使用！',
].join('\n')

const groups = [
  { label: '系统变量', icon: SlidersHorizontal, variables: ['SYS.a', 'SYS.b'] },
  { label: '用户变量', icon: UserRound, variables: ['USER.a', 'USER.b'] },
  { label: '临时变量', icon: Layers3, variables: ['a', 'b'] },
]

const snippets = [
  { name: 'if 条件块', tag: '#if … #endif', text: '{#if USER.a == USER.b}\n  \n{#endif}', select: 'USER.a' },
  { name: 'elseif 分支', tag: '#elseif', text: '{#elseif USER.a == USER.b}\n  ', select: 'USER.a' },
  { name: 'else 分支', tag: '#else', text: '{#else}\n  ' },
  { name: 'endif 结束', tag: '#endif', text: '{#endif}' },
]

const source = ref(example)
const editor = ref(null)
const highlight = ref(null)
const gutter = ref(null)
const help = ref(null)
const position = ref({ line: 1, column: 1 })
const selection = ref({ start: 0, end: 0 })
const feedback = ref('')
const collapsed = ref({})
let feedbackTimer

const lines = computed(() => source.value.split('\n'))
const highlighted = computed(() => source.value.split(/(\{#[^{}\n]*\}|\{[\w.]+\})/g).flatMap((part, index) => {
  if (index % 2 === 0) return [{ text: part }]
  if (!part.startsWith('{#')) return [{ text: part, type: 'variable' }]
  return part.split(/(\b(?:SYS|USER)\.[\w]+\b|===|!==|==|!=|>=|<=|&&|\|\||[<>])/g)
    .map((text, index) => ({ text, type: index % 2 === 0 ? 'directive' : /^\w/.test(text) ? 'variable' : 'operator' }))
}))

const issue = computed(() => {
  const blocks = []
  for (const [index, line] of lines.value.entries()) {
    for (const [tag] of line.matchAll(/\{#[^{}]*(?:\}|(?=\{|$))/g)) {
      const lineNumber = index + 1
      const directive = tag.match(/^\{#(if|elseif|else|endif)(?=\s|\})([^{}]*)\}$/)
      if (!directive) return `第 ${lineNumber} 行的条件标签格式有误`
      const [, keyword, expression] = directive
      if (['if', 'elseif'].includes(keyword) && !expression.trim()) return `第 ${lineNumber} 行缺少条件表达式`
      if (['else', 'endif'].includes(keyword) && expression.trim()) return `第 ${lineNumber} 行的 #${keyword} 不应带条件`
      if (keyword === 'if') {
        blocks.push({ lineNumber, hasElse: false })
        continue
      }
      const block = blocks.at(-1)
      if (!block) return `第 ${lineNumber} 行的 #${keyword} 缺少对应的 #if`
      if (keyword === 'endif') blocks.pop()
      else if (block.hasElse) return `第 ${lineNumber} 行的 #${keyword} 不能位于 #else 之后`
      else if (keyword === 'else') block.hasElse = true
    }
  }
  return blocks.length ? `第 ${blocks.at(-1).lineNumber} 行的 #if 缺少 #endif` : ''
})

function rememberSelection() {
  const { value, selectionStart, selectionEnd } = editor.value
  selection.value = { start: selectionStart, end: selectionEnd }
  const before = value.slice(0, selectionStart).split('\n')
  position.value = { line: before.length, column: before.at(-1).length + 1 }
}

function syncScroll() {
  highlight.value.scrollTop = editor.value.scrollTop
  highlight.value.scrollLeft = editor.value.scrollLeft
  gutter.value.scrollTop = editor.value.scrollTop
}

async function insert(text, offset = text.length, length = 0) {
  const { start, end } = selection.value
  const input = editor.value
  input.focus()
  input.setSelectionRange(start, end)
  // The native editing command preserves the textarea's undo history.
  if (!document.execCommand('insertText', false, text)) {
    input.setRangeText(text, start, end, 'end')
  }
  source.value = input.value
  await nextTick()
  input.setSelectionRange(start + offset, start + offset + length)
  rememberSelection()
  syncScroll()
}

function insertVariable(variable) {
  const { start, end } = selection.value
  const before = source.value.slice(0, start).split('\n').at(-1)
  const after = source.value.slice(end).split('\n')[0]
  const insideCondition = /\{#(?:if|elseif)\s+[^{}]*$/.test(before) && !/[{}]/.test(after.split('}')[0])
  const insidePlaceholder = /\{[\w.]*$/.test(before) && /^[\w.]*\}/.test(after)
  insert(insideCondition || insidePlaceholder ? variable : `{${variable}}`)
}

function insertSnippet(snippet) {
  const { start, end } = selection.value
  const before = source.value.slice(0, start).split('\n').at(-1)
  const after = source.value.slice(end).split('\n')[0]
  const prefix = before.trim() ? '\n' : ''
  const suffix = after.trim() ? '\n' : ''
  const offset = snippet.select ? snippet.text.indexOf(snippet.select) : snippet.text.length
  insert(prefix + snippet.text + suffix, prefix.length + offset, snippet.select?.length)
}

function notify(message) {
  feedback.value = message
  clearTimeout(feedbackTimer)
  feedbackTimer = setTimeout(() => { feedback.value = '' }, 2200)
}

function loadExample() {
  selection.value = { start: 0, end: source.value.length }
  insert(example, 0)
  notify('已载入示例')
}

async function copySource() {
  try {
    await navigator.clipboard.writeText(source.value)
  } catch {
    editor.value.focus()
    editor.value.select()
    const copied = document.execCommand('copy')
    editor.value.setSelectionRange(selection.value.start, selection.value.end)
    if (!copied) {
      notify('复制失败，请手动选择文本')
      return
    }
  }
  notify('已复制模板')
}
</script>

<template>
  <header class="site-header">
    <div class="header-inner">
      <div class="brand">
        <span class="brand-icon"><Braces :size="22" :stroke-width="2.2" /></span>
        <span>Condition<span class="brand-accent">Edit</span></span>
      </div>
      <button class="help-button" type="button" @click="help.showModal()">
        <BookOpen :size="16" /> 使用说明
      </button>
    </div>
  </header>

  <main class="workspace">
    <section class="intro" aria-labelledby="page-title">
      <div class="eyebrow"><CodeXml :size="15" /> TEMPLATE EDITOR</div>
      <h1 id="page-title">条件文本编辑器<span>.</span></h1>
      <p>将普通文本、变量和条件语句组合成一个模板。</p>
    </section>

    <div class="workbench">
      <section class="panel editor-panel" aria-labelledby="editor-title">
        <div class="panel-header editor-header">
          <div class="panel-title">
            <FileText :size="19" class="accent" />
            <h2 id="editor-title">模板内容</h2>
          </div>
          <div class="editor-actions">
            <button class="button secondary" type="button" @click="loadExample"><RotateCcw :size="15" /> 载入示例</button>
            <button class="button primary" type="button" @click="copySource"><Copy :size="15" /> 复制文本</button>
          </div>
        </div>

        <div class="editor-body">
          <div ref="gutter" class="gutter" aria-hidden="true">
            <div v-for="(_, index) in lines" :key="index" :class="{ current: position.line === index + 1 }">{{ index + 1 }}</div>
          </div>
          <div class="input-area">
            <pre ref="highlight" class="highlight" aria-hidden="true"><span v-for="(token, index) in highlighted" :key="index" :class="token.type">{{ token.text }}</span>{{ '\n' }}</pre>
            <textarea ref="editor" v-model="source" aria-label="模板内容" placeholder="输入文本、变量或条件语句…" spellcheck="false"
              autocapitalize="off" autocomplete="off" wrap="off" @input="rememberSelection" @select="rememberSelection"
              @click="rememberSelection" @keyup="rememberSelection" @scroll="syncScroll"></textarea>
          </div>
        </div>

        <div class="editor-status">
          <span class="status-message" :class="{ invalid: issue }" aria-live="polite">
            <CircleAlert v-if="issue" :size="15" />
            <span v-else class="status-dot"></span>
            {{ issue || '条件结构正常' }}
          </span>
          <span class="status-position">行 {{ position.line }}，列 {{ position.column }} <span class="status-divider"></span> {{ source.length }} 字符</span>
        </div>
      </section>

      <aside class="sidebar" aria-label="插入工具">
        <section class="panel tools-panel" aria-labelledby="snippets-title">
          <div class="panel-header">
            <div class="panel-title"><GitBranch :size="18" class="accent" /><h2 id="snippets-title">快捷句式</h2></div>
          </div>
          <div class="snippet-list">
            <button v-for="snippet in snippets" :key="snippet.name" class="snippet" type="button" @click="insertSnippet(snippet)">
              <span class="snippet-marker"><CornerDownRight :size="15" /></span>
              <span class="snippet-name">{{ snippet.name }}</span>
              <code>{{ snippet.tag }}</code>
              <Plus :size="15" class="add-icon" />
            </button>
          </div>
        </section>

        <section class="panel tools-panel" aria-labelledby="variables-title">
          <div class="panel-header">
            <div class="panel-title"><Brackets :size="18" class="accent" /><h2 id="variables-title">可用变量</h2></div>
            <span class="variable-count">6 个变量</span>
          </div>
          <div class="variable-list">
            <div v-for="group in groups" :key="group.label" class="variable-group">
              <button class="group-heading" type="button" :aria-expanded="!collapsed[group.label]" @click="collapsed[group.label] = !collapsed[group.label]">
                <component :is="group.icon" :size="15" />
                {{ group.label }}
                <ChevronDown :size="15" class="chevron" :class="{ closed: collapsed[group.label] }" />
              </button>
              <div v-if="!collapsed[group.label]" class="variable-chips">
                <button v-for="variable in group.variables" :key="variable" class="chip" type="button" @click="insertVariable(variable)">
                  <Plus :size="13" /> {{ variable }}
                </button>
              </div>
            </div>
          </div>
        </section>
      </aside>
    </div>

    <footer class="workspace-footer"><Terminal :size="15" /> 句式和变量会插入到光标位置，或替换选中的文本。</footer>
  </main>

  <dialog ref="help" class="help-dialog" @click.self="help.close()">
    <div class="dialog-content">
      <div class="panel-title">
        <BookOpen :size="20" class="accent" />
        <h2>使用说明</h2>
        <button class="help-button close-button" type="button" aria-label="关闭" @click="help.close()"><X :size="19" /></button>
      </div>
      <p>在编辑器中直接输入普通文本。点击右侧的变量可插入占位符，例如 <code v-text="'{USER.a}'"></code>。</p>
      <p>点击 <strong>if 条件块</strong> 插入成对的 <code>#if</code> 和 <code>#endif</code>。选中条件中的变量后，可点击右侧变量替换它。</p>
      <p>在条件块内可添加多个 <code>#elseif</code>，以及一个位于最后的 <code>#else</code>。</p>
    </div>
  </dialog>
  <Transition><div v-if="feedback" class="toast" role="status"><Check :size="17" /> {{ feedback }}</div></Transition>
</template>
