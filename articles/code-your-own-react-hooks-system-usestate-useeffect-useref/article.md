---
title: "Every React Developer Uses Hooks. Almost None Can Explain How They Work."
description: A from-scratch build of React's hooks system reveals that the rules aren't arbitrary — they're inevitable consequences of a linked list with a cursor.
tags: [react, hooks, javascript, internals, deep-dive]
date: 2026-04-08
---

My friend Marcos has been writing React for nearly three years. Production dashboards, complex state, the whole thing. One evening, over coffee, hooks came up.

"Do you know how hooks actually work under the hood?" I asked. Not as a quiz — genuinely curious.

The look on his face wasn't embarrassment. It was just honest: "Not really, no."

I thought about it. Neither did I — not really. I knew the rules. I'd followed them for years. But *why* they existed? I'd accepted them as received wisdom and moved on.

That's not unusual. Ask any React developer how hooks work internally and you'll get the same look. Not because developers are careless — because nobody explains it, and the official docs don't try.

This article does, by building the hooks system from scratch.

By the end, you'll understand not just what the rules are, but why they couldn't be anything else. The rules exist because of a specific data structure. Build the data structure, and the rules stop being rules. They become physics.

---

## The Rules You Follow Without Knowing Why

React ships with two rules about hooks. You've almost certainly seen them:

1. Only call hooks at the top level — never inside loops, conditions, or nested functions.
2. Only call hooks from React function components or custom hooks — not from regular JavaScript functions.

These rules are enforced by an ESLint plugin — `eslint-plugin-react-hooks` — that ships alongside React itself. The rule `react-hooks/rules-of-hooks` will mark violations in red before your code even runs. React doesn't just recommend these rules. It built tooling to prevent you from breaking them.

And yet, ask a developer *why* these rules exist and you'll get vague answers: "React needs hooks to run in the same order" or "it's just how React works." True — but that's a circular reference, not an explanation.

The confusion shows up in practice. The State of React 2025 survey found that `useEffect` is used by 98% of React developers. It also has the highest frustration rate of any hook: 37% of developers report issues, with the dependency array as the #1 pain point.

The dependency array problem isn't a `useEffect` problem. It's a symptom of not understanding how the hooks system works underneath. Understand the system, and the dependency array stops being mysterious.

So: why do these rules exist? Build the system and the answer becomes obvious.

---

## One Global Variable (And Why It Breaks)

Let's start building. The simplest possible `useState` needs to do three things: store a value, return it, and provide a way to update it.

```javascript
let state;

function useState(initialValue) {
  state = state ?? initialValue;

  function setState(newValue) {
    state = newValue;
    rerender();
  }

  return [state, setState];
}
```

Wire up a minimal component:

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  return `Count: ${count}`;
}

function rerender() {
  console.log(Counter());
}

console.log(Counter()); // Count: 0
```

It works. One hook, one global variable — the state persists between renders, the setter triggers a re-render, the value updates correctly.

Now add a second hook:

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName]   = useState('Alice');
  return `${name}: ${count}`;
}
```

Both calls hit the same `state`. The second `useState('Alice')` finds `state` already set to `0` from the first call, skips initialization, and returns `[0, setter]`. Both hooks share one slot. They overwrite each other.

This isn't a bug you can patch — it's a structural problem. One variable holds one value. Two hooks need two storage locations.

The fix is obvious once you see it: instead of one variable, use a list. One slot per hook, filled in order. That change is small. The consequences are everything.

> *Note: this implementation is intentionally incomplete — we're building in layers. The next version fixes the multi-hook problem.*

---

## The Array Fix: Where the Rules Come From

Replace the single variable with an array and add a cursor — an index that tracks which hook is being called:

```javascript
let hooks = [];
let cursor = 0;

function useState(initialValue) {
  const index = cursor;
  hooks[index] = hooks[index] ?? initialValue;

  function setState(newValue) {
    hooks[index] = newValue;
    rerender();
  }

  cursor++;
  return [hooks[index], setState];
}

function rerender() {
  cursor = 0; // reset before each render
  console.log(Counter());
}
```

The critical line is `cursor = 0` inside `rerender`. Before every render, the cursor goes back to the start. The first `useState` call always gets slot 0, the second always gets slot 1, the third always gets slot 2. The array persists between renders; the cursor resets.

Now two hooks work correctly:

```javascript
function Counter() {
  const [count, setCount] = useState(0);      // reads hooks[0]
  const [name,  setName]  = useState('Alice'); // reads hooks[1]
  return `${name}: ${count}`;
}

cursor = 0;
console.log(Counter()); // Alice: 0
```

Each call gets its own slot. State is independent. This version works.

### Where the first rule comes from

Now put the first hook inside a condition:

```javascript
function Counter() {
  if (someCondition) {
    const [count, setCount] = useState(0); // only runs sometimes
  }
  const [name, setName] = useState('Alice');
  return name;
}
```

Walk through two renders:

```
Render 1 (someCondition = true):
  useState(0)       → cursor 0 → hooks[0] = 0       → cursor: 1
  useState('Alice') → cursor 1 → hooks[1] = 'Alice'  → cursor: 2

Render 2 (someCondition = false):
  useState('Alice') → cursor 0 → hooks[0] = 0       ← reads count's slot!
```

![diagram-01](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-react-hooks-system-usestate-useeffect-useref/images/diagram-01.png)

*When a hook is skipped, every subsequent hook reads the wrong slot.*

On the second render, `useState('Alice')` lands at cursor 0 — the slot that belongs to `count`. It reads `0` instead of `'Alice'`. The hooks didn't move. The cursor did.

This is why you can't call hooks conditionally. Not because React decided to make a rule — but because the cursor can't know a hook was skipped. From the array's perspective, something just disappeared from the middle of the list.

The same logic applies to loops and nested functions: any structure that might cause a hook to run a different number of times across renders will shift the cursor, and every hook after it will read the wrong slot.

The linter isn't guessing at intent. It's enforcing the only constraint the data structure can survive.

---

## The Dispatcher: Why Hooks Can't Live Outside Components

The array model explains the first rule. But it doesn't explain the second: why can't you call hooks outside a React function component?

The problem isn't the array itself — it's ownership. Each component instance needs its own separate storage. `useState` in `<Counter />` and `useState` in `<UserProfile />` can't share the same array. React needs a way to know, at the moment a hook is called, which component's storage to use.

The solution is a dispatcher: a global object that React sets before calling your component and clears immediately after. In the source it lives as the `H` field on [`ReactSharedInternals`](https://github.com/facebook/react/blob/404b38c764cf86e6f2ec42f873bb33ce114256d3/packages/react/src/ReactSharedInternalsClient.js#L61):

```javascript
const ReactSharedInternals = {
  H: null, // the active hooks dispatcher
  // ...
};
```

React maintains two sets of hook implementations — one for the first render, one for every re-render. Both are defined at the bottom of [`ReactFiberHooks.js`](https://github.com/facebook/react/blob/404b38c764cf86e6f2ec42f873bb33ce114256d3/packages/react-reconciler/src/ReactFiberHooks.js#L3898):

```javascript
const HooksDispatcherOnMount = {  // line 3898
  useState: mountState,
  // useEffect: mountEffect, etc.
};

const HooksDispatcherOnUpdate = { // line 3926
  useState: updateState,
  // useEffect: updateEffect, etc.
};
```

![diagram-02](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-react-hooks-system-usestate-useeffect-useref/images/diagram-02.png)

*The dispatcher exists only during a render. Outside it, H is null — and any hook call throws.*

Before calling your component, React sets the dispatcher via [`renderWithHooks`](https://github.com/facebook/react/blob/404b38c764cf86e6f2ec42f873bb33ce114256d3/packages/react-reconciler/src/ReactFiberHooks.js#L502) at line 502:

```javascript
function renderWithHooks(Component, isFirstRender) {
  ReactSharedInternals.H = isFirstRender
    ? HooksDispatcherOnMount
    : HooksDispatcherOnUpdate;

  const result = Component(); // your component runs here

  ReactSharedInternals.H = null; // cleared immediately after
  return result;
}
```

The `useState` you import from React reads from this dispatcher. If it's null, React throws the error you've probably seen:

```javascript
function useState(initialValue) {
  const dispatcher = ReactSharedInternals.H;
  if (dispatcher === null) {
    throw new Error(
      'Invalid hook call. Hooks can only be called inside a function component.'
    );
  }
  return dispatcher.useState(initialValue);
}
```

That error message isn't a React policy statement — it's a null check. When `useState` runs outside a component render, there's no dispatcher to route the call to, and no component storage to write into.

This is why calling hooks in event handlers, `setTimeout`, or at the module's top level fails. By the time those run, the render is over. The dispatcher was cleared. The door is closed.

The two-dispatcher design also explains something subtle: React knows automatically whether to *create* a hook's storage (first render) or *read* from it (re-render) — without you passing any flags. The dispatcher carries that context.

---

## How React Actually Does It

The array model taught us the logic. React's actual implementation uses the same logic with a different data structure: not an array, but a linked list.

Each component fiber has a `memoizedState` property that points to the head of that component's hook list. Every hook call in the component corresponds to one node in that list:

```javascript
// The actual hook node shape — ReactFiberHooks.js, line 980
const hook = {
  memoizedState: null, // the stored value
  baseState: null,
  baseQueue: null,
  queue: null,         // update queue for setState calls
  next: null,          // pointer to the next hook
};
```

On the first render, [`mountWorkInProgressHook`](https://github.com/facebook/react/blob/404b38c764cf86e6f2ec42f873bb33ce114256d3/packages/react-reconciler/src/ReactFiberHooks.js#L979) creates each node and appends it to the list:

```javascript
function mountWorkInProgressHook() {
  const hook = { memoizedState: null, baseState: null,
                 baseQueue: null, queue: null, next: null };

  if (workInProgressHook === null) {
    // First hook — set as the list head
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

On every re-render, [`updateWorkInProgressHook`](https://github.com/facebook/react/blob/404b38c764cf86e6f2ec42f873bb33ce114256d3/packages/react-reconciler/src/ReactFiberHooks.js#L1000) walks the existing list in order — returning each node at its position, exactly like our cursor walking the array:

```javascript
function updateWorkInProgressHook() {
  // Walk to the next node in the list
  nextCurrentHook = currentHook === null
    ? currentlyRenderingFiber.alternate.memoizedState
    : currentHook.next;
  // ... clone or reuse the node
}
```

![diagram-03](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-react-hooks-system-usestate-useeffect-useref/images/diagram-03.png)

*Each hook call produces one node. The fiber holds a pointer to the head.*

The cursor and the linked list pointer are the same idea. The mechanism differs; the constraint is identical.

**Why a linked list and not an array?** React's fiber tree mounts and unmounts components dynamically during reconciliation. Linked lists handle arbitrary insertion and removal more naturally than arrays, and the traversal pattern — always walking forward from the head — maps directly to how React re-renders components.

**What `memoizedState` actually stores** varies by hook type. For `useState`, it's the current value. For `useEffect`, it's an effect object with the callback and dependencies. For `useRef`, it's a `{ current }` object. The node shape is the same; the contents differ.

**One deliberate exception.** React 19 introduced the `use()` hook, which *can* be called conditionally. It isn't stored in the linked list at all — it uses a completely different mechanism tied to React's Suspense infrastructure. Even the React team found a way to relax this rule, but only by building a different system underneath it. The rule doesn't bend. The architecture changes.

---

## The Rules Were Never Arbitrary

If you had to explain hooks to Marcos today, you wouldn't say "React needs them in the same order." You'd say: each hook gets a slot in a linked list. The slot is assigned by position. Change the position, and you corrupt every hook that comes after.

That's it. That's the whole rule.

Here's the path we walked:

1. **One global variable** — works for one hook, breaks for two. Not enough slots.
2. **Array + cursor** — one slot per hook, cursor resets before each render. Works. And the conditional rule falls out immediately: skip a hook, the cursor drifts, every subsequent hook reads the wrong value.
3. **The dispatcher** — a global object React sets before render and clears after. Outside render, it's null. That's the "React functions only" rule. Not a policy. A null check.
4. **The linked list** — React's actual implementation. Same logic as the array, different data structure. `mountWorkInProgressHook` builds the list on first render; `updateWorkInProgressHook` walks it on every re-render.

The linter doesn't guess at your intentions. It enforces the only invariant the data structure can survive: hooks must run in the same order, every render, without exception.

Marcos still writes React the same way. But now when he adds a hook, he knows exactly what's happening underneath — a new node appended to a linked list, assigned a position it will hold for the lifetime of that component instance.

The blank stare is gone.

---

**Part 2** takes this foundation and builds on it: `useEffect`, `useRef`, `useMemo`, and `useCallback` — implemented from scratch using the same system we just built. Once you've seen the dispatcher, the rest follows.

*If this article cleared up something you'd been taking on faith — share it. There are a lot of developers still staring blankly at the rules.*
