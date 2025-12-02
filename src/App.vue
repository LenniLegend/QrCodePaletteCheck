<script setup>
import { ref, onUnmounted } from 'vue';
import InputText from 'primevue/inputtext';
import Button from 'primevue/button';
import Card from 'primevue/card';
import Message from 'primevue/message';
import ProgressSpinner from 'primevue/progressspinner';
import FloatLabel from 'primevue/floatlabel';
import QrScanner from 'qr-scanner';

// Reaktive Variablen
const barcode = ref('');
const result = ref(null);
const error = ref(null);
const loading = ref(false);
const searched = ref(false); // Um "Kein Eintrag gefunden" nur nach Suche anzuzeigen
const showScanner = ref(false);
let qrScanner = null;

// API-Konfiguration
// Wir nutzen nun den Proxy (in vite.config.js oder nginx.conf), um CORS-Fehler zu vermeiden
const API_BASE_URL = '/api/barcode';

const version = __APP_VERSION__;

// QR-Code Scanner Funktion
const scanQRCode = async () => {
  showScanner.value = true;
  error.value = null;
  
  // Warte kurz, damit das Video-Element im DOM ist
  await new Promise(resolve => setTimeout(resolve, 100));
  
  const videoElement = document.getElementById('qr-video');
  if (!videoElement) {
    error.value = 'Scanner konnte nicht initialisiert werden.';
    showScanner.value = false;
    return;
  }
  
  try {
    qrScanner = new QrScanner(
      videoElement,
      result => {
        barcode.value = result.data;
        stopScanner();
        searchBarcode();
      },
      {
        returnDetailedScanResult: true,
        highlightScanRegion: true,
        highlightCodeOutline: true,
        preferredCamera: 'environment'
      }
    );
    
    await qrScanner.start();
  } catch (err) {
    console.error('Scanner error:', err);
    error.value = 'Kamera-Zugriff verweigert oder nicht verfügbar.';
    showScanner.value = false;
  }
};

const stopScanner = () => {
  if (qrScanner) {
    qrScanner.stop();
    qrScanner.destroy();
    qrScanner = null;
  }
  showScanner.value = false;
};

// Cleanup beim Unmount
onUnmounted(() => {
  stopScanner();
});

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
      <p class="main-subtitle">QR-Code Klammer scannen um zu erfahren, was auf der Palette geladen ist.</p>
    </div>

    <Card class="search-card glass-effect">
      <template #content>
        <div class="search-form">
          <div class="input-group">
            <FloatLabel class="input-wrapper">
                <InputText 
                  id="barcode" 
                  v-model="barcode" 
                  @keydown.enter="searchBarcode" 
                  class="w-full modern-input" 
                  :disabled="loading"
                />
                <label for="barcode">Barcode eingeben</label>
            </FloatLabel>
            <Button 
              v-if="barcode"
              icon="pi pi-times" 
              @click="barcode = ''; result = null; error = null; searched = false"
              class="clear-button"
              text
              rounded
              severity="secondary"
              :disabled="loading"
            />
          </div>
          <Button 
            icon="pi pi-camera" 
            @click="scanQRCode" 
            :disabled="loading"
            class="camera-button mobile-only"
            severity="secondary"
            v-tooltip.top="'QR-Code scannen'"
          />
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

    <!-- QR-Code Scanner Overlay -->
    <div v-if="showScanner" class="scanner-overlay" @click.self="stopScanner">
      <div class="scanner-container">
        <div class="scanner-header">
          <h3>QR-Code scannen</h3>
          <Button 
            icon="pi pi-times" 
            @click="stopScanner" 
            class="close-scanner-button"
            text
            rounded
            severity="secondary"
          />
        </div>
        <div class="scanner-video-wrapper">
          <video id="qr-video" class="scanner-video"></video>
          <div class="scanner-frame"></div>
        </div>
        <p class="scanner-hint">Halte den QR-Code in den Rahmen</p>
      </div>
    </div>

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
  background: 
    linear-gradient(to bottom, rgba(0, 0, 0, 0.4), rgba(0, 0, 0, 0.6)),
    url('@/assets/background.png') center center / cover no-repeat fixed;
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
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(15px);
  animation: float 6s ease-in-out infinite;
  box-shadow: 0 8px 32px rgba(255, 255, 255, 0.1);
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
  text-shadow: 0 4px 30px rgba(0, 0, 0, 0.5), 0 2px 10px rgba(0, 0, 0, 0.3);
  filter: drop-shadow(0 0 20px rgba(255, 255, 255, 0.3));
}

.main-title {
  font-size: 2.5rem;
  font-weight: 700;
  color: white;
  margin: 0.5rem 0;
  text-shadow: 0 4px 20px rgba(0, 0, 0, 0.6), 0 2px 8px rgba(0, 0, 0, 0.4);
  letter-spacing: -0.5px;
}

.main-subtitle {
  font-size: 1.1rem;
  color: rgba(255, 255, 255, 0.95);
  margin: 0;
  font-weight: 400;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.5);
}

.version-display {
  margin-top: 2rem;
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.95);
  display: flex;
  align-items: center;
  gap: 0.5rem;
  z-index: 1;
  padding: 0.5rem 1rem;
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(15px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  text-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
}

.search-card {
  width: 100%;
  max-width: 650px;
  z-index: 1;
  animation: fadeIn 1s ease-out;
}

.glass-effect {
  background: rgba(255, 255, 255, 0.98) !important;
  backdrop-filter: blur(30px) saturate(180%);
  border-radius: 24px !important;
  border: 1px solid rgba(255, 255, 255, 0.4);
  box-shadow: 0 12px 48px rgba(0, 0, 0, 0.25), 0 0 0 1px rgba(255, 255, 255, 0.1) inset;
}

.search-form {
  display: flex;
  gap: 1rem;
  margin-bottom: 0;
  align-items: stretch;
}

.input-group {
  flex: 1;
  display: flex;
  align-items: center;
  position: relative;
  gap: 0.5rem;
}

.input-wrapper {
  flex: 1;
  display: flex;
}

.modern-input {
  font-size: 1.05rem;
  transition: all 0.3s ease;
  height: 100%;
  padding-right: 3rem;
}

.modern-input:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
}

.clear-button {
  position: absolute;
  right: 0.5rem;
  z-index: 10;
  width: 2rem;
  height: 2rem;
  min-width: 2rem;
  padding: 0;
  opacity: 0.6;
  transition: all 0.2s ease;
}

.clear-button:hover {
  opacity: 1;
  transform: scale(1.1);
}

.search-button {
  padding: 0.75rem 2rem;
  font-weight: 600;
  transition: all 0.2s ease;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Höhere Spezifität für PrimeVue Button-Elemente */
.p-button.search-button,
.search-button.p-button {
  background-color: #002E62 !important;
  color: #ffffff !important;
  border: 1px solid rgba(0, 46, 98, 0.95) !important;
  box-shadow: 0 6px 18px rgba(0, 46, 98, 0.18) !important;
}

.p-button.search-button .p-button-icon,
.search-button.p-button .p-button-icon,
.p-button.search-button .pi,
.search-button.p-button .pi {
  color: #ffffff !important;
}

.p-button.search-button:hover,
.search-button.p-button:hover {
  transform: translateY(-2px);
  background-color: #00407a !important;
  border-color: rgba(0, 64, 122, 0.95) !important;
  box-shadow: 0 8px 24px rgba(0, 46, 98, 0.25) !important;
}

.search-button:disabled,
.search-button[disabled],
.p-button.search-button:disabled,
.p-button.search-button[disabled] {
  opacity: 0.65;
  cursor: not-allowed;
  box-shadow: none !important;
}

.camera-button {
  padding: 0.75rem;
  font-weight: 600;
  transition: all 0.3s ease;
  height: 100%;
  min-width: 3rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.camera-button:hover {
  transform: translateY(-2px);
}

/* Kamera-Button nur auf mobilen Geräten anzeigen */
.mobile-only {
  display: none;
}

@media (max-width: 768px) {
  .mobile-only {
    display: flex;
  }
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
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.08) 0%, rgba(255, 255, 255, 0.04) 100%);
  border-radius: 16px;
  padding: 1.5rem;
  border: 1px solid rgba(102, 126, 234, 0.15);
  backdrop-filter: blur(10px);
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
  background: rgba(255, 255, 255, 0.98);
  border-radius: 12px;
  border: 1px solid rgba(102, 126, 234, 0.12);
  transition: all 0.3s ease;
  animation: slideIn 0.5s ease-out backwards;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.grid-item:hover {
  transform: translateX(5px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.2);
  border-color: rgba(102, 126, 234, 0.4);
  background: rgba(255, 255, 255, 1);
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

/* QR-Code Scanner Overlay */
.scanner-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.95);
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: fadeIn 0.3s ease-out;
}

.scanner-container {
  width: 90%;
  max-width: 500px;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.scanner-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 1rem;
}

.scanner-header h3 {
  color: white;
  margin: 0;
  font-size: 1.5rem;
}

.close-scanner-button {
  color: white !important;
}

.scanner-video-wrapper {
  position: relative;
  width: 100%;
  aspect-ratio: 1;
  border-radius: 16px;
  overflow: hidden;
  background: #000;
}

.scanner-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.scanner-frame {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 70%;
  height: 70%;
  border: 3px solid #00ff00;
  border-radius: 12px;
  box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.5);
  pointer-events: none;
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% {
    border-color: #00ff00;
    box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.5), 0 0 20px rgba(0, 255, 0, 0.5);
  }
  50% {
    border-color: #00cc00;
    box-shadow: 0 0 0 9999px rgba(0, 0, 0, 0.5), 0 0 30px rgba(0, 255, 0, 0.8);
  }
}

.scanner-hint {
  text-align: center;
  color: rgba(255, 255, 255, 0.9);
  font-size: 1rem;
  margin: 0;
  padding: 0 1rem;
}
</style>
