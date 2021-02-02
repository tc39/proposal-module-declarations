# JavaScript Module Fragments

Champion: Daniel Ehrenberg
Stage 0

JavaScript module fragments are a syntax for named, inline JS modules, which can be used for bundling multiple modules into a single JavaScript file.

A quick example:

```js
// filename: app.js
module "#count" {
  let i = 0;

  export function count() {
    i++;
    return i;
  }
}

module "#uppercase" {
  export function uppercase(string) {
    return string.toUpperCase();
  }
}

import { count } from "#count";
import { uppercase } from "#uppercase";

console.log(count()); // 1
console.log(uppercase("daniel")); // "DANIEL"
```

This module, containing multiple module fragments, is referenced from an HTML file via a script tag:

```html
<script type=module src="./app.js"></script>
```

Module fragments can also be used outside of the file where they are defined, by referencing the fragment relative to the URL of the broader JS which contains it.

```html
<script type=module>
  import { uppercase } from "./app.js#uppercase";
  console.log(uppercase("yes")); // "YES"
</script>
```

In this case, because we never import `./app.js`, and only import `./app.js#uppercase`, the top-level statements with `console.log` in `app.js` are never run, and only one value is logged, not three.

## Syntax

`ModuleFragment` is a new nonterminal which can exist at the top level of a module, like `import` and `export` statements. Note that, as in the case of module blocks, there is no shared lexical scope between module fragments and the module that contains each other; they are simply side by side, like modules fetched from different URLs.

```
ModuleItem :
    ImportDeclaration
    ExportDeclaration
    StatementListItem[~Yield, ~Await, ~Return]
    <ins>ModuleFragment</ins>

ModuleFragment : module [no LineTerminator here] ModuleSpecifier { Module }
```

Module fragments may not be nested inside of other module fragments; they can only be defined at the top level.

This grammar does not *syntactically* conflict with [JS module blocks](https://github.com/tc39/proposal-js-module-blocks), which also use the `module` contextual keyword, but without a module specifier.

Host environments may limit the kinds of module specifiers permitted, or where module fragments are used at all. See [HTML integration](#html-integration) below for an example.

## Semantics

Each module fragment contained in a JS module is "top level":

- Syntactically, they can only be declared as a top-level statement of a module, not within another construct.
- They are available outside of the module they are contained in (by providing a module specifier with the appropriate context).
- Each bundled module contained in the module bundle has its own top-level lexical scope. There is no shared scope.

Like other modules, the host is required to return back the same value of that module when it is imported, or an error.

## HTML integration

Note: The following is framed to give details for HTML/the Web platform, but other platforms which aim to be analogous to the Web where appropriate (e.g., Node.js) may wish to follow these designs as well.

### Module fragments are named by URL fragments

When used on the Web, the string which follows the `module` keyword is required to be the character `#` followed by a [URL fragment](https://url.spec.whatwg.org/#concept-url-fragment).

```js
// In https://example.com/bundle.js
module "#counts" {
  let i = 0;

  export function count() {
    i++;
    return i;
  }
}
```

This module fragment `#counts` may be imported from within that particular JS file as `#counts`, or from anywhere else from an absolute URL, e.g., `"https://example.com/bundle.js#counts"` (or a relative URL where appropriate).

Some further rules limiting how module fragments can be used in the Web:
- No two module fragments may have the same specifier
- Modules which are inline `<script type=module> /* ... */ </script>` tags in HTML may not use module fragments (since there would be no clear URL for them)

### `import.meta.url`

The `import.meta.url` inside a module fragment is the module specifier of the surrounding module, followed by `#` and the fragment identifier. For example,

```js
// https://example.com/xyz.js
module "#fragment" { console.log(import.meta.url); }
import "#fragment";
```

The above code will log `https://example.com/xyz.js#fragment`.

### Relative URL resolution

Relative module specifiers within a module fragment are resolved just as if they were defined in the outer module. This behavior is the same as if they were calculated by `new URL(moduleSpecifier, import.meta.url)`. Module fragments can import other module fragments as well as top-level modules, based on these relative URLs. If a module specifier begins with `#`, it is interpreted to be a module fragment in the same outer module (as in the above examples). As an example of some further resolution:

```js
// https://example.com/a.js
module "#b" { import "/c.js#d"; }
```

```js
// https://example.com/c.js
module "#d" { document.write("d"); }
```

```html
<!-- https://example.com/index.html -->
<script type=module src="a.js#b"></script>
```

The document `index.html` will contain `d` when run.

### Module map semantics

JS module fragments, like all other modules, are kept track of in the [module map](https://html.spec.whatwg.org/#module-map). Whenever a JavaScript module which contains module fragments is imported (whether or not the fragment was imported), there is a module map entry made for each module fragment. The module fragments (as well as the top-level module) only have their dependencies loaded if they are *directly* imported; other entries may exist as a side effect, since they were found in the same file, but it's not time to fetch and parse their dependencies until that particular module is imported. This means that modules in the module map will need an extra bit to track whether they are "loaded" in this sense.

### Relationship to privacy/URL semantics concerns

Brave has [expressed concerns](https://brave.com/webbundles-harmful-to-content-blocking-security-tools-and-the-open-web/) about the possibility that bundling could be used to let servers remap URLs more easily, which cuts against privacy techniques for blocking tracking, etc. This proposal has significantly less expressivity than Web Bundles, making these issues not as big of a risk: JS module bundles are restricted to just same-origin JS, so they are analogous in scope to what is currently done with popular bundlers like webpack and rollup, not adding more power. Although it is possible to rotate/scramble fragment identifiers, these will not contain 

Martin Thompson of Mozilla has articulated a preference for bundling schemes to be based on URLs which accurately identify the identity of the resource. By identifying module fragments with URLs which include both where they were fetched from and the name of the component, the identity is clearly represented. (TODO: confirm this with MT)

## Relationship to resource bundles

JS module fragments provide a very limited subset of the functionality of [resource bundle loading](https://github.com/littledan/resource-bundles/blob/main/subresource-loading.md) behavior.

As a point-by-point comparison to how resource bundles and module fragments compare:
- **Level**: Resource bundle loading causes new assets to be available at the "fetch"/network level, engaging larger amounts of the browser. By contrast, JS module fragments are contained to the module loading mechanism, in a more limited scope.
- **Types**: JS module fragments can only contain JavaScript, but resource bundles can contain resources of any MIME type. Resource bundles can even support other headers alongside Content-Type for more information about individual responses, whereas JS module fragments have no syntax for any of this metadata.
- **Security/privacy considerations**: Because JS module bundles only affect how JavaScript is loaded, and do things equivalent to what bundlers do today, there's little additional security/privacy surface to worry about. By contrast, the story is more complicated with resource bundles.
- **HTTP caching**: JS module fragments are cached together in the HTTP cache with the enclosing module. There is no way for the browser to request just the fragments it is missing. By contrast, resource bundle loading divides resources into chunks, and only the chunks which are not present in cache are loading.
- **Versioning/cache busting**: Resource bundle loading allows chunk IDs to be rotated to cause some resources to be reloaded, even if they are present in cache, removing the need to change the URL. By contrast, JS module fragments do not provide such a mechanism, so a solution at some other level is needed.
- **Parsing performance**: Resource bundles are a breeze to parse because of their binary format which is clearly layered apart from their payloads, and an attention to detail to support both streaming and random-access efficiently. JS module fragments, instead, require parsing the whole JS file linearly, and do not support random access (and streaming is only possible if we weaken early error semantics).
- **Per-asset overhead**: Because resource bundle loading takes place at a more broad level in the browser, there are more codepaths that each resource hints (e.g., renderer-internal data structures, content blocking, etc). This makes them more difficult to optimize. By contrast, JS module fragments go through more specific code paths, so they may be easier to optimize for per-asset overhead.
- **Complexity**: Module fragments are a very simple mechanism. Tools and engines which know how to read and write JavaScript can be incrementally updated to support them. Resource bundles are a heavier lift, but bring certain benefits in exchange for that.

It's my (Dan Ehrenberg's) hypothesis at this point that, for best performance, JS module fragments should be *nested inside* resource bundles. This way, the expressiveness of resource bundles can be combined with the low per-asset overhead of JS module fragments: most of the "blow-up" in terms of the number of assets today is JS modules, so it makes sense to have a specialized solution for that case, which can be contained inside the JS engine. The plan from here will be to develop prototype implementations (both in browsers and build tools) to validate this hypothesis before shipping.

## Relationship to JS module blocks

[JS module blocks](https://github.com/tc39/proposal-js-module-blocks) are a separate concept for a module syntax which acts like a specifier: it can be imported with `import()` or `new Worker()`, but not with a static import statement, as it is not in the module map. Instead, it is a *key* in the module map. JS module blocks may be imported in other Realms.

Module blocks are more flexible in where they can appear, given that they don't need to be addressed from a fixed place like module fragments are. Some examples of where module blocks can appear and module fragments cannot:
- Within an `eval`
- In a legacy script
- In an inline module
- Nested inside of a function
- Dynamically created with the `ModuleBlock` constructor

This flexibility helps module blocks be deployed more flexibly, allowing, e.g., increased uptake of Workers in diverse environments.

Module fragments have a module specifier expressed as a string, which they can be imported from. Module blocks don't have such an identifying string. This makes module fragments more useful for bundling than module blocks. See more context in [this FAQ](https://github.com/tc39/proposal-js-module-blocks#can-module-blocks-help-with-bundling).

## Relationship to import maps

[Import maps](https://github.com/WICG/import-maps) can be used in conjunction with module fragments, in that the import map can redirect bare specifiers to be found in module fragments, rather than independent fetches. Mechanically: The lookup in the import map (which is done as part of "resolving a module specifier") precedes the interpretation of fragments in the module specifier (which are treated as part of the module map key). (TODO: add example of use together.)

## Next steps

The plan for this proposal is to present it for Stage 1 at a future TC39 meeting, and to prototype its use *in conjunction with* [resource bundle loading](https://github.com/littledan/resource-bundles/blob/main/subresource-loading.md) for a high-performance, native bundling solution on the Web platform.
