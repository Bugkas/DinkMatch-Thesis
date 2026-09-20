# 🏓 DinkMatch - Queueing (Open Play)

DinkMatch is a smart, local-first matchmaking system for singles and doubles pickleball/tennis matches. It uses a self-calibrating zero-sum rating system (similar to Elo but optimized for doubles pairing and weakest-link carrying mechanics) to automatically balance matches and organize player queues for racket clubs.

---

## 🚀 Key Features

- **Balanced Team Generation**: Algorithms designed to optimize pairings in both singles (1v1) and doubles (2v2) matches.
- **Double-Specific Rating Mechanics**:
  - **Weakest Link**: Doubles team strength is calculated prioritizing the weaker player's rating to prevent extreme pairing imbalances.
  - **Carry/Forgiveness Modifier**: High-rated players carrying low-rated players are rewarded/penalized more dynamically based on match outcomes.
- **Local-First & Offline Support**: Fully functional PWA that saves data locally via `LocalStorage` and works entirely offline.
- **Interactive Play Screen**: Drag-and-drop or tap-to-swap player adjustments with responsive layouts.
- **Voice Announcer**: TTS capability for reading out match pairings and queue status.

---

## 📂 Project Structure

A standard Quasar + Vite + TypeScript PWA directory structure:

```
├── android-twa/        # Android Trusted Web Activity configuration (Bubblewrap bundle)
├── docs/               # Built static deployment folder
├── public/             # Static assets (images, icons, CSV templates)
├── src/                # Core application source
│   ├── assets/         # App logos and visuals
│   ├── boot/           # Quasar boot initialization scripts
│   ├── components/     # Reusable UI components (Play, Club, Landing UI)
│   ├── composables/    # Reactive state and composables (Auth, P2P, Announcer, Matchmaking)
│   ├── css/            # Global styling stylesheets (Sass/SCSS variables)
│   ├── layouts/        # Page layouts (e.g. MainLayout)
│   ├── pages/          # Application views/routes (Landing, Play, Club, Login, settings)
│   ├── router/         # Application router configuration
│   ├── services/       # Matchmaking, Profile, and Cloud sync business logic
│   └── utils/          # Rating replays and player utilities
├── package.json        # Node dependency configuration
├── quasar.config.ts    # Quasar CLI and builder configurations
├── tsconfig.json       # TypeScript options configuration
└── vitest.config.ts    # Testing environment setup
```

---

## 🛠️ Development & Building

### 1. Install Dependencies

```bash
npm install
# or
yarn install
```

### 2. Run Local Development Server

Start the hot-reloading development environment:

```bash
npx quasar dev
# or
npm run dev
```

### 3. Build for Web Production (PWA)

Generate the production-ready build output:

```bash
npx quasar build
# or
npm run build
```

### 4. Code Quality & Formatting

```bash
# Run lint check
npm run lint

# Format codebase
npm run format
```

### 5. Build Android App Bundle (AAB)

Bubblewrap signing and building for Android distribution:

```bash
npm run build:aab
```

---

## 📖 Matchmaking Logic & Rules

For detailed information on the rating formulas, player validation, and tournament structures, refer to the documentation templates and guidelines.
