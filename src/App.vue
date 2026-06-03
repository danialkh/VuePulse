<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// --- State Management ---
const username = ref('vuejs') // Default profile to showcase
const userData = ref<any>(null)
const repos = ref<any[]>([])
const loading = ref(false)
const error = ref('')
const searchHistory = ref<string[]>([])

// --- Computed Properties ---
const totalStars = computed(() => {
  return repos.value.reduce((acc, repo) => acc + repo.stargazers_count, 0)
})

const topLanguages = computed(() => {
  const langs: Record<string, number> = {}
  repos.value.forEach((repo) => {
    if (repo.language) {
      langs[repo.language] = (langs[repo.language] || 0) + 1
    }
  })
  return Object.entries(langs)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 3)
})

// --- Methods / Actions ---
const fetchGitHubData = async () => {
  if (!username.value.trim()) return

  loading.value = true
  error.value = ''
  userData.value = null
  repos.value = []

  try {
    // Fetch User Profile
    const userResponse = await fetch(`https://api.github.com/users/${username.value}`)
    if (!userResponse.ok) throw new Error('User not found!')
    userData.value = await userResponse.json()

    // Fetch User Repos (Sorted by stars)
    const reposResponse = await fetch(
      `https://api.github.com/users/${username.value}/repos?sort=stars&per_page=6`,
    )
    if (reposResponse.ok) {
      repos.value = await reposResponse.json()
    }

    // Update Search History (Keep last 5 unique searches)
    if (!searchHistory.value.includes(username.value)) {
      searchHistory.value = [username.value, ...searchHistory.value].slice(0, 5)
    }
  } catch (err: any) {
    error.value = err.message || 'Something went wrong'
  } finally {
    loading.value = false
  }
}

const selectHistory = (pastUsername: string) => {
  username.value = pastUsername
  fetchGitHubData()
}

// Fetch initial data on load
onMounted(() => {
  fetchGitHubData()
})
</script>

<template>
  <div class="app-container">
    <header class="header">
      <div class="logo"><span class="vue-accent">Vue</span>Pulse <span>⚡</span></div>
      <p class="subtitle">Reactive GitHub Portfolio Explorer</p>
    </header>

    <section class="search-box">
      <div class="input-group">
        <input
          v-model.lazy="username"
          @keyup.enter="fetchGitHubData"
          type="text"
          placeholder="Enter GitHub username (e.g., EvanYou)..."
          :disabled="loading"
        />
        <button @click="fetchGitHubData" :disabled="loading">
          {{ loading ? 'Scanning...' : 'Explore' }}
        </button>
      </div>

      <div v-if="searchHistory.length > 1" class="history-tags">
        <span>Recent:</span>
        <button
          v-for="hist in searchHistory"
          :key="hist"
          @click="selectHistory(hist)"
          class="chip"
          :class="{ active: hist === userData?.login }"
        >
          {{ hist }}
        </button>
      </div>
    </section>

    <div v-if="error" class="error-banner">
      ⚠️ {{ error }} - Try searching another GitHub handle.
    </div>

    <div v-if="loading" class="spinner-container">
      <div class="spinner"></div>
      <p>Querying GitHub Graph API...</p>
    </div>

    <main v-if="userData && !loading" class="dashboard animate-fade-in">
      <aside class="profile-card">
        <img :src="userData.avatar_url" :alt="userData.name" class="avatar" />
        <h2>{{ userData.name || userData.login }}</h2>
        <p class="bio">{{ userData.bio || "This developer hasn't written a bio yet." }}</p>

        <div class="stats-grid">
          <div class="stat-item">
            <span class="stat-num">{{ userData.public_repos }}</span>
            <span class="stat-label">Repos</span>
          </div>
          <div class="stat-item">
            <span class="stat-num">{{ userData.followers }}</span>
            <span class="stat-label">Followers</span>
          </div>
          <div class="stat-item">
            <span class="stat-num">{{ totalStars }}</span>
            <span class="stat-label">Total Stars</span>
          </div>
        </div>

        <div class="tags">
          <span v-for="[lang] in topLanguages" :key="lang" class="badge">
            {{ lang }}
          </span>
        </div>

        <a :href="userData.html_url" target="_blank" class="github-btn">
          View Official GitHub Profile
        </a>
      </aside>

      <section class="repos-section">
        <h3>🔥 Top Repositories</h3>
        <div class="repos-grid">
          <div v-for="repo in repos" :key="repo.id" class="repo-card">
            <h4>
              <a :href="repo.html_url" target="_blank">{{ repo.name }}</a>
            </h4>
            <p>{{ repo.description || 'No description provided for this repository.' }}</p>
            <div class="repo-meta">
              <span v-if="repo.language" class="repo-lang">
                <span class="dot"></span> {{ repo.language }}
              </span>
              <span class="repo-stars">⭐ {{ repo.stargazers_count }}</span>
            </div>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
/* --- Design System & Styling --- */
:global(body) {
  margin: 0;
  font-family:
    'Inter',
    system-ui,
    -apple-system,
    sans-serif;
  background-color: #0d1117;
  color: #c9d1d9;
}

.app-container {
  max-width: 1100px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

.header {
  text-align: center;
  margin-bottom: 2.5rem;
}

.logo {
  font-size: 2.5rem;
  font-weight: 800;
  color: #ffffff;
}

.vue-accent {
  color: #42b883;
}

.subtitle {
  color: #8b949e;
  margin-top: 0.5rem;
}

/* Search Box */
.search-box {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.input-group {
  display: flex;
  gap: 1rem;
}

input {
  flex: 1;
  background: #0d1117;
  border: 1px solid #30363d;
  border-radius: 6px;
  padding: 0.75rem 1rem;
  color: white;
  font-size: 1rem;
  transition: border-color 0.2s;
}

input:focus {
  outline: none;
  border-color: #42b883;
}

button {
  background: #238636;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.75rem 1.5rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

button:hover {
  background: #2ea043;
}

button:disabled {
  background: #213547;
  cursor: not-allowed;
}

.history-tags {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  margin-top: 1rem;
  font-size: 0.85rem;
  color: #8b949e;
}

.chip {
  background: #21262d;
  color: #c9d1d9;
  border: 1px solid #30363d;
  padding: 0.2rem 0.6rem;
  border-radius: 20px;
  font-size: 0.8rem;
}

.chip.active {
  border-color: #42b883;
  color: #42b883;
}

/* Dashboard Layout */
.dashboard {
  display: grid;
  grid-template-columns: 320px 1fr;
  gap: 2rem;
}

@media (max-width: 768px) {
  .dashboard {
    grid-template-columns: 1fr;
  }
}

/* Profile Sidebar */
.profile-card {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 12px;
  padding: 2rem;
  text-align: center;
  height: fit-content;
}

.avatar {
  width: 120px;
  height: 120px;
  border-radius: 50%;
  border: 3px solid #42b883;
  margin-bottom: 1rem;
}

.bio {
  color: #8b949e;
  font-size: 0.9rem;
  line-height: 1.4;
  margin-bottom: 1.5rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.5rem;
  background: #0d1117;
  padding: 0.75rem;
  border-radius: 8px;
  margin-bottom: 1.5rem;
}

.stat-num {
  display: block;
  font-weight: bold;
  color: white;
  font-size: 1.1rem;
}

.stat-label {
  font-size: 0.75rem;
  color: #8b949e;
}

.tags {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.badge {
  background: rgba(66, 184, 131, 0.1);
  color: #42b883;
  border: 1px solid rgba(66, 184, 131, 0.2);
  padding: 0.25rem 0.75rem;
  border-radius: 12px;
  font-size: 0.8rem;
  font-weight: 600;
}

.github-btn {
  display: block;
  background: #21262d;
  color: #c9d1d9;
  text-decoration: none;
  padding: 0.75rem;
  border-radius: 6px;
  border: 1px solid #30363d;
  font-weight: 600;
  transition: background 0.2s;
}

.github-btn:hover {
  background: #30363d;
}

/* Repositories */
.repos-section h3 {
  margin-top: 0;
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
}

.repos-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.25rem;
}

.repo-card {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  transition:
    transform 0.2s,
    border-color 0.2s;
}

.repo-card:hover {
  transform: translateY(-2px);
  border-color: #42b883;
}

.repo-card h4 a {
  color: #58a6ff;
  text-decoration: none;
  font-size: 1.1rem;
}

.repo-card h4 a:hover {
  text-decoration: underline;
}

.repo-card p {
  color: #8b949e;
  font-size: 0.875rem;
  line-height: 1.5;
  margin: 0.5rem 0 1rem 0;
}

.repo-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  color: #8b949e;
}

.dot {
  display: inline-block;
  width: 10px;
  height: 10px;
  background-color: #42b883;
  border-radius: 50%;
  margin-right: 4px;
}

/* Status States styling */
.error-banner {
  background: rgba(248, 81, 73, 0.1);
  color: #f85149;
  border: 1px solid rgba(248, 81, 73, 0.2);
  padding: 1rem;
  border-radius: 6px;
  margin-bottom: 1.5rem;
}

.spinner-container {
  text-align: center;
  padding: 3rem 0;
  color: #8b949e;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #30363d;
  border-top-color: #42b883;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem auto;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.animate-fade-in {
  animation: fadeIn 0.4s ease-out;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
