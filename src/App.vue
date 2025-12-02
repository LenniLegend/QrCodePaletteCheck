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
      const data = await response.json();
      // Prüfen ob Daten vorhanden sind
      // Wir nehmen an, dass die API ein Objekt zurückgibt. Wenn es leer ist oder null, wurde nichts gefunden.
      if (data && Object.keys(data).length > 0) {
        result.value = data;
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
    <Card class="search-card">
      <template #title>Barcode Check</template>
      <template #subtitle>Geben Sie einen Barcode ein, um Details abzurufen.</template>
      <template #content>
        <div class="search-form">
          <div class="input-group">
            <FloatLabel>
                <InputText id="barcode" v-model="barcode" @keydown.enter="searchBarcode" class="w-full" />
                <label for="barcode">Barcode</label>
            </FloatLabel>
          </div>
          <Button label="Suchen" icon="pi pi-search" @click="searchBarcode" :loading="loading" />
        </div>

        <!-- Ladeanzeige -->
        <div v-if="loading" class="loading-container">
          <ProgressSpinner style="width: 50px; height: 50px" strokeWidth="4" />
          <p>Lade Daten...</p>
        </div>

        <!-- Fehlermeldung -->
        <Message v-if="error" severity="error" class="mt-3" :closable="false">{{ error }}</Message>

        <!-- Ergebnis -->
        <div v-if="result" class="result-container mt-4">
          <h3>Ergebnis</h3>
          <div class="grid-container">
            <div class="grid-item">
              <span class="label">Barcode:</span>
              <span class="value">{{ result.barcode || barcode }}</span>
            </div>
            <div class="grid-item">
              <span class="label">Auftrag:</span>
              <span class="value">{{ result.auftrag || '-' }}</span>
            </div>
            <div class="grid-item">
              <span class="label">Artikel:</span>
              <span class="value">{{ result.artikel || '-' }}</span>
            </div>
            <div class="grid-item">
              <span class="label">Menge:</span>
              <span class="value">{{ result.menge || '-' }}</span>
            </div>
            <div class="grid-item">
              <span class="label">Zeitstempel:</span>
              <span class="value">{{ result.timestamp || '-' }}</span>
            </div>
          </div>
        </div>

        <!-- Keine Ergebnisse -->
        <Message v-if="searched && !loading && !result && !error" severity="info" class="mt-3" :closable="false">
          Kein Eintrag gefunden.
        </Message>
      </template>
    </Card>
    <div class="version-display">v{{ version }}</div>
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  min-height: 100vh;
  padding-top: 4rem;
  background-color: var(--p-surface-100);
}

.version-display {
  margin-top: 1rem;
  font-size: 0.8rem;
  color: var(--p-text-color-secondary);
  opacity: 0.7;
}

.search-card {
  width: 100%;
  max-width: 600px;
  margin: 0 1rem;
}

.search-form {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
  align-items: flex-start;
}

.input-group {
  flex: 1;
}

.w-full {
  width: 100%;
}

.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 2rem 0;
  color: var(--p-text-color-secondary);
}

.mt-3 {
  margin-top: 1rem;
}

.mt-4 {
  margin-top: 1.5rem;
}

.grid-container {
  display: grid;
  grid-template-columns: 1fr;
  gap: 0;
  background: var(--p-surface-0);
  border-radius: var(--p-border-radius);
  border: 1px solid var(--p-surface-200);
  overflow: hidden;
}

.grid-item {
  display: flex;
  justify-content: space-between;
  padding: 1rem;
  border-bottom: 1px solid var(--p-surface-200);
}

.grid-item:last-child {
  border-bottom: none;
}

.label {
  font-weight: 600;
  color: var(--p-text-color-secondary);
}

.value {
  font-weight: 500;
  color: var(--p-text-color);
  text-align: right;
}

@media (max-width: 480px) {
  .search-form {
    flex-direction: column;
  }
  
  .input-group {
    width: 100%;
  }
  
  .p-button {
    width: 100%;
  }
}
</style>
