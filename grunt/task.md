# Grunt源码解析之'/grunt/task.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/task.js

本模块是grunt中的核心模块，主要涉及`grunt`中任务相关的操作，以及常用方法的封装。

## grunt内部调用模块

这里调用了[grunt.js][]模块，这也是Grunt最核心的模块（最终暴露给外界的模块）。

```javascript
var grunt = require('../grunt');
```
## node.js内部调用模块

```javascript
// Nodejs Libs
var path = require('path');
```
* [path][]模块包含了一系列的处理和转换文件路径的工具方法，实际上几乎所有的方法都在是做字符串转换，而不检查路径是否真实有效。

## 模块内部代码解析

### 变量parent

通过`grunt.util.task`即[util/task.js][]模块的`create`方法获得一个Task实例并将变量`parent`指向这个实例。

```javascript
// Extend generic "task" util lib.
var parent = grunt.util.task.create();
```
### 变量task

通过[Object.create][]方法获得一个新的对象实像并赋值给`module.exports`变量，然后将`module.exports`指向`task`变量，使`task`变量作为本模块的输出。

因为`task`变量是由`Object.create(parent)`获得的，所以`task`变量的`prototype`对象为`parent`变量，这意味着`task`变量继承了[util/task.js][]模块的Task对象所有属性和方法。

```javascript
// The module to be exported.
var task = module.exports = Object.create(parent);
```

### 变量registry

定义`registry`变量来存储与任务相关的变量信息，其中`tasks`为；`untasks`为；`meta`为。

```javascript
// A temporary registry of tasks and metadata.
var registry = {tasks: [], untasks: [], meta: {}};
```

### 变量lastInfo

用来存储最后一个任务的信息。

```javascript
// The last specified tasks message.
var lastInfo;
```
### 变量loadTaskDepth

用来记录加载任务时，任务集合的递归层次。

```javascript
// Number of levels of recursion when loading tasks in collections.
var loadTaskDepth = 0;
```

### 变量errorcount

用来记录错误数量。

```javascript
// Keep track of the number of log.error() calls.
var errorcount;
```

### task.registerTask方法

在[util/task.js][]模块的Task对象中有一个`registerTask`的方法，这里的registerTask对原方法进行了重写。

```javascript
// Override built-in registerTask.
task.registerTask = function(name) {
```
首先，将任务名称`name`压入`registry`变量对象的`tasks`数组中。

```javascript
  // Add task to registry.
  registry.tasks.push(name);
```
然后，通过调用`parent`的`registerTask`方法，对当前任务进行注册。

```javascript
  // Register task.
  parent.registerTask.apply(task, arguments);
```
将已经注册的任务对象通过`task._tasks[name]`取出并赋值给`thisTask`变量。并且通过调用`grunt.util._.clone`方法将`registry`的`meta`对象克隆病赋值给`thisTask.meta`。

```javascript
  // This task, now that it's been registered.
  var thisTask = task._tasks[name];
  // Metadata about the current task.
  thisTask.meta = grunt.util._.clone(registry.meta);
```
将`thisTask.fn`赋值给变量`_fn`，然后对`thisTask.fn`进行重写。

```javascript
  // Override task function.
  var _fn = thisTask.fn;
  thisTask.fn = function(arg) {
```

将实际任务名`thisTask.name`赋值给`name`变量，将`grunt.file.errorcount`赋值给`errorcount`变量进行初始化。

```javascript
    // Guaranteed to always be the actual task name.
    var name = thisTask.name;
    // Initialize the errorcount for this task.
    errorcount = grunt.fail.errorcount;
```
通过[Object.defineProperty][]为`thisTask`对象增加`errorCount`属性，属性值为`grunt.fail.errorcount`减去全局`errorcount`变量的值。

```javascript
    // Return the number of errors logged during this task.
    Object.defineProperty(this, 'errorCount', {
      enumerable: true,
      get: function() {
        return grunt.fail.errorcount - errorcount;
      }
    });
```
将`task`的`requires`方法绑定`task`为上下文后指向给`thisTask`的`requires`方法。

将`grunt.config.requires`方法指向`thisTask`的`requireConfig`方法。



```javascript
    // Expose task.requires on `this`.
    this.requires = task.requires.bind(task);
    // Expose config.requires on `this`.
    this.requiresConfig = grunt.config.requires;
    // Return an options object with the specified defaults overwritten by task-
    // specific overrides, via the "options" property.
    this.options = function() {
      var args = [{}].concat(grunt.util.toArray(arguments)).concat([
        grunt.config([name, 'options'])
      ]);
      var options = grunt.util._.extend.apply(null, args);
      grunt.verbose.writeflags(options, 'Options');
      return options;
    };
```

```javascript
    // If this task was an alias or a multi task called without a target,
    // only log if in verbose mode.
    var logger = _fn.alias || (thisTask.multi && (!arg || arg === '*')) ? 'verbose' : 'log';
    // Actually log.
    grunt[logger].header('Running "' + this.nameArgs + '"' +
      (this.name !== this.nameArgs ? ' (' + this.name + ')' : '') + ' task');
    // If --debug was specified, log the path to this task's source file.
    grunt[logger].debug('Task source: ' + thisTask.meta.filepath);
    // Actually run the task.
    return _fn.apply(this, arguments);
  };
  return task;
};
```

### isValidMultiTaskTarget方法

通过正则表达式来验证当前的`target`是否是一个有效的多任务目标。

```javascript
// Multi task targets can't start with _ or be a reserved property (options).
function isValidMultiTaskTarget(target) {
  return !/^_|^options$/.test(target);
}
```

### normalizeMultiTaskFiles方法



// Normalize multi task files.
task.normalizeMultiTaskFiles = function(data, target) {
  var prop, obj;
  var files = [];
  if (grunt.util.kindOf(data) === 'object') {
    if ('src' in data || 'dest' in data) {
      obj = {};
      for (prop in data) {
        if (prop !== 'options') {
          obj[prop] = data[prop];
        }
      }
      files.push(obj);
    } else if (grunt.util.kindOf(data.files) === 'object') {
      for (prop in data.files) {
        files.push({src: data.files[prop], dest: grunt.config.process(prop)});
      }
    } else if (Array.isArray(data.files)) {
      grunt.util._.flatten(data.files).forEach(function(obj) {
        var prop;
        if ('src' in obj || 'dest' in obj) {
          files.push(obj);
        } else {
          for (prop in obj) {
            files.push({src: obj[prop], dest: grunt.config.process(prop)});
          }
        }
      });
    }
  } else {
    files.push({src: data, dest: grunt.config.process(target)});
  }

  // If no src/dest or files were specified, return an empty files array.
  if (files.length === 0) {
    grunt.verbose.writeln('File: ' + '[no files]'.yellow);
    return [];
  }

  // Process all normalized file objects.
  files = grunt.util._(files).chain().forEach(function(obj) {
    if (!('src' in obj) || !obj.src) { return; }
    // Normalize .src properties to flattened array.
    if (Array.isArray(obj.src)) {
      obj.src = grunt.util._.flatten(obj.src);
    } else {
      obj.src = [obj.src];
    }
  }).map(function(obj) {
    // Build options object, removing unwanted properties.
    var expandOptions = grunt.util._.extend({}, obj);
    delete expandOptions.src;
    delete expandOptions.dest;

    // Expand file mappings.
    if (obj.expand) {
      return grunt.file.expandMapping(obj.src, obj.dest, expandOptions).map(function(mapObj) {
        // Copy obj properties to result.
        var result = grunt.util._.extend({}, obj);
        // Make a clone of the orig obj available.
        result.orig = grunt.util._.extend({}, obj);
        // Set .src and .dest, processing both as templates.
        result.src = grunt.config.process(mapObj.src);
        result.dest = grunt.config.process(mapObj.dest);
        // Remove unwanted properties.
        ['expand', 'cwd', 'flatten', 'rename', 'ext'].forEach(function(prop) {
          delete result[prop];
        });
        return result;
      });
    }

    // Copy obj properties to result, adding an .orig property.
    var result = grunt.util._.extend({}, obj);
    // Make a clone of the orig obj available.
    result.orig = grunt.util._.extend({}, obj);

    if ('src' in result) {
      // Expose an expand-on-demand getter method as .src.
      Object.defineProperty(result, 'src', {
        enumerable: true,
        get: function fn() {
          var src;
          if (!('result' in fn)) {
            src = obj.src;
            // If src is an array, flatten it. Otherwise, make it into an array.
            src = Array.isArray(src) ? grunt.util._.flatten(src) : [src];
            // Expand src files, memoizing result.
            fn.result = grunt.file.expand(expandOptions, src);
          }
          return fn.result;
        }
      });
    }

    if ('dest' in result) {
      result.dest = obj.dest;
    }

    return result;
  }).flatten().value();

  // Log this.file src and dest properties when --verbose is specified.
  if (grunt.option('verbose')) {
    files.forEach(function(obj) {
      var output = [];
      if ('src' in obj) {
        output.push(obj.src.length > 0 ? grunt.log.wordlist(obj.src) : '[no src]'.yellow);
      }
      if ('dest' in obj) {
        output.push('-> ' + (obj.dest ? String(obj.dest).cyan : '[no dest]'.yellow));
      }
      if (output.length > 0) {
        grunt.verbose.writeln('Files: ' + output.join(' '));
      }
    });
  }

  return files;
};

// This is the most common "multi task" pattern.
task.registerMultiTask = function(name, info, fn) {
  // If optional "info" string is omitted, shuffle arguments a bit.
  if (fn == null) {
    fn = info;
    info = 'Custom multi task.';
  }
  // Store a reference to the task object, in case the task gets renamed.
  var thisTask;
  task.registerTask(name, info, function(target) {
    // Guaranteed to always be the actual task name.
    var name = thisTask.name;
    // Arguments (sans target) as an array.
    this.args = grunt.util.toArray(arguments).slice(1);
    // If a target wasn't specified, run this task once for each target.
    if (!target || target === '*') {
      return task.runAllTargets(name, this.args);
    } else if (!isValidMultiTaskTarget(target)) {
      throw new Error('Invalid target "' + target + '" specified.');
    }
    // Fail if any required config properties have been omitted.
    this.requiresConfig([name, target]);
    // Return an options object with the specified defaults overwritten by task-
    // and/or target-specific overrides, via the "options" property.
    this.options = function() {
      var targetObj = grunt.config([name, target]);
      var args = [{}].concat(grunt.util.toArray(arguments)).concat([
        grunt.config([name, 'options']),
        grunt.util.kindOf(targetObj) === 'object' ? targetObj.options : {}
      ]);
      var options = grunt.util._.extend.apply(null, args);
      grunt.verbose.writeflags(options, 'Options');
      return options;
    };
    // Expose the current target.
    this.target = target;
    // Recreate flags object so that the target isn't set as a flag.
    this.flags = {};
    this.args.forEach(function(arg) { this.flags[arg] = true; }, this);
    // Expose data on `this` (as well as task.current).
    this.data = grunt.config([name, target]);
    // Expose normalized files object.
    this.files = task.normalizeMultiTaskFiles(this.data, target);
    // Expose normalized, flattened, uniqued array of src files.
    Object.defineProperty(this, 'filesSrc', {
      enumerable: true,
      get: function() {
        return grunt.util._(this.files).chain().pluck('src').flatten().uniq().value();
      }.bind(this)
    });
    // Call original task function, passing in the target and any other args.
    return fn.apply(this, this.args);
  });

  thisTask = task._tasks[name];
  thisTask.multi = true;
};

// Init tasks don't require properties in config, and as such will preempt
// config loading errors.
task.registerInitTask = function(name, info, fn) {
  task.registerTask(name, info, fn);
  task._tasks[name].init = true;
};

// Override built-in renameTask to use the registry.
task.renameTask = function(oldname, newname) {
  var result;
  try {
    // Actually rename task.
    result = parent.renameTask.apply(task, arguments);
    // Add and remove task.
    registry.untasks.push(oldname);
    registry.tasks.push(newname);
    // Return result.
    return result;
  } catch(e) {
    grunt.log.error(e.message);
  }
};

// If a property wasn't passed, run all task targets in turn.
task.runAllTargets = function(taskname, args) {
  // Get an array of sub-property keys under the given config object.
  var targets = Object.keys(grunt.config.getRaw(taskname) || {});
  // Fail if there are no actual properties to iterate over.
  if (targets.length === 0) {
    grunt.log.error('No "' + taskname + '" targets found.');
    return false;
  }
  // Iterate over all valid target properties, running a task for each.
  targets.filter(isValidMultiTaskTarget).forEach(function(target) {
    // Be sure to pass in any additionally specified args.
    task.run([taskname, target].concat(args || []).join(':'));
  });
};

// Load tasks and handlers from a given tasks file.
var loadTaskStack = [];
function loadTask(filepath) {
  // In case this was called recursively, save registry for later.
  loadTaskStack.push(registry);
  // Reset registry.
  registry = {tasks: [], untasks: [], meta: {info: lastInfo, filepath: filepath}};
  var filename = path.basename(filepath);
  var msg = 'Loading "' + filename + '" tasks...';
  var regCount = 0;
  var fn;
  try {
    // Load taskfile.
    fn = require(path.resolve(filepath));
    if (typeof fn === 'function') {
      fn.call(grunt, grunt);
    }
    grunt.verbose.write(msg).ok();
    // Log registered/renamed/unregistered tasks.
    ['un', ''].forEach(function(prefix) {
      var list = grunt.util._.chain(registry[prefix + 'tasks']).uniq().sort().value();
      if (list.length > 0) {
        regCount++;
        grunt.verbose.writeln((prefix ? '- ' : '+ ') + grunt.log.wordlist(list));
      }
    });
    if (regCount === 0) {
      grunt.verbose.warn('No tasks were registered or unregistered.');
    }
  } catch(e) {
    // Something went wrong.
    grunt.log.write(msg).error().verbose.error(e.stack).or.error(e);
  }
  // Restore registry.
  registry = loadTaskStack.pop() || {};
}

// Log a message when loading tasks.
function loadTasksMessage(info) {
  // Only keep track of names of top-level loaded tasks and collections,
  // not sub-tasks.
  if (loadTaskDepth === 0) { lastInfo = info; }
  grunt.verbose.subhead('Registering ' + info + ' tasks.');
}

// Load tasks and handlers from a given directory.
function loadTasks(tasksdir) {
  try {
    var files = grunt.file.glob.sync('*.{js,coffee}', {cwd: tasksdir, maxDepth: 1});
    // Load tasks from files.
    files.forEach(function(filename) {
      loadTask(path.join(tasksdir, filename));
    });
  } catch(e) {
    grunt.log.verbose.error(e.stack).or.error(e);
  }
}

// Load tasks and handlers from a given directory.
task.loadTasks = function(tasksdir) {
  loadTasksMessage('"' + tasksdir + '"');
  if (grunt.file.exists(tasksdir)) {
    loadTasks(tasksdir);
  } else {
    grunt.log.error('Tasks directory "' + tasksdir + '" not found.');
  }
};

// Load tasks and handlers from a given locally-installed Npm module (installed
// relative to the base dir).
task.loadNpmTasks = function(name) {
  loadTasksMessage('"' + name + '" local Npm module');
  var root = path.resolve('node_modules');
  var pkgfile = path.join(root, name, 'package.json');
  var pkg = grunt.file.exists(pkgfile) ? grunt.file.readJSON(pkgfile) : {keywords: []};

  // Process collection plugins.
  if (pkg.keywords && pkg.keywords.indexOf('gruntcollection') !== -1) {
    loadTaskDepth++;
    Object.keys(pkg.dependencies).forEach(function(depName) {
      // Npm sometimes pulls dependencies out if they're shared, so find
      // upwards if not found locally.
      var filepath = grunt.file.findup('node_modules/' + depName, {
        cwd: path.resolve('node_modules', name),
        nocase: true
      });
      if (filepath) {
        // Load this task plugin recursively.
        task.loadNpmTasks(path.relative(root, filepath));
      }
    });
    loadTaskDepth--;
    return;
  }

  // Process task plugins.
  var tasksdir = path.join(root, name, 'tasks');
  if (grunt.file.exists(tasksdir)) {
    loadTasks(tasksdir);
  } else {
    grunt.log.error('Local Npm module "' + name + '" not found. Is it installed?');
  }
};

// Initialize tasks.
task.init = function(tasks, options) {
  if (!options) { options = {}; }

  // Were only init tasks specified?
  var allInit = tasks.length > 0 && tasks.every(function(name) {
    var obj = task._taskPlusArgs(name).task;
    return obj && obj.init;
  });

  // Get any local Gruntfile or tasks that might exist. Use --gruntfile override
  // if specified, otherwise search the current directory or any parent.
  var gruntfile = allInit ? null : grunt.option('gruntfile') ||
    grunt.file.findup('Gruntfile.{js,coffee}', {nocase: true});

  var msg = 'Reading "' + (gruntfile ? path.basename(gruntfile) : '???') + '" Gruntfile...';
  if (gruntfile && grunt.file.exists(gruntfile)) {
    grunt.verbose.writeln().write(msg).ok();
    // Change working directory so that all paths are relative to the
    // Gruntfile's location (or the --base option, if specified).
    process.chdir(grunt.option('base') || path.dirname(gruntfile));
    // Load local tasks, if the file exists.
    loadTasksMessage('Gruntfile');
    loadTask(gruntfile);
  } else if (options.help || allInit) {
    // Don't complain about missing Gruntfile.
  } else if (grunt.option('gruntfile')) {
    // If --config override was specified and it doesn't exist, complain.
    grunt.log.writeln().write(msg).error();
    grunt.fatal('Unable to find "' + gruntfile + '" Gruntfile.', grunt.fail.code.MISSING_GRUNTFILE);
  } else if (!grunt.option('help')) {
    grunt.verbose.writeln().write(msg).error();
    grunt.log.writelns(
      'A valid Gruntfile could not be found. Please see the getting ' +
      'started guide for more information on how to configure grunt: ' +
      'http://gruntjs.com/getting-started'
    );
    grunt.fatal('Unable to find Gruntfile.', grunt.fail.code.MISSING_GRUNTFILE);
  }

  // Load all user-specified --npm tasks.
  (grunt.option('npm') || []).forEach(task.loadNpmTasks);
  // Load all user-specified --tasks.
  (grunt.option('tasks') || []).forEach(task.loadTasks);
};
```



[grunt.js]: https://github.com/gruntjs/grunt/blob/master/lib/grunt.js "grunt.js"
[util/task.js]: https://github.com/gruntjs/grunt/blob/master/lib/util/task.js "util/task.js"
[Object.create]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create "Object.create"
[Object.defineProperty]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty "Object.defineProperty"
