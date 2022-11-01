# Module Declarations in Blocks and Functions

Like other declarations, module declarations are not restricted to the top-level of modules: they can appear anywhere a class declaration can appear.

```js
if (check) {
    module mod {}
    await import(mod);
}

function fn() {
    module mod {}
    return mod;
}
```

Module declarations are not hoisted out of the block scope where they are defined, so you cannot statically import them in outer scopes:
```js
{
    module mod {}
}
import mod; // Error! mod is not visible here
```

However, you can statically import them in other module declarations or expressions:
```js
{
    module mod {}
    module outer {
        import mod; // ok!
    }
    await import(outer);
}
```
