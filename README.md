# React Coding Standards

This guide is focusing on the recent versions of react featuring [React Hooks](https://reactjs.org/docs/hooks-intro.html).

## Don't forget about adding keys for list items

You know they're [quite important](https://reactjs.org/docs/reconciliation.html#recursing-on-children), aren't you? ;)

Bad:

```jsx
const Component = ({ items }) => (
  <ul>
    {items.map((item) => (
      <li>{item.name}</li>
    ))}
  </ul>
);
```

__Good:__

```jsx
const Component = ({ items }) => (
  <ul>
    {items.map((item) => (
      <li key={item.id}>{item.name}</li>
    ))}
  </ul>
);
```

### What if the items within the iterated array don't have IDs?

Maybe they hold an other value which can identify them? Lists tend to hold _unique_ items: different text, colours. E.g., when dealing with an array of strings, consider __filtering out duplicates__. This will allow to safely use those strings as keys.

  Example:

  ```js
  const items = ['a', 'a', 'b', 'c'];
  const uniqueItems = Array.from(new Set(items));
  // ^ Once filtered, the unique items are ready to use as keys.
  ```

It's tempting to use `index` to quickly deal with `key` property. Usually, there are better options but if we know the list is constant and not going to be altered (sorting, filtering, etc.), then `index` might work just fine.

Example:

```jsx
const Component = ({ items }) => (
  <ul>
    {new Array(3).fill(0).map((_, index) => (
      <li key={index}>{index + 1}. Item</li>
    ))}
  </ul>
);
// ^ This component is supposed to always print 3 list items.
```

## Be careful when using objects as defaults

‼️ Remember, [primitive values](https://developer.mozilla.org/en-US/docs/Glossary/Primitive) can be safely compared for equality even when assigned to different variables. E.g.,

```js
const a = '2'
const b = '2'

a === a; // returns true
b === b; // returns true

// Comparing different variables
a === b; // returns true
```

However, objects (all things which __aren't__ primitives) can't rely on such comparison:

```js
const a = [];
const b = [];

a === a; // returns true
b === b; // returns true

// Comparing different variables
a === b; // returns false
``` 

It's important to keep that in mind when dealing with various values in React and providing them as "deps" for various hooks.

Here are the common patterns you might want to use in your components or custom hooks:

Bad:

```js
const useMyHook = ({ values = [] }) => {
  // ^ We provide an empty array to guard from `undefined`.

  useEffect(function doSomethingWithValues() {
      // Code fires when `value` changes.
  }, [values])
};
```

__Good:__

```js
const DEFAULT_VALUE = [];

const useMyHook = ({ values = DEFAULT_VALUE }) => {
  // ^ This time `DEFAULT_VALUE` is alawyas _the same_.

  useEffect(function doSomethingWithValues() {
      // Code fires when `value` changes.
  }, [value])
};
```

☝️ In the "bad" example the empty array would be _assigned on each re-render_ resulting in firing the `doSomethingWithValue` effect **too often**. The "good" example fixes that by assigning an empty array to a variable outside of the function.

It applies to this similar pattern as well:

Bad:

```js
  const MyComponent = () => {
    const values = useValue(); // returns an array or null
    const safeValues = values || [];

    useEffect(function doSomethingWithValues() {
        // Code fires when `value` changes.
    }, [value])

    return null;
  }
```

__Good:__

```js
  const MyComponent = () => {
    const values = useValue(); // returns an array or null
    const safeValues = useMemo(() => values || [], [values]);
    // ^ Use memo guards from reassigning the empty array
    // on each render.

    useEffect(function doSomethingWithValues() {
        // Code fires when `value` changes.
    }, [value])

    return null;
  }
```

☝️It's worth pointing out the previous solution with `DEFAULT_VALUE` could work here as well. Choose whatever is suitable for your use case.

## Clear side effects

Watch out for events, promises, timers, or observers which can run _after_ the effect has changed (due to its properties) OR the component holding it has been unmounted. It's important to realise useEffect's return value (and the function itself) _fires each time one of the dependencies changes_. Not only when component is mounted and unmounted.

### Example with events:

Bad:

```js
useEffect(function exampleEffect() {
  const handleKeyUp = ({ altKey }) => {
    if (altKey) someFunction();
  }

  document.addEventListener('keyup', handleKeyUp);
}, [someFunction]);
```

__Good:__

```js
useEffect(function exampleEffect() {
  const handleKeyUp = ({ altKey }) => {
    if (altKey) someFunction();
  }

  document.addEventListener('keyup', handleKeyUp);

  return () => {
    document.removeEventListener('keyup', handleKeyUp);
  };
}, [someFunction]);
```

### Example with timers:

Bad:

```js
useEffect(function exampleEffect() {
  setTimeout(() => setValueState(value), 2000);
}, [value]);
```

__Good:__

```js
useEffect(function exampleEffect() {
  const timerId = setTimeout(() => setValueState(value), 2000);

  return () => {
    clearTimerout(timerId);
  };
}, [value]);
// ^ Aside from debouncing, clearing the timeout guards from
// setting the state on an unmounted component.
```

### Promises

Promise based code is more complex and requires different strategies depending on the case. E.g., you may want to [abort signal](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal#examples) for a fetch request _or_ it might be a better idea to _ignore_ a promise. See: [an effective pattern to deal with promises firing _after_ unmounting the component](https://www.robinwieruch.de/react-hooks-fetch-data#abort-data-fetching-in-effect-hook). The link targets one specific section of a larger article. It's totally worth to read the whole piece though!

## Name functions in effects

Named function increase the code readability. Aside from that, they output _more accurate error messages_. The named function is visible in the stack trace helping to quickly navigate to the source of the problem. It's especially valuable in case of complex components with many effects.

Example:

Bad:

```js
useEffect(() => {
  console.log('message');
}, []);
```

__Good:__

```js
useEffect(function logMessage() {
  console.log('message');
}, []);
```

## Don't forget about dependencies

Many of the [Hooks coming with React](https://reactjs.org/docs/hooks-reference.html) require a _dependency array_: `useEffect`, `useLayoutEffect`, `useMemo`, `useCallback`. The purpose of this array is to keep refreshing __the function inserted in each of those hooks__ according to the changing dependencies so that it __always has access to the up to date values__.

Quick example:

Bad:

```jsx
const Component = () => {
  const [val, setVal] = React.useState(0);

  const alertValue = React.useCallback(() => {
    alert(val);
  }, []); // <- Missing `val` in the dependencies. ❌
  // ^ The function will always display an alert with `0`.

  return (
    <>
      <button type="button" onClick={() => setVal((prev) => prev + 1)}>
        Increase Val
      </button>
      <button type="button" onClick={alertValue}>
        Display Val
      </button>
    </>
  );
};
```

__Good:__

```jsx
const Component = () => {
  const [val, setVal] = React.useState(0);

  const alertValue = React.useCallback(() => {
    alert(val);
  }, [val]); // <- `val` present. ✅
  // ^ The function will display an alert with the current `val`.

  return (
    <>
      <button type="button" onClick={() => setVal((prev) => prev + 1)}>
        Increase Val
      </button>
      <button type="button" onClick={alertValue}>
        Display Val
      </button>
    </>
  );
};
```

---

* You're going to stumble upon _much more_ complicated cases when working on real world applications. However, skipping the dependencies is hardly ever a good idea.

* You might be under impression omitting dependencies might increase the overall performance but more often it can lead to all kind of weird, hard to track errors.

* If you're looking for a quality article shedding more light on hook's dependencies then look no further: https://overreacted.io/a-complete-guide-to-useeffect/ This article feels more like a small book but __it's well worth your time__. It focuses on one `useEffect` but, even despite that, serves as a thorough overview of the mechanics behind hooks.
