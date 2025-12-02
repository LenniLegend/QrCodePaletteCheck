<template>
  <div class="inline-camera">
    <div class="video-container">
      <!-- native video path (BarcodeDetector) -->
      <video v-if="useNative" ref="videoEl" autoplay playsinline muted></video>

      <!-- html5-qrcode container (fallback) -->
      <div v-if="useHtml5" ref="html5El" class="html5qrcode-container"></div>

      <div class="camera-overlay" v-if="!supported">{{ errorMsg || 'Kamera-Erkennung nicht verfügbar' }}</div>
    </div>

    <div class="camera-controls">
      <Button label="Schließen" icon="pi pi-times" class="p-button-secondary" @click="stop" />
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import Button from 'primevue/button'

export default {
  name: 'InlineCameraScanner',
  components: { Button },
  emits: ['scan-result', 'error', 'close'],
  setup (props, { emit }) {
    const videoEl = ref(null)
    const html5El = ref(null)
    let stream = null
    let detector = null
    let rafId = null
    let html5Scanner = null
    const supported = ref(true)
    const errorMsg = ref('')
    const useHtml5 = ref(false)
    const useNative = ref(false)

    async function startCamera () {
      try {
        // Prefer native BarcodeDetector when available
        if ('BarcodeDetector' in window) {
          try {
            detector = new BarcodeDetector({ formats: ['qr_code'] })
            useNative.value = true
            useHtml5.value = false
          } catch (e) {
            detector = null
          }
        }

        if (detector) {
          // native path
          stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } })
          if (videoEl.value) {
            videoEl.value.srcObject = stream
            await videoEl.value.play().catch(() => {})
          }
          supported.value = true
          scanLoop()
          return
        }

        // Fallback: use html5-qrcode
        useHtml5.value = true
        useNative.value = false
        try {
          const module = await import('html5-qrcode')
          const { Html5Qrcode } = module
          // ensure container exists
          await nextTick()
          if (!html5El.value) throw new Error('HTML5 QR container missing')
          const id = html5El.value.id || ('html5qr_' + Math.random().toString(36).slice(2, 9))
          html5El.value.id = id
          html5Scanner = new Html5Qrcode(id)

          // Try multiple camera configs to avoid OverconstrainedError on some devices
          const configs = [
            { facingMode: 'environment' },
            // older or picky browsers may prefer no facingMode
            undefined
          ]

          let started = false
          let lastErr = null
          for (const cfg of configs) {
            try {
              await html5Scanner.start(
                cfg,
                { fps: 10, qrbox: { width: 250, height: 250 } },
                (decodedText) => {
                  emit('scan-result', { success: true, payload: decodedText, raw: decodedText })
                  stop()
                },
                (_errorMessage) => {
                  // per-frame decode error; ignore
                }
              )
              started = true
              break
            } catch (e) {
              lastErr = e
              // stop and try next config
              try { await html5Scanner.stop() } catch (_) {}
            }
          }

          if (!started) {
            const msg = lastErr && lastErr.message ? lastErr.message : String(lastErr)
            supported.value = false
            errorMsg.value = 'Kamera konnte nicht gestartet werden: ' + msg
            emit('error', new Error(errorMsg.value))
          } else {
            supported.value = true
            errorMsg.value = ''
          }
        } catch (e) {
          supported.value = false
          errorMsg.value = 'Kamera konnte nicht gestartet werden: ' + (e && e.message ? e.message : String(e))
          emit('error', new Error(errorMsg.value))
        }
      } catch (err) {
        supported.value = false
        errorMsg.value = 'Kamera-Zugriff verweigert oder nicht verfügbar: ' + (err && err.message ? err.message : String(err))
        emit('error', new Error(errorMsg.value))
      }
    }

    async function scanLoop () {
      if (!videoEl.value || !detector) return
      try {
        const results = await detector.detect(videoEl.value)
        if (results && results.length > 0) {
          const r = results[0]
          const raw = r.rawValue || (r.raw && r.raw.value) || ''
          emit('scan-result', { success: true, payload: raw, raw })
          stop()
          return
        }
      } catch (e) {
        // ignore intermittent detector errors
      }

      rafId = requestAnimationFrame(scanLoop)
    }

    function stop () {
      if (rafId) {
        cancelAnimationFrame(rafId)
        rafId = null
      }
      if (stream) {
        stream.getTracks().forEach(t => t.stop())
        stream = null
      }
      if (html5Scanner) {
        html5Scanner.stop().catch(() => {})
        html5Scanner = null
      }
      useHtml5.value = false
      useNative.value = false
      emit('close')
    }

    onMounted(() => {
      startCamera()
    })

    onBeforeUnmount(() => {
      stop()
    })

    return { videoEl, html5El, supported, stop, useHtml5, useNative, errorMsg }
  }
}
</script>

<style scoped>
.inline-camera {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 100%;
}

.video-container {
  position: relative;
  width: 100%;
  padding-top: 75%;
  background: #000;
  border-radius: 16px;
  overflow: hidden;
}

.video-container video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.html5qrcode-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.camera-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  background: rgba(0, 0, 0, 0.7);
  font-weight: 700;
  padding: 1rem;
  text-align: center;
}

.camera-controls {
  display: flex;
  justify-content: center;
}
</style>
