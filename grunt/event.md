# Grunt源码解析之'/grunt/event.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/event.js

该模块主要涉及`grunt`中所有事件类操作。

```javascript
// External lib.
var EventEmitter2 = require('eventemitter2').EventEmitter2;

// Awesome.
module.exports = new EventEmitter2({wildcard: true});
```
从源码中可以得知，该模块只是简单地调用[eventemitter2][]外部库的`EventEmitter2`构造函数，并将EventEmitter的实例作为本模块的输出。


[eventemitter2]: https://github.com/asyncly/EventEmitter2 "EventEmitter2"


