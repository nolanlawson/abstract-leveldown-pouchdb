# DEPRECATED - see https://github.com/Level/memdown/pull/56#issuecomment-246451933

# abstract-leveldown-pouchdb [![Build Status](https://secure.travis-ci.org/nolanlawson/abstract-leveldown-pouchdb.png)](http://travis-ci.org/nolanlawson/abstract-leveldown-pouchdb)

Fork of [abstract-leveldown](https://github.com/nolanlawson/abstract-leveldown) because stuff broke in v2.5.0 and [@nolanlawson](https://github.com/nolanlawson) was too lazy to fix it.

Consider it a running fork of 2.4.1. See [this comment](https://github.com/Level/abstract-leveldown/pull/96#issuecomment-246122818) for backstory.

### Changelog

- **2.4.3**: avoid allocating extra NotFoundError ([5d7a780](https://github.com/nolanlawson/abstract-leveldown-pouchdb/commit/5d7a78021c4a4a0123f125f0bc3a57b4acbdc606))
- **2.4.2**: fork


<img alt="LevelDB Logo" height="100" src="http://leveldb.org/img/logo.svg">

An abstract prototype matching the **[LevelDOWN](https://github.com/level/leveldown/)** API. Useful for extending **[LevelUP](https://github.com/level/levelup)** functionality by providing a replacement to LevelDOWN.

As of version 0.7, LevelUP allows you to pass a `'db'` option when you create a new instance. This will override the default LevelDOWN store with a LevelDOWN API compatible object.

**Abstract LevelDOWN** provides a simple, operational *noop* base prototype that's ready for extending. By default, all operations have sensible "noops" (operations that essentially do nothing). For example, simple operations such as `.open(callback)` and `.close(callback)` will simply invoke the callback (on a *next tick*). More complex operations  perform sensible actions, for example: `.get(key, callback)` will always return a `'NotFound'` `Error` on the callback.

You add functionality by implementing the underscore versions of the operations. For example, to implement a `put()` operation you add a `_put()` method to your object. Each of these underscore methods override the default *noop* operations and are always provided with **consistent arguments**, regardless of what is passed in by the client.

Additionally, all methods provide argument checking and sensible defaults for optional arguments. All bad-argument errors are compatible with LevelDOWN (they pass the LevelDOWN method arguments tests). For example, if you call `.open()` without a callback argument you'll get an `Error('open() requires a callback argument')`. Where optional arguments are involved, your underscore methods will receive sensible defaults. A `.get(key, callback)` will pass through to a `._get(key, options, callback)` where the `options` argument is an empty object.

## Example

A simplistic in-memory LevelDOWN replacement

```js
var util = require('util')
  , AbstractLevelDOWN = require('./').AbstractLevelDOWN

// constructor, passes through the 'location' argument to the AbstractLevelDOWN constructor
function FakeLevelDOWN (location) {
  AbstractLevelDOWN.call(this, location)
}

// our new prototype inherits from AbstractLevelDOWN
util.inherits(FakeLevelDOWN, AbstractLevelDOWN)

// implement some methods

FakeLevelDOWN.prototype._open = function (options, callback) {
  // initialise a memory storage object
  this._store = {}
  // optional use of nextTick to be a nice async citizen
  process.nextTick(function () { callback(null, this) }.bind(this))
}

FakeLevelDOWN.prototype._put = function (key, value, options, callback) {
  key = '_' + key // safety, to avoid key='__proto__'-type skullduggery 
  this._store[key] = value
  process.nextTick(callback)
}

FakeLevelDOWN.prototype._get = function (key, options, callback) {
  var value = this._store['_' + key]
  if (value === undefined) {
    // 'NotFound' error, consistent with LevelDOWN API
    return process.nextTick(function () { callback(new Error('NotFound')) })
  }
  process.nextTick(function () {
    callback(null, value)
  })
}

FakeLevelDOWN.prototype._del = function (key, options, callback) {
  delete this._store['_' + key]
  process.nextTick(callback)
}

// now use it in LevelUP

var levelup = require('levelup')

var db = levelup('/who/cares/', {
  // the 'db' option replaces LevelDOWN
  db: function (location) { return new FakeLevelDOWN(location) }
})

db.put('foo', 'bar', function (err) {
  if (err) throw err
  db.get('foo', function (err, value) {
    if (err) throw err
    console.log('Got foo =', value)
  })
})
```

See [MemDOWN](https://github.com/rvagg/memdown/) if you are looking for a complete in-memory replacement for LevelDOWN.

## Extensible API

Remember that each of these methods, if you implement them, will receive exactly the number and order of arguments described. Optional arguments will be converted to sensible defaults.

### AbstractLevelDOWN(location)
### AbstractLevelDOWN#_open(options, callback)
### AbstractLevelDOWN#_close(callback)
### AbstractLevelDOWN#_get(key, options, callback)
### AbstractLevelDOWN#_put(key, value, options, callback)
### AbstractLevelDOWN#_del(key, options, callback)
### AbstractLevelDOWN#_batch(array, options, callback)

If `batch()` is called without argument or with only an options object then it should return a `Batch` object with chainable methods. Otherwise it will invoke a classic batch operation.

### AbstractLevelDOWN#_chainedBatch()

By default an `batch()` operation without argument returns a blank `AbstractChainedBatch` object. The prototype is available on the main exports for you to extend. If you want to implement chainable batch operations then you should extend the `AbstractChaindBatch` and return your object in the `_chainedBatch()` method.

### AbstractLevelDOWN#_approximateSize(start, end, callback)
### AbstractLevelDOWN#_iterator(options)

By default an `iterator()` operation returns a blank `AbstractIterator` object. The prototype is available on the main exports for you to extend. If you want to implement iterator operations then you should extend the `AbstractIterator` and return your object in the `_iterator(options)` method.

`AbstractIterator` implements the basic state management found in LevelDOWN. It keeps track of when a `next()` is in progress and when an `end()` has been called so it doesn't allow concurrent `next()` calls, it does it allow `end()` while a `next()` is in progress and it doesn't allow either `next()` or `end()` after `end()` has been called.

### AbstractIterator(db)

Provided with the current instance of `AbstractLevelDOWN` by default.

### AbstractIterator#_next(callback)
### AbstractIterator#_end(callback)

### AbstractChainedBatch
Provided with the current instance of `AbstractLevelDOWN` by default.

### AbstractChainedBatch#_put(key, value)
### AbstractChainedBatch#_del(key)
### AbstractChainedBatch#_clear()
### AbstractChainedBatch#_write(options, callback)

### isLevelDown(db)

Returns `true` if `db` has the same public api as `AbstractLevelDOWN`, otherwise `false`. This is a utility function and it's not part of the extensible api.

<a name="contributing"></a>
Contributing
------------

AbstractLevelDOWN is an **OPEN Open Source Project**. This means that:

> Individuals making significant and valuable contributions are given commit-access to the project to contribute as they see fit. This project is more like an open wiki than a standard guarded open source project.

See the [contribution guide](https://github.com/Level/community/blob/master/CONTRIBUTING.md) for more details.

<a name="license"></a>
License &amp; Copyright
-------------------

Copyright &copy; 2012-2015 **AbstractLevelDOWN** [contributors](https://github.com/level/community#contributors).

**AbstractLevelDOWN** is licensed under the MIT license. All rights not explicitly granted in the MIT license are reserved. See the included `LICENSE.md` file for more details.
