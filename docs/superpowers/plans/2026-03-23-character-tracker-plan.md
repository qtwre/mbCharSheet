# Mythic Bastionland Character Tracker — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file HTML webapp for tracking a Mythic Bastionland player character — stats, gear, and combat mechanics with a parchment/medieval theme.

**Architecture:** Single HTML file containing embedded CSS, game databases as JS objects, a state management layer using localStorage, and vanilla JS rendering functions. Three tabs (Character Sheet, Equipment, Battle) share a single character state object. All mutations go through a central `updateChar()` function that auto-saves and re-renders the active tab only (not all tabs). Modal dialogs (scar flow, character creation) use a callback pattern for async flows. Character objects include a `version` field for future data migrations.

**Tech Stack:** HTML5, vanilla CSS3, vanilla JavaScript (ES6+), localStorage. No build step, no dependencies, no frameworks.

**Spec:** `docs/superpowers/specs/2026-03-19-character-tracker-design.md`

**Testing:** Since this is a single HTML file with no test framework, testing is manual — open the file in a browser and verify behavior. Each task includes specific manual test steps. A future version could add a test harness, but for v1 manual testing is appropriate for the scope.

---

## File Structure

```
mythicBastion/
  index.html          # The entire app — HTML structure, CSS, JS, game databases
```

Single file. All CSS in a `<style>` block, all JS in a `<script>` block. Game databases (knights, equipment, scars) are JS const objects at the top of the script. The file is organized into clearly commented sections:

```
<!-- HTML: Shell, tabs, panels -->
<style>  /* CSS: Parchment theme, layout, responsive */ </style>
<script>
  // === GAME DATABASES === (knights, equipment, scars, beasts)
  // === STATE MANAGEMENT === (load, save, update, create, delete)
  // === RENDERING === (per-panel render functions)
  // === EVENT HANDLERS === (user interactions, damage calc, scar flow)
  // === INIT === (boot, hydrate, first render)
</script>
```

---

## Task 1: HTML Shell + Parchment Theme + Tab Navigation

**Files:**
- Create: `index.html`

This task builds the visual foundation — the themed shell that everything else renders into.

- [ ] **Step 1: Create `index.html` with HTML structure**

The HTML contains:
- Header area (character name, knight type, glory, age, season, character switcher)
- Oath banner
- Three tab buttons (Character Sheet, Equipment, Battle)
- Three tab content divs (initially empty placeholder text)
- A modal container for dialogs (character creation, scar flow, confirmations)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Mythic Bastionland — Character Tracker</title>
</head>
<body>
  <div id="app">
    <header id="header">
      <div id="char-switcher"><!-- character dropdown, new/delete --></div>
      <div id="char-header"><!-- name, knight type, glory, rank, age, season --></div>
    </header>
    <div id="oath" class="oath-banner">Seek the Myths. Honour the Seers. Protect the Realm.</div>
    <nav id="tabs">
      <button class="tab active" data-tab="sheet">Character Sheet</button>
      <button class="tab" data-tab="equipment">Equipment</button>
      <button class="tab" data-tab="battle">Battle</button>
    </nav>
    <main>
      <section id="tab-sheet" class="tab-content active"><!-- Character Sheet --></section>
      <section id="tab-equipment" class="tab-content"><!-- Equipment --></section>
      <section id="tab-battle" class="tab-content"><!-- Battle --></section>
    </main>
    <div id="modal-overlay" class="modal-overlay hidden">
      <div id="modal" class="modal"></div>
    </div>
  </div>
</body>
</html>
```

- [ ] **Step 2: Add the parchment/medieval CSS theme**

All CSS goes in a single `<style>` block. Key design tokens:

```css
:root {
  --parchment-bg: #f4e4c1;
  --parchment-dark: #e8d5a3;
  --ink: #2c1810;
  --ink-light: #5c4033;
  --gold: #b8860b;
  --gold-light: #d4a843;
  --danger: #8b0000;
  --warning: #cc7722;
  --success: #2d5a27;
  --panel-bg: rgba(255, 248, 230, 0.7);
  --panel-border: #c4a56e;
  --font-serif: Georgia, 'Times New Roman', serif;
  --font-body: 'Palatino Linotype', 'Book Antiqua', Palatino, serif;
  --radius: 4px;
}
```

Parchment background via CSS gradient (no external images):
```css
body {
  background:
    linear-gradient(135deg, rgba(139,119,80,0.08) 25%, transparent 25%),
    linear-gradient(225deg, rgba(139,119,80,0.05) 25%, transparent 25%),
    linear-gradient(315deg, rgba(139,119,80,0.08) 25%, transparent 25%),
    linear-gradient(45deg, rgba(139,119,80,0.05) 25%, transparent 25%),
    var(--parchment-bg);
  color: var(--ink);
  font-family: var(--font-body);
}
```

Style the following:
- `.oath-banner` — centered italic text, decorative top/bottom borders using CSS border-image or repeated `―` characters
- `.tab` buttons — styled as ribbon bookmarks with active state using gold accent, bottom border trick to connect to content
- `.tab-content` — panels with subtle box-shadow and border
- `.panel` — reusable card component with header and body, worn-edge border
- `.btn` — embossed leather look via gradient + inset shadow
- `.btn-danger` — dark red variant for destructive actions
- `.stat-box` — compact display for virtue/GD values with +/- buttons
- `.condition-tag` — small banner/shield shape for conditions
- `.modal-overlay` / `.modal` — centered overlay with parchment-styled dialog
- Mobile responsive: single column below 768px, tabs become full-width stacked buttons

- [ ] **Step 3: Add tab switching JavaScript**

Minimal JS at the bottom of the file to handle tab clicks:

```javascript
document.querySelectorAll('.tab').forEach(btn => {
  btn.addEventListener('click', () => {
    document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
    document.querySelectorAll('.tab-content').forEach(t => t.classList.remove('active'));
    btn.classList.add('active');
    document.getElementById('tab-' + btn.dataset.tab).classList.add('active');
    renderActiveTab(); // re-render the newly visible tab with fresh state
  });
});
```

- [ ] **Step 4: Verify in browser**

Open `index.html` in a browser. Verify:
- Parchment background renders
- Oath banner displays
- Three tabs switch content areas
- Responsive: narrow the window, tabs should stack
- No console errors

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: scaffold HTML shell with parchment theme and tab navigation"
```

---

## Task 2: Game Databases (Knights, Equipment, Scars, Beasts)

**Files:**
- Modify: `index.html` (add JS const objects in the `<script>` block)

This is a data entry task. Extract all game data from the PDF and encode as JS objects. This is the largest single task by volume but is straightforward copy work.

- [ ] **Step 1: Add the equipment database**

At the top of the `<script>` block, add:

```javascript
// === GAME DATABASES ===

const EQUIPMENT_DB = {
  weapons: [
    // Common
    { id: 'dagger', name: 'Dagger', dice: 'd6', tags: [], rarity: 'common' },
    { id: 'club', name: 'Club', dice: 'd6', tags: [], rarity: 'common' },
    { id: 'handaxe', name: 'Handaxe', dice: 'd6', tags: [], rarity: 'common' },
    { id: 'hefty_tools', name: 'Hefty Tools (pitchfork, hatchet)', dice: 'd6', tags: ['hefty'], rarity: 'common' },
    { id: 'long_tools', name: 'Long Tools (staff, logging axe, pick)', dice: 'd8', tags: ['long'], rarity: 'common' },
    { id: 'spear', name: 'Spear', dice: 'd8', tags: ['hefty'], rarity: 'common' },
    { id: 'mace', name: 'Mace', dice: 'd8', tags: ['hefty'], rarity: 'common' },
    { id: 'axe', name: 'Axe', dice: 'd8', tags: ['hefty'], rarity: 'common' },
    { id: 'poleaxe', name: 'Poleaxe', dice: 'd10', tags: ['long'], rarity: 'common' },
    { id: 'billhook', name: 'Billhook', dice: 'd10', tags: ['long'], rarity: 'common' },
    { id: 'sling', name: 'Sling', dice: 'd4', tags: ['hefty'], rarity: 'common' },
    { id: 'javelin', name: 'Javelin', dice: 'd6', tags: ['hefty'], rarity: 'common' },
    { id: 'shortbow', name: 'Shortbow', dice: 'd6', tags: ['long'], rarity: 'common' },
    // Uncommon
    { id: 'shortsword', name: 'Shortsword', dice: '2d6', tags: [], rarity: 'uncommon' },
    { id: 'lance', name: 'Lance', dice: 'd10', tags: ['long'], rarity: 'uncommon', notes: 'Counts as hefty if mounted' },
    { id: 'brutal_weapon', name: 'Brutal Weapon (greataxe, maul)', dice: '2d10', tags: ['slow'], rarity: 'uncommon' },
    { id: 'longbow', name: 'Longbow', dice: 'd8', tags: ['slow'], rarity: 'uncommon' },
    // Rare
    { id: 'longsword', name: 'Longsword', dice: '2d8', tags: ['hefty'], rarity: 'rare' },
    { id: 'greatsword', name: 'Greatsword', dice: '2d10', tags: ['long'], rarity: 'rare' },
    { id: 'curvebow', name: 'Curvebow', dice: '2d6', tags: ['long'], rarity: 'rare' },
    { id: 'crossbow', name: 'Crossbow', dice: '2d8', tags: ['slow'], rarity: 'rare' },
  ],
  armour: [
    { id: 'shield', name: 'Shield', type: 'shield', score: 1, dice: 'd4', rarity: 'common' },
    { id: 'coat', name: 'Coat (mail, gambeson)', type: 'coat', score: 1, rarity: 'uncommon' },
    { id: 'helm', name: 'Helm', type: 'helm', score: 1, rarity: 'uncommon' },
    { id: 'plates', name: 'Plates (cuirass, brigandine)', type: 'plates', score: 1, rarity: 'rare' },
  ],
  beasts: [
    { id: 'hound', name: 'Hound', vig: 5, cla: 10, spi: 5, gd: 4, attack: 'd6 bite', rarity: 'common' },
    { id: 'pony', name: 'Pony', vig: 7, cla: 7, spi: 5, gd: 2, rarity: 'common' },
    { id: 'mule', name: 'Mule', vig: 10, cla: 5, spi: 5, gd: 1, rarity: 'common' },
    { id: 'ox', name: 'Ox', vig: 15, cla: 5, spi: 5, gd: 3, rarity: 'uncommon' },
    { id: 'hawk', name: 'Hawk', vig: 5, cla: 15, spi: 5, gd: 4, attack: 'd4 talons', rarity: 'uncommon' },
    { id: 'riding_steed', name: 'Riding Steed', vig: 10, cla: 10, spi: 5, gd: 3, rarity: 'uncommon' },
    { id: 'heavy_steed', name: 'Heavy Steed', vig: 15, cla: 5, spi: 5, gd: 2, rarity: 'uncommon' },
    { id: 'charger', name: 'Charger', vig: 10, cla: 5, spi: 5, gd: 5, trample: 'd8', rarity: 'rare' },
  ]
};
```

- [ ] **Step 2: Add the scar table**

```javascript
const SCAR_TABLE = [
  { roll: 1, name: 'Distress', description: 'A lucky escape', effect: 'Lose d6 SPI' },
  { roll: 2, name: 'Disfigurement', description: 'A permanent mark (d6: 1-Eye, 2-Cheek, 3-Neck, 4-Torso, 5-Nose, 6-Jaw)', effect: 'If max GD is 2 or less, increase by d6', gdThreshold: 2 },
  { roll: 3, name: 'Smash', description: 'A sudden spray of blood', effect: 'Lose d6 VIG' },
  { roll: 4, name: 'Stun', description: 'Pain drowns the senses', effect: 'Lose d6 CLA. If max GD is 4 or less, increase by d6', gdThreshold: 4 },
  { roll: 5, name: 'Rupture', description: 'Innards pierced and compressed', effect: 'Lose 2d6 VIG' },
  { roll: 6, name: 'Gouge', description: 'Flesh torn from bone', effect: 'If max GD is 6 or less, increase by d6', gdThreshold: 6 },
  { roll: 7, name: 'Concussion', description: 'A heavy blow numbs the mind', effect: 'Lose 2d6 CLA' },
  { roll: 8, name: 'Tear', description: 'Something taken in violent struggle (d6: 1-Nose, 2-Ear, 3-Finger, 4-Thumb, 5-Eye, 6-Chunk of Scalp)', effect: 'If max GD is 8 or less, increase by d6', gdThreshold: 8 },
  { roll: 9, name: 'Agony', description: 'With a crack, a torturous break', effect: 'Lose 2d6 SPI' },
  { roll: 10, name: 'Mutilation', description: 'A limb rendered lost or useless (d6: 1-2 Leg, 3-4 Shield Arm, 5-6 Sword Arm). Prosthetic by next Season.', effect: 'If max GD is 10 or less, increase by d6', gdThreshold: 10 },
  { roll: 11, name: 'Doom', description: 'A cheated death haunts you', effect: 'If you take a Mortal Wound this Season, you are Slain instead', isDoom: true },
  { roll: 12, name: 'Humiliation', description: 'A most dolorous stroke', effect: 'When you achieve revenge, if max GD is 12 or less, increase by d6', gdThreshold: 12 },
];
```

- [ ] **Step 3: Add the rank thresholds and feat definitions**

```javascript
const RANK_THRESHOLDS = [
  { glory: 0, rank: 'Knight-Errant', worthy: 'Worthy of leading a Warband' },
  { glory: 3, rank: 'Knight-Gallant', worthy: 'Worthy of a seat in Council or Court' },
  { glory: 6, rank: 'Knight-Tenant', worthy: 'Worthy of ruling a Holding' },
  { glory: 9, rank: 'Knight-Dominant', worthy: 'Worthy of ruling a Seat of Power' },
  { glory: 12, rank: 'Knight-Radiant', worthy: 'Worthy of the City Quest' },
];

const FEATS = {
  smite: {
    name: 'Smite',
    description: 'Release your righteous fury. Use before rolling a melee Attack. The Attack gains either +d12 or Blast.',
    save: 'VIG',
  },
  focus: {
    name: 'Focus',
    description: 'Create an opening to exploit. Use after rolling an Attack. Perform a Gambit without using a die.',
    save: 'CLA',
  },
  deny: {
    name: 'Deny',
    description: 'Rebuff an attack before it lands. Use after an Attack roll against you or an ally within arm\'s reach. Discard one Attack die from the roll.',
    save: 'SPI',
  },
};

const GAMBITS = [
  { name: 'Bolster', effect: 'Bolster the Attack for +1 total Damage', strong: false },
  { name: 'Move', effect: 'Move after the Attack, even if you already moved or are unable to move', strong: false },
  { name: 'Repel', effect: 'Repel a foe away from you', strong: true },
  { name: 'Stop', effect: 'Stop a foe from moving next turn', strong: true },
  { name: 'Impair', effect: 'Impair a weapon on their next turn', strong: true },
  { name: 'Trap', effect: 'Trap a shield until your next turn', strong: true },
  { name: 'Dismount', effect: 'Dismount a foe', strong: true },
  { name: 'Other', effect: 'Other effect of a similar level of impact', strong: true },
];
```

- [ ] **Step 4: Add the knight database**

This is the largest data block. Extract all 72 knights from the PDF. Each entry follows this structure:

```javascript
const KNIGHT_DB = [
  {
    id: 'true_knight',
    number: '1-1',
    name: 'The True Knight',
    flavor: 'A heart of gold...',
    property: [
      { type: 'weapon', name: 'Longsword', dice: '2d8', tags: ['hefty'] },
      { type: 'armour', name: 'Mail', armourType: 'coat', score: 1 },
      // ... other starting items
    ],
    ability: { name: 'Ability Name', description: 'What it does' },
    passion: { name: 'Passion Name', description: 'How to restore SPI' },
    seer: { name: 'The X Seer', description: 'What they look like and want' },
  },
  // ... 71 more knights
];
```

Extract each knight's data from the PDF pages. Knights are organized in 6 groups of 12 (numbered 1-1 through 6-12). Each knight page has: Property, Ability, Passion, a backstory table, and a Knighting Seer.

**Note to implementer:** This is a large data entry task. Extract knight data page by page from the PDF using `pdftotext`. Focus on the mechanically relevant fields (property, ability, passion, seer). Flavor text and backstory tables are nice-to-have but lower priority. If time is short, start with 12-24 knights and add the rest incrementally.

- [ ] **Step 5: Verify databases load without errors**

Open `index.html`, open browser console, type:
```
KNIGHT_DB.length    // should be 72 (or however many extracted)
EQUIPMENT_DB.weapons.length  // should be 21
SCAR_TABLE.length   // should be 12
```

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add game databases (equipment, scars, ranks, feats, knights)"
```

---

## Task 3: State Management (localStorage CRUD + Auto-Save)

**Files:**
- Modify: `index.html` (add state management JS)

The central nervous system of the app. All character state flows through these functions.

- [ ] **Step 1: Add the default character factory and state management functions**

```javascript
// === STATE MANAGEMENT ===

let currentCharId = null;
let currentChar = null;

function generateId() {
  return Date.now().toString(36) + Math.random().toString(36).substr(2, 5);
}

function getRank(glory) {
  for (let i = RANK_THRESHOLDS.length - 1; i >= 0; i--) {
    if (glory >= RANK_THRESHOLDS[i].glory) return RANK_THRESHOLDS[i];
  }
  return RANK_THRESHOLDS[0];
}

function createDefaultChar(knightId, name) {
  const knight = KNIGHT_DB.find(k => k.id === knightId);
  const id = generateId();
  return {
    version: 1,              // for future data migrations
    id,
    name: name || 'New Knight',
    knightType: knightId,
    glory: 0,
    age: 'Young',
    season: 'Spring',
    virtues: {
      vig: { current: 10, max: 10 },
      cla: { current: 10, max: 10 },
      spi: { current: 10, max: 10 },
    },
    gd: { current: 1, max: 1 },
    ability: knight ? { ...knight.ability } : { name: '', description: '' },
    passion: knight ? { ...knight.passion } : { name: '', description: '' },
    armour: [],
    weapons: [],
    carried: [],
    remedies: { sustenance: 0, stimulant: 0, sacrament: 0 },
    inventory: [],
    mount: null,
    squire: null,
    scars: [],
    feats: { smite: 'available', focus: 'available', deny: 'available' },
    conditions: { doomScar: false, wounded: false, mortalWound: false, customConditions: [] },
    contacts: [],
    seasonLog: [],
    journal: [],
  };
}

function saveChar(char) {
  localStorage.setItem('mythicBastion_char_' + char.id, JSON.stringify(char));
}

function loadChar(id) {
  const raw = localStorage.getItem('mythicBastion_char_' + id);
  return raw ? JSON.parse(raw) : null;
}

function deleteChar(id) {
  localStorage.removeItem('mythicBastion_char_' + id);
  const ids = getCharIds().filter(cid => cid !== id);
  localStorage.setItem('mythicBastion_charIds', JSON.stringify(ids));
}

function getCharIds() {
  const raw = localStorage.getItem('mythicBastion_charIds');
  return raw ? JSON.parse(raw) : [];
}

function registerCharId(id) {
  const ids = getCharIds();
  if (!ids.includes(id)) {
    ids.push(id);
    localStorage.setItem('mythicBastion_charIds', JSON.stringify(ids));
  }
}

function updateChar(mutator) {
  mutator(currentChar);
  saveChar(currentChar);
  render();
}

function addJournalEntry(text, auto = false) {
  currentChar.journal.unshift({
    timestamp: new Date().toISOString(),
    season: currentChar.season,
    age: currentChar.age,
    text,
    auto,
  });
  saveChar(currentChar);
  renderActiveTab(); // ensure journal entry is visible immediately
}

function switchToChar(id) {
  currentCharId = id;
  currentChar = loadChar(id);
  localStorage.setItem('mythicBastion_currentCharId', id);
  render();
}
```

- [ ] **Step 2: Add the dice rolling utility**

```javascript
function rollDie(sides) {
  return Math.floor(Math.random() * sides) + 1;
}

function parseDice(diceStr) {
  // Parse strings like "d6", "2d8", "d10"
  // Returns { count, sides } or null if invalid
  const match = diceStr.match(/(\d*)d(\d+)/);
  if (!match) return null;
  const count = parseInt(match[1] || '1');
  const sides = parseInt(match[2]);
  return { count, sides };
}

function rollDice(diceStr) {
  const { count, sides } = parseDice(diceStr);
  const results = [];
  for (let i = 0; i < count; i++) results.push(rollDie(sides));
  return results;
}
```

- [ ] **Step 3: Add the init/boot function**

```javascript
// === INIT ===

function init() {
  const lastId = localStorage.getItem('mythicBastion_currentCharId');
  const ids = getCharIds();
  if (lastId && ids.includes(lastId)) {
    switchToChar(lastId);
  } else if (ids.length > 0) {
    switchToChar(ids[0]);
  } else {
    showNewCharacterModal();
  }
}

function getActiveTab() {
  const active = document.querySelector('.tab.active');
  return active ? active.dataset.tab : 'sheet';
}

function render() {
  renderHeader();
  renderActiveTab();
}

function renderActiveTab() {
  // Only re-render the visible tab to avoid destroying in-progress form inputs on other tabs
  const tab = getActiveTab();
  if (tab === 'sheet') renderCharSheet();
  else if (tab === 'equipment') renderEquipment();
  else if (tab === 'battle') renderBattle();
}

// Stub render functions — will be implemented in later tasks
function renderHeader() {}
function renderCharSheet() {}
function renderEquipment() {}
function renderBattle() {}

document.addEventListener('DOMContentLoaded', init);
```

- [ ] **Step 4: Verify state management in browser console**

Open `index.html`, open console:
```javascript
// Create a test character
const testChar = createDefaultChar(KNIGHT_DB[0]?.id || 'test', 'Sir Test');
registerCharId(testChar.id);
saveChar(testChar);
switchToChar(testChar.id);
console.log(currentChar.name); // "Sir Test"

// Modify and verify persistence
updateChar(c => { c.glory = 3; });
console.log(getRank(currentChar.glory).rank); // "Knight-Gallant"

// Reload page, verify character persists
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add state management with localStorage CRUD and auto-save"
```

---

## Task 4: Character Creation Flow + Character Switcher

**Files:**
- Modify: `index.html` (implement `showNewCharacterModal()`, `renderHeader()`, delete flow)

- [ ] **Step 1: Implement the character creation modal**

```javascript
function showNewCharacterModal() {
  const modal = document.getElementById('modal');
  // Build a form with:
  // 1. Knight dropdown (grouped by number — "1-1: The True Knight", etc.)
  // 2. Character name text input
  // 3. VIG, CLA, SPI number inputs (min 0, max 19)
  // 4. GD number input (min 0, max 19)
  // 5. Optional squire checkbox + name input
  // 6. Optional mount section (name, description, VIG/CLA/SPI/GD, trample die, traits)
  // 7. "Create Knight" button
  //
  // On knight selection: preview the Ability, Passion, and starting Property below the dropdown
  // On create: build character object, populate property from knight DB, save, switch to it
  // Note: when populating mount from beast DB, convert flat stats to {current, max} pairs:
  //   e.g. beast.vig: 10 → mount.vig: { current: 10, max: 10 }
  document.getElementById('modal-overlay').classList.remove('hidden');
}

function hideModal() {
  document.getElementById('modal-overlay').classList.add('hidden');
}
```

The creation form should:
- Pre-fill weapons/armour/inventory from the selected knight's `property` array
- Let the user manually enter rolled Virtues and GD
- Cap values at 0-19
- If squire checked: add squire with default pony mount, 2d6 Virtues (user enters), 1 GD

- [ ] **Step 2: Implement the header render with character switcher**

```javascript
function renderHeader() {
  const ids = getCharIds();
  const rank = getRank(currentChar.glory);
  // Render into #char-switcher:
  //   <select> with all character names, current selected
  //   "New" button → showNewCharacterModal()
  //   "Delete" button → confirm dialog → deleteChar()
  //
  // Render into #char-header:
  //   Character name (contenteditable span, blur saves)
  //   Knight type label
  //   Glory: number with +/- buttons (auto-update rank display)
  //   Rank badge
  //   Age selector: <select> Young/Mature/Old
  //   Season selector: <select> Spring/Harvest/Winter
}
```

- [ ] **Step 3: Wire up character switching**

The character dropdown `<select>` onChange calls `switchToChar(selectedId)`. New/Delete buttons wire to their respective flows.

Delete flow: confirmation dialog ("Delete Sir Batman? This cannot be undone."), then `deleteChar()`, switch to another character or show creation modal if none left.

- [ ] **Step 4: Verify in browser**

- Create a new character — select a knight, enter stats, verify it saves
- Create a second character — switch between them via dropdown
- Delete one — verify it's gone, other remains
- Reload page — verify last active character loads
- Verify Glory +/- updates rank display
- Verify Age/Season selectors save changes

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add character creation modal and header with character switcher"
```

---

## Task 5: Character Sheet Tab — Virtues, Guard, Conditions

**Files:**
- Modify: `index.html` (implement `renderCharSheet()` — core stats section)

- [ ] **Step 1: Implement Virtues panel**

```javascript
function renderCharSheet() {
  const el = document.getElementById('tab-sheet');
  el.innerHTML = ''; // Clear and rebuild

  // Virtues panel
  el.appendChild(renderVirtuesPanel());
  // ... more panels added in subsequent tasks
}

function renderVirtuesPanel() {
  // Create a .panel with three .stat-box rows:
  // VIG: [current] / [max]  [-] [+]  [Restore]
  // CLA: [current] / [max]  [-] [+]  [Restore]
  // SPI: [current] / [max]  [-] [+]  [Restore]
  //
  // +/- buttons call updateChar() to adjust current value (clamped 0 to max)
  // Restore button: if remedy available, prompt "Use Sustenance? (X remaining)" → decrement remedy, set current = max
  //   If no remedy: still allow restore (for between-seasons), just set current = max
  // When VIG is restored to max: also clear conditions.wounded and conditions.mortalWound
  // Color-coded warnings when current = 0:
  //   VIG 0: red banner "EXHAUSTED — Cannot attack if you moved this turn"
  //   CLA 0: red banner "EXPOSED — Treated as 0 GD"
  //   SPI 0: red banner "IMPAIRED — Attacks roll d4 only"
}
```

- [ ] **Step 2: Implement Guard panel with scar flow**

```javascript
function renderGuardPanel() {
  // GD: [current] / [max]  [-] [+]  [Restore GD]
  //
  // Restore GD button:
  //   Sets current = max
  //   Clears all fatigued feats to "available"
  //   Does NOT clear wounded or mortalWound
  //   Adds journal entry "Guard restored (moment of peace)"
  //
  // When - button or applyDamage reduces GD to 0:
  //   Trigger scarFlow()
}

function scarFlow(excessDamage, onComplete) {
  // Show modal — this is async (user must select a die).
  // The onComplete callback fires AFTER the scar is resolved,
  // passing any excess damage to VIG.
  //
  // Modal content:
  // "GD reduced to 0 — Scar earned!"
  // "Select the attack die that caused this:" [d4] [d6] [d8] [d10] [d12] buttons
  //
  // On die selection:
  //   Roll the selected die
  //   Look up SCAR_TABLE[result - 1]
  //   Display: scar name, description, effect
  //   Apply mechanical effects:
  //     - Virtue loss scars: reduce the relevant virtue (roll the dice specified in effect)
  //     - GD increase scars: if max GD <= threshold, roll d6 and add to max GD
  //     - Doom scar (#11): set conditions.doomScar = true
  //   Record to character.scars[]
  //   Add journal entry (auto)
  //   "OK" button closes modal, then:
  //     if (onComplete) onComplete(excessDamage);
  //     // This triggers applyVigDamage(excessDamage) from the damage calculator
}
```

- [ ] **Step 3: Implement Conditions banner**

```javascript
function renderConditions() {
  // Toggleable tags for stored conditions:
  //   Doom Scar (if conditions.doomScar), Wounded, Mortal Wound
  //   Custom conditions with add/remove
  //
  // Auto-derived (read-only display, computed each render):
  //   "EXHAUSTED" if vig.current === 0
  //   "EXPOSED" if cla.current === 0
  //   "CHARACTER IMPAIRED" if spi.current === 0
  //   "FATIGUED" if any feat is "fatigued"
  //
  // Each toggleable condition has a click handler to toggle on/off via updateChar()
  // Custom conditions: small "+" button opens inline input to add a new one, "x" to remove
}
```

- [ ] **Step 4: Verify in browser**

- Adjust VIG/CLA/SPI with +/- buttons — verify current stays within 0-max range
- Set VIG to 0 — verify Exhausted warning appears
- Click Restore on VIG — verify it returns to max and prompts about remedy if available
- Reduce GD to 0 — verify scar flow modal appears
- Select a die in scar flow — verify scar is rolled, displayed, recorded
- Toggle Wounded condition — verify it persists on page reload
- Add a custom condition — verify it appears and can be removed

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add virtues panel, guard panel with scar flow, and conditions banner"
```

---

## Task 6: Character Sheet Tab — Ability, Passion, Mount, Squire

**Files:**
- Modify: `index.html` (add panels to `renderCharSheet()`)

- [ ] **Step 1: Implement Ability & Passion cards**

```javascript
function renderAbilityPassion() {
  // Two side-by-side cards (or stacked on mobile):
  //
  // ABILITY card:
  //   Name (bold), description text
  //   Read-only (sourced from knight DB, but stored on character for custom edits)
  //
  // PASSION card:
  //   Name (bold), description text
  //   "Indulge Passion" button → sets spi.current = spi.max, adds journal entry
}
```

- [ ] **Step 2: Implement Mount panel**

```javascript
function renderMountPanel() {
  // Collapsible panel, only shown if currentChar.mount !== null
  // Header: mount name + collapse toggle
  // Body:
  //   Description, traits list
  //   VIG/CLA/SPI/GD with +/- buttons (same stat-box component as virtues)
  //   Trample die display (if any)
  //   "Remove Mount" button (with confirmation)
  //   "Add Mount" button shown if mount is null — opens inline form or modal
  //     with name, description, VIG/CLA/SPI/GD inputs, trample die dropdown, traits text
}
```

- [ ] **Step 3: Implement Squire panel**

```javascript
function renderSquirePanel() {
  // Collapsible panel, only shown if currentChar.squire !== null
  // Header: squire name + collapse toggle
  // Body:
  //   Description
  //   VIG/CLA/SPI (rolled 2d6 each, entered by user) and GD (starts at 1) with +/- buttons
  //   Equipment list (editable)
  //   Squire's pony stats (mini stat block)
  //   "Knight this Squire" button:
  //     Rolls d6 for each Virtue, shows results
  //     Adds results to both current and max
  //     Archives squire info to journal: "Squire [name] knighted! VIG+X, CLA+Y, SPI+Z"
  //     Sets currentChar.squire = null
  //     Re-renders
  //
  // "Add Squire" button shown if squire is null — only for small companies (2 knights or fewer)
  //   Opens form: name, description, VIG/CLA/SPI inputs, equipment
}
```

- [ ] **Step 4: Verify in browser**

- Ability and Passion display correctly from knight data
- "Indulge Passion" button restores SPI to max
- Add a mount — verify stats display with +/- buttons
- Remove mount — verify panel hides
- Add a squire — verify stats and equipment display
- Knight the squire — verify dice rolls shown, virtues updated, squire removed, journal entry created

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add ability/passion cards, mount panel, and squire panel"
```

---

## Task 7: Character Sheet Tab — Remedies, Contacts, Season Log, Journal

**Files:**
- Modify: `index.html` (add remaining panels to `renderCharSheet()`)

- [ ] **Step 1: Implement Remedies panel**

```javascript
function renderRemediesPanel() {
  // Three rows with +/- counters:
  //   Sustenance (VIG) — [count] [-] [+]
  //   Stimulant (CLA) — [count] [-] [+]
  //   Sacrament (SPI) — [count] [-] [+]
  // Small note: "Using a Remedy requires a whole Phase. Benefits all company present."
  // Counts are stored in currentChar.remedies
}
```

- [ ] **Step 2: Implement Contacts panel**

```javascript
function renderContactsPanel() {
  // Collapsible panel
  // List of contacts, each showing: name, description, favor (if any)
  // Each has an "edit" (inline) and "remove" button
  // "Add Contact" button at bottom — inline form: name, description, favor (optional)
}
```

- [ ] **Step 3: Implement Season Log panel**

```javascript
function renderSeasonLog() {
  // Collapsible panel
  // List of log entries: "[Season], [Age] — [Pursuit]: [notes]"
  // "New Season" button:
  //   Restores all Virtues to max
  //   Shows pursuit picker: Pilgrimage / Courtesy / Service
  //   Optional notes field
  //   Suggests next season (Spring→Harvest→Winter→Spring) but lets user override — season is ultimately user-set per spec
  //   Adds journal entry (auto)
  // "New Age" button:
  //   Restores all Virtues to max
  //   Adds 1 Glory
  //   Shows pursuit picker: Duty / Succession / Legacy
  //   Optionally advance Age (Young→Mature→Old) with reminders:
  //     Mature: "Reroll each Virtue on d12+d6, keep if higher"
  //     Old: "Reroll each Virtue on d12+d6, keep if lower. Lose d12 VIG at end of each Age."
  //   Adds journal entry (auto)
}
```

- [ ] **Step 4: Implement Scar History panel**

```javascript
function renderScarHistory() {
  // Collapsible panel: "SCAR HISTORY"
  // List of all scars from currentChar.scars[], each showing:
  //   Scar name (bold), roll number, description
  //   Mechanical effect in muted text
  //   Season earned
  // Empty state: "No scars yet."
}
```

- [ ] **Step 5: Implement Journal panel**

```javascript
function renderJournal() {
  // Newest-first list of journal entries
  // Each entry shows:
  //   Timestamp (formatted date), season badge, age badge
  //   Text content
  //   Auto entries: muted/italic styling, small "auto" indicator
  //   Manual entries: normal styling
  //   Delete button (x) on each entry
  //
  // "Add Entry" at top:
  //   Text input + "Add" button
  //   Creates entry with current timestamp, season, age
  //   auto: false
}
```

- [ ] **Step 6: Verify in browser**

- Remedy counts adjust with +/- buttons, min 0
- Add/edit/remove a contact
- Click "New Season" — verify Virtues restored, pursuit selector shown, season suggested, journal entry added
- Click "New Age" — verify Glory increments, age advancement prompt appears
- Scar history displays all scars earned so far
- Add a manual journal entry — verify it appears at top
- Verify auto-entries from previous actions (scar, knighting) are styled differently
- Delete a journal entry

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add remedies, contacts, season log, and journal panels"
```

---

## Task 8: Equipment Tab

**Files:**
- Modify: `index.html` (implement `renderEquipment()`)

- [ ] **Step 1: Implement Equipped section**

```javascript
function renderEquipment() {
  const el = document.getElementById('tab-equipment');
  el.innerHTML = '';

  el.appendChild(renderEquippedSection());
  el.appendChild(renderCarriedSection());
  el.appendChild(renderInventorySection());
  el.appendChild(renderAddItemPanel());
}

function renderEquippedSection() {
  // ARMOUR subsection:
  //   List each equipped armour piece: name, type, A[score]
  //   Total Armour score calculated and displayed prominently
  //     (sum of scores, excluding trapped shields)
  //   Each piece has: "Trapped" toggle (for shields), "Unequip" button (moves to carried)
  //
  // WEAPONS subsection:
  //   List each equipped weapon: name, dice, tags
  //   Tag reminders: "Hefty: one hand, only one hefty at a time"
  //                  "Long: two hands, impaired in confined spaces"
  //                  "Slow: cannot attack if moved, also Long"
  //   Each has: "Impaired" toggle, "Unequip" button
  //
  // Validation on equip:
  //   Armour: max one of each type (shield, coat, helm, plates)
  //   Show warning if trying to equip a second of the same type
}
```

- [ ] **Step 2: Implement Carried and Inventory sections**

```javascript
function renderCarriedSection() {
  // List of carried items (not equipped)
  // Each has: name, type, details, "Equip" button (with validation), "Drop" button (removes)
}

function renderInventorySection() {
  // Freeform list of misc items
  // Each item is a text string with an "x" to remove
  // "Add" button with text input at bottom
}
```

- [ ] **Step 3: Implement Add Item panel**

```javascript
function renderAddItemPanel() {
  // Two modes: "From Database" and "Custom"
  //
  // From Database:
  //   Search input that filters EQUIPMENT_DB.weapons and EQUIPMENT_DB.armour
  //   Results shown as a list with name, dice/score, tags, rarity
  //   Click to add to Carried or Equipped (buttons on each result)
  //
  // Custom:
  //   Form fields: name, type (weapon/armour/item), dice, armour score, tags, notes
  //   "Add" button → adds to weapons/armour/carried with custom:true flag
}
```

- [ ] **Step 4: Verify in browser**

- View equipped weapons and armour from character creation
- Unequip a weapon — verify it moves to Carried
- Equip it back — verify it returns to Equipped
- Try to equip a second shield — verify warning
- Toggle "Impaired" on a weapon
- Search equipment database — add a Longsword from results
- Add a custom item (like "The Black Axe: d10 long, +d10 vs Wounded")
- Add/remove inventory items
- Verify total Armour score updates correctly

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add equipment tab with equipped/carried/inventory and database search"
```

---

## Task 9: Battle Tab — Stat Bar + Dice Pool Builder

**Files:**
- Modify: `index.html` (implement `renderBattle()` — first half)

- [ ] **Step 1: Implement compact stat bar**

```javascript
function renderBattle() {
  const el = document.getElementById('tab-battle');
  el.innerHTML = '';

  el.appendChild(renderBattleStatBar());
  el.appendChild(renderDicePoolBuilder());
  el.appendChild(renderAttackFlow());
  el.appendChild(renderFeatsPanel());
  el.appendChild(renderGambitsPanel());
  el.appendChild(renderDamageCalculator());
  if (currentChar.mount) el.appendChild(renderMountedCombat());
}

function renderBattleStatBar() {
  // Sticky bar at top of battle tab:
  //   GD: [current]/[max]  |  VIG: [current]  |  Armour: [total]
  //   Active conditions inline (derived + stored)
  //   Mount: [Mounted/Dismounted] if mount exists
  // Compact, always visible when scrolling
}
```

- [ ] **Step 2: Implement dice pool builder**

```javascript
function renderDicePoolBuilder() {
  // Panel: "YOUR ATTACK"
  //
  // Weapon checkboxes: for each equipped weapon, checkbox + name + dice
  //   Checking adds that weapon's dice to the pool
  //
  // Shield die: auto-checked if shield equipped, shows "d4 (shield)"
  //
  // Mount section (if mounted):
  //   "Charging" toggle → adds trample die to pool
  //
  // Bonus dice row:
  //   Quick-add buttons: [+d8] [+d10]
  //   Custom: dropdown (d4-d12) + [Add] button
  //   List of added bonus dice with [x] to remove each
  //
  // "Character Impaired" toggle:
  //   If checked (or auto-set from SPI 0): overrides entire pool to "d4 only"
  //   Shows warning: "IMPAIRED — Rolling d4 only, no bonus dice, no Feats"
  //
  // Display: "Dice Pool: d10 + d4 (shield) + d8 (bonus)" — large, clear
  //
  // [ROLL] button (optional convenience):
  //   Rolls all dice in pool, displays individual results
  //   Highlights the highest die
  //   Shows which dice are eligible for Gambits (4+) and Strong Gambits (8+)
}
```

- [ ] **Step 3: Verify in browser**

- Select weapons via checkboxes — verify pool updates
- Shield die auto-appears if shield equipped
- Add bonus dice — verify they appear in pool and can be removed
- Toggle Impaired — verify pool shows d4 only
- Click Roll (if implemented) — verify results display with highest highlighted

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add battle stat bar and dice pool builder"
```

---

## Task 10: Battle Tab — Attack Flow, Feats, Gambits

**Files:**
- Modify: `index.html` (continue `renderBattle()`)

- [ ] **Step 1: Implement attack flow reference**

```javascript
function renderAttackFlow() {
  // Collapsible panel: "ATTACK STEPS"
  // 7 numbered steps, each a collapsible row:
  //   Step number, title, brief rule text
  //   Collapsed by default, click to expand
  //
  // Steps:
  // 1. "Roll Dice" — All combatants attacking same target roll simultaneously
  // 2. "Deny" — Target or nearby ally may use Deny feat to discard one die
  // 3. "Gambits" — Discard dice showing 4+ for Gambit effects
  // 4. "Highest Die" — Take the highest remaining die as base damage
  // 5. "Bolster" — Add +1 per die used for Bolster gambit
  // 6. "Subtract Armour" — Reduce by target's total Armour score
  // 7. "Apply Damage" — Final damage dealt
  //
  // Reference only, no state modifications
}
```

- [ ] **Step 2: Implement Feats panel**

```javascript
function renderFeatsPanel() {
  // Panel: "FEATS"
  // Three feat cards side by side (stacked on mobile):
  //
  // Each card shows:
  //   Feat name, brief description
  //   Save type: "VIG Save" / "CLA Save" / "SPI Save"
  //   Current Save value (from character virtues)
  //   State indicator: green "Available" / yellow "Pending Save" / red "Fatigued"
  //
  // Interaction:
  //   Available → click "Use" → state becomes "pending"
  //     Two buttons appear: "Passed Save" (→ available) and "Failed Save" (→ fatigued)
  //   Pending → click Pass or Fail
  //   Fatigued → button disabled, shows "Fatigued" in red
  //     Cleared by "Restore GD" on character sheet
  //
  // Note: "Each Feat can only be used once per Attack by each combatant"
}
```

- [ ] **Step 3: Implement Gambits panel**

```javascript
function renderGambitsPanel() {
  // Panel: "GAMBITS"
  // Reference list of all Gambit options:
  //   Bolster — +1 total Damage
  //   Move — Move after Attack
  //   Repel — Push foe away (VIG Save to resist)
  //   Stop — Prevent foe moving next turn (VIG Save)
  //   Impair — Impair weapon next turn (VIG Save)
  //   Trap — Trap shield until next turn (VIG Save)
  //   Dismount — Dismount foe (VIG Save)
  //   Other — Similar impact (VIG Save)
  //
  // "Strong Gambit" section (highlighted box):
  //   "When using a die of 8+ in melee:"
  //   - No Save granted to target
  //   - Greater effect: disarm, break wooden shield/weapon, remove helm
  //
  // Note: "Discard any Attack die showing 4+ to perform a Gambit"
  // Note: "Bolster and Move do not grant a VIG Save to the target"
  //
  // This panel is reference only — no state changes
}
```

- [ ] **Step 4: Verify in browser**

- Attack flow steps expand/collapse
- Feat buttons cycle through states: Available → Pending → Pass/Fail
- Failed feat shows Fatigued state
- Go to Character Sheet, click Restore GD — return to Battle tab, verify feats cleared
- Gambits panel displays all options with Strong Gambit section

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add attack flow reference, feats panel, and gambits panel"
```

---

## Task 11: Battle Tab — Incoming Damage Calculator + Mounted Combat

**Files:**
- Modify: `index.html` (complete `renderBattle()`)

- [ ] **Step 1: Implement incoming damage calculator**

```javascript
function renderDamageCalculator() {
  // Panel: "INCOMING DAMAGE"
  //
  // Input: [raw damage number] (number input)
  //
  // "Apply Damage" button triggers applyDamage(rawDamage):
  //
  // Display damage flow step by step:
  //   1. "Raw damage: X"
  //   2. "Armour reduces by Y → Z damage"  (Y = total armour, Z = max(0, X - Y))
  //   3. If Z <= currentGD: "GD absorbs all damage: GD [before] → [after]"
  //      If Z > currentGD:
  //        "GD reduced to 0 (was [before])"
  //        → Trigger scar flow modal (select attack die, roll scar)
  //        "Excess [overflow] goes to VIG"
  //        → Apply overflow to VIG
  //   4. If VIG took damage:
  //        Set conditions.wounded = true
  //        Check mortal wound: overflow >= ceil(vigBefore / 2)
  //          If yes AND doomScar: "SLAIN — Doom Scar triggered"
  //          If yes: "MORTAL WOUND — Dies in 1 hour without aid"
  //            Set conditions.mortalWound = true
  //        Check slain: vig.current <= 0
  //          "SLAIN"
  //
  // All state changes happen via updateChar()
  // Journal auto-entry for significant events (Wounded, Mortal Wound, Scar)
}

function applyDamage(raw) {
  const armour = calcTotalArmour(); // sum of equipped armour scores (excluding trapped)
  let damage = Math.max(0, raw - armour);
  if (damage === 0) { /* show "Armour absorbed all damage" */ return; }

  const gdBefore = currentChar.gd.current;
  let excess = 0;

  if (damage >= gdBefore && gdBefore > 0) {
    excess = damage - gdBefore;
    updateChar(c => { c.gd.current = 0; });
    // Trigger scar flow — async (modal). Excess VIG damage applied in callback after scar resolves.
    scarFlow(excess, (excessDmg) => {
      if (excessDmg > 0) applyVigDamage(excessDmg);
    });
    return;
  } else if (gdBefore > 0) {
    updateChar(c => { c.gd.current -= damage; });
    return;
  }

  // GD already 0, all damage goes to VIG
  applyVigDamage(damage);
}

function applyVigDamage(damage) {
  const vigBefore = currentChar.virtues.vig.current;
  const newVig = Math.max(0, vigBefore - damage);
  const vigLost = vigBefore - newVig;

  updateChar(c => {
    c.virtues.vig.current = newVig;
    c.conditions.wounded = true;
  });

  if (newVig <= 0) {
    addJournalEntry(currentChar.name + ' has been SLAIN.', true);
    // Show slain alert
    return;
  }

  if (vigLost >= Math.ceil(vigBefore / 2)) {
    if (currentChar.conditions.doomScar) {
      addJournalEntry('MORTAL WOUND with Doom Scar active — SLAIN.', true);
      // Show slain alert
    } else {
      updateChar(c => { c.conditions.mortalWound = true; });
      addJournalEntry('MORTAL WOUND — dies in 1 hour without aid.', true);
      // Show mortal wound warning
    }
  } else {
    addJournalEntry('Wounded — took ' + vigLost + ' VIG damage.', true);
  }
}

function calcTotalArmour() {
  return currentChar.armour
    .filter(a => !a.trapped)
    .reduce((sum, a) => sum + a.score, 0);
}
```

- [ ] **Step 2: Implement mounted combat section**

```javascript
function renderMountedCombat() {
  // Panel: "MOUNTED COMBAT" (only if mount exists)
  //
  // Toggle: [Mounted] / [Dismounted] — stored in a UI state variable (not persisted)
  //
  // When Mounted:
  //   "Charge" toggle → when active, adds mount's trample die to dice pool
  //   Reminder: "Charging doesn't work against spearwalls"
  //   Reminder: "Lance counts as hefty when mounted"
  //
  // "Dismounted in Combat" button:
  //   Shows: "Dismounted! d6 damage to rider, added to attacker's dice"
  //   Sets toggle to Dismounted
  //
  // Link: "Mount Stats" → scrolls to or highlights mount panel on Character Sheet
  // Reminder: "Mount can be targeted separately by attacks"
}
```

- [ ] **Step 3: Verify in browser**

- Enter 5 damage with Armour 2 → verify 3 applied to GD
- Enter damage that exceeds GD → verify scar flow triggers, excess goes to VIG
- Enter damage causing Mortal Wound (half+ VIG) → verify warning and condition set
- Enter damage with Doom Scar active → verify Slain alert on Mortal Wound
- Enter damage reducing VIG to 0 → verify Slain alert
- Test mounted combat toggle — charge adds trample to pool
- Test dismount button

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add incoming damage calculator and mounted combat section"
```

---

## Task 12: Polish + Mobile Responsive + Final Integration

**Files:**
- Modify: `index.html` (CSS polish, responsive, edge cases)

- [ ] **Step 1: Add mobile responsive CSS**

```css
@media (max-width: 768px) {
  /* Tabs: full-width stacked buttons */
  /* Stat boxes: stack vertically */
  /* Feat cards: stack instead of side-by-side */
  /* Panels: full-width, no side margins */
  /* +/- buttons: minimum 44px touch target */
  /* Modal: full-width with padding */
}
```

- [ ] **Step 2: Add visual polish**

- Decorative borders on panels (CSS border-image or box-shadow tricks)
- Scar history entries styled with the scar name in bold, effect in muted text
- Journal auto-entries in italic with muted color
- Condition tags styled as small shields/banners (border-radius + background)
- Transitions on tab switching (fade or slide)
- Gold accent on active tab, hover states on buttons
- Section divider ornaments between panels (CSS pseudo-elements with decorative characters like `⚔` or `―`)

- [ ] **Step 3: Edge case handling**

- Verify behavior with no characters (should show creation modal)
- Verify behavior with only one character (delete should show creation modal)
- Verify Virtue values can't exceed 19 or go below 0
- Verify GD max increases from scars are applied correctly
- Verify trapped shield reduces Armour total in battle tab
- Verify weaponImpaired shows on both Equipment and Battle tabs
- Verify all auto-journal entries generate correctly across all flows

- [ ] **Step 4: Cross-browser test**

Open in Chrome, Firefox, and Edge. Verify:
- Theme renders consistently
- localStorage works
- All interactions function
- Mobile responsive at 375px width

- [ ] **Step 5: Final commit**

```bash
git add index.html
git commit -m "feat: add mobile responsive layout and visual polish"
```

---

## Task Summary

| Task | Description | Depends On |
|------|-------------|------------|
| 1 | HTML Shell + Parchment Theme + Tabs | — |
| 2 | Game Databases (Knights, Equipment, Scars) | 1 |
| 3 | State Management (localStorage CRUD) | 1 |
| 4 | Character Creation + Switcher | 2, 3 |
| 5 | Character Sheet: Virtues, Guard, Conditions | 3, 4 |
| 6 | Character Sheet: Ability, Passion, Mount, Squire | 5 |
| 7 | Character Sheet: Remedies, Contacts, Season Log, Journal | 5 |
| 8 | Equipment Tab | 3, 4 |
| 9 | Battle Tab: Stat Bar + Dice Pool | 8 |
| 10 | Battle Tab: Attack Flow, Feats, Gambits | 9 |
| 11 | Battle Tab: Damage Calculator + Mounted Combat | 5 (scar flow), 9 |
| 12 | Polish + Mobile Responsive | All above |

Tasks 2 and 3 can run in parallel after Task 1. Tasks 6 and 7 can run in parallel. Tasks 8-11 are sequential within the Battle tab but can overlap with 6-7.
