---
id = "redux-observable-provider"
title = "Observable & Provider Patterns in Redux"
abstract = "Redux still powers many production apps despite newer alternatives. Building Redux from scratch reveals how Observable and Provider patterns work together to manage state."
tags = ["javascript", "react"]
date = 2024-02-23
status = "show"
---

Redux has been around long enough that you've probably used it, or at least heard coworkers complain about its boilerplate. Despite newer alternatives like React-Query, Jotai, and Recoil, Redux still runs in production apps everywhere.

The interesting thing about Redux isn't the action creators or reducers—it's the patterns underneath. Redux is built on two foundational patterns: Observable and Provider. Understanding these patterns explains not just how Redux works, but why it works the way it does.

## The Observable Pattern

The Observable pattern is about watching for changes and letting interested parties know when something happens. Think of it like hosting a watch party—you notify everyone when the next episode starts.

Here's how you might implement one:

```jsx
function createObservable() {
  let observers = []; // Array to store observers who will monitor state changes

  // Function to add an observer
  function subscribe(fn) {
    observers.push(fn); // Add the observer to the list

    // Return a function to unsubscribe, using closure to remember fn
    return function () {
      unSubscribe(fn);
    };
  }

  // Function to unsubscribe an observer
  function unSubscribe(fn) {
    observers = observers.filter((ob) => ob !== fn); // Remove the specified observer
  }

  // Function to notify all observers of a state change
  function notify(value) {
    if (observers.length === 0) {
      console.log("---- Sorry, no more subscriptions ----");
      return;
    }

    observers.forEach((ob) => {
      ob(value); // Call each observer function with the new value
    });
  }

  return {
    subscribe,
    unSubscribe,
    notify,
  };
}
```

The `subscribe` function returns an unsubscribe function that remembers which observer to remove later. This is handy when you're using anonymous functions—otherwise you'd need to store a reference to unsubscribe.

```jsx
const observable = createObservable();

const unsubscribe = observable.subscribe((value) => {
  console.log("Received value from observer: ", value);
});

observable.notify("Tada!!"); // Output: Received value from observer: Tada!!

unsubscribe();

observable.notify("Tada!!"); // No output
```

The first `notify` call reaches the observer and logs the message. After unsubscribing, the second call goes nowhere.

Going back to the watch party analogy: you're the host managing who gets updates about new episodes. Friends can join your notification list (subscribe), you tell everyone when episodes are available (notify), and friends can leave the list anytime (unsubscribe). The Observable pattern works the same way—it's just a more formal way of organizing who gets told about what.

## Redux as an Observable

Redux is essentially an Observable with some extra features. The store holds state, components subscribe to changes, and when you dispatch an action, all subscribers get notified. Here's what a minimal Redux implementation looks like:

```jsx
// createStore function: Takes a reducer and an initial state to create the store
function createStore(reducer, initialState) {
  let state = initialState; // Current state of the store
  let listeners = []; // List of listeners to notify on state change

  // Function to get the current state
  function getState() {
    return state;
  }

  // Function to subscribe a listener to state changes
  function subscribe(listener) {
    listeners.push(listener); // Add the listener to the list
    // Return a function to unsubscribe, using closure to remember the listener
    return function () {
      unSubscribe(listener);
    };
  }

  // Function to unsubscribe a listener
  function unSubscribe(listener) {
    listeners = listeners.filter((lis) => lis !== listener); // Remove the specified listener
  }

  // Function to dispatch an action and notify listeners of state changes
  function dispatch(action) {
    state = reducer(state, action); // Update state using the reducer
    listeners.forEach((listener) => {
      listener(); // Notify all listeners of the state change
    });
  }

  // Return the public methods of the store
  return {
    getState, // Get the current state
    dispatch, // Dispatch an action
    subscribe, // Subscribe to state changes
    unSubscribe, // Unsubscribe from state changes
  };
}
```

The key difference from our basic Observable is the `dispatch` method—it runs the action through a reducer to get the new state, then notifies all listeners. Everything else works the same way.

## Using Redux in React

Here's how you'd use our Redux implementation in a React component:

```jsx
const store = createStore((state = 0, action) => {
  const { type, payload } = action;
  switch (type) {
    case "INCREMENT":
      return state + payload;
    case "DECREMENT":
      return state - payload;
    default:
      return state;
  }
}, 0);
```

```jsx
// App function component
function App() {
  // useState hook to manage the component's state (count). Initial value is the store's state.
  const [count, setCount] = useState(store.getState());

  // onClick function: Called when the button is clicked. Dispatches an 'INCREMENT' action to increase the count.
  const onClick = () => {
    store.dispatch({ type: "INCREMENT", payload: 1 });
  };

  // useEffect hook to subscribe to the store when the component mounts.
  useEffect(() => {
    // Subscribe to the store and update the component's state whenever the store's state changes.
    const unsubscribe = store.subscribe(() => {
      const newCount = store.getState();
      setCount(newCount);
    });
    return unsubscribe; // Return the unsubscribe function to clean up the subscription.
  }, []); // Empty dependency array to ensure this runs only once when the component mounts.

  // Render the component UI
  return (
    <div className="App">
      <h1>COUNTER</h1>
      <span>{count}</span> {/* Display the current count */}
      <div style={{ marginTop: "10px" }}>
        <button onClick={onClick}>+</button>{" "}
        {/* Button to increment the count */}
      </div>
    </div>
  );
}
```

The component subscribes to the store on mount and updates its local state whenever the store changes. It's a bit manual, but it works.

## The Provider Pattern

The manual Redux approach works but gets tedious quickly. You have to subscribe to the store in every component that needs state, manage subscriptions, and handle cleanup. React-Redux solves this with the Provider pattern.

The Provider pattern uses React's Context API to make the Redux store available to any component in the tree without manually passing it down. Here's the basic idea:

```jsx
import React, { createContext, useContext } from "react";

const MessageContext = createContext();

export function MessageProvider({ children }) {
  const message = "Hello from Context!";
  return (
    <MessageContext.Provider value={message}>
      {children}
    </MessageContext.Provider>
  );
}

function ChildComponent() {
  const message = useContext(MessageContext);
  return <div>{message}</div>;
}
```

Any component inside `MessageProvider` can access the message without prop drilling. This same pattern works for Redux stores.

## Building React-Redux

Here's how you'd implement the essential parts of React-Redux:

```jsx
const StoreContext = createContext(null);

export function StoreProvider({ store, children }) {
  return (
    <StoreContext.Provider value={store}>{children}</StoreContext.Provider>
  );
}

function useStore() {
  const context = useContext(StoreContext);
  if (!context) {
    throw new Error("useStore must be used within a StoreProvider");
  }
  return context;
}

export function useDispatch() {
  const store = useStore();
  return store.dispatch;
}

export function useSelector(selector) {
  const store = useStore();
  const [selectedState, setSelectedState] = useState(() =>
    selector(store.getState())
  );

  useEffect(() => {
    const unsubscribe = store.subscribe(() => {
      const newState = selector(store.getState());
      setSelectedState(newState);
    });
    return unsubscribe;
  }, [store, selector]);

  return selectedState;
}
```

`useSelector` does the subscription management for you—it subscribes to the store, runs your selector function when the store changes, and triggers a re-render if the selected value changed.

Now instead of manually subscribing in every component, you can just use `useSelector` and `useDispatch`:

```jsx
function Counter() {
  const dispatch = useDispatch();
  const count = useSelector((state) => state);

  const onClick = () => {
    dispatch({ type: "INCREMENT", payload: 1 });
  };

  return (
    <div>
      <h1>COUNTER</h1>
      <span>{count}</span>
      <button onClick={onClick}>+</button>
    </div>
  );
}
```

Much cleaner than the manual subscription approach.

## Patterns All the Way Down

Redux might seem complex with all its action creators, middleware, and boilerplate, but at its core it's just two simple patterns working together. The Observable pattern handles the subscription logic, and the Provider pattern handles getting the store to your components without passing it everywhere.

Understanding these patterns explains why Redux works the way it does, and why it's been so durable despite all the "Redux killers" that have come along. The patterns are solid—they just sometimes get buried under layers of abstraction and tooling.

---

References:

- [**Building Redux from scratch**](https://medium.com/@guokai83524/building-redux-from-scratch-e12eb0e484c8)
- [**JavaScript**: *Build Yourself a Redux*](https://zapier.com/engineering/how-to-build-redux/)
- [**JavaScript**: *Let’s Write Redux!*](https://www.jamasoftware.com/blog/lets-write-redux/)
- [Provider pattern](https://www.patterns.dev/vanilla/provider-pattern)
- [Observer pattern](https://www.patterns.dev/vanilla/observer-pattern)
