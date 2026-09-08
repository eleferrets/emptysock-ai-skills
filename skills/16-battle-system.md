# BattleSystem

`BattleSystem` is an opt-in turn-based RPG battle module. Import it only if your game uses battles — the module is tree-shaken out of bundles that never import it, so games without combat pay nothing in bundle size. It handles turn order, damage resolution, status effects, and event emission, leaving all presentation (UI, animation, sound) to your code.

---

## Database setup

Define skills and status effects once and load them into every `BattleSystem` instance you create.

```typescript
import {
  type BattleDatabase,
  type SkillDef,
  type StatusEffectDef,
} from '@emptysock/engine'

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
} from '@emptysock/engine'

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

battle.addPartyMember(hero)
battle.addEnemy(goblin)

// db can also be loaded separately after construction
battle.loadDatabase(db)

// --- Event subscription (returns unsubscribe) ---
const unsub = battle.onEvent((event: BattleEvent): void => {
  handleBattleEvent(event)
})

// --- Start ---
battle.start()
// fires: battle-start → round-start → action-needed (for first party member by speed)
```

When the player chooses an action, submit it:

```typescript
import { type BattleAction } from '@emptysock/engine'

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

## Custom damage formula

Override the built-in formula when your game uses a different damage model.

```typescript
import { BattleSystem } from '@emptysock/engine'

const battle = new BattleSystem()

// Arguments: atk, def, power, isCrit, critMultiplier
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
import { BattleSystem, SaveSystem, type Combatant, type BattlePhase } from '@emptysock/engine'
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

// --- Snapshot (call before transitioning away mid-battle) ---
async function saveBattle(slot: string): Promise<void> {
  const snapshot: BattleSave = {
    round: battle.getRound(),
    phase: battle.getPhase(),
    party: battle.getParty().map((c) => ({ ...c })),
    enemies: battle.getEnemies().map((c) => ({ ...c })),
  }
  await SaveSystem.save(slot, snapshot)
}

// --- Restore ---
async function resumeBattle(slot: string): Promise<BattleSystem | null> {
  const raw = await SaveSystem.load(slot)
  if (raw === null) return null

  const saved = BattleSaveSchema.parse(raw.data)

  const resumed = new BattleSystem({ db })
  saved.party.forEach((c) => resumed.addPartyMember(c as Combatant))
  saved.enemies.forEach((c) => resumed.addEnemy(c as Combatant))

  // Re-subscribe and start — the system fires battle-start from the beginning;
  // use saved.round and saved.phase to restore any UI state your scene manages.
  resumed.onEvent(handleBattleEvent)
  resumed.start()
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
