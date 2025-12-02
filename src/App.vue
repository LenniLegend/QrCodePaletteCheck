<script setup>
import { ref } from 'vue';
import InputText from 'primevue/inputtext';
import Button from 'primevue/button';
import Card from 'primevue/card';
import Message from 'primevue/message';
import ProgressSpinner from 'primevue/progressspinner';
import FloatLabel from 'primevue/floatlabel';

// Reaktive Variablen
const barcode = ref('');
const result = ref(null);
const error = ref(null);
const loading = ref(false);
const searched = ref(false); // Um "Kein Eintrag gefunden" nur nach Suche anzuzeigen

// API-Konfiguration
// Wir nutzen nun den Proxy (in vite.config.js oder nginx.conf), um CORS-Fehler zu vermeiden
const API_BASE_URL = '/api/barcode';

const version = __APP_VERSION__;

// Suchfunktion
const searchBarcode = async () => {
  if (!barcode.value.trim()) return;

  loading.value = true;
  error.value = null;
  result.value = null;
  searched.value = true;

  try {
    // GET-Anfrage an die API
    const response = await fetch(`${API_BASE_URL}?barcode=${encodeURIComponent(barcode.value)}`);
    
    if (!response.ok) {
      if (response.status === 404) {
        // Kein Eintrag gefunden (falls API 404 zurückgibt)
        result.value = null;
      } else {
        throw new Error(`API Fehler: ${response.statusText} (${response.status})`);
      }
    } else {
      const json = await response.json();
      // Erwartetes Format laut API:
      // { data: { artikel, auftrag, barcode, id, menge, timestamp }, found: true }
      // Wir mappen auf ein flaches Objekt für die Anzeige und trimmen den Barcode (\r am Ende).
      if (json && json.found && json.data) {
        const d = json.data;
        result.value = {
          barcode: (d.barcode || '').trim(),
          auftrag: d.auftrag ?? null,
          artikel: d.artikel ?? null,
          menge: d.menge ?? null,
          timestamp: d.timestamp ?? null,
          id: d.id ?? null,
        };
      } else {
        result.value = null;
      }
    }
  } catch (err) {
    console.error(err);
    error.value = 'Fehler bei der Verbindung zur API. Bitte prüfen Sie die Verbindung.';
  } finally {
    loading.value = false;
  }
};
</script>

<template>
  <div class="app-container">
    <div class="background-shapes">
      <div class="shape shape-1"></div>
      <div class="shape shape-2"></div>
      <div class="shape shape-3"></div>
    </div>
    
    <div class="header-section">
      <div class="logo-container">
        <i class="pi pi-qrcode logo-icon"></i>
      </div>
      <h1 class="main-title">Barcode Check</h1>
      <p class="main-subtitle">Schnell und einfach Barcode-Informationen abrufen</p>
    </div>

    <Card class="search-card glass-effect">
      <template #content>
        <div class="search-form">
          <div class="input-group">
            <FloatLabel>
                <InputText 
                  id="barcode" 
                  v-model="barcode" 
                  @keydown.enter="searchBarcode" 
                  class="w-full modern-input" 
                  :disabled="loading"
                />
                <label for="barcode">Barcode eingeben</label>
            </FloatLabel>
          </div>
          <Button 
            label="Suchen" 
            icon="pi pi-search" 
            @click="searchBarcode" 
            :loading="loading"
            class="search-button"
            severity="primary"
          />
        </div>

        <!-- Ladeanzeige -->
        <div v-if="loading" class="loading-container fade-in">
          <ProgressSpinner style="width: 60px; height: 60px" strokeWidth="3" />
          <p class="loading-text">Daten werden geladen...</p>
        </div>

        <!-- Fehlermeldung -->
        <Message v-if="error" severity="error" class="mt-4 message-modern fade-in" :closable="false">
          <div class="message-content">
            <i class="pi pi-exclamation-circle"></i>
            <span>{{ error }}</span>
          </div>
        </Message>

        <!-- Ergebnis -->
        <div v-if="result" class="result-container mt-4 fade-in">
          <div class="result-header">
            <i class="pi pi-check-circle success-icon"></i>
            <h3>Ergebnis gefunden</h3>
          </div>
          <div class="grid-container">
            <div class="grid-item" v-for="(item, index) in [
              { label: 'Barcode', value: result.barcode || barcode, icon: 'pi-qrcode' },
              { label: 'Auftrag', value: result.auftrag || '-', icon: 'pi-file' },
              { label: 'Artikel', value: result.artikel || '-', icon: 'pi-box' },
              { label: 'Menge', value: result.menge || '-', icon: 'pi-hashtag' },
              { label: 'Zeitstempel', value: result.timestamp || '-', icon: 'pi-clock' }
            ]" :key="index" :style="{ animationDelay: `${index * 0.1}s` }">
              <div class="item-label">
                <i class="pi" :class="item.icon"></i>
                <span>{{ item.label }}</span>
              </div>
              <span class="item-value">{{ item.value }}</span>
            </div>
          </div>
        </div>

        <!-- Keine Ergebnisse -->
        <Message v-if="searched && !loading && !result && !error" severity="info" class="mt-4 message-modern fade-in" :closable="false">
          <div class="message-content">
            <i class="pi pi-info-circle"></i>
            <span>Kein Eintrag gefunden</span>
          </div>
        </Message>
      </template>
    </Card>
    <div class="version-display">
      <i class="pi pi-tag"></i>
      v{{ version }}
    </div>
  </div>
</template>

<style scoped>
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

@keyframes float {
  0%, 100% {
    transform: translateY(0) rotate(0deg);
  }
  50% {
    transform: translateY(-20px) rotate(5deg);
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateX(-10px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.fade-in {
  animation: fadeIn 0.5s ease-out;
}

.app-container {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  min-height: 100vh;
  padding: 2rem 1rem 3rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 50%, #f093fb 100%);
  position: relative;
  overflow: hidden;
}

.background-shapes {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  z-index: 0;
}

.shape {
  position: absolute;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  animation: float 6s ease-in-out infinite;
}

.shape-1 {
  width: 300px;
  height: 300px;
  top: 10%;
  left: -100px;
  animation-delay: 0s;
}

.shape-2 {
  width: 200px;
  height: 200px;
  top: 60%;
  right: -50px;
  animation-delay: 2s;
}

.shape-3 {
  width: 150px;
  height: 150px;
  bottom: 20%;
  left: 50%;
  animation-delay: 4s;
}

.header-section {
  text-align: center;
  margin-bottom: 2rem;
  z-index: 1;
  animation: fadeIn 0.8s ease-out;
}

.logo-container {
  margin-bottom: 1rem;
}

.logo-icon {
  font-size: 4rem;
  color: white;
  text-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  animation: float 3s ease-in-out infinite;
}

.main-title {
  font-size: 2.5rem;
  font-weight: 700;
  color: white;
  margin: 0.5rem 0;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  letter-spacing: -0.5px;
}

.main-subtitle {
  font-size: 1.1rem;
  color: rgba(255, 255, 255, 0.9);
  margin: 0;
  font-weight: 400;
}

.version-display {
  margin-top: 2rem;
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  gap: 0.5rem;
  z-index: 1;
  padding: 0.5rem 1rem;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.search-card {
  width: 100%;
  max-width: 650px;
  z-index: 1;
  animation: fadeIn 1s ease-out;
}

.glass-effect {
  background: rgba(255, 255, 255, 0.95) !important;
  backdrop-filter: blur(20px);
  border-radius: 20px !important;
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

.search-form {
  display: flex;
  gap: 1rem;
  margin-bottom: 0;
  align-items: flex-start;
}

.input-group {
  flex: 1;
}

.modern-input {
  font-size: 1.05rem;
  transition: all 0.3s ease;
}

.modern-input:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
}

.search-button {
  padding: 0.75rem 2rem;
  font-weight: 600;
  transition: all 0.3s ease;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
}

.search-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.w-full {
  width: 100%;
}

.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 3rem 0;
  gap: 1rem;
}

.loading-text {
  color: var(--p-text-color-secondary);
  font-size: 1.05rem;
  margin: 0;
}

.mt-4 {
  margin-top: 1.5rem;
}

.message-modern {
  border-radius: 12px;
  border-left-width: 4px;
}

.message-content {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 1rem;
}

.message-content i {
  font-size: 1.25rem;
}

.result-container {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.05) 0%, rgba(118, 75, 162, 0.05) 100%);
  border-radius: 16px;
  padding: 1.5rem;
  border: 1px solid rgba(102, 126, 234, 0.1);
}

.result-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 1.5rem;
}

.result-header h3 {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 600;
  color: var(--p-text-color);
}

.success-icon {
  font-size: 1.75rem;
  color: #22c55e;
}

.grid-container {
  display: grid;
  grid-template-columns: 1fr;
  gap: 0.75rem;
}

.grid-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.25rem;
  background: white;
  border-radius: 12px;
  border: 1px solid rgba(102, 126, 234, 0.1);
  transition: all 0.3s ease;
  animation: slideIn 0.5s ease-out backwards;
}

.grid-item:hover {
  transform: translateX(5px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.15);
  border-color: rgba(102, 126, 234, 0.3);
}

.item-label {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-weight: 600;
  color: var(--p-text-color-secondary);
  font-size: 0.95rem;
}

.item-label i {
  font-size: 1.25rem;
  color: #667eea;
}

.item-value {
  font-weight: 600;
  color: var(--p-text-color);
  text-align: right;
  font-size: 1.05rem;
}

@media (max-width: 768px) {
  .app-container {
    padding: 1.5rem 0.75rem 2rem;
  }

  .main-title {
    font-size: 2rem;
  }

  .main-subtitle {
    font-size: 1rem;
  }

  .logo-icon {
    font-size: 3rem;
  }
}

@media (max-width: 480px) {
  .search-form {
    flex-direction: column;
  }
  
  .input-group {
    width: 100%;
  }
  
  .search-button {
    width: 100%;
  }

  .grid-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }

  .item-value {
    text-align: left;
  }
}
</style>
