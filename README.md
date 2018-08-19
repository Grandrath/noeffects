# noeffects

Decouple side effects from business logic using generator functions.

Write your business logic inside generator functions and `yield`
"effect descriptors". An effect descriptor is a plain array, that does
not actually perform a side effect but describes what side effect
should be performed. Execute the generator functions inside a call to
`run` together with a `context` map. The `run` function translates the
effect descriptors to values inside the `context` map. When a value
happens to be a function `run` calls it with any arguments provided in
the effect descriptor. The result of this function call is passed back
to the generator function.

`run` will automatically `await` promises, can `promisify` callback
style functions and can run multiple side effects in parallel.

`noeffects` also provides a test harness that allows you to run you
code using fixed results for expected effect descriptors. This enables
you to unit test your code without performing any side effects
whatsoever.

## Installation

```
npm install -S noeffects
```

### Requirements

NodeJS >= v7.6.0

## Usage

### Effect descriptors

An effect descriptor is a plain array with the following structure:

```
[<special action>, <context path>, ...<arguments>]
```

* `special action` is optional and can be one of the symbols
  `noeffects.PARALLEL` or `noeffects.CALLBACK_STYLE`. See below for
  details.
* `context path` is an array of strings that describes the path within
  the context map to the desired value, e.g. `["path", "to", "value"]`
  would point to `context.path.to.value`. A path of length one can be
  written as a string, e.g. `"foo"` instead of `["foo"]`.
* `arguments` are zero or more values that are passed to the function
  found at `context path`. If the value at `context path` is not a
  function `arguments` are ignored.

### Reading values

**Reading a nested value**

```javascript
const {run} = require("noeffects");

const myProgram = function* () {
  return yield [["path", "to", "value"]];
};

const context = {
  path: {
    to: {
      value: "foo"
    }
  }
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// => "foo"
```

**Reading a top-level value**

```javascript
const {run} = require("noeffects");

const myProgram = function* () {
  // paths of length 1 can be written as strings
  // (i.e. ["value"] instead of [["value"]])
  return yield ["value"];
};

const context = {
  value: "foo"
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// => "foo"
```

### Invoking functions

When the value at `context path` happens to be a function it is
invoked with all remaining array items as arguments. When this
function returns a promise its eventual value is returned from
`yield`.

```javascript
const {run} = require("noeffects");

const myProgram = function* () {
  return yield [["path", "to", "fn"], 1, 2, 3];
};

const context = {
  path: {
    to: {
      fn: (a, b, c) => Promise.resolve(a + b + c)
    }
  }
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// => 6
```

### Invoking callback style functions

In order to use an asynchronus callback style function you need to use
the `noeffects.CALLBACK_STYLE` symbol. Otherwise `run` will assume
that the invoked function returns a promise.

```javascript
const {run, CALLBACK_STYLE} = require("noeffects");

const myProgram = function* () {
  const home = yield [["env", "HOME"]];
  return yield [CALLBACK_STYLE, ["fs", "readdir"], home];
};

const context = {
  env: process.env,
  fs: require("fs")
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// => <a listing of your $HOME directory>
```

### Performing multiple asynchronous side effects in parallel

When the effect descriptor's first item is the symbol
`noeffects.PARALLEL` the remaining items are interpreted as effect
descriptors and are preformed in parallel. `yield` returns an array
containing the eventual values similar to `Promise.all`.

```javascript
const {run, PARALLEL} = require("noeffects");

const myProgram = function* () {
  return yield [PARALLEL,
                ["fetch", "http://google.com"]
                ["fetch", "http://bing.com"]
               ];
};

const timeout = millis => new Promise(resolve => setTimeout(resolve, millis));

const context = {
  fetch: async (url) => {
    await timeout(2000);
    return `Response from ${url}`;
  }
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// after 2 seconds (instead of 4)
// => [
//      "Response from http://google.com",
//      "Response from http://bing.com"
//    ]
```

### Using functions to create effect descriptors

Since effect descriptors are plain arrays, it is easy to write simple
helper functions to construct them. This can make your code easier to
read.


```javascript
const {run, CALLBACK_STYLE} = require("noeffects");

const env = name => [["env", name]];
const readDir = path => [CALLBACK_STYLE, ["fs", "readdir"], path];

const myProgram = function* () {
  const home = yield env("HOME");
  return yield readDir(home);
};

const context = {
  env: process.env,
  fs: require("fs")
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// => <a listing of your $HOME directory>
```

### Nesting generator functions

Generator functions can be nested.

```javascript
const {run} = require("noeffects");

const isDateInFuture = function* (date) {
  const now = yield [["Date", "now"]];
  return date.getTime() > now;
};

const myProgram = function* () {
  const date = new Date("1990-01-01T00:00:00Z"));
  return yield isDateInFuture(date);
};

const context = {
  Date
};

run(context, myProgram())
  .then(result => console.log(result))
  .catch(error => console.error(error));

// => false
```

### Preventing unhandled promise rejections

As in the examples above you should always add a call to `catch` to
the promise returned by `run` to avoid unhandled promise rejections.

## Testing generator functions

Being able to test business logic that needs to interact with the
outside world is the primary reason why you want to decouple it from
side effects. Noeffects provides a test harness to do exactly that.

You provide a mapping from expected effect descriptors to the values
that these should `yield`.

```javascript
// code under test
const myProgram = function* () {
  const increment = yield [["config", "increment"]];
  const value = yield [["someDatabase", "getValue"]];
  yield [["someDatabase", "setValue"], value + increment];
};

// test
const {testRun} = require("noeffects/test-harness");
const test = require("tape");

test("myProgram", t => {
  let value = 2;

  const expectedEffects = [
    {effect: [["config", "increment"]],
     yields: 5},

    {effect: [["someDatabase", "getValue"]],
     yields: () => value},

    {effect: [["someDatabase", "setValue"]],
     yields: v => { value = v; }}
  ];

  testRun(expectedEffects, myProgram());
  t.equal(value, 7, "Value was 2 and has been incremented by 5");

  t.end();
});
```

## License

[MIT](./LICENSE)
