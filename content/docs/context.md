---
id: context
title: Context
permalink: docs/context.html
---

Context provides a way to pass data through the component tree without having to pass props down manually at every level.

In a typical React application, data is passed top-down (parent to child) via props, but this can be cumbersome for certain types of props (e.g. locale preference, UI theme) that are required by many components within an application. Context provides a way to share values like these between components without having to explicitly pass a prop through every level of the tree.

- [When to Use Context](#when-to-use-context)
- [Before You Use Context](#before-you-use-context)
- [API](#api)
  - [React.createContext](#reactcreatecontext)
  - [Context.Provider](#contextprovider)
  - [Class.contextType](#classcontexttype)
  - [Context.Consumer](#contextconsumer)
  - [Context.displayName](#contextdisplayname)
- [Examples](#examples)
  - [Dynamic Context](#dynamic-context)
  - [Updating Context from a Nested Component](#updating-context-from-a-nested-component)
  - [Consuming Multiple Contexts](#consuming-multiple-contexts)
- [Caveats](#caveats)
- [Legacy API](#legacy-api)

## When to Use Context {#when-to-use-context}

Context is designed to share data that can be considered "global" for a tree of React components, such as the current authenticated user, theme, or preferred language. For example, in the code below we manually thread through a "theme" prop in order to style the Button component:

`embed:context/motivation-problem.js`

Using context, we can avoid passing props through intermediate elements:

`embed:context/motivation-solution.js`

## Before You Use Context {#before-you-use-context}

Context is primarily used when some data needs to be accessible by *many* components at different nesting levels. Apply it sparingly because it makes component reuse more difficult.

**If you only want to avoid passing some props through many levels, [component composition](/docs/composition-vs-inheritance.html) is often a simpler solution than context.**

For example, consider a `Page` component that passes a `user` and `avatarSize` prop several levels down so that deeply nested `Link` and `Avatar` components can read it:

```js
<Page user={user} avatarSize={avatarSize} />
// ... which renders ...
<PageLayout user={user} avatarSize={avatarSize} />
// ... which renders ...
<NavigationBar user={user} avatarSize={avatarSize} />
// ... which renders ...
<Link href={user.permalink}>
  <Avatar user={user} size={avatarSize} />
</Link>
```

It might feel redundant to pass down the `user` and `avatarSize` props through many levels if in the end only the `Avatar` component really needs it. It's also annoying that whenever the `Avatar` component needs more props from the top, you have to add them at all the intermediate levels too.

One way to solve this issue **without context** is to [pass down the `Avatar` component itself](/docs/composition-vs-inheritance.html#containment) so that the intermediate components don't need to know about the `user` or `avatarSize` props:

```js
function Page(props) {
  const user = props.user;
  const userLink = (
    <Link href={user.permalink}>
      <Avatar user={user} size={props.avatarSize} />
    </Link>
  );
  return <PageLayout userLink={userLink} />;
}

// Now, we have:
<Page user={user} avatarSize={avatarSize} />
// ... which renders ...
<PageLayout userLink={...} />
// ... which renders ...
<NavigationBar userLink={...} />
// ... which renders ...
{props.userLink}
```

With this change, only the top-most Page component needs to know about the `Link` and `Avatar` components' use of `user` and `avatarSize`.

This *inversion of control* can make your code cleaner in many cases by reducing the amount of props you need to pass through your application and giving more control to the root components. However, this isn't the right choice in every case: moving more complexity higher in the tree makes those higher-level components more complicated and forces the lower-level components to be more flexible than you may want.

You're not limited to a single child for a component. You may pass multiple children, or even have multiple separate "slots" for children, [as documented here](/docs/composition-vs-inheritance.html#containment):

```js
function Page(props) {
  const user = props.user;
  const content = <Feed user={user} />;
  const topBar = (
    <NavigationBar>
      <Link href={user.permalink}>
        <Avatar user={user} size={props.avatarSize} />
      </Link>
    </NavigationBar>
  );
  return (
    <PageLayout
      topBar={topBar}
      content={content}
    />
  );
}
```

This pattern is sufficient for many cases when you need to decouple a child from its immediate parents.

However, sometimes the same data needs to be accessible by many components in the tree, and at different nesting levels. Context lets you "broadcast" such data, and changes to it, to all components below. Common examples where using context might be simpler than the alternatives include managing the current locale, theme, or a data cache.

## API {#api}

### `React.createContext` {#reactcreatecontext}

```js
const MyContext = React.createContext(defaultValue);
```

Creates a Context object. When React renders a component that subscribes to this Context object it will read the current context value from the closest matching `Provider` above it in the tree.

The `defaultValue` argument is **only** used when a component does not have a matching Provider above it in the tree. This can be helpful for testing components in isolation without wrapping them. Note: passing `undefined` as a Provider value does not cause consuming components to use `defaultValue`.

### `Context.Provider` {#contextprovider}

```js
<MyContext.Provider value={/* some value */}>
```

Every Context object comes with a Provider React component that allows consuming components to subscribe to context changes.

Accepts a `value` prop to be passed to consuming components that are descendants of this Provider. One Provider can be connected to many consumers. Providers can be nested to override values deeper within the tree.

All consumers that are descendants of a Provider will re-render whenever the Provider's `value` prop changes. The propagation from Provider to descendant consumers is updated even when an ancestor component skips an update.

Changes are determined by comparing the new and old values using the same algorithm as [`Object.is`](//developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is#Description).

> Note
>
> The way changes are determined can cause some issues when passing objects as `value`: see [Caveats](#caveats).

### `Context.displayName` {#contextdisplayname}

Context object accepts a `displayName` string property. React DevTools uses this string to determine what to display for the context.

For example, the following component will appear as MyDisplayName in the DevTools:

```js{2}
const MyContext = React.createContext(/* some value */);
MyContext.displayName = 'MyDisplayName';

<MyContext.Provider> // "MyDisplayName.Provider" in DevTools
<MyContext.Consumer> // "MyDisplayName.Consumer" in DevTools
```

## Examples {#examples}

### Dynamic Context {#dynamic-context}

A more complex example with dynamic values for the theme:

**theme-context.js**
`embed:context/theme-detailed-theme-context.js`

**themed-button.js**
`embed:context/theme-detailed-themed-button.js`

**app.js**
`embed:context/theme-detailed-app.js`

### Updating Context from a Nested Component {#updating-context-from-a-nested-component}

It is often necessary to update the context from a component that is nested somewhere deeply in the component tree. In this case you can pass **"context functions"** down as part of the context to allow consumers to update properties in the context itself.  In this example, we add a new `toggleTheme` property to the `ThemeContext` object and then assign a `customThemeToggler` implementation function to that property.

Remember: In JavaScript, functions are [**"first-class objects"**](https://stackoverflow.com/questions/705173/what-is-meant-by-first-class-object) which, among other things, means that ***functions themeselves*** can actually be passed around ***in an object***, just like any other property (string, number, etc). 🙂

[Here is a working code sandbox of this example.](https://codesandbox.io/s/react-hooks-update-context-from-nested-component-forked-t70up?file=/src/App.js)

**theme-context.js**
```js{1-3,6-10}
// We now add this new 'toggleTheme' property which points to a function that
// will be responsible for toggling the theme.
// Reminder: These default settings are never seen if overridden in /App.js
export const ThemeContext = React.createContext({
  theme: themes.dark,
  toggleTheme: () => {
    console.log(
      "Default 'toggleTheme' function has been called! " +
        "If I am overriden, I will never appear!"
    );
  }
});
```

**app.js**
```js{5-7,11,14,17}
import { useState } from "react";
import { ThemeContext, themes } from "./theme-context";
import ThemeTogglerButton from "./theme-toggler-button";

// ThemeContext's 'toggleTheme' property (again, a function), now needs to be
// actually implemented! You can assign any function to the context's
// 'toggleTheme' property. In our case, we assign 'customThemeToggler'
export const App = () => {
  const [themeContext, setThemeContext] = useState({
    theme: themes.light,
    toggleTheme: customThemeToggler
  });

  function customThemeToggler() {
    setThemeContext((oldContext) => ({
      theme: oldContext.theme === themes.light ? themes.dark : themes.light,
      toggleTheme: customThemeToggler
    }));
  }

  return (
    <ThemeContext.Provider value={themeContext}>
      <Content />
    </ThemeContext.Provider>
  );
};

function Content() {
  return (
    <div>
      <ThemeTogglerButton />
    </div>
  );
}
```

**theme-toggler-button.js**
```js{5-6,11}
import { useContext } from "react";
import { ThemeContext } from "./theme-context";

function ThemeTogglerButton() {
  // The Theme Toggler Button receives not only the theme
  // but also a 'toggleTheme' function from ThemeContext
  const themeContext = useContext(ThemeContext);

  return (
    <button
      onClick={themeContext.toggleTheme}
      style={{ backgroundColor: themeContext.theme.background }}
    >
      Toggle Theme
    </button>
  );
}

export default ThemeTogglerButton;
```

### Consuming Multiple Contexts {#consuming-multiple-contexts}

To keep context re-rendering fast, React needs to make each context consumer a separate node in the tree.

`embed:context/multiple-contexts.js`

If two or more context values are often used together, you might want to consider creating your own render prop component that provides both.

## Caveats {#caveats}

Because context uses reference identity to determine when to re-render, there are some gotchas that could trigger unintentional renders in consumers when a provider's parent re-renders. For example, the code below will re-render all consumers every time the Provider re-renders because a new object is always created for `value`:

`embed:context/reference-caveats-problem.js`


To get around this, lift the value into the parent's state:

`embed:context/reference-caveats-solution.js`

## Legacy API {#legacy-api}

> Note
>
> React previously shipped with an experimental context API. The old API will be supported in all 16.x releases, but applications using it should migrate to the new version. The legacy API will be removed in a future major React version. Read the [legacy context docs here](/docs/legacy-context.html).

