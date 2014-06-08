strong-log-transformer
======================

A stream filter for performing common log stream transformations like
timestamping and joining multi-line messages.

**This is not a logger!** But it may be useful for rolling your own logger.

## Usage

Install strong-log-transformer and add it to your dependencies list.
```sh
npm install --save strong-log-transformer
```

### Line Merging

In order to keep things flowing when line merging is enabled (disabled by
default) there is a sliding 10ms timeout for flushing the buffer. This means
that whitespace leading lines are only considered part of the previous line if
they arrive within 10ms of the previous line, which should be reasonable
considering the lines were likely written in the same `write()`.

### Example

Here's an example using the transformer to annotate log messages from cluster
workers.

```js
var cluster = require('cluster');

if (cluster.isMaster) {
  // Make sure workers get their own stdout/stderr streams
  cluster.setupMaster({silent: true});

  // require log transformer module
  var transformer = require('strong-log-transformer');

  // Following the 12-factor app model, we pipe to stdout, but we could easily
  // pipe to any other stream(s), such as a FileStream for a log file.

  // stdout is plain line-oriented logs, but we want to add timestamps
  var info = transformer({ timeStamp: true,
                           destination: process.stdout });
  // stderr will only be used for strack traces on crash, which are multi-line
  var error = transformer({ timeStamp: true,
                            destination: process.stdout,
                            mergeMultiline: true });

  // Each worker's stdout/stderr gets piped into our info and erro transformers
  cluster.on('fork', function(worker) {
    console.error('connecting worker');
    worker.process.stdout.pipe(info);
    worker.process.stderr.pipe(error);
  });

  //... cluster fork logic goes here ...
  cluster.fork();

} else {
  //... worker code here ...

  console.log('new worker, this line will be timestamped!');
  throw new Error('This will generate a multi-line message!');
}

```

When we run the example code as `example.js` we get:
```sh
$ node example.js
connecting worker
2014-06-08T18:54:00.920Z new worker, this line will be timestamped!
2014-06-08T18:54:00.926Z /Users/ryan/work/strong-log-transformer/e.js:33\n    throw new Error('This will generate a multi-line message!');\n          ^
2014-06-08T18:54:00.926Z Error: This will generate a multi-line message!\n    at null._onTimeout (/Users/ryan/work/strong-log-transformer/e.js:33:11)\n    at Timer.listOnTimeout [as ontimeout] (timers.js:110:15)
```
