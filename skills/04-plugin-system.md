# Plugin System

**Use this when** you're wiring up something process-global that only needs to exist once for the whole app's life — an analytics SDK, an ads library, a platform achievements hook. `pluginSystem` installs optional engine extensions at runtime; plugins can register services, wrap systems, or add global utilities.

## Usage

```typescript
import { pluginSystem, Plugin, PluginContext } from '@emptysock/engine';

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
- `pluginSystem` is a singleton — import it and use it directly, nothing to construct.
- Registering two plugins under the same name throws. Names are the whole identity here, so pick one that won't collide.
