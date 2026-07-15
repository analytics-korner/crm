<template>
  <LayoutHeader>
    <template #left-header>
      <div class="flex h-8 items-center text-lg-semibold text-ink-gray-8">
        {{ __('WhatsApp') }}
      </div>
    </template>
  </LayoutHeader>
  <div class="flex h-full overflow-hidden">
    <div class="flex w-80 shrink-0 flex-col overflow-y-auto border-r">
      <div
        v-if="conversations.loading"
        class="p-4 text-base text-ink-gray-5"
      >
        {{ __('Loading...') }}
      </div>
      <div
        v-else-if="!conversations.data?.length"
        class="p-4 text-base text-ink-gray-5"
      >
        {{ __('No WhatsApp conversations yet') }}
      </div>
      <div
        v-for="c in conversations.data"
        :key="`${c.reference_doctype}-${c.reference_name}`"
        class="cursor-pointer border-b px-4 py-3 hover:bg-surface-gray-2"
        :class="isSelected(c) ? 'bg-surface-gray-3' : ''"
        @click="selectConversation(c)"
      >
        <div class="flex items-center justify-between gap-2">
          <div class="truncate text-base-medium text-ink-gray-8">
            {{ c.title || c.mobile_no || c.reference_name }}
          </div>
          <div class="shrink-0 text-xs text-ink-gray-5">
            {{ __(timeAgo(c.last_message_at)) }}
          </div>
        </div>
        <div class="mt-1 truncate text-sm text-ink-gray-5">
          <span v-if="c.last_message_type == 'Outgoing'">{{ __('You') }}: </span
          >{{ c.last_message }}
        </div>
      </div>
    </div>

    <div class="flex flex-1 flex-col overflow-hidden">
      <template v-if="selected">
        <div
          class="mx-4 my-3 flex items-center justify-between text-lg-medium sm:mx-10 sm:mb-4 sm:mt-8"
        >
          <div class="flex h-8 items-center text-2xl-semibold text-ink-gray-8">
            {{ selectedTitle }}
          </div>
          <div class="flex shrink-0 gap-2">
            <Button
              :label="__('Send Template')"
              @click="showWhatsappTemplates = true"
            />
            <Button
              variant="solid"
              :label="__('New Message')"
              iconLeft="plus"
              @click="whatsappBox?.show()"
            />
          </div>
        </div>
        <FadedScrollableDiv class="flex-1 overflow-y-auto">
          <div v-if="messages.loading" class="px-3 text-ink-gray-5 sm:px-10">
            {{ __('Loading...') }}
          </div>
          <WhatsAppArea
            v-else
            v-model="messages"
            v-model:reply="replyMessage"
            class="px-3 sm:px-10"
            :messages="messages.data || []"
          />
        </FadedScrollableDiv>
        <WhatsAppBox
          ref="whatsappBox"
          v-model="doc"
          v-model:reply="replyMessage"
          v-model:whatsapp="messages"
          :doctype="selected.reference_doctype"
        />
        <WhatsappTemplateSelectorModal
          v-model="showWhatsappTemplates"
          :doctype="selected.reference_doctype"
          @send="(template) => sendTemplate(template)"
        />
      </template>
      <div
        v-else
        class="flex flex-1 items-center justify-center text-base text-ink-gray-5"
      >
        {{ __('Select a conversation to view messages') }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { createResource, Button, toast } from 'frappe-ui'
import { useTelemetry } from 'frappe-ui/frappe'
import LayoutHeader from '@/components/LayoutHeader.vue'
import FadedScrollableDiv from '@/components/FadedScrollableDiv.vue'
import WhatsAppArea from '@/components/Activities/WhatsAppArea.vue'
import WhatsAppBox from '@/components/Activities/WhatsAppBox.vue'
import WhatsappTemplateSelectorModal from '@/components/Modals/WhatsappTemplateSelectorModal.vue'
import { timeAgo } from '@/utils'
import { globalStore } from '@/stores/global'

const { $socket } = globalStore()
const { capture } = useTelemetry()

const selected = ref(null)
const doc = ref({})
const replyMessage = ref({})
const showWhatsappTemplates = ref(false)
const whatsappBox = ref(null)

const conversations = createResource({
  url: 'crm.api.whatsapp.list_whatsapp_conversations',
  cache: 'whatsapp_inbox_conversations',
  auto: true,
})

const messages = createResource({
  url: 'crm.api.whatsapp.get_whatsapp_messages',
})

const selectedTitle = computed(() => {
  if (!selected.value) return ''
  let c = conversations.data?.find(
    (c) =>
      c.reference_doctype === selected.value.reference_doctype &&
      c.reference_name === selected.value.reference_name,
  )
  return c?.title || c?.mobile_no || selected.value.reference_name
})

function isSelected(c) {
  return (
    selected.value &&
    c.reference_doctype === selected.value.reference_doctype &&
    c.reference_name === selected.value.reference_name
  )
}

async function selectConversation(c) {
  selected.value = {
    reference_doctype: c.reference_doctype,
    reference_name: c.reference_name,
  }
  replyMessage.value = {}
  doc.value = {}

  createResource({
    url: 'frappe.client.get',
    params: { doctype: c.reference_doctype, name: c.reference_name },
    auto: true,
    onSuccess: (data) => {
      doc.value = data
    },
  })

  messages.update({
    params: {
      reference_doctype: c.reference_doctype,
      reference_name: c.reference_name,
    },
  })
  await messages.reload()
}

function sendTemplate(template) {
  showWhatsappTemplates.value = false
  capture('whatsapp_send_template')
  createResource({
    url: 'crm.api.whatsapp.send_whatsapp_template',
    params: {
      reference_doctype: selected.value.reference_doctype,
      reference_name: selected.value.reference_name,
      to: doc.value.mobile_no,
      template,
    },
    auto: true,
    onError: (error) => {
      toast.error(error.messages?.[0] || __('Failed to send WhatsApp template'))
    },
    onSuccess: () => messages.reload(),
  })
}

function handleIncomingMessage(data) {
  conversations.reload()
  if (
    selected.value &&
    data.reference_doctype === selected.value.reference_doctype &&
    data.reference_name === selected.value.reference_name
  ) {
    messages.reload()
  }
}

onMounted(() => {
  $socket.on('whatsapp_message', handleIncomingMessage)
})

onUnmounted(() => {
  $socket.off('whatsapp_message', handleIncomingMessage)
})
</script>
