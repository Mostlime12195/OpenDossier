<script setup lang="ts">
import { ref, nextTick, watch, onMounted, onBeforeUnmount } from 'vue';
import { marked } from 'marked';

// --- STATE MANAGEMENT ---
const JINA_SEARCH_API = 'https://s.jina.ai/';
const JINA_READER_API = 'https://r.jina.ai/';
const HACK_CLUB_AI_API = 'https://ai.hackclub.com/chat/completions';

const view = ref<'initial' | 'chat'>('initial');
const jinaApiKey = ref('');
const isApiKeySaved = ref(false);
const showSettings = ref(false);

const initialQuery = ref('');
const followUpQuery = ref('');
const currentTopic = ref('');

const isLoading = ref(false);
const loadingStatus = ref('');

const sources = ref<Array<{ title: string; url: string }>>([]);
const messages = ref<Array<{ role: 'user' | 'assistant'; content: string }>>([]);

const chatHistoryEl = ref<HTMLElement | null>(null);
const chatInputEl = ref<HTMLTextAreaElement | null>(null);
const mainContentEl = ref<HTMLElement | null>(null);

// --- RESIZABLE PANEL LOGIC ---
const sidebarWidth = ref(380);
const isResizing = ref(false);

const handleMouseDown = (e: MouseEvent) => {
  e.preventDefault();
  isResizing.value = true;
  document.addEventListener('mousemove', handleMouseMove);
  document.addEventListener('mouseup', handleMouseUp);
};
const handleMouseMove = (e: MouseEvent) => {
  if (isResizing.value && mainContentEl.value) {
    const newWidth = e.clientX - 60; // 60px is the icon bar width
    sidebarWidth.value = Math.max(320, Math.min(newWidth, 700));
  }
};
const handleMouseUp = () => {
  isResizing.value = false;
  document.removeEventListener('mousemove', handleMouseMove);
  document.removeEventListener('mouseup', handleMouseUp);
};

// --- API & CORE LOGIC ---

/**
 * A robust fetch wrapper with error handling and flexible response parsing.
 * @param url The URL to fetch.
 * @param options The request options.
 * @returns The parsed JSON response or text content.
 */
async function robustFetch(url: string, options: RequestInit) {
  try {
    const response = await fetch(url, options);
    if (!response.ok) {
      const errorBody = await response.text();
      throw new Error(`API request failed with status ${response.status}: ${errorBody}`);
    }
    const contentType = response.headers.get('content-type');
    if (contentType && contentType.includes('application/json')) {
      return response.json();
    }
    return response.text();
  } catch (error) {
    if (error instanceof Error) {
      throw new Error(`Network error or failed request: ${error.message}`);
    }
    throw new Error('An unknown network error occurred.');
  }
}

/**
 * Uses the Hack Club AI to refine a user's query for better search results.
 * @param query The user's initial query.
 * @returns An optimized search query.
 */
async function optimizeQuery(query: string): Promise<string> {
  loadingStatus.value = 'Optimizing search query...';
  try {
    const response: any = await robustFetch(HACK_CLUB_AI_API, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        messages: [
          { role: 'system', content: 'You are a search query optimization assistant. Your task is to take a user\'s question or claim and rephrase it into a neutral, concise, and effective search engine query. Return only the optimized query text and nothing else.' },
          { role: 'user', content: query }
        ]
      }),
    });
    const optimized = response?.choices?.[0]?.message?.content?.trim();
    if (!optimized) {
      console.warn('Query optimization failed, using original query.');
      return query;
    }
    return optimized.replace(/["']/g, '');
  } catch (error) {
    console.error('Error optimizing query:', error);
    return query;
  }
}

async function handleInitialSubmit() {
  if (!initialQuery.value.trim() || isLoading.value) return;

  resetState(true);
  isLoading.value = true;
  view.value = 'chat';
  currentTopic.value = initialQuery.value;
  messages.value.push({ role: 'user', content: initialQuery.value });

  try {
    const optimizedQuery = await optimizeQuery(initialQuery.value);

    loadingStatus.value = 'Searching for relevant sources...';
    const searchUrl = `${JINA_SEARCH_API}${encodeURIComponent(optimizedQuery)}`;
    const searchResponse = await robustFetch(searchUrl, {
      headers: { 'Authorization': `Bearer ${jinaApiKey.value.trim()}`, 'Accept': 'application/json' },
    });

    if (!searchResponse?.data || !Array.isArray(searchResponse.data) || searchResponse.data.length === 0) {
      throw new Error("Could not find any relevant sources. Try rephrasing your query.");
    }
    const topSources = searchResponse.data.slice(0, 5);
    sources.value = topSources;

    loadingStatus.value = `Reading ${topSources.length} sources...`;
    const readerPromises = topSources.map(source =>
      robustFetch(`${JINA_READER_API}${source.url}`, {
        headers: { 'Authorization': `Bearer ${jinaApiKey.value.trim()}`, 'Accept': 'application/json' },
      })
    );
    const readerResults = await Promise.allSettled(readerPromises);

    let combinedContext = '';
    let successfulReads = 0;
    readerResults.forEach((result, index) => {
      if (result.status === 'fulfilled' && result.value) {
        const content = result.value?.data?.content || result.value?.content || '';
        if (content) {
          successfulReads++;
          combinedContext += `--- SOURCE [${index + 1}] ---\nTitle: ${topSources[index].title}\nURL: ${topSources[index].url}\n\n${content}\n\n`;
        }
      } else if (result.status === 'rejected') {
        console.warn(`Failed to read source [${index + 1}]: ${topSources[index].url}`, result.reason);
      }
    });

    if (successfulReads === 0) {
      throw new Error("Found sources, but was unable to read their content. They may be inaccessible or in an unsupported format.");
    }

    loadingStatus.value = 'Synthesizing information...';
    const systemPrompt = `You are a neutral research assistant for OpenDossier. Your goal is to provide a comprehensive, evidence-based answer to the user's question based *only* on the provided source materials. Synthesize information from all sources, quote them with markdown, and cite them like [1], [2], etc. If the answer isn't in the sources, say so. Note: The source content may be truncated.`;
    const userPrompt = `User's Question: "${initialQuery.value}"\n\n--- PROVIDED SOURCES ---\n${combinedContext}`;

    await streamAiResponse([{ role: 'system', content: systemPrompt }, { role: 'user', content: userPrompt }]);

  } catch (e: any) {
    messages.value.push({ role: 'assistant', content: `**An error occurred during research:**\n\n${e.message}` });
  } finally {
    isLoading.value = false;
    loadingStatus.value = '';
  }
}

async function handleFollowUpSubmit() {
  if (!followUpQuery.value.trim() || isLoading.value) return;
  const userMessage = followUpQuery.value;
  messages.value.push({ role: 'user', content: userMessage });
  followUpQuery.value = '';
  isLoading.value = true;

  try {
    const history = messages.value.slice(0, -1);
    const systemPrompt = { role: 'system', content: "You are a helpful research assistant. Continue the conversation based on the previous context." };
    await streamAiResponse([systemPrompt, ...history, { role: 'user', content: userMessage }]);
  } catch (e: any) {
    messages.value.push({ role: 'assistant', content: `**An error occurred:**\n\n${e.message}` });
  } finally {
    isLoading.value = false;
  }
}

/**
 * Processes a non-standard stream of concatenated JSON objects from the Hack Club AI API.
 * @param messageHistory The conversation history to send to the AI.
 */
async function streamAiResponse(messageHistory: Array<{ role: string; content: string }>) {
  let assistantResponse = '';
  messages.value.push({ role: 'assistant', content: '' });
  const lastMessageIndex = messages.value.length - 1;

  try {
    const response = await fetch(HACK_CLUB_AI_API, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messages: messageHistory, stream: true }),
    });

    if (!response.ok || !response.body) {
      const errorText = await response.text();
      throw new Error(`AI model request failed: ${errorText}`);
    }

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      // Add the new data chunk to our buffer.
      buffer += decoder.decode(value, { stream: true });

      // The API sends a stream of multiple JSON objects concatenated together,
      // without any newlines. We need to find the boundary of each complete object.
      while (buffer.length > 0) {
        let objectEndIndex = -1;
        let openBraces = 0;
        let inString = false;

        // This manual parser finds the end of the first complete JSON object
        // by tracking curly braces, ignoring any inside of strings.
        for (let i = 0; i < buffer.length; i++) {
          const char = buffer[i];
          if (char === '"' && (i === 0 || buffer[i - 1] !== '\\')) {
            inString = !inString;
          }
          if (!inString) {
            if (char === '{') openBraces++;
            else if (char === '}') {
              openBraces--;
              if (openBraces === 0) {
                objectEndIndex = i + 1;
                break;
              }
            }
          }
        }

        // If no complete object is found, we need more data from the stream.
        if (objectEndIndex === -1) break;

        const jsonStr = buffer.slice(0, objectEndIndex);
        buffer = buffer.slice(objectEndIndex); // Remove the processed object from the buffer.

        try {
          const parsed = JSON.parse(jsonStr);
          const content = parsed.choices?.[0]?.delta?.content || '';
          assistantResponse += content;
          messages.value[lastMessageIndex].content = assistantResponse;

          // The API signals the end with a "stop" reason.
          if (parsed.choices?.[0]?.finish_reason === 'stop') {
            return; // Gracefully exit the function.
          }
        } catch (e) {
          console.warn("Failed to parse JSON object from stream:", jsonStr, e);
        }
      }
    }
  } catch (e: any) {
    messages.value[lastMessageIndex].content += `\n\n**An error occurred:**\n\n${e.message}`;
  } finally {
    isLoading.value = false;
  }
}

function saveApiKey() {
  if (jinaApiKey.value.trim()) {
    localStorage.setItem('jinaApiKey', jinaApiKey.value.trim());
    isApiKeySaved.value = true;
    showSettings.value = false;
  }
}

function resetState(keepQuery = false) {
  view.value = 'initial';
  if (!keepQuery) {
    initialQuery.value = '';
  }
  followUpQuery.value = '';
  currentTopic.value = '';
  sources.value = [];
  messages.value = [];
  isLoading.value = false;
}

// --- UI HELPERS & LIFECYCLE ---
watch(messages, async () => {
  await nextTick();
  if (chatHistoryEl.value) {
    chatHistoryEl.value.scrollTop = chatHistoryEl.value.scrollHeight;
  }
}, { deep: true });

watch(followUpQuery, async () => {
  if (chatInputEl.value) {
    await nextTick();
    chatInputEl.value.style.height = 'auto';
    chatInputEl.value.style.height = `${chatInputEl.value.scrollHeight}px`;
  }
});

onMounted(() => {
  const savedKey = localStorage.getItem('jinaApiKey');
  if (savedKey) {
    jinaApiKey.value = savedKey;
    isApiKeySaved.value = true;
  }
});

onBeforeUnmount(() => {
  document.removeEventListener('mousemove', handleMouseMove);
  document.removeEventListener('mouseup', handleMouseUp);
});
</script>

<template>
  <div class="app-layout" :class="{ 'is-resizing': isResizing }">
    <nav class="icon-bar">
      <div class="icon-logo" title="OpenDossier">
        <svg viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path
            d="M4 6C4 4.89543 4.89543 4 6 4H18C19.1046 4 20 4.89543 20 6V18C20 19.1046 19.1046 20 18 20H6C4.89543 20 4 19.1046 4 18V6Z"
            stroke="currentColor" stroke-width="2.5" />
          <path d="M9 9.5H15" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
          <path d="M9 14.5H12" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
        </svg>
      </div>
      <button class="icon-button" @click="resetState(false)" title="New Research">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M12 5V19M5 12H19" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round" />
        </svg>
      </button>
      <button class="icon-button" @click="showSettings = true" title="Settings">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M5 12H19" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
          <path d="M5 17H13" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
          <path d="M11 7H19" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </button>
    </nav>

    <div class="main-content-wrapper" ref="mainContentEl">
      <div v-if="!isApiKeySaved" class="welcome-view">
        <div class="welcome-box">
          <h1>Welcome to OpenDossier</h1>
          <p>To begin, please enter your Jina AI API key. This is required to search for and read online sources.</p>
          <form @submit.prevent="saveApiKey" class="api-key-form">
            <input v-model="jinaApiKey" type="password" placeholder="Enter your Jina AI key..." />
            <button type="submit" :disabled="!jinaApiKey.trim()">Save and Continue</button>
          </form>
        </div>
      </div>

      <div v-else-if="view === 'initial'" class="initial-view">
        <div class="initial-box">
          <h1>Start a New Research Topic</h1>
          <p>Enter a statement or question. The AI will find and read credible sources to provide a neutral,
            evidence-based summary.</p>
          <form @submit.prevent="handleInitialSubmit" class="initial-form">
            <textarea v-model="initialQuery"
              placeholder="e.g., What are the economic impacts of universal basic income?"
              @keydown.enter.exact.prevent="handleInitialSubmit"></textarea>
            <button type="submit" :disabled="isLoading || !initialQuery.trim()">
              <span v-if="isLoading">{{ loadingStatus || 'Researching...' }}</span>
              <span v-else>Start Research</span>
            </button>
          </form>
        </div>
      </div>

      <div v-else class="chat-view">
        <aside class="sidebar" :style="{ width: `${sidebarWidth}px` }">
          <div class="sidebar-header">
            <h2>Research Context</h2>
          </div>
          <div class="sidebar-content">
            <div class="sidebar-section">
              <h3>Initial Query</h3>
              <p>{{ currentTopic }}</p>
            </div>
            <div class="sidebar-section sources-section">
              <h3>Sources Found</h3>
              <div class="sources-list">
                <a v-for="(source, index) in sources" :key="source.url" :href="source.url" target="_blank"
                  class="source-item">
                  <span class="source-index">[{{ index + 1 }}]</span>
                  <span class="source-title">{{ source.title }}</span>
                </a>
              </div>
            </div>
          </div>
        </aside>

        <div class="resizer" @mousedown="handleMouseDown"></div>

        <main class="chat-panel">
          <div class="chat-history" ref="chatHistoryEl">
            <div v-for="(message, index) in messages" :key="index" :class="['message-bubble', message.role]">
              <div class="message-avatar" :class="message.role">
                <!-- NEW ICON: User -->
                <svg v-if="message.role === 'user'" width="24" height="24" viewBox="0 0 24 24" fill="none"
                  xmlns="http://www.w3.org/2000/svg">
                  <path
                    d="M12 12C14.2091 12 16 10.2091 16 8C16 5.79086 14.2091 4 12 4C9.79086 4 8 5.79086 8 8C8 10.2091 9.79086 12 12 12Z"
                    stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                  <path d="M19.2352 20C19.2352 16.5425 15.9928 14 12 14C8.00719 14 4.76471 16.5425 4.76471 20"
                    stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                </svg>
                <!-- NEW ICON: Assistant -->
                <svg v-else width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M12 3L10.5 8L6 9.5L10.5 11L12 16L13.5 11L18 9.5L13.5 8L12 3Z" stroke="currentColor"
                    stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                  <path d="M18 17L17 19L15 20L17 21L18 23L19 21L21 20L19 19L18 17Z" stroke="currentColor"
                    stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                  <path d="M6 17L5 19L3 20L5 21L6 23L7 21L9 20L7 19L6 17Z" stroke="currentColor" stroke-width="2"
                    stroke-linecap="round" stroke-linejoin="round" />
                </svg>
              </div>
              <div class="message-content" v-html="marked(message.content)"></div>
            </div>
            <div v-if="isLoading && loadingStatus" class="message-bubble assistant">
              <div class="message-avatar assistant">
                <!-- NEW ICON: Assistant (Loading) -->
                <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M12 3L10.5 8L6 9.5L10.5 11L12 16L13.5 11L18 9.5L13.5 8L12 3Z" stroke="currentColor"
                    stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                  <path d="M18 17L17 19L15 20L17 21L18 23L19 21L21 20L19 19L18 17Z" stroke="currentColor"
                    stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                  <path d="M6 17L5 19L3 20L5 21L6 23L7 21L9 20L7 19L6 17Z" stroke="currentColor" stroke-width="2"
                    stroke-linecap="round" stroke-linejoin="round" />
                </svg>
              </div>
              <div class="message-content">
                <div class="loading-status">{{ loadingStatus }}</div>
                <div class="typing-indicator"><span></span><span></span><span></span></div>
              </div>
            </div>
          </div>
          <div class="chat-form-container">
            <form @submit.prevent="handleFollowUpSubmit" class="chat-form">
              <textarea ref="chatInputEl" v-model="followUpQuery" placeholder="Ask a follow-up question..."
                class="chat-input" :disabled="isLoading" @keydown.enter.exact.prevent="handleFollowUpSubmit"></textarea>
              <button type="submit" class="chat-button" :disabled="isLoading || !followUpQuery.trim()">
                <!-- NEW ICON: Send -->
                <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                  <path d="M21 5L2 12.5L9 13.5M21 5L15.5 22L9 13.5M21 5L9 13.5" stroke="currentColor" stroke-width="2"
                    stroke-linecap="round" stroke-linejoin="round" />
                </svg>
              </button>
            </form>
          </div>
        </main>
      </div>
    </div>

    <div v-if="showSettings" class="modal-overlay" @click.self="showSettings = false">
      <div class="modal-box">
        <h2>Settings</h2>
        <p>Manage your Jina AI API key here. Your key is saved securely in your browser's local storage.</p>
        <form @submit.prevent="saveApiKey" class="api-key-form">
          <label for="settingsApiKey">Jina AI API Key</label>
          <input id="settingsApiKey" v-model="jinaApiKey" type="password" />
          <div class="modal-actions">
            <button type="button" class="btn-secondary" @click="showSettings = false">Cancel</button>
            <button type="submit" :disabled="!jinaApiKey.trim()">Save</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<style>
/* --- Global Styles & Theme --- */
:root {
  --bg-color: #FFFAF0;
  /* FloralWhite */
  --sidebar-bg-color: #FBF5EB;
  /* A slightly darker warm tone for contrast */
  --panel-bg-color: #FFFFFF;
  --text-color: #3D3D3D;
  --text-light: #7A7A7A;
  --border-color: #EAE1D6;
  /* More visible border */
  --accent-color: #E57373;
  /* Soft Red */
  --accent-color-warm-hover: #F0A07B;
  /* Soft, warm orange for hover */
  --user-msg-bg: #FFF3E0;
  /* PapayaWhip */
  --assistant-msg-bg: #F5F5F5;
  /* WhiteSmoke */
  --shadow-soft: 0 2px 8px rgba(0, 0, 0, 0.04);
  --font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --transition-fast: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 20px;
  --space-6: 24px;
  --space-8: 32px;
  --space-10: 40px;
  --space-15: 60px;
}

@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap');

/* --- Base Layout --- */
*,
*::before,
*::after {
  box-sizing: border-box;
}

html,
body {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
  overflow: hidden;
  font-family: var(--font-family);
  background-color: var(--bg-color);
}

.app-layout {
  display: grid;
  grid-template-columns: var(--space-15) 1fr;
  height: 100vh;
  width: 100vw;
}

.app-layout.is-resizing {
  cursor: col-resize;
  user-select: none;
}

.icon-bar {
  background-color: var(--sidebar-bg-color);
  border-right: 1px solid var(--border-color);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: var(--space-4) 0;
  gap: var(--space-4);
}

.main-content-wrapper {
  display: flex;
  flex-direction: column;
  min-width: 0;
  overflow: hidden;
}

/* --- Welcome & Initial Views --- */
.welcome-view,
.initial-view {
  flex: 1;
  display: grid;
  place-items: center;
  padding: var(--space-6);
}

.welcome-box,
.initial-box {
  max-width: 600px;
  width: 100%;
  background: var(--panel-bg-color);
  padding: var(--space-10);
  border-radius: 12px;
  border: 1px solid var(--border-color);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.05);
}

.welcome-box h1,
.initial-box h1 {
  font-size: 1.8rem;
  color: var(--text-color);
  margin: 0 0 var(--space-2);
}

.welcome-box p,
.initial-box p {
  color: var(--text-light);
  line-height: 1.6;
  margin: 0 0 var(--space-6);
}

.api-key-form,
.initial-form {
  display: flex;
  flex-direction: column;
  gap: var(--space-4);
}

.api-key-form input,
.initial-form textarea {
  width: 100%;
  padding: var(--space-3);
  font-size: 1rem;
  border: 1px solid #E0E0E0;
  border-radius: 8px;
  transition: var(--transition-fast);
  background-color: #FAFAFA;
}

.api-key-form input:focus,
.initial-form textarea:focus {
  border-color: var(--accent-color);
  box-shadow: 0 0 0 3px rgba(229, 115, 115, 0.15);
  background-color: white;
}

.initial-form textarea {
  min-height: 100px;
  resize: vertical;
}

button {
  padding: var(--space-3) var(--space-4);
  font-size: 1rem;
  font-weight: 600;
  background-color: var(--accent-color);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: var(--transition-fast);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);
}

button:disabled {
  background-color: #BDBDBD;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: var(--accent-color-warm-hover);
}

/* --- Chat View Layout --- */
.chat-view {
  flex: 1;
  display: flex;
  min-height: 0;
}

.sidebar {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  background-color: var(--sidebar-bg-color);
}

.sidebar-header {
  padding: var(--space-5) var(--space-6);
  border-bottom: 1px solid var(--border-color);
}

.sidebar-header h2 {
  font-size: 1.1rem;
  font-weight: 600;
  margin: 0;
}

.sidebar-content {
  flex: 1;
  overflow-y: auto;
  min-height: 0;
  padding: var(--space-4) 0;
}

.sidebar-section {
  padding: var(--space-4) var(--space-6);
}

.sidebar-section h3 {
  font-size: 0.85rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: var(--text-light);
  margin: 0 0 var(--space-3);
}

.sidebar-section p {
  font-size: 0.9rem;
  color: var(--text-color);
  line-height: 1.5;
  margin: 0;
}

.sources-list {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
}

.source-item {
  display: flex;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: 8px;
  text-decoration: none;
  color: var(--text-color);
  transition: var(--transition-fast);
}

.source-item:hover {
  background-color: var(--border-color);
}

.source-index {
  color: var(--accent-color);
  font-weight: 600;
}

.source-title {
  font-size: 0.9rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.resizer {
  width: 4px;
  flex-shrink: 0;
  background-color: var(--border-color);
  cursor: col-resize;
  transition: var(--transition-fast);
}

.resizer:hover {
  background-color: var(--accent-color);
}

/* --- Chat Panel --- */
.chat-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
  background-color: var(--panel-bg-color);
}

.chat-history {
  flex: 1;
  padding: var(--space-8);
  overflow-y: auto;
  min-height: 0;
}

.message-bubble {
  display: flex;
  gap: var(--space-3);
  max-width: 85%;
  animation: message-fade-in 0.4s cubic-bezier(0.4, 0, 0.2, 1);
}

.message-bubble+.message-bubble {
  margin-top: var(--space-6);
}

.message-bubble.user {
  margin-left: auto;
  flex-direction: row-reverse;
}

.message-avatar {
  flex-shrink: 0;
  width: var(--space-8);
  height: var(--space-8);
  border-radius: 50%;
  display: grid;
  place-items: center;
  margin-top: var(--space-1);
  /* NEW: Soft background colors for avatars */
  background-color: var(--user-msg-bg);
  color: #E65100;
  /* Dark Orange */
}

.message-avatar.assistant {
  background-color: #E8F5E9;
  /* Soft Green */
  color: #2E7D32;
  /* Dark Green */
}

.message-avatar svg {
  width: 20px;
  height: 20px;
}

.message-content {
  padding: var(--space-3) var(--space-4);
  border-radius: 12px;
  line-height: 1.6;
  background-color: var(--assistant-msg-bg);
  border: 1px solid #E0E0E0;
  overflow-wrap: break-word;
  word-break: break-word;
}

.message-bubble.user .message-content {
  background-color: var(--user-msg-bg);
}

.message-content *:first-child {
  margin-top: 0;
}

.message-content *:last-child {
  margin-bottom: 0;
}

.message-content blockquote {
  border-left: 3px solid var(--accent-color);
  padding-left: 1em;
  margin-left: 0;
  color: var(--text-light);
}

.chat-form-container {
  padding: var(--space-4) var(--space-8);
  border-top: 1px solid var(--border-color);
  background-color: #F9F9F9;
}

.chat-form {
  display: flex;
  align-items: flex-end;
  background-color: var(--panel-bg-color);
  border-radius: 12px;
  border: 1px solid #E0E0E0;
  padding: var(--space-2);
  transition: var(--transition-fast);
}

.chat-form:focus-within {
  border-color: var(--accent-color);
  box-shadow: 0 0 0 3px rgba(229, 115, 115, 0.15);
}

.chat-input {
  flex: 1;
  padding: var(--space-2);
  border: none;
  background: transparent;
  font-size: 1rem;
  outline: none;
  resize: none;
  font-family: inherit;
  max-height: 150px;
  color: var(--text-color);
}

.chat-button {
  height: var(--space-8);
  width: var(--space-8);
  border-radius: 8px;
  padding: 0;
}

.chat-button svg {
  width: 22px;
  height: 22px;
}

/* --- Icons & Modal --- */
.icon-logo {
  color: var(--accent-color);
  padding: var(--space-2);
}

.icon-logo svg {
  width: var(--space-8);
  height: var(--space-8);
}

.icon-button {
  background: none;
  border: none;
  cursor: pointer;
  width: var(--space-10);
  height: var(--space-10);
  border-radius: var(--space-2);
  color: var(--text-light);
  transition: var(--transition-fast);
  display: grid;
  place-items: center;
  padding: 0;
}

.icon-button:hover {
  background-color: var(--border-color);
  color: var(--text-color);
}

.icon-button svg {
  width: 26px;
  height: 26px;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.4);
  display: grid;
  place-items: center;
  animation: fade-in 0.2s ease;
  z-index: 100;
}

.modal-box {
  background: var(--panel-bg-color);
  padding: var(--space-8);
  border-radius: 12px;
  max-width: 500px;
  width: 90%;
  animation: scale-in 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  border: 1px solid var(--border-color);
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.modal-box h2 {
  margin-top: 0;
}

.modal-box p {
  color: var(--text-light);
}

.modal-box label {
  display: block;
  font-weight: 500;
  margin-bottom: var(--space-2);
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: var(--space-3);
  margin-top: var(--space-6);
}

.modal-actions button {
  padding: var(--space-2) var(--space-4);
}

.modal-actions .btn-secondary {
  background-color: #F5F5F5;
  color: var(--text-color);
  border: 1px solid #E0E0E0;
}

.modal-actions .btn-secondary:hover {
  background-color: #EEEEEE;
}

/* --- Animations & Loading States --- */
@keyframes fade-in {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}

@keyframes scale-in {
  from {
    opacity: 0;
    transform: scale(0.95);
  }

  to {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes message-fade-in {
  from {
    opacity: 0;
    transform: translateY(10px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.loading-status {
  font-size: 0.9rem;
  color: var(--text-light);
  margin-bottom: var(--space-2);
}

.typing-indicator span {
  height: 8px;
  width: 8px;
  background-color: #BDBDBD;
  border-radius: 50%;
  display: inline-block;
  animation: wave 1.3s infinite;
  margin: 0 2px;
}

.typing-indicator span:nth-of-type(2) {
  animation-delay: 0.2s;
}

.typing-indicator span:nth-of-type(3) {
  animation-delay: 0.4s;
}

@keyframes wave {

  0%,
  60%,
  100% {
    transform: initial;
  }

  30% {
    transform: translateY(-8px);
  }
}
</style>
