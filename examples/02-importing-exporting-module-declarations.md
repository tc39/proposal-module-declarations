# Importing and Exporting Module Declarations

Module declarations can be exported, re-exported, and imported from other modules:

```js
// file-a.js

export module modY {
    export const y = 1;

    export module modX {
        export const x = 2;
    }
}

export { modX } from modY;
```

```js
// file-b.js

import { modX, modY } from "./file-a.js";
import { x } from modX;
import { y } from modY;

console.log({ x, y });
```

> **TODO**: In the current proposal spec, `export * from "./mod"` doesn't re-export module declarations.
