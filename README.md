# Virtual Console

[![npm version](https://img.shields.io/npm/v/@steve02081504/virtual-console.svg)](https://www.npmjs.com/package/@steve02081504/virtual-console)
[![GitHub license](https://img.shields.io/github/license/steve02081504/virtual-console)](https://github.com/steve02081504/virtual-console/blob/main/LICENSE)
[![GitHub issues](https://img.shields.io/github/issues/steve02081504/virtual-console)](https://github.com/steve02081504/virtual-console/issues)

A powerful and flexible virtual console for Node.js that allows you to capture, manipulate, and redirect terminal output. Built with modern asynchronous contexts (`AsyncLocalStorage`) for robust, concurrency-safe operations.

`VirtualConsole` is perfect for:

- **Testing:** Assert console output from your modules without polluting the test runner's output.
- **Logging Frameworks:** Create custom logging solutions that can buffer, format, or redirect logs.
- **CLI Tools:** Build interactive command-line interfaces with updatable status lines or progress bars.
- **Debugging:** Isolate and inspect output from specific parts of your application, even in highly concurrent scenarios.

---

## Core Concept: The Global Console Proxy

**This library is designed for zero-refactoring integration.** Upon import, it replaces `globalThis.console` with a smart proxy. Here’s how it works:

1. **Default Behavior (No-op):** By default, the proxy simply forwards all console calls (`console.log`, etc.) to the original, real console. It doesn't record or change anything. This makes the library safe to include anywhere without side effects.

2. **Activation via `hookAsyncContext`:** To capture or manipulate output, you create a `new VirtualConsole()` instance and activate it for a specific block of code using `vc.hookAsyncContext(yourFunction)`.

3. **Context-Aware Routing:** Inside the `hookAsyncContext` block, the proxy detects the active `VirtualConsole` instance and routes all `console` calls to it. This allows your instance to capture output, handle stateful methods like `freshLine`, or apply custom logic, all while being completely isolated from other asynchronous operations.

This architecture means you **don't need to pass console instances around**. Just keep using the global `console` as you always have, and wrap the code you want to monitor.

## Features

- **Zero-Configuration Capturing:** Capture output from any module without changing its source code.
- **Concurrency-Safe Isolation:** Uses `AsyncLocalStorage` to guarantee that output from concurrent operations is captured independently and correctly.
- **Output Recording:** Captures all `stdout` and `stderr` output to a string property for inspection.
- **Real Console Passthrough:** Optionally, print to the actual console while also capturing.
- **TTY Emulation:** Behaves like a real TTY, inheriting properties like `columns`, `rows`, and color support.
- **Updatable Lines (`freshLine`)**: A stateful method for creating overwritable lines, perfect for progress indicators.
- **Extensible:** Provide a custom base console, define a dedicated `Error` handler, and more.

## Installation

```bash
npm install @steve02081504/virtual-console
```

## Quick Start: Testing Console Output

The most common use case is testing. Wrap your function call in `hookAsyncContext` and then assert the captured output.

```javascript
import { VirtualConsole } from '@steve02081504/virtual-console';
import { strict as assert } from 'node:assert';

// 1. A function that logs to the console
function greet(name) {
  console.log(`Hello, ${name}!`);
  console.error('An example error.');
}

// 2. In your test:
async function testGreeting() {
  const vc = new VirtualConsole();

  // 3. Run the function inside the hook to capture its output.
  // All `console.*` calls inside `greet` are now routed to `vc`.
  await vc.hookAsyncContext(() => greet('World'));

  // 4. Assert the captured output
  const expectedOutput = 'Hello, World!\nAn example error.\n';
  assert.strictEqual(vc.outputs, expectedOutput);
  console.log('Test passed!');
}

testGreeting();
```

## Advanced Usage

### Use Case: Concurrent Progress Bars with `freshLine`

This example demonstrates the power of async context isolation. We run two progress updates concurrently. Each `hookAsyncContext` creates an isolated "session," ensuring that each `console.freshLine` call updates its own line without interfering with the other.

```javascript
import { VirtualConsole } from '@steve02081504/virtual-console';

const vc = new VirtualConsole({ realConsoleOutput: true });

async function updateProgress(taskName, duration) {
  for (let i = 0; i <= 100; i += 20) {
    // `console.freshLine` is a stateful method. It works correctly here
    // because each task has its own isolated VirtualConsole state.
    console.freshLine(taskName, `[${taskName}]: ${i}%`);
    await new Promise(res => setTimeout(res, duration));
  }
  console.log(`[${taskName}]: Done!`);
}

console.log('Starting concurrent tasks...');

// Run two tasks, each in its own isolated hook.
// Without this isolation, they would conflict and corrupt the output.
await Promise.all([
  vc.hookAsyncContext(() => updateProgress('Upload-A', 50)),
  vc.hookAsyncContext(() => updateProgress('Process-B', 75)),
]);

console.log('All tasks finished!');
```

### Use Case: Activating a Hook for an Entire Async Flow

You can call `hookAsyncContext()` without a function to activate an instance for the *rest of the current asynchronous context*. This is useful in frameworks like Express or Koa middleware.

```javascript
const vc = new VirtualConsole({ realConsoleOutput: true });

async function middleware(next) {
  console.log('Middleware: Activating virtual console for this request.');
  // Activate `vc` for the rest of this async flow
  vc.hookAsyncContext();
  
  // Now, `next()` and any subsequent code in this request's
  // async chain will have its console output handled by `vc`.
  await next();
}

async function handler() {
  console.log('Handler: This output is being captured.');
}

// Simulate a request
await middleware(handler);
```

## API Reference

### `new VirtualConsole(options?)`

Creates a new `VirtualConsole` instance.

- `options` `<object>`
  - `realConsoleOutput` `<boolean>`: If `true`, output is also sent to the base console. **Default:** `false`.
  - `recordOutput` `<boolean>`: If `true`, output is captured in the `outputs` property. **Default:** `true`.
  - `base_console` `<Console>`: The console instance for passthrough. **Default:** The original `global.console`.
  - `error_handler` `<function(Error): void>`: A dedicated handler for `Error` objects passed to `console.error`.
  - `supportsAnsi` `<boolean>`: Manually set ANSI support. **Default:** Inherited from `base_console`.

### `virtualConsole.hookAsyncContext(fn?)`

Hooks the virtual console into an asynchronous context.

- **`hookAsyncContext(fn)`**: Runs `fn` in a new context. All `console` calls within `fn` (and any functions it `await`s) are routed to this instance. Returns the result of `fn`.
- **`hookAsyncContext()`**: Activates the instance for the *current* asynchronous context. Useful for "activating" a console and leaving it active for the remainder of an async flow (e.g., within middleware).

### `virtualConsole.outputs`

- `<string>`

A string containing all captured `stdout` and `stderr` output.

### `console.freshLine(id, ...args)`

*Note: This stateful method is available on the global `console` object but only works as intended inside a `hookAsyncContext`.*

Prints a line. If the previously printed line (within the same hook) had the same `id`, it overwrites the previous line.

- `id` `<string>`: A unique identifier for the overwritable line.
- `...args` `<any>`: The content to print, same as `console.log()`.

### `virtualConsole.clear()`

Clears the captured `outputs` string. If `realConsoleOutput` is enabled, it also attempts to call `clear()` on the base console.

### For Library Authors: `setGlobalConsoleReflect(...)`

`VirtualConsole` exposes its `AsyncLocalStorage`-based reflection mechanism. If you are building another library that manages async context, you can use `setGlobalConsoleReflect` to integrate `VirtualConsole`'s logic into your own context manager, preventing conflicts. See the source code for details.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request on [GitHub](https://github.com/steve02081504/VirtualConsole).
