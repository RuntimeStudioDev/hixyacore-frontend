<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue';

// --- Types ---
type PlayerList = string[];
type PlayerCount = number;

// --- State ---
const players = ref<PlayerList>([]);
const onlineCount = ref<PlayerCount>(0);
const isLoading = ref<boolean>(true);
const error = ref<string | null>(null);
const timerId = ref<number | null>(null);
const isCopied = ref(false);

// --- Config ---
// Using a placeholder IP since one wasn't provided, but the button label says "Copy IP"
const SERVER_IP = "hixyamc.com"; 
const API_BASE = "https://hixyacore-backend.onrender.com/api/server";

// --- Logic ---
const fetchData = async () => {
  try {
    // Only show loading on initial load to avoid UI flicker during polling
    if (players.value.length === 0 && !error.value) {
      isLoading.value = true;
    }
    
    // Parallel fetch for efficiency
    const [playersRes, countRes] = await Promise.all([
      fetch(`${API_BASE}/players`),
      fetch(`${API_BASE}/players/count`)
    ]);

    if (!playersRes.ok || !countRes.ok) {
      throw new Error('API Error');
    }

    const playersData = await playersRes.json();
    const countData = await countRes.json();

    // Updating state
    players.value = Array.isArray(playersData) ? playersData : [];
    
    // Handle count: API might return number or object or string, safe parsing
    if (typeof countData === 'number') {
      onlineCount.value = countData;
    } else if (typeof countData === 'object' && countData !== null && 'count' in countData) {
       // Just in case the API structure wraps it
       onlineCount.value = Number(countData.count);
    } else {
       onlineCount.value = Number(countData);
    }

    error.value = null;
  } catch (err) {
    console.error("Failed to fetch server data:", err);
    error.value = "SERVER UNREACHABLE";
  } finally {
    isLoading.value = false;
  }
};

const copyIp = async () => {
  try {
    await navigator.clipboard.writeText(SERVER_IP);
    isCopied.value = true;
    setTimeout(() => {
      isCopied.value = false;
    }, 2000);
  } catch (err) {
    console.error("Failed to copy IP", err);
  }
};

// --- Lifecycle ---
onMounted(() => {
  fetchData();
  // Poll every 10 seconds
  timerId.value = window.setInterval(fetchData, 10000);
});

onUnmounted(() => {
  if (timerId.value) {
    clearInterval(timerId.value);
  }
});

// Computed for display
const displayCount = computed(() => {
  // If count is 0 but we have players in the list, use list length (failsafe)
  if (onlineCount.value === 0 && players.value.length > 0) {
    return players.value.length;
  }
  return onlineCount.value;
});

</script>

<template>
  <div class="app-container">
    
    <!-- Hero Section -->
    <header class="hero">
      <h1 class="glitch-text" data-text="HIXYA NETWORK">HIXYA NETWORK</h1>
      <p class="subtitle">IMMERSE YOURSELF IN THE VOID</p>
      
      <div class="stats-card glass-panel">
        <div class="stat-item">
          <span class="label">STATUS</span>
          <span class="value online" v-if="!error">ONLINE</span>
          <span class="value offline" v-else>OFFLINE</span>
        </div>
        <div class="stat-item">
          <span class="label">PLAYERS</span>
          <span class="value count">{{ displayCount }}</span>
        </div>
      </div>

      <button @click="copyIp" class="cta-btn" :class="{ copied: isCopied }">
        <span v-if="!isCopied">IP: {{ SERVER_IP }}</span>
        <span v-else>COPIED!</span>
      </button>
    </header>

    <!-- Main Content -->
    <main class="content">
      
      <section class="players-section">
        <h2 class="section-title">ACTIVE PLAYERS</h2>
        
        <!-- Loading State -->
        <div v-if="isLoading" class="loading-state">
           <div class="spinner"></div>
           <p>ESTABLISHING CONNECTION...</p>
        </div>

        <!-- Error State -->
        <div v-else-if="error" class="error-state">
          <p class="error-msg">⚠ {{ error }}</p>
          <button @click="fetchData" class="retry-btn">RETRY SIGNAL</button>
        </div>

        <!-- Empty State -->
        <div v-else-if="players.length === 0" class="empty-state">
          <p>NO PLAYERS DETECTED IN THE VOID.</p>
        </div>

        <!-- Player Grid -->
        <div v-else class="player-grid">
          <div 
            v-for="player in players" 
            :key="player" 
            class="player-card"
            :title="player"
          >
            <div class="avatar-wrapper">
              <img 
                :src="`https://minotar.net/cube/${player}/100.png`" 
                :alt="player" 
                loading="lazy"
              />
            </div>
            <span class="player-name">{{ player }}</span>
          </div>
        </div>
      </section>

    </main>

    <footer class="footer">
      <p>&copy; {{ new Date().getFullYear() }} HIXYA NETWORK. ALL RIGHTS RESERVED.</p>
    </footer>

  </div>
</template>

<style scoped>
/* Layout */
.app-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 100vh;
  position: relative;
  z-index: 10; /* Above scanlines */
}

/* Hero Section */
.hero {
  width: 100%;
  height: 60vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background: linear-gradient(180deg, rgba(18,18,18,0.8) 0%, rgba(18,18,18,1) 100%),
              url(''); /* Placeholder dark bg gif/image could go here, strictly using gradients for now as no asset provided */
  text-align: center;
  padding: 2rem;
  border-bottom: 2px solid var(--color-obsidian-light);
}

.glitch-text {
  font-family: var(--font-pixel);
  font-size: 4rem;
  font-weight: 400;
  color: var(--color-text-main);
  text-shadow: 4px 4px 0px var(--color-accent-blue);
  letter-spacing: 4px;
  margin-bottom: 0.5rem;
  animation: float 6s ease-in-out infinite;
}

.subtitle {
  font-family: var(--font-sans);
  color: var(--color-text-muted);
  font-size: 1.2rem;
  letter-spacing: 2px;
  margin-bottom: 2rem;
  text-transform: uppercase;
}

/* Stats Card */
.glass-panel {
  background: rgba(30, 30, 30, 0.4);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  padding: 1.5rem 3rem;
  border-radius: var(--border-radius-md);
  display: flex;
  gap: 3rem;
  margin-bottom: 2rem;
  box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
}

.stat-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.label {
  font-family: var(--font-sans);
  font-size: 0.8rem;
  color: var(--color-text-muted);
  margin-bottom: 0.25rem;
}

.value {
  font-family: var(--font-pixel);
  font-size: 1.8rem;
}

.value.online { color: var(--color-accent-green); text-shadow: 0 0 10px var(--color-accent-green); }
.value.offline { color: var(--color-accent-red); text-shadow: 0 0 10px var(--color-accent-red); }
.value.count { color: var(--color-accent-gold); text-shadow: 0 0 10px var(--color-accent-gold); }

/* CTA Button */
.cta-btn {
  background-color: transparent;
  color: var(--color-accent-blue);
  border: 2px solid var(--color-accent-blue);
  font-family: var(--font-pixel);
  font-size: 1.5rem;
  padding: 0.8rem 2rem;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 4px 0 var(--color-accent-blue);
  position: relative;
  top: 0;
}

.cta-btn:active {
  top: 4px;
  box-shadow: 0 0 0 var(--color-accent-blue);
}

.cta-btn:hover {
  background-color: rgba(85, 255, 255, 0.1);
  text-shadow: 0 0 8px var(--color-accent-blue);
}

.cta-btn.copied {
  color: var(--color-accent-green);
  border-color: var(--color-accent-green);
  box-shadow: 0 4px 0 var(--color-accent-green);
}

/* Content */
.content {
  width: 100%;
  max-width: 1200px;
  padding: 3rem 1.5rem;
  flex: 1;
}

.section-title {
  font-family: var(--font-pixel);
  color: var(--color-text-main);
  font-size: 2rem;
  margin-bottom: 2rem;
  border-bottom: 1px solid var(--color-obsidian-light);
  padding-bottom: 0.5rem;
}

/* Player Grid */
.player-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 1.5rem;
}

.player-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: var(--color-obsidian);
  border: 1px solid var(--color-obsidian-light);
  padding: 1rem;
  border-radius: var(--border-radius-sm);
  transition: transform 0.2s, box-shadow 0.2s;
}

.player-card:hover {
  transform: translateY(-5px);
  border-color: var(--color-accent-gold);
  box-shadow: 0 5px 15px rgba(255, 170, 0, 0.2);
}

.avatar-wrapper {
  width: auto;
  height: auto;
  margin-bottom: 0.5rem;
  overflow: hidden;
  border-radius: 4px;
  border: 2px solid #000;
}

.avatar-wrapper img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  image-rendering: pixelated; /* Essential for minecraft heads */
}

.player-name {
  font-family: var(--font-sans);
  font-size: 0.9rem;
  font-weight: 600;
  color: var(--color-text-main);
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  width: 100%;
}

/* States */
.loading-state, .error-state, .empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 3rem;
  font-family: var(--font-pixel);
  font-size: 1.2rem;
  color: var(--color-text-muted);
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid var(--color-obsidian-light);
  border-top-color: var(--color-accent-blue);
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error-msg {
  color: var(--color-accent-red);
  margin-bottom: 1rem;
}

.retry-btn {
  background: var(--color-obsidian);
  color: var(--color-text-main);
  border: 1px solid var(--color-text-muted);
  padding: 0.5rem 1rem;
  font-family: var(--font-pixel);
  cursor: pointer;
}
.retry-btn:hover {
  background: var(--color-obsidian-light);
}

/* Footer */
.footer {
  width: 100%;
  padding: 2rem;
  text-align: center;
  border-top: 1px solid var(--color-obsidian-light);
  margin-top: auto;
  font-family: var(--font-sans);
  font-size: 0.8rem;
  color: var(--color-text-muted);
}

/* Config Responsive */
@media (max-width: 768px) {
  .hero {
    height: auto;
    padding: 4rem 1rem;
  }
  
  .glitch-text {
    font-size: 2.5rem;
  }
  
  .glass-panel {
    flex-direction: column;
    padding: 1.5rem;
    gap: 1.5rem;
    width: 90%;
  }
  
  .player-grid {
    grid-template-columns: repeat(auto-fill, minmax(80px, 1fr));
  }
}
</style>
