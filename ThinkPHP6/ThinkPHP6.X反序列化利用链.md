本系列文章将针对 **ThinkPHP** 的历史漏洞进行分析，今后爆出的所有 **ThinkPHP** 漏洞分析，也将更新于 [ThinkPHP-Vuln](https://github.com/Mochazz/ThinkPHP-Vuln) 项目上。本篇文章，将分析存在于 **ThinkPHP 6.0.X** 中的反序列化利用链。

本篇文章，将记录存在于 **ThinkPHP6.x** 中的反序列化POP链。

## 环境搭建

```bash
➜  html composer create-project --prefer-dist topthink/think=6.0.x-dev tp6x
➜  html cd tp6x
➜  tp6x ./think run
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
        return 'ThinkPHP V6.x';
    }
}
```

## 利用条件

- 有一个内容完全可控的反序列化点，例如： `unserialize(可控变量)` 

- 存在文件上传、文件名完全可控、使用了文件操作函数，例如： `file_exists('phar://恶意文件')` 

（满足以上任意一个条件即可）

## 漏洞链

在 **ThinkPHP5.x** 的POP链中，入口都是 **think\process\pipes\Windows** 类，通过该类触发任意类的 **__toString** 方法。但是 **ThinkPHP6.x** 的代码移除了 **think\process\pipes\Windows** 类，而POP链 **__toString** 之后的 **Gadget** 仍然存在，所以我们得继续寻找可以触发 **__toString** 方法的点。

这里我们找到一个可利用的 **Model** 类，其 **__destruct** 方法中调用了 **save** 方法，而 **save** 方法调用了 **updateData** 方法，我们跟进该方法看其具体实现。（下图对应文件 vendor/topthink/think-orm/src/Model.php ）

![1](/img/ThinkPHP6.X反序列化利用链/1.png)

在 **updateData** 方法中，我们发现其调用了 **checkAllowFields** 方法，而这个方法恰恰存在字符串拼接（对应下图584行）。这里，我们就可以将 **$this->table** 或 **$this->suffix** 设置成类对象，然后在拼接的时候，触发其 **__toString** 方法，接着配合原先的链就可以完成整条POP链。

![2](/img/ThinkPHP6.X反序列化利用链/2.png)

我们刚刚看的都是 **Model** 类的代码，而 **Model** 是一个抽象类，我们找到它的继承类就好了。这里我选取 **Pivot** 类，所以这条链的 **EXP** 如下（例如这里执行 `curl 127.0.0.1:8888` ）：

```php
已删除
```

![3](/img/ThinkPHP6.X反序列化利用链/3.png)

最后整理一下攻击链的流程图：

![4](/img/ThinkPHP6.X反序列化利用链/4.png)

## 参考

[thinkphp v6.0.x 反序列化利用链挖掘](https://www.anquanke.com/post/id/187393) 