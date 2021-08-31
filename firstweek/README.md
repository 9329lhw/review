<h1 align="center">php基础与框架</h1>

## 命名空间
    命名空间是一种封装事物的方法
    命名空间用来解决在编写类库或应用程序时创建可重用的代码如类或函数
    时碰到的两类问题：
    1.用户编写的代码与PHP内部的类/函数/常量或第三方类/函数/常量之间的
    名字冲突。
    2.为很长的标识符名称(通常是为了缓解第一类问题而定义的)创建一个别名（
    或简短）的名称，提高源代码的可读性。
## 面向对象
### 面向对象的三大要素：
    封装
    继承
    多态
    

## 自动加载原理
### PHP 自动加载功能的由来
    在 PHP 开发过程中，如果希望从外部引入一个 class，通常会使用 include 
    和 require 方法，去把定义这个 class 的文件包含进来。这个在小规模开
    发的时候，没什么大问题。但在大型的开发项目中，使用这种方式会带来一
    些隐含的问题：如果一个 PHP 文件需要使用很多其它类，那么就需要很多的 
    require/include 语句，这样有可能会造成遗漏或者包含进不必要的类文件。
    如果大量的文件都需要使用其它的类，那么要保证每个文件都包含正确的类
    文件肯定是一个噩梦， 况且 require_once 的代价很大。
    PHP5 为这个问题提供了一个解决方案，这就是类的自动装载 (autoload) 机制。 
    autoload 机制可以使得 PHP 程序有可能在使用类时才自动包含类文件，而不是
    一开始就将所有的类文件 include 进来，这种机制也称为 lazy loading。
    总结起来，自动加载功能带来了几处优点：
    1.使用类之前无需 include 或者 require。
    2.使用类的时候才会 require/include 文件，
    实现了 lazy loading，避免了 require/include 多余文件。
    3.无需考虑引入类的实际磁盘地址，实现了逻辑和实体文件的分离。
    
    参考链接
    https://learnku.com/articles/4681/analysis-of-the-principle-of-php
    -automatic-loading-function
    
    
    https://segmentfault.com/a/1190000014948542
## [依赖注入，控制反转](/firstweek/DI&IOC.md)
## 容器
## 框架
### laravel框架
#### laravel框架生命周期
#### laravel路由原理
#### laravel中间件
#### laravel队列
#### laravel广播
#### laravel任务调度



	
