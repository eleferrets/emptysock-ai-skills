# Plugin System

Install optional engine extensions at runtime. Plugins can register services, wrap systems, or add global utilities.

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
- `install()` may be async (returns `Promise<void>`).
- `pluginSystem` is a singleton — import and use directly, no instantiation needed.
- Registering a plugin with a duplicate name throws.
