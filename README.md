# ZeroMQ.js Next Generation

[![Greenkeeper monitoring](https://img.shields.io/badge/dependencies-monitored-brightgreen.svg)](https://greenkeeper.io/) [![Travis build status](https://img.shields.io/travis/zeromq/zeromq.js.svg)](https://travis-ci.org/zeromq/zeromq.js)

## ⚠️⚠️⚠️ This is work in progress and published only as a beta version ⚠️⚠️⚠️
[ØMQ](http://zeromq.org) bindings for Node.js. The goals of this library are:
* Semantically as similar as possible to the [native](https://github.com/zeromq/libzmq) ØMQ library.
* High performance.
* Use modern JavaScript and Node.js features such as  `async`/`await` and async iterators.
* Fully usable with TypeScript.

# Table of contents

* [Installation](#installation)
* [Examples](#examples)
   * [Push/Pull](#pushpull)
   * [Pub/Sub](#pubsub)
   * [Compatibility layer for version 4/5](#compatibility-layer-for-version-45)
* [Contribution](#contribution)
* [History](#history)


# Installation

Install ZeroMQ.js with prebuilt binaries:

```sh
npm install zeromq@6.0.0-beta.1
```

Requirements for prebuilt binaries:

* Node.js 10+ or Electron 3+ (requires a [N-API](https://nodejs.org/api/n-api.html) version 3+)

The following platforms have a prebuilt binary available:

* Linux on x86-64/armv7/armv8 with glibc
* Linux on x86-64 with musl (e.g. Alpine)
* MacOS on x86-64
* Windows on x86 or x86-64

If a prebuilt binary is not available for your platform, installing will attempt to start a build from source.
If you want to link against a shared ZeroMQ library, you can build and link with the shared library as follows:

```sh
npm install zeromq@6.0.0-beta.1 --zmq-shared
```

If you wish to use any DRAFT sockets then it is also necessary to compile the library from source:

```sh
npm install zeromq@6.0.0-beta.1 --zmq-draft
```

Make sure you have the following installed before attempting to build from source:

* Node.js 10+ or Electron 3+
* A working C/C++ compiler toolchain with make
* Python 2 (2.7 recommended, 3+ does not work)
* ZeroMQ 4.0+ with development headers

# Examples


More examples can be found in the [examples directory](examples).

## Push/Pull

This example demonstrates how a producer pushes information onto a
socket and how a worker pulls information from the socket.

### producer.js

```js
const zmq = require("zeromq")

async function run() {
  const sock = new zmq.Push

  await sock.bind("tcp://127.0.0.1:3000")
  console.log("Producer bound to port 3000")

  while (!sock.closed) {
    await sock.send("some work")
    await new Promise(resolve => setTimeout(resolve, 500))
  }
}

run()
```

### worker.js

```js
const zmq = require("zeromq")

async function run() {
  const sock = new zmq.Pull

  sock.connect("tcp://127.0.0.1:3000")
  console.log("Worker connected to port 3000")

  while (!sock.closed) {
    const [msg] = await sock.receive()
    console.log("work: %s", msg.toString())
  }
}

run()
```


## Pub/Sub

This example demonstrates using `zeromq` in a classic Pub/Sub,
Publisher/Subscriber, application.

### publisher.js

```js
const zmq = require("zeromq")

async function run() {
  const sock = new zmq.Publisher

  await sock.bind("tcp://127.0.0.1:3000")
  console.log("Publisher bound to port 3000")

  while (!sock.closed) {
    console.log("sending a multipart message envelope")
    await sock.send(["kitty cats", "meow!"])
    await new Promise(resolve => setTimeout(resolve, 500))
  }
}

run()
```

### subscriber.js

```js
const zmq = require("zeromq")

async function run() {
  const sock = new zmq.Subscriber

  sock.connect("tcp://127.0.0.1:3000")
  sock.subscribe("kitty cats")
  console.log("Subscriber connected to port 3000")

  while (!sock.closed) {
    const [topic, msg] = await sock.receive()
    console.log("received a message related to:", topic, "containing message:", msg)
  }
}

run()
```


## Compatibility layer for version 4/5

The next generation version of the library features a compatibility layer for ZeroMQ.js versions 4 and 5. This is recommended for users upgrading from previous versions.

Example:

```js
const zmq = require("zeromq/v5-compat")

const pub = zmq.socket("pub")
const sub = zmq.socket("sub")

pub.bind("tcp://*:3456", err => {
  if (err) throw err

  sub.connect("tcp://127.0.0.1:3456")

  pub.send("message")

  sub.on("message", msg => {
    // Handle received message...
  })
})
```


# Contribution


## Dependencies

In order to develop and test the library, you'll need the following:

* A working C/C++ compiler toolchain with make
* Python 2.7
* Node.js 10+
* CMake 2.8+
* curl
* clang-format is strongly recommended


## Defining new options

Socket and context options can be set at runtime, even if they are not implemented by this library. By design, this requires no recompilation if the built version of ZeroMQ has support for them. This allows library users to test and use options that have been introduced in recent versions of ZeroMQ without having to modify this library. Of course we'd love to include support for new options in an idiomatic way.

Options can be set as follows:

```js
const {Dealer} = require("zeromq")

/* This defines an accessor named 'sendHighWaterMark', which corresponds to
   the constant ZMQ_SNDHWM, which is defined as '23' in zmq.h. The option takes
   integers. The accessor name has been converted to idiomatic JavaScript.
   Of course, this particular option already exists in this library. */
class MyDealer extends Dealer {
  get sendHighWaterMark(): number {
    return this.getInt32Option(23)
  }

  set sendHighWaterMark(value: number) {
    this.setInt32Option(23, value)
  }
}

const sock = new MyDealer({sendHighWaterMark: 456})
```

When submitting pull requests for new socket/context options, please consider the following:

* The option is documented in the TypeScript interface.
* The option is only added to relevant socket types, and if the ZMQ_ constant has a prefix indicating which type it applies to, it is stripped from the name as it is exposed in JavaScript.
* The name as exposed in this library is idiomatic for JavaScript, spelling out any abbreviations and using proper `camelCase` naming conventions.
* The option is a value that can be set on a socket, and you don't think it should actually be a method.


## Testing

The test suite can be run with:

```sh
npm install
npm run dev:test
```

Or, if you prefer:

```sh
yarn
yarn run dev:test
```

The test suite will validate and fix the coding style, run all unit tests and verify the validity of the included TypeScript type definitions.

Some tests are not enabled by default:

* API Compatibility tests from ZeroMQ 5.x have been disabled by default. You can include the tests with `INCLUDE_COMPAT_TESTS=1 npm run dev:test`
* Some transports are not reliable on some older versions of ZeroMQ, the relevant tests will be skipped for those versions automatically.


## Publishing

To publish a new version, run:

```sh
npm version <new version>
git push && git push --tags
```

Wait for continuous integration to finish. Prebuilds will be generated for all supported platforms and attached to a Github release. Documentation is automatically generated and committed to `gh-pages`. Finally, a new NPM package version will be automatically released.


# History

Version 6+ is a complete rewrite of previous versions of ZeroMQ.js in order to be more reliable, correct, and usable in modern JavaScript & TypeScript code as first outlined in [this issue](https://github.com/zeromq/zeromq.js/issues/189). Previous versions of ZeroMQ.js were based on `zmq` and a fork that included prebuilt binaries.

See detailed changes in the [CHANGELOG](CHANGELOG).
