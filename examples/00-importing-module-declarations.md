# Importing Module Declarations

Module declarations can be imported by sibling or nested module declarations, and by the module where they are declared.

```js
module naturals {
    export const zero = {};
    export const succ = prev => ({ prev });

    module operations {
        import { zero, succ } from naturals;

        export const add = (x, y) => x === zero ? y : succ(add(x.prev, y));
    }

    export { add } from operations;
}

module conversion {
    import { zero, succ } from naturals;

    export const fromJS = n => n === 0 ? zero : succ(fromJS(n - 1));
    export const toJS = n => n === zero ? 0 : 1 + toInt(n.prev);
}

import { add } from naturals;
import { toJS, fromJS } from conversion;

const two = fromJS(2);
const three = fromJS(3);

console.log(`2 + 3 = ${toJS(succ(two, three))}`);
```
