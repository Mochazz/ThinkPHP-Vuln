本系列文章将针对 **ThinkPHP** 的历史漏洞进行分析，今后爆出的所有 **ThinkPHP** 漏洞分析，也将更新于 [ThinkPHP-Vuln](https://github.com/Mochazz/ThinkPHP-Vuln) 项目上。本篇文章，将分析存在于 **ThinkPHP 5.2.X** 中的反序列化利用链。

## 环境搭建

```bash
➜ html composer create-project topthink/think=5.2.x-dev tp52x
➜ html cd tp52x
➜ tp52x ./think run
```

将 **application/index/controller/Index.php** 代码修改成如下：

```php
<?php
namespace app\controller;

class Index
{
    public function index()
    {
        $u = unserialize($_GET['c']);
        return 'ThinkPHP V5.2';
    }
}
```

## 利用条件

- 有一个内容完全可控的反序列化点，例如： `unserialize(可控变量)` 

- 存在文件上传、文件名完全可控、使用了文件操作函数，例如： `file_exists('phar://恶意文件')` 

（满足以上任意一个条件即可）

## POP链1

这个漏洞链个人认为比较有意思的是：通过 **file_exists** 函数触发类的 **__toString** 方法。下面，我们具体分析一下整个漏洞攻击链。

在 **think\process\pipes\Windows** 类的 **__destruct** 方法中，存在一个删除文件功能，而这里的文件名 **$filename** 变量是可控。如果我们将一个类赋值给 **$filename** 变量，那么在 **file_exists($filename)** 的时候，就会触发这个类的 **__toString** 方法。因为 **file_exists** 函数需要的是一个字符串类型的参数，如果传入一个对象，就会先调用该类 **__toString** 方法，获取其字符串返回值，然后再判断。（下图对应文件 vendor/topthink/framework/src/think/process/pipes/Windows.php）

![1](/img/ThinkPHP5.2.X反序列化利用链/1.png)

接下来，我们就来寻找可利用的 **__toString** 方法。全局搜索到的 **__toString** 方法其实不多，这里有两处都可以利用。它们的区别在于利用 **think\Collection** 构造的链要多构造一步，我们这里只分析链较短的 **think\model\concern\Conversion** 。

![2](/img/ThinkPHP5.2.X反序列化利用链/2.png)

如下图所示，原先 **ThinkPHP5.1.x** 中 **$relation->visible($name)** 已经不见了，其实这段代码被移到了 **第180行** 的 **appendAttrToArray** 方法中。这里，我们先关注 **第172行** 的 **getAttr** 方法，这里传入的 **$key** 变量是来自第158行的可控变量 **$data** 。（下图对应文件 vendor/topthink/framework/src/think/model/concern/Conversion.php）

![3](/img/ThinkPHP5.2.X反序列化利用链/3.png)

在 **getAttr** 方法中，程序先通过第451行的 **getData** 获取了 **$value** 变量。从下图右侧的获取过程中，可以看出最终获得的 **$value** 变量值可控。然后在第457调用 **getValue** 方法，传入该方法的前两个变量值均可控，最后一个 **$relation** 值为 **false** 。我们跟进 **getValue** 方法，看其具体代码。（下图对应文件 vendor/topthink/framework/src/think/model/concern/Attribute.php ）

![4](/img/ThinkPHP5.2.X反序列化利用链/4.png)

可以看到在 **getValue** 方法中，使用了动态调用（上图第481行），而且这里的 **$closure、$value、$this->data** 均可控。我们只要让 **$closure='system'** 并且 **$value='要执行的命令'** ，就可以触发命令执行。但是上面的 **Attribute、Conversion** 是 **trait** ，不能直接用来构造 **EXP** ，我们得找使用了这两个 **trait** 的类。

![5](/img/ThinkPHP5.2.X反序列化利用链/5.png)

这里我们找到了符合条件的 **Pivot** 类，所以这条链的 **EXP** 如下（例如这里执行 `curl 127.0.0.1:8888` ）：

```php
已删除
```

![6](/img/ThinkPHP5.2.X反序列化利用链/6.png)

## POP链2

第二条POP链其实不太好用，需要目标站点可以上传 **route.php** 文件，且知道上传后文件的存储路径，下面我们来看下具体POP链。

这个POP链的前半部分，和原先 **ThinkPHP5.1.x** 中的POP链是一样的。只不过在执行到下图第193行时， **ThinkPHP5.1.x** 中的POP链会去触发 **Request** 类的 **__call** 方法，而在 **ThinkPHP5.2.x** 中移除了 **Request** 类的 **__call** 方法，所以我们需要寻找新的可用 **__call** 方法。

![7](/img/ThinkPHP5.2.X反序列化利用链/7.png)

这里，我们使用 **Db** 类的 **__call** 方法，因为该方法可以实例化任意类（下图第203行）。结合 **Url** 类的 **__construct** 方法，从而进行文件包含。如果攻击者可以上传 **route.php** 文件，并知道文件存储位置，即可 **getshell** 。

![8](/img/ThinkPHP5.2.X反序列化利用链/8.png)

最终，这条链的 **EXP** 如下（这里我事先上传了 **route.php** 到 **/tmp/** 目录下）：

```php
已删除
```

![9](/img/ThinkPHP5.2.X反序列化利用链/9.png)

## 参考

[N1CTF2019 sql_manage出题笔记](https://xz.aliyun.com/t/6300) 
[thinkphp v5.2.x 反序列化利用链挖掘](https://www.anquanke.com/post/id/187332) 