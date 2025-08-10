---
id = "redux-observable-provider"
title = "Observable & Provider Patterns in Redux"
abstract = "If I were to choose one of the most important aspects of front-end development, it would undoubtedly be state management. If you have any experience in front-end development, you have likely encountered the Redux library at least once, whether you liked it or not."
tags = ["javascript", "react"]
date = 2024-02-23
status = "show"
---

If I were to choose one of the most important aspects of front-end development, it would undoubtedly be state management. If you have any experience in front-end development, you have likely encountered the Redux library at least once, whether you liked it or not.

Although many state management libraries like React-Query, Jotai, and Recoil have emerged after Redux, many production projects still rely on Redux's state management architecture.

Today, we'll dive into the core concepts of Redux, specifically the Observable and Provider patterns, by implementing the Redux library ourselves and understanding how these patterns are utilized within Redux.

## What is the Observable Pattern?

The Observable pattern operates by monitoring changes in the state of an object and notifying one or more observers whenever there is a change.

Let's look at a basic implementation of an Observable.

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
      console.log('---- Sorry, no more subscriptions ----');
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

Creating an Observable object: The createObservable function creates an Observable object. This object manages all registered observers through an internal observers array.

The subscribe function: This function adds the observer function passed as an argument to the observers array of the Observable instance.


>💡 Although not essential, the subscribe function often returns an unsubscribe function, which uses a closure to remember the observer function initially passed to subscribe. 

This is especially useful when using anonymous functions for subscribing. Otherwise, to unsubscribe, you would need to store and reference the observer function in a variable.


## Example Usage of the Observable Pattern

```jsx
const observable = createObservable();

// Register an observer that logs the new value to the console on state change
const unsubscribe = observable.subscribe((value) => {
  console.log('Received value from observer: ', value);
});

// Notify all observers with the value 'Tada!!'
observable.notify('Tada!!'); // Output: Received value from observer: Tada!!

// Unsubscribe the observer
unsubscribe();

// After unsubscribing, no notification will be received
observable.notify('Tada!!'); // No output

```

In this example, we create an observable object using createObservable and add an observer function via the subscribe method that logs the new value to the console on state changes.

On the first notify call, the observer is registered, so it logs "Received value from observer: Tada!!". After calling unsubscribe, the observer is unsubscribed, and the second notify call results in no output.

```jsx
/*
A fun analogy:

Observable: You, hosting a drama watch party, are the center of attention for sharing the new episode information with your friends.

Observers: Your friends attending the party are eagerly waiting for the new episode information.

Subscribe: Adding a new friend to the watch party when they want to join.

Notify: Informing all friends about the new episode when it airs.

Unsubscribe: Removing a friend from the watch party list if they no longer want to join.
*/

```

## Redux and the Observable Pattern

Redux applies the Observable pattern to state management. The core of Redux is the central state store, actions to update the state, and reducers that handle the state changes. The Redux store can be seen as an Observable object, and the components that subscribe to state changes are the observers.

Here is a sample implementation that closely resembles the actual Redux structure.

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

The createStore function is the starting point of Redux. It manages the application's state, the reducer that updates the state, and the listeners array that holds functions to notify on state changes. It returns a store object with methods similar to our Observable object.

- getState() returns the current state.
- subscribe(listener) registers a listener for state changes and returns a function to unsubscribe.
- unSubscribe(listener) removes a registered listener.
- dispatch(action) dispatches an action to update the state using the reducer and notifies all listeners.

## Example: React Counter Application

Using the Redux sample we created, let's build a simple counter application.

```jsx
const store = createStore((state = 0, action) => {
  const { type, payload } = action;
  switch (type) {
    case 'INCREMENT':
      return state + payload;
    case 'DECREMENT':
      return state - payload;
    default:
      return state;
  }
}, 0);
```

The reducer function takes the current state and action as parameters, and updates the counter state based on the action type. The initial state of the counter is set to 0.

- INCREMENT action: Increments the state by the payload value when the action type is 'INCREMENT'.
- DECREMENT action: Decrements the state by the payload value when the action type is 'DECREMENT'.
- Default state: Returns the current state if the action type is neither 'INCREMENT' nor 'DECREMENT'.

```jsx
// App function component
function App() {
  // useState hook to manage the component's state (count). Initial value is the store's state.
  const [count, setCount] = useState(store.getState());

  // onClick function: Called when the button is clicked. Dispatches an 'INCREMENT' action to increase the count.
  const onClick = () => {
    store.dispatch({ type: 'INCREMENT', payload: 1 });
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
      <div style={{ marginTop: '10px' }}>
        <button onClick={onClick}>+</button> {/* Button to increment the count */}
      </div>
    </div>
  );
}

```

- The App function component sets up the UI for the counter and displays the current count.
- The useState hook manages the component's count state, with the initial value obtained from the store's state.
- The onClick function dispatches an action to increment the count when the button is clicked.
- The useEffect hook subscribes to the store when the component mounts, ensuring that the component updates its state whenever the store's state changes.

Does this give you a better understanding of how the Observable pattern is used and how Redux works internally?

## What is the Provider Pattern?

Although vanilla Redux is sufficient for state management, most React applications use React-Redux. React-Redux provides utilities like Provider, useDispatch, and useSelector that make declarative development easier in complex applications.

Let's explore the Provider pattern and implement React-Redux to see how it can be used.

The Provider pattern is based on React's Context API, which allows you to share specific data (state) globally and make it easily accessible to any component that needs it. The Provider component provides values to the Context, and components that need these values subscribe to the Context.

Let's implement a simple Provider pattern using React Context API to share messages across the application.

- Step 1: Create a Context

```jsx
import React, { createContext } from 'react';

// Create a Context to store the message
const MessageContext = createContext();

```

- Step 2: Implement the Provider Component

```jsx
export function MessageProvider({ children }) {
  const message = 'Hello from Context!';

  return (
    <MessageContext.Provider value={message}>
      {children}
    </MessageContext.Provider>
  );
}
```

The MessageProvider component provides the message value to all its child components through the MessageContext.

- Step 3: Use the Context

```jsx
import React, { useContext } from 'react';
import { MessageContext } from './MessageContext';

function ChildComponent() {
  // Consume the message from the MessageContext
  const message = useContext(MessageContext);

  return <div>{message}</div>;
}

```

The ChildComponent uses the useContext hook to access the MessageContext value provided by MessageProvider.

## Implementing React-Redux

### Creating the Provider

React-Redux provides the Provider component to connect the Redux store to the React component tree.

Let's create a StoreProvider component that provides the store to all its child components using Context.

```jsx
const StoreContext = createContext(null);

export function StoreProvider({ store, children }) {
  return (
    <StoreContext.Provider value={store}>{children}</StoreContext.Provider>
  );
}

```

### Creating Custom Hooks

React-Redux provides hooks like useDispatch and useSelector to simplify accessing the store's state and dispatching actions from components. Let's implement these custom hooks.

```jsx
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

- The useDispatch hook allows components to access the store's dispatch function to dispatch actions.
- The useSelector hook allows components to select a part of the store's state and re-render when that state changes.

**Example: Counter Component with React-Redux**

```jsx
// Creating StoreContext: Uses React Context to manage the application's state (store) globally.
const StoreContext = createContext(null);

// StoreProvider component: Receives store and children as props, wrapping children with StoreContext.Provider.
export function StoreProvider({ store, children }) {
  return (
    <StoreContext.Provider value={store}>{children}</StoreContext.Provider>
  );
}

// useStore hook: Custom hook to get the store from the context.
function useStore() {
  const context = useContext(StoreContext);
  if (!context) {
    throw new Error('useStore must be used within a StoreProvider');
  }
  return context;
}

// useDispatch hook: Custom hook to return the store's dispatch function.
export function useDispatch() {
  const store = useStore();
  return store.dispatch;
}

// useSelector hook: Custom hook to select and return a part of the state.
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
    return unsubscribe; // Return cleanup function to unsubscribe.
  }, [store, selector]);

  return selectedState;
}

// App component: Uses StoreProvider to wrap Counter component and provide the store.
function App() {
  return (
    <div className="App">
      <StoreProvider store={store}>
        <Counter />
      </StoreProvider>
    </div>
  );
}

export default App;

```

```jsx
// Counter component defined in ./Counters.js

import { useDispatch, useSelector } from './App';

export default function Counter() {
  // Get the dispatch function using useDispatch hook.
  const dispatch = useDispatch();
  // Get the current state from the store using useSelector hook.
  const count = useSelector((state) => state);

  // onClick function: Called when the button is clicked, dispatches an 'INCREMENT' action.
  const onClick = () => {
    dispatch({ type: 'INCREMENT', payload: 1 });
  };

  // Render the UI, showing the current count and an increment button.
  return (
    <div className="App">
      <h1>COUNTER</h1> {/* Display the title. */}
      <span>{count}</span> {/* Display the current count. */}
      <div style={{ marginTop: '10px' }}>
        <button onClick={onClick}>+</button> {/* Button to increment the count. */}
      </div>
    </div>
  );
}

```

The Counter component uses useDispatch and useSelector to read the state from the store and dispatch the INCREMENT action. This entire process is facilitated by the store provided by the StoreProvider.

By implementing the core concepts of Redux, such as the Observable pattern and Provider pattern, we gained a better understanding of how Redux works. The Observable pattern involves observing changes in an object's state and notifying observers whenever there is a change. Redux uses this pattern to implement its core state management logic. This pattern is widely used not only in Redux but also in many other places, making it a valuable tool for solving event-handling problems.

The Provider pattern, based on React's Context API, is a method to provide state or functionality globally across an application. As we saw while implementing Redux-React, this pattern allows for easy access and management of global state, such as the Redux store, across the entire React component tree. The Provider pattern is also used in various component design patterns, like Compound Components, making it a highly versatile pattern that is worth mastering.

---

References:

- [**Building Redux from scratch**](https://medium.com/@guokai83524/building-redux-from-scratch-e12eb0e484c8)
- [**JavaScript**: *Build Yourself a Redux*](https://zapier.com/engineering/how-to-build-redux/)
- [**JavaScript**: *Let’s Write Redux!*](https://www.jamasoftware.com/blog/lets-write-redux/)
- [Provider pattern](https://www.patterns.dev/vanilla/provider-pattern)
- [Observer pattern](https://www.patterns.dev/vanilla/observer-pattern)