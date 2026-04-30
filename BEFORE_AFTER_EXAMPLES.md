# Before & After: Code Examples

This document shows side-by-side comparisons of the old vs. new implementation.

---

## 1. ARROW ICON (Safari Fix)

### BEFORE: Emoji (Renders as emoji on Safari)
```jsx
<button>
  ▶  {/* ❌ Emoji - renders differently across browsers */}
</button>
```

### AFTER: SVG (Consistent across all browsers)
```jsx
<button>
  <svg width="16" height="16" viewBox="0 0 24 24">
    <polyline 
      points="9 18 15 12 9 6" 
      fill="none"
      stroke={groupColor}
      strokeWidth="2"
      strokeLinecap="round"
      strokeLinejoin="round"
    />
  </svg>
</button>
```

**Result:** ✅ Works perfectly on Safari, Chrome, Firefox, Edge

---

## 2. GROUP DATA MODEL

### BEFORE: String Array + Habit.group Property
```typescript
// groups.ts
const groups: string[] = ['default', 'fitness', 'health'];

// A habit object
const habit: Habit = {
  id: 'h1',
  name: 'Run',
  group: 'fitness'  // ❌ Single group only
  // No multi-group support
};

// Filtering habits by group
function getHabitsForGroup(habitAll: Habit[], groupName: string) {
  return habitAll.filter(h => h.group === groupName);
  // ❌ If habit belongs to multiple groups, must duplicate data
}

// Problem: To show same habit in two groups
// Solution: Duplicate the habit object (wrong!)
```

### AFTER: HabitGroup Interface + ID References
```typescript
// groups.ts
const groups: HabitGroup[] = [
  {
    id: 'default',
    name: 'Default',
    habitIds: []  // ✅ References only
  },
  {
    id: 'fitness',
    name: 'Fitness',
    habitIds: ['h1', 'h2', 'h3']  // ✅ Just IDs
  },
  {
    id: 'health',
    name: 'Health',
    habitIds: ['h1', 'h4', 'h5']  // ✅ h1 in multiple groups!
  }
];

// A habit object
const habit: Habit = {
  id: 'h1',
  name: 'Run',
  // ✅ No group property - independent from groups
};

// Rendering habits for a group
function getHabitsForGroup(group: HabitGroup, habitsMap: Record<string, Habit>) {
  return group.habitIds
    .map(id => habitsMap[id])
    .filter((habit): habit is Habit => habit !== undefined);
}

// Result: h1 appears in both 'fitness' and 'health' groups
// Same habit object, no duplication, single source of truth
```

---

## 3. AUTO DEFAULT GROUP ASSIGNMENT

### BEFORE: Always Assigned to Default
```typescript
const handleCreateHabit = (
  name: string,
  type: 'boolean' | 'numeric',
  unit?: string,
  color?: string,
  group?: string
) => {
  const newHabit: Habit = {
    id: Date.now().toString(),
    name,
    color: color || '#4FC3F7',
    type,
    // ...
    group: group || 'default'  // ❌ Auto-assign to default
  };

  setHabits([...habits, newHabit]);
};
```

**Problem:** Every new habit appears in default group whether user wants it or not.

### AFTER: Independent Creation
```typescript
const handleCreateHabit = (
  name: string,
  type: 'boolean' | 'numeric',
  unit?: string,
  color?: string,
  groupId?: string  // ✅ Changed parameter name
) => {
  const newHabit: Habit = {
    id: Date.now().toString(),
    name,
    color: color || '#4FC3F7',
    type,
    // ... other fields
    // ✅ NO group property - habit is independent
  };

  // Add habit to state
  const updatedHabits = [...habits, newHabit];
  setHabits(updatedHabits);

  // If creating in a group, add to that group
  if (groupId) {
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
  }
};
```

**Result:** ✅ Habit independent, optional group assignment

---

## 4. DELETE HABIT HANDLING

### BEFORE: Only Remove from Habits List
```typescript
const handleDeleteHabit = (habitId: string) => {
  const updatedHabits = habits.filter((h) => h.id !== habitId);
  setHabits(updatedHabits);
  // ❌ If habit was in a group with group property, cleanup forgotten
};
```

**Problem:** Habit removed from main list but still referenced in groups array.

### AFTER: Remove from Habits + All Groups
```typescript
const handleDeleteHabit = (habitId: string) => {
  // Remove habit from main list
  const updatedHabits = habits.filter((h) => h.id !== habitId);
  setHabits(updatedHabits);

  // ✅ Remove from all groups
  const updatedGroups = groups.map((group) => ({
    ...group,
    habitIds: group.habitIds.filter((id) => id !== habitId)
  }));
  setGroups(updatedGroups);
};
```

**Result:** ✅ Clean removal, no orphaned IDs

---

## 5. GROUPED HABIT GRID RENDER

### BEFORE: Filter by String Group Name
```typescript
interface GroupedHabitGridProps {
  habits: Habit[];
  groups: string[];  // ❌ Just names
  groupModeActive?: boolean;
  onUpdateCell: (...) => void;
  // ...
}

function groupHabitsByName(habits: Habit[], allGroups: string[]): Array<[string, Habit[]]> {
  const groupMap = new Map<string, Habit[]>();
  
  // Initialize groups
  for (const groupName of allGroups) {
    groupMap.set(groupName, []);
  }

  // Add habits to groups
  for (const habit of habits) {
    const groupName = habit.group || 'default';  // ❌ Uses habit.group
    if (!groupMap.has(groupName)) {
      groupMap.set(groupName, []);
    }
    groupMap.get(groupName)!.push(habit);
  }
  
  return Array.from(groupMap.entries());
}

// Rendering
groupedHabits
  .filter(([groupName]) => groupName !== 'default')
  .map(([groupName, groupHabits]) => (
    <HabitGroup
      groupId={groupName}
      groupName={groupName}
      habits={groupHabits}
    />
  ));
```

### AFTER: Use HabitGroup Objects with ID References
```typescript
interface GroupedHabitGridProps {
  habits: Habit[];
  groups: HabitGroup[];  // ✅ Full objects
  habitsMap?: Record<string, Habit>;  // ✅ Optional performance optimization
  groupModeActive?: boolean;
  onUpdateCell: (...) => void;
  // ...
}

function getHabitsForGroup(
  group: HabitGroup,
  habitsMap: Record<string, Habit>
): Habit[] {
  return group.habitIds
    .map(id => habitsMap[id])
    .filter((habit): habit is Habit => habit !== undefined);
}

// In component
const hMap = habitsMap || habits.reduce((acc, h) => {
  acc[h.id] = h;
  return acc;
}, {});

// Rendering
groups.map((group) => {
  const groupHabits = getHabitsForGroup(group, hMap);
  
  return (
    <div key={group.id}>
      <HabitGroup
        groupId={group.id}  // ✅ Use group.id
        groupName={group.name}
        habits={groupHabits}
      />
    </div>
  );
});
```

**Result:** ✅ Cleaner, more flexible, supports multi-group habits

---

## 6. ADD HABIT MODAL INTERFACE

### BEFORE: Default Group Parameter
```typescript
interface AddHabitModalProps {
  onClose: () => void;
  onCreate: (
    name: string,
    type: 'boolean' | 'numeric',
    unit?: string,
    color?: string,
    group?: string  // ❌ Confusing: is this default or specific?
  ) => void;
  defaultGroup?: string;  // ❌ Unclear naming
}

export function AddHabitModal({
  onClose,
  onCreate,
  defaultGroup = 'default'
}: AddHabitModalProps) {
  const handleCreate = () => {
    onCreate(
      habitName,
      habitType,
      unit,
      color,
      defaultGroup  // ❌ Passed as 6th param
    );
  };
}
```

### AFTER: Explicit Group ID
```typescript
interface AddHabitModalProps {
  onClose: () => void;
  onCreate: (
    name: string,
    type: 'boolean' | 'numeric',
    unit?: string,
    color?: string,
    groupId?: string  // ✅ Clear: this is the group ID
  ) => void;
  groupId?: string | null;  // ✅ Null = no group, string = specific group
}

export function AddHabitModal({
  onClose,
  onCreate,
  groupId = null  // ✅ Null by default (independent)
}: AddHabitModalProps) {
  const handleCreate = () => {
    onCreate(
      habitName,
      habitType,
      unit,
      color,
      groupId || undefined  // ✅ Passed as intended groupId
    );
  };

  return (
    <h2>
      {groupId ? 'New Habit in Group' : 'New Habit'}
      {/* ✅ Clear messaging */}
    </h2>
  );
}
```

**Result:** ✅ Clear intent, explicit null handling

---

## 7. MIGRATION FUNCTION

### BEFORE: Auto-Assign Group
```typescript
export function migrateHabit(habit: any): Habit {
  if (habit.log && typeof habit.log === 'object') {
    return {
      ...habit,
      archived: habit.archived ?? false,
      group: habit.group ?? 'default'  // ❌ Always assign default
    };
  }

  // ... conversion logic ...

  return {
    // ...
    group: habit.group ?? 'default'  // ❌ Always assign default
  };
}
```

### AFTER: Preserve But Don't Auto-Assign
```typescript
export function migrateHabit(habit: any): Habit {
  if (habit.log && typeof habit.log === 'object') {
    return {
      ...habit,
      archived: habit.archived ?? false,
      // ✅ Keep existing group if present, otherwise undefined
      group: habit.group
    };
  }

  // ... conversion logic ...

  return {
    // ...
    // ✅ Keep existing group (for backwards compat), don't add if missing
    group: habit.group
  };
}
```

**Result:** ✅ Backwards compatible, no forced assignments

---

## 8. FILTER HABITS

### BEFORE: Filter by Group Property
```typescript
export interface HabitFilters {
  hideArchived?: boolean;
  hideEntered?: boolean;
  greyEntered?: boolean;
  group?: string;  // ❌ Habit group filter
}

export function filterHabits(habits: Habit[], filters: HabitFilters): Habit[] {
  return habits.filter((habit) => {
    if (filters.hideArchived && habit.archived) return false;
    if (filters.hideEntered) {
      const log = habit.log || {};
      const today = getTodayDateString();
      const todayValue = log[today];
      if (todayValue === 1 || todayValue === -1) return false;
    }
    if (filters.group && habit.group !== filters.group) {  // ❌ Filter by habit.group
      return false;
    }
    return true;
  });
}
```

### AFTER: Remove Group Filtering
```typescript
export interface HabitFilters {
  hideArchived?: boolean;
  hideEntered?: boolean;
  greyEntered?: boolean;
  // ✅ Removed group property - not applicable
}

export function filterHabits(habits: Habit[], filters: HabitFilters): Habit[] {
  return habits.filter((habit) => {
    if (filters.hideArchived && habit.archived) return false;
    if (filters.hideEntered) {
      const log = habit.log || {};
      const today = getTodayDateString();
      const todayValue = log[today];
      if (todayValue === 1 || todayValue === -1) return false;
    }
    // ✅ No group filtering here
    // Groups are managed at UI level via GroupedHabitGrid
    return true;
  });
}
```

**Result:** ✅ Cleaner separation of concerns

---

## 9. CREATE GROUP

### BEFORE: Just Add String
```typescript
const handleCreateGroup = (groupName: string) => {
  // ❌ Just add to array
  if (!groups.includes(groupName)) {
    setGroups([...groups, groupName]);
  }
};
```

### AFTER: Create HabitGroup Object
```typescript
const handleCreateGroup = (groupName: string) => {
  // ✅ Create proper group object
  if (!groups.find((g) => g.name === groupName)) {
    const newGroup: HabitGroup = {
      id: Date.now().toString(),  // Unique ID
      name: groupName,
      habitIds: [],  // Start empty
      sortOrder: groups.length,
      createdAt: Date.now()
    };
    setGroups([...groups, newGroup]);
  }
};
```

**Result:** ✅ Proper object structure, unique IDs

---

## 10. STATE INITIALIZATION

### BEFORE: Simple String Array
```typescript
const [groups, setGroups] = useState<string[]>(['default']);
const [addHabitToGroup, setAddHabitToGroup] = useState<string>('default');
```

### AFTER: Proper Objects
```typescript
const [groups, setGroups] = useState<HabitGroup[]>(getDefaultGroups());
const [selectedGroupForAdd, setSelectedGroupForAdd] = useState<string | null>(null);

// getDefaultGroups() returns:
// [{ id: 'default', name: 'Default', habitIds: [], sortOrder: 0, createdAt: timestamp }]
```

**Result:** ✅ Type-safe, structured data

---

## Summary Table

| Aspect | Before | After |
|--------|--------|-------|
| Arrow icon | Emoji `▶` | SVG chevron |
| Group model | String array | HabitGroup objects |
| Habit field | `habit.group: string` | (removed) |
| Group reference | Habit owns group | Group owns habitIds |
| Multi-group | Impossible (duplication) | Supported (references) |
| Auto-default | Always | Never |
| Data duplication | Yes (same habit in multiple groups = copies) | No (references only) |
| Single source of truth | No | Yes |
| Source of truth | Scattered (habits + groups) | Centralized (habits map) |
| Consistency | Manual | Automatic |
| Type safety | Weak | Strong |
| Safari fix | ❌ | ✅ |

