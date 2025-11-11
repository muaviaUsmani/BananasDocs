---
sidebar_position: 1
---

# Main API

Complete reference for the core Bananas API.

## Classes

### `Bananas`

The main class for interacting with Bananas.

```typescript
class Bananas {
  constructor(options?: BananasOptions);

  // Methods
  initialize(): Promise<void>;
  process(data: any): Promise<Result>;
  destroy(): void;
}
```

#### Constructor

Creates a new Bananas instance.

**Parameters:**

- `options` (optional) - Configuration options
  - `mode`: `'development' | 'production'` - Application mode
  - `debug`: `boolean` - Enable debug mode

**Example:**

```typescript
import { Bananas } from 'bananas';

const bananas = new Bananas({
  mode: 'production',
  debug: false
});
```

#### Methods

##### `initialize()`

Initializes the Bananas instance.

**Returns:** `Promise<void>`

**Example:**

```typescript
await bananas.initialize();
console.log('Bananas initialized!');
```

##### `process(data: any)`

Processes the provided data.

**Parameters:**

- `data` - Data to process

**Returns:** `Promise<Result>`

**Example:**

```typescript
const result = await bananas.process({
  input: 'test data'
});

console.log(result.output);
```

##### `destroy()`

Cleans up and destroys the instance.

**Returns:** `void`

**Example:**

```typescript
bananas.destroy();
```

## Interfaces

### `BananasOptions`

Configuration options for Bananas.

```typescript
interface BananasOptions {
  mode?: 'development' | 'production';
  debug?: boolean;
  port?: number;
}
```

### `Result`

Result object returned by process operations.

```typescript
interface Result {
  success: boolean;
  output: any;
  errors?: Error[];
}
```

## Type Aliases

### `Mode`

```typescript
type Mode = 'development' | 'production';
```

### `ProcessCallback`

```typescript
type ProcessCallback = (result: Result) => void;
```

## Related Topics

- [Configuration](/docs/core-concepts/configuration)
- [Guides](/docs/category/guides)
