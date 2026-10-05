# Plugin System

**Use this when** you're wiring up something process-global that only needs to exist once for the whole app's life — an analytics SDK, an ads library, a platform achievements hook. `PluginSystem` installs optional engine extensions at runtime; plugins provide named services other code can inject. Each `Game` has one, reachable as `ctx.plugins` inside a scene.

## Usage

```typescript
import { PluginSystem, type Plugin, type PluginContext } from '@emptysock/engine';

const pluginSystem = new PluginSystem();   // inside a scene, use ctx.plugins instead

const analyticsPlugin: Plugin = {
  name: 'analytics',
  version: '1.0.0',
  install(ctx: PluginContext): void {
    const tracker = new AnalyticsTracker();
    ctx.provide('analytics', tracker);
  },
  uninstall(): void {
    // cleanup
  },
};

await pluginSystem.register(analyticsPlugin);

// Anywhere in code:
const tracker = pluginSystem.inject<AnalyticsTracker>('analytics');
tracker?.trackEvent('level_complete', { level: 1 });

// Remove when done:
await pluginSystem.unregister('analytics');

console.log(pluginSystem.registeredPlugins); // ['analytics']
```

## Notes
- `install()` can be async (returns `Promise<void>`) if setup needs it.
- `PluginSystem` is not a module singleton: construct one, or use the instance on the lifecycle context (`ctx.plugins`).
- Registering a second plugin under an existing name logs a warning and is skipped. Names are the whole identity, so pick one that won't collide.
- `PluginContext` offers `provide(key, value)` and `inject(key)`; `Plugin` has `name`, optional `version`, `install(ctx)`, optional `uninstall()`.
