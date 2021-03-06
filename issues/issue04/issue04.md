[Webpack热更新问题解决及分析（一）](https://github.com/Jade05/jade.github.io/issues/3)

在前一篇文章最后，我们已经锁定问题——热更新过程中导致入口文件module.exports输出的对象无法被正确挂载到window上。而至于真实情况是就是没有挂载还是挂载后被冲掉（复写）了，这个就是这篇文章中需要探索的。

探索前务必要先看一下Webpack热更新的源码知道其原理。有关Webpack热更新原理在此将不再赘述，推荐两篇相关文章：
#### [Webpack HMR 原理解析](https://zhuanlan.zhihu.com/p/30669007)
#### [Webpack 热更新实现原理分析](https://zhuanlan.zhihu.com/p/30623057)

在此，我再用通俗的语言来阐述一下Webpack热更新的原理。

不得不再提一下我们的项目架构：本项目以express搭建服务端，本地开发配合webpack-dev-middleware、wepack-hot-middleware、webpack.HotModuleReplacementPlugin实现热加载更新。我们看到有webpack-dev-middlerware、webpack-hot-middleware、webpack.HotModuleReplacementPlugin各种工具，但是我们要明白一点，打包这个活儿本质上还是webpack干的，并不是加了热更新相关的插件或者中间件后，打包这项活就是插件或者中间件干了。插件和中间件的作用仅仅是在webpack编译打包核心的外围做了一些事儿（手脚）。所以在看待这几个插件的时候，我们只要抓住热更新这一个关键功能就可以了。以下这个图来自文章-[Webpack HMR 原理解析](https://zhuanlan.zhihu.com/p/30669007)

这样理解起来就简化不少，热更新的机制本质也是client-server的机制，client就在我们的浏览器端，它需要知道文件变化了并且做出一系列的动作然后把最终结果通过浏览器让我们看到，那server端的职责就很明确了，它需要监控模块文件的变化，如果变化了就要让webpack（webpack对外暴露的API）重新打包，期间还要和client保持通信，把自己这边的状况告诉client，client根据信息采取动作。

以上就是进入问题分析前的准备工作。

#### 深入实际问题分析
为了便于看清问题，实验的例子越简单越好。所以我们仍旧采用[上一篇文章](https://github.com/Jade05/jade.github.io/issues/3)中使用的例子。
入口文件 - examples/test/index.js
```javascript
const Test = {
  text: 'hello world!'
}

module.exports = Test
```
html - example/test/index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>test</title>
</head>
<body>
    Test
    <!-- <script src="../../dist/components/test.js"></script> -->
    <script src="/examples/test.js"></script>
    <script>
      console.log(test)
      console.log(test.text)
    </script>
</body>
</html>
```
然后用生产的webpack打包配置和本地开发的打包配置分别打包出test_s.js（这个是直接打到物理磁盘的，即为发布包）和test_f.js（开发包是打在内存的，但是浏览器能访问到，我们直接右键另存为即可），test_s.js能够按照预期运行，test_f.js无法拿到预期入口。

我们看下test_s.js和test_f.js文件内容，我们可以在注释中找到这么一句话『Load entry module and return exports』,没错，这就是最终打包后暴露出来的方法。看截图：
![image](https://user-images.githubusercontent.com/5029635/35479853-3eeffca0-043c-11e8-9f6a-69e908bbdad0.png)

不管是test_s.js还是test_f.js，入口都是一个叫做341的模块，这没毛病，毕竟是同样的一份代码嘛。继续看。
![image](https://user-images.githubusercontent.com/5029635/35479855-6442474c-043c-11e8-89f1-19d7d0c79b50.png)

test_s.js是生产包，摒除了热加载，本身test.js也没啥代码，所以打包后的代码也很少，一眼就能看到 341是入口，341里头_webpack_require_(342)，而342就是我们test.js的代码内容，成功找到了Test。对于_webpack_require_有疑问的同学可以查看(这篇文章)[https://github.com/ShowJoy-com/showjoy-blog/issues/39],在这里只要了解 在入口模块中调用了 _webpack_require_(342)，就会得到342这个模块返回的 module.exports，所以就能将Test找到，并且赋给module.exports，所以341作为入口文件，最终暴露出来的就是Test。

同样在test_f.js中，我们也能找到入口引用，看下图：
![image](https://user-images.githubusercontent.com/5029635/35479828-8235027c-043b-11e8-86cf-b345362ce47d.png)

341作为入口模块，341中_webpack_require_(342)，342模块就是我们的test.js,出乎意料的是341中同时引入了_webpack_require_(32)，而且module.exports上挂载的不是342是32！！！。至此问题彻底明了，热加载过程中导致入口文件module.exports输出的对象无法被正确挂载到window上。事实情况是只引入了我们的入口文件并没有对输出进行挂载（不是冲掉）挂载的是32模块。

那这是不是bug呢？是bug我就去提issue啦，看来还是太天真:)。

继续看32模块。
![image](https://user-images.githubusercontent.com/5029635/35479865-aefb5332-043c-11e8-8427-b89d62d385a3.png)
![image](https://user-images.githubusercontent.com/5029635/35479864-9772a292-043c-11e8-8d88-f2e9a785ec28.png)

如果你认真看过最前面推荐的文章——[Webpack 热更新实现原理分析](https://zhuanlan.zhihu.com/p/30623057)，并且浏览过HRM源码，就会有深刻印象。这是HRM的client模块代码，核心功能是通过EventSource，将与HRM server建立连接，进行模块更新通信。看一下代码注释/* WEBPACK VAR INFECTION */,没错，就是这段注入代码，成了真正的入口函数。

再回头看一下 webpack-develop.config.js:
```javascript
entry: {
    test: [path.resolve(__dirname, 'examples/test/index.js'), 'webpack-hot-middleware/client?path=/examples/__what&reload=true'],
  },
```
entry test的数组依次是index.js，然后是 webpack-hot-middleware。所以index.js的包引到了，但是入口确是后者 webpack-hot-middleware。

#### 问题的最终方法
webpack-develop.config.js:
```javascript
entry: {
    // test: [path.resolve(__dirname, 'examples/test/index.js'), 'webpack-hot-middleware/client?path=/examples/__what&reload=true'],

    // 更换顺序，正确书写entry
    test: ['webpack-hot-middleware/client?path=/examples/__what&reload=true'， path.resolve(__dirname, 'examples/test/index.js')],
  },
  output: {
    filename: '[name].js',
    publicPath: '/examples/',
    library: '[name]',
    libraryTarget: 'umd',
  },
```
![image](https://user-images.githubusercontent.com/5029635/35479873-db95463c-043c-11e8-81cd-a7d09ee636f5.png)

至此，该问题完美解决。

>后记：依样画葫芦的坑有时候更大呀，而且极难理出排查思路。「别的组件入口都没问题，偏偏这个组件有问题」。有种情况叫负负得正了:)。
