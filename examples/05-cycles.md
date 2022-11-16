# Cycles

Module declarations can introduce a new type of cycle: imports that depend on eachother.

```js
import { x } from y;
import { y } from x;
```

These cycles throw a linking error, because there is no meaningful way to resolve them. It's similar to how re-export cycles are an error:

```js
module a {
    export { x } from b;
}
module b {
    export { x } from a;
}

import { x } from a;
```

> **TODO**: The error in the first code snipped is not spec'ed yet, right now it just deadlocks.
