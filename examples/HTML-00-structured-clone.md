# `structuredClone`

> **NOTE**: The spec for HTML integrtion has not been written yet.

`structuredClone` of a Module object that refers to other module declarations will clone the whole graph:

```js
globalThis.evaluations = 0;

module dep {
    evaluations++;
    export default {};
}

module mod {
    import obj from dep;
    export default obj;
}

const modClone = structuredClone(mod);

assert((await import(mod)).default === (await import(mod)).default);
assert(evaluations === 1);

assert((await import(mod)).default !== (await import(modClone)).default);
assert(evaluations === 2);
```

If a module is imported twice in a graph, its copy will be deduplicated:
```js
globalThis.evaluations = 0;

module dep {
    evaluations++;
    export default {};
}

module a { export { default } from dep; }
module b { export { default } from dep; }

module mod {
    import A from a;
    import B from b;
    export default A === B;
}

const modClone = structuredClone(mod);

assert((await import(modClone)).default === true);
assert(evaluations === 1);
```

However, cloning a Module object twice will result in two different Module objects:
```js
globalThis.evaluations = 0;

module mod {
    evaluations++;
    export default {};
}

const clone1 = structuredClone(mod);
const clone2 = structuredClone(mod);

assert(clone1 !== clone2);
assert((await import(clone1)).default !== (await import(clone2)).default);
assert(evaluations === 2);
```

This behavior matches the existing behavior of cloning plain objects:
```js
const dep = {};
const obj = {
    a: { dep },
    b: { dep },
};

const clone = structuredClone(obj);
assert(clone.a.dep !== dep);
assert(clone.a.dep === clone.b.dep);

const clone2 = structuredClone(obj);
assert(clone.a.dep !== clone2.b.dep);
```
