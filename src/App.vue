<template>
  <div class="parent-page-content">
    <h1>私の素晴らしいウェブサイト</h1>
    <p>これは親ページのコンテンツです。ここに他の情報や要素を追加できます。</p>
    <button @click="toggleChat" class="open-chat-button">チャットを開く</button>
  </div>

  <div id="app" :style="windowStyle" v-if="isChatOpen">
    <div class="chat-header" @mousedown="startDrag">
      <span>AI チャットボット</span>
      <button @click="toggleChat" class="close-button">X</button>
    </div>
    <div class="chat-window">
      <div class="messages" ref="messageContainer">
        <div
          v-for="(message, index) in messages"
          :key="index"
          :class="['message', message.sender]"
        >
          <p>{{ message.text }}</p>
        </div>
        <div v-if="isLoading" class="message ai loading">
          <p>AIが思考中...</p>
        </div>
      </div>
      <div class="input-area">
        <input
          type="text"
          v-model="userInput"
          @keyup.enter="sendMessage"
          :disabled="isLoading"
          placeholder="メッセージを入力..."
        />
        <button @click="sendMessage" :disabled="isLoading">送信</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, computed } from 'vue';
import { API_ENDPOINT } from './config.js';

const messages = ref([]);
const userInput = ref('');
const isLoading = ref(false);
const messageContainer = ref(null);
const isChatOpen = ref(false);

// ドラッグ機能関連のリアクティブ変数
const isDragging = ref(false);
const initialX = ref(0);
const initialY = ref(0);
const offsetX = ref(0);
const offsetY = ref(0);

// ウィンドウのスタイルを計算する算出プロパティ
const windowStyle = computed(() => ({
  left: `${offsetX.value}px`,
  top: `${offsetY.value}px`,
  cursor: isDragging.value ? 'grabbing' : 'grab',
}));

// チャットウィンドウの表示/非表示を切り替える
const toggleChat = () => {
  isChatOpen.value = !isChatOpen.value;
  if (isChatOpen.value) {
    nextTick(() => {
      centerWindow();
      scrollToBottom();
      // チャットが開いたときに初期メッセージを表示
      if (messages.value.length === 0) {
        messages.value.push({ text: 'こんにちは！何か質問はありますか？', sender: 'ai' });
      }
    });
  }
};

// ウィンドウを中央に配置
const centerWindow = () => {
  const appElement = document.getElementById('app');
  if (appElement) {
    offsetX.value = (window.innerWidth - appElement.offsetWidth) / 2;
    offsetY.value = (window.innerHeight - appElement.offsetHeight) / 2;
  }
};

// メッセージをスクロールする関数
const scrollToBottom = () => {
  nextTick(() => {
    if (messageContainer.value) {
      messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
    }
  });
};

// API 呼び出し
const sendMessage = async () => {
  if (!userInput.value.trim()) return;

  const userMessage = userInput.value;
  messages.value.push({ text: userMessage, sender: 'user' });
  userInput.value = '';
  isLoading.value = true;
  scrollToBottom();

  try {
    const response = await fetch(API_ENDPOINT, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ question: userMessage }),
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const data = await response.json();
    messages.value.push({ text: data.answer, sender: 'ai' });
  } catch (error) {
    console.error('API Error:', error);
    messages.value.push({ text: 'エラー: AIからの応答を取得できませんでした。', sender: 'ai error' });
  } finally {
    isLoading.value = false;
    scrollToBottom();
  }
};

// ドラッグ開始
const startDrag = (e) => {
  // クリックされた要素がクローズボタンでないことを確認
  if (e.target.classList.contains('close-button')) {
    return; // クローズボタンがクリックされた場合はドラッグを開始しない
  }

  isDragging.value = true;
  initialX.value = e.clientX - offsetX.value;
  initialY.value = e.clientY - offsetY.value;
  document.addEventListener('mousemove', doDrag);
  document.addEventListener('mouseup', stopDrag);
};

// ドラッグ中
const doDrag = (e) => {
  if (isDragging.value) {
    offsetX.value = e.clientX - initialX.value;
    offsetY.value = e.clientY - initialY.value;
  }
};

// ドラッグ終了
const stopDrag = () => {
  isDragging.value = false;
  document.removeEventListener('mousemove', doDrag);
  document.removeEventListener('mouseup', stopDrag);
};

onMounted(() => {
  // 初期メッセージはチャットが開いたときに表示されるように変更
});
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  flex-direction: column; /* 縦方向に要素を配置 */
  justify-content: flex-start; /* 上から配置 */
  align-items: center; /* 中央揃え */
  min-height: 100vh;
  background-color: #f0f2f5; /* 背景色を追加 */
  padding-top: 50px; /* 親ページコンテンツとチャットウィンドウの間にスペース */
}

.parent-page-content {
  text-align: center;
  margin-bottom: 30px; /* ボタンとチャットウィンドウの間にスペース */
}

.open-chat-button {
  padding: 15px 30px;
  font-size: 1.2em;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  box-shadow: 0 4px 8px rgba(0, 123, 255, 0.2);
  transition: background-color 0.3s ease, transform 0.2s ease;
}

.open-chat-button:hover {
  background-color: #0056b3;
  transform: translateY(-2px);
}

#app {
  position: fixed; /* ドラッグ可能にするためにfixed */
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
  background-color: #f9f9f9;
  box-sizing: border-box;
  width: 400px; /* チャットウィンドウの幅 */
  height: 500px; /* チャットウィンドウの高さ */
  display: flex;
  flex-direction: column;
  resize: both; /* リサイズ可能にする */
  overflow: hidden; /* リサイズ時にスクロールバーを表示 */
  z-index: 1000; /* 他の要素の上に表示 */
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background-color: #007bff;
  color: white;
  border-top-left-radius: 8px;
  border-top-right-radius: 8px;
  cursor: grab; /* ドラッグ可能であることを示す */
  font-weight: bold;
}

.chat-header span {
  flex-grow: 1;
  text-align: center;
}

.close-button {
  background: none;
  border: none;
  color: white;
  font-size: 1.2em;
  cursor: pointer;
  padding: 0 5px;
}

.chat-window {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  overflow: hidden; /* 親要素のoverflowをhiddenに */
}

.messages {
  flex-grow: 1;
  overflow-y: auto;
  padding: 10px;
  border-bottom: 1px solid #eee;
  text-align: left;
}

.message {
  margin-bottom: 10px;
  padding: 8px 12px;
  border-radius: 15px;
  max-width: 80%;
  word-wrap: break-word;
}

.message.user {
  background-color: #007bff;
  color: white;
  align-self: flex-end;
  margin-left: auto;
}

.message.ai {
  background-color: #e2e2e2;
  color: #333;
  align-self: flex-start;
  margin-right: auto;
}

.message.ai.error {
  background-color: #ffcccc;
  color: #cc0000;
}

.message.ai.loading {
  font-style: italic;
  color: #666;
}

.input-area {
  display: flex;
  padding: 10px;
  border-top: 1px solid #eee;
  flex-shrink: 0; /* 縮小しない */
}

.input-area input {
  flex-grow: 1;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-right: 10px;
}

.input-area button {
  padding: 8px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.input-area button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
</style>