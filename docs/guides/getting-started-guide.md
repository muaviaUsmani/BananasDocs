---
sidebar_position: 1
---

# Getting Started Guide

A comprehensive guide to help you build your first application with Bananas.

## Overview

In this guide, you'll learn how to:

- Set up a new Bananas project
- Configure your environment
- Build your first feature
- Deploy your application

## Step 1: Installation

First, install Bananas in your project:

```bash
npm install bananas
```

## Step 2: Create Your Configuration

Create a `bananas.config.js` file:

```javascript
module.exports = {
  mode: 'development',
  debug: true
};
```

## Step 3: Initialize Bananas

Create your main application file:

```typescript
import { Bananas } from 'bananas';

async function main() {
  // Create instance
  const bananas = new Bananas({
    mode: 'development'
  });

  // Initialize
  await bananas.initialize();
  console.log('Bananas is ready!');

  // Your code here
}

main().catch(console.error);
```

## Step 4: Build Your First Feature

Let's create a simple data processor:

```typescript
import { Bananas } from 'bananas';

async function processUserData(userData) {
  const bananas = new Bananas();
  await bananas.initialize();

  const result = await bananas.process({
    user: userData,
    transform: 'normalize'
  });

  return result.output;
}

// Use it
const processed = await processUserData({
  name: 'John Doe',
  email: 'john@example.com'
});

console.log(processed);
```

## Step 5: Testing

Test your implementation:

```typescript
import { Bananas } from 'bananas';
import { expect, test } from 'vitest';

test('processes data correctly', async () => {
  const bananas = new Bananas();
  await bananas.initialize();

  const result = await bananas.process({ test: 'data' });

  expect(result.success).toBe(true);
  expect(result.output).toBeDefined();
});
```

## Step 6: Production Build

Prepare for production:

1. Update your configuration:

```javascript
module.exports = {
  mode: 'production',
  debug: false
};
```

2. Build your project:

```bash
npm run build
```

3. Deploy to your preferred platform

## Next Steps

Now that you have a working application:

- Learn about [Advanced Features](./advanced-features.md)
- Explore the [API Reference](/docs/category/api-reference)
- Check out [Best Practices](./best-practices.md)

## Troubleshooting

### Common Issues

**Issue: Module not found**

Make sure you've installed all dependencies:

```bash
npm install
```

**Issue: Configuration not loading**

Verify your config file is in the project root and properly formatted.

## Getting Help

If you encounter issues:

- Check the [API Reference](/docs/category/api-reference)
- Visit [GitHub Issues](https://github.com/muaviaUsmani/BananasDocs/issues)
- Join the [Discussions](https://github.com/muaviaUsmani/BananasDocs/discussions)
