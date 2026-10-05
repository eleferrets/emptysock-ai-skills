# BattleSystem

**Use this when** you're building turn-based combat — an RPG battle screen, a card-battle loop, anything with turn order and damage resolution.

`BattleSystem` lives in the optional `@emptysock/battle` package, not the core engine — a game with no combat never imports it and pays nothing for it. It handles turn order, damage resolution, status effects, and event emission, leaving all presentation (UI, animation, sound) to your code.

---

## Database setup

Define skills and status effects once and load them into every `BattleSystem` instance you create.

```typescript
import {
  type BattleDatabase,
  type SkillDef,
  type StatusEffectDef,
} from '@emptysock/battle'

const db: BattleDatabase = {
  skills: [
    {
      id: 'fireball',
      name: 'Fireball',
      mpCost: 8,
      targetType: 'single-enemy',
      formula: 'magical',
      power: 1.4,
    },
    {
      id: 'heal',
      name: 'Heal',
      mpCost: 6,
      targetType: 'single-ally',
      formula: 'fixed',
      power: 80,        // flat 80 HP
      isHeal: true,
    },
    {
      id: 'slash',
      name: 'Slash',
      mpCost: 0,
      targetType: 'single-enemy',
      formula: 'physical',
      power: 1,
      statusEffect: { effectId: 'bleed', chance: 0.2 },
    },
  ] satisfies SkillDef[],

  statusEffects: [
    {
      id: 'bleed',
      name: 'Bleeding',
      hpDrainPercentPerTurn: 0.05,
    },
    {
      id: 'poison',
      name: 'Poison',
      hpDrainPercentPerTurn: 0.08,
    },
    {
      id: 'weakened',
      name: 'Weakened',
      attackMultiplier: 0.6,
    },
  ] satisfies StatusEffectDef[],
}
```

---

## Wiring up the battle

Build combatants, create the system, load the database, subscribe to events, then call `start()`.

```typescript
import {
  BattleSystem,
  type BattleSystemOptions,
  type Combatant,
  type BattleEvent,
} from '@emptysock/battle'

// --- Combatants ---
const hero: Combatant = {
  id: 'hero',
  name: 'Hero',
  isParty: true,
  stats: { hp: 120, maxHp: 120, mp: 40, maxMp: 40, attack: 30, defense: 12, speed: 14, luck: 5 },
  statusEffects: [],
}

const goblin: Combatant = {
  id: 'goblin-1',
  name: 'Goblin',
  isParty: false,
  stats: { hp: 60, maxHp: 60, mp: 0, maxMp: 0, attack: 18, defense: 6, speed: 10, luck: 2 },
  statusEffects: [],
}

// --- System setup ---
const opts: BattleSystemOptions = {
  db,
  critChance: 0.0625,     // default
  critMultiplier: 1.5,    // default
  fleeChance: 0.5,        // default
}

const battle = new BattleSystem(opts)

// db can also be loaded separately after construction
battle.loadDatabase(db)

// --- Event subscription (returns unsubscribe) ---
const unsub = battle.subscribe((event: BattleEvent): void => {
  handleBattleEvent(event)
})

// --- Start — party and enemy roster passed together, replacing any previous battle ---
battle.start([hero], [goblin])
// fires: battle-start → round-start → action-needed (for first party member by speed)
```

When the player chooses an action, submit it:

```typescript
import { type BattleAction } from '@emptysock/battle'

// Basic attack
function onAttackButton(targetId: string): void {
  if (battle.getPhase() !== 'input') return
  const action: BattleAction = { type: 'attack', targetId }
  battle.submitAction('hero', action)
}

// Skill use
function onSkillButton(skillId: string, targetId: string): void {
  if (battle.getPhase() !== 'input') return
  const action: BattleAction = { type: 'skill', skillId, targetId }
  battle.submitAction('hero', action)
}

// Flee attempt
function onFleeButton(): void {
  if (battle.getPhase() !== 'input') return
  battle.submitAction('hero', { type: 'flee' })
}
```

---

## Event handling

React to `BattleEvent` to drive your UI. All events arrive synchronously during resolution.

```typescript
function handleBattleEvent(event: BattleEvent): void {
  if (event.kind === 'battle-start') {
    showBattleUI()
    return
  }

  if (event.kind === 'round-start') {
    ui.setRoundLabel(`Round ${event.round}`)
    return
  }

  if (event.kind === 'action-needed') {
    // Only fires for party members — enemies act automatically
    const c = battle.getCombatant(event.combatantId)
    if (c !== undefined) {
      showActionMenu(c)
    }
    return
  }

  if (event.kind === 'damage') {
    spawnDamageNumber(event.targetId, event.amount, event.isCrit)
    refreshHpBar(event.targetId)
    return
  }

  if (event.kind === 'heal') {
    spawnHealNumber(event.targetId, event.amount)
    refreshHpBar(event.targetId)
    return
  }

  if (event.kind === 'mp-cost') {
    refreshMpBar(event.combatantId)
    return
  }

  if (event.kind === 'status-applied') {
    ui.showStatusIcon(event.combatantId, event.effectId, event.name)
    return
  }

  if (event.kind === 'status-expired') {
    ui.hideStatusIcon(event.combatantId, event.effectId)
    return
  }

  if (event.kind === 'combatant-defeated') {
    playDeathAnimation(event.combatantId)
    return
  }

  if (event.kind === 'victory') {
    hideBattleUI()
    showVictoryScreen(battle.getEnemies())
    return
  }

  if (event.kind === 'defeat') {
    showGameOverScreen()
    return
  }

  if (event.kind === 'fled') {
    hideBattleUI()
    leaveBattleScene()   // your own scene transition
    return
  }
}
```

---

## Status effects

Apply a status effect through a skill's `statusEffect` field. The engine checks the `chance` roll each time the skill hits and applies `effectId` from the database on success.

```typescript
// Database entry
const poisonDef: StatusEffectDef = {
  id: 'poison',
  name: 'Poison',
  hpDrainPercentPerTurn: 0.08,  // 8 % of max HP drained at turn start
}

// Skill that may apply it
const toxicDart: SkillDef = {
  id: 'toxic-dart',
  name: 'Toxic Dart',
  mpCost: 4,
  targetType: 'single-enemy',
  formula: 'physical',
  power: 0.6,
  statusEffect: { effectId: 'poison', chance: 0.45 },
}
```

A status effect applied by a skill's `statusEffect` is created with `turnsRemaining: -1` (permanent for the battle) and is not re-applied if already active. To give an effect a duration, put a `StatusEffect` with a positive `turnsRemaining` on a `Combatant.statusEffects` array when calling `start()`; the system decrements it each turn and fires `status-expired` at zero. (`-1` never expires.)

---

## Stats and `statMap`

`BattleStats` requires `hp`, `maxHp`, `mp`, `maxMp`, and allows any extra numeric keys. Tell `BattleSystem` which keys play the attack/defense/speed/luck roles with the `statMap` option (each defaults to the canonical name):

```typescript
import { BattleSystem, type Combatant } from '@emptysock/battle'

const battle = new BattleSystem({
  db,
  statMap: { attack: 'atk', defense: 'def', speed: 'spd', luck: 'lck' },
})

const hero: Combatant = {
  id: 'hero', name: 'Hero', isParty: true,
  stats: { hp: 120, maxHp: 120, mp: 40, maxMp: 40, atk: 30, def: 12, spd: 14, lck: 5, spellPower: 25 },
  statusEffects: [],
}
```

Extra keys (`spellPower` above) are yours to read from a custom formula via `ctx.attacker.stats`.

---

## Skill `power` semantics

`power` means different things per `formula`:

| `formula` | Result |
|-----------|--------|
| `'physical'` | The (replaceable) physical formula; default `max(1, floor((effAtk - effDef / 2) * power * (crit ? critMultiplier : 1)))`. Basic attacks use `power = 1`. |
| `'magical'` | `max(1, floor((effAtk * 1.5 - effDef * 0.5) * power * crit))` |
| `'fixed'` | `floor(power)` flat amount |
| `'percent-max-hp'` | `floor(target.maxHp * power)`; use a fraction such as `0.25` |

`isHeal: true` applies the computed amount as healing instead of damage. If the actor lacks the MP for a skill, the action fails silently with no event.

---

## Custom damage formula

`setDamageFormula(fn)` replaces the formula used for basic attacks and `'physical'` skills only (not `'magical'`, `'fixed'`, or `'percent-max-hp'`). Call it before `start()`.

```typescript
import { type DamageContext } from '@emptysock/battle'

// DamageContext: { attacker: Combatant; target: Combatant; effectiveAttack: number;
//   effectiveDefense: number; power: number; isCrit: boolean; critMultiplier: number }
battle.setDamageFormula((ctx: DamageContext): number => {
  const sp = ctx.attacker.stats['spellPower'] ?? 0
  const base = Math.max(1, (ctx.effectiveAttack + sp) * ctx.power - ctx.effectiveDefense / 2)
  return Math.floor(ctx.isCrit ? base * ctx.critMultiplier : base)
})
```

`effectiveAttack`/`effectiveDefense` already include status multipliers. Return the final integer damage.

---

## Save and resume

`BattleSystem` has no snapshot API. To persist a battle, store the roster yourself (read it with `getParty()` / `getEnemies()` / `getRound()`) in your own save data, and rebuild with `start(party, enemies)` on load; `start()` replays `battle-start` and begins at round 1. Persist through a `StorageAdapter` (see `skills/33-save-system.md`) or a component-based `SaveSystem`.

```typescript
import { BattleSystem, type Combatant } from '@emptysock/battle'

async function saveBattle(adapter: StorageAdapter, battle: BattleSystem): Promise<void> {
  await adapter.set('battle_save', JSON.stringify({
    party: battle.getParty(),
    enemies: battle.getEnemies(),
  }))
}

async function resumeBattle(adapter: StorageAdapter, db: BattleDatabase): Promise<BattleSystem | null> {
  const raw = await adapter.get('battle_save')
  if (raw === null) return null
  const saved = JSON.parse(raw) as { party: Combatant[]; enemies: Combatant[] }   // validate with a schema in real code
  const resumed = new BattleSystem({ db })
  resumed.subscribe(handleBattleEvent)
  resumed.start(saved.party, saved.enemies)
  return resumed
}
```

(`StorageAdapter` and `BattleDatabase` are imported from `@emptysock/engine` and `@emptysock/battle` respectively.)

---

## Rules

| Wrong | Right |
|---|---|
| `async onUpdate() { battle.submitAction(...) }` | Submit actions from UI callbacks, never from `onUpdate` |
| Forgetting `battle.destroy()` on scene unload | Call `battle.destroy()` in the scene's `onUnload` |
| `JSON.parse(raw) as SavedBattle` | Validate loaded save data with a schema before starting a battle |
| `battle.submitAction('goblin-1', action)` | Only call `submitAction` for party members (`isParty: true`) |
| Calling `submitAction` while `battle.getPhase() !== 'input'` | Guard with `if (battle.getPhase() !== 'input') return` |
