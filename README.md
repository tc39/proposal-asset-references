# Asset References

This repository contains a proposal to add syntax to get first class references to a module identity without actually loading or initializing that module. This can also be expanded to include other asset types. It's a way to access an asset relative to the current module. It reached [Stage 1 in November 2018](https://github.com/tc39/notes/blob/master/meetings/2018-11/nov-28.md).

This proposal builds on top of the [dynamic import proposal](https://tc39.github.io/proposal-dynamic-import/).

## Syntax

```js
asset Foo from "foo";
```

This syntax mirrors the `import` statement form, but only allows the single identifier form (no destructuring). No newline after `asset`. It allows a static declaration of a weak dependency on an asset relative to this module.

That "asset" could be another ECMAScript module. Unlike `import`, the `asset` statement doesn't actually load the other module. It's just a reference to it. This reference can be passed to any dynamic `import` call to actually load it and asynchronously resolve it to a module instance.

```js
async function Bar() {
  let foo = await import(Foo);
}
```

An asset doesn't have to refer to an ECMAScript module. It can also be passed to an external loader that can resolve it to an image, CSS, font, or any other resource needed by the program.

## Semantics

The `asset` statement creates a const binding for the name of the identifier. The module specifier conceptually gets resolved to the same canonical URL, file path or identity as it would've if it was used in an `import`. However, it doesn't actually trigger a load or initialization of that module if one hasn't happened elsewhere.

The binding gets immediately initialized with an empty object whose prototype is `AssetReference.prototype`. A new object gets created for each module statement to avoid creating an implicit back channel. It also allows an implementation to defer the actual canonical resolution until later. E.g. in Node.js canonicalizing can be an expensive operation that requires disk access.

The internal slot `[ReferencingModule]` of this object gets set to the current module. The internal slot `[AssetSpecifier]` gets set to the static module specifier.

### `import(AssetReference)`

If an asset reference is passed to the dynamic `import()` call, then it `HostImportModuleDynamically` using the `[ReferencingModule]` and `[AssetSpecifier]` internals slots of that asset reference.

This allows that module to be loaded dynamically if it hasn't already been loaded.

NOTE: The asset reference doesn't have to come from the same module as the `import()` call is in. This lets a third module be the one to actually initiate the loading.

### `AssetReference`

`AssetReference` is a constructor that throws if called. `AssetReference.prototype` is an empty object with a `constructor` field set to `AssetReference`.

This is effectively just an opaque object at this point but it could be expanded with more methods or fields in the future by ECMA262 (or possibly even the host environment).

### Loaders

An asset reference can also be passed into a module loaders defined by the host environment to perform more advanced optimizations. Such as synchronously test if it has been loaded, clear it from the cache entry, etc.

## Motivation

In this case special syntax is warranted because the specifier is relative to the module executing which causes two issues:

- This can't be built at a library level since there is no way to expose the relevant info from an `import()` call. The syntax is needed for the same reasons it is needed for `import()`.
- If you want to externalize the logic into a library by passing some context to it, it needs to be very ergonomic to actually pass that context from the module to that libary.

### Importing from Another File

If we look at only ECMAScript without additional access to the host environment, the main use case is to defer when importing and reimporting happens using a library.

Asset references lets us perform the import from an external library.

```js
import ResourceManager from "my-library/resource-manager";
asset utility from "utility";
export async function foo() {
  let u = await ResourceManager.load(utility);
  return u.bar();
}
```

This lets the external resource manager be responsible for the complex interactions that happen when I/O is involved. Such as:

- Passing additional arguments or setting up the loading environment.
- Handle I/O errors and fallback gracefully.
- Retry the fetch if fails the first time.
- Display a UI that asks the user to connect to WiFi, and retry again after a button is clicked.
- Pick one of multiple possible resources depending on what's available.

This is all possible by using dynamic `import` but since those are scoped to a local file, a lot of that logic gets hoisted out into the calling function. First class references lets us create abstractions for that loading behavior.

### CSS/Image Loading

It's a common pattern to refer to additional resources, such as image files, CSS files, font files from a JavaScript file. However, since there is no standard way of doing this there are various different pattern for achieving this.

In [Webpack](https://webpack.js.org/guides/asset-management/) this is done using `import` to refer to them as if they were JavaScript modules. Their export is defined by different conventions for what types should export. E.g. CSS files gets inserted into the HTML document as a side-effect. Images and font files export their URL as a string.

This has the downside that you can't have custom control over when to load individual files or how to use them differently in different modules. However, since there is no other standard way to refer to files relative to a module, it's what we're stuck with.

With asset references, the web platform could add them to the [`URL.createObjectURL()`](https://w3c.github.io/FileAPI/#dfn-createObjectURL) for example. That way they can be used as file with any web API.

```js
asset Logo from "./logo.gif";
async function loadLogo() {
  let img = document.createElement("img");
  img.src = URL.createObjectURL(Logo);
  return img;
}
```

This also lets libraries control how this image gets loaded, whether it gets decoded first or not, and retry if it fails, or fallback to an alternative image.

```js
asset Logo from "./logo.gif";
import ResourceManager from "my-library/resource-manager";
function loadLogo() {
  return ResourceManager.loadImage(Logo);
}
```

### `require.resolve`

Node.js has a mechanism to [resolve](https://nodejs.org/api/modules.html#modules_require_resolve_request_options) a specifier to a canonical file path for that specifier.

This has later been adapted by bundlers. Webpack has both [`require.resolve`](https://webpack.js.org/api/module-methods/#require-resolve) and [`require.resolveWeak`](https://webpack.js.org/api/module-methods/#require-resolveweak). However in this scenario, the constraints of static bundling tooling has put further requirements.

1) The return value is an opaque "module id" instead of a file path.
2) The specifier has to be defined as an inline literal so that you can resolve the dependency graph statically.

These IDs are then used to interact with the module loader/cache at runtime - such as `require.cache`.

These use cases still require these environments to keep the local scoped `require` hack and pseudo-static string resolution even when people are mostly moving on to completely ECMAScript compatible modules approach.

Other bundlers either doesn't have this mechanism or implement it slightly differently. It's a missing piece crying for standardization.

### Interacting with a Loader or Realm API

Depending on the host environment, the loader can be exposed to user code like libraries and frameworks.

The AssetReference can be used to interact with the loader where it would otherwise expect a canonicalized name.

It could be used to check the registry to see if a module record has already been fetched or initialized. This is used to only conditionally import a module if it is already in progress. Sometimes it is used to prevent a UI from initializing until it's known it could initialize synchronously. You could synchronously get a record from the cache, it has already been initialized which is sometimes a perf benefit or avoids generating temporary loading indicator UIs.

The AssetReference could be used to clear a specific entry from the cache, assuming that is allowed by the host. This is often useful when writing unit tests.

It could be used to implement hot reloading of individual modules and link them to specific asset resources. E.g. for development mode tooling.

This proposal doesn't actually define what loaders are allowed to do with this Asset Reference but we expect those proposals to take advantage of this proposal.

### Static Asset Specifier

The static specifiers isn't strictly a requirement since there isn't anything at run that needs to be resolve it in a web browser. However, even if it's not going to be in the standard, bundlers are going to require it anyway. To write interoperable code you need to know how to write it so either this is going to be a pseudo standard or we can make explicit syntax available that clarifies this constraint.

### Packaging

By providing a static specifier. Tooling like bundlers and packagers can know that a resource will be needed at some point, even though it doesn't have to be loaded eagerly. That means that it can use it to replace the URL with something internal to the packager tool and/or bundle it into the same ECMAScript file, tar file or stream it over HTTP/2 requests.

### Typed Specifiers

If an asset refers to a module, then type systems, e.g. Flow and TypeScript, can associate a type with a static specifier.

```js
// The type system knows that this file will export a signature. E.g. {bar: number}
// It can type this. E.g. AssetReference<Module<{bar: number}>>
asset Foo from "foo.js";
async function load() {
   // The type system now knows that this is wrong:
  let foo: {bar: string} = await import(Foo);
}
```

If this was an arbitrary string, it becomes less precise what that means and when it can be tracked.

### Lock Down Asset Access per Module

When dealing with arbitrary URLs it is difficult to lock down access in small sandboxed environments like SES. By providing an idiomatic way to reference assets relative to a module, we provide a way to create references that are only owned by that module until explicitly passed along, e.g. to a loading library. That same loading library can be used by two modules without giving access to the same asset to both modules.

### Changing Scheduling Priority

There is possible proposals to build scheduling priorities into the web platform. In those cases, this API can be used to change the priority of loading this module.

## Possible Additions

### Resource Hint

We should possibly allow the identifier to be excluded.

```js
asset "foo";
```

This is a no-op at runtime but it would give tooling like static bundlers a hint that a resource will be needed as is acquired or initialized using other means.

### Dynamic Asset Resolution

The primary form should be static form that mirrors the import statement form as the more common one. It provides bundlers and security scanners with a standard syntactic form that is guaranteed to receive a static string. It's easily explainable as opposed to the non-standard semi-static requirements that bundlers invent on top of dynamic import today.

The static from can also be used by browsers or web packaging tools to preload further dependencies by giving the hint earlier.

```js
asset Foo from "foo"; // statically defined dependency
async function Bar() {
  await import(Foo); // deferred loading
}
```

However, there are still cases where fully dynamically resolved URLs are useful and for those use cases we could add this additional syntax.

```js
let assetReference = import.resolve("./foo" + fileExtension);
```

## Alternative Solutions

### Module Syntax

An earlier version of this proposal used the `module` contextual keyword instead, and it was limited to be used with only asset references.

```js
module Foo from "foo";
```

This would provide some compatibility with the idea of nested modules.

```js
module "foo" {
  export default function() { };
}

import Foo from "foo";
// or
import("foo");
```

Another way to model this is:

```js
module Foo {

}
```

Getting a reference to a nested module in a different file:

```js
module { Foo } from "foo";
```

It might be bad to conflate these problems and still leaves the a gap to support assets other than modules.

### String instead of AssetReference Object

A possible alternative design could return a string of a fully resolved canonical path instead of an object. This is what `require.resolve` has traditionally done.

This suffers from a number of problems. It leaks too much about the implementation details of the environment. Not every environment will resolve to a URL path. E.g. Webpack uses a generated ID. Node.js uses a file path (which also differs between Unix/Windows based environments).

Another problem is that not every environment can synchronously resolve a canonical URL. E.g. Node.js needs to search the file system. So the return value cannot be guaranteed to be canonical at this point.

A string invites manual manipulation of the string path. This risks a number of security related bugs that can give access to arbitrary paths.

Another issue is that a string doesn't have a life time associated with it. If the asset has some kind of temporary memory representation associated with it, there is no way to clean it up on garbage collection since the string could be recreated. This is a problem that the [`URL.createObjectURL()`](https://w3c.github.io/FileAPI/#dfn-createObjectURL) API suffers from.

### Symbol instead of AssetReference Object

It's possible that we could use a Symbol instead of a string or object. This doesn't suffer from most of the issues of strings. It'd have to be a new Symbol to avoid synchronously canonicalizing it.

This suffers from some unclear garbage collection semantics. Symbols shouldn't need to require garbage collection semantics.

It also means that we can't put instance fields, properties or prototype methods on these objects in the future. This doesn't seem idiomatic to JavaScript.

### `import.meta`

Even today you could do relative resolution by passing `import.meta` along to a library function, because host implementations may expose the `url` field.

```js
import {load} from "library";
load(import.meta, "../relative/path");
```

This is not sufficient because there isn't access to the actual resolution algorithm of the host loader.

A previous ideas was to expose `resolve` on `import.meta` in the host environments.

```js
let Foo = import.meta.resolve("foo");
```

This namespace is defined by the host environment and there is no portable way to express this dependency.

Neither of these solution encourages to define the actual dependency in a static way that can be used by bundlers.

### Backup Syntax

If the grammar doesn't allow the contextual keyword `asset`, we could possibly use `import asset Foo from "foo";` instead. That is a worst case scenario though. People go through great lengths such as custom loaders and pseudo-syntax to avoid unnecessarily long syntax for something as frequent as this. Which can end up undermining the value of this feature as divergent pseudo-syntax emerges.

## Use Cases

### Node.js Resource Loading

We expect this mechanism to be used to be able to load and return an asset by never loading it into JS memory but staying in the C++ implementation for stream management.

Possible API:

```js
asset Image from "myimage.jpg";

export handleRequest(request) {
  request.respondWith(new Response(Image));
}
```

### React Module Loading

In React, the current way to use a component is to import it synchronously into the parent. Then the parent passes this reference to `createElement` to use that component.

```js
import {createElement} from "react";
import MyOtherComponent from "other-component";

function MyComponent() {
  return createElement(MyOtherComponent, {
    some: "data"
  });
}
```

In React, we would use this mechanism to allow the UI library control when loading of other components happen.

The recommended way to write a React component would be to declare an asset reference instead of directly importing it:

```js
import {createElement} from "react";
asset MyOtherComponent from "other-component";

function MyComponent() {
  return createElement(MyOtherComponent, {
    some: "data"
  });
}
```

That way the UI library is free to choose when it's best to load a component, control its scheduling, pass global configuration options (such as authentication/cancelation), synchronously resolve it if available, clear its cache entry, retry it if it fails, etc.

The best you can do today with standard:

```js
import {createElement, createWrapper} from "react";
const MyOtherComponent = createWrapper(() => import("other-component"));

function MyComponent() {
  return createElement(MyOtherComponent, {
    some: "data"
  });
}
```

However, this suffers from a number of issues. The string isn't guaranteed to be static. While it does allow the framework to control the loading and reloading. It doesn't provide a way to interop with the underlying cache or synchronously resolve an existing module. It also doesn't provide a way to add authentication or other options.

If we do end up adding a second argument to import it's possible that we could forward some configuration from the library to the import but that ends up being even more boilerplate.

```js
const MyOtherComponent = createWrapper(options => import("other-component", options));
```

That's a lot of boilerplate for something as common as importing, and it still doesn't provide all the capabilities needed.

With today's tooling it is unfortunately better to simply fallback to the non-standard `require.resolve`.

```js
const MyOtherComponent = require.resolve("other-component");
```

### Deno Resource Caching

Remote third-party modules are imported with URLs and cached in the filesystem at compile-time. However, these modules cannot include non-code assets in the dependency graph. Such assets must instead be downloaded at runtime. This harms startup performance and requires granting [net permission](https://deno.land/manual/getting_started/permissions) to the program, or necessitates an extra build step to do these things.

This proposal allows remote modules to declare static dependencies on relatively located non-code assets, which means they can be fetched and cached at compile-time just like JS/TS modules.

```ts
// This is a remote module at https://deno.land/x/foo@0.1.0/mod.ts.
// It depends on a plugin at https://deno.land/x/foo@0.1.0/plugins/foo.so.

asset fooPlugin from "./plugins/foo.so";

Deno.openPlugin(fooPlugin);

export function foo(): void {
  Deno.core.dispatch(Deno.core.ops()["op_foo"], argUi8);
}
```
