# Code Your Own Virtual DOM in 100 Lines of JavaScript

---

## Introduction: Open the Black Box

You've been using React for years. You know how to lift state, split components, and reach for `useMemo` when things get slow. But ask yourself: what does React's `render()` actually do?

Most developers never look. The framework works, so they don't have to. That's fine — until they hit a bug they can't explain, a performance profile that makes no sense, or a mental model with a hole in it the size of a reconciliation algorithm.

Here's the thing that most introductions get wrong: the Virtual DOM is not a performance optimization. Rich Harris, creator of Svelte, made this argument in 2018 and it still holds. A VDOM doesn't make DOM operations faster — it adds a layer of work *on top* of them. What it gives you is something more valuable: a programming model where you describe what the UI should look like, not how to get there. The framework figures out the minimal set of changes. You just write the output.

That abstraction is worth understanding from the inside.

In about 100 lines of vanilla JavaScript — no build step, no npm, no JSX — we're going to build the engine that powers it. Four functions. A complete, working Virtual DOM that can mount a tree to the DOM, diff two trees, and apply the minimum necessary changes. When you're done, React's internals won't feel like magic. They'll feel like an engineering tradeoff you understand.

Open a text editor and a browser tab. That's all you need.

---

## What We're Building

You already write UI in HTML: `<div id="app"><p>Count: 0</p></div>`. The problem is HTML is a string — your JavaScript can't inspect the tree, compare two versions, or compute what changed. What if you described the same structure as a plain JavaScript object instead? That single idea is what the Virtual DOM is built on.

Before writing a single function, here's the API we'll end up with:

<!-- no-test -->
```js
// Create virtual nodes
const vdom1 = h('div', { id: 'app' },
  h('p', {}, 'Count: 0')
);

// Mount to the real DOM
const container = render(vdom1);
document.body.appendChild(container);

// Later: the count changes
const vdom2 = h('div', { id: 'app' },
  h('p', {}, 'Count: 1')
);

// Compute what changed
const patches = diff(vdom1, vdom2);

// Apply only the changes
patch(container, patches);
// Result: only the text node is updated. The div and p stay untouched.
```

Four functions. No classes, no event bus, no reactivity system. Just `h`, `render`, `diff`, and `patch`.

A vnode — what `h()` returns — is a plain JavaScript object:

```js
const vnode = {
  type: 'p',
  props: {},
  children: [{ type: 'TEXT_NODE', props: { nodeValue: 'Count: 0' } }]
};
```

![diagram-01](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-virtual-dom-in-100-lines-of-javascript/images/diagram-01.png)

*A vnode tree mirrors the DOM structure — but lives entirely in memory as plain JS objects.*

That's it. No special prototype, no framework magic. A vnode is a description of a DOM node — nothing more. The real DOM knows nothing about it until `render()` walks the tree and builds actual elements.

---

## The Mental Model

The engine works in two phases. Get this picture in your head before reading a single line of implementation.

![diagram-02](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-virtual-dom-in-100-lines-of-javascript/images/diagram-02.png)

*The engine is four functions with a clean separation of concerns. Only `patch()` touches the DOM.*

**Phase 1 — Initial render:**

![diagram-03](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-virtual-dom-in-100-lines-of-javascript/images/diagram-03.png)

*Both phases start with `h()` — but one creates, the other updates.*

When something changes, you call `h()` again to describe what the UI should look like now. `diff()` compares the old vnode tree to the new one and produces a list of patch descriptors — plain objects describing what changed. Then `patch()` applies those changes to the real DOM.

Notice the division of responsibility:

- `diff()` never touches the DOM. It only compares two trees and produces data.
- `patch()` never reads vnodes. It only applies the patch descriptors to real nodes.

This separation is why the engine is testable and predictable. The diffing logic is a pure function — same inputs, same output, every time. The DOM mutation is isolated to `patch()`.

One more thing to lock in: `diff()` compares trees by **position**. Child at index 0 in the old tree is compared against child at index 0 in the new tree. This is a deliberate simplification. Real React uses `key` props to track element identity across reorders — we won't. We'll cover what that costs in section 9.

---

## Step 1 — `h()`: Creating Virtual Nodes

You've been looking at vnodes since the previous section — plain objects with `type`, `props`, and `children`. Now you need a function to create them. You could write those objects by hand, but nesting three levels of them gets unreadable fast. `h()` is the shorthand.

`h()` is the hyperscript function — the same pattern used by React's `createElement`, Vue's `h`, and Snabbdom. It takes a type, props, and children and returns a plain object.

```js
function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: children.flat().map(child =>
      typeof child === 'string' || typeof child === 'number'
        ? { type: 'TEXT_NODE', props: { nodeValue: String(child) } }
        : child
    )
  };
}
```

A few things happening here:

**`children.flat()`** — callers sometimes pass arrays of children (e.g. from `.map()`). Flattening one level keeps the tree clean without recursive complexity.

**Text node normalization** — the DOM distinguishes between element nodes and text nodes, and so does our engine. Strings and numbers become `{ type: 'TEXT_NODE', props: { nodeValue: '...' } }` so `render()` and `diff()` can treat all children uniformly without special-casing strings.

**`props || {}`** — callers often pass `null` for props when there are none: `h('br', null)`. The fallback prevents `Object.entries(null)` errors downstream.

That's 11 lines. Test it:

<!-- no-test -->
```js
console.log(h('p', { class: 'greeting' }, 'Hello'));
// {
//   type: 'p',
//   props: { class: 'greeting' },
//   children: [{ type: 'TEXT_NODE', props: { nodeValue: 'Hello' } }]
// }
```

The output is a plain object you can inspect, serialize, or pass around. Nothing framework-specific about it.

---

## Step 2 — `render()`: Mounting to the DOM

At this point, your vnode tree is a JavaScript object sitting in memory — the browser can't display it. Open DevTools right now and you'll find nothing in the Elements panel. To put anything on screen, you need to walk that tree and call real browser APIs: `createElement`, `setAttribute`, `createTextNode`. That's all `render()` does.

`render()` takes a vnode and returns a real DOM node. It's a recursive tree walk.

```js
function render(vnode) {
  if (vnode.type === 'TEXT_NODE') {
    return document.createTextNode(vnode.props.nodeValue);
  }

  const el = document.createElement(vnode.type);

  for (const [key, value] of Object.entries(vnode.props)) {
    if (key.startsWith('on') && typeof value === 'function') {
      el.addEventListener(key.slice(2).toLowerCase(), value);
    } else {
      el.setAttribute(key, value);
    }
  }

  for (const child of vnode.children) {
    el.appendChild(render(child));
  }

  return el;
}
```

Text nodes take a separate path — `document.createTextNode()` instead of `createElement()`. For everything else: create the element, apply props, recurse on children.

The event handler convention (`onClick` → `addEventListener('click', ...)`) lets you write `h('button', { onClick: handleClick }, 'Click me')` without any event delegation machinery. It's not as efficient as a production implementation, but it's correct and readable at this scale.

At this point you have a working static renderer. Try it:

<!-- no-test -->
```js
const vnode = h('ul', {},
  h('li', {}, 'Apples'),
  h('li', {}, 'Oranges'),
  h('li', {}, 'Bananas')
);

document.body.appendChild(render(vnode));
```

Open your browser. You'll see a list. The VDOM engine is half done — `render()` handles the initial mount. Now we need to handle updates.

---

## Step 3 — `diff()`: Comparing Two Trees

The user clicked the button. `count` is now 1. You call `h()` again and get a new vnode tree — but the DOM still reflects the old one. You have two trees: what is and what should be. You could call `render()` again and replace the whole DOM, but that defeats the point. Instead, you need to compare the two trees and produce a precise description of only what changed. That's `diff()`.

This is the core of the engine. `diff()` takes two vnode trees and returns a patch descriptor — a plain object describing what changed. It never touches the DOM.

There are three cases to handle.

Start with the function signature and the two edge cases: a node that didn't exist before, and a node that no longer exists.

```js
function diff(oldVnode, newVnode) {
  if (!oldVnode) return { kind: 'CREATE', newVnode };
  if (!newVnode) return { kind: 'REMOVE' };
  // ...
}
```

When `diff()` recurses on children, the old and new trees may have different lengths. An index that exists in the new tree but not the old means a node was added — `CREATE`. The reverse means it was removed — `REMOVE`.

**Case 1: Different types**

If the old node is a `div` and the new one is a `span`, we don't attempt reconciliation. We replace the whole subtree. This is the same heuristic React uses: different types produce different trees, so tearing down and rebuilding is cheaper than diffing across types.

```js
function diff(oldVnode, newVnode) {
  if (!oldVnode) return { kind: 'CREATE', newVnode };
  if (!newVnode) return { kind: 'REMOVE' };

  if (oldVnode.type !== newVnode.type) {
    return { kind: 'REPLACE', newVnode };
  }
  // ...
}
```

**Case 2: Both text nodes**

Text nodes have no children or props to recurse into — just compare `nodeValue`. If it changed, return an update. If not, return `null`: no patch needed, skip the DOM entirely.

```js
function diff(oldVnode, newVnode) {
  if (!oldVnode) return { kind: 'CREATE', newVnode };
  if (!newVnode) return { kind: 'REMOVE' };

  if (oldVnode.type !== newVnode.type) {
    return { kind: 'REPLACE', newVnode };
  }

  if (newVnode.type === 'TEXT_NODE') {
    return oldVnode.props.nodeValue !== newVnode.props.nodeValue
      ? { kind: 'TEXT', value: newVnode.props.nodeValue }
      : null;
  }
  // ...
}
```

**Case 3: Same element type**

If we reach this point, both nodes are the same element type. We need to do two things: diff the props and diff the children.

For props, collect all keys from both old and new, and record any that changed. A key present in old but missing in new gets `undefined` — that's our signal to remove it in `patch()`.

<!-- no-test -->
```js
  // Inside diff() — same element type case:
  const propPatches = {};
  const allKeys = new Set([
    ...Object.keys(oldVnode.props),
    ...Object.keys(newVnode.props)
  ]);

  for (const key of allKeys) {
    if (newVnode.props[key] !== oldVnode.props[key]) {
      propPatches[key] = newVnode.props[key];
    }
  }
```

For children, iterate up to the longest of the two child arrays and call `diff()` recursively on each pair by index:

<!-- no-test -->
```js
  // Inside diff() — same element type case (continued):
  const childrenLen = Math.max(
    oldVnode.children.length,
    newVnode.children.length
  );
  const childPatches = Array.from({ length: childrenLen }, (_, i) =>
    diff(oldVnode.children[i], newVnode.children[i])
  );
```

If nothing changed — no prop diffs, no child diffs — return `null`. No patch needed at this node.

**The complete function:**

```js
function diff(oldVnode, newVnode) {
  if (!oldVnode) return { kind: 'CREATE', newVnode };
  if (!newVnode) return { kind: 'REMOVE' };

  if (oldVnode.type !== newVnode.type) {
    return { kind: 'REPLACE', newVnode };
  }

  if (newVnode.type === 'TEXT_NODE') {
    return oldVnode.props.nodeValue !== newVnode.props.nodeValue
      ? { kind: 'TEXT', value: newVnode.props.nodeValue }
      : null;
  }

  const propPatches = {};
  const allKeys = new Set([
    ...Object.keys(oldVnode.props),
    ...Object.keys(newVnode.props)
  ]);
  for (const key of allKeys) {
    if (newVnode.props[key] !== oldVnode.props[key]) {
      propPatches[key] = newVnode.props[key];
    }
  }

  const childrenLen = Math.max(
    oldVnode.children.length,
    newVnode.children.length
  );
  const childPatches = Array.from({ length: childrenLen }, (_, i) =>
    diff(oldVnode.children[i], newVnode.children[i])
  );

  if (Object.keys(propPatches).length === 0 && childPatches.every(p => p === null)) {
    return null;
  }

  return { kind: 'UPDATE', propPatches, childPatches };
}
```

![diagram-04](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-virtual-dom-in-100-lines-of-javascript/images/diagram-04.png)

*`diff()` makes exactly four decisions. Every case returns a plain object — or `null` if nothing changed.*

The child diffing is index-based: child 0 in the old tree is always compared to child 0 in the new tree. If you reorder a list of 100 items, the engine generates up to 100 UPDATE patches instead of recognising the move. That's the cost of skipping `key` support. For a counter demo it's invisible; for a sortable table it would matter. We'll come back to this in section 9.

**What React does differently:** React uses `key` props for list identity, a Fiber architecture that makes reconciliation interruptible, and batched state updates. Each adds significant complexity. Our engine has none of them — and for most UIs under a few hundred nodes, the difference is undetectable.

---

## Step 4 — `patch()`: Applying Changes

`diff()` returned a patch descriptor — a plain object saying "this text node's value changed", "this child was removed". But the DOM hasn't moved yet. Nothing has. Someone has to carry out those instructions against real browser nodes. That's `patch()`.

`patch()` takes a real DOM node and a patch descriptor from `diff()`, and mutates the DOM. This is the only function in the engine that touches real nodes.

```js
function patch(domNode, patchDescriptor) {
  if (patchDescriptor === null) return;

  const parent = domNode.parentNode;

  if (patchDescriptor.kind === 'REMOVE') {
    parent.removeChild(domNode);
    return;
  }

  if (patchDescriptor.kind === 'CREATE') {
    parent.appendChild(render(patchDescriptor.newVnode));
    return;
  }

  if (patchDescriptor.kind === 'REPLACE') {
    parent.replaceChild(render(patchDescriptor.newVnode), domNode);
    return;
  }

  if (patchDescriptor.kind === 'TEXT') {
    domNode.nodeValue = patchDescriptor.value;
    return;
  }

  if (patchDescriptor.kind === 'UPDATE') {
    for (const [key, value] of Object.entries(patchDescriptor.propPatches)) {
      if (key.startsWith('on') && typeof value === 'function') {
        // Event handler updates are out of scope — skip for now
      } else if (value === undefined) {
        domNode.removeAttribute(key);
      } else {
        domNode.setAttribute(key, value);
      }
    }

    const children = Array.from(domNode.childNodes);
    patchDescriptor.childPatches.forEach((childPatch, i) => {
      if (childPatch && childPatch.kind === 'CREATE') {
        domNode.appendChild(render(childPatch.newVnode));
      } else {
        patch(children[i], childPatch);
      }
    });
  }
}
```

Each `kind` maps directly to a DOM operation:

- `REMOVE` — the node was in the old tree but not the new. Remove it.
- `CREATE` — the node is in the new tree but wasn't in the old. Render and append it.
- `REPLACE` — the types diverged. Render the new vnode and swap it in.
- `TEXT` — set `nodeValue` directly on the text node. No element recreation needed.
- `UPDATE` — apply changed props, then recurse on children using their index positions in `domNode.childNodes`.

One honest limitation worth calling out: event handler updates (changing an `onClick` from one function to another) are skipped here. Handling them correctly requires tracking the old listener to call `removeEventListener` before adding the new one — which means storing refs to functions, which adds state. It's solvable, but not in our line budget. For the demo ahead, event handlers are set once at render time and never change.

---

## Putting It Together

All four functions are done. Here's a counter that updates the DOM on every click — the demo code lives outside the 100-line engine:

```html
<!DOCTYPE html>
<html>
<body>
<script>
// ── engine (71 lines) ──────────────────────────────────
function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: children.flat().map(child =>
      typeof child === 'string' || typeof child === 'number'
        ? { type: 'TEXT_NODE', props: { nodeValue: String(child) } }
        : child
    )
  };
}

function render(vnode) {
  if (vnode.type === 'TEXT_NODE') {
    return document.createTextNode(vnode.props.nodeValue);
  }
  const el = document.createElement(vnode.type);
  for (const [key, value] of Object.entries(vnode.props)) {
    if (key.startsWith('on') && typeof value === 'function') {
      el.addEventListener(key.slice(2).toLowerCase(), value);
    } else {
      el.setAttribute(key, value);
    }
  }
  for (const child of vnode.children) {
    el.appendChild(render(child));
  }
  return el;
}

function diff(oldVnode, newVnode) {
  if (!oldVnode) return { kind: 'CREATE', newVnode };
  if (!newVnode) return { kind: 'REMOVE' };
  if (oldVnode.type !== newVnode.type) return { kind: 'REPLACE', newVnode };
  if (newVnode.type === 'TEXT_NODE') {
    return oldVnode.props.nodeValue !== newVnode.props.nodeValue
      ? { kind: 'TEXT', value: newVnode.props.nodeValue }
      : null;
  }
  const propPatches = {};
  const allKeys = new Set([...Object.keys(oldVnode.props), ...Object.keys(newVnode.props)]);
  for (const key of allKeys) {
    if (newVnode.props[key] !== oldVnode.props[key]) propPatches[key] = newVnode.props[key];
  }
  const childrenLen = Math.max(oldVnode.children.length, newVnode.children.length);
  const childPatches = Array.from({ length: childrenLen }, (_, i) =>
    diff(oldVnode.children[i], newVnode.children[i])
  );
  if (Object.keys(propPatches).length === 0 && childPatches.every(p => p === null)) return null;
  return { kind: 'UPDATE', propPatches, childPatches };
}

function patch(domNode, patchDescriptor) {
  if (patchDescriptor === null) return;
  const parent = domNode.parentNode;
  if (patchDescriptor.kind === 'REMOVE') { parent.removeChild(domNode); return; }
  if (patchDescriptor.kind === 'CREATE') { parent.appendChild(render(patchDescriptor.newVnode)); return; }
  if (patchDescriptor.kind === 'REPLACE') { parent.replaceChild(render(patchDescriptor.newVnode), domNode); return; }
  if (patchDescriptor.kind === 'TEXT') { domNode.nodeValue = patchDescriptor.value; return; }
  if (patchDescriptor.kind === 'UPDATE') {
    for (const [key, value] of Object.entries(patchDescriptor.propPatches)) {
      if (key.startsWith('on') && typeof value === 'function') {
        // skip — event handler updates not supported
      } else if (value === undefined) {
        domNode.removeAttribute(key);
      } else {
        domNode.setAttribute(key, value);
      }
    }
    const children = Array.from(domNode.childNodes);
    patchDescriptor.childPatches.forEach((childPatch, i) => {
      if (childPatch && childPatch.kind === 'CREATE') {
        domNode.appendChild(render(childPatch.newVnode));
      } else {
        patch(children[i], childPatch);
      }
    });
  }
}
// ── end engine ──────────────────────────────────────────

// Demo: counter (not counted in the 79 lines)
let count = 0;

function view(n) {
  return h('div', {},
    h('p', {}, `Count: ${n}`),
    h('button', { onClick: increment }, 'Increment')
  );
}

function increment() {
  const oldVdom = view(count);
  count++;
  const newVdom = view(count);
  const patches = diff(oldVdom, newVdom);
  patch(container, patches);
}

const initialVdom = view(count);
const container = render(initialVdom);
document.body.appendChild(container);
</script>
</body>
</html>
```

Save this as `index.html` and open it in a browser. Click the button. Open DevTools and watch the Elements panel — only the text node inside `<p>` flashes on each click. The `<div>`, `<p>`, and `<button>` elements are never recreated.

**Line count audit:**

| Function | Lines |
|---|---|
| `h()` | 11 |
| `render()` | 17 |
| `diff()` | 21 |
| `patch()` | 22 |
| **Total** | **71 lines** |

71 lines. The title promised 100 and we came in under budget — because every line had to earn its place.

![diagram-05](https://raw.githubusercontent.com/luizcg/articles/main/articles/code-your-own-virtual-dom-in-100-lines-of-javascript/images/diagram-05.png)

*`patch()` only touches what changed. The `div` and `p` are never recreated.*

---

## What We Skipped — and Why

The engine works. It's also deliberately incomplete. Here's what didn't make the cut and why each omission was a real decision, not an oversight.

| Feature | Why we skipped it | Where to go next |
|---|---|---|
| **Keyed diffing** | Tracking element identity across reorders requires a map-based algorithm — roughly 40 extra lines just for this case. Without keys, reordering a list of 10 items produces 10 patches instead of 1 move operation. Acceptable for small lists; not for production. | [Snabbdom source](https://github.com/snabbdom/snabbdom) — the keyed diff is readable and well-commented |
| **Event handler updates** | Correctly swapping a listener requires storing the previous function to call `removeEventListener`. That means tracking state outside the vnode, which breaks the stateless-function model of our engine. | Preact handles this cleanly — worth reading |
| **Components** | A component is just a function that returns a vnode: `const Button = (props) => h('button', props, 'Click')`. Adding component support means resolving these functions during `diff()` — another ~20 lines. | [Didact](https://pomb.us/build-your-own-react/) adds components in step 5 |
| **Fiber / async rendering** | React's Fiber architecture makes reconciliation interruptible — it can pause diffing mid-tree and yield to the browser. This prevents long trees from blocking the main thread. Our synchronous diff would freeze on trees with thousands of nodes. | [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture) |
| **SVG support** | SVG elements require `document.createElementNS('http://www.w3.org/2000/svg', tag)` instead of `createElement`. A two-line change, but it complicates the `render()` function's element creation path. | MDN: `createElementNS` |
| **Batched updates** | Calling `diff` + `patch` synchronously on every state change means 10 rapid updates = 10 separate DOM mutations. React batches these into one. | React 18 automatic batching docs |

The point isn't that our engine is naive — it's that each of these features exists because a real problem demanded it. Keyed diffing came from list performance bugs. Fiber came from animation jank in complex trees. Every line in React or Vue's codebase is a solved problem that our 79 lines haven't encountered yet.

That's not a failure. That's scope.

---

## Conclusion

71 lines of vanilla JavaScript. Four functions. A working Virtual DOM that mounts trees, diffs them, and applies the minimum necessary changes to the real DOM.

That's all React does at its core too. The rest — hooks, Fiber, concurrent features, the dev tools, the ecosystem — is years of engineering on top of this same foundation. Understanding the foundation doesn't make the rest trivial, but it makes it legible.

If you want to keep going, here's a path with increasing depth:

1. **[Snabbdom](https://github.com/snabbdom/snabbdom)** — A production-grade Virtual DOM in ~800 lines. Modular, readable, and the library Vue's original VDOM was based on. Start here to see how keyed diffing and a module system extend what we built.

2. **[Preact](https://github.com/preactjs/preact)** — A React-compatible implementation in ~4KB gzipped. The source is approachable and shows how components, hooks, and event handling layer on top of a minimal VDOM core.

3. **[Build your own React](https://pomb.us/build-your-own-react/)** by Rodrigo Pombo — The definitive deep-dive. Walks through Fiber, reconciliation, and hooks from scratch. Pick this up when you're ready to understand React's scheduler.

One last thing worth holding onto: the 100-line constraint wasn't a gimmick. It forced every function to do exactly one job and nothing else. `diff()` doesn't touch the DOM. `patch()` doesn't read vnodes. `h()` doesn't know what `render()` does. That separation is why each piece is understandable in isolation.

Constraints are how you find out what actually matters.
