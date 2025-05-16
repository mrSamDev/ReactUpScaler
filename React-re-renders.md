## React Re-renders: A Friendly Guide to Props, Functions, and Speed

React is pretty smart about updating what you see on the screen, but sometimes it re-renders things more often than it needs to. This can slow your app down. If we get how React decides to re-render, especially with props (data we pass around) and functions, we can make our apps much faster and smoother for our users.

### The Main Culprit: React's "Quick Check" (Shallow Comparison)

So, when does React decide to re-render a part of your app (a component)? Usually, it's when its "props" (data passed into it) or its internal "state" (data it manages itself) changes. For props, React does a quick check called a "shallow comparison":

- **Simple Stuff (Numbers, Text, True/False):** For basic values like numbers (e.g., `5`), text (e.g., `"hello"`), or true/false, React just looks at the values. If it was `5` and it's still `5`, React knows nothing changed. Easy!
- **Complex Stuff (Objects & Arrays):** But for more complex things like objects (think `{ name: 'Sijo' }`) or arrays (like `[1, 2, 3]`), React looks at their **memory address**. Think of this as where they live in the computer's memory. If you create a _new_ object or array, even if it looks exactly the same as an old one, it gets a _new_ memory address. To React, this new address means "it changed!"

This "memory address check" is often why a component re-renders even if the data inside its props _looks_ identical to you.

**Example: The New Object Sneak Attack**

```jsx
import React, { useState } from "react";

// ChildComponent is wrapped in React.memo,
// meaning it tries *not* to re-render if its props don't change.
const ChildComponent = React.memo(({ data }) => {
  console.log("ChildComponent re-rendered because 'data' prop seemed to change");
  return <div>Value: {data.value}</div>;
});

function ParentComponent() {
  const [count, setCount] = useState(0);

  // The Problem: We make a brand new 'data' object every time ParentComponent renders.
  const data = { value: count };

  console.log("ParentComponent rendered");
  return (
    <div>
      <ChildComponent data={data} />
      <button onClick={() => setCount(count + 1)}>Increment Count</button>
      {/* This button also causes ParentComponent to re-render, making a new 'data' object */}
      <button onClick={() => setCount(count)}>Re-render Parent (same count)</button>
    </div>
  );
}

export default ParentComponent;
```

See how `ChildComponent` re-renders every time `ParentComponent` does, even if you click the "Re-render Parent (same count)" button? That's because we're creating a brand new `data` object (`{ value: count }`) every single time `ParentComponent` renders. Even if `count` is the same, that `data` object is fresh out of the oven with a new memory address. `React.memo` (which tries to stop unnecessary re-renders) sees this new address and thinks, "Oh, `data` changed! Better re-render `ChildComponent`."

### Functions Can Be Sneaky Too!

It's the same story with functions. If you create a function inside your component, React makes a new version of that function (with a new memory address) every time the component re-renders.

This matters a lot when you're:

1.  Using the `useEffect` hook (it checks its "dependencies" – the list of things to watch – for changes).
2.  Passing functions as props to other components (especially if those components are trying to avoid re-renders with `React.memo`).

**Example: `useEffect` and the Ever-Changing Function**

```jsx
import React, { useState, useEffect } from "react";

function MyComponent() {
  const [count, setCount] = useState(0);

  // Problem: A new 'logMessage' function is made every time MyComponent renders.
  const logMessage = () => {
    console.log(`Current count: ${count}`);
  };

  useEffect(() => {
    console.log("Effect re-ran because logMessage (the function itself) changed");
    logMessage();
  }, [logMessage]); // 'logMessage' is a dependency

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

export default MyComponent;
```

In this case, the `useEffect` hook runs again and again after every render. Why? Because `logMessage` is a brand new function every time `MyComponent` re-renders. Since `logMessage` is in `useEffect`'s dependency list, `useEffect` thinks something important changed (the function itself!) and runs its code again.

### The Fix: Telling React to "Remember That!"

Luckily, React gives us tools to tell it, "Hey, remember this value or function, and don't remake it unless these _specific things_ I tell you about change." This keeps their memory addresses stable.

1.  **`useMemo` for Objects and Arrays:**
    For objects and arrays, use `useMemo`. It's like saying, "React, figure out this object/array, but only do the work again if these specific values [the dependencies] change. Otherwise, just give me the one you remembered from last time."

    ```jsx
    // In ParentComponent from the first example:
    import React, { useState, useMemo } from "react";
    // ... ChildComponent definition ...

    function ParentComponent() {
      const [count, setCount] = useState(0);

      // Solution: 'data' object is only re-created if 'count' actually changes.
      const data = useMemo(() => ({ value: count }), [count]); // Only remakes if 'count' changes

      console.log("ParentComponent rendered");
      return (
        <div>
          <ChildComponent data={data} />
          <button onClick={() => setCount(count + 1)}>Increment Count</button>
          <button onClick={() => setCount(count)}>Re-render Parent (same count)</button>
        </div>
      );
    }
    ```

    Now, `ChildComponent` will only re-render if `count` _actually_ changes, because the `data` prop's memory address stays the same otherwise. Sweet!

2.  **`useCallback` for Functions:**
    For functions, use `useCallback`. It tells React, "Remember this function. Don't create a new version of it unless these specific values [the dependencies] change." This is super important for functions you pass to other components or use in `useEffect`.

    ```jsx
    // In MyComponent from the second example:
    import React, { useState, useEffect, useCallback } from "react";

    function MyComponent() {
      const [count, setCount] = useState(0);

      // Solution: 'logMessage' function is "remembered."
      // It only gets re-created if 'count' changes.
      const logMessage = useCallback(() => {
        console.log(`Current count: ${count}`);
      }, [count]);

      // Tip: For setting state based on previous state, do this:
      const increment = useCallback(() => {
        setCount((prevCount) => prevCount + 1);
        // No need to list 'count' as a dependency here!
      }, []); // Empty array means: make this function once and remember it forever.

      useEffect(() => {
        console.log("Effect ran");
        logMessage();
      }, [logMessage]); // Now, effect only runs if 'logMessage' *really* changes (i.e., if 'count' changed).

      return (
        <div>
          <p>Count: {count}</p>
          <button onClick={increment}>Increment</button>
        </div>
      );
    }
    ```

### More Tips for Happy Rendering

- **Tell Components to Chill with `React.memo`:** You can wrap your components in `React.memo()`. This tells the component, "Only re-render if your props _actually_ change (based on that quick check)." It works best when the props you give it (like objects or functions) are stable, thanks to `useMemo` and `useCallback`.

  - For extra control, `React.memo` can take a special function you write to decide if props changed, but use this carefully as it can be tricky and sometimes slower if not done right.

- **Move Things Out:** If a function or some data doesn't need anything from _inside_ your component (like its state or props), just define it _outside_ the component. It'll only be created once when your app loads, not every render.

- **Smart State Updates:** When you update state based on its previous value (like incrementing a counter), use the version of `setState` that takes a function: `setCount(prevCount => prevCount + 1)`. This can sometimes help you avoid needing to list that state in `useCallback`'s dependencies, keeping your functions more stable.

- **Keys for Lists are a MUST:** When you show a list of things (like a list of names), always give each item a unique and stable `key` prop. Think of it like a name tag for each item. It helps React figure out what changed, what's new, or what's gone, without messing up or re-rendering the whole list unnecessarily.

- **Don't Change Things Directly (Immutability):** It's a good habit not to change objects or arrays directly after you create them. Instead, if you need to change something, create a _new_ copy with your changes. This "immutability" makes it super easy for React to see that something changed because, remember, new copy = new memory address!

### How React Figures Out What to Update (The Gist of Reconciliation)

So, how does React actually use all this to update the screen? It's a process with a fancy name: **reconciliation**. Here's the basic idea:

1.  **The Virtual Sketchpad (Virtual DOM):** When something changes (like state or props), React first creates a new sketch of your UI in its memory. This is called the Virtual DOM. It's like a blueprint of what the screen _should_ look like.
2.  **Spot the Difference:** Then, React compares this new sketch with the old one to see what's different. It's pretty smart about this:
    - If you change a `<div>` to a `<p>`, React just scraps the old thing and builds the new one.
    - If it's the same type of thing (like a `<div>` is still a `<div>`), React just updates what changed inside it (like text or styles). For your own components, it re-runs them if their props changed (and this is where `React.memo`, `useMemo`, and `useCallback` help React know if they _really_ changed!). Then it does the same for all the things inside that component.
    - For lists, those `key` props we talked about are super important here so React can efficiently update just the items that changed, added, or left.
3.  **Update in One Go:** React collects all the changes it found and updates the actual webpage in one efficient batch. This is faster than doing lots of tiny updates one by one.

Using `useMemo` and `useCallback` helps this "spot the difference" game a lot. If a prop's memory address hasn't changed, React can quickly say, "Nope, nothing new here for this prop!" and might skip re-rendering a whole component.

### Finding and Fixing Sneaky Re-renders

Okay, so how do you find these unnecessary re-renders if you suspect they're happening?

1.  **React DevTools Profiler:** This is your best friend. It's usually a tab in your browser's developer tools (if you have the React DevTools extension installed). It lets you record what your app is doing. It shows you which components re-rendered, _why_ they re-rendered (did props or state change?), and how much time they took.
2.  **Good Ol' `console.log`:** Don't underestimate putting a `console.log('MyComponent rendered!')` inside your components. It's a quick and dirty way to see if they're rendering when you don't expect them to.
3.  **`react-scan` for a Visual (Advanced):** There's also a tool called `react-scan` (you'd install it separately) that can give you a nice visual of what's rendering and help you see exactly which props or state changes are causing it. It can help you:
    - See who's re-rendering and how often.
    - Figure out the cause.
    - Focus your fixing efforts where they'll do the most good.

### Wrapping Up

So, to stop your React app from re-rendering too much, the main trick is to understand that React gets a bit jumpy with new objects, arrays, and functions (because of their ever-changing memory addresses). Using tools like `React.memo` (for components), `useMemo` (for objects/arrays), and `useCallback` (for functions) helps keep things stable and tells React, "Only re-render if something _truly_ important changed."

Don't go overboard trying to optimize everything from the start! But keeping an eye on how you create and pass props (especially objects and functions) can make your app feel much snappier. And remember, always use tools like the Profiler to find the _real_ slow spots before you start changing lots of code. Happy coding!
