<template>
  <div class="sidebar">
    <button class="menu-section-hide" @click="closeRightPanel">
      <svg xmlns="http://www.w3.org/2000/svg" fill="none" stroke="currentColor" stroke-width="2"
        stroke-linecap="round" stroke-linejoin="round" class="feather feather-x" viewBox="0 0 24 24">
        <path d="M18 6L6 18M6 6l12 12" />
      </svg>
    </button>

    <!-- Header -->
    <div class="sidebar-header">
      <!-- Botón nuevo chat (deshabilitado por ahora)
      <button class="new-chat-btn justify-between" @click="startNewChat">
        <div class="flex items-center gap-2">
          <i class="bi bi-globe"></i>
          <span>New Chat</span>
        </div>
        <i class="bi bi-pencil-square"></i>
      </button>-->
    </div>

    <!-- Lista de juegos -->
    <div class="conversations-list">
      <div
        v-for="game in filteredGames"
        :key="game.game_number"
        class="conversation-item"
        :class="{ active: selectedGame === game.game_number }"
        @click="selectGame(game.game_number)"
      >
        <div class="conversation-icon">
          <i class="bi bi-chat"></i>
        </div>
        <div class="conversation-details">
          <span class="conversation-text">
            Game #{{ game.game_number }}
          </span>
          <span class="conversation-subtext">
            Questions: {{ game.question_number }}/20 · {{ formatDate(game.created_at) }}
          </span>
        </div>
        <button class="conversation-menu">
          <i class="bi bi-three-dots"></i>
        </button>
      </div>

      <div v-if="filteredGames.length === 0" class="no-games">
        <p>No games found</p>
      </div>
    </div>

    <!-- Footer -->
    <div class="sidebar-footer">
      <div class="flex gap-2 sm:flex-row flex-col"></div>
      <button class="footer-btn" @click="logout">
        <i class="bi bi-box-arrow-right"></i>
        <span>Logout</span>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";
import axios from "axios";
import { useRouter } from "vue-router";
import { useAuthStore } from "../store/auth";

// ✅ API Base dinámica
const API_BASE = import.meta.env.VITE_API_URL;

const router = useRouter();
const authStore = useAuthStore();

const username = ref(localStorage.getItem("username") || "guest");
const games = ref([]);
const selectedGame = ref(null);
const searchQuery = ref("");

// 🔹 Cargar juegos del usuario
const loadGames = async () => {
  try {
    const { data } = await axios.get(`${API_BASE}/user/get_stats/${username.value}`);
    // Orden descendente: últimos juegos primero
    games.value = (data.games || []).sort((a, b) => b.game_number - a.game_number);
  } catch (err) {
    console.error("Error loading games:", err);
  }
};

// 🔹 Filtro por búsqueda
const filteredGames = computed(() =>
  games.value.filter((g) =>
    g.game_number.toString().includes(searchQuery.value.trim())
  )
);

// 🔹 Formato de fecha
const formatDate = (iso) => {
  if (!iso) return "";
  return new Date(iso).toLocaleString();
};

// 🔹 Al seleccionar un juego, cargar mensajes
const selectGame = async (gameNumber) => {
  selectedGame.value = gameNumber;

  try {
    // 1️⃣ Traer mensajes del juego seleccionado
    const { data } = await axios.get(`${API_BASE}/user/get_game_messages/${username.value}/${gameNumber}`);

    // 2️⃣ Buscar el juego correspondiente (para sus stats)
    const gameData = games.value.find((g) => g.game_number === gameNumber);

    // 3️⃣ Emitir evento global con mensajes y stats
    const event = new CustomEvent("load-conversation", {
      detail: {
        gameNumber,
        messages: data.messages,
        stats: {
          question_number: gameData?.question_number || 0,
          correct_count: gameData?.correct_count || 0,
        },
      },
    });
    window.dispatchEvent(event);
  } catch (err) {
    console.error("Error loading messages:", err);
  }
};

// 🔹 Logout
const logout = () => {
  authStore.logout();
  router.push("/login");
};

// 🔹 Cerrar panel lateral
const closeRightPanel = () => {
  document.querySelector(".sidebar")?.classList.remove("isOpen");
  document.querySelector(".main-content")?.classList.remove("hide");
};

// 🔹 Escuchar cuando se cree un nuevo juego desde el chat
const handleGamesUpdated = async (event) => {
  if (event.detail.username === username.value) {
    console.log("♻️ Refrescando lista de juegos...");
    await loadGames();

    // 👇 Opcional: selecciona automáticamente el nuevo juego
    if (games.value.length > 0) {
      selectedGame.value = games.value[0].game_number;
    }
  }
};

// 🔹 Montar y desmontar listeners
onMounted(() => {
  loadGames();
  window.addEventListener("games-updated", handleGamesUpdated);
});

onUnmounted(() => {
  window.removeEventListener("games-updated", handleGamesUpdated);
});
</script>

