# grunt源码解析之'/grunt/template.js'

源码路径：https://github.com/gruntjs/grunt/blob/master/lib/grunt/template.js

乍一看，又是template，这年头template简直是太多太多了，从[tmpl][]到[underscore.template][]到[dot][]，再到国内模板引擎中的[arttemplate][]，至于如何选择可谓仁者见仁智者见智，在jsperf上有各种的性能测试的例子，包括国内开发者提供的[引擎渲染速度竞赛](http://aui.github.io/arttemplate/test/test-speed.html)，都可以作为选择的参考，甚至有[template-engine-chooser](http://garann.github.io/template-chooser/)这样的选择器帮助你去选择，如果遇到相关template选择的问题可自行查看选择。

但是grunt中的这个template其实并没有重复造轮子，它只是使用了[lo-dash][]的template engine，然后又在模块上封装了一些常用的工具方法。

言归正传，让我们开始源码解析。

## 模块内部代码解析

在模块起始处，还是先引入了`grunt`核心模块。

```javascript
var grunt = require('../grunt');
```
紧接着，定义了`template`对象变量作为模块返回的对象。


### template变量

```javascript
// the module to be exported.
var template = module.exports = {};
```
### template.date

这里引入了[dateformat][]模块，并指向`template.date`。

```javascript
// external libs.
template.date = require('dateformat');
```
这个模块将steven levithan的[javascript date format](http://blog.stevenlevithan.com/archives/date-time-format)中提到的日期格式化函数进行了node.js的打包封装并进行了一些修改，提供如下各种日期格式化功能：

```javascript
var dateformat = require('dateformat');
    var now = new date();

    // basic usage
    dateformat(now, "dddd, mmmm ds, yyyy, h:mm:ss tt");
    // saturday, june 9th, 2007, 5:46:21 pm

    // you can use one of several named masks
    dateformat(now, "isodatetime");
    // 2007-06-09t17:46:21

    // ...or add your own
    dateformat.masks.hammertime = 'hh:mm! "can\'t touch this!"';
    dateformat(now, "hammertime");
    // 17:46! can't touch this!

    // when using the standalone dateformat function,
    // you can also provide the date as a string
    dateformat("jun 9 2007", "fulldate");
    // saturday, june 9, 2007

    // note that if you don't include the mask argument,
    // dateformat.masks.default is used
    dateformat(now);
    // sat jun 09 2007 17:46:21

    // and if you don't include the date argument,
    // the current date and time is used
    dateformat();
    // sat jun 09 2007 17:46:22

    // you can also skip the date argument (as long as your mask doesn't
    // contain any numbers), in which case the current date/time is used
    dateformat("longtime");
    // 5:46:22 pm est

    // and finally, you can convert local time to utc time. simply pass in
    // true as an additional argument (no argument skipping allowed in this case):
    dateformat(now, "longtime", true);
    // 10:46:21 pm utc

    // ...or add the prefix "utc:" or "gmt:" to your mask.
    dateformat(now, "utc:h:mm:ss tt z");
    // 10:46:21 pm utc

    // you can also get the iso 8601 week of the year:
    dateformat(now, "w");
    // 42

    // and also get the iso 8601 numeric representation of the day of the week:
    dateformat(now,"n");
    // 6
```
### template.today

通过将当前时间`new date()`作为`template.date`的参数进行封装，打造了`template.today`方法，该方法仅接受一个`format`参数，并根据不同的`format`返回指定格式的当前时间。

```javascript
// format today's date.
template.today = function(format) {
  return template.date(new date(), format);
};
```

### alldelimiters变量

作为模块内的共有变量，用来存放各种分隔符配置。

```javascript
// template delimiters.
var alldelimiters = {};
```

### template.adddelimiters

该方法接受`name`，`opener`，`closer`三个参数，用来向`alldelimiters`变量对象中添加相关的分隔符配置。

```javascript
// initialize template delimiters.
template.adddelimiters = function(name, opener, closer) {
```
首先，在`alldelimiters`对象中添加属性`name`值为空对象，并且定义变量`delimiters`指向该对象，同时为该对象添加"opener"和"closer"属性，值分别为`opener`和`closer`。

```javascript
  var delimiters = alldelimiters[name] = {};
  // used by grunt.
  delimiters.opener = opener;
  delimiters.closer = closer;
```
接下来，将`delimiters.operner`值中的每一个字符都替换为"\\\\原字符"，并将替换后的结果赋值给`a`变量。将字符串"([\\\\s\\\\s]+?)"与`delimiters.closer`值中的每一个字符替换为"\\\\原字符"的结果进行连接，将结果赋值给`b`变量。

```javascript
  // generate regexp patterns dynamically.
  var a = delimiters.opener.replace(/(.)/g, '\\$1');
  var b = '([\\s\\s]+?)' + delimiters.closer.replace(/(.)/g, '\\$1');
```
__用例子看看以上这段代码的功效：__

```javascript
> var delimiters = {opener: "<%", closer: "%>"};
> var a = delimiters.opener.replace(/(.)/g, '\\$1');
> a
'\\<\\%'
> var b = '([\\s\\s]+?)' + delimiters.closer.replace(/(.)/g, '\\$1');
> b
'([\\s\\s]+?)\\%\\>'
```
接下来，为`delimiters`变量增加`lodash`属性，属性值中包含由变量`a`和`b`组合成的三组正则表达式，分别对应`evaluate`,`interpolate`和`escape`。

```javascript
  // used by lo-dash.
  delimiters.lodash = {
    evaluate: new regexp(a + b, 'g'),
    interpolate: new regexp(a + '=' + b, 'g'),
    escape: new regexp(a + '-' + b, 'g')
  };
};
```
__调用`template.adddelimiters`来设置默认的名为"config"的配置，这里的模板分隔符使用了[underscore.template][]中用到的"<%"和"%>"__

```javascript
// the underscore default template syntax should be a pretty sane default for
// the config system.
template.adddelimiters('config', '<%', '%>');
```
关于添加完毕后的`delimiters`变量，我们可以看一下它现在的状态：

```javascript
> template.adddelimiters('config', '<%', '%>');

> alldelimiters
{ config:
   { opener: '<%',
     closer: '%>',
     lodash:
      { evaluate: /\<\%([\s\s]+?)\%\>/g,
        interpolate: /\<\%=([\s\s]+?)\%\>/g,
        escape: /\<\%-([\s\s]+?)\%\>/g 
      } 
   } 
}

```
### template.setdelimiters

这个方法仅接受一个参数`name`用来匹配`alldelimiters`中的配置项。如果当前指定的`name`配置不存在则使用默认的"config"配置。

```javascript
// set lo-dash template delimiters.
template.setdelimiters = function(name) {
  // get the appropriate delimiters.
  var delimiters = alldelimiters[name in alldelimiters ? name : 'config'];

```
获得的配置选项`delimiters`的`lodash`值将被设置到`grunt.util._.templatesettings`，这个设置[\_.templateSettings][]用来指定模板匹配时用到的各种正则表达式。

配置选项`delimiters`则作为返回值返回。

```javascript
  // tell lo-dash which delimiters to use.
  grunt.util._.templatesettings = delimiters.lodash;
  // return the delimiters.
  return delimiters;
};
```
### template.process

该方法通过调用`grunt.util._.template`将数据用`tmpl`模板进行渲染。

方法接受两个参数，`tmpl`为模板对象，`options`则是一个可能包含着`data`，`delimiter`的配置选项对象。

```javascript
// process template + data with lo-dash.
template.process = function(tmpl, options) {
```
对于`options`的判断和处理不再赘述。

```javascript
  if (!options) { options = {}; }
```

```javascript
  // set delimiters, and get a opening match character.
  var delimiters = template.setdelimiters(options.delimiters);
```


```javascript
  // clone data, initializing to config data or empty object if omitted.
  var data = object.create(options.data || grunt.config.data || {});
  // expose grunt so that grunt utilities can be accessed, but only if it
  // doesn't conflict with an existing .grunt property.
  if (!('grunt' in data)) { data.grunt = grunt; }
  // keep track of last change.
  var last = tmpl;
  try {
    // as long as tmpl contains template tags, render it and get the result,
    // otherwise just use the template string.
    while (tmpl.indexof(delimiters.opener) >= 0) {
      tmpl = grunt.util._.template(tmpl, data);
      // abort if template didn't change - nothing left to process!
      if (tmpl === last) { break; }
      last = tmpl;
    }
  } catch (e) {
    // in upgrading to lo-dash (or underscore.js 1.3.3), \n and \r in template
    // tags now causes an exception to be thrown. warn the user why this is
    // happening. https://github.com/documentcloud/underscore/issues/553
    if (string(e) === 'syntaxerror: unexpected token illegal' && /\n|\r/.test(tmpl)) {
      grunt.log.errorlns('a special character was detected in this template. ' +
        'inside template tags, the \\n and \\r special characters must be ' +
        'escaped as \\\\n and \\\\r. (grunt 0.4.0+)');
    }
    // slightly better error message.
    e.message = 'an error occurred while processing a template (' + e.message + ').';
    grunt.warn(e, grunt.fail.code.template_error);
  }
  // normalize linefeeds and return.
  return grunt.util.normalizelf(tmpl);
};
```

[tmpl]: https://github.com/borismoore/jquery-tmpl "tmpl"
[dot]: https://github.com/olado/dot "dot"
[_.template]: http://lodash.com/docs/#template "_.template"
[lo-dash]: http://lodash.com/ "lo-dash"
[dateformat]: https://github.com/felixge/node-dateformat "dateformat"
[artTemplate]: https://github.com/aui/artTemplate "artTemplate"
[_.templateSettings]: http://lodash.com/docs/#templateSettings "_.templateSettings"
