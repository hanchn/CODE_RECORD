<template>
  <div class="ide-container">
    <!-- 代码编辑器 -->
    <div class="editor-container">
      <div ref="editorRef" class="editor"></div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import { EditorView, keymap, highlightActiveLine, highlightActiveLineGutter, lineNumbers, foldGutter } from '@codemirror/view'
import { EditorState } from '@codemirror/state'
import { javascript } from '@codemirror/lang-javascript'
import { oneDark } from '@codemirror/theme-one-dark'
import { defaultKeymap, history, historyKeymap } from '@codemirror/commands'
import { searchKeymap, highlightSelectionMatches } from '@codemirror/search'
import { autocompletion, completionKeymap, closeBrackets, closeBracketsKeymap } from '@codemirror/autocomplete'
import { foldKeymap } from '@codemirror/language'
import { lintKeymap } from '@codemirror/lint'

export default {
  name: 'App',
  setup() {
    const editorRef = ref(null)
    const editor = ref(null)

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