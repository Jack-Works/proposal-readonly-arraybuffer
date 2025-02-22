# Limited ArrayBuffer

## Status

Champion(s): *[Jack Works](https://github.com/Jack-Works)*

Author(s): *Jack Works*

Stage: 1

## Presentations

- [TC39 meeting notes on 82th tc39 meeting](https://github.com/tc39/notes/blob/main/meetings/2021-04/apr-21.md#read-only-arraybuffer-and-fixed-view-of-arraybuffer-for-stage-1) ([slides](https://docs.google.com/presentation/d/1TGLvflOG63C5iHush597ffKTenoYowc3MivQEhAM20w/edit?usp=sharing))
- [TC39 meeting notes on 106 tc39 meeting](https://github.com/tc39/notes/blob/main/meetings/2025-02/) ([slides](https://docs.google.com/presentation/d/1u6JsSeInvm6F4OrmCSLubtDvFVdjw1ESeE5-c_YflHE/))

## Problem to be resolved

All of the following are helpful to archive the minimal permission/information principle.

1. Cannot give others a read-only view to the `ArrayBuffer` and keep the read-write view internally.
2. Cannot give others a view that range is limited.

```js
function createView() {
    const buffer = new ArrayBuffer(128)
    const offset = 4
    const length = 16
    const view = new Uint8Array(buffer, offset, length)
    view[0] = 0x12
    return view
}

const view = createView()
// oops!
const wholeBufferView = new Uint8Array(view.buffer)
```

## Design goal

1. Read-only `TypedArray`/`DataView` to a read-write `ArrayBuffer`.
    1. Must not be able to construct a read-write view from a read-only view.
1. Range-limited `TypedArray`/`DataView` to a read-write `ArrayBuffer`.
    1. Must not be able to construct a bigger view range from a smaller view range.
1. Not adding too much complexity to the implementor.

## Proposed changes

1. New API on the `TypedArray`/`DataView` constructor.

```ts
interface TypedArrayConstructor {
    new (buffer: ArrayBuffer, options?: TypedArrayConstructorOptionsBag): Uint8Array<ArrayBuffer>;
}
interface DataViewConstructor {
    new (buffer: ArrayBuffer, options?: ViewConstructorOptionsBag): DataView<ArrayBuffer>;
}
interface ViewConstructorOptionsBag {
    byteOffset?: number;
    length?: number;
    readonly?: boolean;
    limited?: boolean;
}
```

1. If `readonly` or `limited` is set, the view returned does not have a `.buffer` property on it.

```ts
const buffer = new ArrayBuffer(1024);
const view = new Uint8Array(buffer, { limited: true, byteOffset: 12 });
view.buffer; // undefined
```

1. If `readonly` is set, the `TypedArray` returned is an exotic object (behaves like a [`ModuleNamespaceExoticObject`](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#module-namespace-exotic-object)), the returned `DataView` throws for set functions.

```ts
const buffer = new ArrayBuffer(8);
const view = new Uint8Array(buffer, { readonly: true });
view.buffer; // undefined
// view[0] = 1; // Error
Object.getOwnPropertyDescriptor(view, 0);
// { configurable: false, writable: true, enumerable: true, value: 0 }

const view2 = new DataView(buffer, { readonly: true });
view2.buffer; // undefined
// view2.setUint8(0, 0); // Error
```

1. Change the constructor of `TypedArray` and `DataView`, to accept `TypedArray` and `DataView` as source.

```ts
const buffer = new ArrayBuffer(128)
const view = new Uint8Array(buffer, { offset: 16, length: 32, readonly: true })

const dataView = new DataView(view)
dataView.setUint32(0, 1) // throws, inherits readonly, offset and length of `view`

const dataView2 = new DataView(view, 16, 16)
// inherits readonly, offset += 16 (32 in total), length = 16
```
