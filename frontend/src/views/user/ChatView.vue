<template>
  <div class="chat-page flex h-[calc(100vh-64px)] overflow-hidden rounded-xl border border-gray-200 bg-white dark:border-dark-700 dark:bg-dark-900">
    <!-- 左侧会话列表 -->
    <aside class="chat-sidebar flex w-64 flex-shrink-0 flex-col border-r border-gray-200 dark:border-dark-700 lg:w-72" :class="{ 'hidden md:flex': !showSidebar }">
      <div class="flex items-center justify-between border-b border-gray-200 px-4 py-3 dark:border-dark-700">
        <h2 class="text-base font-semibold text-gray-900 dark:text-white">Chat</h2>
        <button
          @click="createNewConversation"
          class="rounded-lg p-1.5 text-gray-500 transition-colors hover:bg-gray-100 hover:text-gray-700 dark:text-gray-400 dark:hover:bg-dark-700 dark:hover:text-gray-200"
          :title="t('chat.newConversation')"
        >
          <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="1.5">
            <path stroke-linecap="round" stroke-linejoin="round" d="M12 4.5v15m7.5-7.5h-15" />
          </svg>
        </button>
      </div>
      <div class="flex-1 overflow-y-auto p-2">
        <div v-if="conversations.length === 0" class="px-3 py-8 text-center text-sm text-gray-400 dark:text-gray-500">
          {{ t('chat.noConversations') }}
        </div>
        <button
          v-for="conv in conversations"
          :key="conv.id"
          @click="selectConversation(conv.id)"
          class="mb-1 w-full rounded-lg px-3 py-2.5 text-left transition-colors"
          :class="currentConversationId === conv.id
            ? 'bg-blue-50 text-blue-700 dark:bg-blue-900/20 dark:text-blue-300'
            : 'text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-dark-700'"
        >
          <div class="truncate text-sm font-medium">{{ conv.title || t('chat.untitled') }}</div>
          <div class="mt-0.5 flex items-center gap-2 text-xs text-gray-400 dark:text-gray-500">
            <span>{{ conv.messages.length }} {{ t('chat.messages') }}</span>
            <span>&middot;</span>
            <span>{{ formatTime(conv.updatedAt) }}</span>
          </div>
        </button>
      </div>
    </aside>

    <!-- 右侧聊天区 -->
    <div class="flex flex-1 flex-col overflow-hidden">
      <!-- 顶部栏 -->
      <div class="flex items-center justify-between border-b border-gray-200 px-4 py-3 dark:border-dark-700">
        <div class="flex items-center gap-2">
          <button
            class="rounded-lg p-1.5 text-gray-500 hover:bg-gray-100 dark:text-gray-400 dark:hover:bg-dark-700 md:hidden"
            @click="showSidebar = !showSidebar"
          >
            <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="1.5">
              <path stroke-linecap="round" stroke-linejoin="round" d="M3.75 6.75h16.5M3.75 12h16.5m-16.5 5.25h16.5" />
            </svg>
          </button>
          <h3 class="text-sm font-medium text-gray-900 dark:text-white">
            {{ currentConversation?.title || t('chat.newConversation') }}
          </h3>
        </div>
        <div v-if="currentConversation" class="text-xs text-gray-400 dark:text-gray-500">
          {{ currentConversation.messages.length }} {{ t('chat.messages') }}
        </div>
      </div>

      <!-- 消息区 -->
      <div ref="messagesContainer" class="flex-1 overflow-y-auto px-4 py-4">
        <div v-if="!currentConversation" class="flex h-full items-center justify-center">
          <div class="text-center text-gray-400 dark:text-gray-500">
            <svg class="mx-auto mb-3 h-12 w-12" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="1">
              <path stroke-linecap="round" stroke-linejoin="round" d="M8.625 12a.375.375 0 11-.75 0 .375.375 0 01.75 0zm0 0H8.25m4.125 0a.375.375 0 11-.75 0 .375.375 0 01.75 0zm0 0H12m4.125 0a.375.375 0 11-.75 0 .375.375 0 01.75 0zm0 0h-.375M21 12c0 4.556-4.03 8.25-9 8.25a9.764 9.764 0 01-2.555-.337A5.972 5.972 0 015.41 20.97a5.969 5.969 0 01-.474-.065 4.48 4.48 0 00.978-2.025c.09-.457-.133-.901-.467-1.226C3.93 16.178 3 14.189 3 12c0-4.556 4.03-8.25 9-8.25s9 3.694 9 8.25z" />
            </svg>
            <p class="text-sm">{{ t('chat.selectOrCreate') }}</p>
          </div>
        </div>

        <template v-else>
          <div v-for="(msg, idx) in currentConversation.messages" :key="idx" class="mb-4">
            <!-- 用户消息 -->
            <div v-if="msg.role === 'user'" class="flex justify-end">
              <div class="max-w-[80%] rounded-2xl rounded-br-md bg-blue-600 px-4 py-2.5 text-sm text-white">
                <p class="whitespace-pre-wrap">{{ msg.content }}</p>
              </div>
            </div>
            <!-- 助手消息 -->
            <div v-else-if="msg.role === 'assistant'" class="flex justify-start">
              <div class="max-w-[80%] rounded-2xl rounded-bl-md bg-gray-100 px-4 py-2.5 text-sm text-gray-900 dark:bg-dark-700 dark:text-gray-100">
                <p class="whitespace-pre-wrap">{{ msg.content }}</p>
              </div>
            </div>
          </div>
          <!-- 发送中指示器 -->
          <div v-if="sending" class="mb-4 flex justify-start">
            <div class="rounded-2xl rounded-bl-md bg-gray-100 px-4 py-2.5 text-sm text-gray-500 dark:bg-dark-700 dark:text-gray-400">
              <span class="inline-flex items-center gap-1">
                <span class="animate-pulse">●</span>
                <span class="animate-pulse" style="animation-delay: 0.2s">●</span>
                <span class="animate-pulse" style="animation-delay: 0.4s">●</span>
              </span>
            </div>
          </div>
        </template>
      </div>

      <!-- 底部输入区 -->
      <div class="border-t border-gray-200 px-4 py-3 dark:border-dark-700">
        <!-- 错误提示 -->
        <div v-if="errorMessage" class="mb-2 rounded-lg bg-red-50 px-3 py-2 text-xs text-red-600 dark:bg-red-900/20 dark:text-red-400">
          {{ errorMessage }}
        </div>
        <!-- 控制栏 -->
        <div class="mb-2 flex flex-wrap items-center gap-2">
          <!-- API Key 下拉 -->
          <select
            v-model="selectedKeyId"
            class="rounded-lg border border-gray-200 bg-white px-2.5 py-1.5 text-xs text-gray-700 dark:border-dark-600 dark:bg-dark-800 dark:text-gray-300"
          >
            <option value="" disabled>{{ t('chat.selectApiKey') }}</option>
            <option v-for="k in apiKeys" :key="k.id" :value="k.id">
              {{ k.name }} ({{ maskKey(k.key) }})
            </option>
          </select>
          <!-- 模型下拉 -->
          <select
            v-model="selectedModel"
            class="rounded-lg border border-gray-200 bg-white px-2.5 py-1.5 text-xs text-gray-700 dark:border-dark-600 dark:bg-dark-800 dark:text-gray-300"
          >
            <option value="" disabled>{{ t('chat.selectModel') }}</option>
            <option v-for="m in availableModels" :key="m" :value="m">{{ m }}</option>
          </select>
        </div>
        <!-- 输入框 + 发送 -->
        <div class="flex items-end gap-2">
          <textarea
            ref="inputRef"
            v-model="inputText"
            :placeholder="t('chat.inputPlaceholder')"
            :disabled="sending"
            rows="1"
            class="max-h-32 min-h-[40px] flex-1 resize-none rounded-xl border border-gray-200 bg-gray-50 px-4 py-2.5 text-sm text-gray-900 outline-none transition-colors focus:border-blue-400 focus:bg-white dark:border-dark-600 dark:bg-dark-800 dark:text-gray-100 dark:focus:border-blue-500 dark:focus:bg-dark-700"
            @keydown="handleKeydown"
            @input="autoResize"
          ></textarea>
          <button
            @click="sendMessage"
            :disabled="!canSend"
            class="flex h-10 w-10 flex-shrink-0 items-center justify-center rounded-xl bg-blue-600 text-white transition-colors hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-50"
          >
            <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 12L3.269 3.126A59.768 59.768 0 0121.485 12 59.77 59.77 0 013.27 20.876L5.999 12zm0 0h7.5" />
            </svg>
          </button>
        </div>
        <!-- 提示 -->
        <div v-if="!apiKeys.length" class="mt-2 text-xs text-amber-600 dark:text-amber-400">
          {{ t('chat.noApiKeys') }}
        </div>
        <div v-else-if="!availableModels.length" class="mt-2 text-xs text-amber-600 dark:text-amber-400">
          {{ t('chat.noModels') }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, nextTick, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useAuthStore } from '@/stores/auth'
import { keysAPI } from '@/api/keys'
import { userChannelsAPI } from '@/api/channels'
import type { ApiKey } from '@/types'

const { t } = useI18n()
const authStore = useAuthStore()

interface ChatMessage {
  role: 'user' | 'assistant' | 'system'
  content: string
  createdAt: string
}

interface Conversation {
  id: string
  title: string
  model: string
  apiKeyId: number | null
  apiKeyLabel: string
  messages: ChatMessage[]
  createdAt: string
  updatedAt: string
}

const showSidebar = ref(true)
const conversations = ref<Conversation[]>([])
const currentConversationId = ref<string | null>(null)
const inputText = ref('')
const sending = ref(false)
const errorMessage = ref('')
const apiKeys = ref<ApiKey[]>([])
const availableModels = ref<string[]>([])
const selectedKeyId = ref<number | ''>('')
const selectedModel = ref('')
const messagesContainer = ref<HTMLElement | null>(null)
const inputRef = ref<HTMLTextAreaElement | null>(null)

const MAX_CONTEXT_MESSAGES = 20

const currentConversation = computed(() =>
  conversations.value.find(c => c.id === currentConversationId.value) || null
)

const selectedKeyObj = computed(() =>
  apiKeys.value.find(k => k.id === selectedKeyId.value) || null
)

const canSend = computed(() =>
  inputText.value.trim() &&
  selectedKeyId.value &&
  selectedModel.value &&
  !sending.value
)

function maskKey(key: string): string {
  if (!key || key.length < 8) return '****'
  return key.slice(0, 3) + '****' + key.slice(-4)
}

function getStorageKey(): string {
  const userId = authStore.user?.id || 'anonymous'
  return `sub2api_chat_conversations_${userId}`
}

function loadConversations() {
  try {
    const raw = localStorage.getItem(getStorageKey())
    if (raw) {
      conversations.value = JSON.parse(raw)
    }
  } catch { /* ignore */ }
}

function saveConversations() {
  const toSave = conversations.value.map(c => ({
    ...c,
    apiKeyId: c.apiKeyId,
    apiKeyLabel: c.apiKeyLabel,
  }))
  localStorage.setItem(getStorageKey(), JSON.stringify(toSave))
}

function createNewConversation() {
  const conv: Conversation = {
    id: Date.now().toString(36) + Math.random().toString(36).slice(2, 8),
    title: '',
    model: selectedModel.value,
    apiKeyId: selectedKeyId.value ? Number(selectedKeyId.value) : null,
    apiKeyLabel: selectedKeyObj.value ? `${selectedKeyObj.value.name} (${maskKey(selectedKeyObj.value.key)})` : '',
    messages: [],
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  }
  conversations.value.unshift(conv)
  currentConversationId.value = conv.id
  saveConversations()
}

function selectConversation(id: string) {
  currentConversationId.value = id
  showSidebar.value = false
  nextTick(scrollToBottom)
}

function scrollToBottom() {
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

function formatTime(iso: string): string {
  const d = new Date(iso)
  const now = new Date()
  if (d.toDateString() === now.toDateString()) {
    return d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
  }
  return d.toLocaleDateString([], { month: 'short', day: 'numeric' })
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
}

function autoResize() {
  const el = inputRef.value
  if (el) {
    el.style.height = 'auto'
    el.style.height = Math.min(el.scrollHeight, 128) + 'px'
  }
}

async function sendMessage() {
  if (!canSend.value) return
  const text = inputText.value.trim()
  if (!text) return

  errorMessage.value = ''

  if (!currentConversation.value) {
    createNewConversation()
  }

  const conv = currentConversation.value!
  if (!conv.title) {
    conv.title = text.slice(0, 20)
  }

  conv.messages.push({
    role: 'user',
    content: text,
    createdAt: new Date().toISOString(),
  })
  conv.updatedAt = new Date().toISOString()
  conv.model = selectedModel.value
  conv.apiKeyId = selectedKeyId.value ? Number(selectedKeyId.value) : null
  conv.apiKeyLabel = selectedKeyObj.value ? `${selectedKeyObj.value.name} (${maskKey(selectedKeyObj.value.key)})` : ''

  inputText.value = ''
  if (inputRef.value) {
    inputRef.value.style.height = 'auto'
  }
  saveConversations()
  await nextTick()
  scrollToBottom()

  sending.value = true

  const contextMessages = conv.messages
    .filter(m => m.role !== 'system')
    .slice(-MAX_CONTEXT_MESSAGES)
    .map(m => ({ role: m.role, content: m.content }))

  const apiKeyValue = selectedKeyObj.value?.key
  if (!apiKeyValue) {
    errorMessage.value = t('chat.errorNoKey')
    sending.value = false
    return
  }

  try {
    const res = await fetch('/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKeyValue}`,
      },
      body: JSON.stringify({
        model: selectedModel.value,
        messages: [
          { role: 'system', content: 'You are a helpful assistant.' },
          ...contextMessages,
        ],
        stream: false,
      }),
    })

    if (!res.ok) {
      const errBody = await res.json().catch(() => null)
      const errMsg = errBody?.error?.message || `HTTP ${res.status}`
      throw new Error(errMsg)
    }

    const data = await res.json()
    const assistantContent = data?.choices?.[0]?.message?.content || ''

    conv.messages.push({
      role: 'assistant',
      content: assistantContent,
      createdAt: new Date().toISOString(),
    })
    conv.updatedAt = new Date().toISOString()
    saveConversations()
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : String(err)
    errorMessage.value = msg
  } finally {
    sending.value = false
    await nextTick()
    scrollToBottom()
  }
}

async function fetchApiKeys() {
  try {
    const res = await keysAPI.list(1, 100, { status: 'active' })
    apiKeys.value = res.items || []
    if (apiKeys.value.length && !selectedKeyId.value) {
      selectedKeyId.value = apiKeys.value[0].id
    }
  } catch { /* ignore */ }
}

async function fetchModels() {
  if (!selectedKeyId.value) {
    availableModels.value = []
    selectedModel.value = ''
    return
  }

  try {
    const channels = await userChannelsAPI.getAvailable()
    const modelSet = new Set<string>()

    for (const channel of channels) {
      for (const platform of channel.platforms) {
        for (const model of platform.supported_models) {
          if (model.name) {
            modelSet.add(model.name)
          }
        }
      }
    }

    availableModels.value = Array.from(modelSet).sort()
    if (availableModels.value.length && !availableModels.value.includes(selectedModel.value)) {
      selectedModel.value = availableModels.value[0]
    }
    if (!availableModels.value.length) {
      selectedModel.value = ''
    }
  } catch {
    availableModels.value = []
    selectedModel.value = ''
  }
}

watch(currentConversationId, () => {
  nextTick(scrollToBottom)
})

watch(selectedKeyId, () => {
  fetchModels()
})

onMounted(async () => {
  loadConversations()
  await fetchApiKeys()
  await fetchModels()
})
</script>
