<template>
  <NuxtLayout :name="'app-wrapper'">
    <template #main>
      <div class="flex h-screen flex-col lg:h-[92vh]">
        <div class="flex items-center justify-center py-4 lg:shadow-sm">
          <Logo class="h-6 w-6" />
          <h2 class="ml-4 text-xl font-bold">UCC MindCare</h2>
        </div>

        <div
          v-if="!userToken"
          class="m-4 flex w-fit items-center justify-center rounded-3xl bg-blue-200 px-4 py-3"
        >
          <p class="message-text">
            <strong>Quote:</strong> {{ quoteOfTheDay }}
          </p>
        </div>

        <div
          v-if="userToken"
          ref="chatBodyRef"
          class="chat-body flex-1 overflow-y-auto px-2 py-4"
        >
          <div class="block lg:hidden">
            <ChatbotAlarm />
          </div>
          <div
            class="mt-2 flex w-fit items-start space-x-3 rounded-lg border border-gray-200 bg-blue-50 px-4 py-3"
          >
            <p class="message-text">
              <strong>Hello! {{ userData?.given_name }}.</strong> <br />
              How can I assist you in your mental wellness journey today?
            </p>
          </div>
          <div class="mt-4 flex h-[10vh] flex-col gap-4">
            <LazyChatbotMessage
              v-for="(chat, index) in chatHistory"
              :key="index"
              :chat="chat"
            />
          </div>
        </div>

        <CommonResourceLibrary
          v-if="userToken"
          :userToken="userToken"
          :showResourceLibrary="showResourceLibrary"
          :toggleResourceLibrary="
            () => (showResourceLibrary = !showResourceLibrary)
          "
        />

        <div
          v-else
          class="flex h-full flex-col items-center justify-center gap-2"
        >
          <img
            src="https://img.freepik.com/free-vector/doctor-hospital-healthcare-staff-work-front-side-rear-view-medic-male-character-white-robe-with-stethoscope-neck-take-gloves-hold-xray-cartoon-linear-flat-vector-illustration-set_107791-11919.jpg?t=st=1738385166~exp=1738388766~hmac=fc4d88a9d35f98046c1d499b871fe0489601dd62201c5eb1aca4d2bdeb9dbd6f&w=1380"
            alt=""
          />
          <h1 class="text-md font-bold text-black">
            Mental Health | Support | Counseling
          </h1>
          <p>Sign in with Google to start a conversation</p>
          <GoogleAuth />
          <p class="py-10 text-sm text-black">
            © 2025, Group 5, UCC MindCare.
          </p>
        </div>

        <div
          v-if="userToken"
          class="mb-4 flex flex-col items-center justify-center gap-4 bg-white p-4"
        >
          <form
            @submit.prevent="handleSubmit"
            class="flex w-full max-w-3xl items-center gap-4"
          >
            <UDropdown
              :items="exploredTopics"
              :ui="{ item: { disabled: 'cursor-text select-text' } }"
              :popper="{ arrow: true }"
            >
              <img
                src="/assets/icons/menu.svg"
                alt="menu"
                class="block cursor-pointer xl:block 2xl:hidden"
              />
            </UDropdown>

            <input
              v-model="inputRef"
              placeholder="Message..."
              class="w-full rounded-3xl border border-gray-200 p-2 outline-none"
              required
            />
            <button type="submit">
              <img src="/assets/icons/send-msg.svg" alt="Send" />
            </button>
          </form>
          <p class="text-sm text-gray-500">
            Type "clear" for new chat or "logout" to sign out
          </p>
        </div>
      </div>
    </template>

    <template #sidebar>
      <CommonSidebar
        :userData="userData"
        :handleExploredTopicSelection="handleExploredTopicSelection"
      />
    </template>
  </NuxtLayout>
</template>

<script setup>
import { generateBotResponse } from '@/utils/botResponse'
import { companyInfo } from '@/prompts/chatbot'
import Logo from '@/assets/icons/ChatbotIcon.vue'
import GoogleAuth from '@/pages/auth/google-auth.vue'
import { quotes } from '@/constants/index'

const chatBodyRef = ref(null)
const inputRef = ref('')
const userToken = ref(null)
const userData = ref(null)
const moodTracker = ref(null)
const chatHistory = ref([])
const showResourceLibrary = ref(false)
const quoteOfTheDay = ref(quotes[0])
let quoteInterval = null

onMounted(() => {
  userToken.value = localStorage.getItem('userToken')
  userData.value = JSON.parse(localStorage.getItem('userData')) || null
  moodTracker.value = JSON.parse(localStorage.getItem('moodTracker')) || null

  if (userData.value?.email) {
    const savedChatHistory = localStorage.getItem(
      `chatHistory_${userData.value.email}`
    )
    chatHistory.value = savedChatHistory
      ? JSON.parse(savedChatHistory)
      : [{ hideInChat: true, role: 'model', text: companyInfo }]
  }

  let index = 0
  quoteInterval = setInterval(() => {
    index = (index + 1) % quotes.length
    quoteOfTheDay.value = quotes[index]
  }, 10000)
})

// Save chat history to localStorage
const saveChatHistory = () => {
  if (!userData.value?.email) return
  localStorage.setItem(
    `chatHistory_${userData.value.email}`,
    JSON.stringify(chatHistory.value)
  )
}

// Update chat history
const setChatHistory = fn => {
  chatHistory.value = fn(chatHistory.value)
  saveChatHistory()
}

const handleSubmit = () => {
  setTimeout(() => {
    chatBodyRef.value?.scrollTo({
      top: chatBodyRef.value.scrollHeight,
      behavior: 'smooth'
    })
  }, 100)

  if (!inputRef.value.trim()) return
  if (!userData.value?.email) return

  const userMessage = inputRef.value
  const userEmail = userData.value.email

  if (userMessage.toLowerCase() === 'clear') {
    localStorage.removeItem(`chatHistory_${userData.value.email}`)
    chatHistory.value = [{ hideInChat: true, role: 'model', text: companyInfo }]
    inputRef.value = ''
    return
  } else if (userMessage.toLowerCase() === 'logout') {
    localStorage.removeItem('userToken')
    localStorage.removeItem('userData')
    window.location.reload()
    return
  }

  inputRef.value = ''

  setChatHistory(history => {
    const updatedHistory = [
      ...history,
      { role: 'user', text: userMessage, email: userEmail }
    ]
    const thinkingHistory = [
      ...updatedHistory,
      { role: 'model', text: 'Thinking...', email: userEmail }
    ]

    let userModdText = ''
    if (moodTracker.value) {
      userModdText = `When a user want help with their mood use ${moodTracker.value.mood}. Assist them with their mood till they are satisfied. Make your conversation with the mood concise.`
    }

    setTimeout(() => {
      generateBotResponse(
        [
          ...thinkingHistory.filter(msg => msg.text !== 'Thinking...'),
          {
            role: 'user',
            text: `Using the details provided above, please address this query: ${userMessage}. When you want to mention the user, the name is ${userData.value.given_name} or you can address them by their full name ${userData.value.given_name} ${userData.value.family_name}. ${userModdText}`
          }
        ],
        chatHistory
      )
    }, 600)

    return thinkingHistory
  })
}

const handleExploredTopicSelection = topic => {
  if (topic === 'Resource Library') {
    showResourceLibrary.value = !showResourceLibrary.value
  }
  setTimeout(() => {
    chatBodyRef.value?.scrollTo({
      top: chatBodyRef.value.scrollHeight,
      behavior: 'smooth'
    })
  }, 100)

  if (!userToken.value) return

  setChatHistory(history => {
    const updatedHistory = [...history, { role: 'user', text: topic }]
    const thinkingHistory = [
      ...updatedHistory,
      { role: 'model', text: 'Thinking...' }
    ]

    setTimeout(() => {
      generateBotResponse(
        [
          ...thinkingHistory.filter(msg => msg.text !== 'Thinking...'),
          {
            role: 'user',
            text: `Use information in ${companyInfo} to answer ${topic} for the support and service you provide on mental health`
          }
        ],
        chatHistory
      )
    }, 600)

    return thinkingHistory
  })
}

const exploredTopics = [
  [
    {
      label: 'Self-Assessment Tools',
      click: () => handleExploredTopicSelection('Self-Assessment Tools')
    },
    {
      label: 'Resource Library',
      click: () => handleExploredTopicSelection('Resource Library')
    },
    {
      label: 'Guided Relaxation',
      click: () => handleExploredTopicSelection('Guided Relaxation')
    },
    {
      label: 'Emergency Support',
      click: () => handleExploredTopicSelection('Emergency Support')
    }
  ]
]
</script>
