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
        <!-- 语言选择 -->
        <div class="control-group">
          <label for="language">语言:</label>
          <select 
            id="language" 
            v-model="currentLanguage" 
            @change="changeLanguage"
            class="select-primary"
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
        
        <!-- 设置面板切换 -->
        <button 
          class="btn btn-ghost settings-toggle"
          @click="showSettings = !showSettings"
          :class="{ active: showSettings }"
        >
          ⚙️ 设置
        </button>
      </div>
      
      <div class="toolbar-right">
        <!-- 主要操作按钮 -->
        <div class="primary-actions">
          <button 
            class="btn btn-success btn-large" 
            @click="runCode" 
            :disabled="isRunning"
          >
            <span v-if="isRunning">⚡</span>
            <span v-else>▶️</span>
            {{ isRunning ? '运行中...' : '运行' }}
          </button>
        </div>
        
        <!-- 次要操作按钮 -->
        <div class="secondary-actions">
          <button 
            class="btn btn-info btn-compact"
            @click="autoTypeOutput"
            :disabled="isAutoTyping"
            title="自动化输出"
          >
            <span v-if="isAutoTyping">⌨️</span>
            <span v-else>🤖</span>
          </button>
          
          <button 
            class="btn btn-secondary btn-compact"
            @click="generateCodeImage"
            :disabled="isGeneratingImage"
            title="生成图片"
          >
            <span v-if="isGeneratingImage">📷</span>
            <span v-else>📸</span>
          </button>
          
          <div class="btn-group">
            <button 
              class="btn btn-warning btn-compact" 
              @click="formatCode"
              title="格式化代码"
            >
              🎨
            </button>
            
            <button 
              class="btn btn-secondary btn-compact" 
              @click="clearOutput"
              title="清空输出"
            >
              🗑️
            </button>
            
            <button 
              class="btn btn-primary btn-compact" 
              @click="resetCode"
              title="重置代码"
            >
              🔄
            </button>
          </div>
        </div>
      </div>
    </header>
    
    <!-- 设置面板 -->
    <div v-if="showSettings" class="settings-panel">
      <div class="settings-content">
        <div class="settings-group">
          <h4>🎨 编辑器设置</h4>
          <div class="settings-row">
            <div class="setting-item">
              <label for="fontSize">字体大小:</label>
              <select 
                id="fontSize" 
                v-model="fontSize"
                @change="updateEditorStyle"
                class="select-compact"
              >
                <option value="12">12px</option>
                <option value="14">14px</option>
                <option value="16">16px</option>
                <option value="18">18px</option>
                <option value="20">20px</option>
              </select>
            </div>
            
            <div class="setting-item">
              <label for="lineHeight">行高:</label>
              <select 
                id="lineHeight" 
                v-model="lineHeight"
                @change="updateEditorStyle"
                class="select-compact"
              >
                <option value="1.4">1.4</option>
                <option value="1.6">1.6</option>
                <option value="1.8">1.8</option>
                <option value="2.0">2.0</option>
                <option value="2.2">2.2</option>
              </select>
            </div>
          </div>
        </div>
        
        <div class="settings-group">
          <h4>⚡ 自动化设置</h4>
          <div class="settings-row">
            <div class="setting-item">
              <label for="speed">输出速度:</label>
              <select 
                id="speed" 
                v-model="autoTypeSpeed"
                class="select-compact"
              >
                <option value="normal">正常</option>
                <option value="fast">快速</option>
                <option value="very-fast">非常快</option>
              </select>
            </div>
          </div>
        </div>
      </div>
    </div>
    
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
    
    <!-- 图片预览弹窗 -->
    <div v-if="showImageModal" class="modal-overlay" @click="closeImageModal">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h3>代码截图预览</h3>
          <button class="close-btn" @click="closeImageModal">×</button>
        </div>
        <div class="modal-body">
          <img v-if="generatedImageUrl" :src="generatedImageUrl" alt="代码截图" class="code-image" />
        </div>
        <div class="modal-footer">
          <button class="btn btn-primary" @click="downloadImage">📥 下载图片</button>
          <button class="btn btn-secondary" @click="closeImageModal">关闭</button>
        </div>
      </div>
    </div>
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
    const isAutoTyping = ref(false)
    const autoTypeSpeed = ref('normal')
    const fontSize = ref('16')
    const lineHeight = ref('1.8')
    const lastRunSuccess = ref(true)
    const hasRun = ref(false)
    const isGeneratingImage = ref(false)
    const showImageModal = ref(false)
    const generatedImageUrl = ref('')
    const showSettings = ref(false)

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

        // 应用初始样式
        setTimeout(() => {
          updateEditorStyle()
        }, 100)

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

    // 更新编辑器样式
    const updateEditorStyle = () => {
      if (editor.value) {
        const editorElement = editor.value.dom
        const scroller = editorElement.querySelector('.cm-scroller')
        if (scroller) {
          scroller.style.fontSize = fontSize.value + 'px'
          scroller.style.lineHeight = lineHeight.value
        }
        
        // 更新所有行的样式
        const lines = editorElement.querySelectorAll('.cm-line')
        lines.forEach(line => {
          line.style.lineHeight = lineHeight.value
        })
      }
    }

    // 生成代码图片
    const generateCodeImage = async () => {
      if (isGeneratingImage.value) return
      
      isGeneratingImage.value = true
      
      try {
        // 获取编辑器内容
        const codeContent = editor.value.state.doc.toString()
        const lines = codeContent.split('\n')
        
        // 创建高分辨率canvas
        const canvas = document.createElement('canvas')
        const ctx = canvas.getContext('2d')
        
        // 获取设备像素比，提高清晰度
        const dpr = window.devicePixelRatio || 2
        
        // 设置画布样式（高分辨率）
        const lineHeight = 32 * dpr
        const padding = 30 * dpr
        const lineNumberWidth = 80 * dpr
        const fontSize = 16 * dpr
        const baseWidth = 1200
        const baseHeight = Math.max(600, lines.length * (lineHeight / dpr) + (padding * 2 / dpr))
        
        // 设置canvas实际尺寸（高分辨率）
        canvas.width = baseWidth * dpr
        canvas.height = baseHeight * dpr
        
        // 设置canvas显示尺寸
        canvas.style.width = baseWidth + 'px'
        canvas.style.height = baseHeight + 'px'
        
        // 缩放绘图上下文以匹配设备像素比
        ctx.scale(dpr, dpr)
        
        // 启用抗锯齿
        ctx.imageSmoothingEnabled = true
        ctx.imageSmoothingQuality = 'high'
        
        // 设置背景
        ctx.fillStyle = '#1e1e1e'
        ctx.fillRect(0, 0, baseWidth, baseHeight)
        
        // 设置字体（使用更大的字体以提高清晰度）
        ctx.font = `${fontSize / dpr}px 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', 'Consolas', monospace`
        ctx.textBaseline = 'middle'
        
        // 绘制行号和代码
        lines.forEach((line, index) => {
          const y = (padding / dpr) + (index + 1) * (lineHeight / dpr)
          
          // 绘制行号背景
          ctx.fillStyle = '#2d2d2d'
          ctx.fillRect(0, y - (lineHeight / dpr / 2), lineNumberWidth / dpr, lineHeight / dpr)
          
          // 绘制行号
          ctx.fillStyle = '#858585'
          ctx.textAlign = 'right'
          ctx.fillText((index + 1).toString(), (lineNumberWidth / dpr) - 15, y)
          
          // 绘制分隔线
          ctx.fillStyle = '#444444'
          ctx.fillRect(lineNumberWidth / dpr, y - (lineHeight / dpr / 2), 1, lineHeight / dpr)
          
          // 绘制代码
          ctx.fillStyle = '#d4d4d4'
          ctx.textAlign = 'left'
          ctx.fillText(line, (lineNumberWidth / dpr) + 15, y)
        })
        
        // 转换为高质量图片URL
        generatedImageUrl.value = canvas.toDataURL('image/png', 1.0)
        showImageModal.value = true
        
      } catch (error) {
        console.error('生成图片失败:', error)
        alert('生成图片失败，请重试')
      } finally {
        isGeneratingImage.value = false
      }
    }

    // 关闭图片预览弹窗
    const closeImageModal = () => {
      showImageModal.value = false
      generatedImageUrl.value = ''
    }

    // 下载图片
    const downloadImage = () => {
      if (!generatedImageUrl.value) return
      
      const link = document.createElement('a')
      link.download = `code-screenshot-${new Date().getTime()}.png`
      link.href = generatedImageUrl.value
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    }

    const autoTypeOutput = async () => {
      if (!editor.value || isAutoTyping.value) return

      const code = editor.value.state.doc.toString()
      if (!code.trim()) {
        // 如果编辑器为空，直接返回
        return
      }

      isAutoTyping.value = true
      
      try {
        // 根据速度设置获取延迟配置
        const getSpeedConfig = () => {
          switch (autoTypeSpeed.value) {
            case 'normal':
              return { charDelay: [80, 120], lineDelay: [300, 600] }
            case 'fast':
              return { charDelay: [30, 60], lineDelay: [100, 200] }
            case 'very-fast':
              return { charDelay: [5, 15], lineDelay: [20, 50] }
            default:
              return { charDelay: [80, 120], lineDelay: [300, 600] }
          }
        }
        
        const speedConfig = getSpeedConfig()
        
        // 清空编辑器
        editor.value.dispatch({
          changes: {
            from: 0,
            to: editor.value.state.doc.length,
            insert: ''
          }
        })

        // 逐字符输入代码
        let currentPos = 0
        for (let i = 0; i < code.length; i++) {
          if (!isAutoTyping.value) break
          
          const char = code[i]
          
          // 在当前位置插入字符
          editor.value.dispatch({
            changes: {
              from: currentPos,
              to: currentPos,
              insert: char
            }
          })
          
          currentPos++
          
          // 字符间延迟
          const charDelay = Math.random() * (speedConfig.charDelay[1] - speedConfig.charDelay[0]) + speedConfig.charDelay[0]
          await new Promise(resolve => setTimeout(resolve, charDelay))
          
          // 如果是换行符，额外停顿
          if (char === '\n') {
            const lineDelay = Math.random() * (speedConfig.lineDelay[1] - speedConfig.lineDelay[0]) + speedConfig.lineDelay[0]
            await new Promise(resolve => setTimeout(resolve, lineDelay))
          }
        }
        
      } catch (error) {
        console.error('自动化输出错误:', error)
      } finally {
        isAutoTyping.value = false
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
      isAutoTyping,
      autoTypeSpeed,
      fontSize,
      lineHeight,
      isGeneratingImage,
      showImageModal,
      generatedImageUrl,
      showSettings,
      changeLanguage,
      runCode,
      formatCode,
      clearOutput,
      resetCode,
      autoTypeOutput,
      updateEditorStyle,
      generateCodeImage,
      closeImageModal,
      downloadImage
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

.speed-selector {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: 20px;
}

.speed-selector label {
  font-size: 14px;
  font-weight: 500;
}

.speed-selector select {
  padding: 6px 12px;
  background: #3c3c3c;
  color: #ffffff;
  border: 1px solid #555;
  border-radius: 4px;
  font-size: 14px;
}

.speed-selector select:focus {
  outline: none;
  border-color: #007acc;
}

.font-selector {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: 20px;
}

.font-selector label {
  font-size: 14px;
  font-weight: 500;
}

.font-selector select {
  padding: 6px 12px;
  background: #3c3c3c;
  color: #ffffff;
  border: 1px solid #555;
  border-radius: 4px;
  font-size: 14px;
}

.font-selector select:focus {
  outline: none;
  border-color: #007acc;
}

.lineheight-selector {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: 20px;
}

.lineheight-selector label {
  font-size: 14px;
  font-weight: 500;
}

.lineheight-selector select {
  padding: 6px 12px;
  background: #3c3c3c;
  color: #ffffff;
  border: 1px solid #555;
  border-radius: 4px;
  font-size: 14px;
}

.lineheight-selector select:focus {
  outline: none;
  border-color: #007acc;
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

/* 按钮样式 */
.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-primary {
  background: #007acc;
  color: #fff;
}

.btn-primary:hover {
  background: #005a9e;
}

.btn-success {
  background: #28a745;
  color: #fff;
}

.btn-success:hover {
  background: #218838;
}

.btn-success:disabled {
  background: #6c757d;
  cursor: not-allowed;
}

.btn-warning {
  background: #ffc107;
  color: #000;
}

.btn-warning:hover {
  background: #e0a800;
}

.btn-info {
  background: #17a2b8;
  color: #fff;
}

.btn-info:hover {
  background: #138496;
}

.btn-info:disabled {
  background: #6c757d;
  cursor: not-allowed;
}

.btn-secondary {
  background: #6c757d;
  color: #fff;
}

.btn-secondary:hover {
  background: #5a6268;
}

.btn-secondary:disabled {
  background: #495057;
  opacity: 0.6;
  cursor: not-allowed;
}

select {
  padding: 6px 12px;
  background: #3c3c3c;
  color: #ffffff;
  border: 1px solid #555;
  border-radius: 4px;
  font-size: 14px;
}

select:focus {
  outline: none;
  border-color: #007acc;
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
  line-height: 1.8;
}

.code-editor :deep(.cm-line) {
  line-height: 1.8;
  padding: 2px 0;
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

/* 弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: #2d2d2d;
  border-radius: 8px;
  max-width: 90vw;
  max-height: 90vh;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid #444;
}

.modal-header h3 {
  margin: 0;
  color: #ffffff;
  font-size: 18px;
}

.close-btn {
  background: none;
  border: none;
  color: #ffffff;
  font-size: 24px;
  cursor: pointer;
  padding: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.close-btn:hover {
  background: #444;
}

.modal-body {
  padding: 20px;
  max-height: 70vh;
  overflow: auto;
}

.code-image {
  max-width: 100%;
  height: auto;
  border-radius: 4px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.modal-footer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 12px;
  padding: 16px 20px;
  border-top: 1px solid #444;
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