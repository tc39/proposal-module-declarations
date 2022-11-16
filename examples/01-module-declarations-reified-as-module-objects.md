# Module Declarations reified as Module Objects

$$ {\text{module declarations} \over \text{module expressions}} = {\text{function declarations} \over \text{function expressions}} $$

Except for the "they are static declaration parts", module declarations behave exactly like module expressions assigned to a `const` variable.

These two snippets are equivalent:

```js
module mod {
    export let y = 1;
}

mod instanceof Module; // true
await import(mod);
```
```js
const mod = module {
    export let y = 1;
};

mod instanceof Module; // true
await import(mod);
```

You cannot reassign to a module declaration:
```js
module mod {}

mod = 2; // Error!
```

> **TODO**: Should the Module objects be available in nested modules?
> ```js
> module outer {
>   module inner {
>     outer instanceof Module; // ?
>     export const mod = outer;
>   }
>   export { mod } from inner;
> }
> import { mod } from outer;
> mod === outer; // ?
> ```
