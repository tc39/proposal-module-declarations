# Hoisting

Module declarations are hoisted to the beginning of the block scope, and can be imported before their declaration (this is similar to strict-mode function declarations):

```js
import { log } from loggers;

log(1, 2, 3);

module loggers {
    export const log = console.log;
}
```

```js
{
    mod; // ok
    module mod {}
}
mod; // ReferenceError: mod is not visible outside of the block
```

This helps when working with cross-module cycles, so that in the following example it doesn't matter which module is executed first:
```js
// comparison.js
import { numbers } from "./numbers-container.js";
import { three } from numbers;

export function isLessThan3(number) {
    return number <= three;
}
```
```js
// numbers-container.js
import { isLessThan3 } from "./comparison.js";
import { two } from numbers;

export const is2lessThan3 = isLessThan3(two);

export module numbers {
    export const zero  = 0;
    export const one   = 1;
    export const two   = 2;
    export const three = 3;
    export const four  = 4;
    // ...
}
```
