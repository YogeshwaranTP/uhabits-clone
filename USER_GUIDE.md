# Quick User Guide - New Features

## 🐛 Bug Fixed: Data Shifting Issue
The app now properly tracks each day as its own entry. When you log in on a new day, your previous data stays in place and a new blank day is created. No more data overriding!

## 📝 Edit Habit Names
1. **Long press** on a habit name (hold for ~600ms)
2. Click **"✏️ Edit name"**
3. Type the new name
4. Press **Enter** to save or **Escape** to cancel

## 🔍 Filter & Sort Your Habits

### Filters (Toggle on/off)
- **📦 Hide Archived**: Hide archived habits from view
- **👁️ Hide Entered**: Hide habits already completed/skipped today
- **🟦 Grey Entered**: Keep habits visible but dim them if done today

### Sorting Options
- **⋮↕️ Manual Sort** (default): Drag habits up/down to reorder
- **📝 Sort by Name**: A→Z alphabetical order
- **🎨 Sort by Color**: Group by color palette
- **⭐ Sort by Score**: Highest completion rate first
- **✓ Sort by Status**: Completed habits appear first

## 🎮 Long Press Menu
Hold down on a habit name for 600ms to open the context menu:

- **✏️ Edit name**: Rename the habit
- **⚡ Toggle today**: Quick mark as done/skip/undone
- **📂 Group management**: Assign to groups (morning, evening, fitness, etc.)
- **📦 Archive**: Hide habit without deleting
- **🗑️ Delete**: Permanently remove habit
- **✕ Close**: Close menu

## 📂 Organize with Groups
1. Long press on a habit → **📂 Group: [current]**
2. Select a group:
   - default
   - morning
   - evening
   - fitness
   - health
   - learning
   - work
3. Use **🔍 Filters** → **📂 [Group]** to filter by group

## 💬 Daily Prompt
At the top of the grid, see "Did you <habit>?" for incomplete habits:
- Click **✓ Yes** to mark as done
- Click **✕** to dismiss
- Automatically rotates through incomplete habits every 5 seconds
- Disappears when all habits are done

## 🎯 Manual Sorting
1. Select **⋮↕️ Manual Sort** from sort menu
2. **Drag and drop** habit rows to reorder
3. Your custom order is saved automatically

## 📊 Controls Bar
Below the header, see:
- **SORT & FILTER** label
- **Sort button**: Select sorting method
- **Filters button**: Toggle various filters
- **Group button**: Filter by group
- **Stats**: Shows 📦 (archived count) and ✓ (completed today count)

## 💬 Vibration Feedback
Feel haptic feedback on your device:
- **✓ Done**: Short vibration
- **✕ Skip**: Double vibration pattern
- **Reset**: Longer vibration
- Long press: Double-tap pattern

## 💡 Pro Tips
1. **Quickly complete habits**: Use the daily prompt's "Yes" button
2. **Organize your morning**: Sort by "morning" group
3. **Focus mode**: Hide archived and entered habits
4. **Track progress**: Sort by score to see your best habits
5. **Custom order**: Use manual sort to prioritize habits visually

## ⚙️ Technical Details
- **No concept of "last 7 days"** in backend - purely UI display
- **Daily data**: New blank entry added automatically each day
- **Date binding**: Each cell binds to specific dates, not history indices
- **Archive**: Hides habits but keeps all historical data
- **Groups**: No hidden meaning - purely organizational

## 🔄 How the Date Fix Works
Previously: Data was stored in a fixed-size array, causing issues when days shifted
Now: 
- Each habit tracks `lastRecordedDate`
- New day = new entry added to history
- Your data grows naturally over time
- No more overwriting issues!
