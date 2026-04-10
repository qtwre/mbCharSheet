# Mythic Bastionland Character Tracker — Design Spec

## Overview

A single-file HTML webapp for tracking a player character in Mythic Bastionland. Designed as a player companion at the table — tracks stats, gear, and combat mechanics without replacing the Referee. Parchment/medieval visual theme.

## Constraints

- **Single HTML file** — vanilla JS, no build step, no dependencies
- **localStorage** for persistence — one JSON object per character (key: `mythicBastion_char_{id}`), multiple characters supported
- **Solo use** — no server, no sharing. Data model keeps a clean per-character boundary so multiplayer could be added later
- **Player companion, not Referee tool** — the app tracks state and assists with mechanics, the Referee owns narrative, NPC actions, and ruling calls

## Data Model

```
Character {
  id: string                // unique ID
  name: string              // "Sir Batman"
  knightType: string        // key into knight database
  glory: number             // 0-12
  age: string               // "Young" | "Mature" | "Old" — user sets manually
  season: string            // "Spring" | "Harvest" | "Winter" — user sets manually, used for scar records and Doom Scar tracking
  // rank auto-derived from glory using RANK_THRESHOLDS:
  // { 0: "Knight-Errant", 3: "Knight-Gallant", 6: "Knight-Tenant", 9: "Knight-Dominant", 12: "Knight-Radiant" }

  virtues: {
    vig: { current: number, max: number }
    cla: { current: number, max: number }
    spi: { current: number, max: number }
  }
  gd: { current: number, max: number }

  ability: { name: string, description: string }
  passion: { name: string, description: string }

  armour: [                 // equipped armour pieces
    { name: string, type: string, score: number, notes: string, trapped: boolean }
  ]

  weapons: [                // equipped weapons
    { name: string, dice: string, tags: string[], notes: string, weaponImpaired: boolean, custom: boolean }
    // weaponImpaired = Gambit effect (weapon impaired next turn), distinct from character Impaired (SPI 0)
  ]

  carried: [                // owned but not equipped
    { name: string, type: string, details: string }
  ]

  remedies: {
    sustenance: number      // count of Sustenance remedies carried (restores VIG)
    stimulant: number       // count of Stimulant remedies carried (restores CLA)
    sacrament: number       // count of Sacrament remedies carried (restores SPI)
  }

  inventory: string[]       // freeform misc items

  mount: {
    name: string
    description: string
    vig: { current: number, max: number }
    cla: { current: number, max: number }
    spi: { current: number, max: number }
    gd: { current: number, max: number }
    trample: string         // damage die, e.g. "d8", or null
    traits: string[]
  } | null

  squire: {
    name: string
    description: string
    vig: { current: number, max: number }
    cla: { current: number, max: number }
    spi: { current: number, max: number }
    gd: { current: number, max: number }
    equipment: string[]
    mount: {                // squire mounts are always ponies per rules, but model is flexible
      name: string, description: string,
      vig: { current: number, max: number }, cla: { current: number, max: number },
      spi: { current: number, max: number }, gd: { current: number, max: number },
      trample: string | null, traits: string[]
    } | null
    knighted: boolean       // when knighted: add d6 to each Virtue, archive squire data to notes
  } | null

  scars: [
    { roll: number, name: string, description: string, effect: string, season: string }
  ]

  feats: {
    smite: string           // "available" | "pending" | "fatigued"
    focus: string           // "available" | "pending" | "fatigued"
    deny: string            // "available" | "pending" | "fatigued"
  }

  conditions: {
    doomScar: boolean       // scar #11 — Mortal Wound = Slain this Season
    wounded: boolean         // took VIG damage this combat
    mortalWound: boolean     // lost half+ remaining VIG from one hit
    customConditions: string[]  // user-defined conditions
  }
  // Derived conditions (not stored, computed from state):
  //   Exhausted: VIG current = 0
  //   Exposed: CLA current = 0
  //   Character Impaired: SPI current = 0 (distinct from weapon weaponImpaired)
  //   Fatigued: any feat in "fatigued" state

  contacts: [               // NPCs met, favors owed/earned via Courtesy pursuit
    { name: string, description: string, favor: string | null }
  ]

  seasonLog: [              // between-season/age pursuits chosen
    { season: string, age: string, pursuit: string, notes: string }
  ]

  journal: [                // campaign journal, newest first
    { timestamp: string, season: string, age: string, text: string, auto: boolean }
    // auto = true for system-generated entries (scar, knighting, age change, etc.)
  ]
}
```

## Embedded Databases

### Knight Database

All 36+ knights from the book. Each entry:

- `id`: string key
- `name`: display name (e.g. "The Vulture Knight")
- `number`: knight number from the book
- `flavor`: the two-line verse
- `property`: array of starting items with full stats
- `ability`: { name, description }
- `passion`: { name, description }
- `seer`: { name, description } (reference only)

On knight selection, the app pre-fills Property (into weapons/armour/inventory), Ability, and Passion. Player then inputs rolled Virtues and GD.

### Equipment Database

From Arms & Goods (p12):

**Weapons:**
- Common: hand weapons (d6), hefty weapons (d8 hefty), long weapons (d10 long), sling (d4 hefty), javelin (d6 hefty), shortbow (d6 long)
- Uncommon: shortsword (2d6), lance (d10 long, hefty if mounted), brutal weapons (2d10 slow), longbow (d8 slow)
- Rare: longsword (2d8 hefty), greatsword (2d10 long), curvebow (2d6 long), crossbow (2d8 slow)

**Armour:**
- Common: shield (d4, A1)
- Uncommon: coat (A1), helm (A1)
- Rare: plates (A1)

Max A4 from stacking one of each type. App enforces this.

**Beasts:**
- Common: hound, pony, mule
- Uncommon: ox, hawk, riding steed, heavy steed
- Rare: charger

Custom items supported for homebrew or unique gear (e.g. "The Black Axe: d10 long, +d10 vs Wounded").

### Scar Table

The scar table from p9 (entries 1-12). The die rolled is the attack die that caused the scar (not always d12 — a d6 dagger can only cause scars 1-6). Each entry has name, description, mechanical effect, and any GD max increase condition.

## Tab Structure

### Tab 1: Character Sheet

**Header area:**
- Character name (editable), knight type, Glory counter with +/- (rank auto-displays)
- Age selector (Young / Mature / Old) and Season selector (Spring / Harvest / Winter)
- Character switcher dropdown (for multiple saved characters)
- New Character / Delete Character controls (Delete requires confirmation dialog)

**The Oath** (displayed as a persistent banner or decorative element):
- "Seek the Myths. Honour the Seers. Protect the Realm."

**New Character flow** (inline panel or modal):
1. Select knight from dropdown (grouped by number) — auto-fills Ability, Passion, and starting Property
2. Enter character name
3. Input rolled Virtues (VIG, CLA, SPI) and GD — values capped at 0-19 per rules
4. Optionally add a squire (for small companies) and mount
5. "Create" saves to localStorage and switches to the new character

**Virtues panel:**
- VIG, CLA, SPI — each shows current/max with +/- buttons
- "Restore" button per virtue (sets current = max, for when a Remedy is used or between seasons)
- Restore buttons are linked to Remedies: clicking "Restore VIG" prompts to consume a Sustenance if available (decrement count), or allows manual restore without one
- Color-coded warnings at 0:
  - VIG 0: "Exhausted — cannot attack if you moved this turn"
  - CLA 0: "Exposed — treated as 0 GD"
  - SPI 0: "Impaired — attacks roll d4 only"

**Guard panel:**
- Current/max GD with +/- buttons
- "Restore GD" button (moment of peace — sets current = max, clears all Fatigued feats). Does NOT clear Wounded or Mortal Wound — those require VIG restoration via Remedy.
- When GD reaches 0 (whether damage stops at 0 or overflows into VIG): auto-triggers Scar flow. This is the canonical scar/damage flow — the Battle tab's Incoming damage calculator uses this same logic.
  - Prompts user to select the attack die that caused the scar (d4-d12), then rolls that die on the scar table
  - Displays result from the scar table
  - Records to scar history
  - Applies mechanical effects (GD max increase if applicable, Doom Scar flag, etc.)
  - Any excess damage beyond GD is then applied to VIG

**Conditions banner:**
- Active conditions as toggleable tags: Doom Scar, Wounded, Mortal Wound, plus user-defined custom conditions
- Auto-derived conditions (not toggleable, computed from state): Exhausted (VIG 0), Exposed (CLA 0), Character Impaired (SPI 0), Fatigued (any feat fatigued)
- "Character Impaired" label distinguishes from weapon impairment in the Equipment tab

**Ability & Passion cards:**
- Read-only display from knight database
- Passion includes a "Restore SPI" button (indulging passion)

**Squire panel** (collapsible, shown if squire exists):
- Name, description
- Virtues (2d6 each) and GD (1) with +/- buttons
- Equipment list
- Pony stats
- "Knight this Squire" button: rolls d6 separately for each Virtue, adds result to both current and max. Archives squire info to character notes, then removes squire from data model.

**Mount panel** (collapsible, shown if mount exists):
- Name, description, traits
- VIG, CLA, SPI, GD with +/- buttons
- Trample die display

**Remedies panel:**
- Three remedy types displayed with count and +/- buttons:
  - Sustenance (restores VIG) — bulky, one per person/beast
  - Stimulant (restores CLA) — bulky, one per person/beast
  - Sacrament (restores SPI) — bulky, one per person/beast
- Note: "Using a Remedy requires a whole Phase. Benefits all company present."

**Contacts panel** (collapsible):
- List of NPCs with name, description, and any favor owed/earned
- Add/edit/remove contacts
- Gained through the Courtesy pursuit between seasons, or through play

**Season Log** (collapsible):
- History of between-season and between-age pursuits chosen
- Each entry: season, age, pursuit type, and notes
- Between-Season pursuits: Pilgrimage, Courtesy, Service
- Between-Age pursuits: Duty, Succession, Legacy
- "New Season" button: prompts to restore all Virtues and select a pursuit
- "New Age" button: prompts to restore Virtues, gain 1 Glory, select a pursuit, and optionally advance Age (with Virtue reroll reminders — Mature: reroll d12+d6 keep higher; Old: reroll d12+d6 keep lower)

**Scar history:**
- List of all scars with name, effect, and when earned

**Journal:**
- Chronological list of entries, newest first
- Each entry shows: timestamp, season, age, and text
- "Add Entry" button with text input — creates a manual entry tagged with current season/age
- Auto-entries generated for significant events: scar gained, squire knighted, age/season advanced, Glory gained, Mortal Wound survived
- Auto-entries are visually distinct (muted/italic) from manual entries
- Entries can be deleted but not edited (preserves the record)

### Tab 2: Equipment

**Equipped section:**
- Armour: list of worn pieces, total Armour score calculated and displayed
  - Enforces one-of-each-type rule (shield, coat, helm, plates)
- Weapons: list of wielded weapons with dice and tags
  - Shows hefty/long/slow restrictions as reminders
- Item condition tags: "Impaired" (from Gambit), "Trapped" (shield, until next turn)
- Unequip button moves items to Carried

**Carried section:**
- Items owned but not equipped
- Equip button moves to Equipped (with rule validation)

**Inventory section:**
- Freeform list for misc items (Spidernip Nuts, torches, rope, etc.)
- Add/remove entries

**Add Item panel:**
- Search/browse the equipment database by name or category
- Select to add to Carried or Equipped
- "Add Custom Item" — name, type, dice/stats, tags, notes

### Tab 3: Battle

**Compact stat bar** (sticky at top):
- GD, VIG (current), total Armour, active conditions
- Mount status if mounted

**Dice pool builder:**
- Auto-populated from equipped weapons
- Checkboxes to select which weapons you're using this attack
- Mount trample die auto-included when "Mounted + Charging" is toggled
- Bonus dice: +d8 / +d10 / custom buttons
- Shield attack die (d4) included if shield equipped — per rules, shields contribute a d4 to the attack roll in addition to providing A1 armour
- Displays full pool: e.g. "d10 + d4 (shield) + d8 bonus"
- "Impaired" toggle overrides everything to d4 only

**Attack flow** (guided step display):
1. All combatants attacking same target roll dice simultaneously
2. Deny check — can target or nearby ally use Deny?
3. Gambits — discard dice 4+ for effects
4. Take highest remaining die
5. Bolster damage from discarded dice
6. Subtract target's Armour
7. Final damage

Each step is a collapsible row with the rule summary. Not enforced — just a reference checklist.

**Feats panel:**
- Three buttons: Smite, Focus, Deny
- Each shows:
  - What it does (brief rule text)
  - Which Save it requires (Smite=VIG, Focus=CLA, Deny=SPI)
  - Current state: Available / Pending Save / Fatigued
- On use: state moves to "Pending Save". Two inline buttons appear: "Passed" (→ Available) and "Failed" (→ Fatigued)
- All Fatigued feats clear on "Restore GD" (moment of peace)

**Gambits panel:**
- Lists available Gambits: Bolster, Move, Repel, Stop, Impair, Trap, Dismount, Other
- Highlights "Strong Gambit" options when applicable (die 8+ in melee)
- Strong Gambit extras: No Save for target, Greater effect (disarm, break shield/weapon, remove helm)
- Reference only — doesn't modify state automatically

**Incoming damage calculator:**
- Input raw damage number
- "Apply Damage" button:
  - Subtracts Armour automatically
  - Deducts from GD first
  - If GD reaches 0: triggers Scar flow (user selects attack die), then applies excess to VIG
  - If any damage reaches VIG: sets Wounded condition
  - If damage to VIG >= ceil(currentVIG / 2): sets Mortal Wound (or Slain if Doom Scar active). Shows warning: "Mortally Wounded — dies in 1 hour without aid"
  - If VIG reaches 0: shows Slain alert

**Mounted combat section** (shown when mount exists):
- Toggle: Mounted / Dismounted
- Charge toggle: adds trample to dice pool
- "Dismounted in combat" button: applies d6 Damage to the rider (added to the existing attack's damage dice against them)
- Mount can be targeted separately (link to mount stats)

## Visual Design

**Theme: Parchment/Medieval**
- Aged parchment background texture (CSS gradient, no external images to keep single-file)
- Serif fonts (system serif stack, or embedded Google Font if size permits)
- Dark brown/ink text
- Muted gold accents for headers, borders, and active states
- Heraldic/shield motifs for section dividers
- Subtle worn-edge effects on panels
- Tab styling as wax-sealed or ribbon bookmarks
- Buttons styled as embossed leather or wax stamps
- Condition tags as heraldic banners or shields

## Interactions & UX

- **Auto-save:** Every state change writes to localStorage immediately
- **+/- buttons:** Tap-friendly for table use, large enough for mobile
- **Keyboard shortcuts:** Not required for v1, but data model supports it
- **Mobile responsive:** Single-column layout on narrow screens, tabs stack or become a hamburger menu
- **Undo:** Not in v1, but the auto-save makes it low-risk (can always manually adjust)

## Out of Scope (v1)

- Multiplayer / shared state
- Referee tools (NPC tracking, Myth progression, Wilderness rolls)
- Domain management (Holdings, Council, Crises, Mustering) — deferred to future version when player reaches Knight-Tenant rank
- Spark Tables / random generators
- Import/export (future: JSON export for backup or sharing)
- Offline PWA features
- Printing / PDF export
