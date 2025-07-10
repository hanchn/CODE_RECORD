<template>
  <div class="web-ide">
    <!-- 顶部工具栏 -->
    <header class="toolbar">
      <div class="toolbar-left">
        <h1 class="logo">
          <span class="logo-icon">💻</span>
          Web IDE
        </h1>
        <div class="file-info">
          <span class="file-name">{{ currentFile }}</span>
        </div>
      </div>
      
      <div class="toolbar-center">
        <div class="language-selector">
          <label for="language">语言:</label>
          <select 
            id="language" 
            v-model="currentLanguage" 
            @change="changeLanguage"
          >
            <option value="javascript">JavaScript</option>
            <option value="python">Python</option>
            <option value="java">Java</option>
            <option value="go">Go</option>
            <option value="shell">Shell/Bash</option>
            <option value="html">HTML</option>
            <option value="css">CSS</option>
          </select>
        </div>
      </div>
      
      <div class="toolbar-right">
        <div class="action-buttons">
          <button 
            class="btn btn-success" 
            @click="runCode" 
            :disabled="isRunning"
          >
            <span v-if="isRunning">⚡</span>
            <span v-else>▶️</span>
            {{ isRunning ? '运行中...' : '运行' }}
          </button>
          
          <button 
            class="btn btn-warning" 
            @click="formatCode"
          >
            🎨 格式化
          </button>
          
          <button 
            class="btn btn-secondary" 
            @click="clearOutput"
          >
            🗑️ 清空
          </button>
          
          <button 
            class="btn btn-primary" 
            @click="resetCode"
          >
            🔄 重置
          </button>
        </div>
      </div>
    </header>
    
    <!-- 主要内容区域 -->
    <main class="main-content">
      <!-- 左侧编辑器 -->
      <div class="editor-panel">
        <div class="panel-header">
          <h3>📝 代码编辑器</h3>
        </div>
        <div class="editor-container">
          <div ref="editorRef" class="code-editor"></div>
        </div>
      </div>
      
      <!-- 右侧输出面板 -->
      <div class="output-panel">
        <div class="panel-header">
          <h3>📋 运行输出</h3>
          <div class="output-controls">
            <button 
              class="btn btn-secondary btn-sm" 
              @click="clearOutput"
            >
              清空
            </button>
          </div>
        </div>
        <div class="output-container">
          <div 
            v-if="!output && !hasRun" 
            class="output-placeholder"
          >
            <div class="placeholder-content">
              <span class="placeholder-icon">🚀</span>
              <p>点击运行按钮执行代码</p>
              <p class="placeholder-hint">支持 JavaScript、Python、HTML、CSS</p>
            </div>
          </div>
          
          <div 
            v-else-if="isRunning" 
            class="output-loading"
          >
            <div class="loading-content">
              <span class="spinning">⚡</span>
              <p>代码执行中...</p>
            </div>
          </div>
          
          <div 
            v-else 
            class="output-content"
            :class="{ 'has-error': !lastRunSuccess }"
          >
            <pre v-if="output">{{ output }}</pre>
            <div v-else class="no-output">
              <span>✅</span>
              <p>代码执行完成，无输出内容</p>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { ref, onMounted, computed, nextTick } from 'vue'
import { EditorView, keymap, highlightActiveLine, highlightActiveLineGutter, lineNumbers } from '@codemirror/view'
import { EditorState } from '@codemirror/state'
import { javascript } from '@codemirror/lang-javascript'
import { python } from '@codemirror/lang-python'
import { html } from '@codemirror/lang-html'
import { css } from '@codemirror/lang-css'
import { java } from '@codemirror/lang-java'
import { go } from '@codemirror/lang-go'
import { StreamLanguage } from '@codemirror/language'
import { shell } from '@codemirror/legacy-modes/mode/shell'
import { oneDark } from '@codemirror/theme-one-dark'
import { defaultKeymap, history, historyKeymap } from '@codemirror/commands'
import { searchKeymap, highlightSelectionMatches } from '@codemirror/search'
import { autocompletion, completionKeymap, closeBrackets, closeBracketsKeymap } from '@codemirror/autocomplete'
import { foldKeymap, foldGutter } from '@codemirror/language'
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

    // 语言映射
    const languageMap = {
      javascript: javascript(),
      python: python(),
      java: java(),
      go: go(),
      shell: StreamLanguage.define(shell),
      html: html(),
      css: css()
    }

    // 默认代码模板
    const defaultCodes = {
      javascript: '',
      python: '',
      java: '',
      go: '',
      shell: '',
      html: '',
      css: ''
    }

    // 计算属性
    const currentFile = computed(() => {
      const extensions = {
        javascript: '.js',
        python: '.py',
        java: '.java',
        go: '.go',
        shell: '.sh',
        html: '.html',
        css: '.css'
      }
      return `main${extensions[currentLanguage.value]}`
    })

    // 初始化编辑器
    const initEditor = () => {
      console.log('初始化编辑器开始')
      console.log('editorRef.value:', editorRef.value)
      
      if (!editorRef.value) {
        console.error('编辑器容器未找到')
        return
      }

      if (editor.value) {
        editor.value.destroy()
      }

      try {
        const state = EditorState.create({
          doc: defaultCodes[currentLanguage.value] || '',
          extensions: [
            lineNumbers(),
            highlightActiveLineGutter(),
            highlightActiveLine(),
            foldGutter(),
            EditorView.lineWrapping,
            history(),
            autocompletion(),
            closeBrackets(),
            highlightSelectionMatches(),
            languageMap[currentLanguage.value],
            oneDark,
            keymap.of([
              ...defaultKeymap,
              ...historyKeymap,
              ...searchKeymap,
              ...completionKeymap,
              ...closeBracketsKeymap,
              ...foldKeymap,
              ...lintKeymap
            ])
          ]
        })

        editor.value = new EditorView({
          state,
          parent: editorRef.value
        })
        
        console.log('编辑器创建成功:', editor.value)

        nextTick(() => {
          if (editor.value) {
            editor.value.focus()
            console.log('编辑器获得焦点')
          }
        })
      } catch (error) {
        console.error('编辑器初始化失败:', error)
      }
    }

    // 切换语言
    const changeLanguage = () => {
      if (editor.value) {
        editor.value.dispatch({
          changes: {
            from: 0,
            to: editor.value.state.doc.length,
            insert: defaultCodes[currentLanguage.value]
          }
        })
        
        editor.value.dispatch({
          effects: EditorState.reconfigure.of([
            lineNumbers(),
            highlightActiveLineGutter(),
            highlightActiveLine(),
            foldGutter(),
            EditorView.lineWrapping,
            history(),
            autocompletion(),
            closeBrackets(),
            highlightSelectionMatches(),
            languageMap[currentLanguage.value],
            oneDark,
            keymap.of([
              ...defaultKeymap,
              ...historyKeymap,
              ...searchKeymap,
              ...completionKeymap,
              ...closeBracketsKeymap,
              ...foldKeymap,
              ...lintKeymap
            ])
          ])
        })
      }
    }

    // 运行代码
    const runCode = async () => {
      if (!editor.value || isRunning.value) return

      const code = editor.value.state.doc.toString()
      if (!code.trim()) {
        output.value = '错误: 代码不能为空'
        lastRunSuccess.value = false
        hasRun.value = true
        return
      }

      isRunning.value = true
      hasRun.value = true
      output.value = ''

      try {
        let result = ''
        
        switch (currentLanguage.value) {
          case 'javascript':
            result = await runJavaScript(code)
            break
          case 'python':
            result = await runPython(code)
            break
          case 'java':
            result = await runJava(code)
            break
          case 'go':
            result = await runGo(code)
            break
          case 'shell':
            result = await runShell(code)
            break
          case 'html':
            result = await runHTML(code)
            break
          case 'css':
            result = await runCSS(code)
            break
          default:
            result = '不支持的语言类型'
        }
        
        output.value = result
        lastRunSuccess.value = true
      } catch (error) {
        output.value = `错误: ${error.message}`
        lastRunSuccess.value = false
      } finally {
        isRunning.value = false
      }
    }

    // JavaScript 执行
    const runJavaScript = async (code) => {
      return new Promise((resolve) => {
        const logs = []
        const originalLog = console.log
        
        console.log = (...args) => {
          logs.push(args.map(arg => 
            typeof arg === 'object' ? JSON.stringify(arg, null, 2) : String(arg)
          ).join(' '))
        }
        
        try {
          const func = new Function(code)
          const result = func()
          
          if (result !== undefined) {
            logs.push('返回值: ' + (typeof result === 'object' ? JSON.stringify(result, null, 2) : String(result)))
          }
        } catch (error) {
          logs.push('执行错误: ' + error.message)
        } finally {
          console.log = originalLog
        }
        
        resolve(logs.length > 0 ? logs.join('\n') : '代码执行完成，无输出')
      })
    }

    // Python 执行 (模拟)
    const runPython = async (code) => {
      return new Promise((resolve) => {
        setTimeout(() => {
          if (code.includes('print(')) {
            const matches = code.match(/print\(([^)]+)\)/g)
            if (matches) {
              const outputs = matches.map(match => {
                const content = match.match(/print\(([^)]+)\)/)[1]
                return content.replace(/['"`]/g, '')
              })
              resolve(outputs.join('\n'))
            } else {
              resolve('Python 代码执行完成')
            }
          } else {
            resolve('Python 代码执行完成，无输出')
          }
        }, 1000)
      })
    }

    // Java 执行 (模拟)
    const runJava = async (code) => {
      return new Promise((resolve) => {
        setTimeout(() => {
          if (code.includes('System.out.print')) {
            const matches = code.match(/System\.out\.print(?:ln)?\(([^)]+)\)/g)
            if (matches) {
              const outputs = matches.map(match => {
                const content = match.match(/System\.out\.print(?:ln)?\(([^)]+)\)/)[1]
                return content.replace(/["]]/g, '')
              })
              resolve(outputs.join('\n'))
            } else {
              resolve('Java 程序执行完成')
            }
          } else {
            resolve('Java 程序编译并执行完成，无输出')
          }
        }, 1500)
      })
    }

    // Go 执行 (模拟)
    const runGo = async (code) => {
      return new Promise((resolve) => {
        setTimeout(() => {
          if (code.includes('fmt.Print')) {
            const matches = code.match(/fmt\.Print(?:ln|f)?\(([^)]+)\)/g)
            if (matches) {
              const outputs = matches.map(match => {
                const content = match.match(/fmt\.Print(?:ln|f)?\(([^)]+)\)/)[1]
                return content.replace(/["\`]/g, '')
              })
              resolve(outputs.join('\n'))
            } else {
              resolve('Go 程序执行完成')
            }
          } else {
            resolve('Go 程序编译并执行完成，无输出')
          }
        }, 1200)
      })
    }

    // Shell 执行 (模拟)
    const runShell = async (code) => {
      return new Promise((resolve) => {
        setTimeout(() => {
          if (code.includes('echo')) {
            const matches = code.match(/echo\s+["']?([^"'\n]+)["']?/g)
            if (matches) {
              const outputs = matches.map(match => {
                const content = match.replace(/echo\s+["']?/, '').replace(/["']$/, '')
                return content
              })
              resolve(outputs.join('\n'))
            } else {
              resolve('Shell 脚本执行完成')
            }
          } else {
            resolve('Shell 脚本执行完成，无输出')
          }
        }, 800)
      })
    }

    // HTML 执行
    const runHTML = async (code) => {
      return new Promise((resolve) => {
        try {
          const newWindow = window.open('', '_blank')
          if (newWindow) {
            newWindow.document.write(code)
            newWindow.document.close()
            resolve('HTML 页面已在新窗口中打开')
          } else {
            resolve('无法打开新窗口，请检查浏览器弹窗设置')
          }
        } catch (error) {
          resolve('HTML 执行错误: ' + error.message)
        }
      })
    }

    // CSS 执行
    const runCSS = async (code) => {
      return new Promise((resolve) => {
        try {
          const htmlContent = `<!DOCTYPE html>\n<html>\n<head>\n<style>\n${code}\n</style>\n</head>\n<body>\n<div class="container">\n<div class="card">\n<h2>CSS 样式预览</h2>\n<p>这是一个演示段落</p>\n<button class="btn">示例按钮</button>\n</div>\n</div>\n</body>\n</html>`
          
          const newWindow = window.open('', '_blank')
          if (newWindow) {
            newWindow.document.write(htmlContent)
            newWindow.document.close()
            resolve('CSS 样式预览已在新窗口中打开')
          } else {
            resolve('无法打开新窗口，请检查浏览器弹窗设置')
          }
        } catch (error) {
          resolve('CSS 执行错误: ' + error.message)
        }
      })
    }

    // 格式化代码
    const formatCode = () => {
      if (!editor.value) return
      
      const code = editor.value.state.doc.toString()
      let formattedCode = code
      
      try {
        switch (currentLanguage.value) {
          case 'javascript':
            formattedCode = code.replace(/{/g, '{\n').replace(/}/g, '\n}').replace(/;/g, ';\n').replace(/\n\s*\n/g, '\n').trim()
            break
          case 'html':
            formattedCode = code.replace(/></g, '>\n<')
            break
          case 'css':
            formattedCode = code.replace(/{/g, ' {\n').replace(/}/g, '\n}\n').replace(/;/g, ';\n')
            break
          default:
            formattedCode = code.split('\n').map(line => line.trim()).join('\n')
        }
        
        editor.value.dispatch({
          changes: {
            from: 0,
            to: editor.value.state.doc.length,
            insert: formattedCode
          }
        })
      } catch (error) {
        console.error('格式化失败:', error)
      }
    }

    // 清空输出
    const clearOutput = () => {
      output.value = ''
      hasRun.value = false
      lastRunSuccess.value = true
    }

    // 重置代码
    const resetCode = () => {
      if (editor.value) {
        editor.value.dispatch({
          changes: {
            from: 0,
            to: editor.value.state.doc.length,
            insert: defaultCodes[currentLanguage.value]
          }
        })
      }
    }

    // 组件挂载
    onMounted(() => {
      console.log('组件已挂载')
      // 延迟初始化编辑器，确保DOM完全渲染
      nextTick(() => {
        setTimeout(() => {
          initEditor()
        }, 100)
      })
    })

    return {
      editorRef,
      currentLanguage,
      currentFile,
      output,
      isRunning,
      lastRunSuccess,
      hasRun,
      changeLanguage,
      runCode,
      formatCode,
      clearOutput,
      resetCode
    }
  }
}
</script>

<style scoped>
.web-ide {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #1e1e1e;
  color: #ffffff;
}

.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 20px;
  background: #2d2d2d;
  border-bottom: 1px solid #444;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.toolbar-left {
  display: flex;
  align-items: center;
  gap: 20px;
}

.logo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 20px;
  font-weight: 700;
  color: #007acc;
  margin: 0;
}

.logo-icon {
  font-size: 24px;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 12px;
  background: #3c3c3c;
  border-radius: 6px;
  font-size: 14px;
}

.file-name {
  color: #ffd700;
  font-weight: 500;
}

.toolbar-center {
  display: flex;
  align-items: center;
}

.language-selector {
  display: flex;
  align-items: center;
  gap: 8px;
}

.language-selector label {
  font-size: 14px;
  font-weight: 500;
}

.toolbar-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.action-buttons {
  display: flex;
  gap: 8px;
}

.btn-sm {
  padding: 4px 8px;
  font-size: 12px;
}

.main-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

.editor-panel,
.output-panel {
  display: flex;
  flex-direction: column;
  background: #1e1e1e;
}

.editor-panel {
  flex: 1;
  border-right: 1px solid #444;
}

.output-panel {
  width: 400px;
  min-width: 300px;
  max-width: 50%;
}

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: #2d2d2d;
  border-bottom: 1px solid #444;
}

.panel-header h3 {
  margin: 0;
  font-size: 14px;
  font-weight: 600;
  color: #ffffff;
}

.output-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.editor-container {
  flex: 1;
  overflow: hidden;
}

.code-editor {
  height: 100%;
  width: 100%;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
}

/* CodeMirror 编辑器样式 */
.code-editor :deep(.cm-editor) {
  height: 100%;
  font-size: 14px;
}

.code-editor :deep(.cm-focused) {
  outline: none;
}

.code-editor :deep(.cm-scroller) {
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
}

.code-editor :deep(.cm-gutters) {
  background-color: #2d2d2d;
  border-right: 1px solid #444;
}

.code-editor :deep(.cm-lineNumbers .cm-gutterElement) {
  color: #858585;
  font-size: 13px;
}

.code-editor :deep(.cm-cursor) {
  border-left: 2px solid #ffffff;
}

.code-editor :deep(.cm-activeLine) {
  background-color: rgba(255, 255, 255, 0.05);
}

.code-editor :deep(.cm-activeLineGutter) {
  background-color: rgba(255, 255, 255, 0.1);
}

.output-container {
  flex: 1;
  overflow: auto;
  background: #1a1a1a;
}

.output-placeholder {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #888;
}

.placeholder-content {
  text-align: center;
}

.placeholder-icon {
  font-size: 48px;
  display: block;
  margin-bottom: 16px;
}

.placeholder-content p {
  margin: 8px 0;
}

.placeholder-hint {
  font-size: 12px;
  color: #666;
}

.output-loading {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #ffc107;
}

.loading-content {
  text-align: center;
}

.loading-content span {
  font-size: 32px;
  display: block;
  margin-bottom: 12px;
}

.output-content {
  padding: 16px;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  font-size: 13px;
  line-height: 1.5;
}

.output-content.has-error {
  color: #ff6b6b;
}

.output-content pre {
  margin: 0;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.no-output {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #28a745;
  font-style: italic;
}

.no-output span {
  font-size: 16px;
}

.spinning {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@media (max-width: 1024px) {
  .output-panel {
    width: 350px;
  }
  
  .toolbar {
    flex-wrap: wrap;
    gap: 12px;
  }
  
  .toolbar-center {
    order: 3;
    flex-basis: 100%;
  }
}

@media (max-width: 768px) {
  .main-content {
    flex-direction: column;
  }
  
  .editor-panel {
    border-right: none;
    border-bottom: 1px solid #444;
  }
  
  .output-panel {
    width: 100%;
    max-width: 100%;
    height: 300px;
  }
  
  .toolbar {
    padding: 8px 12px;
  }
  
  .action-buttons {
    flex-wrap: wrap;
  }
  
  .btn {
    padding: 6px 12px;
    font-size: 12px;
  }
}
</style>