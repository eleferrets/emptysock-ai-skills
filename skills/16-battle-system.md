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
      power: 140,
    },
    {
      id: 'heal',
      name: 'Heal',
      mpCost: 6,
      targetType: 'single-ally',
      formula: 'fixed',
      power: 80,
      isHeal: true,
    },
    {
      id: 'slash',
      name: 'Slash',
      mpCost: 0,
      targetType: 'single-enemy',
      formula: 'physical',
      power: 100,
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
    SceneManager.pop()
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
  power: 60,
  statusEffect: { effectId: 'poison', chance: 0.45 },
}
```

`turnsRemaining` of -1 in `StatusEffect` means the effect is permanent until explicitly removed via a skill that clears it. The system decrements `turnsRemaining` at each turn start and fires `status-expired` when it reaches zero.

---

## User-defined stats

`BattleSystem` accepts any stat names beyond the built-in set. Define extras in the `stats` object of each `Combatant` and reference them in a custom damage formula or event handler. The built-in formula only reads `attack`, `defense`, `hp`, `maxHp`, `mp`, `maxMp`, `speed`, and `luck` — any additional keys are yours to use.

```typescript
import { type Combatant } from '@emptysock/battle'

// Built-in stats plus three custom stats
const hero: Combatant = {
  id: 'hero',
  name: 'Hero',
  isParty: true,
  stats: {
    hp: 120, maxHp: 120,
    mp: 40,  maxMp: 40,
    attack: 30, defense: 12, speed: 14, luck: 5,
    // user-defined:
    agility: 18,
    spellPower: 25,
    ward: 8,
  },
  statusEffects: [],
}

// Access custom stats in a formula or event handler:
battle.setDamageFormula(
  (atk, def, power, isCrit, critMult, context): number => {
    const sp = (context.attacker.stats['spellPower'] ?? 0) as number
    const base = Math.max(1, (atk + sp) * power / 100 - def / 2)
    return isCrit ? Math.floor(base * critMult) : Math.floor(base)
  },
)
```

The `context` argument passed to the formula is `DamageContext`:

```typescript
import { type DamageContext } from '@emptysock/battle'

// DamageContext shape:
// {
//   attacker: Combatant,
//   defender: Combatant,
//   skill: SkillDef | null,   // null for basic attacks
// }
```

Stat keys are `string`-indexed; always guard with `?? 0` when reading user-defined keys so the formula stays safe against version drift or missing fields.

---

## Custom damage formula

Override the built-in formula when your game uses a different damage model.

```typescript
import { BattleSystem } from '@emptysock/battle'

const battle = new BattleSystem()

// Arguments: atk, def, power, isCrit, critMultiplier, context
battle.setDamageFormula(
  (atk: number, def: number, power: number, isCrit: boolean, critMultiplier: number): number => {
    const base = Math.max(1, (atk * power) / 100 - def / 2)
    return isCrit ? Math.floor(base * critMultiplier) : Math.floor(base)
  },
)
```

Call `setDamageFormula` before `start()`. The callback receives raw numbers; return the final integer damage (or healing) amount.

---

## Save and resume

Snapshot mid-battle state using `SaveSystem` and restore it on load. Always validate with a Zod schema before applying anything to a live `BattleSystem`.

```typescript
import { SaveSystem } from '@emptysock/engine'
import { BattleSystem, type Combatant, type BattlePhase } from '@emptysock/battle'
import { z } from 'zod'

// --- Zod schema ---
const StatsSchema = z.object({
  hp: z.number(), maxHp: z.number(),
  mp: z.number(), maxMp: z.number(),
  attack: z.number(), defense: z.number(), speed: z.number(), luck: z.number(),
})

const StatusEffectSchema = z.object({
  id: z.string(),
  name: z.string(),
  turnsRemaining: z.number(),
})

const CombatantSchema = z.object({
  id: z.string(),
  name: z.string(),
  stats: StatsSchema,
  statusEffects: z.array(StatusEffectSchema),
  isParty: z.boolean(),
})

const BattleSaveSchema = z.object({
  round: z.number(),
  phase: z.enum(['idle', 'input', 'resolving', 'victory', 'defeat']),
  party: z.array(CombatantSchema),
  enemies: z.array(CombatantSchema),
})

type BattleSave = z.infer<typeof BattleSaveSchema>

const battleSaves = new SaveSystem<{ id: string; data: BattleSave }>(
  'battle_save_',
  z.object({ id: z.string(), data: BattleSaveSchema }),
)

// --- Snapshot (call before transitioning away mid-battle) ---
function saveBattle(slot: string): void {
  const snapshot: BattleSave = {
    round: battle.getRound(),
    phase: battle.getPhase(),
    party: battle.getParty().map((c) => ({ ...c })),
    enemies: battle.getEnemies().map((c) => ({ ...c })),
  }
  battleSaves.save(slot, { data: snapshot })
}

// --- Restore ---
function resumeBattle(slot: string): BattleSystem | null {
  const raw = battleSaves.load(slot)
  if (raw === null) return null

  const saved = raw.data

  const resumed = new BattleSystem({ db })

  // Re-subscribe and start — the system fires battle-start from the beginning;
  // use saved.round and saved.phase to restore any UI state your scene manages.
  resumed.subscribe(handleBattleEvent)
  resumed.start(saved.party as Combatant[], saved.enemies as Combatant[])
  return resumed
}
```

---

## Rules

| Wrong | Right |
|---|---|
| `async onUpdate() { battle.submitAction(...) }` | Submit actions from UI callbacks, never from `onUpdate` |
| Forgetting `battle.destroy()` in `onDestroy` | Always call `battle.destroy()` in `onDestroy` |
| `raw.data as BattleSave` | `BattleSaveSchema.parse(raw.data)` — always validate save data |
| `battle.submitAction('goblin-1', action)` | Only call `submitAction` for party members (`isParty: true`) |
| Calling `submitAction` while `battle.getPhase() !== 'input'` | Guard with `if (battle.getPhase() !== 'input') return` |
