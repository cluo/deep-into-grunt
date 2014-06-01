# Grunt源码解析之'/grunt/help.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/help.js

本模块用来输出与`-help`相关的信息到终端。

## grunt内部调用模块

这里调用了[grunt.js][]模块，这也是Grunt最核心的模块（最终暴露给外界的模块）。

```javascript
var grunt = require('../grunt');
```
## node.js内部调用模块

```javascript
// Nodejs libs.
var path = require('path');
```
[path][]模块包含了一系列的处理和转换文件路径的工具方法，实际上几乎所有的方法都在是做字符串转换，而不检查路径是否真实有效。

## 模块内部代码解析

### col1len变量

该变量初始值为0，用来确定信息按表格式输出时的第一列宽度。

```javascript
// Set column widths.
var col1len = 0;
```

### initCol1方法

通过[Math.max][]方法得到参数`str`的字符长度与`col1len`变量值中的较大数值并赋值给`col1len`变量。`col1len`变量用来确保`table`方法以表格形式输出内容时，格式第一列的宽度合适。

```javascript
exports.initCol1 = function(str) {
  col1len = Math.max(col1len, str.length);
};
```
### initWidths方法

通过`col1len`变量进行计算来决定一个4列表格各列宽度的合适值，并将宽度值以数组形式赋值给`widths`属性。

```javascript
exports.initWidths = function() {
  // Widths for options/tasks table output.
  exports.widths = [1, col1len, 2, 76 - col1len];
};
```
### table方法

通过封装调用`grunt.log.writetableln`方法，将传入的`arr`数组进行表格化输出。

```javascript
// Render an array in table form.
exports.table = function(arr) {
  arr.forEach(function(item) {
    grunt.log.writetableln(exports.widths, ['', grunt.util._.pad(item[0], col1len), '', item[1]]);
  });
};
```
对于输出的结果，`exports.widths`和`col1len`在格式部分都起着关键的作用。

### queue属性

将一个字符串数组赋值给`queue`属性，这些数组项将在稍后的`display`方法中进行遍历并调用。

实际上这些数组项的内容都是本模块的方法名。

```javascript
// Methods to run, in-order.
exports.queue = [
  'initOptions',
  'initTasks',
  'initWidths',
  'header',
  'usage',
  'options',
  'optionsFooter',
  'tasks',
  'footer',
];
```
### display方法

遍历`queue`数组中的每一个值，并且对每一个对应的方法进行调用。

通过这种方式实现了一个简单的按步骤初始化并且输出显示的功能。

```javascript
// Actually display stuff.
exports.display = function() {
  exports.queue.forEach(function(name) { exports[name](); });
};
```

### header方法

调用`grunt.log.writeln`并且引用`grunt.version`的值，输出信息到终端。

```javascript
// Header.
exports.header = function() {
  grunt.log.writeln('Grunt: The JavaScript Task Runner (v' + grunt.version + ')');
};
```
信息输出如下：

```bash
Grunt: The JavaScript Task Runner (v0.4.1)
```
### usage方法

调用`grunt.log.header`和`grunt.log.writeln`按格式输出如何使用的信息到终端。

```javascript
// Usage info.
exports.usage = function() {
  grunt.log.header('Usage');
  grunt.log.writeln(' ' + path.basename(process.argv[1]) + ' [options] [task [task ...]]');
};
```
信息输出如下：

```bash
Usage
 grunt [options] [task [task ...]]
```

在这里`process.argv[1]`将得到当前运行环境下的全局的grunt-cli路径，例如在我的机器上，该路径如下：

```javascript
C:\Users\user1\AppData\Roaming\npm\node_modules\grunt-cli\bin\grunt
```
因为`path.basename`将会得到一个路径的最后一部分，所以这里的表达式`path.asename(process.argv[1])`得到的值是grunt.

查看[path.basename](http://nodejs.org/api/path.html#path_path_basename_p_ext)和[process.argv](http://nodejs.org/api/process.html#process_process_argv)了解详情。

### initOptions方法

该方法用来给本模块的`_options`属性进行初始化。

```javascript
// Options.
exports.initOptions = function() {
```
通过调用[Object.keys][]遍历`grunt.cli.optlist`的值，然后通过`initCol1`方法对`col1len`变量值进行修正，使得`col1len`变量值始终为表格式输出第一列中出现的最长内容的字符长度值。

```javascript
  // Build 2-column array for table view.
  exports._options = Object.keys(grunt.cli.optlist).map(function(long) {
    var o = grunt.cli.optlist[long];
    var col1 = '--' + (o.negate ? 'no-' : '') + long + (o.short ? ', -' + o.short : '');
    exports.initCol1(col1);
    return [col1, o.info];
  });
};
```

### options方法

通过调用`grunt.log.header`和本模块的`table`方法输出帮助信息的"Options"部分。

```javascript
exports.options = function() {
  grunt.log.header('Options');
  exports.table(exports._options);
};
```
信息输出如下：

```bash
Options
    --help, -h  Display this help text.
        --base  Specify an alternate base path. By default, all file paths are
                relative to the Gruntfile. (grunt.file.setBase) *
    --no-color  Disable colored output.
   --gruntfile  Specify an alternate Gruntfile. By default, grunt looks in the
                current or parent directories for the nearest Gruntfile.js or
                Gruntfile.coffee file.
   --debug, -d  Enable debugging mode for tasks that support it.
       --stack  Print a stack trace when exiting with a warning or fatal error.
   --force, -f  A way to force your way past warnings. Want a suggestion? Don't
                use this option, fix your code.
       --tasks  Additional directory paths to scan for task and "extra" files.
                (grunt.loadTasks) *
         --npm  Npm-installed grunt plugins to scan for task and "extra" files.
                (grunt.loadNpmTasks) *
    --no-write  Disable writing files (dry run).
 --verbose, -v  Verbose mode. A lot more information output.
 --version, -V  Print the grunt version. Combine with --verbose for more info.
  --completion  Output shell auto-completion rules. See the grunt-cli
                documentation for more information.

```

### optionsFooter方法

调用`grunt.log.writeln`方法在终端中为"Options"帮助信息输出结尾部分。

```javascript
exports.optionsFooter = function() {
  grunt.log.writeln().writelns(
    'Options marked with * have methods exposed via the grunt API and should ' +
    'instead be specified inside the Gruntfile wherever possible.'
  );
};
```
信息输出如下：

```javascript
Options marked with * have methods exposed via the grunt API and should instead be specified inside the Gruntfile wherever possible.
```

### initTasks方法

将当前项目下的任务相关信息以数组形式赋值给`_tasks`属性。

```javascript
// Tasks.
exports.initTasks = function() {
```
调用`grunt.task.init`方法对当前任务系统进行初始化。

```javascript
  // Initialize task system so that the tasks can be listed.
  grunt.task.init([], {help: true});
```
初始化`_tasks`属性，通过[Object.keys][]将`grunt.task._tasks`进行遍历，并且将任务的相关内容压入`_tasks`队列。

```javascript
  // Build object of tasks by info (where they were loaded from).
  exports._tasks = [];
  Object.keys(grunt.task._tasks).forEach(function(name) {
    exports.initCol1(name);
    var task = grunt.task._tasks[name];
    exports._tasks.push(task);
  });
};
```
这里的`exports.initCol1`调用将会保持`col1len`始终未所有任务名称中最长的值，以保证输出表格的美观。

### tasks方法

本方法将当前gruntfile中的可用任务进行输出。

```javascript
exports.tasks = function() {
  grunt.log.header('Available tasks');
```

如果可用任务列表长度为0,
```javascript
  if (exports._tasks.length === 0) {
    grunt.log.writeln('(no tasks found)');
  } else {
```
则直接输出如下信息：

```bash
(no tasks found)
```
否则的话，调用本模块的`table`方法进行输出。

在正式输出前，将`_tasks`数组中的数组项通过[Array.prototype.map]方法进行处理，对于多任务属性的任务，在任务的`info`属性值后追加` *`来标示多任务属性。

```javascript
    exports.table(exports._tasks.map(function(task) {
      var info = task.info;
      if (task.multi) { info += ' *'; }
      return [task.name, info];
    }));
```
这部分的输出结果如下示，（示例信息来源于jquery的gruntfile）：

```bash
Available tasks
        compare_size  Compare working size to saved sizes *
   compare_size:list  List saved sizes
    compare_size:add  Add to saved sizes
 compare_size:remove  Remove from saved sizes
  compare_size:prune  Clear all saved sizes except those specified
  compare_size:empty  Custom task.
   compare_size_list  Custom task.
    compare_size_add  Custom task.
 compare_size_remove  Custom task.
  compare_size_empty  Custom task.
             authors  Generate a list of authors in order of first contributio
               watch  Run predefined tasks whenever watched files change.
              jshint  Validate files with JSHint. *
              uglify  Minify files with UglifyJS. *
            jsonlint  Validate JSON files. *
               docco  Docco processor. *
               build  Concatenate source, remove sub AMD definitions,
                      (include/exclude modules with +/- flags), embed
                      date/version *
              custom  Custom task.
                dist  Custom task.
           testswarm  Custom task.
          pre-uglify  Custom multi task. *
         post-uglify  Custom multi task. *
                 dev  Alias for "build:*:*", "jshint" tasks.
             default  Alias for "jsonlint", "dev", "pre-uglify", "uglify",
                      "post-uglify", "dist:*", "compare_size" tasks.
```
在可用任务输出完毕后，输出任务列表的说明信息：

```javascript
    grunt.log.writeln().writelns(
      'Tasks run in the order specified. Arguments may be passed to tasks that ' +
      'accept them by using colons, like "lint:files". Tasks marked with * are ' +
      '"multi tasks" and will iterate over all sub-targets if no argument is ' +
      'specified.'
    );
  }
```
信息输出如下：

```bash
Tasks run in the order specified. Arguments may be passed to tasks that accept
them by using colons, like "lint:files". Tasks marked with * are "multi tasks"
and will iterate over all sub-targets if no argument is specified.
```
当然，无论可用任务列表是否为空，都会执行下面的代码。


```javascript
  grunt.log.writeln().writelns(
    'The list of available tasks may change based on tasks directories or ' +
    'grunt plugins specified in the Gruntfile or via command-line options.'
  );
};
```
在任务帮助信息部分输出如下内容：

```bash
The list of available tasks may change based on tasks directories or grunt
plugins specified in the Gruntfile or via command-line options.
```
### footer方法

调用`grunt.log.writeln`方法在终端中输出帮助信息的结尾部分。

```javascript
// Footer.
exports.footer = function() {
  grunt.log.writeln().writeln('For more information, see http://gruntjs.com/');
};
```
输出结果如下：

```bash
For more information, see http://gruntjs.com/
```

__如果想弄明白本模块的实际作用，最简单的方法就是在已经安装好grunt的项目目录下运行`grunt -help`，通过对比以上的方法说明应该会有更深的理解。__

[grunt.js]: https://github.com/gruntjs/grunt/blob/master/lib/grunt.js "grunt.js"

[path]: http://nodejs.org/api/path.html "path"

[Object.keys]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys "Object.keys"

[Array.prototype.map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map "Array.prototype.map"

[Math.max]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/max "Math.max"
