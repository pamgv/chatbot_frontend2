<template>
  <div class="chat-container">
    <!-- Header -->
    <Motion :initial="{ opacity: 0, y: -20 }" :animate="{ opacity: 1, y: 0 }" :transition="{ duration: 0.6 }">
      <div class="chat-header">
        <div class="chat-title">
          <i class="bi bi-pencil-square"></i>
          <h3>NMSU-TutoringBot-MeatScience</h3>
        </div>
        <button class="menu-toggle-btn" @click="toggleSidebar">
          <i class="bi bi-list"></i>
        </button>
      </div>
    </Motion>

    <!-- Bot Info -->
    <Motion :initial="{ opacity: 0, x: -30 }" :animate="{ opacity: 1, x: 0 }" :transition="{ duration: 0.5, delay: 0.2 }">
      <div class="bot-info justify-between">
        <div>
          <h2>IntelligentTutor-MeatScience</h2>
          <p class="bot-status">User: {{ username }}</p>
          <p class="bot-status">
            Game: {{ gameNumber }} · Question: {{ userMessageCount }}/20 · Score: {{ gameStore.score }}
          </p>
        </div>
      </div>
    </Motion>

    <!-- Messages -->
    <div ref="chatMessages" class="chat-messages" v-if="messages.length > 0 || isLoading">
      <Motion
        v-for="(message, index) in messages"
        :key="index"
        :initial="{ opacity: 0, y: 20, scale: 0.95 }"
        :animate="{ opacity: 1, y: 0, scale: 1 }"
        :transition="{ duration: 0.5, type: 'spring', stiffness: 200, damping: 20 }"
      >
        <div class="message-container" :class="message.sender">
          <div class="message-bubble" :class="message.sender">
            <div v-if="message.sender === 'user'" class="user-avatar"><i class="bi bi-person"></i></div>
            <div v-else class="bot-avatar">
              <div class="bot-logo"><span class="bot-logo-text">NMSU</span></div>
            </div>
            <div class="message-content">
              <div class="message-header">
                <span class="sender-name">{{ message.sender === "user" ? "You" : "NMSU" }}</span>
              </div>
              <div class="message-text" v-html="message.text"></div>
            </div>
          </div>
        </div>
      </Motion>

      <!-- QUIZ card -->
      <Motion
        v-if="showQuiz"
        :initial="{ opacity: 0, y: 20 }"
        :animate="{ opacity: 1, y: 0 }"
        :transition="{ duration: 0.5 }"
      >
        <div class="quiz-box">
          <h3>{{ quizQuestion }}</h3>
          <div v-if="!quizAnswered">
            <div v-for="option in quizOptions" :key="option">
              <button @click="selectAnswer(option)">{{ option }}</button>
            </div>
          </div>
          <div v-else class="quiz-result" :class="{ correct: isAnswerCorrect, incorrect: !isAnswerCorrect }">
            <p v-if="isAnswerCorrect">
              ✅ Correct! Great job!
            </p>

            <p v-else>
              ❌ Incorrect.<br>
              Correct answer was:
              <strong>{{ correctAnswerLetter }} — {{ correctAnswer }}</strong>
            </p>

          </div>
        </div>
      </Motion>

      <!-- Typing indicator -->
      <Motion
        v-if="isLoading"
        :initial="{ opacity: 0, y: 20, scale: 0.9 }"
        :animate="{ opacity: 1, y: 0, scale: 1 }"
        :transition="{ duration: 0.5 }"
      >
        <div class="message-container bot">
          <div class="message-bubble bot loading">
            <div class="bot-avatar">
              <div class="bot-logo"><span class="bot-logo-text">NMSU</span></div>
            </div>
            <div class="message-content">
              <div class="message-header">
                <span class="sender-name">NMSU</span>
                <span class="bot-status">Typing...</span>
              </div>
              <div class="message-text">
                <Motion :animate="{ opacity: [0.3, 1, 0.3] }" :transition="{ duration: 1.5, repeat: Infinity }">
                  <div class="typing-indicator">
                    <span class="typing-dot"></span><span class="typing-dot"></span><span class="typing-dot"></span>
                  </div>
                </Motion>
              </div>
            </div>
          </div>
        </div>
      </Motion>
    </div>

    <!-- Welcome -->
    <Motion v-else :initial="{ opacity: 0, y: 30 }" :animate="{ opacity: 1, y: 0 }" :transition="{ duration: 0.8 }">
      <div class="chat-welcome">
        <h3 class="welcome-message">How can I help you today?</h3>
      </div>
    </Motion>

    <!-- Input -->
    <Motion :initial="{ opacity: 0, y: 20 }" :animate="{ opacity: 1, y: 0 }" :transition="{ duration: 0.6, delay: 0.4 }">
      <div class="chat-input-container">
        <div class="input-wrapper">
          <input
            type="text"
            placeholder="Send a message"
            class="message-input"
            v-model="messageText"
            @keyup.enter="sendMessage"
          />
          <button class="send-button" @click="sendMessage">
            <i class="bi bi-send mt-1"></i>
          </button>
        </div>
        <p class="disclaimer">LLMs can make mistakes. Verify important information.</p>
      </div>
    </Motion>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, watch } from "vue";
import { Motion } from "motion-v";
import { useRouter } from "vue-router";
import { useAuthStore } from "../store/auth";
import { useGameStore } from "../store/gameStore";
import axios from "axios";
import { storeToRefs } from "pinia";

const API_BASE = import.meta.env.VITE_API_URL || "http://localhost:8000";

const router = useRouter();
const authStore = useAuthStore();
const gameStore = useGameStore();

const { userMessageCount, score, gameNumber, highestScore } = storeToRefs(gameStore);
const username = ref("");
const messages = ref([]);
const messageText = ref("");
const isLoading = ref(false);
const chatMessages = ref(null);
const hasPersistedCurrentGame = ref(false);

// Quiz
const showQuiz = ref(false);
const quizQuestion = ref("");
const quizOptions = ref([]);
const correctAnswer = ref("");
const quizAnswered = ref(false);
const isAnswerCorrect = ref(false);
const correctAnswerLetter = ref("");


const toggleSidebar = () => {
  const sidebar = document.querySelector(".sidebar");
  const mainContent = document.querySelector(".main-content");
  if (sidebar && mainContent) {
    sidebar.classList.toggle("isOpen");
    mainContent.classList.toggle("hide");
  }
};

const scrollToBottom = () => {
  nextTick(() => {
    if (chatMessages.value) {
      chatMessages.value.scrollTo({ top: chatMessages.value.scrollHeight, behavior: "smooth" });
    }
  });
};
watch([messages, isLoading, showQuiz], () => setTimeout(scrollToBottom, 300), { deep: true });

const loadUserData = async () => {
  if (!username.value) return;
  try {
    const { data } = await axios.get(`${API_BASE}/user/get_stats/${username.value}`);
    let newGameNumber = 1;
    if (data.games && data.games.length > 0) {
      const lastGame = data.games[data.games.length - 1];
      newGameNumber = lastGame.game_number + 1;
    }
    gameStore.gameNumber = newGameNumber;
    gameStore.userMessageCount = 0;
    gameStore.score = 0;
    gameStore.highestScore = data.best_score || 0;
    hasPersistedCurrentGame.value = false;

    messages.value = [
      { id: Date.now(), text: `🎮 Welcome back, ${username.value}! Starting new Game #${gameStore.gameNumber}.`, sender: "bot" },
    ];
  } catch (error) {
    console.error("Error loading user data:", error);
  }
};

const sendMessage = async () => {
  if (!messageText.value.trim() || showQuiz.value) return;

  const text = messageText.value.trim();
  messageText.value = "";
  messages.value.push({ id: Date.now(), text, sender: "user" });
  scrollToBottom();
  isLoading.value = true;

  const nextCount = gameStore.userMessageCount + 1;
  if (nextCount > 20) {
    gameStore.gameNumber++;
    gameStore.userMessageCount = 0;
    hasPersistedCurrentGame.value = false;

    await axios.post(`${API_BASE}/user/update_game`, {
      username: username.value,
      game_number: gameStore.gameNumber,
      question_number: 1,
      correct_count: 0,
      highest_score: gameStore.highestScore,
    });

    await loadUserData();
    window.dispatchEvent(new CustomEvent("games-updated", { detail: { username: username.value } }));
    messages.value = [
      { id: Date.now(), text: `🎮 Game #${gameStore.gameNumber} started!`, sender: "bot" },
    ];
    isLoading.value = false;
    return;
  }

  const isFirstOfThisGame = gameStore.userMessageCount === 0;
  gameStore.userMessageCount = nextCount;

  try {
    if (isFirstOfThisGame && !hasPersistedCurrentGame.value) {
      await axios.post(`${API_BASE}/user/update_game`, {
        username: username.value,
        game_number: gameStore.gameNumber,
        question_number: 1,
        correct_count: gameStore.score,
        highest_score: gameStore.highestScore,
      });
      hasPersistedCurrentGame.value = true;
    }

    const { data } = await axios.post(`${API_BASE}/user/save_message`, {
      username: username.value,
      text,
      game_number: gameStore.gameNumber,
      question_number: gameStore.userMessageCount,
    });

    messages.value.push({ id: Date.now() + 1, text: data.bot_response, sender: "bot" });

    // ✅ Generar quiz cada 2 preguntas (no cada 5)
    if (nextCount % 2 === 0) {
      await nextTick();

      // ✅ Contexto: toda la conversación actual del juego
      const context = messages.value.map(m => `${m.sender}: ${m.text}`).join("\n");

      try {
        const quizData = await axios.post(`${API_BASE}/chatbot/generate_quiz`, {
          username: username.value,
          context,
        });

        const q = quizData?.data;
        if (!q || !q.question || !q.options?.length) {
          console.warn("⚠️ API devolvió respuesta vacía, usando fallback.");
          quizQuestion.value = "What is the main component of meat?";
          quizOptions.value = ["Protein", "Carbohydrates", "Lipids", "Vitamins"];
          correctAnswer.value = "Protein";
        } else {
          quizQuestion.value = q.question;
          quizOptions.value = q.options;
          correctAnswer.value = q.correct_answer_text;   // texto
          correctAnswerLetter.value = q.correct_answer_letter; // letra

        }

        quizAnswered.value = false;
        showQuiz.value = true;
        console.log("✅ Quiz card activado:", quizQuestion.value);
      } catch (err) {
        console.error("Error generating quiz:", err);
        quizQuestion.value = "What is the main component of meat?";
        quizOptions.value = ["Protein", "Carbohydrates", "Lipids", "Vitamins"];
        correctAnswer.value = "Protein";
        quizAnswered.value = false;
        showQuiz.value = true;
      }
    }
  } catch (error) {
    console.error(error);
    messages.value.push({ id: Date.now() + 2, text: "Error contacting API or DB.", sender: "bot" });
  } finally {
    isLoading.value = false;
    scrollToBottom();
  }
};

const selectAnswer = async (option) => {
  quizAnswered.value = true;

  // 🧠 Normaliza comparación (admite letra o texto)
  const normalizedCorrect = correctAnswer.value.trim().toLowerCase();
  const normalizedOption = option.trim().toLowerCase();

  const letterMap = ["a", "b", "c", "d", "e"];
  let correctText = normalizedCorrect;

  // Si el correctAnswer es letra, obtenemos texto real
  if (letterMap.includes(normalizedCorrect)) {
    const index = letterMap.indexOf(normalizedCorrect);
    correctText = quizOptions.value[index]?.trim().toLowerCase() || "";
  }

  isAnswerCorrect.value = normalizedOption === correctText;

  // ✅ Mostrar texto completo de la respuesta correcta (si era letra)
  const correctDisplay =
    letterMap.includes(normalizedCorrect)
      ? quizOptions.value[letterMap.indexOf(normalizedCorrect)]
      : correctAnswer.value;

  if (isAnswerCorrect.value) gameStore.score++;

  try {
    // ✅ Guardar resultado del quiz con detalle completo
    await axios.post(`${API_BASE}/user/save_quiz_result`, {
      username: username.value,
      game_number: gameStore.gameNumber,
      question_number: gameStore.userMessageCount,
      quiz_question: quizQuestion.value,
      quiz_options: quizOptions.value,
      selected_option: option,
      correct_answer: correctDisplay,
      is_correct: isAnswerCorrect.value,
    });

    // ✅ Actualizar progreso del juego
    await axios.post(`${API_BASE}/user/update_game`, {
      username: username.value,
      game_number: gameStore.gameNumber,
      question_number: gameStore.userMessageCount,
      correct_count: gameStore.score,
      highest_score: gameStore.highestScore,
    });

    console.log("✅ Quiz result saved and game updated");
  } catch (err) {
    console.error("Error saving quiz result:", err);
  }

  setTimeout(() => {
    showQuiz.value = false;
  }, 3000);
  scrollToBottom();
};

onMounted(async () => {
  authStore.initialize();
  username.value = authStore.user?.username || localStorage.getItem("username") || "guest";
  await loadUserData();
});

onMounted(() => {
  window.addEventListener("load-conversation", (event) => {
    const { gameNumber, messages: rawMessages, stats } = event.detail;
    gameStore.gameNumber = gameNumber;
    gameStore.userMessageCount = stats?.question_number || 0;
    gameStore.score = stats?.correct_count || 0;
    messages.value = rawMessages.flatMap((m) => [
      { sender: "user", text: m.user_message },
      { sender: "bot", text: m.bot_response },
    ]);
    console.log(
      `📚 Loaded Game #${gameNumber}: ${gameStore.userMessageCount} questions, ${gameStore.score} correct`
    );
  });
});
</script>

<style scoped src="../styles/chat.css"></style>
<style scoped src="../styles/chatMessages.css"></style>
<style scoped src="../styles/animation.css"></style>

<style scoped>
.quiz-box {
  background-color: #f8f8f8;
  color: #111;
  padding: 15px;
  border-radius: 10px;
  margin: 10px 70px 0 70px;
  border: 1px solid #ddd;
  text-align: center;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
}
.quiz-box h3 {
  margin-bottom: 10px;
  color: #000;
  font-weight: 600;
}
.quiz-box button {
  margin: 5px;
  background-color: #8b1538;
  color: #fff;
  border: none;
  padding: 8px 12px;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.quiz-box button:hover { background-color: #a91b47; }
.quiz-result.correct p { color: green; font-weight: 600; }
.quiz-result.incorrect p { color: #b91c1c; font-weight: 600; }
</style>



