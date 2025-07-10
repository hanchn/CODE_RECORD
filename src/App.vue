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
          
          <button 
            class="btn btn-compact"
            @click="toggleVideoRecording"
            :disabled="isAutoTyping && !isRecording"
            :class="{ 'btn-danger': isRecording, 'btn-secondary': !isRecording }"
            title="录制视频"
          >
            <span v-if="isRecording">⏹️ 停止录制</span>
            <span v-else>🎥 录制视频</span>
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
    
    <!-- IDE水印 -->
    <div class="ide-watermark" v-if="watermarkEnabled && watermarkText">
      <div 
        v-for="(pos, index) in watermarkPositions" 
        :key="index"
        class="watermark-text"
        :style="{
          left: pos.x + 'px',
          top: pos.y + 'px'
        }"
      >
        {{ watermarkText }}
      </div>
    </div>
    

    
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
        
        <div class="settings-group">
          <h4>🏷️ 水印设置</h4>
          <div class="settings-row">
            <div class="setting-item">
              <label>
                <input 
                  type="checkbox" 
                  v-model="watermarkEnabled"
                  class="watermark-checkbox"
                >
                启用水印
              </label>
            </div>
          </div>
          <div class="settings-row" v-if="watermarkEnabled">
            <div class="setting-item">
              <label for="watermarkText">水印文本:</label>
              <input 
                id="watermarkText" 
                type="text" 
                v-model="watermarkText"
                class="watermark-input"
                placeholder="请输入水印文本"
                maxlength="20"
              >
            </div>
          </div>
          <div class="settings-row" v-if="watermarkEnabled" style="margin-top: 12px;">
            <div class="setting-item">
              <label for="watermarkOpacity">透明度:</label>
              <select 
                id="watermarkOpacity" 
                v-model="watermarkOpacity"
                @change="updateWatermarkStyle"
                class="select-compact"
              >
                <option value="0.1">较淡</option>
                <option value="0.25">适中</option>
                <option value="0.4">较浓</option>
              </select>
            </div>
          </div>

        </div>
      </div>
    </div>
    
    <!-- 主要内容区域 -->
    <main class="main-content">
      <!-- 倒计时覆盖层 -->
      <div v-if="isCountingDown" class="countdown-overlay">
        <div class="countdown-content">
          <div class="countdown-number">{{ countdownNumber }}</div>
          <div class="countdown-text">录制即将开始...</div>
        </div>
      </div>
      
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
    
    <!-- 视频预览弹窗 -->
    <div v-if="showVideoModal" class="modal-overlay" @click="closeVideoModal">
      <div class="modal-content video-modal" @click.stop>
        <div class="modal-header">
          <h3>代码录制预览</h3>
          <button class="close-btn" @click="closeVideoModal">×</button>
        </div>
        <div class="modal-body">
          <video v-if="recordedVideoUrl" :src="recordedVideoUrl" controls class="recorded-video">
            您的浏览器不支持视频播放。
          </video>
        </div>
        <div class="modal-footer">
          <button class="btn btn-primary" @click="downloadVideo">📥 下载视频</button>
          <button class="btn btn-secondary" @click="closeVideoModal">关闭</button>
        </div>
      </div>
    </div>
    

  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, computed, nextTick } from 'vue'
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
    const isRecording = ref(false)
    const showVideoModal = ref(false)
    const recordedVideoUrl = ref('')
    const mediaRecorder = ref(null)
    const recordedChunks = ref([])
    const isCountingDown = ref(false)
    const countdownNumber = ref(3)
    const initialCode = ref('')
    const code = ref('')
    
    // 水印相关配置
    const watermarkEnabled = ref(true)
    const watermarkText = ref('AUVWEB')
    const watermarkOpacity = ref('0.25')
    


    // 计算水印位置
    const watermarkPositions = computed(() => {
      if (!watermarkEnabled.value || !watermarkText.value) return []
      
      const positions = []
      const spacing = 400 // 水印间距
      const screenWidth = window.innerWidth
      const screenHeight = window.innerHeight
      
      // 计算需要的行数和列数，确保完全覆盖屏幕
      const cols = Math.ceil(screenWidth / spacing) + 2
      const rows = Math.ceil(screenHeight / spacing) + 2
      
      for (let row = -1; row < rows; row++) {
        for (let col = -1; col < cols; col++) {
          // 奇偶行错峰排列，偶数行向右偏移半个间距
          const offsetX = (row % 2 === 0) ? 0 : spacing / 2
          positions.push({
            x: col * spacing + offsetX,
            y: row * spacing
          })
        }
      }
      
      return positions
    })

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
          parent: editorRef.value,
          updateListener: EditorView.updateListener.of((update) => {
            if (update.docChanged) {
              code.value = update.state.doc.toString()
            }
          })
        })
        
        // 初始化code变量
        code.value = defaultCodes[currentLanguage.value] || ''
        
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
        const newCode = defaultCodes[currentLanguage.value]
        editor.value.dispatch({
          changes: {
            from: 0,
            to: editor.value.state.doc.length,
            insert: newCode
          }
        })
        
        // 同步更新code变量
        code.value = newCode
        
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

    // 更新水印样式
    const updateWatermarkStyle = () => {
      document.documentElement.style.setProperty('--watermark-opacity', watermarkOpacity.value)
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
        const padding = 0  // 完全移除上下边距
        const lineNumberWidth = 80 * dpr
        const fontSize = 16 * dpr
        const baseWidth = 1200
        // 根据实际代码行数计算高度，无边距
        const baseHeight = lines.length * (lineHeight / dpr)
        
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
        
        // 获取编辑器中的高亮样式
        const editorElement = editorRef.value.querySelector('.cm-editor')
        const cmLines = editorElement.querySelectorAll('.cm-line')
        
        // 绘制行号和代码
        lines.forEach((line, index) => {
          const y = (lineHeight / dpr / 2) + index * (lineHeight / dpr)
          
          // 绘制行号背景
          ctx.fillStyle = '#1e1e1e'
          ctx.fillRect(0, y - (lineHeight / dpr / 2), lineNumberWidth / dpr, lineHeight / dpr)
          
          // 绘制行号
          ctx.fillStyle = '#858585'
          ctx.textAlign = 'right'
          ctx.fillText((index + 1).toString(), (lineNumberWidth / dpr) - 15, y)
          
          // 绘制分隔线
          ctx.fillStyle = '#2d2d30'
          ctx.fillRect(lineNumberWidth / dpr, y - (lineHeight / dpr / 2), 1, lineHeight / dpr)
          
          // 绘制代码（带语法高亮）
          ctx.textAlign = 'left'
          let x = (lineNumberWidth / dpr) + 15
          
          if (cmLines[index]) {
            // 重置x位置
            x = (lineNumberWidth / dpr) + 15
            
            // 递归处理节点并绘制文本
            const processNode = (node) => {
              if (node.nodeType === Node.TEXT_NODE) {
                const text = node.textContent
                if (text) {
                  ctx.fillText(text, x, y)
                  x += ctx.measureText(text).width
                }
              } else if (node.nodeType === Node.ELEMENT_NODE) {
                // 根据CSS类名设置颜色
                const className = node.className || ''
                let color = '#d4d4d4' // 默认颜色
                
                if (className.includes('tok-keyword')) {
                  color = '#569cd6' // 关键字 - 蓝色
                } else if (className.includes('tok-string')) {
                  color = '#ce9178' // 字符串 - 橙色
                } else if (className.includes('tok-comment')) {
                  color = '#6a9955' // 注释 - 绿色
                } else if (className.includes('tok-number')) {
                  color = '#b5cea8' // 数字 - 浅绿色
                } else if (className.includes('tok-operator')) {
                  color = '#d4d4d4' // 操作符 - 白色
                } else if (className.includes('tok-variableName')) {
                  color = '#9cdcfe' // 变量名 - 浅蓝色
                } else if (className.includes('tok-typeName')) {
                  color = '#4ec9b0' // 类型名 - 青色
                } else if (className.includes('tok-function')) {
                  color = '#dcdcaa' // 函数名 - 黄色
                } else if (className.includes('tok-punctuation')) {
                  color = '#d4d4d4' // 标点符号
                } else if (className.includes('tok-bracket')) {
                  color = '#ffd700' // 括号 - 金色
                }
                
                ctx.fillStyle = color
                
                // 递归处理子节点
                for (let child of node.childNodes) {
                  processNode(child)
                }
              }
            }
            
            // 处理整行的所有子节点
            ctx.fillStyle = '#d4d4d4' // 设置默认颜色
            for (let child of cmLines[index].childNodes) {
              processNode(child)
            }
          } else {
            // 如果没有对应的编辑器行，使用默认颜色绘制原始文本
            ctx.fillStyle = '#d4d4d4'
            ctx.fillText(line, x, y)
          }
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

    // 开始/停止视频录制
    const toggleVideoRecording = async () => {
      if (isRecording.value) {
        stopRecording()
      } else {
        await startRecording()
      }
    }





    // 开始录制
    const startRecording = async () => {
      try {
        // 检查浏览器兼容性
        if (!navigator.mediaDevices || !navigator.mediaDevices.getDisplayMedia) {
          alert('您的浏览器不支持屏幕录制功能，请使用最新版本的 Chrome、Firefox 或 Edge 浏览器。')
          return
        }
        
        // 提前提示用户录制选项
        const userChoice = confirm(
          '即将开始屏幕录制。\n\n' +
          '在弹出的屏幕共享对话框中，您可以选择：\n' +
          '• 整个屏幕 - 录制完整桌面\n' +
          '• 窗口 - 录制特定应用窗口\n' +
          '• 标签页 - 仅录制当前浏览器标签\n\n' +
          '建议选择"窗口"并选择浏览器窗口以获得最佳效果。\n\n' +
          '是否继续？'
        )
        
        if (!userChoice) {
          return
        }
        
        // 请求屏幕录制权限，提供更多选项
        const stream = await navigator.mediaDevices.getDisplayMedia({
          video: {
            mediaSource: 'screen', // 允许用户选择屏幕、窗口或标签页
            width: { ideal: 1920, max: 1920 },
            height: { ideal: 1080, max: 1080 },
            frameRate: { ideal: 30, max: 60 }
          },
          audio: {
            echoCancellation: true,
            noiseSuppression: true,
            sampleRate: 44100
          }
        })
        
        // 动态检测支持的MIME类型
        const getSupportedMimeType = () => {
          const types = [
            'video/webm;codecs=vp9,opus',
            'video/webm;codecs=vp8,opus',
            'video/webm;codecs=vp9',
            'video/webm;codecs=vp8',
            'video/webm',
            'video/mp4'
          ]
          
          for (const type of types) {
            if (MediaRecorder.isTypeSupported(type)) {
              return type
            }
          }
          return 'video/webm' // 默认回退
        }
        
        const mimeType = getSupportedMimeType()
        mediaRecorder.value = new MediaRecorder(stream, {
          mimeType: mimeType,
          videoBitsPerSecond: 2500000, // 2.5 Mbps
          audioBitsPerSecond: 128000   // 128 kbps
        })
        
        recordedChunks.value = []
        
        mediaRecorder.value.ondataavailable = (event) => {
          if (event.data.size > 0) {
            recordedChunks.value.push(event.data)
          }
        }
        
        mediaRecorder.value.onstop = () => {
          const blob = new Blob(recordedChunks.value, { type: mimeType })
          recordedVideoUrl.value = URL.createObjectURL(blob)
          
          // 录制完成后恢复界面显示
          restoreUIAfterRecording()
          
          // 恢复代码到初始状态
          if (initialCode.value && editor.value) {
            code.value = initialCode.value
            editor.value.dispatch({
              changes: {
                from: 0,
                to: editor.value.state.doc.length,
                insert: initialCode.value
              }
            })
          }
          
          // 自动弹出预览窗口
          showVideoModal.value = true
          isRecording.value = false
          
          // 停止所有轨道
          stream.getTracks().forEach(track => track.stop())
        }
        
        // 监听用户手动停止屏幕共享
        stream.getVideoTracks()[0].addEventListener('ended', () => {
          if (isRecording.value) {
            stopRecording()
          }
        })
        
        // 保存当前代码状态
        if (editor.value) {
          initialCode.value = editor.value.state.doc.toString()
        }
        
        // 隐藏界面元素
        hideUIForRecording()
        
        // 清空编辑器内容
        if (editor.value) {
          editor.value.dispatch({
            changes: {
              from: 0,
              to: editor.value.state.doc.length,
              insert: ''
            }
          })
          code.value = ''
        }
        
        // 开始3秒倒计时
        isCountingDown.value = true
        countdownNumber.value = 3
        isRecording.value = true // 提前设置录制状态以显示水印
        
        // 倒计时逻辑
        const countdown = () => {
          if (countdownNumber.value > 0) {
            setTimeout(() => {
              countdownNumber.value--
              if (countdownNumber.value > 0) {
                countdown()
              } else {
                // 倒计时结束，开始实际录制
                isCountingDown.value = false
                mediaRecorder.value.start(1000) // 每秒收集一次数据
                
                // 延迟开始自动化输出，给用户时间准备
                setTimeout(() => {
                  if (isRecording.value) {
                    autoTypeOutput()
                  }
                }, 1000) // 录制开始后1秒开始自动输出
              }
            }, 1000)
          }
        }
        
        countdown()
        
      } catch (error) {
        console.error('录制启动失败:', error)
        
        // 提供更友好的错误信息
        if (error.name === 'NotAllowedError') {
          alert('屏幕录制权限被拒绝。请刷新页面并允许屏幕共享权限。')
        } else if (error.name === 'NotSupportedError') {
          alert('您的浏览器不支持屏幕录制功能。请使用最新版本的 Chrome、Firefox 或 Edge 浏览器。')
        } else {
          alert(`录制启动失败: ${error.message}`)
        }
        
        isRecording.value = false
        restoreUIAfterRecording()
      }
    }

    // 停止录制
    const stopRecording = () => {
      if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') {
        mediaRecorder.value.stop()
a        
        // 停止所有轨道
        mediaRecorder.value.stream.getTracks().forEach(track => {
          track.stop()
        })
      }
      
      // 立即恢复界面（如果录制被手动停止）
      if (isRecording.value) {
        isRecording.value = false
        restoreUIAfterRecording()
        
        // 恢复代码到初始状态
        if (initialCode.value && editor.value) {
          code.value = initialCode.value
          editor.value.setValue(initialCode.value)
        }
      }
    }

    // 关闭视频预览弹窗
    const closeVideoModal = () => {
      showVideoModal.value = false
      if (recordedVideoUrl.value) {
        URL.revokeObjectURL(recordedVideoUrl.value)
        recordedVideoUrl.value = ''
      }
    }

    // 下载视频
    const downloadVideo = () => {
      if (!recordedVideoUrl.value) return
      
      const link = document.createElement('a')
      link.download = `code-recording-${new Date().getTime()}.webm`
      link.href = recordedVideoUrl.value
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    }

    // 录制时隐藏界面元素
    const hideUIForRecording = () => {
      // 隐藏设置面板
      showSettings.value = false
      
      // 隐藏整个顶部工具栏
      const toolbar = document.querySelector('.toolbar')
      if (toolbar) {
        toolbar.style.display = 'none'
      }
      
      // 隐藏右侧输出面板
      const outputPanel = document.querySelector('.output-panel')
      if (outputPanel) {
        outputPanel.style.display = 'none'
      }
      // 隐藏编辑器面板的标题
      const editorPanelHeader = document.querySelector('.editor-panel .panel-header')
      if (editorPanelHeader) {
        editorPanelHeader.style.display = 'none'
      }
      
      // 调整编辑器面板宽度占满整个容器
      const editorPanel = document.querySelector('.editor-panel')
      if (editorPanel) {
        editorPanel.style.width = '100%'
        editorPanel.style.flex = '1'
      }
      
      // 调整主内容区域，移除顶部边距
      const mainContent = document.querySelector('.main-content')
      if (mainContent) {
        mainContent.style.paddingTop = '0'
        mainContent.style.height = '100vh'
      }
    }

    // 录制完成后恢复界面元素
    const restoreUIAfterRecording = () => {
      // 恢复顶部工具栏
      const toolbar = document.querySelector('.toolbar')
      if (toolbar) {
        toolbar.style.display = ''
      }
      
      // 恢复右侧输出面板
      const outputPanel = document.querySelector('.output-panel')
      if (outputPanel) {
        outputPanel.style.display = ''
      }
      
      // 恢复编辑器面板的标题
      const editorPanelHeader = document.querySelector('.editor-panel .panel-header')
      if (editorPanelHeader) {
        editorPanelHeader.style.display = ''
      }
      
      // 恢复编辑器面板原始样式
      const editorPanel = document.querySelector('.editor-panel')
      if (editorPanel) {
        editorPanel.style.width = ''
        editorPanel.style.flex = ''
      }
      
      // 恢复主内容区域原始样式
      const mainContent = document.querySelector('.main-content')
      if (mainContent) {
        mainContent.style.paddingTop = ''
        mainContent.style.height = ''
      }
    }

    const autoTypeOutput = async () => {
      if (!editor.value || isAutoTyping.value) return

      // 在录制时使用保存的初始代码，否则使用当前编辑器内容
      const codeContent = isRecording.value && initialCode.value ? initialCode.value : editor.value.state.doc.toString()
      if (!codeContent.trim()) {
        // 如果没有代码内容，直接返回
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
        for (let i = 0; i < codeContent.length; i++) {
          if (!isAutoTyping.value) break
          
          const char = codeContent[i]
           
          // 在当前位置插入字符并更新光标位置
          editor.value.dispatch({
            changes: {
              from: currentPos,
              to: currentPos,
              insert: char
            },
            selection: {
              anchor: currentPos + 1,
              head: currentPos + 1
            },
            scrollIntoView: true
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

    // ESC键取消录制功能
    const handleEscapeKey = (event) => {
      if (event.key === 'Escape' && (isRecording.value || isCountingDown.value)) {
        event.preventDefault()
        const confirmCancel = confirm(
          '是否要取消当前录制？\n\n' +
          '取消录制将丢失已录制的内容，无法恢复。\n\n' +
          '点击"确定"取消录制，点击"取消"继续录制。'
        )
        
        if (confirmCancel) {
          // 停止倒计时
          isCountingDown.value = false
          
          // 停止自动输出
          isAutoTyping.value = false
          
          // 恢复到录制开始前的代码状态
          if (initialCode.value && editor.value) {
            code.value = initialCode.value
            editor.value.dispatch({
              changes: {
                from: 0,
                to: editor.value.state.doc.length,
                insert: initialCode.value
              }
            })
          }
          
          // 停止录制（如果已经开始录制）
          if (isRecording.value) {
            stopRecording()
          } else {
            // 如果只是在倒计时阶段，恢复界面
            restoreUIAfterRecording()
          }
        }
      }
    }

    // 组件挂载
    onMounted(() => {
      console.log('组件已挂载')
      
      // 添加键盘事件监听器
      document.addEventListener('keydown', handleEscapeKey)
      
      // 初始化水印样式
      updateWatermarkStyle()
      
      // 延迟初始化编辑器，确保DOM完全渲染
      nextTick(() => {
        setTimeout(() => {
          initEditor()
        }, 100)
      })
    })
    
    // 组件卸载时清理事件监听器
    onUnmounted(() => {
      document.removeEventListener('keydown', handleEscapeKey)
    })

    return {
      editorRef,
      currentLanguage,
      currentFile,
      code,
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
      isRecording,
      showVideoModal,
      recordedVideoUrl,
      isCountingDown,
      countdownNumber,
      watermarkEnabled,
      watermarkText,
      watermarkOpacity,
      watermarkPositions,

      changeLanguage,
      runCode,
      formatCode,
      clearOutput,
      resetCode,
      autoTypeOutput,
      updateEditorStyle,
      updateWatermarkStyle,
      generateCodeImage,
      closeImageModal,
      downloadImage,
      toggleVideoRecording,
      closeVideoModal,
      downloadVideo
    }
  }
}
</script>

<style src="./App.css" scoped></style>

