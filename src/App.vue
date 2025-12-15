<script setup>
import { ref, onMounted, computed } from "vue";

/* --------------------
   Config
-------------------- */

function isFutureDate(dateString) {
  const inputDate = new Date(dateString);
  const today = new Date();

  inputDate.setHours(0,0,0,0);
  today.setHours(0,0,0,0);

  return inputDate > today;
}

function isDuplicateTitle(title, ignoreId = null) {
  const normalised = title.trim().toLowerCase();

  return games.value.some(g =>
    g.title.trim().toLowerCase() === normalised &&
    g.id !== ignoreId
  );
}

const todayIso = new Date().toISOString().split("T")[0];

const API_URL = "https://localhost:7234/api/games";

/* --------------------
   State
-------------------- */

const games = ref([]);
const loading = ref(false);
const error = ref(null);
const message = ref(null);

const showAddForm = ref(false);

const newGame = ref({
  title: "",
  releaseDate: "",
  rating: "",
  genre: "",
  platform: ""
});

const editingId = ref(null);
const editGame = ref({});

const sortBy = ref("title");
const filterGenre = ref("");
const searchTitle = ref("");

/* --------------------
   API Helpers
-------------------- */

async function loadGames() {
  loading.value = true;
  error.value = null;

  try {
    const res = await fetch(API_URL);
    games.value = await res.json();
  } catch {
    error.value = "Failed to load games from server.";
  } finally {
    loading.value = false;
  }
}

async function addGame() {
  error.value = null;
  message.value = null;

  if (isFutureDate(newGame.value.releaseDate)) {
    error.value = "Sorry — this catalogue is for games that have already been released.";
    return;
  }

  if (isDuplicateTitle(newGame.value.title)) {
    error.value = "A game with this title already exists in the database.";
    return;
  }

  try {
    await fetch(API_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        title: newGame.value.title,
        releaseDate: newGame.value.releaseDate,
        rating: Number(newGame.value.rating),
        genre: newGame.value.genre,
        platform: newGame.value.platform
      })
    });

    message.value = "Game added successfully ✔";

    newGame.value = {
      title: "",
      releaseDate: "",
      rating: "",
      genre: "",
      platform: ""
    };

    showAddForm.value = false;
    loadGames();
  } catch {
    error.value = "Failed to add game.";
  }
}

async function deleteGame(game) {
  if (!confirm(`Delete "${game.title}" from the database?`)) return;

  try {
    await fetch(`${API_URL}/${game.id}`, { method: "DELETE" });
    message.value = "Game deleted.";
    loadGames();
  } catch {
    error.value = "Failed to delete game.";
  }
}

function startEdit(game) {
  editingId.value = game.id;
  editGame.value = { ...game };
}

async function saveEdit() {
  error.value = null;

  if (isFutureDate(editGame.value.releaseDate)) {
    error.value = "Sorry — this catalogue is for games that have already been released.";
    return;
  }

  if (isDuplicateTitle(editGame.value.title, editingId.value)) {
    error.value = "A game with this title already exists in the database.";
    return;
  }

  try {
    await fetch(`${API_URL}/${editingId.value}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        title: editGame.value.title,
        releaseDate: editGame.value.releaseDate,
        rating: Number(editGame.value.rating),
        genre: editGame.value.genre,
        platform: editGame.value.platform
      })
    });

    message.value = "Game updated successfully ✔";
    editingId.value = null;
    loadGames();
  } catch {
    error.value = "Failed to update game.";
  }
}

function cancelEdit() {
  editingId.value = null;
}

/* --------------------
   Sorting & Filtering
-------------------- */

const genres = computed(() =>
  [...new Set(games.value.map(g => g.genre))].sort()
);

const visibleGames = computed(() => {
  let list = [...games.value];

  if (searchTitle.value) {
    list = list.filter(g =>
      g.title.toLowerCase().includes(searchTitle.value.toLowerCase())
    );
  }

  if (filterGenre.value) {
    list = list.filter(g => g.genre === filterGenre.value);
  }

  return list.sort((a, b) =>
    sortBy.value === "rating"
      ? b.rating - a.rating
      : a.title.localeCompare(b.title)
  );
});

/* --------------------
   Init
-------------------- */

onMounted(loadGames);
</script>

<template>
  <main class="page">
    <h1>The Definitive Game Database</h1>

    <!-- CONTROLS -->
    <section class="controls">
      <button class="primary" @click="showAddForm = !showAddForm">
        {{ showAddForm ? "Cancel" : "Add Game To Database" }}
      </button>

      <input v-model="searchTitle" placeholder="Search by title" />

      <select v-model="filterGenre">
        <option value="">All genres</option>
        <option v-for="g in genres" :key="g" :value="g">{{ g }}</option>
      </select>

      <select v-model="sortBy">
        <option value="title">Sort by Title</option>
        <option value="rating">Sort by Rating</option>
      </select>
    </section>

    <transition name="fade">
      <p v-if="message" class="success">{{ message }}</p>
    </transition>

    <transition name="fade">
      <p v-if="error" class="error">{{ error }}</p>
    </transition>

    <!-- ADD GAME -->
    <section v-if="showAddForm" class="card">
      <h2>Add Game</h2>

      <form @submit.prevent="addGame" class="form-grid">
        <input v-model="newGame.title" placeholder="Title" required />

        <transition name="fade">
          <p
            v-if="newGame.title && isDuplicateTitle(newGame.title)"
            class="error"
          >
            A game with this title already exists in the database.
          </p>
        </transition>

        <input v-model="newGame.releaseDate" type="date" :max="todayIso" required />
        <input v-model="newGame.rating" type="number" min="0" max="100" placeholder="Rating" required />
        <input v-model="newGame.genre" placeholder="Genre" required />
        <input v-model="newGame.platform" placeholder="Platform" required />

        <button class="primary">Save Game</button>
      </form>
    </section>

    <p v-if="!loading && visibleGames.length === 0" class="error">
      No games match your current filters.
    </p>

    <!-- LIST -->
    <section class="grid">
      <div class="card" v-for="game in visibleGames" :key="game.id">
        <template v-if="editingId === game.id">
          <input v-model="editGame.title" />
          <input v-model="editGame.releaseDate" type="date" :max="todayIso" />
          <input v-model="editGame.rating" type="number" min="0" max="100" />
          <input v-model="editGame.genre" />
          <input v-model="editGame.platform" />

          <div class="actions">
            <button class="primary" @click="saveEdit">Save</button>
            <button @click="cancelEdit">Cancel</button>
          </div>
        </template>

        <template v-else>
          <h3>{{ game.title }}</h3>
          <p>
            {{ new Date(game.releaseDate).getFullYear() }}<br />
            {{ game.genre }} • {{ game.platform }}<br />
            ⭐ {{ game.rating }}
          </p>

          <div class="actions">
            <button @click="startEdit(game)">Edit</button>
            <button class="danger" @click="deleteGame(game)">Delete</button>
          </div>
        </template>
      </div>
    </section>
  </main>
</template>

<style>
:root {
  --bg: #0f172a;
  --card: #1e293b;
  --text: #e5e7eb;
  --accent: #38bdf8;
  --danger: #ef4444;
  --success: #22c55e;
}

body {
  background: var(--bg);
  color: var(--text);
  font-family: system-ui, sans-serif;
}

.page {
  max-width: 1200px;
  margin: 3rem auto;
  padding: 2.5rem;
  background: linear-gradient(
    180deg,
    rgba(30,41,59,0.35),
    rgba(15,23,42,0.35)
  );
  border-radius: 18px;
}

h1 {
  text-align: center;
  margin-bottom: 2.5rem;
  font-size: 2rem;
}

h1::after {
  content: "";
  display: block;
  width: 120px;
  height: 3px;
  margin: 0.75rem auto 0;
  background: linear-gradient(90deg, transparent, var(--accent), transparent);
  border-radius: 999px;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  justify-content: center;
  align-items: center;
  margin-bottom: 2rem;
  padding: 1rem;
  background: rgba(30,41,59,0.6);
  border-radius: 12px;
}

input,
select {
  background: #0f172a;
  color: var(--text);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 8px;
  padding: 0.55rem 0.65rem;
  transition: border 0.2s ease, box-shadow 0.2s ease;
}

input:focus,
select:focus {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 2px rgba(56,189,248,0.25);
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 1.25rem;
}

.card {
  background: linear-gradient(180deg, #1e293b, #162033);
  padding: 1.25rem;
  border-radius: 14px;
  box-shadow:
    0 10px 25px rgba(0,0,0,0.35),
    inset 0 1px 0 rgba(255,255,255,0.03);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.card:hover {
  transform: translateY(-2px);
}

button {
  padding: 0.5rem 0.9rem;
  border-radius: 8px;
  border: none;
  cursor: pointer;
  font-weight: 500;
  transition: transform 0.1s ease;
}

button:hover {
  transform: translateY(-1px);
}

button.primary {
  background: linear-gradient(180deg, #38bdf8, #0ea5e9);
  color: #000;
}

button.danger {
  background: linear-gradient(180deg, #ef4444, #dc2626);
  color: white;
}

.actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

.success {
  text-align: center;
  color: var(--success);
  margin-bottom: 0.75rem;
}

.error {
  text-align: center;
  color: var(--danger);
  margin: 0.25rem 0;
  font-size: 0.9rem;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
