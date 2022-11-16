# Shadowing

Module declarations can be shadowed by other modules declarations declared in nested modules:

```js
module foo { export const x = 1; }

module bar {
    module foo { export const x = 2; }

    import { x } from foo;
    console.log(x); // 2
}

import { x } from foo;
console.log(x); // 1
```

Module declarations can also be shadowed by other variables: in that case they cannot be imported anymore.

```js
module foo { export const x = 1; }

import foo; // ok
await import(foo); // ok

{
    let foo = 2;
    await import(foo); // runtime error, we are trying to import "2"
}

module nested {
    import foo; // linking error when importing 'nested':
                // foo is a let-variable and not a module.

    let foo = 3;
}
import nested;
```

This means that in the following example `import bar` either imports the `bar` module exported from `"./container.js"` or fails, and the module which gets imported does not depend on the type of the `bar` export of `"./container.js"`:

```js
module bar { throw new Error("Unreachable!"); }

module foo {
    import { bar } from "./container.js";
    import { result } from bar;
    console.log(result);
}
import foo;
```
