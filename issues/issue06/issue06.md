## [Draft] AMP项目实战分享

### 大纲
1. AMP是什么
2. 我们为什么选择AMP
3. AMP项目开发及部署
4. 我们踩过的"坑"儿
5. 如何优化AMP SEO页面
6. 项目收益
7. eamp框架，让开发AMP项目更简单

### AMP是什么

Google 前沿的 AMP 「 Accelerated Mobile Pages 」技术，使用户从搜索引擎当中进入我们页面的体验得到了一个极大的提升。确切地说，AMP并不是一门新的技术，它只是让页面打开得更快的一种解决方案。你只要会HTML,CSS，略懂JS，就可以开发你自己的AMP页面。

AMP更多内容可以参看（官网）[https://www.ampproject.org/zh_cn/learn/overview/]。

### 我们为什么选择AMP

看看AMP能给我们带来什么：
1. AMP能够带来SEO排名优化。
2. AMP Cache能够让我们充分借助Google CDN cache的优势。虽然我们内部已经做了很多优化，包括DNS预热（有关DNS预热可参见我们组同事写的一篇文章（网站性能优化——DNS预热与合并HTTP请求）[https://zhuanlan.zhihu.com/p/32168340]），但如果能有Google全球CDN也是件好事。
3. Google搜索结果对AMP页面的预加载，能让用户更快地到达我们的着陆页。

所以，作为携程的海外业务部门，我们率先试水了AMP项目。

### AMP页面开发及部署

为了最大限度提升性能，AMP页面有不少要求。比如：
1. 为了避免 JavaScript 延缓页面渲染，AMP页面不能包含自己编写的avaScript。
2. CSS都必须内联，以减少HTTP的请求，并且有50KB的大小限制。
3. AMP 页面允许第三方 JavaScript，但仅限在沙盒环境下的 iframe 中。
4. 用户几乎可以使用所有原生的HTML标签，但是对img等会产生外部资源依赖的标签，只能使用amp-img自定义标签。

所以，基于以上几点，页面上所有交互逻辑都必须纯css实现，而且

当开发完成后，必须保证页面是符合AMP规范的，只有符合AMP规范的页面才会被搜索引擎收录。在chrome中安装AMP Validate插件，当你的页面是完全符合AMP规范的时候，chrome valid AMP按钮会呈现绿色。如下图：


### 我们踩过的坑儿

AMP有不少限制要求，开发中难免碰到不好解决的问题。以下对我们碰到过的问题及解决方法进行分享。

1. AMP对表达式复杂度有限制要求，最大值是50。复杂度50是什么概念呢，比如以下例子，只有纯粹的字符串连接，但是以下复杂度已经接近是46，如果再增加一个连接，会达到53，的复杂度，控制台会直接报错：
```HTML
<a href='<%- searchData.defaultSearchUrl %>'
  [href]='"/m/hotels/" + location.concatResult
    + "/?checkin=" + date.selectedStartDate
    + "&checkout=" + date.selectedEndDate
    + "&starID=" + activeStarKeys.keys
    + "&adult=" + guestsSelectResult.adultsNum
    + "&children=" + guestsSelectResult.childNum
    + "&ages=" + guestsSelectResult.childAges'>
```
同样一次运算里头三元表达式也要谨慎使用，因为复杂度很高。
解决方法：除非AMP放开复杂度，我们开发的时候能够做的就是能提前运算的尽量提前运算，可以直接使用结果，同时尽量避免复杂的交互。

2. AMP限制编写JavaScript，也就不允许我们读写cookie、localstorage，但是记录用户行为是很重要的事情，比如一些表单信息的填写，用户选择，我们需要记住内容，等用户下次打开我们的页面时，能够显示用户上一次的选择。我们的解决方法是通过http set-cookie的方式解决前端无法记录cookie的问题。

3. AMP form只能提交ajax post请求，无法做到以post表单形式跳转
通过增加非amp的中间路由，做二次跳转后再提交数据。

4. AMP CROS，用户最终访问的是AMP Cache，在AMP launch新版本之前，命中AMP Cache，页面地址并非是真实地址而是google amp cache地址，如果页面上有额外的异步请求数据，就会有跨域限制，所以我们要在服务端开启跨域，返回头设置AMP-Access-Control-Allow-Source-Origin。

yzh ck补充。

### 如何优化AMP SEO页面

#### 最大化提速

通过对非SEO数据异步化加载，让AMP更快。

#### amp-install-serviceworker，让你打开的不仅仅是一个amp页面

AMP SEO页面作为搜索排名优化页面的同时，还兼具引流功能。虽然AMP页面能让用户从搜索结果中最快速地到达我们的landing页面，但是只有用户最终从landing页又回到原始页面走完必要的业务流程，才是真正意义上的转换。例如，在我们的业务中，用户可以通过搜索引擎快速到达酒店列表、酒店详情的AMP页面，但只有从酒店详情AMP页面跳转到支付页（非AMP），并完成支付，才算转换成功。如果原始页面体面不好，用户依旧可能中途流失。这似乎不是AMP的"错"，但AMP确实能做些什么。

通过amp-install-serviceworker安装原始站点的sw.js，提前加载好原始页面的资源，当用户从AMP页面跳出，进入原始主站的时候，让主站体验更好，从而提高实际意义上的转换率。

### 项目收益

#### seo页面和原始页面在速度上的对比
以首页为例（因其他页面在seo项目中精简过需求，所以无法只管看出对比性），但首页功能是一样的，所以完全可以对比。

等AMP页面收录后，用户访问的实际上是AMP cache，也就是当我们的AMP页面出现在Google搜索结果中的时候，Google会预加载AMP页面，当用户点击搜索结果的链接到达我们的页面，时间耗费几乎为0s。

#### 给原始站点带来的PV UV

此数据有待补充，如果允许，后续补上。

### eamp框架，让开发AMP项目更简单

(eamp)[https://github.com/Jade05/eamp]
1.快速搭建基于node的AMP项目
2.内置自动化上线打包工具，自动将css打包内联，符合AMP规范
3.目录结构清晰，职责划分清楚，便于多人团队开发

eamp是从我们AMP SEO项目中提取出来的简化版框架，能够让我们快速开启AMP Node项目，使用者无需从0开始搭建，更能专注amp页面开发。

### 推荐资料
#### [AMP官网](https://www.ampproject.org/)
#### [ampproject github](https://github.com/ampproject/amphtml)