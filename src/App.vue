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
            <option value="html">HTML</option>
            <option value="css">CSS</option>
          </select>
        </div>
      </div>
      
      <div class="toolbar-right">
        <div class="status-info">
          <div 
            class="status-indicator"
            :class="{
              'status-ready': !hasRun,
              'status-running': isRunning,
              'status-success': hasRun && !isRunning && lastRunSuccess,
              'status-error': hasRun && !isRunning && !lastRunSuccess
            }"
          >
            <span v-if="isRunning" class="spinning">⚡</span>
            <span v-else-if="!hasRun">✅</span>
            <span v-else-if="lastRunSuccess">✅</span>
            <span v-else>❌</span>
            {{ runStatus }}
          </div>
        </div>
        
        <div class="action-buttons">
          <button 
            class="btn btn-success tooltip" 
            @click="runCode" 
            :disabled="isRunning"
            data-tooltip="运行代码 (Ctrl+Enter)"
          >
            <span v-if="isRunning" class="spinning">⚡</span>
            <span v-else>▶️</span>
            {{ isRunning ? '运行中...' : '运行' }}
          </button>
          
          <button 
            class="btn btn-warning tooltip" 
            @click="formatCode"
            data-tooltip="格式化代码 (Shift+Alt+F)"
          >
            🎨 格式化
          </button>
          
          <button 
            class="btn btn-secondary tooltip" 
            @click="clearOutput"
            data-tooltip="清空输出"
          >
            🗑️ 清空
          </button>
          
          <button 
            class="btn btn-primary tooltip" 
            @click="resetCode"
            data-tooltip="重置为示例代码"
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
          <div class="editor-info">
            <span class="line-info" v-if="editorStats.lines">
              {{ editorStats.lines }} 行 | {{ editorStats.chars }} 字符
            </span>
          </div>
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
              title="清空输出"
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
    const editorStats = ref({ lines: 0, chars: 0 })

    // 语言映射
    const languageMap = {
      javascript: javascript(),
      python: python(),
      html: html(),
      css: css()
    }

    // 默认代码模板
    const defaultCode = {
      javascript: `// JavaScript 示例代码
console.log('Hello, World!');

// 计算斐波那契数列
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

console.log('斐波那契数列前10项:');
for (let i = 0; i < 10; i++) {
  console.log(`F(${i}) = ${fibonacci(i)}`);
}`,
      python: `# Python 示例代码
print('Hello, World!')

# 计算斐波那契数列
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print('斐波那契数列前10项:')
for i in range(10):
    print(f'F({i}) = {fibonacci(i)}')`,
      html: `<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web IDE Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 40px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            text-align: center;
        }
        .card {
            background: rgba(255, 255, 255, 0.1);
            border-radius: 15px;
            padding: 30px;
            margin: 20px 0;
            backdrop-filter: blur(10px);
        }
        button {
            background: #007acc;
            color: white;
            border: none;
            padding: 12px 24px;
            border-radius: 8px;
            cursor: pointer;
            font-size: 16px;
            transition: all 0.3s ease;
        }
        button:hover {
            background: #005a9e;
            transform: translateY(-2px);
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="card">
            <h1>🎉 欢迎使用 Web IDE</h1>
            <p>这是一个功能强大的在线代码编辑器</p>
            <button onclick="showMessage()">点击我试试</button>
        </div>
    </div>
    
    <script>
        function showMessage() {
            alert('Hello from Web IDE! 🚀');
            console.log('按钮被点击了!');
        }
    </script>
</body>
</html>`,
      css: `/* CSS 示例代码 - 现代卡片设计 */
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin: 0;
  padding: 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
}

.card {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 15px;
  padding: 30px;
  backdrop-filter: blur(10px);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.card:hover {
  transform: translateY(-10px);
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.2);
}

.card h2 {
  color: #fff;
  margin-bottom: 15px;
  font-size: 1.8em;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
}

.card p {
  color: rgba(255, 255, 255, 0.9);
  line-height: 1.6;
  margin-bottom: 20px;
}

.btn {
  background: linear-gradient(45deg, #007acc, #0056b3);
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 25px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.btn:hover {
  background: linear-gradient(45deg, #0056b3, #004085);
  transform: translateY(-2px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.2);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .container {
    grid-template-columns: 1fr;
    padding: 10px;
  }
  
  .card {
    padding: 20px;
  }
}`
    }

    // 计算属性
    const currentFile = computed(() => {
      const extensions = {
        javascript: '.js',
        python: '.py',
        html: '.html',
        css: '.css'
      }
      return `main${extensions[currentLanguage.value]}`
    })

    const runStatus = computed(() => {
      if (!hasRun.value) return '就绪'
      if (isRunning.value) return '运行中...'
      return lastRunSuccess.value ? '运行成功' : '运行失败'
    })

    // 更新编辑器统计信息
    const updateEditorStats = () => {
      if (editor.value) {
        const doc = editor.value.state.doc
        editorStats.value = {
          lines: doc.lines,
          chars: doc.length
        }
      }
    }

    // 初始化编辑器
    const initEditor = () => {
      if (!editorRef.value) return

      // 清理旧编辑器
      if (editor.value) {
        editor.value.destroy()
      }

      const state = EditorState.create({
        doc: defaultCode[currentLanguage.value],
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
            ...lintKeymap,
            {
              key: 'Ctrl-Enter',
              run: () => {
                runCode()
                return true
              }
            },
            {
              key: 'Shift-Alt-f',
              run: () => {
                formatCode()
                return true
              }
            }
          ]),
          EditorView.updateListener.of((update) => {
            if (update.docChanged) {
              updateEditorStats()
            }
          })
        ]
      })

      editor.value = new EditorView({
        state,
        parent: editorRef.value
      })

      // 初始化统计信息
      updateEditorStats()

      // 设置焦点
      nextTick(() => {
        if (editor.value) {
          editor.value.focus()
        }
      })
    }

    // 切换语言
    const changeLanguage = () => {
      if (editor.value) {
        const currentCode = editor.value.state.doc.toString()
        const isEmpty = !currentCode.trim() || currentCode === defaultCode[Object.keys(defaultCode).find(key => defaultCode[key] === currentCode)]
        
        if (isEmpty) {
          // 如果是空的或者是默认代码，则加载新语言的默认代码
          editor.value.dispatch({
            changes: {
              from: 0,
              to: editor.value.state.doc.length,
              insert: defaultCode[currentLanguage.value]
            }
          })
        }
        
        // 更新语言扩展
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
              ...lintKeymap,
              {
                key: 'Ctrl-Enter',
                run: () => {
                  runCode()
                  return true
                }
              },
              {
                key: 'Shift-Alt-f',
                run: () => {
                  formatCode()
                  return true
                }
              }
            ]),
            EditorView.updateListener.of((update) => {
              if (update.docChanged) {
                updateEditorStats()
              }
            })
          ])
        })
        
        updateEditorStats()
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
        const originalError = console.error
        const originalWarn = console.warn
        
        // 重写 console 方法
        console.log = (...args) => {
          logs.push(args.map(arg => 
            typeof arg === 'object' ? JSON.stringify(arg, null, 2) : String(arg)
          ).join(' '))
        }
        
        console.error = (...args) => {
          logs.push('ERROR: ' + args.map(arg => 
            typeof arg === 'object' ? JSON.stringify(arg, null, 2) : String(arg)
          ).join(' '))
        }
        
        console.warn = (...args) => {
          logs.push('WARN: ' + args.map(arg => 
            typeof arg === 'object' ? JSON.stringify(arg, null, 2) : String(arg)
          ).join(' '))
        }
        
        try {
          // 使用 Function 构造器执行代码
          const func = new Function(code)
          const result = func()
          
          if (result !== undefined) {
            logs.push('返回值: ' + (typeof result === 'object' ? JSON.stringify(result, null, 2) : String(result)))
          }
        } catch (error) {
          logs.push('执行错误: ' + error.message)
        } finally {
          // 恢复原始 console 方法
          console.log = originalLog
          console.error = originalError
          console.warn = originalWarn
        }
        
        resolve(logs.length > 0 ? logs.join('\n') : '代码执行完成，无输出')
      })
    }

    // Python 执行 (模拟)
    const runPython = async (code) => {
      // 这里是模拟的 Python 执行，实际项目中需要后端支持
      return new Promise((resolve) => {
        setTimeout(() => {
          if (code.includes('print(')) {
            const matches = code.match(/print\(([^)]+)\)/g)
            if (matches) {
              const outputs = matches.map(match => {
                const content = match.match(/print\(([^)]+)\)/)[1]
                // 简单的字符串处理
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

    // HTML 执行
    const runHTML = async (code) => {
      return new Promise((resolve) => {
        try {
          // 创建一个新窗口来显示 HTML
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
          // 创建一个包含 CSS 的 HTML 页面
          const htmlContent = `
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS 预览</title>
    <style>
${code}
    </style>
</head>
<body>
    <div class="container">
        <div class="card">
            <h2>CSS 样式预览</h2>
            <p>这是一个演示段落，用于展示 CSS 样式效果。</p>
            <button class="btn">示例按钮</button>
        </div>
        <div class="card">
            <h2>另一个卡片</h2>
            <p>您可以在左侧编辑器中修改 CSS 代码，然后点击运行查看效果。</p>
            <button class="btn">另一个按钮</button>
        </div>
    </div>
</body>
</html>`
          
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
            // 简单的 JavaScript 格式化
            formattedCode = formatJavaScript(code)
            break
          case 'html':
            formattedCode = formatHTML(code)
            break
          case 'css':
            formattedCode = formatCSS(code)
            break
          default:
            // 对于 Python 等其他语言，只做基本的缩进整理
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

    // 简单的 JavaScript 格式化
    const formatJavaScript = (code) => {
      let formatted = code
      // 在 { 后添加换行
      formatted = formatted.replace(/{/g, '{\n')
      // 在 } 前添加换行
      formatted = formatted.replace(/}/g, '\n}')
      // 在 ; 后添加换行
      formatted = formatted.replace(/;/g, ';\n')
      // 清理多余的空行
      formatted = formatted.replace(/\n\s*\n/g, '\n')
      return formatted.trim()
    }

    // 简单的 HTML 格式化
    const formatHTML = (code) => {
      let formatted = code
      // 在标签后添加换行
      formatted = formatted.replace(/></g, '>\n<')
      // 简单的缩进处理
      const lines = formatted.split('\n')
      let indent = 0
      return lines.map(line => {
        const trimmed = line.trim()
        if (trimmed.startsWith('</')) indent--
        const result = '  '.repeat(Math.max(0, indent)) + trimmed
        if (trimmed.startsWith('<') && !trimmed.startsWith('</') && !trimmed.endsWith('/>'))
          indent++
        return result
      }).join('\n')
    }

    // 简单的 CSS 格式化
    const formatCSS = (code) => {
      let formatted = code
      // 在 { 后添加换行
      formatted = formatted.replace(/{/g, ' {\n')
      // 在 } 后添加换行
      formatted = formatted.replace(/}/g, '\n}\n')
      // 在 ; 后添加换行
      formatted = formatted.replace(/;/g, ';\n')
      // 简单的缩进处理
      const lines = formatted.split('\n')
      return lines.map(line => {
        const trimmed = line.trim()
        if (trimmed.includes(':') && !trimmed.includes('{') && !trimmed.includes('}')) {
          return '  ' + trimmed
        }
        return trimmed
      }).filter(line => line.length > 0).join('\n')
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
            insert: defaultCode[currentLanguage.value]
          }
        })
        updateEditorStats()
      }
    }

    // 组件挂载
    onMounted(() => {
      initEditor()
    })

    return {
      editorRef,
      currentLanguage,
      currentFile,
      output,
      isRunning,
      lastRunSuccess,
      hasRun,
      runStatus,
      editorStats,
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

/* 工具栏样式 */
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 20px;
  background: #2d2d2d;
  border-bottom: 1px solid #444;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  z-index: 100;
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

.status-info {
  display: flex;
  align-items: center;
}

.action-buttons {
  display: flex;
  gap: 8px;
}

.btn-sm {
  padding: 4px 8px;
  font-size: 12px;
}

/* 主要内容区域 */
.main-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

/* 面板样式 */
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

.editor-info,
.output-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.line-info {
  font-size: 12px;
  color: #888;
}

/* 编辑器容器 */
.editor-container {
  flex: 1;
  overflow: hidden;
}

.code-editor {
  height: 100%;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
}

/* 输出容器 */
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

/* 响应式设计 */
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