## :triangular_flag_on_post: Why Our `pickErrorKey` Utility Is Risky — and How to Fix It

In our codebase, we’ve been using a utility function `pickErrorKey` to extract error messages from HTTP responses (typically Axios). While it looks neat and functional, it introduces a few subtle but **dangerous risks** that can make debugging harder or cause unexpected crashes in production.

Let’s walk through the problems, why they matter, and how we can improve them.

---

### :boom: The Original Code

```ts
export const pickErrorKey =
  (key: 'error_message' | 'error_details' = 'error_message') =>
  async (error: {
    request: any;
    response?: { data: Record<'error_message' | 'error_details', string> };
  }) => {
    if (error?.response?.data != null) {
      const { error_message: errorMessage, error_details: errorDetails } =
        error.response.data;

      switch (key) {
        case 'error_message':
          return await Promise.reject(errorMessage ?? 'An error occurred.');
        case 'error_details':
          return await Promise.reject(errorDetails ?? 'An error occurred.');
      }
    } else if (error.request) {
      throw new Error(`Unexpected request error`);
    } else {
      throw new Error(`Client error`);
    }
  };
```

---

## :rotating_light: Issue #1: Misuse of `await Promise.reject(...)`

### :x: Problem:

```ts
return await Promise.reject(...);
```

* This adds an **extra layer** to the async stack.
* **Rejecting a promise using `await` can swallow stack traces** or obscure error origin in some environments.

### :white_check_mark: Fix:

Just throw the error or return the rejected promise directly:

```ts
throw new Error(errorMessage ?? 'An error occurred.');
```

Or, if you prefer Promise-based:

```ts
return Promise.reject(errorMessage ?? 'An error occurred.');
```

---

## :rotating_light: Issue #2: Unsafe Destructuring of `error.response.data`

### :x: Problem:

```ts
const { error_message, error_details } = error.response.data;
```

If `data` is missing or malformed (e.g. `null` or a string), this will crash with:

```
TypeError: Cannot destructure property 'error_message' of 'undefined'
```

### :white_check_mark: Fix:

Destructure with fallback or validate the shape first:

```ts
const data = error.response?.data ?? {};
const errorMessage = data.error_message;
const errorDetails = data.error_details;
```

---

## :rotating_light: Issue #3: Assumes a Fixed Response Shape

Many APIs return unexpected formats — arrays, strings, nested errors — especially under failure. We were assuming:

```ts
response: { data: Record<ErrorKeys, string> }
```

This is risky. It should be defensive.

### :white_check_mark: Fix:

Validate the shape or use type guards:

```ts
if (typeof data !== 'object' || data === null) {
  throw new Error('Malformed error response');
}
```

---

## :rotating_light: Issue #4: Poor Error Context and Swallowed Details

These fallbacks:

```ts
throw new Error('Unexpected request error');
throw new Error('Client error');
```

...are not helpful for debugging or user support. They **discard valuable Axios error info** (e.g. `code`, `message`, `isAxiosError`, etc.).

### :white_check_mark: Fix:

Include more context in thrown errors:

```ts
if (error.request) {
  throw new Error(`No response received: ${error.message ?? 'Unknown request error'}`);
} else {
  throw new Error(`Unexpected client error: ${error.message ?? 'Unknown error'}`);
}
```

---

## :white_check_mark: Final Safe Version

Here’s the fully rewritten, safer version of `pickErrorKey`:

```ts
export const pickErrorKey =
  (key: 'error_message' | 'error_details' = 'error_message') =>
  (error: {
    request?: any;
    response?: { data?: Record<string, any> };
    message?: string;
  }) => {
    const data = error.response?.data ?? {};
    const errorMessage = data.error_message;
    const errorDetails = data.error_details;

    switch (key) {
      case 'error_message':
        throw new Error(errorMessage ?? 'An error occurred.');
      case 'error_details':
        throw new Error(errorDetails ?? 'An error occurred.');
    }

    if (error.request) {
      throw new Error(`No response received: ${error.message ?? 'Unknown request error'}`);
    }

    throw new Error(`Unexpected client error: ${error.message ?? 'Unknown error'}`);
  };
```

---

## :brain: Takeaways

* **Avoid `await Promise.reject(...)`** — it’s bad async hygiene.
* Always **validate response shapes defensively** before destructuring.
* **Preserve error context** to help debugging and support.
* Throw meaningful errors with actual content, not vague fallbacks.

By following these guidelines, we make our error handling more robust, debuggable, and predictable across all environments.
