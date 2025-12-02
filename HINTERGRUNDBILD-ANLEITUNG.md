# Hintergrundbild hinzufügen

## 📍 Speicherort
Legen Sie Ihr Hintergrundbild hier ab:
```
/Users/lenn/Desktop/QrCodePaletteCheck/src/assets/
```

## 🖼️ Unterstützte Formate
- `.jpg` / `.jpeg`
- `.png`
- `.svg`
- `.webp`

## 📝 Anleitung

### 1. Bild hinzufügen
Kopieren Sie Ihr gewünschtes Bild in den `src/assets/` Ordner, z.B.:
- `background.jpg`
- `my-background.png`

### 2. Code anpassen
Öffnen Sie `src/App.vue` und suchen Sie die `.app-container` Klasse (ca. Zeile 200).

**Aktueller Code:**
```css
background: 
  linear-gradient(135deg, rgba(102, 126, 234, 0.85) 0%, rgba(118, 75, 162, 0.85) 50%, rgba(240, 147, 251, 0.85) 100%),
  url('@/assets/background-placeholder.svg') center center / cover no-repeat;
```

**Ersetzen Sie `background-placeholder.svg` mit Ihrem Dateinamen:**
```css
background: 
  linear-gradient(135deg, rgba(102, 126, 234, 0.85) 0%, rgba(118, 75, 162, 0.85) 50%, rgba(240, 147, 251, 0.85) 100%),
  url('@/assets/background.jpg') center center / cover no-repeat;
```

### 3. Optionen

**Mit Gradient-Overlay (empfohlen für bessere Lesbarkeit):**
```css
background: 
  linear-gradient(135deg, rgba(102, 126, 234, 0.85) 0%, rgba(118, 75, 162, 0.85) 50%, rgba(240, 147, 251, 0.85) 100%),
  url('@/assets/IHR-BILD.jpg') center center / cover no-repeat;
```

**Ohne Overlay (nur Bild):**
```css
background: url('@/assets/IHR-BILD.jpg') center center / cover no-repeat;
```

**Dunkles Overlay:**
```css
background: 
  linear-gradient(135deg, rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)),
  url('@/assets/IHR-BILD.jpg') center center / cover no-repeat;
```

## 💡 Tipps
- Verwenden Sie hochauflösende Bilder (mindestens 1920x1080px)
- Optimieren Sie große Bilder vorher (z.B. mit TinyPNG)
- Verwenden Sie `.webp` für bessere Performance
- Passen Sie die Overlay-Opacity an (0.85 = 85% Deckkraft)
