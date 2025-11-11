---
sidebar_position: 2
---

# Configuration

Learn how to configure Bananas to match your project requirements.

## Configuration File

Create a `bananas.config.js` file in your project root:

```javascript
module.exports = {
  // Basic settings
  mode: 'development',

  // Feature flags
  features: {
    featureA: true,
    featureB: false
  },

  // Advanced options
  plugins: [
    'plugin-name'
  ]
};
```

## TypeScript Configuration

For TypeScript projects, use `bananas.config.ts`:

```typescript
import type { BananasConfig } from 'bananas';

const config: BananasConfig = {
  mode: 'production',
  features: {
    featureA: true,
    featureB: true
  }
};

export default config;
```

## Environment Variables

Bananas supports environment-based configuration:

```bash
# .env.local
BANANAS_MODE=production
BANANAS_API_KEY=your-api-key
BANANAS_DEBUG=false
```

## Configuration Options

### Core Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `mode` | `string` | `'development'` | Application mode |
| `debug` | `boolean` | `false` | Enable debug logging |
| `port` | `number` | `3000` | Server port |

### Feature Flags

Enable or disable features as needed:

- `featureA` - Description of feature A
- `featureB` - Description of feature B
- `featureC` - Description of feature C

## Best Practices

1. Keep configuration files in version control
2. Use environment variables for sensitive data
3. Document custom configurations
4. Test configurations in different environments

## Related Topics

- [Architecture Overview](./architecture.md)
- [Getting Started](/docs/intro)
