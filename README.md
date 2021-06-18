# React Coding Standards

This guide is focusing on the recent versions of react featuring [React Hooks](https://reactjs.org/docs/hooks-intro.html).

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

This potential problems applies to this similar pattern as well:

Bad:

```jsx
  const MyComponent = () => {
    const values = useValue(); // returns an array or null
    const safeValues = values || [];

    useEffect(function doSomethingWithValues() {
        // Code fires when `value` changes.
    }, [value])

    return <div />
  }
```

__Good:__

```jsx
  const MyComponent = () => {
    const values = useValue(); // returns an array or null
    const safeValues = useMemo(() => values || [], [values]);
    // ^ Use memo guards from reassigning the empty array
    // on each render.

    useEffect(function doSomethingWithValues() {
        // Code fires when `value` changes.
    }, [value])

    return <div />
  }
```

☝️It's worth pointing out the previous solution with `DEFAULT_VALUE` could work here as well. Choose whatever is suitable for your use case.


## Clear side effects

The "bad" and "good" examples below rely on events _but_ it applies to other things like promises, timers, or observers which can run _after_ the effect has changed (due to its properties) OR the component holding it has been unmounted. It's important to realise useEffect's return value (and the function itself) _fires each time one of the dependencies changes_. Not only when component is mounted and unmounted.

Bonus link: [an effective pattern to deal with promises firing _after_ unmounting the componnent](https://www.robinwieruch.de/react-hooks-fetch-data#abort-data-fetching-in-effect-hook). The link targets one specific section of a larger article. It's totally worth to read the whole piece though!

Example:

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

## Name function in effects

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

* You're going to stumble upon _much more_ complicated cases when working on real world applications. However, skipping the dependencies hardly ever a good idea.

* You might be under impression omitting dependencies might increase the overall performance but more often it can lean to all kind of weird, hard to track errors.

* If you're looking for a quality article shedding more light on hook's dependencies then look no further: https://overreacted.io/a-complete-guide-to-useeffect/ This article feels more like a small book but __it's well worth your time__. It focuses on one `useEffect` but, even despite that, serves as a thorough overview of the mechanics behind hooks.
