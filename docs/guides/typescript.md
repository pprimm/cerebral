# Typescript

To get full type safety with Cerebral you need to install the **@cerebral/fluent** addon.

`npm install @cerebral/fluent`

This addon exposes its own set of Cerebral APIs that allows for type safety. In this guide we will go through how you would structure an application with this API.

## Tsconfig

A typical **tsconfig.json** looks like this:

```js
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "fluent": ["src/fluent.ts"]
    },
    "outDir": "dist",
    "module": "esnext",
    "target": "es5",
    "sourceMap": true,
    "strict": true,
    "strictPropertyInitialization": false,
    "jsx": "react"
  },
  "exclude": ["node_modules"]
}
```

The **fluent.ts** file will be explained below.

## Controller

Instantiate the **Controller** exposed by `fluent`:

```ts
import Devtools from 'cerebral/devtools'
import { Controller } from '@cerebral/fluent'
import { State, Signals } from './fluent'

export const controller = Controller<State, Signals>(app.module, {
  devtools: Devtools(...),
  // If you are migrating from existing app, add this option to
  // allow for the legacy state api
  useLegacyStateApi: true
})
```

It is optional to define the **State** and **Signals** type on the controller. This is only useful if you point to the state or the signal on the controller itself:

```ts
controller.state
controller.signals
```

## Exposing the controller to React

You expose the controller using the **Container** from **fluent**:

```ts
import * as React from 'react'
import { render } from 'react-dom'
import { controller } from './controller'
import { Container } from '@cerebral/fluent'
import { App } from './components/App'

render(
  <Container controller={controller}>
    <App />
  </Container>,
  document.querySelector('#app')
)
```

You will later see examples of how you actually connect components.

## Creating the fluent file

To ease development it is recommended to create a **fluent.ts** file which configures your application. It is recommended to create an alias to your fluent
file for easier imports. This can be done in your **tsconfig.json** file:

```ts
{
  "compilerOptions": {
    "baseUrl": ".", // This must be specified if "paths" is.
    "paths": {
      "fluent": ["src/fluent.ts"] // This mapping is relative to "baseUrl"
    }
  }
}
```

From now on when we point to `'fluent'` it is this file:

```ts
import { IContext, IBranchContext, SequenceFactory, SequenceWithPropsFactory, ConnectFactory } from '@cerebral/fluent'
import { Provider as RouterProvider } from '@cerebral/router';
import { State, Signals } from './app/types'

// Create an interface where you compose your providers together
interface Providers {
  router: RouterProvider,
  state: State
}

// Create a type used with your sequences and actions
export type Context<Props = {}> = IContext<Props> & Providers;

// This type is used when you define actions that returns a path
export type BranchContext<Paths, Props = {}> = IBranchContext<Paths, Props> & Providers;

// This function is used to connect components to Cerebral
export const connect = ConnectFactory<State, Signals>();

// This function is used to define sequences
export const sequence = SequenceFactory<Context>();

// This function is used to define sequences that expect to receive some initial
// props
export const sequenceWithProps = SequenceWithPropsFactory<Context>();
```

## Creating a module with types

You will typically create a separate file for the types. This will hold the types for the state and signals of the given module.

*src/app/types.ts*
```ts
import { Dictionary, ComputedValue } from '@cerebral/fluent'
import * as signals from './sequences'

// This is a shortcut where all exported sequences are
// defined as signals
export type Signals = {
  [key in keyof typeof signals]: typeof signals[key]
};

// Alternatively you would have to do this for each signal
export type Signals = {
  doThis: typeof signals.doThis
};

export type State = {
  foo: string,
  stringDictionary: Dictionary<string>,
  isAwesome: ComputedValue<boolean>,
  upperFoo: string
};
```

## Creating the app module

*src/app/index.ts*
```ts
import { Module, Dictionary, Computed } from '@cerebral/fluent'
import Router from '@cerebral/router'
import * as a from './modules/a'
import * as b from './modules/b'
import * as signals from './sequences'
import * as computed from './computed'
import { State } from './types'

const state: State = {
  // Just a normal state value
  foo: 'bar',
  // Dynamic objects, meaning you want to add/remove
  // keys will need to be a Dictionary.
  stringDictionary: Dictionary({}),
  // Computed values receives both the module state and the
  // root state. Normally you will use this for computational things like
  // sorting, filtering etc. Where you want to avoid doing the calculation
  // everytime the value is used. It is used like this: state.isAwesome.get()
  isAwesome: Computed(computed.isAwesome),
  // A getter allows you to make state values derived from other state
  // values. There is no caching involved, but if used it will be observed for
  // changes like normal values. It is used like this: state.upperFoo
  get upperFoo () {
    return this.foo.toUpperCase()
}
}

export const module = Module({
  state,
  signals,
  modules: {
    a: a.module,
    b: b.module,
    router: Router({...})
  }
})
```

By defining your state in its own variable you will get easier to read error reporting from Typescript.


## Scaling up to submodules

The module definition above has two custom submodules. When you have submodules you will need to compose in the complete state and signals. You do this in the **fluent** file like this:

```ts
import { IContext, IBranchContext, SequenceFactory, SequenceWithPropsFactory, ConnectFactory } from '@cerebral/fluent'
import { Provider as RouterProvider } from '@cerebral/router';
import * as app from  './app/types'
import * as admin from './app/modules/admin/types'
import * as dashboard from './app/modules/dashboard/types'

type State = app.State & {
  admin: admin.State,
  dashboard: dashboard.State
}

type Signals = app.Signals & {
  admin: admin.Signals,
  dashboard: dashboard.Signals
}

interface Providers {
  router: RouterProvider,
  state: State
}

export type Context<Props> = IContext<Props> & Providers;

export type BranchContext<Paths, Props> = IBranchContext<Paths, Props> & Providers;

export const connect = ConnectFactory<State, Signals>();

export const sequence = SequenceFactory<Context>();

export const sequenceWithProps = SequenceWithPropsFactory<Context>();
```

The **fluent** file is where you describe your complete application with types.

## Creating sequences

```ts
// remember this is alias to the fluent.ts file
import { sequence } from 'fluent'

export const changeFoo = sequence(s => s
  .action(({ state }) => { state.foo = 'bar2' })
)
```

This sequence gives you full type safety and autosuggestions. If anything is changed you will be notified by Typescript if your sequence does not work. Though inlining actions like this works it is recommended to split them out. The reason is that your sequences will read better and you actions will be composable with other sequences by default. Like this:

```ts
import { sequence } from 'fluent'
import * as actions from './actions'

export const changeFoo = sequence(s => s
  .action(actions.changeFoo)
)
```

You can branch out to paths by using the **branch** method:

```ts
import { sequence } from 'fluent'

export const submitUser = sequence(s => s
  .action(actions.setSubmittingUser(true))
  .branch(actions.submitNewUser)
  .paths({
    success: s => s
      .action(actions.addNewUser),
    error: s => s
      .action(actions.showError)
  })
  .action(actions.setSubmittingUser(false))
)
```

If your sequence expects to have props available you can define that:

```ts
import { sequenceWithProps } from 'fluent'
import * as actions from './actions'

export const changeFoo = sequenceWithProps<{ foo: string }>(s => s
  .action(actions.changeFoo)
)
```

You can also define expected props to be produced through the sequence:

```ts
import { sequence, sequenceWithProps } from 'fluent'
import * as actions from './actions'

export const doThis = sequence<{ foo: string }>(s => s
  // This action must return { foo: 'some string' } for the
  // sequence to be valid
  .action(actions.doSomething)
)

export const changeFoo = sequenceWithProps<{ foo: string }, { bar: number }>(s => s
  // This action must return { bar: 123 } to give a valid sequence
  .action(actions.changeFoo)
)
```

Any sequences used as signals will require the signal to be called with the defined props, for example:

```ts
import { sequence, sequenceWithProps } from 'fluent'
import * as actions from './actions'

export const doThis = sequenceWithProps<{ foo: string }>(s => s
  .action(actions.doSomething)
)
```

When you call the signal it will require you to call it with:

```ts
signals.doThis({
  foo: 'bar'
})
```

## Creating actions

```ts
import { Context, BranchContext } from 'fluent'
import { User } from './types'

// Now you have made this function composable with any
// other sequence, meaning that you can safely reuse it
// wherever you want and instantly be notified if there is a mismatch,
// for example that the action will indeed not receive the name property
// by a previous action, sequence or calling the signal
// Context<Props>
export function changeNewUserName ({ state, props }: Context<{ name: string }>) {
  state.newUserName = props.name
}

// With the BranchContext type you ensure that this action has
// the defined paths available in the sequence it is composed into
// BranchContext<Paths, Props>
export function submitNewUser ({ state, http, path }: BranchContext<{
  success: { user: User },
  error: {}
  }>) {
  return http.post('/users', {
    userName: state.newUserName
  })
    .then(response => path.success({ user: response.result }))
    .catch(() => path.error({}))
}
```

Both **Context** and **BranchContext** as optional **Props**.

## Connecting components

The **connect** factory you defined in the *fluent.ts* file is used to connect to components like this:

```ts
import * as React from 'react'
import { connect } from 'fluent'

type ExternalProps {
  foo: string
}

export default connect<ExternalProps>()
  .with(({ state, signals, props }) => ({
    foo: state.foo,
    onClick: signals.doThis
  }))
  .to(
    function MyComponent ({ foo, onClick }) {
      return <div></div>
    }
  )
```

The **ExternalProps** are used when the component receives props from a parent. This is optional. It is important to take notice that the **with** method has to return the exact observable values you are using in your component. Meaning that:

```ts
import * as React from 'react'
import { connect } from 'fluent'

export default connect()
  .with(({ state }) => ({
    user: state.user
  }))
  .to(
    function UserName ({ user }) {
      return <div>{user.name}</div>
    }
  )
```

would actually not work. You would have to:

```ts
import * as React from 'react'
import { connect } from 'fluent'

export default connect()
  .with(({ state }) => ({
    user: { name: state.user.name }
  }))
  .to(
    function UserName ({ user }) {
      return <div>{user.name}</div>
    }
  )
```

This is because you have to grab (observe) the values being used in the component. This has the benfit of being explicit and allows for easy extending the connect to work with other view layers. You could also do:

```ts
import * as React from 'react'
import { connect } from 'fluent'

export default connect()
  .with(({ state }) => ({
    user: { ...state.user }
  }))
  .to(
    function UserName ({ user }) {
      return <div>{user.name}</div>
    }
  )
```

As this would indeed "get" all the properties on the user, starting to observe them.

To connect to a class:

```ts
import * as React from 'react'
import { connect } from 'fluent'

export default connect()
  .with(({ state }) => ({
    user: { ...state.user }
  }))
  .toClass(props =>
    class UserName extends React.Component<typeof props> {
      render () {
        return <div>{this.props.user.name}</div>
      }
    }
  )
```

Connecting to a class gives a callback with the prop which you can **typeof** into the component class. This gives type safety and auto suggestions on the props in the component itself.

## Dictionary
Typically your state objects are used to define other specifc state, like:

```ts
type State = {
  settings: {
    isAwesome: boolean,
    hasBananas: boolean
  }
}

const state: State = {
  settings: {
    isAwesome: true,
    hasBananas: false
  }
}
```

The **settings** object will never have keys added or removed from it. But a state like:

```ts
type User = {
  name: string
}

type State = {
  users: { [id: string] : User}
}

const state: State = {
  users: {}
}
```

Will dynamically have keys added and removed. That is where you have to use a **Dictionary**:

```ts
import { Dictionary } from '@cerebral/fluent

type User = {
  name: string
}

type State = {
  users: Dictionary<User>
}

const state: State = {
  users: Dictionary({})
}
```

The dictionary is an **ObservableMap** from Mobx. It has its own API surface:

```ts
function someAction ({ state }) {
  const user = state.users.get('123')
  state.users.set('123', { name: 'John ' })
  state.users.delete('123')
  state.users.clear()
  state.users.merge({
    '456': { name: 'Bob' },
    '789': { name: 'Derek' }
  })
}
```

Whenever you have this dynamic nature, use a **Dictionary**. Typescript will of course give you hints about this when you point to it.

## Computing values

Typically you will just use a getter when you are deriving state:

```ts
type State = {
  foo: string,
  upperFoo: string
}

const state: State = {
  foo: 'bar',
  get upperFoo() {
    return this.foo.toUpperCase()
  }
}
```

There is really no computational power needed to do this. When you have more complex getters it is a good idea to move them into their own files:

```ts
import * as getters from './getters'

type State = {
  foo: string,
  upperFoo: string
}

const state: State = {
  foo: 'bar',
  get upperFoo() {
    return getters.upperFoo(this)
  }
}
```

When you are doing more heavy calulations though it is nice to compute. So for example if we created a color based on the text of **foo**:

```ts
import stringToColor from 'string-to-color'
import { ComputedValue } from '@cerebral/fluent

type State = {
  foo: string,
  fooColor: ComputedValue<string>
}

const state: State = {
  foo: 'bar',
  fooColor: Computed((state) => {
    return stringToColor(state.foo)
  })
}
```

There is really no reason to recalculate the color everytime we grab **fooColor**, unless **foo** itself actually changed. This is where a computed helps. It caches the returned value until any of the depending state has changed.
