# Node Schedule with Timezone Support

[![NPM version](http://img.shields.io/npm/v/node-schedule-tz.svg)](https://www.npmjs.com/package/node-schedule-tz)
[![Downloads](https://img.shields.io/npm/dm/node-schedule-tz.svg)](https://www.npmjs.com/package/node-schedule-tz)
[![Build Status](https://travis-ci.org/node-schedule-tz/node-schedule-tz.svg?branch=master)](https://travis-ci.org/node-schedule/node-schedule-tz)
[![Join the chat at https://gitter.im/node-schedule-tz/node-schedule-tz](https://img.shields.io/badge/gitter-chat-green.svg)](https://gitter.im/node-schedule-tz/node-schedule-tz?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![NPM](https://nodei.co/npm/node-schedule-tz.png?downloads=true)](https://nodei.co/npm/node-schedule-tz/)

Node Schedule is a flexible cron-like and not-cron-like job scheduler with timezone support for Node.js.
It allows you to schedule jobs (arbitrary functions) for execution at
specific dates, with optional recurrence rules. It only uses a single timer
at any given time (rather than reevaluating upcoming jobs every second/minute).

## Usage

### Installation

You can install using [npm](https://www.npmjs.com/package/node-schedule-tz).

```
npm install node-schedule-tz
```

### Overview

Node Schedule is for time-based scheduling, not interval-based scheduling.
While you can easily bend it to your will, if you only want to do something like
"run this function every 5 minutes", you'll find `setInterval` much easier to use,
and far more appropriate. But if you want to, say, "run this function at the :20
and :50 of every hour on the third Tuesday of every month," you'll find that
Node Schedule suits your needs better. Additionally, Node Schedule has Windows
support unlike true cron since the node runtime is now fully supported.

Note that Node Schedule is designed for in-process scheduling, i.e. scheduled jobs
will only fire as long as your script is running, and the schedule will disappear
when execution completes. If you need to schedule jobs that will persist even when
your script *isn't* running, consider using actual [cron].

### Jobs and Scheduling

Every scheduled job in Node Schedule is represented by a `Job` object. You can
create jobs manually, then execute the `schedule()` method to apply a schedule,
or use the convenience function `scheduleJob()` as demonstrated below.

`Job` objects are `EventEmitter`'s, and emit a `run` event after each execution.
They also emit a `scheduled` event each time they're scheduled to run, and a
`canceled` event when an invocation is canceled before it's executed (both events
receive a JavaScript date object as a parameter). Note that jobs are scheduled the
first time immediately, so if you create a job using the `scheduleJob()`
convenience method, you'll miss the first `scheduled` event. Also note that
`canceled` is the single-L American spelling.

### Cron-style Scheduling

The cron format consists of:
```
*    *    *    *    *    *
┬    ┬    ┬    ┬    ┬    ┬
│    │    │    │    │    |
│    │    │    │    │    └ day of week (0 - 7) (0 or 7 is Sun)
│    │    │    │    └───── month (1 - 12)
│    │    │    └────────── day of month (1 - 31)
│    │    └─────────────── hour (0 - 23)
│    └──────────────────── minute (0 - 59)
└───────────────────────── second (0 - 59, OPTIONAL)
```

Examples with the cron format:

```js
var schedule = require('node-schedule-tz');

var j = schedule.scheduleJob('42 * * * *', function(){
  console.log('The answer to life, the universe, and everything!');
});
```

Execute a cron job when the minute is 42 (e.g. 19:42, 20:42, etc.).

And:

```js
var j = schedule.scheduleJob('0 17 ? * 0,4-6', function(){
  console.log('Today is recognized by Rebecca Black!');
});
```

Execute a cron job every 5 Minutes = */5 * * * *

You can specify a timezone as well:

If a timezone is specified, a job name must be specified as well as the first parameter.

```js
var j = schedule.scheduleJob('job name', '0 14 * * *', 'Asia/Shanghai', function(){
  console.log('Will execute on 14:00:00 GMT+8:00 (CST) everyday');
});
```

#### Unsupported Cron Features

Currently, `W` (nearest weekday), `L` (last day of month/week), and `#` (nth weekday
of the month) are not supported. Most other features supported by popular cron
implementations should work just fine.

[cron-parser] is used to parse crontab instructions.

### Date-based Scheduling

Say you very specifically want a function to execute at 5:30am on December 21, 2012.
Remember - in JavaScript - 0 - January, 11 - December.

```js
var schedule = require('node-schedule-tz');
var date = new Date(2012, 11, 21, 5, 30, 0);

var j = schedule.scheduleJob(date, function(){
  console.log('The world is going to end today.');
});
```

You can invalidate the job with the `cancel()` method:

```js
j.cancel();
```

To use current data in the future you can use binding:

```js
var schedule = require('node-schedule-tz');
var date = new Date(2012, 11, 21, 5, 30, 0);
var x = 'Tada!';
var j = schedule.scheduleJob(date, function(y){
  console.log(y);
}.bind(null,x));
x = 'Changing Data';
```
This will log 'Tada!' when the scheduled Job runs, rather than 'Changing Data',
which x changes to immediately after scheduling.

### Recurrence Rule Scheduling

You can build recurrence rules to specify when a job should recur. For instance,
consider this rule, which executes the function every hour at 42 minutes after the hour:

```js
var schedule = require('node-schedule-tz');

var rule = new schedule.RecurrenceRule();
rule.minute = 42;

var j = schedule.scheduleJob(rule, function(){
  console.log('The answer to life, the universe, and everything!');
});
```

You can also use arrays to specify a list of acceptable values, and the `Range`
object to specify a range of start and end values, with an optional step parameter.
For instance, this will print a message on Thursday, Friday, Saturday, and Sunday at 5pm:

```js
var rule = new schedule.RecurrenceRule();
rule.dayOfWeek = [0, new schedule.Range(4, 6)];
rule.hour = 17;
rule.minute = 0;
rule.tz = 'Asia/Shanghai'; // You can specify a timezone!

var j = schedule.scheduleJob(rule, function(){
  console.log('Today is recognized by Rebecca Black!');
});
```

#### RecurrenceRule properties

- `second`
- `minute`
- `hour`
- `date`
- `month`
- `year`
- `dayOfWeek`
- `tz`

> **Note**: It's worth noting that the default value of a component of a recurrence rule is
> `null` (except for second, which is 0 for familiarity with cron). *If we did not
> explicitly set `minute` to 0 above, the message would have instead been logged at
> 5:00pm, 5:01pm, 5:02pm, ..., 5:59pm.* Probably not what you want.

#### Object Literal Syntax

To make things a little easier, an object literal syntax is also supported, like
in this example which will log a message every Sunday at 2:30pm:

```js
var j = schedule.scheduleJob({hour: 14, minute: 30, dayOfWeek: 0}, function(){
  console.log('Time for tea!');
});
```

#### Set StartTime and EndTime

It will run after 5 seconds and stop after 10 seconds in this example.
The ruledat supports the above.

```js
let startTime = new Date(Date.now() + 5000);
let endTime = new Date(startTime.getTime() + 5000);
var j = schedule.scheduleJob({ start: startTime, end: endTime, rule: '*/1 * * * * *' }, function(){
  console.log('Time for tea!');
});
```


## Contributing

This module was originally developed by [Matt Patenaude], and is now maintained by
[Tejas Manohar] and [other wonderful contributors].

We'd love to get your contributions. Individuals making significant and valuable
contributions are given commit-access to the project to contribute as they see fit.

Before jumping in, check out our [Contributing] page guide!

## Copyright and license

Copyright 2015 Matt Patenaude.

Licensed under the **[MIT License] [license]**.


[cron]: http://unixhelp.ed.ac.uk/CGI/man-cgi?crontab+5
[Contributing]: https://github.com/node-schedule/node-schedule/blob/master/CONTRIBUTING.md
[Matt Patenaude]: https://github.com/mattpat
[Tejas Manohar]: http://tejas.io
[license]: https://github.com/node-schedule/node-schedule/blob/master/LICENSE
[Tejas Manohar]: https://github.com/tejasmanohar
[other wonderful contributors]: https://github.com/node-schedule/node-schedule/graphs/contributors
[cron-parser]: https://github.com/harrisiirak/cron-parser
