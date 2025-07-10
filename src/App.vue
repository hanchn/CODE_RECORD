<template>
  <div class="ide-container">
    <!-- 顶部工具栏 -->
    <div class="toolbar">
      <div class="toolbar-left">
        <h1 class="title">Web IDE</h1>
        <select v-model="currentLanguage" @change="changeLanguage" class="language-select">
          <option value="javascript">JavaScript</option>
          <option value="python">Python</option>
          <option value="html">HTML</option>
          <option value="css">CSS</option>
        </select>
      </div>
      <div class="toolbar-right">
        <button @click="runCode" class="btn btn-primary" :disabled="isRunning">
          {{ isRunning ? '运行中...' : '运行代码' }}
        </button>
        <button @click="clearOutput" class="btn btn-secondary">清空输出</button>
        <button @click="formatCode" class="btn btn-secondary">格式化</button>
      </div>
    </div>

    <!-- 主要内容区域 -->
    <div class="main-content">
      <!-- 左侧编辑器 -->
      <div class="editor-panel">
        <div class="panel-header">
          <span class="panel-title">代码编辑器</span>
          <span class="file-info">{{ currentFile }}</span>
        </div>
        <div ref="editorRef" class="editor"></div>
      </div>

      <!-- 右侧输出面板 -->
      <div class="output-panel">
        <div class="panel-header">
          <span class="panel-title">输出结果</span>
          <span class="status" :class="{ 'success': lastRunSuccess, 'error': !lastRunSuccess && hasRun }">
            {{ runStatus }}
          </span>
        </div>
        <div class="output-content">
          <pre v-if="output" class="output-text">{{ output }}</pre>
          <div v-else class="output-placeholder">点击"运行代码"查看输出结果</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { EditorView, keymap, highlightActiveLine, highlightActiveLineGutter, lineNumbers, foldGutter } from '@codemirror/view'
import { EditorState } from '@codemirror/state'
import { javascript } from '@codemirror/lang-javascript'
import { python } from '@codemirror/lang-python'
import { html } from '@codemirror/lang-html'
import { css } from '@codemirror/lang-css'
import { oneDark } from '@codemirror/theme-one-dark'
import { defaultKeymap, history, historyKeymap } from '@codemirror/commands'
import { searchKeymap, highlightSelectionMatches } from '@codemirror/search'
import { autocompletion, completionKeymap, closeBrackets, closeBracketsKeymap } from '@codemirror/autocomplete'
import { foldKeymap } from '@codemirror/language'
import { lintKeymap } from '@codemirror/lint'

export default {
  name: 'WebIDE',
  setup() {
    const editorRef = ref(null)
    const editor = ref(null)
    const currentLanguage = ref('javascript')
    const output = ref('')
    const isRunning = ref(false)
    const lastRunSuccess = ref(true)
    const hasRun = ref(false)

    // 初始化编辑器
    const initEditor = () => {
      if (!editorRef.value) return

      // 清理之前的编辑器实例
      if (editor.value) {
        editor.value.destroy()
        editor.value = null
      }

      const extensions = [
        lineNumbers(),
        highlightActiveLineGutter(),
        highlightActiveLine(),
        foldGutter(),
        history(),
        autocompletion(),
        closeBrackets(),
        highlightSelectionMatches(),
        javascript(),
        oneDark,
        keymap.of([
          ...defaultKeymap,
          ...historyKeymap,
          ...searchKeymap,
          ...completionKeymap,
          ...closeBracketsKeymap,
          ...foldKeymap,
          ...lintKeymap
        ]),
        EditorView.lineWrapping,
        EditorView.theme({
          '&': {
            fontSize: '14px',
            height: '100%'
          },
          '.cm-content': {
            padding: '20px',
            minHeight: '100%'
          },
          '.cm-focused': {
            outline: 'none'
          },
          '.cm-editor': {
            height: '100%'
          },
          '.cm-scroller': {
            fontFamily: 'Monaco, Menlo, "Ubuntu Mono", monospace'
          }
        })
      ]

      const startState = EditorState.create({
        doc: '// Welcome to Simple IDE\n// Start coding here...\n\nfunction hello() {\n  console.log("Hello, World!");\n}\n\nhello();',
        extensions
      })

      editor.value = new EditorView({
        state: startState,
        parent: editorRef.value
      })
    }

    // 生命周期
    onMounted(() => {
      setTimeout(() => {
        initEditor()
        // 确保编辑器获得焦点
        setTimeout(() => {
          if (editor.value) {
            editor.value.focus()
          }
        }, 200)
      }, 100)
    })

    return {
      editorRef
    }
  }
}
</script>

<style scoped>
.ide-container {
  height: 100vh;
  width: 100vw;
  display: flex;
  flex-direction: column;
  background: #1e1e1e;
}

.editor-container {
  flex: 1;
  margin: 0;
  border: none;
  overflow: hidden;
  background: #1e1e1e;
}

.editor {
  height: 100%;
  width: 100%;
  cursor: text;
}

.editor :deep(.cm-editor) {
  height: 100%;
  outline: none;
}

.editor :deep(.cm-content) {
  min-height: 100%;
  padding: 20px;
}

.editor :deep(.cm-focused) {
  outline: none;
}

.editor :deep(.cm-scroller) {
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
}

.editor :deep(.cm-line) {
  cursor: text;
}

.editor :deep(.cm-content) {
  cursor: text;
  user-select: text;
}

.editor :deep(.cm-editor.cm-focused) {
  outline: none;
}

.editor :deep(.cm-cursor) {
  border-left: 2px solid #007acc;
}
</style>