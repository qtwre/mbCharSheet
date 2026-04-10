# Restorable Journal Entries — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add "Restore" buttons to journal entries for destructive or hard-to-reverse actions, allowing players to undo mistakes without a full undo system.

**Architecture:** Extend the existing journal entry data model with an optional `restoreData` field. When a destructive action occurs, the journal entry captures a snapshot of the affected data. A [Restore] button on those entries reverses the action by re-applying the snapshot. Once restored, the entry is marked as restored (no double-restore).

**Tech Stack:** Same as existing — vanilla JS in the single `index.html` file.

**Spec:** `docs/superpowers/specs/2026-03-19-character-tracker-design.md`
**Base version:** v0.1 (tag `v0.1`)

---

## File Structure

```
mythicBastion/
  index.html          # All changes in this single file
```

---

## Data Model Changes

Extend the journal entry object:

```javascript
// Existing:
{ timestamp, season, age, text, auto }

// Extended:
{ timestamp, season, age, text, auto, restoreData, restored }
// restoreData: object | null — snapshot of data needed to undo the action
//   { type: string, data: object }
//   type is one of: 'mount', 'item_drop', 'inventory_drop', 'squire_knight',
//                    'contact_remove', 'feat_use', 'scar', 'new_season', 'new_age'
// restored: boolean — true after [Restore] has been clicked (prevents double-restore)
```

### Restore Types and Their Data

| Type | What `restoreData.data` contains | What Restore does |
|------|----------------------------------|-------------------|
| `mount` | Full mount object snapshot | Sets `currentChar.mount` back to snapshot |
| `item_drop` | `{ source: 'carried'/'weapons'/'armour', item: {...} }` | Re-adds item to the source array |
| `inventory_drop` | `{ item: 'string' }` | Re-adds string to `currentChar.inventory` |
| `squire_knight` | `{ squire: {...}, virtueRolls: {vig, cla, spi} }` | Restores squire object, subtracts virtue rolls from knight |
| `contact_remove` | `{ contact: {...} }` | Re-adds contact to `currentChar.contacts` |
| `feat_use` | `{ feat: 'smite'/'focus'/'deny', previousState: 'available' }` | Sets feat back to previous state |
| `scar` | `{ scar: {...}, gdMaxBefore, virtuesBefore: {vig,cla,spi}, doomBefore }` | Removes scar from list, restores GD max and virtue values, restores doom state |
| `new_season` | `{ virtuesBefore: {...}, seasonBefore, logEntry: {...} }` | Restores virtues to before, reverts season, removes log entry |
| `new_age` | `{ virtuesBefore: {...}, ageBefore, seasonBefore, gloryBefore, logEntry: {...} }` | Restores virtues, age, season, glory, removes log entry |

---

## Task 1: Extend Journal Rendering with Restore Buttons

**Files:**
- Modify: `index.html`

Find the journal rendering code (the `buildJournal` or `renderJournal` function). Modify it to:

- [ ] **Step 1: Update journal entry rendering**

For each journal entry, check if `entry.restoreData` exists and `entry.restored` is not true. If so, render a [Restore] button next to the delete button.

If `entry.restored === true`, show a muted "(Restored)" label instead of the button.

```javascript
// In the journal entry rendering loop:
if (entry.restoreData && !entry.restored) {
  // Add restore button
  html += `<button class="btn btn-small btn-gold" onclick="restoreFromJournal(${index})">Restore</button>`;
} else if (entry.restored) {
  html += '<span style="color:var(--success);font-size:0.75rem;font-style:italic"> (Restored)</span>';
}
```

- [ ] **Step 2: Implement the `restoreFromJournal(index)` function**

```javascript
function restoreFromJournal(index) {
  const entry = currentChar.journal[index];
  if (!entry || !entry.restoreData || entry.restored) return;

  const { type, data } = entry.restoreData;

  switch (type) {
    case 'mount':
      updateChar(c => { c.mount = data; });
      break;
    case 'item_drop':
      updateChar(c => {
        if (data.source === 'weapons') c.weapons.push(data.item);
        else if (data.source === 'armour') c.armour.push(data.item);
        else if (data.source === 'carried') c.carried.push(data.item);
      });
      break;
    case 'inventory_drop':
      updateChar(c => { c.inventory.push(data.item); });
      break;
    case 'squire_knight':
      updateChar(c => {
        c.squire = data.squire;
        c.virtues.vig.current -= data.virtueRolls.vig;
        c.virtues.vig.max -= data.virtueRolls.vig;
        c.virtues.cla.current -= data.virtueRolls.cla;
        c.virtues.cla.max -= data.virtueRolls.cla;
        c.virtues.spi.current -= data.virtueRolls.spi;
        c.virtues.spi.max -= data.virtueRolls.spi;
      });
      break;
    case 'contact_remove':
      updateChar(c => { c.contacts.push(data.contact); });
      break;
    case 'feat_use':
      updateChar(c => { c.feats[data.feat] = data.previousState; });
      break;
    case 'scar':
      updateChar(c => {
        c.scars = c.scars.filter(s =>
          !(s.roll === data.scar.roll && s.season === data.scar.season && s.name === data.scar.name));
        c.gd.max = data.gdMaxBefore;
        c.virtues.vig.current = data.virtuesBefore.vig.current;
        c.virtues.vig.max = data.virtuesBefore.vig.max;
        c.virtues.cla.current = data.virtuesBefore.cla.current;
        c.virtues.cla.max = data.virtuesBefore.cla.max;
        c.virtues.spi.current = data.virtuesBefore.spi.current;
        c.virtues.spi.max = data.virtuesBefore.spi.max;
        c.conditions.doomScar = data.doomBefore;
      });
      break;
    case 'new_season':
      updateChar(c => {
        c.virtues.vig = { ...data.virtuesBefore.vig };
        c.virtues.cla = { ...data.virtuesBefore.cla };
        c.virtues.spi = { ...data.virtuesBefore.spi };
        c.season = data.seasonBefore;
        c.seasonLog = c.seasonLog.filter(e =>
          !(e.season === data.logEntry.season && e.pursuit === data.logEntry.pursuit));
      });
      break;
    case 'new_age':
      updateChar(c => {
        c.virtues.vig = { ...data.virtuesBefore.vig };
        c.virtues.cla = { ...data.virtuesBefore.cla };
        c.virtues.spi = { ...data.virtuesBefore.spi };
        c.age = data.ageBefore;
        c.season = data.seasonBefore;
        c.glory = data.gloryBefore;
        c.seasonLog = c.seasonLog.filter(e =>
          !(e.season === data.logEntry.season && e.pursuit === data.logEntry.pursuit));
      });
      break;
  }

  // Mark as restored
  updateChar(c => { c.journal[index].restored = true; });
  addJournalEntry('Restored: ' + entry.text, true);
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add restore function and journal restore buttons"
```

---

## Task 2: Add restoreData to Destructive Actions

**Files:**
- Modify: `index.html`

Find each destructive action and modify it to include `restoreData` in its journal entry. Use `addJournalEntry` or modify the existing journal push to include the extra fields.

You will need a helper to create restorable journal entries:

```javascript
function addRestorableJournalEntry(text, restoreType, restoreSnapshot) {
  currentChar.journal.unshift({
    timestamp: new Date().toISOString(),
    season: currentChar.season,
    age: currentChar.age,
    text,
    auto: true,
    restoreData: { type: restoreType, data: restoreSnapshot },
    restored: false,
  });
  saveChar(currentChar);
  renderActiveTab();
}
```

- [ ] **Step 1: Mount removal**

Find the "Remove Mount" handler. Before setting mount to null, snapshot it:

```javascript
const mountSnapshot = JSON.parse(JSON.stringify(currentChar.mount));
updateChar(c => { c.mount = null; });
addRestorableJournalEntry('Removed mount: ' + mountSnapshot.name, 'mount', mountSnapshot);
```

- [ ] **Step 2: Item drop (carried, weapons, armour)**

Find the "Drop" button handlers for carried items, and the "Unequip" → drop flows. Before removing:

```javascript
// For weapons:
const weaponSnapshot = JSON.parse(JSON.stringify(currentChar.weapons[index]));
updateChar(c => { c.weapons.splice(index, 1); });
addRestorableJournalEntry('Dropped weapon: ' + weaponSnapshot.name, 'item_drop', { source: 'weapons', item: weaponSnapshot });

// For armour:
const armourSnapshot = JSON.parse(JSON.stringify(currentChar.armour[index]));
updateChar(c => { c.armour.splice(index, 1); });
addRestorableJournalEntry('Dropped armour: ' + armourSnapshot.name, 'item_drop', { source: 'armour', item: armourSnapshot });

// For carried:
const carriedSnapshot = JSON.parse(JSON.stringify(currentChar.carried[index]));
updateChar(c => { c.carried.splice(index, 1); });
addRestorableJournalEntry('Dropped item: ' + carriedSnapshot.name, 'item_drop', { source: 'carried', item: carriedSnapshot });
```

- [ ] **Step 3: Inventory item removal**

Find inventory item [x] remove handler:

```javascript
const itemText = currentChar.inventory[index];
updateChar(c => { c.inventory.splice(index, 1); });
addRestorableJournalEntry('Removed from inventory: ' + itemText, 'inventory_drop', { item: itemText });
```

- [ ] **Step 4: Knight squire**

Find the "Knight this Squire" handler. Before removing the squire, snapshot it and capture the virtue rolls:

```javascript
const squireSnapshot = JSON.parse(JSON.stringify(currentChar.squire));
// After rolling d6 for each virtue:
const rolls = { vig: vigRoll, cla: claRoll, spi: spiRoll };
// Apply rolls to knight...
updateChar(c => { c.squire = null; });
addRestorableJournalEntry('Knighted squire: ' + squireSnapshot.name + ' (VIG+' + rolls.vig + ', CLA+' + rolls.cla + ', SPI+' + rolls.spi + ')', 'squire_knight', { squire: squireSnapshot, virtueRolls: rolls });
```

- [ ] **Step 5: Contact removal**

Find contact remove handler:

```javascript
const contactSnapshot = JSON.parse(JSON.stringify(currentChar.contacts[index]));
updateChar(c => { c.contacts.splice(index, 1); });
addRestorableJournalEntry('Removed contact: ' + contactSnapshot.name, 'contact_remove', { contact: contactSnapshot });
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add restore snapshots to destructive actions (mount, items, squire, contacts)"
```

---

## Task 3: Add restoreData to State-Change Actions

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Feat usage**

Find the feat "Use" button handler. Before changing the feat state:

```javascript
const previousState = currentChar.feats[featKey];
updateChar(c => { c.feats[featKey] = 'pending'; });
addRestorableJournalEntry('Used feat: ' + FEATS[featKey].name, 'feat_use', { feat: featKey, previousState: previousState });
```

Also for "Failed" (feat → fatigued):
```javascript
addRestorableJournalEntry('Feat fatigued: ' + FEATS[featKey].name, 'feat_use', { feat: featKey, previousState: 'available' });
```

- [ ] **Step 2: Scar application**

Find the scar flow code where the scar is applied. Before applying any effects, snapshot the state:

```javascript
const scarRestoreData = {
  scar: { ...scarEntry, season: currentChar.season },
  gdMaxBefore: currentChar.gd.max,
  virtuesBefore: JSON.parse(JSON.stringify(currentChar.virtues)),
  doomBefore: currentChar.conditions.doomScar,
};
// ... apply scar effects ...
addRestorableJournalEntry('Scar: ' + scarEntry.name + ' — ' + scarEntry.effect, 'scar', scarRestoreData);
```

- [ ] **Step 3: New Season**

Find the "New Season" apply function. Before restoring virtues and advancing:

```javascript
const seasonRestoreData = {
  virtuesBefore: JSON.parse(JSON.stringify(currentChar.virtues)),
  seasonBefore: currentChar.season,
  logEntry: { season: newSeason, pursuit: selectedPursuit },
};
// ... apply season changes ...
addRestorableJournalEntry('New Season: ' + newSeason + ' — ' + selectedPursuit, 'new_season', seasonRestoreData);
```

- [ ] **Step 4: New Age**

Find the "New Age" apply function. Before applying:

```javascript
const ageRestoreData = {
  virtuesBefore: JSON.parse(JSON.stringify(currentChar.virtues)),
  ageBefore: currentChar.age,
  seasonBefore: currentChar.season,
  gloryBefore: currentChar.glory,
  logEntry: { season: newSeason, pursuit: selectedPursuit },
};
// ... apply age changes ...
addRestorableJournalEntry('New Age: ' + newAge + ' — ' + selectedPursuit, 'new_age', ageRestoreData);
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add restore snapshots to feat usage, scars, and season/age advances"
```

---

## Task 4: Testing and Edge Cases

**Files:**
- Modify: `index.html` (if fixes needed)

- [ ] **Step 1: Verify each restore type works**

Manual test checklist:
- Remove a mount → journal shows [Restore] → click Restore → mount is back
- Drop a weapon → Restore → weapon returns to equipped
- Drop a carried item → Restore → item returns to carried
- Remove inventory item → Restore → item returns
- Knight a squire → Restore → squire returns, virtue rolls subtracted
- Remove contact → Restore → contact returns
- Use a feat → Restore → feat back to available
- Trigger a scar → Restore → scar removed, stats reverted
- New Season → Restore → virtues/season reverted, log entry removed
- New Age → Restore → virtues/age/season/glory reverted

- [ ] **Step 2: Verify double-restore prevention**

- Click Restore on an entry → button disappears, "(Restored)" shows
- Reload page → "(Restored)" persists, no button

- [ ] **Step 3: Verify existing journal entries without restoreData still work**

- Import a v0.1 character JSON → old journal entries render without Restore buttons
- Manual journal entries never have Restore buttons

- [ ] **Step 4: Commit any fixes**

```bash
git add index.html
git commit -m "fix: edge cases in restorable journal entries"
```

- [ ] **Step 5: Tag v0.2**

```bash
git tag v0.2
```

---

## Task Summary

| Task | Description | Depends On |
|------|-------------|------------|
| 1 | Journal restore rendering + restore function | — |
| 2 | Add restoreData to destructive actions | 1 |
| 3 | Add restoreData to state-change actions | 1 |
| 4 | Testing and edge cases | 2, 3 |

Tasks 2 and 3 can run in parallel after Task 1.
