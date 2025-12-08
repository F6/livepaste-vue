<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'
import axios from 'axios'

// State
const backendUri = ref('http://localhost:8000')
const token = ref('')
const username = ref('')
const password = ref('')
const passphrase = ref('')
const content = ref('')
const status = ref('idle')
const autoUpdate = ref(true)
const updateDelay = ref(500)
const updateTimer = ref<ReturnType<typeof setTimeout> | null>(null)
const imagePreview = ref<{ src: string; filename: string } | null>(null)
const files = ref<
  Array<{ filename: string; url: string; size: number; content_type: string }>
>([])
const selectedFiles = ref<string[]>([])
const ws = ref<WebSocket | null>(null)

// Methods
const login = () => {
  if (!username.value || !password.value) {
    status.value = 'username and password required'
    return
  }
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/login`
  axios
    .post(uri, null, { params: { username: username.value, password: password.value } })
    .then((r) => {
      token.value = r.data.access_token
      localStorage.setItem('livepaste_token', token.value)
      localStorage.setItem('livepaste_username', username.value)
      status.value = 'logged in'
    })
    .catch((e) => {
      status.value = 'login error'
      console.error(e)
    })
}

const create = () => {
  if (!token.value) {
    status.value = 'need to login first'
    return
  }
  const pf = passphrase.value || undefined
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions`
  const headers = { Authorization: `Bearer ${token.value}` }
  axios
    .post(uri, null, { params: { passphrase: pf }, headers })
    .then((r) => {
      passphrase.value = r.data.passphrase
      status.value = `created ${passphrase.value}`
    })
    .catch((e) => {
      status.value = 'error'
      console.error(e)
    })
}

const join = () => {
  if (!passphrase.value) {
    status.value = 'need passphrase'
    return
  }
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions/${encodeURIComponent(
    passphrase.value
  )}/join`
  axios
    .post(uri)
    .then((r) => {
      content.value = r.data.content || ''
      files.value = r.data.files || []
      status.value = 'joined'
      connectWS()
    })
    .catch((e) => {
      status.value = 'error'
      console.error(e)
    })
}

const end = () => {
  if (!passphrase.value) {
    status.value = 'need passphrase'
    return
  }
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions/${encodeURIComponent(
    passphrase.value
  )}/end`
  const headers = { Authorization: `Bearer ${token.value}` }
  axios
    .post(uri, {}, { headers })
    .then(() => {
      status.value = 'ended'
      if (ws.value) {
        ws.value.close()
      }
    })
    .catch((e) => {
      status.value = 'error'
      console.error(e)
    })
}

const sendUpdate = () => {
  if (ws.value && ws.value.readyState === WebSocket.OPEN) {
    ws.value.send(JSON.stringify({ type: 'update', content: content.value }))
    status.value = 'sent'
  } else {
    // fallback: polling API
    const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions/${encodeURIComponent(
      passphrase.value
    )}/join`
    axios
      .post(uri, {})
      .then(() => {
        status.value = 'fallback mode (no HTTP update endpoint)'
      })
      .catch(() => {
        status.value = 'error'
      })
  }
}

const connectWS = () => {
  try {
    if (ws.value) {
      ws.value.close()
    }
    const backend = backendUri.value
      .replace('http://', '')
      .replace('https://', '')
    const proto = backendUri.value.startsWith('https') ? 'wss' : 'ws'
    const wsUrl = `${proto}://${backend}/ws/${encodeURIComponent(passphrase.value)}`
    const newWs = new WebSocket(wsUrl)

    newWs.onopen = () => {
      status.value = 'ws open'
    }

    newWs.onmessage = (ev) => {
      try {
        const msg = JSON.parse(ev.data)
        if (msg.type === 'update') {
          content.value = msg.content
        } else if (msg.type === 'image') {
          const url = `${backendUri.value.replace(/\/$/, '')}${msg.url}`
          imagePreview.value = { src: url, filename: msg.filename || 'image' }
          files.value.push({
            filename: msg.filename || 'image',
            url: msg.url,
            size: msg.size,
            content_type: msg.content_type
          })
        } else if (msg.type === 'file') {
          files.value.push({
            filename: msg.filename,
            url: msg.url,
            size: msg.size,
            content_type: msg.content_type
          })
        } else if (msg.type === 'session_ended') {
          status.value = 'ended'
          newWs.close()
        }
      } catch (e) {
        console.error('ws message parse', e)
      }
    }

    newWs.onclose = () => {
      status.value = 'ws closed'
    }

    newWs.onerror = () => {
      status.value = 'ws error'
    }

    ws.value = newWs
  } catch (e) {
    console.error(e)
    status.value = 'ws fail'
  }
}

const onFileChange = (ev: Event) => {
  const input = ev.target as HTMLInputElement
  const f = input.files?.[0]
  if (f) {
    sendImage(f)
  }
}

const sendImage = (file: File) => {
  const reader = new FileReader()
  reader.onload = () => {
    const dataUrl = reader.result as string
    imagePreview.value = { src: dataUrl, filename: file.name || 'image' }
    try {
      if (ws.value && ws.value.readyState === WebSocket.OPEN) {
        ws.value.send(
          JSON.stringify({
            type: 'image',
            filename: file.name || 'image',
            data: dataUrl
          })
        )
        status.value = 'image sent'
      } else {
        status.value = 'no websocket (image not sent)'
      }
    } catch (e) {
      console.error(e)
      status.value = 'image send error'
    }
  }
  reader.readAsDataURL(file)
}

const clearImage = () => {
  imagePreview.value = null
}

const onPaste = (ev: ClipboardEvent) => {
  try {
    const items = ev.clipboardData?.items || []
    for (let i = 0; i < items.length; i++) {
      const it = items[i]
      if (!it) continue
      // guard against undefined properties (TypeScript strict checks)
      if (it.kind === 'file' && it.type?.startsWith('image/')) {
        const file = it.getAsFile?.()
        if (file) {
          sendImage(file)
          ev.preventDefault()
          break
        }
      }
    }
  } catch (e) {
    // ignore
  }
}

const uploadFileHTTP = async (ev: Event) => {
  const input = ev.target as HTMLInputElement
  const f = input.files?.[0]
  if (!f) return
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions/${encodeURIComponent(
    passphrase.value
  )}/upload`
  const fd = new FormData()
  fd.append('file', f)
  try {
    const r = await axios.post(uri, fd, {
      headers: { 'Content-Type': 'multipart/form-data' }
    })
    files.value.push(r.data)
    status.value = 'file uploaded'
  } catch (e) {
    status.value = 'upload error'
    console.error(e)
  }
}

const refreshFiles = async () => {
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions/${encodeURIComponent(
    passphrase.value
  )}/files`
  try {
    const r = await axios.get(uri)
    files.value = r.data.files || []
  } catch (e) {
    console.error(e)
  }
}

const downloadSelectedZip = () => {
  if (!selectedFiles.value || selectedFiles.value.length === 0) {
    status.value = 'no files selected'
    return
  }
  const list = selectedFiles.value.join(',')
  const uri = `${backendUri.value.replace(/\/$/, '')}/api/sessions/${encodeURIComponent(
    passphrase.value
  )}/files/download?files=${encodeURIComponent(list)}`
  window.location.href = uri
}

// Watchers
watch(content, (newVal) => {
  if (!autoUpdate.value) return
  if (updateTimer.value) clearTimeout(updateTimer.value)
  updateTimer.value = setTimeout(() => {
    try {
      sendUpdate()
    } catch (e) {
      console.error(e)
    }
    updateTimer.value = null
  }, updateDelay.value || 500)
})

watch(backendUri, (newVal) => {
  localStorage.setItem('livepaste_backendUri', newVal)
})

watch(token, (newVal) => {
  if (newVal) {
    localStorage.setItem('livepaste_token', newVal)
  } else {
    localStorage.removeItem('livepaste_token')
  }
})

watch(passphrase, (newVal) => {
  if (newVal) {
    localStorage.setItem('livepaste_passphrase', newVal)
  } else {
    localStorage.removeItem('livepaste_passphrase')
  }
})

// Lifecycle
const cleanup = () => {
  document.removeEventListener('paste', onPaste as EventListener)
}

onMounted(() => {
  const savedUri = localStorage.getItem('livepaste_backendUri')
  if (savedUri) {
    backendUri.value = savedUri
  }
  
  const savedToken = localStorage.getItem('livepaste_token')
  if (savedToken) {
    token.value = savedToken
  }
  
  const savedUsername = localStorage.getItem('livepaste_username')
  if (savedUsername) {
    username.value = savedUsername
  }
  
  const savedPassphrase = localStorage.getItem('livepaste_passphrase')
  if (savedPassphrase) {
    passphrase.value = savedPassphrase
  }
  
  document.addEventListener('paste', onPaste as EventListener)
})
</script>

<template>
  <div class="min-h-screen bg-gray-50 p-6">
    <div class="max-w-3xl mx-auto">
      <!-- Header with Status Badge -->
      <div class="flex items-center justify-between mb-6">
        <h1 class="text-2xl font-bold">livepaste</h1>
        <div
          class="px-4 py-2 rounded-full text-white text-sm font-medium shadow-lg"
          :class="{
            'bg-gray-500': status === 'idle',
            'bg-blue-500': status.includes('created') || status.includes('joined'),
            'bg-green-500': status === 'logged in' || status === 'sent' || status === 'file uploaded' || status === 'ws open',
            'bg-yellow-500': status.includes('fallback') || status === 'no websocket',
            'bg-red-500': status.includes('error') || status === 'ended' || status.includes('401') || status.includes('403'),
            'bg-purple-500': status === 'image sent' || status === 'ws closed'
          }"
        >
          {{ status }}
        </div>
      </div>

      <!-- Backend URI -->
      <div class="mb-4">
        <label class="block text-sm font-medium text-gray-700">Backend URI</label>
        <input
          v-model="backendUri"
          placeholder="http://localhost:8000"
          class="mt-1 block w-full rounded border px-3 py-2"
        />
      </div>

      <!-- Login Section -->
      <div class="mb-4 border-b pb-4">
        <label class="block text-sm font-medium text-gray-700 mb-3">Login</label>
        <div class="space-y-2">
          <div>
            <label for="username" class="block text-xs font-medium text-gray-600">Username</label>
            <input
              id="username"
              v-model="username"
              type="text"
              placeholder="Enter username"
              class="mt-1 block w-full rounded border px-3 py-2"
            />
          </div>
          <div>
            <label for="password" class="block text-xs font-medium text-gray-600">Password</label>
            <input
              id="password"
              v-model="password"
              type="password"
              placeholder="Enter password"
              class="mt-1 block w-full rounded border px-3 py-2"
            />
          </div>
          <button
            @click="login"
            class="w-full px-4 py-2 bg-purple-600 text-white rounded hover:bg-purple-700"
          >
            Login
          </button>
        </div>
      </div>

      <!-- Passphrase -->
      <div class="mb-4">
        <label class="block text-sm font-medium text-gray-700">Passphrase</label>
        <input
          v-model="passphrase"
          class="mt-1 block w-full rounded border px-3 py-2"
        />
      </div>

      <!-- Session Controls -->
      <div class="flex gap-2 mb-6">
        <button
          @click="create"
          class="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
        >
          Create
        </button>
        <button
          @click="join"
          class="px-4 py-2 bg-green-600 text-white rounded hover:bg-green-700"
        >
          Join
        </button>
        <button
          @click="end"
          class="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
        >
          End
        </button>
      </div>

      <!-- Text Clipboard -->
      <div class="mb-6 border-t pt-6">
        <div class="flex items-start justify-between gap-4">
          <div class="flex-1">
            <label class="block text-sm font-medium text-gray-700 mb-2">Clipboard</label>
            <textarea
              v-model="content"
              placeholder="Type or paste text here"
              class="block w-full rounded border px-3 py-2 h-40"
            ></textarea>
          </div>
          <div class="flex flex-col gap-4 mt-6">
            <!-- Send Button -->
            <button
              @click="sendUpdate"
              class="px-4 py-2 bg-indigo-600 text-white rounded hover:bg-indigo-700 whitespace-nowrap"
            >
              Send
            </button>
            <!-- Auto-update Checkbox -->
            <div class="flex items-center gap-2">
              <input
                type="checkbox"
                id="auto"
                v-model="autoUpdate"
                class="h-4 w-4"
              />
              <label for="auto" class="text-sm text-gray-700">Auto-update</label>
            </div>
            <!-- Delay Input -->
            <div class="flex items-center gap-2 text-sm">
              <label>Delay (ms)</label>
              <input
                type="number"
                v-model.number="updateDelay"
                min="100"
                step="100"
                class="w-20 rounded border px-2 py-1"
              />
            </div>
          </div>
        </div>
      </div>

      <!-- Image Section -->
      <div class="mb-6 border-t pt-6">
        <label class="block text-sm font-medium text-gray-700 mb-2">
          Image (paste or upload)
        </label>
        <div class="flex items-center gap-2 mb-3">
          <input type="file" accept="image/*" @change="onFileChange" />
          <button
            @click="clearImage"
            class="px-3 py-1 bg-gray-200 rounded hover:bg-gray-300"
          >
            Clear
          </button>
          <span class="text-sm text-gray-500">You can also paste an image (Ctrl+V)</span>
        </div>
        <div v-if="imagePreview" class="mt-3">
          <div class="text-sm text-gray-700 mb-2">Preview: {{ imagePreview.filename }}</div>
          <img :src="imagePreview.src" alt="preview" class="max-w-full border rounded" />
        </div>
      </div>

      <!-- Files Section -->
      <div class="mb-6 border-t pt-6">
        <label class="block text-sm font-medium text-gray-700 mb-2">Files (upload / list)</label>
        <div class="flex items-center gap-2 mb-3">
          <input type="file" @change="uploadFileHTTP" />
          <button
            @click="refreshFiles"
            class="px-3 py-1 bg-gray-200 rounded hover:bg-gray-300"
          >
            Refresh
          </button>
          <button
            @click="downloadSelectedZip"
            class="px-3 py-1 bg-blue-500 text-white rounded hover:bg-blue-600"
          >
            Download Selected (zip)
          </button>
        </div>
        <div v-if="files.length > 0" class="mt-3">
          <ul class="space-y-2">
            <li v-for="(f, idx) in files" :key="idx" class="flex items-center gap-3">
              <input type="checkbox" v-model="selectedFiles" :value="f.filename" />
              <a
                :href="`${backendUri.replace(/\/$/, '')}${f.url}`"
                target="_blank"
                class="text-sm text-blue-600 hover:underline"
              >
                {{ f.filename }}
              </a>
              <span class="text-xs text-gray-500">({{ f.size }} bytes)</span>
            </li>
          </ul>
        </div>
        <div v-else class="text-sm text-gray-500 mt-2">No files yet</div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Optional: add any component-specific styles here */
</style>
