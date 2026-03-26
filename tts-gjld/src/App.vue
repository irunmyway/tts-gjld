<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue'

const API_BASE = 'https://api.siliconflow.cn/v1'
const MODELS = ['FunAudioLLM/CosyVoice2-0.5B', 'fnlp/MOSS-TTSD-v0.5']
const DEFAULT_REFERENCE_TEXT = '今晚的风很温柔，穿过树梢，带来一点清新的凉意。灯光在窗前摇晃，深呼吸，冷静一点。接下来该怎么做？我放慢脚步，听心跳与呼吸，感受此刻的安静与从容。'

const backgrounds = ['/pic/1.jpg', '/pic/2.jpg', '/pic/3.jpg', '/pic/4.jpg', '/pic/5.jpg', '/pic/6.jpg']
const bgIndex = ref(0)
const bgImage = ref('')

const showMarket = ref(false)
const prefill = ref(null)

const apiKey = ref(localStorage.getItem('sf_api_key') || '')
const showApiKey = ref(false)
const selectedModel = ref('FunAudioLLM/CosyVoice2-0.5B')
const selectedFile = ref(null)
const selectedFileName = ref('')
const customName = ref('')
const referenceText = ref('')

const statusText = ref('就绪')
const loadingUpload = ref(false)
const loadingList = ref(false)

const uploadResult = reactive({
  customName: '',
  uri: ''
})

const voiceList = ref([])

const toast = reactive({
  show: false,
  message: '',
  type: 'success'
})
let toastTimer = null

const balance = reactive({
  loading: false,
  value: null,
  error: ''
})

const showPreviewModal = ref(false)
const previewForm = reactive({
  model: 'FunAudioLLM/CosyVoice2-0.5B',
  input: '',
  voice: '',
  response_format: 'mp3',
  sample_rate: 44100,
  stream: false,
  speed: 1,
  gain: 0,
  max_tokens: 2048,
  referencesText: ''
})
const previewState = reactive({
  loading: false,
  error: '',
  audioUrl: ''
})
let previewObjectUrl = null

const showSynthesisErrorModal = ref(false)
const synthesisErrorDetail = ref('')

const marketList = ref([])
const marketState = reactive({
  uploading: false,
  status: '',
  error: ''
})

const maskedApiInputType = computed(() => (showApiKey.value ? 'text' : 'password'))
const pageBgStyle = computed(() => ({ backgroundImage: bgImage.value ? `url(${bgImage.value})` : '' }))

function getAuthHeaders(extra = {}) {
  return {
    Authorization: `Bearer ${apiKey.value.trim()}`,
    ...extra
  }
}

async function parseErrorMessage(response) {
  const text = await response.text()
  if (!text) {
    return `HTTP ${response.status}`
  }
  try {
    const data = JSON.parse(text)
    return data?.message || data?.error?.message || data?.error || text
  } catch {
    return text
  }
}

async function requestApi(path, options = {}) {
  const response = await fetch(`${API_BASE}${path}`, options)
  if (!response.ok) {
    throw new Error(await parseErrorMessage(response))
  }
  return response
}

function showToast(message, type = 'success') {
  if (toastTimer) {
    clearTimeout(toastTimer)
  }
  toast.message = message
  toast.type = type
  toast.show = true
  toastTimer = setTimeout(() => {
    toast.show = false
  }, 3000)
}


function changeBackground() {
  bgIndex.value = (bgIndex.value + 1) % backgrounds.length
  bgImage.value = backgrounds[bgIndex.value]
}

function onChooseFile(event) {
  const file = event.target.files?.[0]
  if (!file) {
    return
  }
  selectedFile.value = file
  selectedFileName.value = file.name
  if (!customName.value) {
    customName.value = file.name.replace(/\.[^/.]+$/, '')
  }
}

function saveApiKey() {
  const value = apiKey.value.trim()
  if (!value) {
    statusText.value = '请输入 API Key'
    return
  }
  if (!value.startsWith('sk-')) {
    statusText.value = 'API Key 格式错误，应以 sk- 开头'
    showToast('API Key 格式错误，应以 sk- 开头', 'error')
    return
  }

  localStorage.setItem('sf_api_key', value)
  statusText.value = '配置已保存'
  showToast('API Key 配置已保存')
}

async function refreshBalance() {
  balance.error = ''
  if (!apiKey.value.trim()) {
    balance.value = null
    statusText.value = '请先配置 API Key'
    showToast('请先配置 API Key', 'error')
    return
  }

  balance.loading = true
  try {
    const response = await requestApi('/user/info', {
      headers: getAuthHeaders()
    })
    const data = await response.json()
    const value = data?.data?.balance ?? data?.data?.totalBalance ?? null
    balance.value = value != null ? String(value) : null
  } catch (error) {
    balance.error = error.message || String(error)
    statusText.value = `获取余额失败: ${balance.error}`
    showToast('获取余额失败', 'error')
  } finally {
    balance.loading = false
  }
}

async function uploadVoice() {
  if (!apiKey.value.trim()) {
    statusText.value = '请先配置 API Key'
    return
  }
  if (!selectedFile.value) {
    statusText.value = '请选择音频文件'
    return
  }
  if (!customName.value.trim()) {
    statusText.value = '请输入音色名称'
    return
  }
  if (!referenceText.value.trim()) {
    statusText.value = '请输入参考文本'
    return
  }

  loadingUpload.value = true
  statusText.value = '正在上传...'

  try {
    const formData = new FormData()
    formData.append('file', selectedFile.value)
    formData.append('model', selectedModel.value)
    formData.append('customName', customName.value.trim())
    formData.append('text', referenceText.value.trim())

    const response = await requestApi('/uploads/audio/voice', {
      method: 'POST',
      headers: getAuthHeaders(),
      body: formData
    })

    const data = await response.json()
    uploadResult.customName = data.customName || data?.data?.customName || customName.value
    uploadResult.uri = data.uri || data?.data?.uri || 'N/A'

    statusText.value = '上传成功'
    showToast('音色上传成功！')

    selectedFile.value = null
    selectedFileName.value = ''
    customName.value = ''
    referenceText.value = ''

    setTimeout(() => {
      refreshVoiceList()
    }, 1000)
  } catch (error) {
    statusText.value = `上传失败: ${error.message || error}`
    showToast('音色上传失败', 'error')
  } finally {
    loadingUpload.value = false
  }
}

async function refreshVoiceList() {
  if (!apiKey.value.trim()) {
    statusText.value = '请先配置 API Key'
    return
  }

  loadingList.value = true
  statusText.value = '正在获取音色列表...'

  try {
    const response = await requestApi('/audio/voice/list', {
      headers: getAuthHeaders({
        'Content-Type': 'application/json'
      })
    })

    const data = await response.json()
    let list = []

    if (Array.isArray(data)) {
      list = data
    } else if (Array.isArray(data?.result)) {
      list = data.result
    } else if (Array.isArray(data?.data)) {
      list = data.data
    } else if (Array.isArray(data?.data?.items)) {
      list = data.data.items
    } else {
      throw new Error('API 返回格式不支持')
    }

    voiceList.value = list
    statusText.value = `音色列表已刷新，共 ${list.length} 个音色`
  } catch (error) {
    statusText.value = `获取音色列表失败: ${error.message || error}`
    showToast('获取音色列表失败', 'error')
  } finally {
    loadingList.value = false
  }
}

async function refreshAll() {
  await Promise.all([refreshVoiceList(), refreshBalance()])
}

async function deleteVoice(uri) {
  if (!apiKey.value.trim()) {
    statusText.value = '请先配置 API Key'
    return
  }
  if (!uri) {
    statusText.value = '无效的 URI'
    return
  }

  statusText.value = '正在删除...'
  try {
    const response = await requestApi('/audio/voice/deletions', {
      method: 'POST',
      headers: getAuthHeaders({
        'Content-Type': 'application/json'
      }),
      body: JSON.stringify({ uri })
    })

    await refreshVoiceList()
    statusText.value = '删除成功'
    showToast('音色删除成功')
  } catch (error) {
    statusText.value = `删除失败: ${error.message || error}`
    showToast('音色删除失败', 'error')
  }
}

async function copyText(value) {
  const text = value != null ? String(value) : ''
  if (!text.trim()) {
    statusText.value = '暂无可复制内容'
    showToast('暂无可复制内容', 'error')
    return
  }

  try {
    await navigator.clipboard.writeText(text)
    statusText.value = '已复制到剪贴板'
    showToast('已复制到剪贴板')
  } catch {
    statusText.value = '复制失败'
    showToast('复制失败', 'error')
  }
}

function truncate(value, length) {
  if (!value) {
    return ''
  }
  return value.length <= length ? value : `${value.slice(0, length)}...`
}

function openPreview(row) {
  const wasOpen = showPreviewModal.value
  showPreviewModal.value = true

  previewForm.model = row?.model || selectedModel.value
  previewForm.voice = row?.uri || ''

  if (!wasOpen) {
    previewForm.input = ''
    previewForm.response_format = 'mp3'
    previewForm.sample_rate = 44100
    previewForm.stream = false
    previewForm.speed = 1
    previewForm.gain = 0
    previewForm.max_tokens = 2048
    previewForm.referencesText = ''
  }

  previewState.error = ''
  if (previewState.audioUrl && previewObjectUrl) {
    URL.revokeObjectURL(previewObjectUrl)
    previewObjectUrl = null
  }
  previewState.audioUrl = ''

  if (wasOpen && !previewState.loading && previewForm.input.trim()) {
    startSynthesis()
  }
}

function closePreview() {
  showPreviewModal.value = false
  if (previewObjectUrl) {
    URL.revokeObjectURL(previewObjectUrl)
    previewObjectUrl = null
  }
}

function onResponseFormatChange() {
  if (previewForm.response_format === 'opus') {
    previewForm.sample_rate = 48000
    return
  }

  if (previewForm.response_format === 'mp3') {
    previewForm.sample_rate = 44100
    return
  }

  if ((previewForm.response_format === 'wav' || previewForm.response_format === 'pcm') && ![8000, 16000, 24000, 32000, 44100].includes(Number(previewForm.sample_rate))) {
    previewForm.sample_rate = 44100
  }
}

async function startSynthesis() {
  if (!apiKey.value.trim()) {
    statusText.value = '请先配置 API Key'
    return
  }
  if (!previewForm.input.trim()) {
    statusText.value = '请输入要合成的文本'
    return
  }

  previewState.loading = true
  previewState.error = ''

  try {
    const payload = {
      model: previewForm.model,
      input: previewForm.input,
      response_format: previewForm.response_format,
      sample_rate: Number(previewForm.sample_rate),
      stream: previewForm.stream === true || previewForm.stream === 'true',
      speed: Number(previewForm.speed),
      gain: Number(previewForm.gain)
    }

    if (previewForm.model === 'fnlp/MOSS-TTSD-v0.5') {
      payload.max_tokens = Number(previewForm.max_tokens)
    }

    const referencesText = previewForm.referencesText.trim()
    if (referencesText) {
      try {
        const references = JSON.parse(referencesText)
        if (!Array.isArray(references)) {
          throw new Error('references 需为数组')
        }
        payload.references = references
      } catch {
        throw new Error('references JSON 无效')
      }
    } else if (previewForm.voice) {
      payload.voice = previewForm.voice
    }

    const response = await requestApi('/audio/speech', {
      method: 'POST',
      headers: getAuthHeaders({
        'Content-Type': 'application/json'
      }),
      body: JSON.stringify(payload)
    })

    const blob = await response.blob()
    if (previewObjectUrl) {
      URL.revokeObjectURL(previewObjectUrl)
    }
    previewObjectUrl = URL.createObjectURL(blob)
    previewState.audioUrl = previewObjectUrl

    statusText.value = '合成完成'
  } catch (error) {
    previewState.error = error.message || String(error)
    statusText.value = '合成失败'
    synthesisErrorDetail.value = `合成失败。\n\n可能原因：\n- 网络问题或请求超时\n- 参考音频体积或时长过大（>30秒或>5MB）\n\n详细错误：${previewState.error}`
    showSynthesisErrorModal.value = true
    showToast('合成失败', 'error')
  } finally {
    previewState.loading = false
  }
}

async function loadMarketList() {
  try {
    const response = await fetch('/voice/voices.json', { cache: 'no-cache' })
    if (!response.ok) {
      throw new Error('读取 voices.json 失败')
    }
    const data = await response.json()

    if (Array.isArray(data)) {
      marketList.value = data.map((item) => ({
        file: item.file,
        title: item.title || item.file,
        path: item.path || `/voice/${item.file}`,
        ext: item.ext || '',
        description: item.description || '',
        tags: item.tags || [],
        referenceText: item.referenceText || ''
      }))
      return
    }

    marketList.value = []
  } catch {
    marketList.value = [
      {
        file: 'Furina.wav',
        title: 'Furina.wav',
        path: '/voice/Furina.wav',
        referenceText: DEFAULT_REFERENCE_TEXT,
        description: ''
      }
    ]
  }
}

async function uploadAndUseMarketVoice(item) {
  marketState.uploading = true
  marketState.error = ''
  marketState.status = '正在读取本地音频...'

  try {
    const sourcePath = item.path || `/voice/${item.file}`
    const sourceResponse = await fetch(sourcePath)
    if (!sourceResponse.ok) {
      throw new Error('读取本地音频失败')
    }

    const sourceBlob = await sourceResponse.blob()
    const fileName = item.file
    const nameBase = fileName.replace(/\.[^/.]+$/, '')
    let sanitized = (nameBase.normalize ? nameBase.normalize('NFKD') : nameBase).replace(/[^A-Za-z0-9_-]/g, '').slice(0, 64)
    if (!sanitized) {
      sanitized = `voice-${Date.now()}`
    }

    const key = (localStorage.getItem('sf_api_key') || '').trim()
    if (!key) {
      throw new Error('请先在主页配置 SiliconFlow API Key')
    }

    const formData = new FormData()
    formData.append('file', new File([sourceBlob], fileName))
    formData.append('model', 'FunAudioLLM/CosyVoice2-0.5B')
    formData.append('customName', sanitized)
    formData.append('text', item.referenceText || DEFAULT_REFERENCE_TEXT)

    marketState.status = '正在上传到 SiliconFlow...'

    const uploadResponse = await requestApi('/uploads/audio/voice', {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${key}`
      },
      body: formData
    })

    const result = await uploadResponse.json()
    const marketUri = result.uri || result?.data?.uri
    if (!marketUri) {
      throw new Error('未返回音色 URI')
    }

    prefill.value = {
      model: 'FunAudioLLM/CosyVoice2-0.5B',
      voice: marketUri
    }

    showMarket.value = false
    openPreview({
      model: prefill.value.model,
      uri: prefill.value.voice
    })
  } catch (error) {
    marketState.error = error.message || String(error)
    showToast('广场音色上传失败', 'error')
  } finally {
    marketState.uploading = false
    marketState.status = marketState.error ? '上传失败' : '上传完成'
  }
}

function openMarket() {
  showMarket.value = true
}

function backToManager() {
  showMarket.value = false
}

function openAuthor() {
  window.open('https://gbkgov.cn', '_blank')
}

function openGsov() {
  window.open('https://tts.acgnai.top/', '_blank')
}

function openMinimax() {
  window.open('https://www.minimaxi.com/audio/voices', '_blank')
}

onMounted(async () => {
  const randomIndex = Math.floor(Math.random() * backgrounds.length)
  bgIndex.value = randomIndex
  bgImage.value = backgrounds[randomIndex]

  if (apiKey.value.trim()) {
    await refreshVoiceList()
  }

  await loadMarketList()
})

watch(
  () => prefill.value,
  (value) => {
    if (value && (value.model || value.voice)) {
      openPreview({ model: value.model || selectedModel.value, uri: value.voice || '' })
    }
  }
)
</script>

<template>
  <div class="bg" :style="pageBgStyle"></div>

  <div class="page">
    <div v-if="showMarket" class="market">
      <div class="market-header">
        <h2 class="title">共享音色广场</h2>
        <div class="actions">
          <button class="btn" style="margin-right: 8px" @click="openGsov">GSOV 生成</button>
          <button class="btn" style="margin-right: 8px" @click="openMinimax">MiniMax 生成</button>
          <button class="btn" @click="backToManager">返回管理</button>
        </div>
      </div>

      <div class="local">
        <div class="subtitle">一键快速使用，如果您有可供分享的高品质角色原声，欢迎QQ联系：1436198709 投稿 。</div>
        <div class="grid">
          <div v-for="item in marketList" :key="item.file" class="card">
            <div class="card-title">{{ item.title }}</div>
            <div v-if="item.description" class="card-desc">{{ item.description }}</div>
            <div class="player">
              <audio :src="item.path || `/voice/${item.file}`" controls preload="none"></audio>
            </div>
            <div class="meta">
              <span>路径</span>
              <span class="mono">{{ item.path || `/voice/${item.file}` }}</span>
            </div>
            <div class="meta">
              <span>参考文本</span>
              <span class="mono">{{ item.referenceText || DEFAULT_REFERENCE_TEXT }}</span>
            </div>
            <div class="card-actions">
              <button class="btn small" :disabled="marketState.uploading" @click="uploadAndUseMarketVoice(item)">
                {{ marketState.uploading ? '正在上传...' : '上传并使用' }}
              </button>
            </div>
          </div>
        </div>

        <div v-if="marketState.status" class="status">{{ marketState.status }}</div>
        <div v-if="marketState.error" class="error-text">{{ marketState.error }}</div>
      </div>
    </div>

    <div v-else class="panel">
      <button class="change-bg-btn" title="换一张背景" @click="changeBackground">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"></path>
          <path d="M21 3v5h-5"></path>
          <path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"></path>
          <path d="M3 21v-5h5"></path>
        </svg>
      </button>

      <section class="section">
        <h2 class="title">API 配置</h2>
        <div class="field">
          <label class="label">API Key</label>
          <div class="input-with-eye">
            <input v-model="apiKey" class="input" :type="maskedApiInputType" placeholder="请输入 SiliconFlow API Key" />
            <button class="icon-btn" type="button" :title="showApiKey ? '隐藏' : '显示'" @click="showApiKey = !showApiKey">
              <svg v-if="showApiKey" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"></path>
                <circle cx="12" cy="12" r="3"></circle>
              </svg>
              <svg v-else width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M17.94 17.94A10.94 10.94 0 0 1 12 20C5 20 1 12 1 12a21.86 21.86 0 0 1 5.06-6.94"></path>
                <path d="M10.58 10.58a2 2 0 1 0 2.83 2.83"></path>
                <line x1="1" y1="1" x2="23" y2="23"></line>
              </svg>
            </button>
          </div>
          <div class="actions">
            <button class="btn" @click="saveApiKey">保存配置</button>
          </div>
          <p class="hint">默认接口地址：{{ API_BASE }}</p>
        </div>
      </section>

      <section class="section">
        <h2 class="title">上传自定义音色</h2>
        <div class="grid grid-2">
          <div class="field">
            <label class="label">音频文件</label>
            <div class="file-row">
              <input id="file-input" type="file" accept=".mp3,.wav,.pcm,.opus" @change="onChooseFile" />
              <label class="file-btn" for="file-input">选择文件</label>
              <div class="file-name">{{ selectedFileName || '未选择文件' }}</div>
            </div>
          </div>

          <div class="field">
            <label class="label">模型</label>
            <select v-model="selectedModel" class="input">
              <option v-for="model in MODELS" :key="model" :value="model">{{ model }}</option>
            </select>
          </div>

          <div class="field">
            <label class="label">音色名称</label>
            <input v-model="customName" class="input" placeholder="请输入音色名称" />
          </div>

          <div class="field">
            <label class="label">参考文本</label>
            <input v-model="referenceText" class="input" placeholder="请输入参考文本" />
          </div>
        </div>

        <div class="actions">
          <button class="btn accent" :disabled="loadingUpload" @click="uploadVoice">{{ loadingUpload ? '正在上传...' : '上传音色' }}</button>
        </div>
      </section>

      <section class="section">
        <h2 class="title">上传结果</h2>
        <div class="result">
          <div class="result-row">
            <span class="k">自定义音色名称</span>
            <span class="v">{{ uploadResult.customName || '—' }}</span>
          </div>
          <div class="result-row">
            <span class="k">音色 URI</span>
            <span class="v">{{ uploadResult.uri || '—' }}</span>
            <button class="btn small" :disabled="!uploadResult.uri" @click="copyText(uploadResult.uri)">复制 URI</button>
          </div>
        </div>
      </section>

      <section class="section">
        <div class="header-row">
          <h2 class="title">音色管理</h2>
          <div class="toolbar">
            <div class="left-actions">
              <button class="btn" :disabled="loadingList" @click="refreshAll">{{ loadingList ? '正在刷新...' : '刷新音色列表' }}</button>
              <button class="btn" @click="openMarket">共享音色广场</button>
            </div>
            <div class="balance" :title="balance.error || '点击刷新'" @click="refreshBalance">
              余额：{{ balance.loading ? '刷新中…' : balance.value ?? '点击刷新' }}
            </div>
          </div>
        </div>

        <div class="table">
          <div class="thead">
            <div class="th">名称</div>
            <div class="th mobile-hidden">URI</div>
            <div class="th mobile-hidden">模型</div>
            <div class="th mobile-hidden">参考文本</div>
            <div class="th actions-th">操作</div>
          </div>

          <div class="tbody">
            <div v-for="row in voiceList" :key="row.uri" class="tr">
              <div class="td name copyable" :title="row.customName || 'N/A'" role="button" tabindex="0" @click="copyText(row.customName)">{{ truncate(row.customName, 15) || 'N/A' }}</div>
              <div class="td uri mobile-hidden copyable" :title="row.uri || 'N/A'" role="button" tabindex="0" @click="copyText(row.uri)">{{ truncate(row.uri, 30) || 'N/A' }}</div>
              <div class="td model mobile-hidden copyable" :title="row.model || 'N/A'" role="button" tabindex="0" @click="copyText(row.model)">{{ truncate(row.model, 20) || 'N/A' }}</div>
              <div class="td text mobile-hidden copyable" :title="row.text || 'N/A'" role="button" tabindex="0" @click="copyText(row.text)">{{ truncate(row.text, 50) || 'N/A' }}</div>
              <div class="td actions">
                <button
                  class="btn small"
                  @click="copyText(`音色名称: ${row.customName || 'N/A'}\nURI: ${row.uri || 'N/A'}\n模型: ${row.model || 'N/A'}\n参考文本: ${row.text || 'N/A'}`)"
                >
                  复制
                </button>
                <button class="btn small" @click="openPreview(row)">试听</button>
                <button class="btn small" :disabled="!row.uri" @click="deleteVoice(row.uri)">删除</button>
              </div>
            </div>

            <div v-if="voiceList.length === 0" class="empty">暂无数据</div>
          </div>
        </div>
      </section>

      <div class="footer">
        <div class="status">{{ statusText }}</div>
        <div class="author" @click="openAuthor">Powered by Chris</div>
      </div>

      <div class="star-callout">
        <a href="https://github.com/Chris95743/astrbot_plugin_VITS_pro" target="_blank" rel="noopener" title="如果您觉得此项目不错，欢迎来个 Star！">
          如果您觉得本项目还不错，欢迎来个 Star，您的 Star 是我更新的动力
        </a>
      </div>
    </div>

    <div class="toast" :class="[{ show: toast.show }, toast.type]">
      <div class="toast-content">
        <svg v-if="toast.type === 'success'" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M20 6L9 17l-5-5"></path>
        </svg>
        <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="12" cy="12" r="10"></circle>
          <line x1="15" y1="9" x2="9" y2="15"></line>
          <line x1="9" y1="9" x2="15" y2="15"></line>
        </svg>
        <span class="toast-message">{{ toast.message }}</span>
      </div>
    </div>

    <div v-if="showPreviewModal" class="modal">
      <div class="modal-overlay" @click="closePreview"></div>
      <div class="modal-content">
        <div class="modal-header">
          <h3 class="modal-title">试听合成</h3>
          <button class="modal-close" @click="closePreview">×</button>
        </div>

        <div class="modal-body">
          <div class="grid grid-2">
            <div class="field">
              <label class="label">模型</label>
              <select v-model="previewForm.model" class="input">
                <option v-for="model in MODELS" :key="model" :value="model">{{ model }}</option>
              </select>
            </div>

            <div class="field">
              <label class="label">输出格式</label>
              <select v-model="previewForm.response_format" class="input" @change="onResponseFormatChange">
                <option value="mp3">mp3</option>
                <option value="wav">wav</option>
                <option value="opus">opus</option>
                <option value="pcm">pcm</option>
              </select>
            </div>

            <div class="field">
              <label class="label">采样率</label>
              <input v-model.number="previewForm.sample_rate" class="input" type="number" placeholder="44100" />
            </div>

            <div class="field">
              <label class="label">流式 (stream)</label>
              <select v-model="previewForm.stream" class="input">
                <option :value="false">false</option>
                <option :value="true">true</option>
              </select>
            </div>

            <div class="field">
              <label class="label">速度 (0.25 - 4.0)</label>
              <input v-model.number="previewForm.speed" class="input" type="number" step="0.01" min="0.25" max="4" />
            </div>

            <div class="field">
              <label class="label">增益 (-10 - 10 dB)</label>
              <input v-model.number="previewForm.gain" class="input" type="number" step="0.1" min="-10" max="10" />
            </div>

            <div v-if="previewForm.model === 'fnlp/MOSS-TTSD-v0.5'" class="field">
              <label class="label">最大 tokens</label>
              <input v-model.number="previewForm.max_tokens" class="input" type="number" min="1" />
            </div>

            <div class="field span-2">
              <label class="label">音色 (voice)</label>
              <input
                v-model="previewForm.voice"
                class="input"
                placeholder="系统音色如 FunAudioLLM/CosyVoice2-0.5B:alex 或自定义URI，如 speech:***"
              />
            </div>

            <div class="field span-2">
              <label class="label">输入文本 (input)</label>
              <textarea v-model="previewForm.input" class="input" rows="4" placeholder="请输入要合成的文本"></textarea>
            </div>

            <div class="field span-2">
              <label class="label">references (JSON 数组，可选，留空与 voice 互斥)</label>
              <textarea
                v-model="previewForm.referencesText"
                class="input"
                rows="4"
                placeholder='例如: [\n  {"audio":"https://.../a.mp3", "text":"参考文本A"}\n]'
              ></textarea>
              <p class="hint">当填写 references 时会忽略 voice；MOSS 支持双音色对话。</p>
            </div>
          </div>

          <div class="actions">
            <button class="btn accent" :disabled="previewState.loading" @click="startSynthesis">
              {{ previewState.loading ? '正在合成...' : '开始合成' }}
            </button>
          </div>

          <div v-if="previewState.error" class="error-text">{{ previewState.error }}</div>

          <div v-if="previewState.audioUrl" class="player">
            <audio :src="previewState.audioUrl" controls autoplay></audio>
            <a class="download" :href="previewState.audioUrl" download="preview-audio">下载音频</a>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showSynthesisErrorModal" class="modal">
      <div class="modal-overlay" @click="showSynthesisErrorModal = false"></div>
      <div class="modal-content">
        <div class="modal-header">
          <h3 class="modal-title">合成失败</h3>
          <button class="modal-close" @click="showSynthesisErrorModal = false">×</button>
        </div>

        <div class="modal-body">
          <p class="error-text" style="white-space: pre-wrap">{{ synthesisErrorDetail }}</p>
          <div class="actions" style="margin-top: 12px">
            <button class="btn" @click="showSynthesisErrorModal = false">我知道了</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped></style>
