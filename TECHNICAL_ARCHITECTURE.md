# Technical Architecture Changes

## Overview
This document describes the technical architecture changes made to implement the new features and fix the critical date shifting bug.

## Data Model Changes

### Problem: Date Shifting Bug
**Original Issue:**
- History stored as fixed-size array (900 elements)
- When a new day started, app tried to append to array
- Array indexing failed, causing data to override instead of shift
- "Last 7 days" concept in backend caused index calculation errors

**Root Cause:**
```typescript
// Problem: assumes fixed array size
const history = [0, 1, 1, 0, 1, -1, 0]; // Only fixed size
const dayIndex = history.length - 1 - dayOffset; // Breaks with growth
```

**Solution Implemented:**
```typescript
// New approach: track actual dates
interface Habit {
  history: number[];              // Grows as needed
  lastRecordedDate: string;       // YYYY-MM-DD format
  // ... other fields
}
```

### Daily Entry Management
**Process:**
1. On app load, `ensureTodayEntriesForAll()` is called
2. Each habit checked: `lastRecordedDate === getTodayDateString()`
3. If not today, new entry (0) added to history array
4. `lastRecordedDate` updated to today
5. Changes saved to storage

**Implementation:**
```typescript
export function ensureTodayEntry(habit: Habit): Habit {
  const todayDate = getTodayDateString();
  if (habit.lastRecordedDate === todayDate) {
    return habit; // Already has today
  }
  
  const newHistory = [...(habit.history || [])];
  newHistory.push(0); // Add blank for today
  
  return {
    ...habit,
    history: newHistory,
    lastRecordedDate: todayDate,
  };
}
```

## UI/Display Architecture

### Display Pipeline
```
Raw Habits (from storage)
    ↓
Migration & Daily Check (ensure today's data exists)
    ↓
Filter Pipeline (hide archived, hide entered, group filter)
    ↓
Sort Pipeline (by name, color, score, status, or manual)
    ↓
Grey Styling Applied (if greyEntered filter enabled)
    ↓
Display to User
```

### Filter Application
```typescript
// Filter logic in lib/habit-display-utils.ts
function filterHabits(habits, filters) {
  return habits.filter(habit => {
    if (filters.hideArchived && habit.archived) return false;
    if (filters.hideEntered) {
      const today = habit.history[habit.history.length - 1] || 0;
      if (today === 1 || today === -1) return false;
    }
    if (filters.group && habit.group !== filters.group) return false;
    return true;
  });
}
```

### Sort Implementation
Each sort type compares habits differently:
- **Manual**: Uses `sortOrder` field (set by drag-drop)
- **Name**: String comparison with `localeCompare()`
- **Color**: Uses predefined color order array
- **Score**: Compares `computeStats(habit).score`
- **Status**: Today's value determines order (1 > 0 > -1)

### Combined Display
```typescript
function getDisplayHabits(habits, filters, sortBy, greyEntered) {
  let displayed = filterHabits(habits, filters); // Apply filters
  displayed = sortHabits(displayed, sortBy);     // Apply sort
  
  if (greyEntered) {
    return displayed.map(habit => ({
      ...habit,
      isGreyed: isTodayCompleted(habit)
    }));
  }
  return displayed;
}
```

## Component Architecture

### New Components

#### DailyPromptBanner (`components/daily-prompt-banner.tsx`)
- **Props**: `habits`, `onToggleHabit`
- **State**: `currentHabitIndex`, `isVisible`
- **Features**: 
  - Finds incomplete habits
  - Rotates every 5 seconds
  - Triggers completion on button click

#### HabitControls (`components/habit-controls.tsx`)
- **Props**: `habits`, `onFilterChange`, `onSortChange`
- **State**: Menus, current sort, current filters
- **Features**:
  - Sort selector menu
  - Filter toggles
  - Group selector
  - Stats display

### Modified Components

#### HabitTracker (hub component)
**State Management:**
```typescript
const [habits, setHabits] = useState<Habit[]>([]);
const [filters, setFilters] = useState<HabitFilters>({
  hideArchived: true // Default
});
const [sortBy, setSortBy] = useState<SortOption>('manual');
```

**Data Flow:**
1. Load raw habits from storage
2. Migrate and ensure today's entries
3. Apply filters/sort to create displayHabits
4. Pass displayHabits to HabitGrid
5. Grid renders filtered/sorted habits

#### HabitGrid (improved)
**Drag & Drop Integration:**
```typescript
const [draggedId, setDraggedId] = useState<string | null>(null);
const [dragOverId, setDragOverId] = useState<string | null>(null);

// Events:
handleDragStart(habitId) -> setDraggedId(habitId)
handleDragOver(habitId) -> setDragOverId(habitId)
handleDrop(toId) -> onReorder(draggedId, toId)
handleDragEnd() -> clear drag state
```

**Reorder Handler:**
```typescript
const handleReorder = (fromId, toId) => {
  const newHabits = reorder(habits, fromId, toId);
  const withSortOrder = newHabits.map((h, i) => ({
    ...h,
    sortOrder: i
  }));
  setHabits(withSortOrder);
  updateHabits(withSortOrder);
};
```

#### HabitRow (enhanced)
**New Features:**
- Inline name editing with keyboard shortcuts
- Group selection sub-menu
- Enhanced long-press menu (6 items + sub-menu)
- Grey styling support
- Edit, Toggle, Archive, Delete, SetGroup options

**Long Press Menu Structure:**
```
Edit name
Toggle today
Group selector (sub-menu)
Archive/Unarchive
Delete
Close
```

#### HabitCell (enhanced)
**Vibration Patterns:**
```typescript
if (value === 1) {
  navigator.vibrate([10, 5, 10]); // Skip
} else if (value === -1) {
  navigator.vibrate([15]);         // Reset
} else {
  navigator.vibrate(10);           // Done
}
```

## State Management

### Global State (HabitTracker)
- `habits`: Full list from storage
- `filters`: Current filter settings
- `sortBy`: Current sort option
- `showAddModal`, `showExportModal`: UI modals

### Component State

**HabitGrid:**
- `draggedId`: Currently dragged habit
- `dragOverId`: Current drag hover target

**HabitControls:**
- `showSortMenu`, `showFilterMenu`, `showGroupMenu`: Menu visibility
- `currentSort`: Selected sort option
- `filters`: Current filter state

**HabitRow:**
- `longPressTimer`: Timer ID for long press
- `showOptions`: Menu visibility
- `isEditingName`: Name edit mode
- `editedName`: Edited name in input
- `showGroupMenu`: Group sub-menu visibility

## Data Flow Diagram

```
User Action
    ↓
Component Handler
    ↓
Update State → Call updateHabits(newHabits)
    ↓
setHabits(newHabits)
    ↓
HabitGrid receives displayHabits
    ↓
Display Update
```

## Storage Layer

### Enhanced Habit Persistence
1. **IndexedDB**: Primary storage
2. **localStorage**: Fallback
3. **Firebase**: Cloud sync (if configured)

**Migration Strategy:**
- On load: Check each habit for missing fields
- Initialize defaults: `archived = false`, `group = 'default'`, `sortOrder = 0`
- Ensure today's entry exists before display

## Performance Optimizations

### Rendering
- Only displayed habits rendered (filtered)
- Memoization opportunity in future (React.memo)
- CSS transitions for drag-drop (GPU accelerated)

### Algorithms
- Single pass for filtering
- Binary-like search for sorting by score
- Color order array pre-computed

### Storage
- Batch updates to avoid multiple writes
- Conditional storage updates (only if changed)

## Browser APIs Used

### New APIs
- **Vibration API**: `navigator.vibrate(pattern)`
- **Drag & Drop API**: `draggable`, drag/drop events
- **localStorage/IndexedDB**: Already in use

### Fallbacks
- Vibration not supported? Silently ignored
- Drag & drop not supported? Falls back to click
- CSS Grid? Already supported in all modern browsers

## Type Safety

### New TypeScript Interfaces
```typescript
// In lib/habit-utils.ts
interface Habit {
  // ... existing
  lastRecordedDate?: string;
  archived?: boolean;
  group?: string;
  sortOrder?: number;
}

// In lib/habit-display-utils.ts
interface HabitFilters {
  hideArchived?: boolean;
  hideEntered?: boolean;
  greyEntered?: boolean;
  group?: string;
}

type SortOption = 'manual' | 'name' | 'color' | 'score' | 'status' | 'creation';
```

## Future Extension Points

1. **Cloud Sync**: Firebase integration already ready
2. **Notifications**: Push notification API for daily reminders
3. **Analytics**: Track completion patterns
4. **Themes**: Already implemented with hooks
5. **Export/Import**: Already implemented (can be enhanced)
6. **Reminders**: Add time-based notifications
7. **Custom Colors**: Already supported via hex codes
8. **Streaks**: Statistics already computed, could add UI

## Security Considerations

- All data client-side by default
- Optional Firebase for cloud sync
- No sensitive data in localStorage
- User email only displayed if authenticated
- No API keys in frontend code

## Browser Quirks Handled

1. **iOS Safari**: Touch events with long press
2. **Android**: Vibration API support
3. **Desktop**: Mouse events + keyboard
4. **PWA**: Works offline with service worker

## Debugging Tips

### Check Daily Entries
```javascript
// In browser console
const habits = await db.habits.getAll();
habits.forEach(h => console.log(h.name, h.lastRecordedDate, h.history.length));
```

### Test Filtering
```javascript
// Components/habit-tracker.tsx
console.log('Display habits:', displayHabits);
console.log('Filters:', filters);
console.log('Sort:', sortBy);
```

### Test Drag & Drop
```javascript
// Enable verbose logging in habit-grid.tsx
console.log('Drag start:', draggedId);
console.log('Drag over:', dragOverId);
```
