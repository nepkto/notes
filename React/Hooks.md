# React Hooks: A Detailed Explanation with Scenarios and Use Cases

---

## 1. What Are Hooks in React?

**Hooks** are special functions introduced in **React v16.8** that let you use state, lifecycle, and other React features in **function components**—which was previously only possible with class components.

> **React Hooks were introduced in React version 16.8 (released February 2019).**

---

## 2. Why Were Hooks Introduced?  
### (The Motivation)

**Problems with Class Components (Before Hooks):**
- **State only in classes:** You had to use class components to use state or lifecycle methods.
- **Logic scattering:** Related logic (e.g., data fetching) was spread across different lifecycle methods (`componentDidMount`, `componentDidUpdate`).
- **Difficult logic reuse:** Sharing stateful logic between components required complicated patterns like **Higher-Order Components (HOC)** or **Render Props**—both adding complexity.
- **Confusing classes:** JavaScript classes, `this` binding, and the component lifecycle were often sources of mistakes and difficult for newcomers.

**Hooks** solved all these by allowing stateful and side effect logic in normal JavaScript functions, no classes or awkward patterns needed.

---

## 3. Stateful Features Without Classes or Complex Workarounds

- _Before hooks_: To manage state, effects, or context, your components **had to be classes**, and if you wanted to reuse/compose logic you needed HOCs or render props.
- _With hooks_: **Function components** can do everything class components can do — including state, effects, refs, context, and more — with **simple function calls**, no classes, no HOCs, and no render props.

---

## 4. The Hooks API: Basic Types

- **`useState`**: Local state variable in function component
- **`useEffect`**: Run code for side effects (fetching, subscriptions, updating the DOM)
- **`useContext`**: Access React context data
- **Advanced hooks:** `useReducer`, `useRef`, `useMemo`, `useCallback`, `useLayoutEffect`, `useImperativeHandle`
- **Custom hooks:** Your own `useX` functions, reusing stateful logic

---

## 5. BEFORE & AFTER: Real Scenarios

### A) Managing State & Updating Title

#### Class Component (Before Hooks)
```jsx
import React from 'react';

class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }

  componentDidMount() {
    document.title = `Count: ${this.state.count}`;
  }

  componentDidUpdate() {
    document.title = `Count: ${this.state.count}`;
  }

  increment = () => {
    this.setState({ count: this.state.count + 1 });
  }

  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.increment}>Increment</button>
      </div>
    );
  }
}
```

**Drawbacks:**
- Boilerplate: constructor, `this` binding, etc.
- Logic is **scattered** across methods.
- Reuse is awkward—need HOCs/Render Props.
- Classes are **harder to read and maintain.**

---

#### Function Component (With Hooks)
```jsx
import React, { useState, useEffect } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```
**Benefits:**
- No class, just a function.
- State and related logic live **together**.
- Easy to read, maintain, and extend.
- Logic can be **abstracted** into custom hooks for reuse.

---

### B) Reusing Logic: Track Window Width

#### BEFORE: With Classes (and HOC)
```jsx
function withWindowWidth(Component) {
  return class extends React.Component {
    state = { width: window.innerWidth };
    handleResize = () => this.setState({ width: window.innerWidth });

    componentDidMount() {
      window.addEventListener('resize', this.handleResize);
    }
    componentWillUnmount() {
      window.removeEventListener('resize', this.handleResize);
    }

    render() {
      return <Component width={this.state.width} {...this.props} />;
    }
  };
}
```
- HOCs add **complexity** and nesting.

---

#### WITH HOOKS: Custom Hook
```jsx
import { useState, useEffect } from 'react';

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return width;
}

// Usage in a functional component:
function MyComponent() {
  const width = useWindowWidth();
  return <div>Window width: {width}px</div>;
}
```
**Benefits:**
- No wrapper components or classes.
- Logic is reusable **everywhere** with a single function call.
- Cleaner and easier to compose.

---

## 6. Common Hooks With Use Cases

### `useState`
To add local state.
```jsx
const [value, setValue] = useState(initialValue);
```

### `useEffect`
To perform side effects (fetch, event listeners, timers, update DOM, cleanup).
```jsx
useEffect(() => {
  // Side effect code
  return () => {/* cleanup */}
}, [dependencies]);
```

### `useContext`
To read values from context providers.
```jsx
const theme = useContext(ThemeContext);
```

---

## 7. Custom Hooks

Custom hooks let you extract component logic into reusable functions:

```jsx
function useDocumentTitle(title) {
  useEffect(() => {
    document.title = title;
  }, [title]);
}

// Use it in any function component
function Page() {
  useDocumentTitle('Welcome!');
  // ...
}
```

---

## 8. Summary Table

| Problem Before Hooks                  | Solution With Hooks                      |
|---------------------------------------|------------------------------------------|
| Must use classes for state/effects    | Use regular functions everywhere         |
| Logic & lifecycle scattered           | Keep related logic together (per hook)   |
| Difficult logic reuse                 | Extract into (custom) hooks, reuse easily|
| Complex patterns (HOC, render props)  | Simple, flat, readable custom hooks      |


---

## 9. References

- [Introducing Hooks (React Official)](https://react.dev/learn/introducing-react-hooks)
- [Hooks API Reference](https://react.dev/reference/react)
- [Rules of Hooks](https://react.dev/reference/react/hooks)

---

## 10. Summary

- Hooks let you use **all of React's stateful features** without classes or boilerplate.
- They **improve code clarity, modularity, and logic reuse**.
- You can now build complex, stateful, effectful components with just functions and custom hooks.
- **Hooks were introduced in React version 16.8.**

