# Grunt源码解析之'/grunt/fail.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/fail.js

该模块主要用来定义一些在发生错误时需要调用的API。

## grunt内部调用模块

这里调用了[grunt.js][]模块，这也是Grunt最核心的模块（最终暴露给外界的模块）。

```javascript
var grunt = require('../grunt');
```
本模块没有调用或使用其他的Node.js的内部模块以及外部模块。

## 模块内部代码解析

### fail变量

定义`fail`变量，默认值为空数组，作为本模块返回的对象。

```javascript
// The module to be exported.
var fail = module.exports = {};
```
### fail.code

定义`fail`中的`code`属性，值为一个key-value对象，对应各种错误等级以及对应的错误码。

```javascript
// Error codes.
fail.code = {
  FATAL_ERROR: 1,
  MISSING_GRUNTFILE: 2,
  TASK_FAILURE: 3,
  TEMPLATE_ERROR: 4,
  INVALID_AUTOCOMPLETE: 5,
  WARNING: 6,
};
```
### writeln方法

定义了功能方法`writeln`，对错误信息进行输出。

方法接受两个参数，错误对象`e`以及输出模式`mode`。

```javascript
// DRY it up!
function writeln(e, mode) {
```
首先，打开`grunt.log.muted`的开关，确认信息可以通过`grunt.log`进行输出。

```javascript
  grunt.log.muted = false;
```
使用[String][]将错误信息进行字符串转换，并复制给变量`msg`。

这里之所以使用[String][]进行字符串转换，对所有的错误信息转换提供了统一的途径，避免个别的`e`对象自定义了与本模块处理预期不符的`toString`方法。

```javascript
  var msg = String(e.message || e);
```
如果`e`中并没有定义`message`属性，则将`e`直接字符串转换，效果如下：

```javascript
> var e = new Error()
> String(e)
'Error'
```
当grunt并没有指定`no-color`选项时，对`msg`变量进行赋值，值`\x07`是一个ASCII码，用来表示终端的Beep声。

```javascript
  if (!grunt.option('no-color')) { msg += '\x07'; } // Beep!
```

如果`mode`值为'warn'则用黄色显示相关警告信息，否则的话，用红色显示错误信息，并最终通过`grunt.log.writeln`将信息输出。

```javascript
  if (mode === 'warn') {
    msg = 'Warning: ' + msg + ' ';
    msg += (grunt.option('force') ? 'Used --force, continuing.'.underline : 'Use --force to continue.');
    msg = msg.yellow;
  } else {
    msg = ('Fatal error: ' + msg).red;
  }
  grunt.log.writeln(msg);
}
```
### dump方法

如果在调用grunt时，使用了`--stack`，则将所有的错误栈信息通过`console.log`进行输出。

```javascript
// If --stack is enabled, log the appropriate error stack (if it exists).
function dumpStack(e) {
  if (grunt.option('stack')) {
```
如果`e`有不为空的`origError`和`origError.stack`属性，则将`e.origError.stack`进行输出，否的的话输出`e.stack`。

```javascript
    if (e.origError && e.origError.stack) {
      console.log(e.origError.stack);
    } else if (e.stack) {
      console.log(e.stack);
    }
  }
}
```

### fail.fatal

对致命错误信息进行处理。

通过`writeln`输出致命错误信息，调用`dumpStack`方法打出错误栈，并且通过`grunt.util.exit`方法退出任务。

```javascript
// A fatal error occurred. Abort immediately.
fail.fatal = function(e, errcode) {
  writeln(e, 'fatal');
  dumpStack(e);
  grunt.util.exit(typeof errcode === 'number' ? errcode : fail.code.FATAL_ERROR);
};
```
### fail.errorcount

记录错误发生数量。

```javascript
// Keep track of error and warning counts.
fail.errorcount = 0;
```
### fail.warncount

记录警告发生数量。

```javascript
fail.warncount = 0;
```

### fail.warn

对警告信息进行处理。

```javascript
// A warning occurred. Abort immediately unless -f or --force was used.
fail.warn = function(e, errcode) {
```
将`fail.wancount`值加1，并且调用`writeln`方法将当前的警告信息进行输出。

```javascript
  var message = typeof e === 'string' ? e : e.message;
  fail.warncount++;
  writeln(message, 'warn');
```
如果调用grunt时，没有使用`-f`或者`--force`，则调用`dumpStack`方法打出错误栈，并且通过`grunt.util.exit`方法退出任务。

```javascript
  // If -f or --force aren't used, stop script processing.
  if (!grunt.option('force')) {
    dumpStack(e);
    grunt.log.writeln().fail('Aborted due to warnings.');
    grunt.util.exit(typeof errcode === 'number' ? errcode : fail.code.WARNING);
  }
};
```

### fail.report

根据`fail.warncount`的值调用`grunt.log.writeln().fail`或者`grunt.log.writeln().success`来输出相应的信息。

这个方法将在每个任务执行结束后被调用。

```javascript
// This gets called at the very end.
fail.report = function() {
  if (fail.warncount > 0) {
    grunt.log.writeln().fail('Done, but with warnings.');
  } else {
    grunt.log.writeln().success('Done, without errors.');
  }
};
```

[String]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String "String"

[grunt.js]: https://github.com/gruntjs/grunt/blob/master/lib/grunt.js "grunt.js"

