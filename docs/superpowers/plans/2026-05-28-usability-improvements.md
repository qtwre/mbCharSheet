# Usability Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Mythic Bastionland Character Tracker friendly for new users by adding contextual help tooltips, a backup status indicator, custom knight creation, and GitHub Pages hosting.

**Architecture:** All changes are additive layers to the single-file `index.html`. Help system adds CSS + a `HELP_TEXT` data object + a `showHelp()` popover function + `?` buttons inserted into existing render functions. Backup indicator adds footer HTML + localStorage timestamp tracking. Custom knight adds a branch to the existing creation modal. GitHub Pages is repo config only.

**Tech Stack:** Vanilla HTML/CSS/JS, localStorage, GitHub Pages

---

### Task 1: Help Tooltip CSS + JS Infrastructure

**Files:**
- Modify: `index.html:7-1797` (CSS section, add help-btn and help-popover styles)
- Modify: `index.html:1833-1835` (JS section, add showHelp/dismissHelp functions before EQUIPMENT_DB)

- [ ] **Step 1: Add CSS for help button and popover**

Insert after the `.mount-reminder` styles (line ~1796), before the closing `</style>` tag:

```css
    /* ===== Help Tooltips ===== */
    .help-btn {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      width: 18px;
      height: 18px;
      border-radius: 50%;
      background: var(--gold);
      color: var(--parchment-bg);
      font-size: 0.7rem;
      font-weight: bold;
      font-family: var(--font-serif);
      cursor: pointer;
      border: 1px solid var(--gold-light);
      line-height: 1;
      margin-left: 6px;
      vertical-align: middle;
      flex-shrink: 0;
      box-shadow: 0 1px 2px rgba(0,0,0,0.15);
      transition: background 0.15s;
    }

    .help-btn:hover {
      background: var(--gold-light);
    }

    .help-popover {
      position: fixed;
      z-index: 1000;
      max-width: 320px;
      padding: 10px 14px;
      background: var(--parchment-bg);
      border: 2px solid var(--gold);
      border-radius: var(--radius);
      box-shadow: 0 4px 16px rgba(0,0,0,0.2);
      font-size: 0.85rem;
      font-family: var(--font-body);
      color: var(--ink);
      line-height: 1.45;
      pointer-events: auto;
    }

    .help-popover::before {
      content: '';
      position: absolute;
      top: -8px;
      left: 16px;
      width: 14px;
      height: 14px;
      background: var(--parchment-bg);
      border-left: 2px solid var(--gold);
      border-top: 2px solid var(--gold);
      transform: rotate(45deg);
    }

    .help-popover-title {
      font-weight: bold;
      font-family: var(--font-serif);
      font-size: 0.8rem;
      text-transform: uppercase;
      letter-spacing: 0.4px;
      color: var(--gold);
      margin-bottom: 4px;
    }

    /* Help btn inside panel headers — don't mess with collapsible click */
    .panel-header .help-btn {
      position: relative;
      z-index: 2;
    }

    /* Inline help btn next to controls */
    .help-inline {
      width: 16px;
      height: 16px;
      font-size: 0.65rem;
      margin-left: 4px;
    }
```

- [ ] **Step 2: Add showHelp/dismissHelp JS functions**

Insert at the start of the `<script>` section (line ~1833), right after `<script>`:

```javascript
    // === HELP TOOLTIP SYSTEM ===

    let activePopover = null;

    function showHelp(event, key) {
      event.stopPropagation();
      dismissHelp();

      const text = HELP_TEXT[key];
      if (!text) return;

      const btn = event.currentTarget;
      const rect = btn.getBoundingClientRect();

      const popover = document.createElement('div');
      popover.className = 'help-popover';

      const title = key.replace(/[-_]/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
      popover.innerHTML = `<div class="help-popover-title">${title}</div>${text}`;

      document.body.appendChild(popover);
      activePopover = popover;

      // Position: prefer below the button, shift left if needed
      const popRect = popover.getBoundingClientRect();
      let top = rect.bottom + 8;
      let left = rect.left - 4;

      // If it would go off the right edge, shift left
      if (left + popRect.width > window.innerWidth - 12) {
        left = window.innerWidth - popRect.width - 12;
      }
      // If it would go off the bottom, show above
      if (top + popRect.height > window.innerHeight - 12) {
        top = rect.top - popRect.height - 8;
        popover.style.setProperty('--arrow-side', 'bottom');
        // Move arrow to bottom
        popover.querySelector('.help-popover-title') && (popover.style.cssText += '');
        const arrow = popover.querySelector('::before');
      }
      if (left < 12) left = 12;

      popover.style.top = top + 'px';
      popover.style.left = left + 'px';

      // Dismiss on next click anywhere
      setTimeout(() => {
        document.addEventListener('click', dismissHelp, { once: true });
      }, 0);
    }

    function dismissHelp() {
      if (activePopover) {
        activePopover.remove();
        activePopover = null;
      }
    }

    function helpBtn(key, inline) {
      const cls = inline ? 'help-btn help-inline' : 'help-btn';
      return `<span class="${cls}" onclick="showHelp(event,'${key}')" title="Help">?</span>`;
    }
```

- [ ] **Step 3: Verify the popover renders**

Open `index.html` in a browser. Open the console and run:

```javascript
showHelp({stopPropagation:()=>{}, currentTarget: document.querySelector('.panel-header')}, 'virtues');
```

Expected: Nothing visible yet (HELP_TEXT not defined), but no JS errors. The function should be defined.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add help tooltip CSS and JS infrastructure"
```

---

### Task 2: HELP_TEXT Data Object

**Files:**
- Modify: `index.html` (JS section, insert HELP_TEXT after the help system functions, before EQUIPMENT_DB)

- [ ] **Step 1: Add the HELP_TEXT object**

Insert after the `helpBtn()` function, before `const EQUIPMENT_DB`:

```javascript
    const HELP_TEXT = {
      // --- Character Sheet: Panel Headers ---
      virtues: 'Your three Virtues: <strong>VIG</strong> (Vigour — physical strength), <strong>CLA</strong> (Clarity — awareness and wit), <strong>SPI</strong> (Spirit — willpower and social force). When a Virtue hits 0, you suffer a penalty. Saves are rolled on d20 — roll equal to or under the Virtue to pass.',
      guard: '<strong>Guard Dice (GD)</strong> represent your skill at avoiding harm. Damage reduces GD first. When GD hits 0, you take a Scar. GD is fully restored after a moment of peace.',
      conditions: 'Active conditions affecting your knight. <strong>Wounded</strong>: took VIG damage. <strong>Mortal Wound</strong>: lost half or more remaining VIG in one hit — die in 1 hour without aid. <strong>Doom Scar</strong>: if you take a Mortal Wound this Season, you are Slain instead.',
      ability: 'Your knight\'s unique Ability — a special rule or power tied to your archetype.',
      passion: 'Your knight\'s Passion — a drive or obsession. <strong>Indulging your Passion</strong> restores SPI to max without needing a Sacrament.',
      mount: 'Your mount has its own Virtues and GD. It can be targeted separately by enemies. <strong>Trample Die</strong>: added to your attack pool when Charging.',
      squire: 'Your squire is a companion with their own stats and equipment. A squire can eventually be <strong>Knighted</strong>, which transfers some of your Virtues to create a new character.',
      remedies: '<strong>Sustenance</strong> restores VIG. <strong>Stimulant</strong> restores CLA. <strong>Sacrament</strong> restores SPI. Using a Remedy takes a whole Phase and benefits everyone present.',
      contacts: 'People your knight has met and built relationships with. Track favors owed and descriptions to remember them by.',
      'season-log': 'A record of seasonal progressions. Each Season you choose a <strong>Pursuit</strong> — what your knight does during downtime.',
      'scar-history': 'Every time your GD is reduced to exactly 0, you roll on the d12 Scar table. Scars can reduce Virtues, increase max GD, or inflict the dreaded <strong>Doom</strong>.',
      journal: 'A log of significant events. Actions that change your character (scars, feat uses, knighting, etc.) create auto-entries with <strong>Restore</strong> buttons to undo them.',

      // --- Character Sheet: Individual Controls ---
      'restore-virtue': 'Restores this Virtue to its max. If you have the matching Remedy, it will be consumed. VIG uses Sustenance, CLA uses Stimulant, SPI uses Sacrament.',
      'restore-gd': 'Restores Guard to max. GD comes back after any moment of peace — a short rest, end of combat, etc. Also refreshes all Fatigued Feats.',
      wounded: '<strong>Wounded</strong>: You have taken VIG damage. Cleared when VIG is restored.',
      'mortal-wound': '<strong>Mortal Wound</strong>: You lost half or more of your remaining VIG in a single hit. You will die within 1 hour unless you receive aid.',
      'doom-scar': '<strong>Doom Scar</strong>: A cheated death haunts you. If you take a Mortal Wound this Season, you are Slain instantly instead.',
      exhausted: '<strong>Exhausted</strong> (VIG = 0): You cannot attack if you moved this turn.',
      exposed: '<strong>Exposed</strong> (CLA = 0): Treated as having 0 GD — all damage goes straight to VIG.',
      impaired: '<strong>Impaired</strong> (SPI = 0): Your entire attack dice pool is reduced to a single d4.',
      fatigued: '<strong>Fatigued</strong>: One or more Feats are spent. Fatigued Feats are restored when GD is fully restored.',
      'indulge-passion': 'Indulging your Passion restores SPI to its maximum without needing a Sacrament remedy.',

      // --- Equipment: Panel Headers ---
      armour: 'Armour reduces incoming damage. Types: <strong>Shield</strong> (A1, adds shield die to attack), <strong>Coat</strong> (A1), <strong>Helm</strong> (A1), <strong>Plates</strong> (A1). Max total is A4. Only one of each type.',
      weapons: 'Your equipped weapons. Tags: <strong>Hefty</strong> (one hand, only one allowed), <strong>Long</strong> (two hands), <strong>Slow</strong> (can\'t attack if you moved, also Long).',
      carried: 'Items you\'re carrying but not actively using. You can equip them when needed.',
      inventory: 'Miscellaneous items, consumables, and supplies. Freeform text entries.',
      'add-item': 'Add equipment from the game database or create custom items with your own stats.',

      // --- Equipment: Individual Controls ---
      'armour-trapped': '<strong>Trapped</strong>: A Gambit can trap a shield, making it useless. A trapped shield doesn\'t count toward your Armour score or add its die to attacks. Toggle to mark it trapped or free.',
      'weapon-impaired': '<strong>Impaired Weapon</strong>: A Gambit can impair a weapon, reducing its die to d4. Toggle to mark it impaired or restored.',

      // --- Battle: Panel Headers ---
      'your-attack': 'Build your attack dice pool by selecting weapons. Roll all dice — take the <strong>highest</strong> as damage, use others (4+) for Gambits. Subtract target\'s Armour from the total.',
      'attack-steps': 'The seven steps of resolving an attack in order: Roll → Deny → Gambits → Highest Die → Bolster → Subtract Armour → Apply Damage.',
      feats: 'Three special abilities usable once per rest. After using a Feat, make a Save — fail and the Feat is <strong>Fatigued</strong> until GD is fully restored.',
      'gambits-ref': 'Discard attack dice showing 4+ to perform Gambits (Bolster, Repel, Stop, Impair, Trap, Dismount). Dice showing 8+ unlock <strong>Strong Gambits</strong>. You cannot discard your highest die.',
      'incoming-damage': 'Calculate damage your knight receives. Enter raw damage — Armour is subtracted automatically. Excess damage after GD goes to VIG, which can trigger Wounds, Mortal Wounds, or Scars.',
      'mounted-combat': 'While mounted, you can <strong>Charge</strong> to add your mount\'s trample die. Being dismounted in combat deals d6 damage. Your mount can be targeted separately.',

      // --- Battle: Individual Controls ---
      'feat-smite': '<strong>Smite</strong> (VIG Save): Release righteous fury. Use before rolling a melee attack — gain either +d12 or Blast (hits all nearby enemies).',
      'feat-focus': '<strong>Focus</strong> (CLA Save): Create an opening. Use after rolling an attack — perform a free Gambit without discarding a die.',
      'feat-deny': '<strong>Deny</strong> (SPI Save): Rebuff an attack. Use after an attack roll against you or a nearby ally — discard one die from the attacker\'s roll.',
      'battle-impaired': 'When <strong>Impaired</strong>, your entire dice pool is replaced with a single d4. This happens automatically when SPI = 0, or can be inflicted by a Gambit.',
      charging: '<strong>Charging</strong>: While mounted, add your mount\'s trample die to the attack pool. Does not work against spearwalls.',
      'shield-die': 'Your shield automatically adds its die (usually d4) to every attack roll. This die can be used for Gambits but not as the highest damage die if it\'s a shield.',
      'bonus-dice': 'Add extra dice from situational advantages — allies attacking the same target, special abilities, or other effects.',
      'damage-calc': 'Enter the <strong>raw damage</strong> you\'re taking (highest die from attack, plus any Bolster). Armour is subtracted automatically. The result reduces GD first, then VIG.',

      // --- Header Controls ---
      glory: '<strong>Glory</strong> measures your knight\'s renown. Earn it by resolving Myths, winning duels or tournaments, and victorious battles. Ranks: 0 Errant → 3 Gallant → 6 Tenant → 9 Dominant → 12 Radiant (City Quest).',
      age: 'Your knight\'s life stage: <strong>Young</strong>, <strong>Mature</strong>, or <strong>Old</strong>. Ages advance during seasonal progressions, bringing Virtue changes.',
      season: 'Current season: <strong>Spring</strong> (Green), <strong>Harvest</strong> (Gold), or <strong>Winter</strong> (Grey). Each Season brings a new Pursuit and seasonal events.',
      'export-char': 'Download this character as a JSON file. Use this to share characters or move them to another browser.',
      'import-char': 'Load a character from a JSON file. Works with single-character exports and full backup files.',
    };
```

- [ ] **Step 2: Quick-test in console**

Open `index.html` in a browser, open console:

```javascript
console.log(Object.keys(HELP_TEXT).length);
// Expected: ~45 entries
console.log(HELP_TEXT['virtues'].substring(0, 30));
// Expected: "Your three Virtues: <strong>VI"
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HELP_TEXT data object with ~45 tooltip entries"
```

---

### Task 3: Wire Help Buttons into Character Sheet Panels

**Files:**
- Modify: `index.html` — functions: `buildVirtuesPanel`, `buildGuardPanel`, `buildConditionsBanner`, `buildAbilityPassionCards`, `buildMountPanel`, `buildSquirePanel`, `buildRemediesPanel`, `buildContactsPanel`, `buildSeasonLog`, `buildScarHistory`, `buildJournal`

- [ ] **Step 1: Add help buttons to Virtues panel header + restore buttons**

In `buildVirtuesPanel()` (~line 3262), change:
```javascript
        <div class="panel-header">Virtues</div>
```
to:
```javascript
        <div class="panel-header">Virtues ${helpBtn('virtues')}</div>
```

In the restore button line (~line 3251), change:
```javascript
        rows += `<button class="btn btn-small" onclick="restoreVirtue('${v}')">Restore</button>`;
```
to:
```javascript
        rows += `<button class="btn btn-small" onclick="restoreVirtue('${v}')">Restore</button>${helpBtn('restore-virtue', true)}`;
```

- [ ] **Step 2: Add help buttons to Guard panel header + restore button**

In `buildGuardPanel()` (~line 3330), change:
```javascript
        <div class="panel-header">Guard</div>
```
to:
```javascript
        <div class="panel-header">Guard ${helpBtn('guard')}</div>
```

In the Restore GD button (~line 3323), change:
```javascript
      html += `<button class="btn btn-small" onclick="restoreGD()">Restore GD</button>`;
```
to:
```javascript
      html += `<button class="btn btn-small" onclick="restoreGD()">Restore GD</button>${helpBtn('restore-gd', true)}`;
```

- [ ] **Step 3: Add help buttons to Conditions panel + individual toggles**

In `buildConditionsBanner()` (~line 3398), change:
```javascript
        <div class="panel-header">Conditions</div>
```
to:
```javascript
        <div class="panel-header">Conditions ${helpBtn('conditions')}</div>
```

For each condition toggle (~line 3378), change:
```javascript
        tags += `<span class="condition-tag condition-toggle ${cls}" onclick="toggleCondition('${t.key}')">${t.label}</span>`;
```
to:
```javascript
        tags += `<span class="condition-tag condition-toggle ${cls}" onclick="toggleCondition('${t.key}')">${t.label}${helpBtn(t.key === 'doomScar' ? 'doom-scar' : t.key === 'mortalWound' ? 'mortal-wound' : t.key, true)}</span>`;
```

For each derived condition (~line 3394), change:
```javascript
        tags += `<span class="condition-tag condition-derived">${d}</span>`;
```
to:
```javascript
        const helpKey = d.toLowerCase();
        tags += `<span class="condition-tag condition-derived">${d}${helpBtn(helpKey, true)}</span>`;
```

- [ ] **Step 4: Add help buttons to Ability and Passion cards**

In `buildAbilityPassionCards()` (~line 3431), change:
```javascript
        <div class="panel-header">Ability</div>
```
to:
```javascript
        <div class="panel-header">Ability ${helpBtn('ability')}</div>
```

Change (~line 3440):
```javascript
        <div class="panel-header">Passion</div>
```
to:
```javascript
        <div class="panel-header">Passion ${helpBtn('passion')}</div>
```

Change the Indulge Passion button (~line 3444):
```javascript
          <button class="btn btn-small btn-gold" style="margin-top:8px" onclick="indulgePassion()">Indulge Passion</button>
```
to:
```javascript
          <button class="btn btn-small btn-gold" style="margin-top:8px" onclick="indulgePassion()">Indulge Passion</button>${helpBtn('indulge-passion', true)}
```

- [ ] **Step 5: Add help buttons to Mount panel**

In `buildMountPanel()`, find the two places where the Mount panel header is rendered (~lines 3489 and 3494):
```javascript
          <div class="panel-header">Mount</div>
```
Change both to:
```javascript
          <div class="panel-header">Mount ${helpBtn('mount')}</div>
```

- [ ] **Step 6: Add help buttons to Squire panel**

In `buildSquirePanel()`, find the two places where the Squire panel header is rendered (~lines 3638 and 3643):
```javascript
          <div class="panel-header">Squire</div>
```
Change both to:
```javascript
          <div class="panel-header">Squire ${helpBtn('squire')}</div>
```

- [ ] **Step 7: Add help buttons to Remedies, Contacts, Season Log, Scar History, Journal**

In `buildRemediesPanel()` (~line 3783):
```javascript
        <div class="panel-header">Remedies</div>
```
→
```javascript
        <div class="panel-header">Remedies ${helpBtn('remedies')}</div>
```

In `buildContactsPanel()` (~line 3820):
```javascript
        <div class="panel-header">Contacts</div>
```
→
```javascript
        <div class="panel-header">Contacts ${helpBtn('contacts')}</div>
```

In `buildSeasonLog()` (~line 3921):
```javascript
        <div class="panel-header">Season Log</div>
```
→
```javascript
        <div class="panel-header">Season Log ${helpBtn('season-log')}</div>
```

In `buildScarHistory()` (~line 4077):
```javascript
        <div class="panel-header">Scar History</div>
```
→
```javascript
        <div class="panel-header">Scar History ${helpBtn('scar-history')}</div>
```

In `buildJournal()` (~line 4119):
```javascript
        <div class="panel-header">Journal</div>
```
→
```javascript
        <div class="panel-header">Journal ${helpBtn('journal')}</div>
```

- [ ] **Step 8: Test Character Sheet tooltips**

Open `index.html` in a browser. On the Character Sheet tab:
1. Click the "?" next to "Virtues" — a popover should appear explaining VIG/CLA/SPI
2. Click anywhere to dismiss
3. Click "?" next to a Restore button — should explain remedy usage
4. Click "?" next to condition toggles — should explain each condition
5. Verify collapsible panels still work (clicking the header text, not the "?")

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: add help tooltips to all Character Sheet panels"
```

---

### Task 4: Wire Help Buttons into Equipment Tab Panels

**Files:**
- Modify: `index.html` — functions: `buildEquippedArmourPanel`, `buildEquippedWeaponsPanel`, `buildCarriedPanel`, `buildInventoryPanel`, `buildAddItemPanel`

- [ ] **Step 1: Add help buttons to equipment panel headers**

In `buildEquippedArmourPanel()` (~line 4256):
```javascript
      html += `<div class="panel-header">Armour</div>`;
```
→
```javascript
      html += `<div class="panel-header">Armour ${helpBtn('armour')}</div>`;
```

In `buildEquippedWeaponsPanel()` (~line 4294):
```javascript
      html += `<div class="panel-header">Weapons</div>`;
```
→
```javascript
      html += `<div class="panel-header">Weapons ${helpBtn('weapons')}</div>`;
```

In `buildCarriedPanel()` (~line 4332):
```javascript
      html += `<div class="panel-header">Carried (Unequipped)</div>`;
```
→
```javascript
      html += `<div class="panel-header">Carried (Unequipped) ${helpBtn('carried')}</div>`;
```

In `buildInventoryPanel()` (~line 4359):
```javascript
      html += `<div class="panel-header">Inventory</div>`;
```
→
```javascript
      html += `<div class="panel-header">Inventory ${helpBtn('inventory')}</div>`;
```

In `buildAddItemPanel()` (~line 4385):
```javascript
      html += `<div class="panel-header">Add Item</div>`;
```
→
```javascript
      html += `<div class="panel-header">Add Item ${helpBtn('add-item')}</div>`;
```

- [ ] **Step 2: Add help buttons to armour trapped + weapon impaired toggles**

In `buildEquippedArmourPanel()`, the trapped toggle (~line 4274):
```javascript
            html += `<span class="trapped-indicator"${trappedClass} onclick="toggleArmourTrapped(${i})" title="Toggle Trapped (Gambit)">${a.trapped ? 'TRAPPED' : 'Trap'}</span>`;
```
→
```javascript
            html += `<span class="trapped-indicator"${trappedClass} onclick="toggleArmourTrapped(${i})" title="Toggle Trapped (Gambit)">${a.trapped ? 'TRAPPED' : 'Trap'}</span>${helpBtn('armour-trapped', true)}`;
```

In `buildEquippedWeaponsPanel()`, the impaired toggle for active state (~line 4310):
```javascript
            html += `<span class="impaired-indicator" style="background:var(--danger);color:#fff" onclick="toggleWeaponImpaired(${i})" title="Toggle Impaired (Gambit)">IMPAIRED</span>`;
```
→
```javascript
            html += `<span class="impaired-indicator" style="background:var(--danger);color:#fff" onclick="toggleWeaponImpaired(${i})" title="Toggle Impaired (Gambit)">IMPAIRED</span>${helpBtn('weapon-impaired', true)}`;
```

And the inactive state (~line 4312):
```javascript
            html += `<span class="impaired-indicator" onclick="toggleWeaponImpaired(${i})" title="Toggle Impaired (Gambit)">Impair</span>`;
```
→
```javascript
            html += `<span class="impaired-indicator" onclick="toggleWeaponImpaired(${i})" title="Toggle Impaired (Gambit)">Impair</span>${helpBtn('weapon-impaired', true)}`;
```

- [ ] **Step 3: Test Equipment tab tooltips**

Open `index.html`, go to Equipment tab:
1. Click "?" next to "Armour" — should explain armour types and stacking
2. Click "?" next to "Trap" on a shield — should explain trapped shields
3. Click "?" next to "Impair" on a weapon — should explain impaired weapons
4. Verify all five panel headers have working "?" buttons

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add help tooltips to all Equipment tab panels"
```

---

### Task 5: Wire Help Buttons into Battle Tab Panels

**Files:**
- Modify: `index.html` — functions: `buildDicePoolPanel`, `buildAttackStepsPanel`, `buildFeatsPanel`, `buildGambitsPanel`, `buildDamageCalcPanel`, `buildMountedCombatPanel`

- [ ] **Step 1: Add help buttons to battle panel headers**

In `buildDicePoolPanel()` (~line 4808):
```javascript
      let html = '<div class="panel"><div class="panel-header">Your Attack</div><div class="panel-body">';
```
→
```javascript
      let html = '<div class="panel"><div class="panel-header">Your Attack ' + helpBtn('your-attack') + '</div><div class="panel-body">';
```

In `buildAttackStepsPanel()` (~line 5078):
```javascript
      html += '<div class="panel-header">Attack Steps</div>';
```
→
```javascript
      html += '<div class="panel-header">Attack Steps ' + helpBtn('attack-steps') + '</div>';
```

In `buildFeatsPanel()` (~line 5096):
```javascript
      html += '<div class="panel-header">Feats</div>';
```
→
```javascript
      html += '<div class="panel-header">Feats ' + helpBtn('feats') + '</div>';
```

In `buildGambitsPanel()` (~line 5165):
```javascript
      html += '<div class="panel-header">Gambits Reference</div>';
```
→
```javascript
      html += '<div class="panel-header">Gambits Reference ' + helpBtn('gambits-ref') + '</div>';
```

In `buildDamageCalcPanel()` (~line 5192):
```javascript
      html += '<div class="panel-header">Incoming Damage</div>';
```
→
```javascript
      html += '<div class="panel-header">Incoming Damage ' + helpBtn('incoming-damage') + '</div>';
```

In `buildMountedCombatPanel()` (~line 5321):
```javascript
      html += '<div class="panel-header">Mounted Combat</div>';
```
→
```javascript
      html += '<div class="panel-header">Mounted Combat ' + helpBtn('mounted-combat') + '</div>';
```

- [ ] **Step 2: Add help buttons to individual battle controls**

In `buildDicePoolPanel()`, the shield die line (~line 4828):
```javascript
        html += `<div style="margin-top:6px;font-size:0.9rem"><span style="color:var(--success)">&#9632;</span> Shield die: <strong>${shieldDice}</strong> (auto-included)</div>`;
```
→
```javascript
        html += `<div style="margin-top:6px;font-size:0.9rem"><span style="color:var(--success)">&#9632;</span> Shield die: <strong>${shieldDice}</strong> (auto-included)${helpBtn('shield-die', true)}</div>`;
```

The charging checkbox (~line 4838):
```javascript
          html += `<div class="weapon-check"><label><input type="checkbox" onchange="battleToggleCharge()"${chk}> Charging &mdash; adds ${trample} trample die</label></div>`;
```
→
```javascript
          html += `<div class="weapon-check"><label><input type="checkbox" onchange="battleToggleCharge()"${chk}> Charging &mdash; adds ${trample} trample die</label>${helpBtn('charging', true)}</div>`;
```

The bonus dice header (~line 4847):
```javascript
      html += '<strong style="font-size:0.85rem;text-transform:uppercase;letter-spacing:0.5px;color:var(--ink-light)">Bonus Dice</strong>';
```
→
```javascript
      html += '<strong style="font-size:0.85rem;text-transform:uppercase;letter-spacing:0.5px;color:var(--ink-light)">Bonus Dice</strong>' + helpBtn('bonus-dice', true);
```

The impaired toggle (~line 4868):
```javascript
      html += `<label style="font-size:0.9rem;cursor:pointer"><input type="checkbox" onchange="battleToggleImpaired()"${impChk}> <strong>Impaired</strong></label>`;
```
→
```javascript
      html += `<label style="font-size:0.9rem;cursor:pointer"><input type="checkbox" onchange="battleToggleImpaired()"${impChk}> <strong>Impaired</strong></label>${helpBtn('battle-impaired', true)}`;
```

- [ ] **Step 3: Add help buttons to individual feats**

In `buildFeatsPanel()`, each feat card (~line 5106):
```javascript
        html += `<div class="feat-name">${feat.name}</div>`;
```
→
```javascript
        html += `<div class="feat-name">${feat.name}${helpBtn('feat-' + key, true)}</div>`;
```

- [ ] **Step 4: Add help button to damage calculator**

In `buildDamageCalcPanel()` (~line 5195):
```javascript
      html += '<label style="font-size:0.9rem;font-weight:bold">Raw Damage:</label>';
```
→
```javascript
      html += '<label style="font-size:0.9rem;font-weight:bold">Raw Damage:' + helpBtn('damage-calc', true) + '</label>';
```

- [ ] **Step 5: Test Battle tab tooltips**

Open `index.html`, go to Battle tab:
1. Click "?" next to "Your Attack" — should explain dice pool mechanics
2. Click "?" next to each feat name — should explain the specific feat
3. Click "?" next to "Impaired" checkbox — should explain impaired state
4. Click "?" next to "Bonus Dice" — should explain bonus dice
5. Verify popover dismisses on click-away

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add help tooltips to all Battle tab panels and controls"
```

---

### Task 6: Wire Help Buttons into Header Controls

**Files:**
- Modify: `index.html` — function: `renderHeader`

- [ ] **Step 1: Add help buttons to header controls**

In `renderHeader()` (~line 5557-5558), the Export/Import buttons:
```javascript
      switcherHtml += ' <button class="btn btn-small" onclick="exportChar()" style="margin-left:8px">Export</button>';
      switcherHtml += ' <button class="btn btn-small" onclick="importChar()">Import</button>';
```
→
```javascript
      switcherHtml += ' <button class="btn btn-small" onclick="exportChar()" style="margin-left:8px">Export</button>' + helpBtn('export-char', true);
      switcherHtml += ' <button class="btn btn-small" onclick="importChar()">Import</button>' + helpBtn('import-char', true);
```

In `renderHeader()` (~line 5580-5582), the Glory display:
```javascript
          Glory: <button class="btn btn-small" onclick="adjustGlory(-1)">-</button>
          <strong id="glory-display">${currentChar.glory}</strong>
          <button class="btn btn-small" onclick="adjustGlory(1)">+</button>
```
→
```javascript
          Glory: <button class="btn btn-small" onclick="adjustGlory(-1)">-</button>
          <strong id="glory-display">${currentChar.glory}</strong>
          <button class="btn btn-small" onclick="adjustGlory(1)">+</button>${helpBtn('glory', true)}
```

The Age select (~line 5586):
```javascript
          Age: <select id="age-select" style="font-size:0.85rem">
```
→
```javascript
          Age:${helpBtn('age', true)} <select id="age-select" style="font-size:0.85rem">
```

The Season select (~line 5593):
```javascript
          Season: <select id="season-select" style="font-size:0.85rem">
```
→
```javascript
          Season:${helpBtn('season', true)} <select id="season-select" style="font-size:0.85rem">
```

- [ ] **Step 2: Test header tooltips**

Open `index.html`:
1. Click "?" next to Glory — should explain glory and rank thresholds
2. Click "?" next to Age — should explain life stages
3. Click "?" next to Season — should explain seasons
4. Click "?" next to Export/Import — should explain file operations

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add help tooltips to header controls (glory, age, season, export/import)"
```

---

### Task 7: Backup Indicator

**Files:**
- Modify: `index.html:1827-1830` (footer HTML)
- Modify: `index.html` (JS section — add backup functions, modify `updateChar`, modify `exportChar`)

- [ ] **Step 1: Add backup indicator CSS**

Insert in the CSS section, after the help tooltip styles, before `</style>`:

```css
    /* ===== Backup Indicator ===== */
    .backup-indicator {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 8px;
      margin-bottom: 8px;
      font-size: 0.8rem;
      font-style: normal;
    }

    .backup-status {
      display: inline-flex;
      align-items: center;
      gap: 4px;
    }

    .backup-dot {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      display: inline-block;
    }

    .backup-dot.green { background: var(--success); }
    .backup-dot.orange { background: var(--warning); }
    .backup-dot.red { background: var(--danger); }

    .backup-btn {
      font-family: var(--font-serif);
      font-size: 0.75rem;
      padding: 3px 10px;
      border: 1px solid var(--panel-border);
      border-radius: var(--radius);
      background: var(--parchment-dark);
      color: var(--ink);
      cursor: pointer;
    }

    .backup-btn:hover {
      background: var(--gold-light);
      color: #fff;
    }
```

- [ ] **Step 2: Modify footer HTML to include backup indicator**

Change the footer (~line 1827):
```html
    <footer style="text-align:center; padding:20px 10px 10px; font-size:0.75rem; color:var(--ink-light); font-style:italic; border-top:1px solid var(--panel-border); margin-top:30px;">
      Mythic Bastionland Character Tracker — Created by Eric Sack<br>
      <span style="font-size:0.7rem;">Mythic Bastionland &copy; Chris McDowall / Bastionland Press</span>
    </footer>
```
→
```html
    <footer style="text-align:center; padding:20px 10px 10px; font-size:0.75rem; color:var(--ink-light); font-style:italic; border-top:1px solid var(--panel-border); margin-top:30px;">
      <div id="backup-indicator" class="backup-indicator"></div>
      Mythic Bastionland Character Tracker — Created by Eric Sack<br>
      <span style="font-size:0.7rem;">Mythic Bastionland &copy; Chris McDowall / Bastionland Press</span>
    </footer>
```

- [ ] **Step 3: Add backup JS functions**

Insert in the JS section, after the help system and HELP_TEXT, before `const EQUIPMENT_DB`:

```javascript
    // === BACKUP SYSTEM ===

    function getBackupTimestamp() {
      return localStorage.getItem('mythicBastion_lastBackup');
    }

    function getChangeCount() {
      return parseInt(localStorage.getItem('mythicBastion_changeCount') || '0');
    }

    function incrementChangeCount() {
      const count = getChangeCount() + 1;
      localStorage.setItem('mythicBastion_changeCount', count.toString());
      renderBackupIndicator();
    }

    function backupAllCharacters() {
      const ids = getCharIds();
      const chars = ids.map(id => loadChar(id)).filter(Boolean);
      if (chars.length === 0) { alert('No characters to back up.'); return; }
      const data = JSON.stringify(chars, null, 2);
      const blob = new Blob([data], { type: 'application/json' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      const date = new Date().toISOString().split('T')[0];
      a.download = `mythic-bastionland-backup-${date}.json`;
      a.click();
      URL.revokeObjectURL(url);
      localStorage.setItem('mythicBastion_lastBackup', new Date().toISOString());
      localStorage.setItem('mythicBastion_changeCount', '0');
      renderBackupIndicator();
    }

    function getBackupAge(timestamp) {
      if (!timestamp) return null;
      const now = new Date();
      const then = new Date(timestamp);
      const diffMs = now - then;
      const diffMins = Math.floor(diffMs / 60000);
      const diffHours = Math.floor(diffMs / 3600000);
      const diffDays = Math.floor(diffMs / 86400000);
      if (diffMins < 1) return 'just now';
      if (diffMins < 60) return diffMins + 'm ago';
      if (diffHours < 24) return diffHours + 'h ago';
      return diffDays + 'd ago';
    }

    function getBackupState() {
      const timestamp = getBackupTimestamp();
      const changeCount = getChangeCount();
      if (!timestamp) {
        return changeCount < 5 ? 'orange' : 'red';
      }
      const diffMs = new Date() - new Date(timestamp);
      const diffDays = diffMs / 86400000;
      if (diffDays < 1) return 'green';
      if (diffDays < 7) return 'orange';
      return 'red';
    }

    function renderBackupIndicator() {
      const el = document.getElementById('backup-indicator');
      if (!el) return;
      const timestamp = getBackupTimestamp();
      const state = getBackupState();
      const age = timestamp ? getBackupAge(timestamp) : 'never';
      el.innerHTML = `
        <span class="backup-status">
          <span class="backup-dot ${state}"></span>
          Last backup: ${age}
        </span>
        <button class="backup-btn" onclick="backupAllCharacters()">Backup Now</button>
      `;
    }
```

- [ ] **Step 4: Hook incrementChangeCount into updateChar**

In `updateChar()` (~line 2986):
```javascript
    function updateChar(mutator) {
      mutator(currentChar);
      saveChar(currentChar);
      render();
    }
```
→
```javascript
    function updateChar(mutator) {
      mutator(currentChar);
      saveChar(currentChar);
      incrementChangeCount();
      render();
    }
```

- [ ] **Step 5: Update exportChar to record backup timestamp**

In `exportChar()` (~line 5662), add at the end before the closing `}`:
```javascript
      localStorage.setItem('mythicBastion_lastBackup', new Date().toISOString());
      localStorage.setItem('mythicBastion_changeCount', '0');
      renderBackupIndicator();
```

- [ ] **Step 6: Update importChar to handle backup arrays**

In `importChar()` (~line 5683), change the `reader.onload` callback:
```javascript
        reader.onload = (ev) => {
          try {
            const char = JSON.parse(ev.target.result);
            if (!char.name || !char.virtues || !char.gd) {
              alert('Invalid character file.');
              return;
            }
            char.id = generateId();
            registerCharId(char.id);
            saveChar(char);
            switchToChar(char.id);
            addJournalEntry('Character imported from file.', true);
          } catch (err) {
            alert('Failed to import: ' + err.message);
          }
        };
```
→
```javascript
        reader.onload = (ev) => {
          try {
            const parsed = JSON.parse(ev.target.result);
            const chars = Array.isArray(parsed) ? parsed : [parsed];
            let imported = 0;
            let lastId = null;
            chars.forEach(char => {
              if (!char.name || !char.virtues || !char.gd) return;
              char.id = generateId();
              registerCharId(char.id);
              saveChar(char);
              lastId = char.id;
              imported++;
            });
            if (imported === 0) {
              alert('No valid characters found in file.');
              return;
            }
            switchToChar(lastId);
            addJournalEntry(`Imported ${imported} character${imported > 1 ? 's' : ''} from file.`, true);
          } catch (err) {
            alert('Failed to import: ' + err.message);
          }
        };
```

- [ ] **Step 7: Call renderBackupIndicator on init**

In `init()` (~line 5705), add at the end before the closing `}`:
```javascript
      renderBackupIndicator();
```

- [ ] **Step 8: Test backup indicator**

Open `index.html` in browser:
1. Footer should show "Last backup: never" with orange or red dot
2. Click "Backup Now" — should download a JSON file, dot turns green, text shows "just now"
3. Make a few changes — dot/text should stay green (within 24h)
4. Import the backup file — all characters should load correctly

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat: add backup status indicator with one-click full backup"
```

---

### Task 8: Custom Knight Creation

**Files:**
- Modify: `index.html` — functions: `showNewCharacterModal`, `createCharFromModal`, `createDefaultChar`, `getKnightLabel`

- [ ] **Step 1: Add "Custom Knight" option to the dropdown**

In `showNewCharacterModal()` (~line 5413), change:
```javascript
      let optionsHtml = '<option value="">— Select a Knight —</option>';
```
→
```javascript
      let optionsHtml = '<option value="">— Select a Knight —</option>';
      optionsHtml += '<option value="custom">— Custom Knight —</option>';
```

- [ ] **Step 2: Add custom knight form fields to the modal**

In `showNewCharacterModal()` (~line 5447), after the GD input div and before the `modal-actions` div, add:
```javascript
          <div id="nc-custom-fields" style="display:none;margin-top:8px">
            <div class="flex-row" style="gap:8px">
              <div class="flex-col" style="flex:1">
                <label><strong>Ability Name</strong></label>
                <input type="text" id="nc-ability-name" placeholder="e.g. Battle Cry" style="width:100%">
              </div>
            </div>
            <div class="flex-col" style="margin-top:4px">
              <label><strong>Ability Description</strong></label>
              <textarea id="nc-ability-desc" placeholder="Describe what the ability does..." rows="2" style="width:100%;font-family:var(--font-body);font-size:0.9rem;padding:6px;border:1px solid var(--panel-border);border-radius:var(--radius)"></textarea>
            </div>
            <div class="flex-row" style="gap:8px;margin-top:8px">
              <div class="flex-col" style="flex:1">
                <label><strong>Passion Name</strong></label>
                <input type="text" id="nc-passion-name" placeholder="e.g. Justice" style="width:100%">
              </div>
            </div>
            <div class="flex-col" style="margin-top:4px">
              <label><strong>Passion Description</strong></label>
              <textarea id="nc-passion-desc" placeholder="Describe the passion..." rows="2" style="width:100%;font-family:var(--font-body);font-size:0.9rem;padding:6px;border:1px solid var(--panel-border);border-radius:var(--radius)"></textarea>
            </div>
          </div>
```

- [ ] **Step 3: Show/hide custom fields based on selection**

In the `setTimeout` callback of `showNewCharacterModal()` where the `change` event listener is added (~line 5462), modify the listener to also toggle custom fields:

Change:
```javascript
          sel.addEventListener('change', () => {
            const knight = KNIGHT_DB.find(k => k.id === sel.value);
            const preview = document.getElementById('nc-preview');
            if (!knight) { preview.innerHTML = ''; return; }
```
→
```javascript
          sel.addEventListener('change', () => {
            const customFields = document.getElementById('nc-custom-fields');
            const preview = document.getElementById('nc-preview');
            if (sel.value === 'custom') {
              if (customFields) customFields.style.display = 'block';
              preview.innerHTML = '<p style="font-size:0.85rem;color:var(--ink-light);margin-top:8px;font-style:italic">Custom knight — add equipment from the Equipment tab after creation.</p>';
              return;
            }
            if (customFields) customFields.style.display = 'none';
            const knight = KNIGHT_DB.find(k => k.id === sel.value);
            if (!knight) { preview.innerHTML = ''; return; }
```

- [ ] **Step 4: Modify createCharFromModal to handle custom knights**

In `createCharFromModal()` (~line 5500), change:
```javascript
      if (!knightId) { alert('Please select a knight type.'); return; }

      const char = createDefaultChar(knightId, name || 'New Knight');
```
→
```javascript
      if (!knightId) { alert('Please select a knight type.'); return; }

      const char = createDefaultChar(knightId === 'custom' ? 'custom' : knightId, name || 'New Knight');

      if (knightId === 'custom') {
        const abilityName = (document.getElementById('nc-ability-name')?.value || '').trim();
        const abilityDesc = (document.getElementById('nc-ability-desc')?.value || '').trim();
        const passionName = (document.getElementById('nc-passion-name')?.value || '').trim();
        const passionDesc = (document.getElementById('nc-passion-desc')?.value || '').trim();
        char.ability = { name: abilityName || 'Custom Ability', description: abilityDesc || '' };
        char.passion = { name: passionName || 'Custom Passion', description: passionDesc || '' };
      }
```

- [ ] **Step 5: Update getKnightLabel to handle custom**

In `getKnightLabel()` (~line 5632):
```javascript
    function getKnightLabel(knightId) {
      const knight = KNIGHT_DB.find(k => k.id === knightId);
      return knight ? `${knight.number}: ${knight.name}` : knightId;
    }
```
→
```javascript
    function getKnightLabel(knightId) {
      if (knightId === 'custom') return 'Custom Knight';
      const knight = KNIGHT_DB.find(k => k.id === knightId);
      return knight ? `${knight.number}: ${knight.name}` : knightId;
    }
```

- [ ] **Step 6: Test custom knight creation**

Open `index.html`:
1. Click "New" to open creation modal
2. Select "— Custom Knight —" from the dropdown
3. Ability/Passion text fields should appear
4. Fill in name, stats, ability, and passion
5. Click "Create Knight" — character should be created
6. Header should show "Custom Knight" instead of a knight number
7. Ability and Passion panels should show the custom text
8. Equipment should be empty (user adds manually)

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add custom knight creation with freeform ability and passion"
```

---

### Task 9: GitHub Pages Setup

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Create .gitignore**

```
# Exclude everything except the app and config
*

# Include the app
!index.html

# Include git config files
!.gitignore
!docs/
!docs/**

# Exclude OS files
.DS_Store
Thumbs.db
```

- [ ] **Step 2: Commit .gitignore**

```bash
git add .gitignore
git commit -m "chore: add .gitignore for GitHub Pages hosting"
```

- [ ] **Step 3: Create GitHub repo and push**

```bash
gh repo create mbCharSheet --public --description "Mythic Bastionland Character Tracker" --source=. --push
```

If `gh` is not installed or authenticated, do it manually:
1. Go to github.com → New Repository → name: `mbCharSheet`, public
2. Follow the "push an existing repository" instructions

- [ ] **Step 4: Enable GitHub Pages**

```bash
gh api repos/{owner}/mbCharSheet/pages -X POST -f source.branch=main -f source.path=/
```

Or manually: Repository Settings → Pages → Source: Deploy from branch → Branch: main, folder: / (root)

- [ ] **Step 5: Verify the site**

Visit `https://<username>.github.io/mbCharSheet/` — should show the character tracker.

- [ ] **Step 6: Commit any final adjustments**

If the URL needs to be noted anywhere or any adjustments are needed, commit them.
