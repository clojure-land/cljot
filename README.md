# Cljot

[![Build Status](https://travis-ci.com/Pietrorossellini/cljot.svg?branch=master)](https://travis-ci.com/Pietrorossellini/cljot)
[![codecov](https://codecov.io/gh/Pietrorossellini/cljot/branch/master/graph/badge.svg)](https://codecov.io/gh/Pietrorossellini/cljot)
[![Clojars Project](https://img.shields.io/clojars/v/cljot.svg)](https://clojars.org/cljot)

Operational transformation format with support for concurrent rich text editing.

## Approach and compatibility

Cljot is designed for concurrent editing with centralized decision-making,
i.e., it is primarily meant to be used with client-server topology where the canonical order of concurrent edits is decided by a central server.
This poses some restrictions on true concurrency and entails an acknowledgement-based application of state changes.

Cljot is also heavily inspired by the [Quill Delta format](https://github.com/quilljs/delta).
Consequently, Cljot is mostly compatible with Deltas created with Quill Delta.

## Usage

Cljot is based on deltas, which are vectors of operations applied successively.
Three operations are provided:
1) Insert: inserts new content, either text or non-opinionated map of attributes
2) Retain: either keeps a given range intact or applies attributes to it
3) Delete: removes a range

With these three operations, any number of different documents can flexibly be expressed.
This covers representing both complete documents (as a lineage of operations that are applied to move from an empty document to the complete document)
and changes between two document states.

### API

The Cljot public API is contained within the `cljot.delta`, `cljot.compose`, and `cljot.transform` namespaces.

### Constructing a delta

Cljot models documents using the `delta` abstraction.
A `delta` is fundamentally a vector of operations.

#### `delta`
Parameterless delta constructor gives a new, empty delta:
  
```clojure
(delta) ;=> []
```

### Operations on delta

Operations can be conjed to a delta by using the `insert`, `retain`, and `delete` functions and threading.

#### `insert`
Used for appending content to a given delta:

```clojure
(-> (delta) (insert "text"))
;=> [#cljot.delta.impl.ops.Insert{:value "text", :attributes nil}]

(-> (delta) (insert "text" {:bold true}))
;=> [#cljot.delta.impl.ops.Insert{:value "text", :attributes {:bold true}}]
```

#### `retain`
Used for either "moving on" within a given delta (to apply some other operation after a given range)
or applying attributes to the range:

```clojure
(-> (delta) (retain 3) (insert "text")) 
;=> [#cljot.delta.impl.ops.Retain{:value 3, :attributes nil}
;    #cljot.delta.impl.ops.Insert{:value "text", :attributes nil}]

(-> (delta) (retain 1 {:bold true}))
;=> [#cljot.delta.impl.ops.Retain{:value 1, :attributes {:bold true}}]
```

#### `delete`
Used for removing a given range:

```clojure
(-> (delta) (delete 1))
;=> [#cljot.delta.impl.ops.Delete{:value 1}]
```

### Operational transformation

Cljot includes two central functions for supporting operational transformation: delta composition and transformation.

#### `compose`
Composes two deltas:

```clojure
(compose
  (-> (delta)
      (insert "a"))
  (-> (delta)
      (insert "b")))

;=> [#cljot.delta.impl.ops.Insert{:value "ba", :attributes nil}]
```

#### `transform`
Transforms a given delta by another.

The order of the two concurrent deltas can be decided:

##### `transform-prioritising-a`
The first delta takes place first.

```clojure
(transform-prioritising-a
  (-> (delta)
      (insert "a"))
  (-> (delta)
      (insert "b")))

;=> [#cljot.delta.impl.ops.Retain{:value 1, :attributes nil}
;    #cljot.delta.impl.ops.Insert{:value "a", :attributes nil}]
```

##### `transform-prioritising-b`
The second delta takes place first.

```clojure
(transform-prioritising-b
  (-> (delta)
      (insert "a"))
  (-> (delta)
      (insert "b")))

;=> [#cljot.delta.impl.ops.Insert{:value "a", :attributes nil}]
```

The `transform` function is an alias for `transform-prioritising-b`.

## License

Copyright © 2018 Petri Myllys

Distributed under the Eclipse Public License 2.0.
