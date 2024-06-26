<script setup>
import { useStorage, useMutationObserver } from '@vueuse/core'
import MarkdownIt from "markdown-it";
import MarkdownItAbbr from "markdown-it-abbr";
import MarkdownItAnchor from "markdown-it-anchor";
import MarkdownItFootnote from "markdown-it-footnote";
import MarkdownItHighlightjs from "markdown-it-highlightjs";
import MarkdownItSub from "markdown-it-sub";
import MarkdownItSup from "markdown-it-sup";
import MarkdownItTasklists from "markdown-it-task-lists";
import MarkdownItTOC from "markdown-it-toc-done-right";
import {
  fetchHeadersOllama,
  fetchHeadersThirdApi,
  loadOllamaInstructions,
} from '@/utils/settings';

const props = defineProps({
  knowledgebase: Object
});

const markdown = new MarkdownIt()
  .use(MarkdownItAbbr)
  .use(MarkdownItAnchor)
  .use(MarkdownItFootnote)
  .use(MarkdownItHighlightjs)
  .use(MarkdownItSub)
  .use(MarkdownItSup)
  .use(MarkdownItTasklists)
  .use(MarkdownItTOC);

const instructions = ref([]);
const selectedInstruction = ref(null);
const chatInputBoxRef = shallowRef()

const model = useStorage(`model${props.knowledgebase?.id || ''}`, null);
const messages = ref([]);
const sending = ref(false);
let mainEl = null;

const visibleMessages = computed(() => {
  return messages.value.filter((message) => message.role !== 'system');
});

watch(model, async (newModel) => {
  messages.value = [];
})

const fetchStream = async (url, options) => {
  const response = await fetch(url, options);

  if (response.body) {
    const reader = response.body.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = new TextDecoder().decode(value);
      chunk.split("\n\n").forEach(async (line) => {
        if (line) {
          console.log('line: ', line);
          const chatMessage = JSON.parse(line);
          const content = chatMessage?.message?.content;
          if (content) {
            if (messages.value.length > 0 && messages.value[messages.value.length - 1].role === 'assistant') {
              messages.value[messages.value.length - 1].content += content;
            } else {
              messages.value.push({ role: 'assistant', content });
            }
          }
        }
      });
    }
  } else {
    console.log("The browser doesn't support streaming responses.");
  }
}

const onSend = async (data) => {
  const input = data.content.trim()
  if (sending.value || !input || !model.value) {
    return;
  }

  sending.value = true;
  chatInputBoxRef.value?.reset()

  if (selectedInstruction.value) {
    if (messages.value.length > 0 && messages.value[0].role === 'system') {
      messages.value[0].content = selectedInstruction.value.instruction;
    } else {
      messages.value = [{
        role: "system",
        content: selectedInstruction.value.instruction
      }, ...messages.value];
    }
  }

  messages.value.push({
    role: "user",
    content: input
  });

  const body = JSON.stringify({
    knowledgebaseId: props.knowledgebase?.id,
    model: model.value,
    messages: [...messages.value],
    stream: true,
  })
  await fetchStream('/api/models/chat', {
    method: 'POST',
    body: body,
    headers: {
      ...fetchHeadersOllama.value,
      ...fetchHeadersThirdApi.value,
      'Content-Type': 'application/json',
    }
  });

  sending.value = false;
}

onMounted(async () => {
  mainEl = document.getElementById('main');

  instructions.value = [(await loadOllamaInstructions()).map(i => {
    return {
      label: i.name,
      click: () => {
        selectedInstruction.value = i;
      }
    }
  })];
});


const messageListEl = ref(null);
useMutationObserver(messageListEl, () => {
  mainEl.scrollTo({
    top: mainEl.scrollHeight,
    behavior: 'smooth'
  });
}, { childList: true, subtree: true });

</script>

<template>
  <div class="flex flex-col flex-1 p-4 box-border dark:text-gray-300">
    <div class="flex items-center justify-between mb-4">
      <div class="flex items-center">
        <span class="mr-2">Chat with</span>
        <ModelsDropdown v-model="model" placeholder="Select a model" />
      </div>
      <div>
        <ClientOnly>
          <UDropdown :items="instructions" :popper="{ placement: 'bottom-start' }">
            <UButton color="white" :label="`${selectedInstruction ? selectedInstruction.name : 'Select Instruction'}`"
              trailing-icon="i-heroicons-chevron-down-20-solid" />
          </UDropdown>
        </ClientOnly>
      </div>
    </div>
    <div ref="messageListEl" dir="ltr" class="relative flex-1">
      <template v-for="(message, index) in visibleMessages" :key="index">
        <div class="text-gray-500 dark:text-gray-400">{{ message.role }}</div>
        <div
          :class="`${message.role == 'assistant' ? 'bg-white/10' : 'bg-primary-50 dark:bg-primary-400/20'} border border-primary/20 rounded mb-4 px-3 py-2`">
          <div v-html="markdown.render(message.content)" />
        </div>
      </template>
    </div>
    <div class="shrink-0 sticky bottom-2 pt-4">
      <ChatInputBox ref="chatInputBoxRef" :disabled="!model" :loading="sending" @submit="onSend" />
    </div>
  </div>
</template>

<style>
code {
  color: rgb(31, 64, 226);
  white-space: pre-wrap;
  margin: 0 0.4em;
}

.dark code {
  color: rgb(125, 179, 250);
}
</style>
