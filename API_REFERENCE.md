# API Reference: New Group System

## Type Definitions

### HabitGroup
```typescript
export interface HabitGroup {
  id: string;                    // Unique identifier (e.g., "fitness-group")
  name: string;                  // Display name (e.g., "Fitness")
  habitIds: string[];            // IDs of habits in this group (references only)
  sortOrder?: number;            // Manual ordering
  createdAt?: number;            // Timestamp
}
```

### Habit (Modified)
```typescript
export interface Habit {
  id: string;
  name: string;
  color: HabitColor | string;
  type: 'boolean' | 'numeric';
  unit?: string;
  target?: string;
  frequency: string;
  time: string;
  log: Record<string, 0 | 1 | -1>; // Date-keyed log
  createdAt: number;
  archived?: boolean;
  group?: string;                 // ⚠️ DEPRECATED - kept for migration only
  sortOrder?: number;
  // Legacy fields...
}
```

---

## New Functions

### `getDefaultGroups(): HabitGroup[]`
Returns initial group structure (default group with no habits).

**Returns:**
```typescript
[
  {
    id: 'default',
    name: 'Default',
    habitIds: [],
    sortOrder: 0,
    createdAt: timestamp
  }
]
```

### `migrateHabit(habit: any): Habit`
Migrates old habit format to new format. Keeps existing `group` property for backwards compatibility but doesn't add it to new habits.

---

## Component API Changes

### HabitTracker
```typescript
// State changes:
groups: HabitGroup[]              // Was: string[]
selectedGroupForAdd: string | null // Was: addHabitToGroup

// Handler changes:
handleCreateHabit(
  name: string,
  type: 'boolean' | 'numeric',
  unit?: string,
  color?: string,
  groupId?: string                 // Was: group?: string
)

handleAddHabitToGroup(groupId: string)
handleDeleteHabit(habitId: string) // Now removes from all groups
handleCreateGroup(groupName: string) // Creates HabitGroup, not string
```

### GroupedHabitGrid
```typescript
interface GroupedHabitGridProps {
  habits: Habit[];
  groups: HabitGroup[];             // Was: string[]
  habitsMap?: Record<string, Habit>;  // NEW - optional lookup map
  // ... rest unchanged
}
```

### AddHabitModal
```typescript
interface AddHabitModalProps {
  groupId?: string | null;          // Was: defaultGroup?: string
  // ... rest unchanged
}
```

---

## Data Relationships

### Before (Wrong)
```
Group {
  habits: [Habit, Habit, Habit]  // ❌ Duplicates habit objects
}
```

When same habit in multiple groups:
- Data duplicated across groups
- Editing requires updates everywhere manually
- No single source of truth

### After (Correct)
```
Group {
  habitIds: ["h1", "h2", "h3"]   // ✅ References only
}

Habits {
  h1: Habit,
  h2: Habit,
  h3: Habit
}
```

When same habit in multiple groups:
- Habit object stored once
- Multiple groups can reference same ID
- Edit once, visible everywhere (true single source of truth)

---

## State Management Pattern

### Creating Independent Habit
```typescript
const newHabit: Habit = {
  id: Date.now().toString(),
  name,
  color,
  type,
  // ... other fields
  // NO group property
};

setHabits([...habits, newHabit]);
// Habit exists but not in any group
```

### Creating Habit in Group
```typescript
const newHabit: Habit = { /* same as above */ };

setHabits([...habits, newHabit]);

// Add to specific group
const updatedGroups = groups.map((group) => {
  if (group.id === groupId && !group.habitIds.includes(newHabit.id)) {
    return {
      ...group,
      habitIds: [...group.habitIds, newHabit.id]
    };
  }
  return group;
});
setGroups(updatedGroups);
```

### Adding Existing Habit to Group
```typescript
const updatedGroups = groups.map((group) => {
  if (group.id === groupId) {
    // Prevent duplicates
    if (!group.habitIds.includes(existingHabitId)) {
      return {
        ...group,
        habitIds: [...group.habitIds, existingHabitId]
      };
    }
  }
  return group;
});
setGroups(updatedGroups);
```

### Removing Habit Entirely (from everywhere)
```typescript
// Remove from state
const updatedHabits = habits.filter(h => h.id !== habitId);
setHabits(updatedHabits);

// Remove from all groups
const updatedGroups = groups.map((group) => ({
  ...group,
  habitIds: group.habitIds.filter(id => id !== habitId)
}));
setGroups(updatedGroups);
```

### Removing Habit from One Group Only
```typescript
const updatedGroups = groups.map((group) => 
  group.id === groupId
    ? { ...group, habitIds: group.habitIds.filter(id => id !== habitId) }
    : group
);
setGroups(updatedGroups);
// Habit still exists and might be in other groups
```

---

## Rendering Pattern

### Get Habits for a Group
```typescript
function getHabitsForGroup(
  group: HabitGroup,
  habitsMap: Record<string, Habit>
): Habit[] {
  return group.habitIds
    .map(id => habitsMap[id])
    .filter((habit): habit is Habit => habit !== undefined);
}
```

### Build Habits Map (for performance)
```typescript
const habitsMap = habits.reduce<Record<string, Habit>>((acc, habit) => {
  acc[habit.id] = habit;
  return acc;
}, {});
```

### Render Groups
```typescript
groups.map((group) => {
  const groupHabits = getHabitsForGroup(group, habitsMap);
  
  return (
    <div key={group.id}>
      <GroupHeader name={group.name} />
      {groupHabits.map(habit => (
        <HabitRow habit={habit} />
      ))}
    </div>
  );
});

// Same habit can appear in multiple group sections
// Same data displayed everywhere (single source of truth)
```

---

## Migration Path

### For Existing Users
1. ✅ Old `habit.group` property is preserved
2. ✅ Old habits with `group: 'default'` continue to work
3. 🔄 Future: Automated migration to new model
4. 🔄 Future: Settings for controlling migration behavior

### For New Users
1. ✅ No `group` property on new habits
2. ✅ Groups are explicit opt-in
3. ✅ Clean data model from day one

---

## UI/UX Flows

### Flow A: Create New Habit (Unassigned)
```
User clicks "+" (header)
  ↓
Chooses "Create New Habit"
  ↓
Fills in: name, type, color, unit
  ↓
Clicks "Create"
  ↓
Habit created (no group)
  ↓
Habit doesn't appear anywhere until user adds to group
```

### Flow B: Create New Habit in Group
```
User clicks "+" (inside group header)
  ↓
Chooses "Create New Habit"
  ↓
Fills in: name, type, color, unit
  ↓
Clicks "Create"
  ↓
Habit created AND added to that group
  ↓
Habit immediately visible in that group
```

### Flow C: Add Existing Habit to Group
```
User clicks "+" (inside group header)
  ↓
Chooses "Add Existing Habit"
  ↓
Modal shows all unassigned + other group habits
  ↓
User selects habit
  ↓
Habit ID added to this group's habitIds
  ↓
Habit now visible in this group (+ other groups it's in)
```

---

## Error Prevention

### Prevent Duplicate Habit in Group
```typescript
if (!group.habitIds.includes(habitId)) {
  group.habitIds.push(habitId);
}
```

### Handle Missing Habit During Render
```typescript
const groupHabits = group.habitIds
  .map(id => habitsMap[id])
  .filter((habit): habit is Habit => habit !== undefined);
```

### Handle Deleted Habit
- Automatically removed from all groups via `handleDeleteHabit`
- No orphaned IDs

---

## Performance Considerations

### Habit Lookup Optimization
```typescript
// Create map once, pass to GroupedHabitGrid
const habitsMap = useMemo(() => 
  habits.reduce((acc, h) => ({ ...acc, [h.id]: h }), {}),
  [habits]
);

<GroupedHabitGrid habitsMap={habitsMap} />
```

### Group Filtering
```typescript
// Filter groups if needed (e.g., hide empty)
const nonEmptyGroups = groups.filter(g => g.habitIds.length > 0);
```

---

## Summary: Key Differences

| Aspect | Old Model | New Model |
|--------|-----------|-----------|
| Habit location | `habit.group` property | Multiple groups via `habitIds` |
| Habit duplication | Yes (habit in multiple groups duplicates data) | No (single reference) |
| Single source of truth | No (habit exists in multiple places) | Yes (habit object owned by data layer) |
| Multi-group support | No | Yes |
| Group data structure | String array `string[]` | Object array `HabitGroup[]` |
| Auto-assignment | Yes (to 'default') | No (explicit only) |
| Edit consistency | Manual updates needed | Automatic (edit once, visible everywhere) |

