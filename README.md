# pull-stream

Minimal Pipeable Pull-stream

In [classic-streams](https://github.com/joyent/node/blob/v0.8/doc/api/stream.markdown),
streams _push_ data to the next stream in the pipeline.
In [new-streams](https://github.com/joyent/node/blob/v0.10/doc/api/stream.markdown),
data is pulled out of the source stream, into the destination.

`pull-stream` is a minimal take on pull streams,
optimized for "object" streams, but still supporting text streams.

## Quick Example

stat some files.

``` js
pull.values(['file1', 'file2', 'file3'])
.pipe(pull.asyncMap(fs.stat))
.pipe(pull.collect(function (err, array) {
  console.log(array)
})
```

The best thing about pull-stream is that it can be completely lazy.
this is perfect for async traversals where you might want to stop early.

stat files recursively

## Examples

What if implementing a stream was this simple:

### Pipeable Streams

`pull.{Source,Through,Sink}` just wrap a function and give it a `.pipe(dest)`!

``` js
var pull = require('pull-stream')

var createSourceStream = pull.Source(function () {
  return function (end, cb) {
    return cb(end, Math.random())
  }
})

var createThroughStream = pull.Through(function (read) {
  return function (end, cb) {
    read(end, cb)
  }
})

var createSinkStream = pull.Sink(function (read) {
  read(null, function next (end, data) {
    if(end) return
    console.log(data)
    read(null, next)
  })
})

createSourceStream().pipe(createThroughStream()).pipe(createSinkStream())
```

### Readable & Reader vs. Readable & Writable

Instead of a readable stream, and a writable stream, there is a `readable` stream,
and a `reader` stream.

See also:
* [Sources](https://github.com/dominictarr/pull-stream/blob/master/docs/sources.md)
* [Throughs](https://github.com/dominictarr/pull-stream/blob/master/docs/throughs.md)
* [Sinks](https://github.com/dominictarr/pull-stream/blob/master/docs/sinks.md)

### Readable

The readable stream is just a `function(end, cb)`,
that may be called many times,
and will (asynchronously) `callback(null, data)` once for each call.

The readable stream eventually `callback(err)` if there was an error, or `callback(true)`
if the stream has no more data.

if the user passes in `end = true`, then stop getting data from wherever.

All [Sources](https://github.com/dominictarr/pull-stream/blob/master/docs/sources.md)
and [Throughs](https://github.com/dominictarr/pull-stream/blob/master/docs/throughs.md)
are readable streams.

``` js
var i = 100
var randomReadable = pull.Source(function () {
  return function (end, cb) {
    if(end) return cb(end)
    //only read 100 times
    if(i-- < 0) return cb(true)
    cb(null, Math.random())
  }
})
```

### Reader (aka, "writable")

A `reader`, is just a function that calls a readable,
until it decideds to stop, or the readable `cb(err || true)`

All [Throughs](https://github.com/dominictarr/pull-stream/blob/master/docs/throughs.md)
and [Sinks](https://github.com/dominictarr/pull-stream/blob/master/docs/sinks.md)
are reader streams.

``` js
var logger = pull.Sink(function (read) {
  read(null, function next(end, data) {
    if(end === true) return
    if(end) throw err

    console.log(data)
    readable(end, next)
  })
})
```

These can be connected together by passing the `readable` to the `reader`

``` js
logger()(randomReadable())
```

Or, if you prefer to read things left-to-right

``` js
randomReadable().pipe(logger())
```

### Through / Duplex

A duplex/through stream is both a `reader` that is also `readable`

A duplex/through stream is just a function that takes a `read` function,
and returns another `read` function.

``` js
var map = pull.Through(function (read, map) {
  //return a readable function!
  return function (end, cb) {
    read(end, function (end, data) {
      cb(end, data != null ? map(data) : null)
    })
  }
})
```

### pipeability

Every pipeline must go from a `source` to a `sink`.
Data will not start moving until the whole thing is connected.

``` js
source.pipe(through).pipe(sink)
```

When setting up pipeability, you must use the right
function, so `pipe` has the right behavior.

Use `Source`, `Through` and `Sink`,
to add pipeability to your pull-streams.

## More Cool Stuff

What if you could do this?

``` js
var trippleThrough = 
  through1().pipe(through2()).pipe(through3())
//THE THREE THROUGHS BECOME ONE

source().pipe(trippleThrough).pipe(sink())

//and then pipe it later!
```

## Design Goals & Rationale


There is a deeper,
[platonic abstraction](http://en.wikipedia.org/wiki/Platonic_idealism),
where a streams is just an array in time, instead of in space.
And all the various streaming "abstractions" are just crude implementations
of this abstract idea.

[classic-streams](https://github.com/joyent/node/blob/v0.8.16/doc/api/stream.markdown),
[new-streams](https://github.com/joyent/node/blob/v0.10/doc/api/stream.markdown),
[reducers](https://github.com/Gozala/reducers)

The objective here is to find a simple realization of the best features of the above.

### Type Agnostic

A stream abstraction should be able to handle both streams of text and streams
of objects.

### A pipeline is also a stream.

This should work: `a.pipe(x.pipe(y).pipe(z)).pipe(b)`
this makes it possible to write a custom stream simply by
combining a few available streams.

### Propagate End/Error conditions.
 
If a stream ends in an unexpected way (error),
then other streams in the pipeline should be notified.
(this is a problem in node streams - when an error occurs,
the stream is disconnected, and the user must handle that specially)

Also, the stream should be able to be ended from either end.

### Transparent Backpressure & Lazyness

Very simple transform streams must be able to transfer back pressure
instantly.

This is a problem in node streams, pause is only transfered on write, so
on a long chain (`a.pipe(b).pipe(c)`), if `c` pauses, `b` will have to write to it
to pause, and then `a` will have to write to `b` to pause.
If `b` only transforms 'a`s output, then `a` will have to write to `b` twice to
find out that `c` is paused.

[reducers](https://github.com/Gozala/reducers) reducers has an interesting method,
where synchronous tranformations propagate back pressure instantly!

This means you can have two "smart" streams doing io at the ends, and lots of dumb 
streams in the middle, and back pressure will work perfectly, as if the dump streams
and not there.

This makes lazyness work right.

### 



## License

MIT
