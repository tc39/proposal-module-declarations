# JavaScript Module Declarations

JavaScript module declarations (previously known as "module fragments") are a syntax for named, inline JS modules, which can be used for bundling multiple modules into a single JavaScript file.

## Status

Stage: 2

Champions:

- [Daniel Ehrenberg](https://github.com/littledan) ([Bloomberg](https://techatbloomberg.com/))
- [Nicolò Ribaudo](https://github.com/nicolo-ribaudo) (Igalia, in parternship with [eyeo](https://eyeo.com/) and [Bloomberg](https://techatbloomberg.com/))

## Motivation

JavaScript developers often write code in many small modules, and uptake of ECMAScript Modules (ESM, introduced in ES6/ES2015) is high as a source format. However, many small files--whether on the Web, servers, or other environments--has a high cost in terms of loading performance. For this reason, developers use tools called "bundlers" to emulate several ES modules in one (or a few) scripts or modules. Some examples are [webpack](https://webpack.js.org/), [rollup](https://rollupjs.org/guide/en/), [Parcel](https://parceljs.org/) and [esbuild](https://esbuild.github.io/).

The need for bundlers to entirely virtualize ES module semantics adds a lot of complexity to their implementation, and this cost increases over time, with new module features such as [top-level await](https://github.com/tc39/proposal-top-level-await/). It also has a cost in terms of runtime performance, as engines need to work through the virtualized code, and they cannot see the previous module structure. For example, modules would be a convenient point to divide up code for parallel bytecode generation, but this structure is not easily visible to JS engines today, if bundlers make everything one big script or module.

Some more general-purpose bundling format such as [resource bundles](https://github.com/littledan/resource-bundles) has a significant benefit over a JS-only bundling system because developers need to combine more things than just JavaScript in practice. An implementation of fetch-level resources is expected to have some degree of overhead, which may be suitable for images, WebAssembly and CSS. However, JavaScript tends to have much higher "blowup" in the number of modules than other resources, so a special-purpose JS-only format could tie in more cheaply at the module level, rather than the network level.

This proposal adds a syntax to JavaScript to allow for several JavaScript modules in one file. This format can be used as output by bundlers, with low overhead on execution, so that bundlers don't have to emulate as much, and JS engines can see what's going on. It's also convenient to be typed directly by JavaScript developers, and it should be low overhead to fit into existing workflows.

## Example

```js
// filename: app.js
module countModule {
  let i = 0;

  export function count() {
    i++;
    return i;
  }
}

export module uppercaseModule {
  export function uppercase(string) {
    return string.toUpperCase();
  }
}

import { count } from countModule;
import { uppercase } from uppercaseModule;

console.log(count()); // 1
console.log(uppercase("daniel")); // "DANIEL"
```

This module, containing multiple module declarations, is referenced from an HTML file via a script tag:

```html
<script type=module src="./app.js"></script>
```

Module declarations, if exported, can also be used outside of the file where they are defined.

```html
<script type=module>
  import { uppercaseModule } from "./app.js";
  import { uppercase } from uppercaseModule;
  console.log(uppercase("yes")); // "YES"
</script>
```

The `countModule` module, on the other hand, is not exported, and can not be used this way.

## Syntax

`ModuleDeclaration` is a new nonterminal which can exist at the top level of a module, like `import` and `export` statements, or of a script. Note that, as in the case of [module expressions](https://github.com/tc39/proposal-module-expressions), there is no shared lexical scope between module declarations and the module that contains each other; they are simply side by side, like modules fetched from different URLs.

```
ModuleItem[Yield, Await, Return] :
    ...
    <ins>ModuleDeclaration</ins>

ScriptItem[Yield, Await, Return] :
    ...
    <ins>ModuleDeclaration</ins>

ModuleDeclaration : module [no LineTerminator here] Identifier { ModuleBody? }
```

Module declarations may be nested inside of other module declarations.

## Semantics

- Module declarations can be imported statically.
- Module declarations are only available outside of the module they are contained in if they are exported explicitly.
- Each module declaration has its own top-level lexical scope. There is no shared scope.

If a module declaration is imported multiple times, the same module "instance" is returned, just like with modules declared in separate JS files. In other words, module declarations are singletons.

## HTML integration

Note: The following is framed to give details for HTML/the Web platform, but other platforms which aim to be analogous to the Web where appropriate (e.g., Node.js) may wish to follow these designs as well.

### `import.meta.url`

The `import.meta.url` inside a module declaration is the module specifier of the surrounding module. For example,

```js
// https://example.com/xyz.js
module mod { console.log(import.meta.url); }
import mod;
```

The above code will log `https://example.com/xyz.js`.

Relative module specifiers within a module declaration are resolved just as if they were defined in the outer module. This behavior is the same as if they were calculated by `new URL(moduleSpecifier, import.meta.url)`.

This behavior follows from the semantics proposed for [module expressions](https://github.com/tc39/proposal-module-expressions).

## FAQ

### Does this proposal meet privacy concerns about bundling?

Brave has [expressed concerns](https://brave.com/webbundles-harmful-to-content-blocking-security-tools-and-the-open-web/) about the possibility that bundling could be used to let servers remap URLs more easily, which cuts against privacy techniques for blocking tracking, etc. This proposal has significantly less expressivity than Web Bundles, making these issues not as big of a risk:

JS module bundles are restricted to just same-origin JS, so they are analogous in scope to what is currently done with popular bundlers like webpack and rollup, not adding more power. Although it is possible to rotate/scramble declaration identifiers, it is reasonable to treat the whole outer module containing several module declarations as a unit, with content blockers targeting either all or none of it.

Martin Thompson of Mozilla has articulated a preference for bundling schemes to be based on URLs which accurately identify the identity of the resource. As module declarations can not be loaded directly, but only through the outer module, and the outer module is fetched by its URL, the identity is clearly represented. (TODO: confirm this with MT)

### Why have module declarations, rather than just focusing on general-purpose resource bundles?

Module declarations provide a very limited subset of the functionality of [resource bundle loading](https://github.com/littledan/resource-bundles/blob/main/subresource-loading.md) behavior.

As a point-by-point comparison to how resource bundles and module declarations compare:
- **Level**: Resource bundle loading causes new assets to be available at the "fetch"/network level, engaging larger amounts of the browser. By contrast, module declarations are contained to the module loading mechanism, in a more limited scope.
- **Types**: Module declarations can only contain JavaScript, but resource bundles can contain resources of any MIME type.
- **Metadata**: Resource bundles can even support other headers alongside Content-Type for more information about individual responses, whereas module declarations have no syntax for any of this metadata.
- **Security/privacy considerations**: Because JS module bundles only affect how JavaScript is loaded, and do things equivalent to what bundlers do today, there's little additional security/privacy surface to worry about. By contrast, the story is more complicated with resource bundles.
- **HTTP caching**: Module declarations are cached together in the HTTP cache with the enclosing module. There is no way for the browser to request just the declarations it is missing. By contrast, [resource bundle loading](https://github.com/littledan/resource-bundles/blob/main/subresource-loading.md) divides resources into chunks, and only the chunks which are not present in cache are loaded.
- **Versioning/cache busting**: Resource bundle loading allows chunk IDs to be rotated to cause some resources to be reloaded, even if they are present in cache, removing the need to change the URL. By contrast, module declarations do not provide such a mechanism, so a solution at some other level is needed.
- **Parsing performance**: Resource bundles are a breeze to parse because of their binary format which is clearly layered apart from their payloads, and an attention to detail to support both streaming and random-access efficiently. Module declarations, instead, require parsing the whole JS file linearly, and do not support random access (and streaming is only possible if we weaken early error semantics).
- **Per-asset overhead**: Because resource bundle loading takes place at a more broad level in the browser, there are more codepaths that each resource hints (e.g., renderer-internal data structures, content blocking, etc). This makes them more difficult to optimize. By contrast, module declarations go through more specific code paths, so they may be easier to optimize for per-asset overhead.
- **Complexity**: Module declarations are a very simple mechanism. Tools and engines which know how to read and write JavaScript can be incrementally updated to support them. Resource bundles are a heavier lift, but bring certain benefits in exchange for that.

It's my (Dan Ehrenberg's) hypothesis at this point that, for best performance, module declarations should be *nested inside* resource bundles. This way, the expressiveness of resource bundles can be combined with the low per-asset overhead of module declarations: most of the "blow-up" in terms of the number of assets today is JS modules, so it makes sense to have a specialized solution for that case, which can be contained inside the JS engine. The plan from here will be to develop prototype implementations (both in browsers and build tools) to validate this hypothesis before shipping.

### Why have this proposal and module expressions as two separate things, rather than one common language feature?

The [module expressions](https://github.com/tc39/proposal-module-expressions) proposal introduces the expressions form of inline modules. Being expressions, they are inherently dynamic: they can be imported with `import()` or `new Worker()`, but not with static import statements.

Module declarations lift this restriction: they can be imported statically if they appear at the top level of a module. This makes module declarations more useful for bundling than module expressions. See more context in [this FAQ](https://github.com/tc39/proposal-module-expressions#can-module-blocks-help-with-bundling).

We are developing these two features as separate proposals because module declarations present additional complexity around static import statements that module expressions don't need, and module expressions have their own [motivations](https://github.com/tc39/proposal-module-expressions#problem-space) independent from module declarations. Module declarations inherit different design decisions from module expressions, so the advancement of this proposal in its current shape depends on the evolution of module expressions.

<!--

### Does this proposal work with import maps?

[Import maps](https://github.com/WICG/import-maps) can be used in conjunction with module declarations, in that the import map can redirect bare specifiers to be found in module declarations, rather than independent fetches. Mechanically: The lookup in the import map (which is done as part of "resolving a module specifier") precedes the interpretation of declarations in the module specifier (which are treated as part of the module map key). (TODO: add example of use together.)

-->

## Next steps

The plan for this proposal is to present it for Stage 1 at a future TC39 meeting, and to prototype its use *in conjunction with* [resource bundle loading](https://github.com/littledan/resource-bundles/blob/main/subresource-loading.md) for a high-performance, native bundling solution on the Web platform.
