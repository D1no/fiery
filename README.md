# fiery 🔥

fiery 🔥 is the quickest and easiest way to use **Firebase Authentication** and **Firebase Realtime Database** in a React app. It uses latest React features and patterns such as [render props](https://reactjs.org/docs/render-props.html) and [hooks](https://reactjs.org/docs/hooks-intro.html).

## Installation

You can install fiery 🔥 from npm:

```
npm install --save fiery
```

A UMD version is also available:

```
<script src="https://unpkg.com/fiery@0.3.0/umd/fiery.js">
</script>
```

## Demo

[Try it out!](https://dtinth.github.io/fiery/#DistributedCounter)

<!-- scripts -->

```js
// Demo: DistributedCounter
// This demo app uses only Stateless Functional Components!

// Normal Firebase stuff...
//
const counterRef = firebase.database().ref('demos/counter')
const counterDecrement = () => counterRef.transaction(c => c - 1)
const counterIncrement = () => counterRef.transaction(c => c + 1)

function DistributedCounter() {
  // The `useFirebaseDatabase` hook makes this component automatically
  // subscribe to Firebase Realtime Database. When the data change,
  // this component is automatically re-rendered.
  //
  // This is possible thanks to the Hooks API, introduced in React 16.7.0-alpha.0.
  //
  const counterState = fiery.useFirebaseDatabase(counterRef)
  return (
    <div style={{ display: 'flex', alignItems: 'center' }}>
      <div>
        <UI.Button onClick={counterDecrement}>-</UI.Button>
      </div>
      <div style={{ textAlign: 'center', margin: '0 auto' }}>
        {counterState.loading ? (
          <UI.Loading message="Loading counter value" />
        ) : counterState.failed ? (
          <UI.ErrorMessage
            error={counterState.error}
            retry={counterState.retry}
          />
        ) : (
          <strong>{counterState.data}</strong>
        )}
      </div>
      <div>
        <UI.Button onClick={counterIncrement}>+</UI.Button>
      </div>
    </div>
  )
}

ReactDOM.render(
  <DistributedCounter />,
  document.getElementById('DistributedCounter')
)
```

```js
// Demo: GuestbookApp
// This demo app uses only Stateless Functional Components!
function GuestbookApp() {
  return (
    <section>
      <Nav />
      <GuestbookList />
      <GuestbookForm />
    </section>
  )
}

/**
 * The navigation bar
 */
function Nav() {
  return (
    <UI.NavBar title="My Guestbook">
      <UI.NavBar.Item label="Contact" />
      {/* Subscribe to the authentication state.
          We use the Render Props technique to localize updates
          to a single <UI.NavBar.Item /> component. */}
      <fiery.Auth>
        {/* Data is represented in 3 states: 'loading', 'completed' and 'error'.
            Use `fiery.unwrap` to handle all these 3 cases. */}
        {authState =>
          authState.loading ? (
            <UI.NavBar.Item label="Checking auth…" disabled />
          ) : authState.failed ? (
            <UI.NavBar.Item
              label="Auth checking failed"
              title={String(authState.error) + ' (click to retry)'}
              onClick={authState.retry}
            />
          ) : authState.data ? (
            <UI.NavBar.Item
              label={'Sign Out (' + authState.data.displayName + ')'}
              onClick={signOut}
            />
          ) : (
            <UI.NavBar.Item label="Sign in with GitHub" onClick={signIn} />
          )
        }
      </fiery.Auth>
    </UI.NavBar>
  )
}

// The `signIn` and `signOut` functions uses the normal Firebase auth functions.
// No new APIs to learn here!
//
function signIn() {
  firebase
    .auth()
    .signInWithPopup(new firebase.auth.GithubAuthProvider())
    .catch(e => window.alert(`Sorry, cannot sign in! ${e}`))
}
function signOut() {
  if (window.confirm('RLY SIGN OUT?')) firebase.auth().signOut()
}

/**
 * The list of guestbook entries.
 */
function GuestbookList() {
  // The `useFirebaseDatabase` hook makes this component automatically
  // subscribe to Firebase Realtime Database. When the data change,
  // this component is automatically re-rendered.
  //
  const guestbookState = fiery.useFirebaseDatabase(
    firebase
      .database()
      .ref('demos/guestbook')
      .orderByKey()
      .limitToLast(8)
  )
  return (
    <UI.EntryList>
      {guestbookState.loading ? (
        <UI.Loading message="Loading messages…" />
      ) : guestbookState.failed ? (
        <UI.ErrorMessage
          error={guestbookState.error}
          retry={guestbookState.retry}
        />
      ) : (
        Object.keys(guestbookState.data).map(key => (
          <UI.EntryList.Item
            key={key}
            text={guestbookState.data[key].text}
            name={guestbookState.data[key].name}
          />
        ))
      )}
    </UI.EntryList>
  )
}

/**
 * The form to submit a guestbook entry.
 */
function GuestbookForm() {
  // The `useFirebaseAuth` hook makes this component automatically
  // subscribe to Firebase Authentication state. When user signs in
  // or signs out, this component will automatically update.
  //
  const userState = fiery.useFirebaseAuth()
  return userState.loading ? (
    <UI.Loading message="Checking authentication status…" />
  ) : userState.failed ? (
    <UI.ErrorMessage error={userState.error} retry={userState.retry} />
  ) : userState.data ? (
    <UI.EntryForm onSend={text => submitForm(text, userState.data)} />
  ) : (
    <UI.AuthenticationWall onSignIn={signIn} />
  )
}

// Write to Firebase Realtime Database using the familiar Firebase SDK!
//
function submitForm(text, user) {
  firebase
    .database()
    .ref('demos/guestbook')
    .push({
      time: firebase.database.ServerValue.TIMESTAMP,
      name: user.displayName,
      text: text
    })
}

// Render the app...
//
ReactDOM.render(<GuestbookApp />, document.getElementById('GuestbookApp'))
```

## API Usage

fiery 🔥 provides both [hooks](https://reactjs.org/hooks)- and [render props](https://reactjs.org/docs/render-props.html)-based APIs ([rationale](https://twitter.com/dtinth/status/1055874999377047553)).

### `fiery.DataState<T>` — Representing remote data.

When loading data from remote sources, the data may not come immediately. In fiery, to represent this, we use a `DataState<T>`, which is a plain JS object with these properties:

- `loading` — A **boolean** representing whether data is being actively loaded or not.
- `failed` — A **boolean** representing whether data failed to load or not. **Note:** When retrying, the `failed` flag will stay `true` until new data has been loaded successfully.
- `data` — The data of type **T**. May be `undefined` if `loading || failed`.
- `error` — The **Error**. May be `undefined` if `!failed`.
- `retry` — A **function** that may be called to retry the operation. May be `undefined` if `!failed || loading`.

If you use TypeScript, our typings file can help preventing you from accessing the `data` in loading or failed state. Refer to this table.

| `loading` | `failed` | `data`          | `error`     | `retry`      | Remarks      |
| --------- | -------- | --------------- | ----------- | ------------ | ------------ |
| `true`    | `false`  | `T | undefined` | `undefined` | `undefined`  | Initial load |
| `true`    | `true`   | `T | undefined` | `Error`     | `undefined`  | Retrying     |
| `false`   | `false`  | `T`             | `undefined` | `undefined`  | Completed    |
| `false`   | `true`   | `T | undefined` | `Error`     | `() => void` | Error        |

### `fiery.useFirebaseAuth()`

Subscribe and use authentication state.

- Returns a `fiery.DataState<firebase.User | null>` wrapping a [`firebase.User`](https://firebase.google.com/docs/reference/js/firebase.User) object (if signed in) or `null` (if signed out).

### `fiery.Auth`

Render prop version of `fiery.useFirebaseAuth`.

Takes a single prop:

- `children` — A **function** that determines how the authentication state should be rendered.
  It will be called with a `RemoteDataState` wrapping a [`firebase.User`](https://firebase.google.com/docs/reference/js/firebase.User) object (if signed in) or `null` (if signed out).

### `fiery.useFirebaseDatabase(dataRef: firebase.database.Query)`

Subscribe and use data from Firebase Realtime Database.

- `dataRef` — A [`firebase.database.Reference`](https://firebase.google.com/docs/reference/js/firebase.database.Reference) representing the data reference to fetch.
- Returns a `fiery.DataState<any>` wrapping the data (if it exists) or `null` otherwise.

### `fiery.Data`

Render prop version of `fiery.useFirebaseDatabase`.

Takes two props:

- `dataRef` — A [`firebase.database.Reference`](https://firebase.google.com/docs/reference/js/firebase.database.Reference) representing the data reference to fetch.
- `children` — A **function** that determines how the data state should be rendered.
  It will be called with a `fiery.DataState<any>` wrapping the data (if it exists) or `null` otherwise.

### Looking for Firebase Firestore bindings?

Please contribute!

## Development

This project uses Yarn.

### Dependencies

To install dependencies, run:

```
yarn
```

### Development

Run:

```
yarn dev
```

This will run Rollup in watch mode and generate `umd/fiery.js`.
It will also re-generate the documentation site on change.

### Building

To build the library once, run:

```
yarn build
```

This will generate `umd/fiery.js`.

### Docs

The documentation website is generated from `README.md`.

To generate the docs, run:

```
yarn docs
```

## License

MIT.
