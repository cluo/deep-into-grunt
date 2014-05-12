# Grunt源码解析之'/grunt/config.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/config.js

该模块主要用来设置并且访问在Gruntfile文件中的配置数据。

## grunt内部调用模块

这里调用了[grunt.js][]模块，这也是Grunt最核心的模块（最终暴露给外界的模块）。

```javascript
var grunt = require('../grunt');
```
本模块没有调用或使用其他的Node.js的内部模块以及外部模块。

## 模块内部代码解析

### config变量

定义`config`变量作为模块返回的对象，该变量实际上指向了一个接受`prop`和`value`参数的方法。

当同时指定了`prop`和`value`的话，这个方法可以看做一个set方法，调用`config.set`方法来设置配置数据，如果只指定`prop`的话，这个方法可以看做一个get方法，调用`config.get`方法来读取相关属性的值。

```javascript
// Get/set config data. If value was passed, set. Otherwise, get.
var config = module.exports = function(prop, value) {
  if (arguments.length === 2) {
    // Two arguments were passed, set the property's value.
    return config.set(prop, value);
  } else {
    // Get the property's value (or the entire data object).
    return config.get(prop);
  }
};
```
### config.data

在`config`对象上定义了data属性，属性值默认为空对象`{}`。主要用这个属性来存储实际的配置数据。

```javascript
// The actual config data.
config.data = {};
```
### config.escape

这里的`config.escape`方法只是简单地封装了[String.prototype.replace][]方法，将参数`str`中的所有`.`替换为`\\.`。

```javascript
// Escape any . in name with \. so dot-based namespacing works properly.
config.escape = function(str) {
  return str.replace(/\./g, '\\.');
};
```

### config.getPropString

该方法用来得到属性名字符串。

它只接受一个参数`prop`，如果`prop`是一个数组的话（通过[Array.isArray][]方法来判断），则调用[Array.prototype.map][]方法对数组中的每一项都运行`config.escape`方法来得到新的数组，然后通过[Array.prototype.join][]方法将数组中的项目用"."号给连接成一个字符串并返回结果；否则的话，则将`prop`直接作为结果返回。

```javascript
// Return prop as a string.
config.getPropString = function(prop) {
  return Array.isArray(prop) ? prop.map(config.escape).join('.') : prop;
};
```
### config.getRaw

该方法用来获取原始的，未经处理的配置数据。

它只接受一个参数`prop`，如果`prop`不是`undefined`，则通过调用`config.getPropString`方法获得`prop`的属性名字符串，然后通过调用`grunt.util.namespace.get`方法从`config.data`对象中获取对应属性名字符串的值并返回。如果`prop`是`undefined`的话，则直接返回`config.data`对象。

```javascript
// Get raw, unprocessed config data.
config.getRaw = function(prop) {
  if (prop) {
    // Prop was passed, get that specific property's value.
    return grunt.util.namespace.get(config.data, config.getPropString(prop));
  } else {
    // No prop was passed, return the entire config.data object.
    return config.data;
  }
};
```

### propStringTmplRe变量

该变量其实是一个忽略大小写的正则表达式。

```javascript
// Match '<%= FOO %>' where FOO is a propString, eg. foo or foo.bar but not
// a method call like foo() or foo.bar().
var propStringTmplRe = /^<%=\s*([a-z0-9_$]+(?:\.[a-z0-9_$]+)*)\s*%>$/i;
```
正则表达式解读如下：

1. `^<%=`: 由"<%="开始

2. `\s*` : 0个或多个空格符

3. `([a-z0-9_$]+(?:\.[a-z0-9_$]+)*)` ，这部分比较负杂，由以下两部分组成：

    1. `[a-z0-9_$]+` : 1个或多个字母，数字，下划线"\_"或者美元符"$"的组合

    2. `(?:\.[a-z0-9_$]+)*` :

        1. `?:` : 匹配`\.[a-z0-9_$]+`，但是却并不捕获匹配的内容，也不给此分组分配组号。 [non-capturing parentheses](http://stackoverflow.com/questions/3512471/non-capturing-group)，这部分不影响正则匹配，但是对于用代码处理匹配的结果和元组有影响。

        2. `\.` : 点号"."

        3. `[a-z0-9_$]+` : 1个或多个字母，数字，下划线"\_"或者美元符"$"的组合

        4. `*` : 0个或多个（2）和（3）的组合

4. `\s*` : 跟着0个或多个空格符

5. `%>$` : 由"%>"结束

6. `/i` : 正则匹配大小写不敏感

综上述，该正则表达式将会匹配如下如下内容：

```javascript
> var test1 = "<%= foo %>";
> test1.match(propStringTmplRe)
[ '<%= foo %>',
  'foo',
  index: 0,
  input: '<%= foo %>' ]

> var test2 = "<%= _foo._bar._zoo9  %>";
> test2.match(propStringTmplRe)
[ '<%= _foo._bar._zoo9  %>',
  '_foo._bar._zoo9',
  index: 0,
  input: '<%= _foo._bar._zoo9  %>' ]

> var test3 = "<%= foo.bar.zoo %>";
> test3.match(propStringTmplRe)
[ '<%= foo.bar.zoo %>',
  'foo.bar.zoo',
  index: 0,
  input: '<%= foo.bar.zoo %>' ]
>

```
### config.get

通过配置属性名`prop`，获得对应配置的原始值并通过`config.process`得到处理后的配置数据。

方法仅接受一个参数`prop`，即配置的属性名称。

```javascript
// Get config data, recursively processing templates.
config.get = function(prop) {
```
首先，通过调用`config.getRaw`方法，获得`prop`配置未经处理的原始数据。然后，将原始数据作为参数通过`config.process`方法进行递归处理，返回经过处理后的配置数据。

```javascript
  return config.process(config.getRaw(prop));
};
```
### config.process

递归展开配置项的值，将从`config`对象中获得的原始值通过模板渲染的方式进行后处理。

方法接受一个参数`raw`，即配置项的原始数据。

```javascript
// Expand a config value recursively. Used for post-processing raw values
// already retrieved from the config.
config.process = function(raw) {
```
简言之，该方法通过调用`grunt.util.recurse`方法，对`raw`进行递归处理，并将结果返回。

```javascript
  return grunt.util.recurse(raw, function(value) {
```
让我们来看一下，这个递归过程中用来处理`raw`的回调函数。

如果当前遍历到的对象并不是一个"string"类型的数据，则返回当前对象`value`。

```javascript
    // If the value is not a string, return it.
    if (typeof value !== 'string') { return value; }
```
否则的话，调用[String.prototype.match][]方法判断是否当前遍历对象`value`符合`propStringTmplRe`的正则表达式，并将返回值赋值给`matches`变量。

```javascript
    // If possible, access the specified property via config.get, in case it
    // doesn't refer to a string, but instead refers to an object or array.
    var matches = value.match(propStringTmplRe);
    var result;
    if (matches) {
```
如果`matches`不为null，则说明`value`通过了正则匹配，将`matches[1]`即正则表达式中匹配的第一元组（上述正则表达式解析中的3.1部分）作为参数调用`config.get`方法，并将结果赋给`result`变量。

__通过了解上面的`config.get`方法，我们会发现由于对它的调用，我们在这个原本就是递归函数的内部又嵌套了一层递归调用。__

```javascript
      result = config.get(matches[1]);
```
接下来，对`result`变量进行判断，如果不等于`null`（既不是`undefined`也不是`null`的话），则将`result`作为结果返回。

```javascript
      // If the result retrieved from the config data wasn't null or undefined,
      // return it.
      if (result != null) { return result; }
    }
```
如果`matches`结果为null，则说明没有匹配正则表达式`propStringTmplRe`，调用`grunt.template.process`方法，把`value`作为模板并用"config.data"进行渲染，将渲染结果作为值返回。

```javascript
    // Process the string as a template.
    return grunt.template.process(value, {data: config.data});
  });
};

```
### config.set

你们也大概猜到了，这个方法是用来对配置数据进行设置的。

方法接受两个参数，一个为`prop`，另一个为`value`。

```javascript
// Set config data.
config.set = function(prop, value) {
```
首先，这里的`prop`通过`config.getPropString`方法进行了简单地处理，其结果作为对应属性的值。

然后，通过调用`grunt.util.namespace.set`方法把`value`值设置给`config.data`中的对应属性。

```javascript
  return grunt.util.namespace.set(config.data, config.getPropString(prop), value);
};
```

### config.merge

这个方法也很简单，见名知意，是一个将参数对象值合并到`config.data`的方法。

方法内部只是对`grunt.util._.merge`方法，即[_.merge][]进行了调用封装，然后将处理后的`config.data`进行返回。

```javascript
// Deep merge config data.
config.merge = function(obj) {
  grunt.util._.merge(config.data, obj);
  return config.data;
};
```
### config.init

`config.data`初始化方法。

将参数中的`obj`对象赋给`config.data`做初始值，或者当`obj`为undefined时，初始值设为`{}`。

```javascript
// Initialize config data.
config.init = function(obj) {
  grunt.verbose.write('Initializing config...').ok();
  // Initialize and return data.
  return (config.data = obj || {});
};
```
### config.requires

配置参数依赖检测方法，如果依赖/要求的配置参数微定义则抛出异常，否则返回`true`。

```javascript
// Test to see if required config params have been defined. If not, throw an
// exception (use this inside of a task).
config.requires = function() {
```
这里引用了`grunt.util.pluralize`方法，并将变量`p`指向该方法，方便该方法调用。然后，通过`grunt.util.toArray`方法将参数列表`arguments`进行数组转换，接着通过[Array.prototype.map][]方法对数组中各项进行`config.getPropString`的处理并将返回值生成新的数组赋值给`props`变量。

```javascript
  var p = grunt.util.pluralize;
  var props = grunt.util.toArray(arguments).map(config.getPropString);
```
然后，通过`p`指向的pluralize方法判断`msg`变量中是要拼接"y"还是"ies"，是单数"exists"还是"exist"，用`grunt.log.wordlist`将数组`props`用逗号分割后进行拼接。然后将拼接好的`msg`变量值输出。

```javascript
  var msg = 'Verifying propert' + p(props.length, 'y/ies') +
    ' ' + grunt.log.wordlist(props) + ' exist' + p(props.length, 's') +
    ' in config...';
  grunt.verbose.write(msg);
```
然后定义`failProps`变量，当`config.data`为`undefined`时，failProps值为`undefined`；当`config.data`不为`undefined`时，

`props`变量通过[Array.prototype.filter][]方法将所有`config.get`返回值为`null`的属性进行了过滤，然后由剩余项目组成的新数组调用[Array.prototype.map][]方法将数组中的每一项用双引号字符串进行连接，并返回新的结果数组赋值给`failProps`变量。

```javascript
  var failProps = config.data && props.filter(function(prop) {
    return config.get(prop) == null;
  }).map(function(prop) {
    return '"' + prop + '"';
  });
```
如果`config.data`不是`undefined`并且`failProps`数组为空，则通过`grunt.verbose.ok`输出顺利完成的消息，并返回`true`。

```javascript
  if (config.data && failProps.length === 0) {
    grunt.verbose.ok();
    return true;
  } else {
```

否则的话，通过`grunt.verbose.or.write`将`msg`变量信息进行输出。

并且通过`grunt.log.error()`输出错误信息。

```javascript
    grunt.verbose.or.write(msg);
    grunt.log.error().error('Unable to process task.');
```
如果`config.data`是`undefined`的，则通过`grunt.util.error`抛出异常，中断当前的`grunt`进程。否则的话，通过`grunt.util.error`抛出异常，并且通过处理`failProps`的值，将详细的配置缺失信息作为异常信息进行输出。

```javascript
    if (!config.data) {
      throw grunt.util.error('Unable to load config.');
    } else {
      throw grunt.util.error('Required config propert' +
        p(failProps.length, 'y/ies') + ' ' + failProps.join(', ') + ' missing.');
    }
  }
};
```

[String.prototype.replace]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/Replace "String.prototype.replace"
[Array.isArray]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray "Array.isArray"
[Array.prototype.join]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join "Array.prototype.join"
[Array.prototype.map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map "Array.prototype.map"
[String.prototype.match]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/string/match "String.prototype.match"
[_.merge]: http://lodash.com/docs#merge "_.merge"
[Array.prototype.filter]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter "Array.prototype.filter"
[grunt.js]: https://github.com/gruntjs/grunt/blob/master/lib/grunt.js "grunt.js"
 

