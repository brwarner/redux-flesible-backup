# Redux Flexible Backup

A fully-typed configurable backup/restore system for your Redux state.

Useful for:
* Saving and restoring state from localStorage (included!)
* Continually backing up your state to sessionStorage (included!)
* Creating a robust, memory-efficient undo/redo system (included!)
* Sending/recieving your state from a server or external database

**What do you mean 'configurable'?**

Sometimes, your Redux state contains data that you'd rather not save or transmit. That might include data which could be computed/recreated from the rest of the state, application data that doesn't need to be undone/redone, and more. 

When creating or loading backups using this package, you always specify a "backup interface". This contains per-slice configuration that decides what gets saved and what doesn't. You'll get to specify a save/load pair of functions for each slice in your state: meaning you could even compress the data if you'd like.

All this is fully-typed, meaning that if you're using Typescript, the results of any backups will be automatically computed from the save function's return values.

## Usage Example

```js
// Given a state
const state = { 
  // With one slice
  slice: {
    // With some data we'd want to save
    dataToSave: 'my data',

    // And some we wouldn't
    dataWeDontNeedToSave: 55
  }
};

// We can define a backup interface for this state
const backupInterface = {
  // That tells it how to store each slice
  slice: { 
    // That converts the slice to what we actually need to save
    save: (slice) => slice.dataToSave,

    // And tells the system how to restore from that save
    load: (stored) => ({dataToSave: stored, dataWeDontNeedToSave: 0})
  }
};

// And use that to create backups
const backup = createBackup(state, backupInterface);

// Which we can later restore
const restored = loadBackup({}, backupInterface, backup);

// Want an easy to use undo/redo system? Just wrap your reducer
const undoableReducer = createUndoableReducer(myReducer, backupInterface);

// Anytime you run an action with '/undoable/' in the type, a backup point is automatically saved
const newState = undoableReducer(state, { type: 'my/undoable/action' });

// Which you can undo/redo
const undoneState = undoableReducer(newState, UndoActions.undo);
const redoneState = undoableReducer(newState, UndoActions.redo);

// You can also iterate the history to see what's happened
for(const moment of iterateUndoHistory(newState)) {
  // .. do something with moment ..
}

```

## Backup Interfaces

The backup/restore feature is configured by backup interfaces. These are responsible for simplifying your state into the minimal object required to save and restore. This is especially useful if your state contains a lot of data that can be re-created from some minimal set or doesn't need to be stored.

There are two types of backup interfaces: slice and state.

A slice backup interface is an object with two methods: `save` and `load`.

The `save` method should take a slice object and return what must be stored. The return value really can be anything.

The `load` method takes whatever type you returned in `save` and must recreate the slice.

```js
// A simple slice backup interface
const sliceBackupInterface = {
  save: (slice) => slice.dataToSave,
  load: (stored) => ({dataToSave: stored, dataWeDontNeedToSave: 0})
}
```

A state backup interface is just an object mapping slices to slice backup interfaces.

```js
// State backup interface
const stateBackupInterface = {
  // Use sliceBackupInterface to backup/restore the `mySlice` slice
  mySlice: sliceBackupInterface
}
```

The fields of your state backup interface should match the field names of your state object.

Any slices in the state that don't match up with some slice backup interface in the state backup interface are ignored in save/load.

If you want your slice to just be copied in its entirety into the backup, use the premade `CopySliceBackupInterface`.

**NEW**: State Backup Interfaces can now also contain State Backup Interfaces as children, allowing you to recursively define state.

## Redux Toolkit Plugin

If you're using Typescript and want to reduce the typing hassle of using this plugin, you can use `createBackupSlice`, a wrapper for the Redux Toolkit `createSlice`. It adds an extra helper function to the resulting slice object to quickly create backup interfaces for your slice.

```js
// Use just like createSlice
const mySlice = createBackupSlice({
  name: 'mySlice',
  initialState: {},
  reducers: {}
});

// Type deduction is automatic. "state" will automatically be typed to the state type of your slice and stored will automatically be typed to the return value of the first function
export const mySliceBackupInterface = slice.createBackupInterface( state => ..., stored => ...);

// The type of mySliceBackupInterface is automatically deduced, so if we use it to create a state backup
const stateBackupInterface = { mySlice: mySliceBackupInterface };

// And call createBackup
const backup = createBackup(currentReduxState, stateBackupInterface);

// all of backup's fields will be automatically typed properly given the return value of the save function above in createBackupInterface
```

It can also automatically create properly named undoable actions (see Undo/Redo below) by adding reducers to the `undoableReducers` option.

```js
const mySliceWithSomeUndoableActions = createBackupSlice({
  name: 'mySlice',
  initialState: {},
  reducers: {
    basicAction: ....
  },
  undoableReducers: {
    undoableAction: ...
  }
});
```

This function plays nicely with Typescript, meaning `mySliceWithSomeUndoableActions.actions` will contain both `basicAction` and `undoableAction`.

# Undo/Redo

To use undo/redo, just wrap your reducer with `createUndoableReducer` (see the Usage Example above).

The resulting state will have three extra fields: `history`, `present`, and `future`.

`present` stores a backup of the state as it was the last time any undoable action was run.

`history` contains the past states. They are stored as diffs so you won't be able to access them directly.

`future` are redoable states created when you use `UndoActions.undo`. This is what will be restored, in order, when `UndoActions.redo` is called.

To iterate the history, use `iterateUndoHistory`. This will automatically unpack diffs and can be used in a `for ... of ...` loop.

# Storage

This package also includes a plugin for saving backups to local or session storage.

```js

// Save to storage using an interface
saveToStorage(localStorage, "STORAGE_KEY", myReduxState, myStateBackupInterface);

// Load from storage
const loadedState = loadFromStorage<MyStateType>({}, localStorage, "STORAGE_KEY", myStateBackupInterface);

```

If you want to save a backup of your state to session storage after each action, you can use the included session storage middleware.

```js

// Create an initial state by loading from the session
const initialState = loadInitialStateFromSession({}, myStateBackupInterface, "SESSION_KEY");

// Create middleware
const middleware = createSessionMiddleware<MyStateType>(myStateBackupInterface, "SESSION_KEY");

// Install it when creating your state
const extensions = compose(applyMiddleware(middleware));
const store = createStore(rootReducer, initialState, extensions);

```