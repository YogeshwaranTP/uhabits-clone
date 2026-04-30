<img width="738" height="615" alt="image" src="https://github.com/user-attachments/assets/3bef467a-7ad9-41c7-872e-cc31dc2dc24c" /># Habits

A mobile-first habit tracker PWA built with Next.js, React, TypeScript, IndexedDB, and optional Firebase sync.

![Habits dashboard preview](<img width="738" height="615" alt="image" src="https://github.com/user-attachments/assets/bee09ad2-f1b9-48ba-b67e-ff7b5e249218" />
)

## Screenshots




<img width="1908" height="474" alt="image" src="https://github.com/user-attachments/assets/0b52540a-f4c8-472a-8ce5-b26386f38b65" />

## INSIDE A HABIT
<img width="825" height="726" alt="image" src="https://github.com/user-attachments/assets/f7a19f7c-8cd8-4a9a-bde6-ff6f436f49a6" />
<img width="760" height="299" alt="image" src="https://github.com/user-attachments/assets/7253f515-30df-4eb1-b5e8-5367b09baf7a" />

## Groups Enabled
<img width="1897" height="490" alt="image" src="https://github.com/user-attachments/assets/9d080624-576f-4105-86e1-eb8bcef816a4" />


## What It Does

- Track boolean and numeric habits with a date-keyed log model
- Use filters like hide archived, hide entered, and grey entered
- Organize habits into groups
- Open a dedicated detail page for each habit
- Export and import habit data as JSON, plus CSV export
- Work offline with IndexedDB and optional cloud sync with Firebase
- Install as a PWA on desktop or mobile

## Tech Stack

- Next.js 16
- React 19
- TypeScript
- Tailwind CSS
- Firebase Auth + Firestore
- IndexedDB with localStorage fallback

## Getting Started

### 1. Install dependencies

```bash
npm install
```

### 2. Start the dev server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### 3. Build for production

```bash
npm run build
npm run start
```

## Firebase Setup

Firebase is optional. Without it, the app still works locally with IndexedDB.

Create `.env.local` with your Firebase values:

```bash
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=...
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=...
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=...
NEXT_PUBLIC_FIREBASE_APP_ID=...
```

## Main Features

### Daily Tracking

- Boolean habits cycle through complete, skip, and empty states
- Numeric habits support value tracking
- Today is added without shifting historical data

### Habit Organization

- Main list and grouped mode
- Archived habits
- Manual ordering
- Per-habit colors

### Detail Experience

- Weekly and monthly graphs
- Calendar-style history view
- Streak and score statistics
- Inline group editing

### Data Portability

- JSON export/import
- CSV export
- Compatibility with older backup formats via migration

## Project Structure

```text
app/
  habit/[id]/page.tsx
  layout.tsx
  page.tsx
components/
  habit-tracker.tsx
  grouped-habit-grid.tsx
  habit-row.tsx
  habit-detail-page.tsx
  export-import-modal.tsx
hooks/
  use-auth.ts
  use-indexeddb.ts
lib/
  habit-utils.ts
  habit-display-utils.ts
  export-utils.ts
public/
  readme/
```

## Notes

- Local-first storage uses IndexedDB and falls back to localStorage when needed.
- Firebase sync is designed to preserve newer data and merge date-keyed logs across devices.
- The app is optimized for phone-sized layouts but also works on desktop.

## Documentation

- [TECHNICAL_ARCHITECTURE.md](./TECHNICAL_ARCHITECTURE.md)
- [USER_GUIDE.md](./USER_GUIDE.md)
- [API_REFERENCE.md](./API_REFERENCE.md)

## License

MIT
