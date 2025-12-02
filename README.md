# QrCodePaletteCheck

Repository: https://github.com/LenniLegend/QrCodePaletteCheck

Kurzes Vite + Vue-Projekt zur Überprüfung von Farbpaletten in QR-Codes.

**Inhalt**
- Build & Run lokal
- Docker & Docker Compose Deployment

**Voraussetzungen**
- Node.js (empfohlen v18+)
- npm
- Optional: Docker & Docker Compose für Container-Deployment

Entwicklung

```bash
npm install
npm run dev
```

Build (Production)

```bash
npm run build
npm run preview # optional, preview des Builds
```

Docker (Build + Serve via nginx)

Die Repository-Root enthält ein `Dockerfile` (multi-stage) und `docker-compose.yml`.

Kurzer Einzeiler zum Deployen auf einem Docker-Host (wenn das Repo bereits auf dem Host liegt):

```bash
docker compose up -d --build
```

Einzeiler, der das Repo klont und direkt startet:

```bash
git clone https://github.com/LenniLegend/QrCodePaletteCheck.git app && cd app && docker compose up -d --build
```

Weitere Hinweise
- Passe Port-Mapping in `docker-compose.yml` an, falls 80 bereits belegt ist.
- Die nginx-Standardkonfiguration dient statischen Builds; für spezielle Routen/Headers `nginx.conf` ergänzen.

Lizenz: MIT (siehe `LICENSE`)
