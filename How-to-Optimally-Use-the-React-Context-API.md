# How to Optimally Use the React Context API (and Validate Context Items)

## Overview

React Context API is a powerful way to share state and logic across your component tree. However, improper use can lead to unnecessary re-renders and performance issues. This guide covers best practices for **optimizing Context API usage** and **validating context items** to ensure robust, maintainable code.

---

## 1. Creating Context

Define your context with a clear shape (object or value).

```jsx
import React, { createContext, useContext, useState } from "react";

const AuthContext = createContext(null);
```

---

## 2. Providing Context

Wrap your app (or subtree) with the provider.  
**Best Practice:**

- Only wrap the part of the tree that needs the context.
- Memoize values to avoid unnecessary re-renders.

```jsx
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  // Memoize the context value
  const value = React.useMemo(() => ({ user, setUser }), [user]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

---

## 3. Consuming Context

Use the `useContext` hook to access context values.

```jsx
function Profile() {
  const { user } = useContext(AuthContext);
  return <div>Hello, {user?.name}</div>;
}
```

---

## 4. Avoiding Unnecessary Re-renders

### Why Do Context Consumers Re-render?

Whenever the value passed to a Context Provider changes, **all components consuming that context will re-render**. This is by design, but can lead to performance issues if not managed carefully.

### Strategies to Optimize Context Usage

#### 1. **Memoize the Context Value**

If your context value is an object or array, always memoize it with `useMemo`.  
This ensures the value only changes when its dependencies change.

```jsx
const value = React.useMemo(() => ({ user, setUser }), [user]);
```

#### 2. **Split Contexts for Unrelated Data**

Don’t put unrelated or frequently changing data in the same context.  
For example, keep authentication and theme in separate contexts.

```jsx
<AuthProvider>
  <ThemeProvider>
    <App />
  </ThemeProvider>
</AuthProvider>
```

#### 3. **Avoid Putting Rapidly Changing Values in Context**

Context is best for app-wide state that changes infrequently (e.g., user, theme, locale).  
Avoid using context for things like form inputs, animation state, or scroll position.

#### 4. **Selector Pattern (Advanced)**

For large apps, consider using a selector pattern (like [use-context-selector](https://github.com/dai-shi/use-context-selector)) to only re-render components that use specific parts of the context.

#### 5. **Provider Placement**

Only wrap the part of your component tree that actually needs the context.  
This limits the number of components that re-render when the context value changes.

---

## 5. Validating Context Items

To ensure consumers don’t access undefined or incorrect context, especially outside a provider, use a custom hook that throws if the context is missing.

```jsx
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

---

## 6. Example: Full Pattern

```jsx
// AuthContext.js
import React, { createContext, useContext, useState } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = React.useMemo(() => ({ user, setUser }), [user]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

---

## 7. Summary of Best Practices

- **Memoize** context values to prevent unnecessary re-renders.
- **Split** context for unrelated or frequently changing data.
- **Validate** context usage with custom hooks.
- **Don’t overuse context** for rapidly changing values.
- **Consider advanced patterns** (like selectors) for large apps.

---

## References

- [No, React Context is Not Causing Too Many Renders](https://blacksheepcode.com/posts/no_react_context_is_not_causing_too_many_renders)
- [React Context Docs](https://react.dev/reference/react/createContext)
