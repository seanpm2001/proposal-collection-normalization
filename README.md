# Collection {coerceKey, coerceValue}

This proposal seeks to add a `coerceKey` and `coerceValue` parameter to collection creation.

[Rendered Spec](https://tc39.github.io/proposal-collection-normalization/)

## Use cases

### Specialized maps

Given an application with User Objects it may be desirable to create collections based upon username and email for separate purposes.

```mjs
new Map(undefined, {
  coerceKey({email}) {
    return email;
  },
  coerceValue(state) {
    return state instanceof AccountState ? 
      state :
      new AccountState(state);
  }
});
```

```mjs
new Set(undefined, {
  coerceValue({username}) {
    return username;
  }
});
```

### Checked keys

It is a common occurrence to want to check types when performing operations on collections. This can be done during keying.

```mjs
new Map(undefined, {
  coerceKey(user) {
    if (user instanceof User !== true) {
      throw new TypeError('Expected User for key');
    }
    return user;
  }
});
```

## FAQ

### How do other languages handle this customized keying?

A collection of references can be found via [this document](https://docs.google.com/document/d/1qxSLyiButKocM6ENufhvnNJcZh18nDAWcFT2HlTJahQ/edit#).

Generally it falls into using container types. If you wanted to create a `Map` of `People` by `person.email`. You would implement a wrapper class `PersonByEmail` to use as your key, and others for keying of other aspects. Static typing and compiler/language enforced coercion can alleviate problems with misusing collections, but wrapping and unwrapping is manual in scenarios with dynamic typing that cannot be coerced automatically.

This proposal would provide a hook to do that manual wrapping and unwrapping without requiring the user of a collection to remain vigilant about properly marshalling keys before providing them to the collection.

### When are the normalization steps applied?

Normalization is applied when data is incoming to find the identity of the key location in `[[MapData]]` and when placing the value in `[[SetData]]` or `[[MapData]]`. e.g.

```mjs
const map = new Map([], {
  coerceKey: String
});
// stored using { [[Key]]: "1", [[Value]]: "one" } in map.[[MapData]]
map.set(1, 'one');
// looks for corresponding { [[Key]]: "1" } in map.[[MapData]]
map.has(1); // true
// functions directly exposing the underlying entry list are unaffected
[...map.entries()]; // [["1", "one"]]

const set = new Set([], {coerceValue: JSON.stringify});
// stored using { [[Value]]: '{"path": "/foo"}' } in set.[[SetData]]
set.add({path: '/foo'});
// looks for corresponding { [[Value]]: '{"path": "/foo"}' } in set.[[SetData]]
set.has({path: '/foo')};
// functions directly exposing the underlying entry list are unaffected
[...set]; // ['{"path": "/foo"}']
```

Normalization is not done when iterating or returning internal data, it is only done on parameters.

### Why would someone want to use `coerceValue` with Map?

A variety of use cases exist to normalize the values of map like structures in different APIs. Even if they do not directly use Map, we can se the utility of this pattern from existing DOM APIs.

* `URLSearchParams`
* `DomStringMap`
* `Header`

Node also has APIs that also normalize values such as `process.env`.

Normalizing values can avoid certain situations as well such as putting invalid values into a Map by either validation errors, coercion, or other means. Existing map like structures such as `require.cache` can produce error if you put the incorrect values in them. A normalization step allows the map to properly handle situations when unexpected values are inserted.

### Why are Sets only given `coerceValue`?

Sets are collections of values, and do not have a mapping operation from one value to another.

#### Why not call it `coerceKey` for Sets?

An audit of other languages was done with their documentation and APIs concerning Sets. The majority of terminology used was "elements"; however, the terms "keys" and "values" were also used. It was noted that whenever "keys" was used "values" was also used, but when "values" was used it did not always also use "keys". To match existing JS usage of terms "values" was chosen as the base for this name.

### Why not `value[Symbol.coerceKey]`?

Having specialized identity conflicts with the idea of having multiple kinds of specialized maps per type of value. It also would cause conflicts when wanting to specialize keys that are based upon primitives.

### Why not encourage extending collections?

1. This would be succeptible to prototype crawling such as:

```mjs
myCustomMap.__proto__.get.call(myCustomMap, key);
```

which would somewhat invalidate the idea of checking types of keys.

2. It prevents needing to synchronize all of the methods which is a fair amount of boilerplate and potential place for code going out of sync. It also means that your custom implementation will work even if new methods are added to collections in the JS standard library:

```mjs
class MyMap extends Map {
  constructor([...entries]) {
    super(entries.map(...));
  }
  delete(k) { ... }
  get(k) { ... }
  has(k) { ... }
  set(k, v) { ... }
}
```

If we add something like `emplace()` this code now needs to be updated or it will have bugs if people expect it to work like a standard Map.

This is roughly [the fragile base class problem](https://en.wikipedia.org/wiki/Fragile_base_class), where `Map` is the base class.

3. Even if this is a userland solution, it seems prudent to allow easier usage of maps. We should aim to alleviate developers without requiring that all new features have new kernel semantics. I spoke of this with respect to [expanding the standard library](https://docs.google.com/presentation/d/1QSwQYJz4c1VESEKTWPqrAPbDn_y9lTBBjaWRjej1c-w/view#slide=id.p).

4. Composition, while extending is nice it doesn't always allow for simple chaining and composition of features. If we introduce `RekeyableMap` as a concrete base class it may conflict with other base classes that may be introduced like if there was `InsertIfMissingMap`. Since both are base classes it would not allow both features to be combined easily.
