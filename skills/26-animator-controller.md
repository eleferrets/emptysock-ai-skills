# AnimatorController

`AnimatorController` is a code-first animation state machine component: named states (each backed by an `AnimationClip`), transitions gated by parameters or triggers, and optional cross-fade blending between states. It is a separate component from `Animator`, which is unchanged — `Animator` stays the minimal single-clip player for straightforward flipbook playback; reach for `AnimatorController` when animation depends on named states and parameters (e.g. `idle`/`run`/`jump` gated on a speed float and a grounded bool).

---

## Setup

```typescript
import { AnimatorController } from '@emptysock/engine'

const anim = new AnimatorController()
player.addComponent(anim)

anim.addState('idle', idleClip)
anim.addState('run', runClip)
anim.addState('jump', jumpClip)

anim.addTransition('idle', { to: 'run', condition: (ctx) => (ctx.getParam('speed') as number) > 0.1 })
anim.addTransition('run', { to: 'idle', condition: (ctx) => (ctx.getParam('speed') as number) <= 0.1 })
anim.addTransition('*', { to: 'jump', condition: (ctx) => ctx.isTriggered('jump'), duration: 0.1 })
anim.addTransition('jump', { to: 'idle', condition: (ctx) => ctx.getParam('grounded') === true })

anim.play('idle')
```

`'*'` as the `from` state matches a transition from any current state — useful for interrupts like a jump that can fire during idle or run.

---

## Driving parameters each frame

```typescript
override onUpdate(dt: number): void {
  const controller = this.entity.requireComponent(CharacterController)
  anim.setFloat('speed', Math.abs(controller.velocityX))
  anim.setBool('grounded', controller.isGrounded())

  if (this._input.isKeyPressed('Space')) anim.setTrigger('jump')

  anim.update(dt)   // engine calls this automatically as a Component; only call directly in a custom test harness
}
```

A trigger set with `setTrigger()` stays armed until a transition condition consumes it (or `resetTrigger()` clears it) — it does not auto-clear on its own until the next `update()` pass evaluates transitions.

---

## Reading the current pose

```typescript
for (const frame of anim.getActiveClips()) {
  // frame.state, frame.clip, frame.frame, frame.weight
  // Normally one entry with weight 1. During a cross-fade (duration > 0
  // on the transition that fired), two entries whose weights sum to 1 and
  // interpolate linearly over the transition's duration.
}

anim.currentState   // string | null
anim.isBlending      // boolean
anim.speed = 1.5     // playback speed multiplier
```

---

## Rules

| Wrong | Right |
|---|---|
| Branching on raw animation frame numbers in gameplay code | Query `anim.currentState` or gate logic on the same parameters the transitions use |
| Expecting `Animator`'s API on `AnimatorController` (or vice versa) | They are separate components — pick one per entity based on whether you need states |
| Calling `anim.play()` every frame to "hold" a state | `play()` is for a one-time jump (e.g. initial setup); let transitions handle ongoing state changes |
| Forgetting `duration` on an interrupt transition | Without it, the cut is instant — set `duration` for a cross-fade |
