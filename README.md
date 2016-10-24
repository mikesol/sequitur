# sequitur
A simple event sequencer for Node.js.

Sequitur is useful for controlling audio, lighting, and anything else that needs to be played, paused, stopped and/or resumed. It is especially well suited for use with RxJS.

Sequitur is build using flow and transpiled to ES5 from ES6 using babel.

## Hello world

```javascript
var Sequitur = require("sequitur");
var ril = Sequitur.rerouteIfLate;
var Rx = require('rx'),
  Observable = Rx.Observable,
  EventEmitter = require('events').EventEmitter;
var e = new EventEmitter();
var sum = 0;
var seq = new Sequitur(e).at('0s', 'foo', 1)
  .at('2.1s', 'foo', 1)
  .at('3.1s', 'foo', 1, 'aGroupName')
  .at('4.5s', 'foo', 1, 'aGroupName')
  .at('5.8s', 'foo', 1);
var subscription = Observable.fromEvent(e, 'foo')
  .subscribe((x) => sum += x);
seq.play();
// wait 2 seconds
// sum will equal 2
seq.stop(); // resets
seq.play();
// wait 2 seconds
// sum will equal 2
seq.pause(); // paused
seq.play(); // resume
// wait 2 seconds
seq.softpause(); // softpause allows pending members of a group to be emitted
// wait, sum will be 4 because of softpause, but last event is not emitted
seq.stop();
```

## API
### `constructor`
```javascript
var mySequitur = new Sequitur(e: EventEmitter, xtra ?: any)
```
Create a `Sequitur` object that emits events from `e`, passing `xtra` to a calling function at the time of emission.

### `at`
```javascript
mySequitur.at(t: string,
    key: string | (x: number, y: any) => void,
    val: mixed | (x: number, y: any) => void,
    group ? : string): Sequitur
```
Schedules an event at time `t`. Time `t` can be expressed in seconds, milliseconds, microseconds or nanoseconds. It uses the same format of timing as [nanotimer][1].

Key `key` and value `value` can be either literals or functions.
- If they are literals, the are emitted in the traditional sense from the event passed into the `Sequitur` constructor, ie: `e.emit(key, value)`.
- If they are functions, then the return value of the functions are used for the key and value to the emitter. The function takes two parameters: the first is the error (meaning how late the emission happens compared to the request) and the second are the extra arguments passed to the `Sequitur` constructor.  See [Functions passed as arguments to `at`](#functions-passed-as-arguments-to-at).

### `play`
```javascript
mySequitur.play(): void
```
Plays a seequence, picking up from the point at which it was paused or to which it was seeked, otherwise starts from `0s`.

### `pause`
```javascript
mySequitur.pause(): void
```
Pauses a sequence.  `pause` differs from `stop` in that `pause` does not set the timeline to `0s`.

### `stop`
```javascript
mySequitur.stop(): void
```
Stops a sequence.  `stop` differs from `pause` in that `stop` sets the timeline to `0s`.

### `seek`
```javascript
mySequitur.seek(t: string): void
```
Fastforwards a sequence to time `t`. This does not interrupt playing if a sequence is currently playing.

### `softpause`
```javascript
mySequitur.softpause(): void
```
Like pause, except events in the event's group will execute after the pause.
For example, in:

```javascript
var e = new EventEmitter();
var seq = new Sequitur(e).at('0s', 'foo', 1)
  .at('2.1s', 'foo', 1)
  .at('3.1s', 'foo', 1, 'aGroupName')
  .at('4.5s', 'foo', 1, 'aGroupName')
  .at('5.8s', 'foo', 1);
```

If `softpause` is called between 3.1 and 4.5 seconds, the event at `4.5s` will be emitted.

### `print`
```javascript
mySequitur.print(): void
```

Prints information about the sequence to the console.

### `rerouteIfLate`
```javascript
Sequitur.rerouteIfLate(ifOnTime: any, ifLate: any): (i: number) => any
```

Convenience `static` method for rerouting an event if it is late (meaning if it has positive error for its time value - see ([Functions passed as arguments to `at`](#functions-passed-as-arguments-to-at).  The value at `ifOnTime` is returned if the function is ontime, otherwise `ifLate` is returned.  For example:

```javascript
mySequence.at('10s', Sequitur.rerouteIfLate('eventICareAbout', 'trashbin'), 1)
```

## Functions passed as arguments to `at`
As stated above, a function passed to `at` for the key or value is in the form:

```javascript
function(i: number, x: any)
```

Where `i` is the error in time and `x` is whateve extra value is passed into the `Sequitur` constructor.  The error parameter is perhaps the most important thing to understand about `Sequitur` and is where its power lies.  If a sequence is paused after 3 seconds and then resumed, any event that happens before the 3 second mark is called again with the difference in time as the first argument.  For example, an event scheduled to be emitted at `0.5s` will be called with an error of `2.5` if the pause happened at 3 seconds.  This allows audio files, lighting cues, whatever to pick up where they left off after pausing.  A utility function, `rerouteIfLate`, helps ignore events that happen before a particular timestamp.

[1]: https://github.com/Krb686/nanotimer