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
  { name: 'if 条件块', tag: '#if … #endif', text: '{#if USER.a == USER.b}\n  \n{#endif}', operands: ['USER.a', 'USER.b'] },
  { name: 'elseif 分支', tag: '#elseif', text: '{#elseif USER.a == USER.b}\n  ', operands: ['USER.a', 'USER.b'] },
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
const picker = ref(null)
const menu = ref(null)
const caretProbe = ref(null)
const activeChoice = ref(0)
const menuPosition = ref({})
const operandStep = ref(null)
let feedbackTimer

const lines = computed(() => source.value.split('\n'))
const variableChoices = groups.flatMap(group => group.variables.map(name => ({
  name, detail: group.label, icon: group.icon,
})))
const variableNames = new Set(variableChoices.map(choice => choice.name.toLowerCase()))
const snippetChoices = snippets.map(snippet => ({
  name: snippet.tag.split(' ')[0], detail: snippet.name, icon: GitBranch, snippet,
}))
const choices = computed(() => {
  if (!picker.value) return []
  const { kind, query } = picker.value
  const options = kind === 'snippet' ? snippetChoices
    : kind === 'all' ? [...variableChoices, ...snippetChoices] : variableChoices
  return options.filter(choice => choice.name.toLowerCase().startsWith(query.toLowerCase()))
})
const highlighted = computed(() => source.value.split(/(\{#[^{}\n]*\}|\{[\w.]+\})/g).flatMap((part, index) => {
  if (index % 2 === 0) return [{ text: part }]
  if (!part.startsWith('{#')) return [{ text: part, type: 'variable' }]
  return part.split(/("(?:\\.|[^"\\])*"|'(?:\\.|[^'\\])*'|[\w.]+|===|!==|==|!=|>=|<=|&&|\|\||[<>])/g)
    .map((text, index) => {
      if (index % 2 === 0) return { text, type: 'directive' }
      if (variableNames.has(text.toLowerCase()) || /^(?:SYS|USER)\.\w+$/i.test(text)) return { text, type: 'variable' }
      return { text, type: /^[=!<>&|]/.test(text) ? 'operator' : 'directive' }
    })
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
  if (picker.value) positionMenu()
}

function positionMenu() {
  if (!menu.value || !caretProbe.value) return
  const input = editor.value
  const lineHeight = Number.parseFloat(getComputedStyle(input).lineHeight)
  const caretLeft = caretProbe.value.offsetWidth + caretProbe.value.offsetLeft - input.scrollLeft
  const caretBottom = lineHeight * position.value.line + caretProbe.value.offsetTop - input.scrollTop
  const width = menu.value.offsetWidth
  const height = menu.value.offsetHeight
  const top = caretBottom + height + 8 <= input.clientHeight
    ? caretBottom + 4 : caretBottom - lineHeight - height - 4
  menuPosition.value = {
    left: `${Math.max(4, Math.min(caretLeft, input.clientWidth - width - 4))}px`,
    top: `${Math.max(4, Math.min(top, input.clientHeight - height - 4))}px`,
  }
}

function completionContext(manual, replace) {
  const { start, end } = selection.value
  const value = editor.value.value
  const before = value.slice(0, start).split('\n').at(-1)
  const after = value.slice(end).split('\n')[0]
  const selected = value.slice(start, end)
  const directive = before.match(/\{(#[a-z]*)$/i)
  if (directive && start === end && !replace) {
    const suffix = after.match(/^[a-z]*\}?/i)[0]
    return { kind: 'snippet', query: directive[1], start: start - directive[0].length, end: end + suffix.length }
  }
  const placeholder = before.match(/\{([\w.]*)$/)
  if (placeholder && !selected.includes('}')) {
    const suffix = after.match(/^[\w.]*\}?/)[0]
    return {
      kind: 'variable', query: manual ? '' : placeholder[1],
      start: start - placeholder[0].length, end: end + suffix.length,
    }
  }
  const condition = /\{#(?:if|elseif)\s+[^{}]*$/.test(before)
  if (condition) {
    const prefix = before.match(/[\w.]*$/)[0]
    const suffix = after.match(/^[\w.]*/)[0]
    if (manual || prefix) {
      return { kind: 'operand', query: manual ? '' : prefix, start: start - prefix.length, end: end + suffix.length }
    }
  }
  if (manual && !replace) return { kind: 'all', query: '', start, end }
  return null
}

async function openPicker(manual = false, replace = false, variablesOnly = false) {
  rememberSelection()
  const context = completionContext(manual, replace)
  if (replace && (!context || !['variable', 'operand'].includes(context.kind))) return
  picker.value = variablesOnly && context
    ? { ...context, kind: context.kind === 'operand' ? 'operand' : 'variable', query: '' }
    : context
  activeChoice.value = 0
  await nextTick()
  positionMenu()
}

function openVariablePicker() {
  if (operandStep.value) {
    picker.value = { ...picker.value, query: '' }
    activeChoice.value = 0
    nextTick(positionMenu)
    return
  }
  const { start, end } = selection.value
  editor.value.focus()
  editor.value.setSelectionRange(start, end)
  openPicker(true, false, true)
}

function closePicker() {
  picker.value = null
  operandStep.value = null
}

async function selectOperand(ranges, index = 0) {
  const { start, end } = ranges[index]
  const input = editor.value
  input.setSelectionRange(start, end)
  rememberSelection()
  operandStep.value = {
    index, ranges,
    before: input.value.slice(0, start),
    after: input.value.slice(end),
  }
  picker.value = { kind: 'operand', query: '', start, end }
  activeChoice.value = 0
  await nextTick()
  positionMenu()
}

function onEditorInput(event) {
  rememberSelection()
  if (event.isComposing) return
  const step = operandStep.value
  if (step) {
    const value = editor.value.value
    const start = step.before.length
    const end = value.length - step.after.length
    const inOperand = start <= end && value.startsWith(step.before) && value.endsWith(step.after)
    if (inOperand && !event.inputType?.startsWith('history')) {
      picker.value = { kind: 'operand', query: value.slice(start, end), start, end }
      activeChoice.value = 0
      nextTick(positionMenu)
      return
    }
    closePicker()
  }
  if (!event.inputType?.startsWith('history')) openPicker()
}

async function confirmOperand(variable) {
  const { index, ranges } = operandStep.value
  const { start, end } = picker.value
  const replacement = variable ?? editor.value.value.slice(start, end)
  const shift = start + replacement.length - ranges[index].end
  closePicker()
  if (variable !== undefined) {
    selection.value = { start, end }
    await insert(replacement)
  }
  if (index + 1 < ranges.length) {
    const adjusted = ranges.map((range, slot) => slot <= index
      ? range
      : { start: range.start + shift, end: range.end + shift })
    await selectOperand(adjusted, index + 1)
  }
}

function choose(choice) {
  if (!choice || !picker.value) return
  if (operandStep.value) {
    confirmOperand(choice.name)
    return
  }
  const { kind, start, end } = picker.value
  selection.value = { start, end }
  picker.value = null
  if (choice.snippet) insertSnippet(choice.snippet)
  else insert(kind === 'operand' ? choice.name : `{${choice.name}}`)
}

async function onEditorKeydown(event) {
  if (event.isComposing || event.keyCode === 229) return
  if ((event.ctrlKey || event.metaKey) && event.code === 'Space') {
    event.preventDefault()
    if (operandStep.value) openVariablePicker()
    else openPicker(true)
    return
  }
  if (!picker.value) return
  if (event.key === 'Escape') {
    event.preventDefault()
    closePicker()
  } else if (choices.value.length && ['ArrowDown', 'ArrowUp'].includes(event.key)) {
    event.preventDefault()
    const count = choices.value.length
    const step = event.key === 'ArrowDown' ? 1 : -1
    activeChoice.value = (activeChoice.value + step + count) % count
    await nextTick()
    menu.value?.querySelector('[aria-selected="true"]')?.scrollIntoView({ block: 'nearest' })
  } else if (choices.value.length && (event.key === 'Enter' || (event.key === 'Tab' && !event.shiftKey))) {
    event.preventDefault()
    choose(choices.value[activeChoice.value])
  } else if (operandStep.value && picker.value.query && (event.key === 'Enter' || (event.key === 'Tab' && !event.shiftKey))) {
    event.preventDefault()
    confirmOperand()
  } else if (['ArrowLeft', 'ArrowRight', 'Home', 'End', 'PageUp', 'PageDown', 'Tab', 'Enter'].includes(event.key)) {
    closePicker()
  }
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
  closePicker()
  source.value = input.value
  await nextTick()
  input.setSelectionRange(start + offset, start + offset + length)
  rememberSelection()
  syncScroll()
}

function insertVariable(variable) {
  if (operandStep.value) {
    confirmOperand(variable)
    return
  }
  const { start, end } = selection.value
  const before = source.value.slice(0, start).split('\n').at(-1)
  const after = source.value.slice(end).split('\n')[0]
  const insideCondition = /\{#(?:if|elseif)\s+[^{}]*$/.test(before) && !/[{}]/.test(after.split('}')[0])
  const insidePlaceholder = /\{[\w.]*$/.test(before) && /^[\w.]*\}/.test(after)
  insert(insideCondition || insidePlaceholder ? variable : `{${variable}}`)
}

async function insertSnippet(snippet) {
  const { start, end } = selection.value
  const before = source.value.slice(0, start).split('\n').at(-1)
  const after = source.value.slice(end).split('\n')[0]
  const prefix = before.trim() ? '\n' : ''
  const suffix = after.trim() ? '\n' : ''
  const text = prefix + snippet.text + suffix
  if (!snippet.operands) {
    await insert(text, prefix.length + snippet.text.length)
    return
  }
  const ranges = snippet.operands.map(operand => {
    const offset = prefix.length + snippet.text.indexOf(operand)
    return { start: start + offset, end: start + offset + operand.length }
  })
  await insert(text, ranges[0].start - start, snippet.operands[0].length)
  await selectOperand(ranges)
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
            <button class="button secondary variable-picker-button" type="button"
              @pointerdown="operandStep && $event.preventDefault()" @click="openVariablePicker">
              <Brackets :size="15" /> 选变量
            </button>
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
            <textarea ref="editor" v-model="source" aria-label="模板内容" aria-describedby="editor-hint"
              aria-autocomplete="list" :aria-controls="picker ? 'editor-choices' : undefined"
              :aria-activedescendant="picker && choices.length ? 'editor-choice-' + activeChoice : undefined"
              placeholder="输入文本、变量或条件语句…" spellcheck="false"
              autocapitalize="off" autocomplete="off" wrap="off" @input="onEditorInput"
              @select="rememberSelection" @click="rememberSelection(); closePicker()"
              @dblclick="openPicker(true, true)" @keydown="onEditorKeydown"
              @keyup="rememberSelection" @blur="closePicker" @scroll="syncScroll"></textarea>
            <template v-if="picker">
              <span ref="caretProbe" class="caret-probe" aria-hidden="true">{{ lines[position.line - 1]?.slice(0, position.column - 1) }}</span>
              <div ref="menu" class="completion-menu" :style="menuPosition">
                <div class="completion-heading">
                  {{ operandStep ? `选择第 ${operandStep.index + 1} 个变量（共 ${operandStep.ranges.length} 个）`
                    : picker.kind === 'snippet' ? '快捷句式' : picker.kind === 'all' ? '插入变量或句式' : '选择变量' }}
                  <span>↑ ↓ · Enter</span>
                </div>
                <div id="editor-choices" role="listbox" aria-label="插入候选项" class="completion-options">
                  <div v-for="(choice, index) in choices" :id="'editor-choice-' + index" :key="choice.name"
                    role="option" :aria-selected="index === activeChoice" class="completion-option"
                    @pointerdown.prevent @click="choose(choice)">
                    <component :is="choice.icon" :size="15" />
                    <code>{{ choice.name }}</code><span>{{ choice.detail }}</span>
                  </div>
                </div>
                <div v-if="!choices.length" class="completion-empty" role="status">
                  {{ operandStep && picker.query ? '没有匹配项，按 Enter 保留输入' : '没有匹配项，可继续输入' }}
                </div>
              </div>
            </template>
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
                <button v-for="variable in group.variables" :key="variable" class="chip" type="button"
                  @pointerdown="operandStep && $event.preventDefault()" @click="insertVariable(variable)">
                  <Plus :size="13" /> {{ variable }}
                </button>
              </div>
            </div>
          </div>
        </section>
      </aside>
    </div>

    <footer id="editor-hint" class="workspace-footer">
      <Terminal :size="15" />
      <span>输入 <kbd>{</kbd> 选变量，输入 <kbd>{#</kbd> 选句式，也可按 <kbd>Ctrl / ⌘ + Space</kbd> 打开候选。</span>
    </footer>
  </main>

  <dialog ref="help" class="help-dialog" @click.self="help.close()">
    <div class="dialog-content">
      <div class="panel-title">
        <BookOpen :size="20" class="accent" />
        <h2>使用说明</h2>
        <button class="help-button close-button" type="button" aria-label="关闭" @click="help.close()"><X :size="19" /></button>
      </div>
      <p>在编辑器中直接输入普通文本。点击右侧的变量可插入占位符，例如 <code v-text="'{USER.a}'"></code>。</p>
      <p>输入左花括号可筛选变量，再输入 <code>#</code> 可筛选句式。双击已有变量可以换成其他变量，也可以直接编辑变量名。</p>
      <p>插入 <code>#if</code> 或 <code>#elseif</code> 后，会依次选中两个变量。可选择候选项，或直接输入后按 Enter 确认。按 Esc 可退出逐项选择。</p>
      <p>在条件块内可添加多个 <code>#elseif</code>，以及一个位于最后的 <code>#else</code>。</p>
    </div>
  </dialog>
  <Transition><div v-if="feedback" class="toast" role="status"><Check :size="17" /> {{ feedback }}</div></Transition>
</template>
