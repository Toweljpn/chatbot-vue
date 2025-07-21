<template>
  <div class="parent-page-content">
    <h2>AI Chatbot(Vue)</h2>
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
          placeholder="質問を入力してください"
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
        messages.value.push({ text: 'こちらでは、人形小辞典に掲載されている内容をもとにAIが回答します。\n内容は必ずしも正しいものとは限りませんことをご了承ください。', sender: 'ai' });
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
/* 全体的なスタイル */
body {
  margin: 0;
  padding: 0;
  font-family: 'Segoe UI', 'Roboto', 'Helvetica Neue', Arial, sans-serif; /* よりモダンなフォント */
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  min-height: 100vh;
  background: linear-gradient(to right, #ece9e6, #ffffff); /* 柔らかいグラデーション背景 */
  padding-top: 50px;
  color: #333;
}

.parent-page-content {
  text-align: center;
  margin-bottom: 40px;
  padding: 20px;
  background-color: #ffffff;
  border-radius: 12px;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.08); /* より洗練されたシャドウ */
  max-width: 600px;
  width: 90%;
}

.parent-page-content h2 {
  color: #2c3e50;
  font-size: 2.2em;
  margin-bottom: 15px;
}

.parent-page-content p {
  color: #555;
  font-size: 1.1em;
  line-height: 1.6;
}

/* チャットを開くボタン */
.open-chat-button {
  padding: 15px 35px;
  font-size: 1.2em;
  background: linear-gradient(45deg, #6a11cb 0%, #2575fc 100%); /* グラデーション */
  color: white;
  border: none;
  border-radius: 30px; /* より丸く */
  cursor: pointer;
  box-shadow: 0 8px 25px rgba(0, 123, 255, 0.3); /* 強めのシャドウ */
  transition: all 0.3s ease;
  letter-spacing: 0.5px;
  font-weight: 600;
  margin-top: 20px;
}

.open-chat-button:hover {
  transform: translateY(-3px) scale(1.02);
  box-shadow: 0 12px 30px rgba(0, 123, 255, 0.4);
}

.open-chat-button:active {
  transform: translateY(0) scale(0.98);
  box-shadow: 0 4px 15px rgba(0, 123, 255, 0.2);
}

/* チャットウィンドウ全体 */
#app {
  position: fixed;
  border: none; /* ボーダーをなくす */
  border-radius: 15px; /* より丸く */
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.25); /* 強めのシャドウで浮遊感 */
  background-color: #ffffff; /* クリーンな白 */
  box-sizing: border-box;
  width: 420px; /* 少し広めに */
  height: 550px; /* 少し高めに */
  display: flex;
  flex-direction: column;
  resize: both;
  overflow: hidden;
  z-index: 1000;
  transition: box-shadow 0.3s ease;
}

#app:hover {
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.35);
}

/* チャットヘッダー */
.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px; /* パディングを増やす */
  background: linear-gradient(90deg, #4a00e0 0%, #8e2de2 100%); /* 深みのあるグラデーション */
  color: white;
  border-top-left-radius: 15px;
  border-top-right-radius: 15px;
  cursor: grab;
  font-weight: 700; /* 太字 */
  font-size: 1.1em;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.chat-header span {
  flex-grow: 1;
  text-align: center;
  letter-spacing: 0.8px;
}

.close-button {
  background: none;
  border: none;
  color: white;
  font-size: 1.5em; /* 大きく */
  cursor: pointer;
  padding: 0 8px;
  transition: transform 0.2s ease;
}

.close-button:hover {
  transform: rotate(90deg);
}

/* チャットメッセージエリア */
.chat-window {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  overflow: hidden;
}

.messages {
  flex-grow: 1;
  overflow-y: auto;
  padding: 15px;
  background-color: #f8f9fa; /* わずかにグレーの背景 */
  border-bottom: 1px solid #e0e0e0;
  text-align: left;
}

/* メッセージバブル */
.message {
  margin-bottom: 12px;
  padding: 10px 15px;
  border-radius: 20px; /* より丸く */
  max-width: 75%; /* 少し狭く */
  word-wrap: break-word;
  line-height: 1.5;
  font-size: 0.95em;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08); /* 柔らかいシャドウ */
}

.message.user {
  background: linear-gradient(45deg, #007bff 0%, #0056b3 100%); /* ユーザーメッセージのグラデーション */
  color: white;
  align-self: flex-end;
  margin-left: auto;
  border-bottom-right-radius: 5px; /* 角を少しシャープに */
}

.message.ai {
  background-color: #e9ecef; /* AIメッセージの背景色 */
  color: #333;
  align-self: flex-start;
  margin-right: auto;
  border-bottom-left-radius: 5px; /* 角を少しシャープに */
}

.message.ai.error {
  background-color: #ffe0e0;
  color: #d9534f;
  border: 1px solid #d9534f;
}

.message.ai.loading {
  font-style: italic;
  color: #888;
  background-color: #f0f0f0;
}

/* 入力エリア */
.input-area {
  display: flex;
  padding: 15px;
  background-color: #ffffff;
  border-top: 1px solid #e0e0e0;
  flex-shrink: 0;
}

.input-area input {
  flex-grow: 1;
  padding: 12px 15px;
  border: 1px solid #ced4da;
  border-radius: 25px; /* より丸く */
  margin-right: 10px;
  font-size: 1em;
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

.input-area input:focus {
  border-color: #8e2de2; /* フォーカス時の色 */
  box-shadow: 0 0 0 0.2rem rgba(142, 45, 226, 0.25);
  outline: none;
}

.input-area button {
  padding: 12px 20px;
  background: linear-gradient(45deg, #8e2de2 0%, #4a00e0 100%); /* 送信ボタンのグラデーション */
  color: white;
  border: none;
  border-radius: 25px; /* より丸く */
  cursor: pointer;
  font-size: 1em;
  font-weight: 600;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
  transition: all 0.3s ease;
}

.input-area button:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2);
}

.input-area button:disabled {
  background: #cccccc; /* 無効時の色 */
  cursor: not-allowed;
  box-shadow: none;
  transform: none;
}
</style>
