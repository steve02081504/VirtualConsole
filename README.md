# Virtual Console

[![npm version](https://img.shields.io/npm/v/@steve02081504/virtual-console.svg)](https://www.npmjs.com/package/@steve02081504/virtual-console)
[![GitHub issues](https://img.shields.io/github/issues/steve02081504/virtual-console)](https://github.com/steve02081504/virtual-console/issues)

A powerful and flexible virtual console for Node.js that allows you to capture, manipulate, and redirect terminal output. Built with modern asynchronous contexts (`AsyncLocalStorage`) for robust, concurrency-safe operations.

`VirtualConsole` is perfect for:

- **Testing:** Assert console output from your modules without polluting the test runner's output.
- **Logging Frameworks:** Create custom logging solutions that can buffer, format, or redirect logs.
- **CLI Tools:** Build interactive command-line interfaces with updatable status lines or progress bars.
- **Debugging:** Isolate and inspect output from specific parts of your application, even in highly concurrent scenarios.

---

## How It Works: The Global Console Proxy

**This library is designed for zero-refactoring integration.** Upon import, it replaces `globalThis.console` with a smart proxy that is aware of the asynchronous execution context.

1. **The Proxy:** The new `console` object is a proxy. When you call a method like `console.log()`, the proxy intercepts the call.

2. **`AsyncLocalStorage` for Isolation:** The proxy uses `AsyncLocalStorage` to look up the currently active `VirtualConsole` instance for the current async operation.

3. **Context-Aware Routing:**
    - If you have activated a `VirtualConsole` instance using `vc.hookAsyncContext()`, the proxy finds your instance and routes all `console` calls to it. This allows your instance to capture output, handle stateful methods like `freshLine`, or apply custom logic.
    - If no instance is active for the current context, the proxy forwards the call to a default, passthrough console that simply prints to the original terminal, just like the real `console` would.

This architecture means you **don't need to refactor your code to pass console instances around**. Just keep using the global `console` as you always have, and wrap the code you want to monitor in a hook.

**Enhanced Compatibility:** The proxy is designed to be a good citizen. If other libraries or your own code assign custom properties to the global `console` object (e.g., `console.myLogger = ...`), these properties are preserved and remain accessible.

## Features

- **Zero-Configuration Capturing:** Capture output from any module without changing its source code.
- **Concurrency-Safe Isolation:** Uses `AsyncLocalStorage` to guarantee that output from concurrent operations is captured independently and correctly.
- **Output Recording:** Captures all `stdout` and `stderr` output to a string property for easy inspection.
- **Real Console Passthrough:** Optionally, print to the actual console while also capturing.
- **Full TTY Emulation:** Behaves like a real TTY, inheriting properties like `columns`, `rows`, and color support from the base console. It also respects terminal resize events.
- **Updatable Lines (`freshLine`)**: A stateful method for creating overwritable lines, perfect for progress indicators. Requires a TTY with ANSI support.
- **Custom Error Handling:** Provide a dedicated handler for `Error` objects passed to `console.error`.
- **Extensible:** Can be integrated with other async-context-aware libraries via `setGlobalConsoleReflect`.

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

### Use Case: Custom Error Handling

You can provide a dedicated `error_handler` to process `Error` objects passed to `console.error`. This is useful for integrating with logging services or custom error-reporting frameworks.

```javascript
import { VirtualConsole } from '@steve02081504/virtual-console';

const reportedErrors = [];

const vc = new VirtualConsole({
  // This handler is called ONLY when a single Error object is passed to console.error
  error_handler: (err) => {
    console.log(`Reporting error: "${err.message}"`);
    reportedErrors.push(err);
  },
  realConsoleOutput: true,
});

await vc.hookAsyncContext(() => {
  console.error(new Error('Something went wrong!')); // Handled by error_handler
  console.error('A regular error message.'); // Not an Error object, logged normally
});

// Verify the custom handler was called
console.log('Reported errors:', reportedErrors.map(e => e.message));
```

## API Reference

### `new VirtualConsole(options?)`

Creates a new `VirtualConsole` instance.

- `options` `<object>`
  - `realConsoleOutput` `<boolean>`: If `true`, output is also sent to the base console. **Default:** `false`.
  - `recordOutput` `<boolean>`: If `true`, output is captured in the `outputs` property. **Default:** `true`.
  - `base_console` `<Console>`: The console instance for passthrough. **Default:** The original `global.console` before it was patched.
  - `error_handler` `<function(Error): void>`: A dedicated handler for `Error` objects passed to `console.error`. If `console.error` is called with a single argument that is an `instanceof Error`, this handler is invoked instead of the standard `console.error` logic. **Default:** `null`.
  - `supportsAnsi` `<boolean>`: Manually set ANSI support, which affects `freshLine`. **Default:** Inherited from `base_console`'s stream, or the result of the `supports-ansi` package.

### `virtualConsole.hookAsyncContext(fn?)`

Hooks the virtual console into an asynchronous context.

- **`hookAsyncContext(fn)`**: Runs `fn` in a new context. All `console` calls within `fn` (and any functions it `await`s) are routed to this instance. Returns a `Promise` that resolves with the return value of `fn`.
- **`hookAsyncContext()`**: Activates the instance for the *current* asynchronous context. Useful for "activating" a console and leaving it active for the remainder of an async flow (e.g., within middleware).

### `virtualConsole.outputs`

- `<string>`

A string containing all captured `stdout` and `stderr` output.

### `console.freshLine(id, ...args)`

*Note: This stateful method is available on the global `console` object but only works as intended inside a `hookAsyncContext`.*

Prints a line. If the previously printed line (within the same hook) had the same `id`, it overwrites the previous line using ANSI escape codes. This requires a TTY environment.

- `id` `<string>`: A unique identifier for the overwritable line.
- `...args` `<any>`: The content to print, same as `console.log()`.

### `virtualConsole.clear()`

Clears the captured `outputs` string. If `realConsoleOutput` is enabled, it also attempts to call `clear()` on the base console.

### `virtualConsole.error(...args)`

The standard `console.error` method. However, if an `error_handler` is configured in the constructor, and `error()` is called with a single argument that is an `instanceof Error`, the handler will be invoked with the error object.

## For Library Authors: `setGlobalConsoleReflect(...)`

`VirtualConsole` exposes its `AsyncLocalStorage`-based reflection mechanism. If you are building another library that also manages async context (e.g., a custom APM or transaction tracer), you can use `setGlobalConsoleReflect` to integrate `VirtualConsole`'s logic into your own context manager. This prevents the two libraries from fighting over control of the async context. See the source code for details on its function signature.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request on [GitHub](https://github.com/steve02081504/virtual-console).
