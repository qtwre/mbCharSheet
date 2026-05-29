# Usability Improvements Design

Date: 2026-05-28

Make the Mythic Bastionland Character Tracker friendly for a general audience to pick up and use immediately.

## Features

### 1. Help Tooltip System

Dense "?" button coverage across the entire app, explaining both what each UI element does and the game mechanic behind it.

**Component:**
- `.help-btn` — small gold circle with "?" text, placed inline next to panel headers and individual controls
- `.help-popover` — floating popover anchored near the clicked "?", parchment-styled with gold border and a caret pointing at the trigger. Dismissed by clicking anywhere else
- `showHelp(event, key)` — JS function that looks up text from a `HELP_TEXT` object, positions the popover, and shows it. A document-level click listener dismisses any open popover

**HELP_TEXT object:** ~40-50 keyed entries, each 1-3 sentences. Content based on Mythic Bastionland rules. Focused on "what does this do in the game" — no page references, no UI-only instructions.

**Placement — panel headers (one "?" per header):**

Character Sheet tab:
- Virtues
- Guard
- Conditions
- Ability
- Passion
- Mount
- Squire
- Remedies
- Contacts
- Season Log
- Scar History
- Journal

Equipment tab:
- Armour
- Weapons
- Carried (Unequipped)
- Inventory
- Add Item

Battle tab:
- Your Attack
- Attack Steps
- Feats
- Gambits Reference
- Incoming Damage
- Mounted Combat

**Placement — individual controls:**
- Each virtue's Restore button (explains remedy usage)
- Restore GD button (GD restoration + feat refresh)
- Each condition toggle: Wounded, Mortal Wound, Doom Scar
- Each derived condition: Exhausted, Exposed, Impaired, Fatigued
- Indulge Passion button
- Impaired checkbox (battle tab)
- Charging checkbox
- Shield die auto-include note
- Each feat: Smite, Focus, Deny (what it does + save mechanic)
- Bonus dice section
- Damage calculator inputs
- Armour trapped toggle
- Weapon impaired toggle
- Glory +/- in header
- Age and Season selectors in header
- Export/Import buttons

### 2. Backup Indicator

A persistent visual indicator of backup status with one-click full backup.

**Location:** Footer, left of existing attribution text.

**Display:** "Last backup: never" or "Last backup: 2 hours ago" with a "Backup Now" button.

**Visual states:**
- Green — backed up within last 24 hours
- Gold/orange — backed up 1-7 days ago, or never backed up with fewer than 5 changes
- Red — never backed up, or more than 7 days since last backup

**Storage:**
- `mythicBastion_lastBackup` — ISO timestamp, updated on "Backup Now" click or existing Export button use
- `mythicBastion_changeCount` — integer, incremented on each `updateChar()` call, reset to 0 on backup. Used for the gold/orange state threshold.

**"Backup Now" behavior:** Exports ALL characters as a single JSON array. File named `mythic-bastionland-backup-YYYY-MM-DD.json`.

**Import compatibility:** Existing Import function extended to detect whether imported JSON is a single character object or a backup array, and handle both.

### 3. Custom Knight Creation

Allow fully freeform character creation without selecting a predefined knight type.

**Modal change:** Add `"-- Custom Knight --"` as the first option in the knight type dropdown (above the grouped knight list).

**When "Custom Knight" selected, the form shows:**
- Name input (existing)
- VIG, CLA, SPI, GD number inputs (existing)
- Ability: name text input + description textarea
- Passion: name text input + description textarea
- No starting equipment — user adds via Equipment tab after creation

**Data:** Character created with `knightType: 'custom'`. Header displays "Custom Knight" where it normally shows knight number/name.

### 4. GitHub Pages Hosting

Host the tracker as a static site for zero-friction access.

**Setup:**
- Create new GitHub repo named `mbCharSheet`
- Add `.gitignore` excluding everything except `index.html` and config files (PDFs, .txt, .js knight data, campaign notes, etc. all excluded)
- Push `main` branch
- Enable GitHub Pages from `main` branch root

**Result:** Public URL like `https://<username>.github.io/mbCharSheet/`

## Architecture

All changes are additive layers to the existing single-file `index.html`:
- Help system: new CSS classes + `HELP_TEXT` data object + `showHelp()` function + "?" buttons inserted in render functions
- Backup indicator: new footer HTML + timestamp tracking logic + multi-character export function
- Custom knight: modified `showNewCharacterModal()` + conditional form fields
- GitHub Pages: repo setup only, no code changes

No refactoring of existing code. No external dependencies. Single-file portability preserved.
