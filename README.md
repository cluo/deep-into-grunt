# Grunt源码解析

## 初衷及方向

创建这个repo的目的一开始很简单，就是想要通过阅读分析GRUNT的源代码来加深对GRUNT的理解。

但是，写着写着从简单分析源码衍生出了与其他看到的优秀代码比较或者总结，以得到某些最佳实践的想法，所以在写的过程中，有时候会穿插一些有的没的，看着相关其实又不太相关的内容，这些内容更多是结合一些工作学习中见到的javascript或者Node.js的最佳实践或者对我有启发的点。

## 目录结构

因为grunt源码本身有些文件是重名的，但是存在与不同的目录中，为了避免不必要的误解和冲突，这里沿用源码的目录结构，而且文件名只是将`js`替换为`md`，这样读起来也许会更方便。

* grunt
    *   file.js [[done](https://github.com/qivhou/deep-into-grunt/blob/master/grunt/file.md)]
    *   option.js [[done](https://github.com/qivhou/deep-into-grunt/blob/master/grunt/file.md)]
    *   cli.js
    *   config.js
    *   event.js
    *   fail.js
    *   help.js
    *   log.js 
    *   task.js
    *   template.js
    *   grunt.js
*   util
    *   task.js
*    grunt.js


## 阶段说明

整个repo可能会分为以下几个阶段（随着过程的进行适时调整）：

+ 对Grunt源码进行解析

+ 在源码解析的基础上，将各模块关系及Grunt工作原理进行整理

+ 对某几个流行的grunt插件进行分析？

+ 在梳理出工作原理及模块关系的基础上，与当前流行的其他任务管理工具进行比较，比如[gulp](http://gulpjs.com)？


## 其他

作为一个非互联网界的码农，非专业前端码农，非专业javascript码农，非Node.js码农，在这个过程中如果出现各种的专业错误或者理解错误，还望大家悉心指教。

因为是第一次就整个成熟的产品进行分析，所以在写的过程中或多或少会有各种纰漏，而且现在这种line by line的方式也只是一种尝试，可能有的时候看着会很乱，但是我会在之后的过程中进行不断的完善。

__另：目前已经有一批Grunt爱好者们集合在了一起，讨论Grunt，讨论前端，讨论JS，期待着你的加入。QQ群[Grunt同学会] : 16613475__

