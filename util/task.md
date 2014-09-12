# Grunt源码解析之'/util/task.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/util/task.js

## 模块内部代码解析

本模块定义了Grunt中最核心的`Task`"类"。

构造函数被包裹在一个立即执行函数体内，接受一个`exports`参数，这样保证了相关参数不会污染整个环境。

```javascript
(function(exports) {
```
### Task构造函数

构造函数本身非常简单，只是初始化了一些属性对象。

```javascript
  // Construct-o-rama.
  function Task() {
    // Information about the currently-running task.
    // 当前运行的任务对象
    this.current = {};
    // Tasks.
    // 任务列表对象，各任务对象存储在相应任务名称的属性中
    this._tasks = {};
    // Task queue.
    // 任务队列
    this._queue = [];
    // Queue placeholder (for dealing with nested tasks).
    // 占位符对象，对于内嵌任务列表的任务适用，最广为人知的内嵌任务是`default`
    this._placeholder = {placeholder: true};
    // Queue marker (for clearing the queue programmatically).
    // 标记对象，其实也是一个占位符对象，对于需要清除列表内容时占位适用
    this._marker = {marker: true};
    // Options.
    this._options = {};
    // Is the queue running?
    // 当前队列运行标记
    this._running = false;
    // Success status of completed tasks.
    // 已完成的任务状态对象
    this._success = {};
  }
```
### exports.Task接口

将构造函数通过模块的`Task`属性进行暴露。

```javascript
  // Expose the constructor function.
  exports.Task = Task;
```
### exports.create接口

将构造`Task`类实例的功能通过模块的`create`属性进行暴露，可以方便的通过模块的`create`方法创建新的任务实例。

```javascript
  // Create a new Task instance.
  exports.create = function() {
    return new Task();
  };
```

### \_throwIfRunning方法

工具方法，在`this.run`方法中被调用，针对当前任务运行状态进行不同的错误处理。

```javascript
  // If the task runner is running or an error handler is not defined, throw
  // an exception. Otherwise, call the error handler directly.
  Task.prototype._throwIfRunning = function(obj) {
```
如果当前任务正在运行或者在\_options对象中并未定义error的响应函数则抛出obj

```javascript
    if (this._running || !this._options.error) {
      // Throw an exception that the task runner will catch.
      throw obj;
    } else {
```
如果当前任务未在运行且定义了error handler则直接调用error handler.

```javascript
      // Not inside the task runner. Call the error handler and abort.
      this._options.error.call({name: null}, obj);
    }
  };
```

### registerTask方法

用来注册新的任务，方法接受三个参数，分别是任务名`name`，任务描述`info`以及任务方法`fn`。

```javascript
  // Register a new task.
  Task.prototype.registerTask = function(name, info, fn) {
```
对于方法签名与实际传入参数长度不符的惯常做法，这里的`info`可以省略，即方法签名可以看做`(name, fn)`，这种情况下，`info`将被设为`null`。

```javascript
    // If optional "info" string is omitted, shuffle arguments a bit.
    if (fn == null) {
      fn = info;
      info = null;
    }
```
    实际上，Grunt的task定义有两种方式，一种是已定义的task或者task组合的方式，还有一种是全新的fn函数定义。

    在第一种方式下，`fn`的类型为`String`或者`Array`，第二种方式`fn`是函数类型即`function`。

在第一种情况下，通过`this.parseArgs`方法来处理`[fn]`，将结果数组赋值给`tasks`变量。

```javascript
    // String or array of strings was passed instead of fn.
    var tasks;
    if (typeof fn !== 'function') {
      // Array of task names.
      tasks = this.parseArgs([fn]);
```
紧接着，通过`this.run`调用[Function.prototype.bind][]方法，将当前的`this`对象绑定到`this.run`并且将`fn`作为方法参数进行传入，构造出了一个新的方法赋值给`fn`变量。

因为这个任务只是若干任务的一种组合，所以`fn.alias`属性设为`true`,认为这是别名的一种。

```javascript
      // This task function just runs the specified tasks.
      fn = this.run.bind(this, fn);
      fn.alias = true;
```
如果当前的任务（组合）在注册时，并没有传入`info`参数，那么通过`tasks`数组变量调用[Array.prototype.join][]方法将组合中的各任务名通过"”，“"进行连接，并将结果赋值给`info`变量。

```javascript
      // Generate an info string if one wasn't explicitly passed.
      if (!info) {
        info = 'Alias for "' + tasks.join('", "') + '" task' +
          (tasks.length === 1 ? '' : 's') + '.';
      }
```
如果是第二种情况，但是也没有传入`info`参数，那么赋予`info`值为"Custom task."。

```javascript
    } else if (!info) {
      info = 'Custom task.';
    }
```
接下来，在`this._tasks`任务列表对象中添加名称为`name`变量值的属性，值为包含属性值"name", "info", "fn"的对象。

最后返回`this`对象。

```javascript
    // Add task into cache.
    this._tasks[name] = {name: name, info: info, fn: fn};
    // Make chainable!
    return this;
  };
```
### isTaskAlias方法

判断`name`名称的任务是否存在别名。

```javascript
  // Is the specified task an alias?
  Task.prototype.isTaskAlias = function(name) {
    return !!this._tasks[name].fn.alias;
  };
```
这里有一个技巧是对`!!`的应用，无论`this._tasks[name]`是否为`undefined`，或者`this._tasks[name].fn`的`alias`属性是否为`undefined`，该返回结果都不会抛出异常。

希望对这一点有更多了解的可以查看[what-is-the-not-not-operator-in-javascript](http://stackoverflow.com/questions/784929/what-is-the-not-not-operator-in-javascript)。

### exists方法

判断`name`名称的任务是否存在任务列表对象`this._tasks`中。

有则返回`true`否则返回`false`。

```javascript
  // Has the specified task been registered?
  Task.prototype.exists = function(name) {
    return name in this._tasks;
  };
```
### renameTask方法

重命名一个任务。

方法接受两个参数，第一个为`oldname`旧的任务名，第二个为`newname`新的任务名。

```javascript
  // Rename a task. This might be useful if you want to override the default
  // behavior of a task, while retaining the old name. This is a billion times
  // easier to implement than some kind of in-task "super" functionality.
  Task.prototype.renameTask = function(oldname, newname) {
```
如果`oldname`命名的任务在`this._tasks`任务列表对象中不存在，则抛出异常。

```javascript
    if (!this._tasks[oldname]) {
      throw new Error('Cannot rename missing "' + oldname + '" task.');
    }
```
否则的话，在`this._tasks`中创建属性名为`newname`变量值的属性，值为旧任务的对象值。

将任务对象的名称`name`进行修正，赋值为`newname`变量值。然后通过[delete][]操作删除`this._tasks`对象中`oldname`属性值所对应的属性。

最后返回`this`。

```javascript
    // Rename task.
    this._tasks[newname] = this._tasks[oldname];
    // Update name property of task.
    this._tasks[newname].name = newname;
    // Remove old name.
    delete this._tasks[oldname];
    // Make chainable!
    return this;
  };
```

### parseArgs方法

本方法是一个工具方法，用来解析传入参数，以数组形式返回。

方法用例如下示：

```javascript
  // Argument parsing helper. Supports these signatures:
  //  fn('foo')                 // ['foo']
  //  fn('foo', 'bar', 'baz')   // ['foo', 'bar', 'baz']
  //  fn(['foo', 'bar', 'baz']) // ['foo', 'bar', 'baz']
  Task.prototype.parseArgs = function(args) {
```
方法主体也很简单，通过[Array.prototype.isArray][]判断第一个参数的类型，如果是数组的话则返回第一个参数，否则的话通过[Array.prototype.slice][]方法将参数列表转为一个数组进行返回。

```javascript
    // Return the first argument if it's an array, otherwise return an array
    // of all arguments.
    return Array.isArray(args[0]) ? args[0] : [].slice.call(args);
  };
```
### splitArgs方法

工具方法，将用冒号`:`分割的字符串转换为数组对象，但是处理时要跳过`\:`的情况。

```javascript
  // Split a colon-delimited string into an array, unescaping (but not
  // splitting on) any \: escaped colons.
  Task.prototype.splitArgs = function(str) {
```
如果`str`为`undefined`，则直接返回空数组`[]`。

否则的话，将`str`中所有的`\\`用`\uFFFF`进行替换，将所有的`\:`用`\uFFFE`进行替换。

```javascript
    if (!str) { return []; }
    // Store placeholder for \\ followed by \:
    str = str.replace(/\\\\/g, '\uFFFF').replace(/\\:/g, '\uFFFE');
```
    这种用法在各种lib中比较常见，因为`\uFFFF`和`\uFFFE`均不是有效的Unicode字符，这里用这两个字符分别替换有效的`\\`和`\:`只是为了各自对应的字符进行占位。

接下来将处理过的`str`变量用[Array.prototype.split][]方法按分割符`:`进行分割，然后通过[Array.prototype.map][]对新生成的数组进行遍历操作，将数组中的每一项的`\uFFFE`替换为`:`，将所有的`\uFFFF`替换为`\\`。

然后将新的数组作为结果进行返回。

```javascript
    // Split on :
    return str.split(':').map(function(s) {
      // Restore place-held : followed by \\
      return s.replace(/\uFFFE/g, ':').replace(/\uFFFF/g, '\\');
    });
  };
```
### \_taskPlusArgs方法

内部方法，在`this.run`方法中被调用，通过传递的`name`参数，来判断实际任务名称以及任务参数。

```javascript
  // Given a task name, determine which actual task will be called, and what
  // arguments will be passed into the task callback. "foo" -> task "foo", no
  // args. "foo:bar:baz" -> task "foo:bar:baz" with no args (if "foo:bar:baz"
  // task exists), otherwise task "foo:bar" with arg "baz" (if "foo:bar" task
  // exists), otherwise task "foo" with args "bar" and "baz".
  Task.prototype._taskPlusArgs = function(name) {
```
通过调用`this.splitArgs`方法处理`name`参数并将处理后的数组对象赋值给`parts`变量。

将`parts`数组的长度值赋值给`i`变量。

```javascript
    // Get task name / argument parts.
    var parts = this.splitArgs(name);
    // Start from the end, not the beginning!
    var i = parts.length;
```
通过[Array.prototype.slice][]方法对`parts`变量进行分割，得到从位置0开始指定长度为`i`的数组，然后使用[Array.prototype.join][]方法用`:`对该数组进行字符串拼接，将拼接的结果字符串作为属性名，并将`this._tasks`任务对象中的该属性的值赋值给`task`变量。

如果过`task`为`undefined`，即在`this._tasks`任务对象中不存在相应属性，并且`i`的值大于1，则继续进行下一次循环。如果循环结束后没能在`this._tasks`任务对象中找到相应属性，则此时的`i`值为0。

一旦`task`不为`undefined`，即在`this._tasks`任务对象中找到相应属性，则`task`为具体的任务对象，`!task`为false，不会再进行`--i`的操作。

```javascript
    var task;
    do {
      // Get a task.
      task = this._tasks[parts.slice(0, i).join(':')];
      // If the task doesn't exist, decrement `i`, and if `i` is greater than
      // 0, repeat.
    } while (!task && --i > 0);
```
通过以上这个[do...while][]语句，将对形如`foo:bar:baz`的`name`参数值，进行如下的判断操作:

任意一次的`task`变量如果不为`undefined`，则不再进行下一次循环，保留当前的`i`变量值。

```javascript
// 第一次执行
task = this._tasks["foo:bar:baz"];
i == 3

// 第二次执行
task = this._tasks["foo:bar"];
i == 2

// 第三次执行
task = this._tasks["foo"]
i == 1
```

接下来，通过[Array.prototype.slice][]获得从`i`位置开始的数组部分，即除任务名以外的参数部分，并赋值给`args`变量。

```javascript
    // Just the args.
    var args = parts.slice(i);
```
为了将参数作为标记使用，这里定义了`flags`变量对象，并且通过[Array.prototype.forEach][]遍历`args`参数列表，给`flags`对象添加对应参数名的属性，属性值为`true`。

然后将一个包含有"task","nameArgs","args"以及"flags"属性的对象作为结果返回。

```javascript
    // Maybe you want to use them as flags instead of as positional args?
    var flags = {};
    args.forEach(function(arg) { flags[arg] = true; });
    // The task to run and the args to run it with.
    return {task: task, nameArgs: name, args: args, flags: flags};
  };
```

### \_push方法

内部方法，在`this.run`和`this.mark`方法中被调用，用来对当前任务队列`this._queue`进行push操作。

方法接受一个数组类型的参数`things`。

```javascript
  // Append things to queue in the correct spot.
  Task.prototype._push = function(things) {
```
首先通过[Array.prototype.indexOf][]来获得`this._queue`中的`this._placeholder`位置，并赋值给`index`变量。

如果`index`为-1，说明不存在`this._placeholder`，则直接通过[Array.prototype.concat][]方法将`things`加到`this._queue`队列队尾。

否则的话，通过`this._queue`调用[Array.prototype.splice][]方法把`things`数组加入到`this._queue`的`index`位置，即加入到`this._placeholder`前面的位置。

```javascript
    // Get current placeholder index.
    var index = this._queue.indexOf(this._placeholder);
    if (index === -1) {
      // No placeholder, add task+args objects to end of queue.
      this._queue = this._queue.concat(things);
    } else {
      // Placeholder exists, add task+args objects just before placeholder.
      [].splice.apply(this._queue, [index, 0].concat(things));
    }
  };
```
### run方法

该方法并不是实际执行任务的方法，它在`this.registerTask`方法中被调用，用来将任务组合类型的任务定义进行处理，并将实际任务通过`this._push`方法加入当前的任务队列`this._queue`中。

```javascript
  // Enqueue a task.
  Task.prototype.run = function() {
```
首先，通过调用`this.parseArgs`方法将当前的参数对象`arguments`进行数组化，然后通过[Array.prototype.map][]方法对结果数组的每一项用`this._taskPlusArgs`方法进行处理，得到每一项所对应的包含任务信息的对象。

```javascript
    // Parse arguments into an array, returning an array of task+args objects.
    var things = this.parseArgs(arguments).map(this._taskPlusArgs, this);
```
    任务信息对象的格式如下：

```javascript
    {task: task, nameArgs: name, args: args, flags: flags};
```
通过[Array.prototype.filter][]方法对`things`数组变量进行处理，对数组项中`task`属性不是`undefined`的进行过滤，并将所有`task`属性未定义的任务数组赋值给`fails`变量。

```javascript
    // Throw an exception if any tasks weren't found.
    var fails = things.filter(function(thing) { return !thing.task; });
```
如果`fails`数组的长度大于0，将`fails`数组中第一个任务信息作为内容来实例化`Error`对象并传递给`this._throwIfRunning`方法，并返回`this`对象。

```javascript
    if (fails.length > 0) {
      this._throwIfRunning(new Error('Task "' + fails[0].nameArgs + '" not found.'));
      return this;
    }
```
否则的话，将`things`数组通过`this._push`方法压入任务队列`this._queue`。

然后返回`this`对象。

```javascript
    // Append things to queue in the correct spot.
    this._push(things);
    // Make chainable!
    return this;
  };
```
### mark方法

通过调用`this._push`方法在当前任务队列`this._queue`中加入`this._marker`标记，来为编程清理当前任务列表`_queue`提供便利。

```javascript
  // Add a marker to the queue to facilitate clearing it programmatically.
  Task.prototype.mark = function() {
    this._push(this._marker);
    // Make chainable!
    return this;
  };
```
###　runTaskFn方法

该方法用于实际运行任务对应的函数方法，在`this.start`方法中被调用。

```javascript
  // Run a task function, handling this.async / return value.
  Task.prototype.runTaskFn = function(context, fn, done, asyncDone) {
```
方法接受4个函数，其中`context`是一个包含有如下属性的对象。

```javascript
var context = {
    nameArgs: "",
    name: "",
    args: "",
    flags: ""
};
```

`fn`是当前任务对应的函数方法，`done`是当前任务完成后的回调方法，`asyncDone`标记当前任务是否为异步回调。

开始查看源码，首先定义`async`变量，用来标记当前任务的同步异步状态，默认为`false`即要运行的任务为同步状态。

```javascript
    // Async flag.
    var async = false;
```
定义`complete`函数，在任务结束后进行调用，接受一个参数`success`。

```javascript
    // Update the internal status object and run the next task.
    var complete = function(success) {
      var err = null;
```
当`success`为`false`时，意味着任务执行常规性失败，将`err`指向一个`Error`实例；如果`success`是一个`Error`实例或者是`Error`类型时，将`success`赋值给`err`变量，并且将`success`设为`false`；否则的话将`success`设为`true`。

```javascript
      if (success === false) {
        // Since false was passed, the task failed generically.
        err = new Error('Task "' + context.nameArgs + '" failed.');
      } else if (success instanceof Error || {}.toString.call(success) === '[object Error]') {
        // An error object was passed, so the task failed specifically.
        err = success;
        success = false;
      } else {
        // The task succeeded.
        success = true;
      }
```
将`this.current`对象重置为`{}`，并将`success`赋值给`this._success`对象的`context.nameArgs`属性。

```javascript
      // The task has ended, reset the current task object.
      this.current = {};
      // A task has "failed" only if it returns false (async) or if the
      // function returned by .async is passed false.
      this._success[context.nameArgs] = success;
```
如果`success`为`false`并且在`this._options`中定义了error handler，则调用错误回调函数。

```javascript
      // If task failed, call error handler.
      if (!success && this._options.error) {
        this._options.error.call({name: context.name, nameArgs: context.nameArgs}, err);
      }
```
如果`asyncDone`为`true`，则通过[process.nextTick][]异步调用`done`回调函数，否则的话直接调用`done`回调函数。

    对于[process.nextTick][]的详细说明，建议查看[理解 Node.js 里的 process.nextTick()](http://www.oschina.net/translate/understanding-process-next-tick).

```javascript
      // only call done async if explicitly requested to
      // see: https://github.com/gruntjs/grunt/pull/1026
      if (asyncDone) {
        process.nextTick(function () {
          done(err, success);
        });
      } else {
        done(err, success);
      }
```
这个[Function.prototype.bind][]绑定this到`complete`函数。

```javascript
    }.bind(this);
```
添加`async`方法到`context`变量。当调用这个`async`方法时，会将`async`变量设为`true`，表明这是一个异步处理任务。并且返回一个异步调用的函数。

```javascript
    // When called, sets the async flag and returns a function that can
    // be used to continue processing the queue.
    context.async = function() {
      async = true;
      // The returned function should execute asynchronously in case
      // someone tries to do this.async()(); inside a task (WTF).
      return function(success) {
```
这里返回的异步函数体使用[setTimeout][]包裹`complete(success)`函数的执行，这里的`setTimeout`只是为了实现延迟来避免类似`this.async()()`这样的调用，使返回的函数立即。

```javascript
        setTimeout(function() { complete(success); }, 1);
      };
    };
```
将`context`对象赋值给`this.current`即当前任务对象。

```javascript
    // Expose some information about the currently-running task.
    this.current = context;
```
使用[Function.prototype.call][]对`fn`进行调用，并将`context`指定为`fn`的this上下文。

如果`async`变量不为`true`（意味着`context.async`方法没有在`fn`执行的过程中被调用），则接着调用`complete(success)`对该任务执行进行收尾工作。

```javascript
    try {
      // Get the current task and run it, setting `this` inside the task
      // function to be something useful.
      var success = fn.call(context);
      // If the async flag wasn't set, process the next task in the queue.
      if (!async) {
        complete(success);
      }
```
如果过在上述过程中出现任何异常，则通过调用`complete(err)`进行任务收尾工作。

```javascript
    } catch (err) {
      complete(err);
    }
  };
```
### start方法

这个方法触发任务列表中的任务开始处理。

方法接受一个`opts`对象作为参数。

```javascript
  // Begin task queue processing. Ie. run all tasks.
  Task.prototype.start = function(opts) {
```
如果`opts`对象为`undefined`则初始化`opts`为一个空对象。

```javascript
    if (!opts) {
      opts = {};
    }
```
如果当前任务`this._running`为`true`，说明当前任务正在运行，则返回`false`。

```javascript
    // Abort if already running.
    if (this._running) { return false; }
```
定义内部函数`nextTask`来实际控制任务的运行，整个函数内部的逻辑模仿一个流水线。

```javascript
    // Actually process the next task.
    var nextTask = function() {
```
定义变量`thing`来指向任务队列中的下一个任务对象。

```javascript
      // Get next task+args object from queue.
      var thing;
```
因为这里采用了[do...while][]循环，这样就保证了至少执行一次循环，在循环体中通过调用[Array.prototype.shift][]方法返回`this._queue`任务队列中的第一个元素赋值给`thing`变量，并将该元素从任务队列中移除。

当`thing`不为`this._placeholder`和`this._marker`时将跳出循环，继续向下执行。

```javascript
      // Skip any placeholders or markers.
      do {
        thing = this._queue.shift();
      } while (thing === this._placeholder || thing === this._marker);
```
如果`thing`变量为`undefined`，意味着当前任务列表为空，则执行如下代码进行相关处理。

__只有当`this._queue`为空时，通过`thing = this._queue.shift()`会赋值`undefined`给`thing`，这意味整个任务列表已经处理完毕。__

```javascript
      // If queue was empty, we're all done.
      if (!thing) {
```
当任务列表遍历完毕后，重新设置`this._running`标记为`false`，说明当前任务已经不再运行状态。

如果`this._options`中定义了`done`的回调函数，那么对`this._options.done`进行调用，否则直接返回。

```javascript
        this._running = false;
        if (this._options.done) {
          this._options.done();
        }
        return;
      }
```
如果任务列表不为空，则通过[Array.prototype.unshift][]将`this._placeholder`压入`this._queue`任务队列的头部。

```javascript
      // Add a placeholder to the front of the queue.
      this._queue.unshift(this._placeholder);
```
定义`context`变量并用当前任务`thing`的各种属性初始化这个变量。

```javascript
      // Expose some information about the currently-running task.
      var context = {
        // The current task name plus args, as-passed.
        nameArgs: thing.nameArgs,
        // The current task name.
        name: thing.task.name,
        // The current task arguments.
        args: thing.args,
        // The current arguments, available as named flags.
        flags: thing.flags
      };
```
调用`this.runTaskFn`对任务对应的函数进行实际调用`thing.task.fn.apply(this,this.args)`。

```javascript
      // Actually run the task function (handling this.async, etc)
      this.runTaskFn(context, function() {
        return thing.task.fn.apply(this, this.args);
      }, nextTask, !!opts.asyncDone);
```
此处的[Function.prototype.bind][]是用来为`nextTask`函数绑定this。

```javascript
    }.bind(this);
```
设置`this._running`标记为`true`，标记当前任务运行状态，然后触发`nextTask`方法开始运行任务。

```javascript
    // Update flag.
    this._running = true;
    // Process the next task.
    nextTask();
  };
```

### clearQueue方法

这个方法将任务队列中还未完成的任务进行清理。

方法接受一个`options`对象作为参数。

```javascript
  // Clear remaining tasks from the queue.
  Task.prototype.clearQueue = function(options) {
```
当`options`对象为`undefined`时，初始化`options`为一个空对象。

```javascript
    if (!options) { options = {}; }
```
如果`options`对象中的`untilMarker`为`true`，则通过[Array.prototype.indexOf][]获得`this._marker`标记在数组`this._queue`中的位置，并且通过[Array.prototype.splice][]将`this._queue`数组从0号位置到`this._marker`后一个元素之间的元素都移除，进行操作后`this._queue`中的第一个元素将是原数组中`this._marker`后面的那个元素（如果有的话）。

如果`options`对象中不存在`untilMarker`或者`untilMarker`值为`false`的话，则直接将`this._queue`数组设为空数组。

```javascript
    if (options.untilMarker) {
      this._queue.splice(0, this._queue.indexOf(this._marker) + 1);
    } else {
      this._queue = [];
    }
```
返回`this`对象，保证方法的链式操作。

```javascript
    // Make chainable!
    return this;
  };
```
### requires方法

这个方法用来对`arguments`中的所有任务进行校验，只要有一个任务执行不成功就会抛出异常。

```javascript
  // Test to see if all of the given tasks have succeeded.
  Task.prototype.requires = function() {
```
方法内部调用了`parseArgs`方法，并且通过[Array.prototype.forEach][]方法对`parseArgs`返回的数组进行遍历处理。

```javascript
    this.parseArgs(arguments).forEach(function(name) {
```
通过`_success`对象来获取当前遍历任务的状态并赋值给`success`变量，如果`success`变量值为`false`或者`undefined`则会抛出异常，其中当值为`false`时说明任务运行失败，当为`undefined`时说明任务没有运行，根据两种不同情况给出相应的异常信息。

需要注意的是这里调用了[Function.prototype.bind][]，将匿名函数的`this`绑定到当前实例对象。

```javascript
      var success = this._success[name];
      if (!success) {
        throw new Error('Required task "' + name +
          '" ' + (success === false ? 'failed' : 'must be run first') + '.');
      }
    }.bind(this));
  };
```
### options方法

用来对`options`对象内容作出添加和修改，即在参数`options`对象中有但是`_options`对象中没有的属性进行添加；对两者都有的属性，用`options`中的值覆盖`_options`中的值。

方法接受一个`options`对象作为参数。

```javascript
  // Override default options.
  Task.prototype.options = function(options) {
```
通过[Object.prototype.keys][]得到`options`对象的属性值数组，然后通过[Array.prototype.forEach][]方法遍历这个数组并对原`_options`对象的值进行添加和修改。

```javascript
    Object.keys(options).forEach(function(name) {
      this._options[name] = options[name];
    }.bind(this));
  };
```
这里需要注意的是[Function.prototype.bind][]方法的使用，该方法用以指定方法内部的上下文`this`，这个匿名方法内部的`this`如果没有通过`bind`指定的话，在Node的运行环境下将指向当前这个模块的`exports`对象；而在Browser的运行环境下将指向window对象。这里通过[Function.prototype.bind][]将`this`指向了当前的实例对象。

乍一看，这个模块可以在Node.js以及Browser环境都可以运行，如果`exports`为null或者类型不是`object`，那么将`this`即`window`对象传入。

但是在上述源码中有对`process.nextTick`的调用，所以这个模块还是与Browser环境无缘。

```javascript
}(typeof exports === 'object' && exports || this));
```

[Function.prototype.bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind "Function.prototype.bind"

[Object.prototype.keys]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys "Object.prototype.keys"

[Array.prototype.slice]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/slice "Array.prototype.slice"

[Array.prototype.splice]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/splice "Array.prototype.splice"

[Array.prototype.isArray]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray "Array.prototype.isArray"

[Array.prototype.forEach]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach "Array.prototype.forEach"

[Array.prototype.indexOf]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/indexOf "Array.prototype.indexOf"

[Array.prototype.shift]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/shift "Array.prototype.shift"

[Array.prototype.unshift]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/unshift "Array.prototype.unshift"

[do...while]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/do...while "do...while"

[process.nextTick]: http://nodejs.org/api/process.html#process_process_nexttick_callback "process.nextTick"

[setTimeout]: http://nodejs.org/api/timers.html#timers_settimeout_callback_delay_arg "setTimeout"

[delete]: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/delete "delete"
