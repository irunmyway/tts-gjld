<script setup>
import { reactive, ref } from 'vue'

const props = defineProps({
  models: {
    type: Array,
    default: () => []
  },
  defaultVoice: {
    type: String,
    default: ''
  },
  requestApi: {
    type: Function,
    required: true
  },
  getAuthHeaders: {
    type: Function,
    required: true
  }
})

const debugForm = reactive({
  model: props.models[0] || 'FunAudioLLM/CosyVoice2-0.5B',
  input: '',
  voice: props.defaultVoice || '',
  response_format: 'mp3',
  sample_rate: 44100,
  stream: false,
  speed: 1,
  gain: 0,
  max_tokens: 2048,
  referencesText: ''
})

const debugState = reactive({
  loading: false,
  error: '',
  audioUrl: ''
})

const audioRef = ref(null)
let debugObjectUrl = null

function onFormatChange() {
  if (debugForm.response_format === 'opus') {
    debugForm.sample_rate = 48000
    return
  }
  if (debugForm.response_format === 'mp3') {
    debugForm.sample_rate = 44100
    return
  }
  if ((debugForm.response_format === 'wav' || debugForm.response_format === 'pcm') && ![8000, 16000, 24000, 32000, 44100].includes(Number(debugForm.sample_rate))) {
    debugForm.sample_rate = 44100
  }
}

async function synthesize() {
  if (!debugForm.input.trim()) {
    debugState.error = '请输入要合成的文本'
    return
  }

  debugState.loading = true
  debugState.error = ''

  try {
    const payload = {
      model: debugForm.model,
      input: debugForm.input,
      voice: debugForm.voice,
      response_format: debugForm.response_format,
      sample_rate: Number(debugForm.sample_rate),
      stream: debugForm.stream === true || debugForm.stream === 'true',
      speed: Number(debugForm.speed),
      gain: Number(debugForm.gain)
    }

    if (debugForm.model === 'fnlp/MOSS-TTSD-v0.5') {
      payload.max_tokens = Number(debugForm.max_tokens)
    }

    const referencesText = debugForm.referencesText.trim()
    if (referencesText) {
      const references = JSON.parse(referencesText)
      if (!Array.isArray(references)) {
        throw new Error('references 需为数组')
      }
      payload.references = references
      delete payload.voice
    }

    const response = await props.requestApi('/audio/speech', {
      method: 'POST',
      headers: props.getAuthHeaders({
        'Content-Type': 'application/json'
      }),
      body: JSON.stringify(payload)
    })

    const blob = await response.blob()
    if (debugObjectUrl) {
      URL.revokeObjectURL(debugObjectUrl)
    }
    debugObjectUrl = URL.createObjectURL(blob)
    debugState.audioUrl = debugObjectUrl
  } catch (error) {
    debugState.error = error.message || String(error)
  } finally {
    debugState.loading = false
  }
}

function playAudio() {
  if (!audioRef.value || !debugState.audioUrl) {
    return
  }
  audioRef.value.play()
}
</script>

<template>
  <section class="section">
    <h2 class="title">音色调试</h2>

    <div class="grid grid-2">
      <div class="field">
        <label class="label">模型</label>
        <select v-model="debugForm.model" class="input">
          <option v-for="model in models" :key="model" :value="model">{{ model }}</option>
        </select>
      </div>

      <div class="field">
        <label class="label">音色 (voice)</label>
        <input v-model="debugForm.voice" class="input" placeholder="输入系统音色或自定义URI" />
      </div>

      <div class="field">
        <label class="label">输出格式</label>
        <select v-model="debugForm.response_format" class="input" @change="onFormatChange">
          <option value="mp3">mp3</option>
          <option value="wav">wav</option>
          <option value="opus">opus</option>
          <option value="pcm">pcm</option>
        </select>
      </div>

      <div class="field">
        <label class="label">流式 (stream)</label>
        <select v-model="debugForm.stream" class="input">
          <option :value="false">false</option>
          <option :value="true">true</option>
        </select>
      </div>

      <div class="field">
        <label class="label">采样率</label>
        <input v-model.number="debugForm.sample_rate" class="input" type="number" placeholder="44100" />
      </div>

      <div class="field">
        <label class="label">速度 (0.25 - 4.0)</label>
        <input v-model.number="debugForm.speed" class="input" type="number" step="0.01" min="0.25" max="4" />
      </div>

      <div class="field">
        <label class="label">增益 (-10 - 10 dB)</label>
        <input v-model.number="debugForm.gain" class="input" type="number" step="0.1" min="-10" max="10" />
      </div>

      <div v-if="debugForm.model === 'fnlp/MOSS-TTSD-v0.5'" class="field">
        <label class="label">最大 tokens</label>
        <input v-model.number="debugForm.max_tokens" class="input" type="number" min="1" />
      </div>

      <div class="field span-2">
        <label class="label">输入文本 (input)</label>
        <textarea v-model="debugForm.input" class="input" rows="4" placeholder="请输入要合成的文本"></textarea>
      </div>

      <div class="field span-2">
        <label class="label">references (JSON 数组，可选)</label>
        <textarea
          v-model="debugForm.referencesText"
          class="input"
          rows="4"
          placeholder='例如: [\n  {"audio":"https://.../a.mp3", "text":"参考文本A"}\n]'
        ></textarea>
      </div>
    </div>

    <div class="actions">
      <button class="btn accent" :disabled="debugState.loading" @click="synthesize">
        {{ debugState.loading ? '正在请求...' : '开始调试' }}
      </button>
      <button class="btn" :disabled="!debugState.audioUrl" @click="playAudio">播放</button>
    </div>

    <div v-if="debugState.error" class="error-text">{{ debugState.error }}</div>

    <div v-if="debugState.audioUrl" class="player">
      <audio ref="audioRef" :src="debugState.audioUrl" controls></audio>
    </div>
  </section>
</template>
