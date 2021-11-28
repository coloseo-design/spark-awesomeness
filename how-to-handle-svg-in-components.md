## <center>`组件开发中如何处理svg？`<center>

<br>

>在组件库开发设计的过程中，图标库是随着业务场景的不断丰富在不断丰富的，如何能够最小成本的持续更新图标库是在组件设计之初就应该考虑的问题。

&emsp;&emsp;在前端开发中，独立的功能我们都会将其封装在模块中，然后发布到包管理服务器，保证功能的正常迭代。



&emsp;&emsp;因此，我们也我们首先想到的是将svg通过包的方式管理起来。



&emsp;&emsp;目前我们前端开发组件库有两个: React和Vue，这是两个不同的框架，构建组件的方式也一样，如果我们将svg图标看成一个一个组件的话，我们需要将svg封装成与之对应的框架组件包发布，但是这样又引入了新问题，每次有新的图标更新的时候，我们都需要同时对两个包进行维护和版本管理。



&emsp;&emsp;图标库和框架耦合，这与我们设计的初衷是不符合的，我们希望svg图标可以独立于框架，而且在框架中又可以很方便的使用。



&emsp;&emsp;或许我们可以换个方向。

    

&emsp;&emsp;我们都知道SVG是xml用来标记的一种信息，我们前端包之间能互相使用的只有js和json，我们能否不使用xml而采用一种能够在前端项目中更加通用的方式来标记这些信息呢？没错，我们想到一块儿去了，json非常适合，我们可以尝试将svg标记为json格式。

&emsp;&emsp;假如有如下的svg图标：
```svg
<svg viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg">
  <path d="M152 464H872C894.091 464 912 481.909 912 504V520C912 542.091 894.091 560 872 560H152C129.909 560 112 542.091 112 520V504C112 481.909 129.909 464 152 464Z"/>
  <path d="M560 152V872C560 894.091 542.091 912 520 912H504C481.909 912 464 894.091 464 872V152C464 129.909 481.909 112 504 112H520C542.091 112 560 129.909 560 152Z"/>
</svg>
```
&emsp;&emsp;可以通过json描述为：

```json
{
  "tag": "svg",
  "attrs": {
    "viewBox": "0 0 1024 1024"
  },
  "children": [{
    "tag": "path",
    "attrs": {
      "d": "M152 464h720a40 40 0 0140 40v16a40 40 0 01-40 40H152a40 40 0 01-40-40v-16a40 40 0 0140-40z"
    }
  }, {
    "tag": "path",
    "attrs": {
      "d": "M560 152v720a40 40 0 01-40 40h-16a40 40 0 01-40-40V152a40 40 0 0140-40h16a40 40 0 0140 40z"
    }
  }]
}
```

&emsp;&emsp;这些json作为通用图标的内容进行发布，各种框架再将这些json转换为组件。
```
[xml] --- [json] |----[React.createElement()/h()]
```
 &emsp;&emsp;所以我们接下来的工作需要将xml转换为json：

 ```
 [xml]----[src/xxx.js]----[es/lib] -> 包服务器
 ```

1. 将xml转为包含json描述xml的js文件
2. 将生成的js通过编译成通用的es/lib
3. 发布到包管理服务器

&emsp;&emsp;谈到前端对文件/文件夹进行处理，不得不提gulp这个工具，平时大家都有接触到比较现代的webpack/esbuild/grunt/rollup将项目文件打包成的经验，但是这些工具绝大多数都是对单入口项目文件使用体验比较好(`大家回忆下webpack里面配置entry的时候，是否几乎都是些d的scr/index.ts`),  但是对于我们目前这种批量将文件进行处理，gulp就会更加得心应手。

&emsp;&emsp;gulp工作流的方式比较简单：

```javascript
src(你的源文件) -> pipe(处理流程plugin1) -> pipe(处理流程plugin2)-> .... -> dest(你的目标文件)
```

&emsp;&emsp;我们可以将gulp看作是工厂的生产流水线，而plugin就相当于对于流水线上的一些操作，然后将这些操作串起来加工成我们需要的东西。

&emsp;&emsp;gulp将加工材料src进流水线，然后通过pipe将一个一个plugin串起来，然后将加工后的成品dest到目的地。

&emsp;&emsp;我们简单来描述一下我们目前转换svg的工作流:
```javascript
src(目标文件夹/*.svg).pipe(svg转换成json).dest(目标文件夹)
```
 
&emsp;&emsp;转换成gulp代码

 ```javascript
 const { src, dest, series } = require('gulp');
gulp.src('src/svgs/*.svg')
  .pipe(todoTransfer)
  .dest('src/js')
 ```

&emsp;&emsp;我们现在需要处理如何把svg转换为json格式，我们当然可以手写，但是借助工具可以让我们更快的完成任务，我们通过在npm搜索“parse xml”搜索看到

```javascript
// @rgrove/parse-xml这个包可以很方便的将xml转换为json
// 官方示例
const parseXml = require('@rgrove/parse-xml');
let doc = parseXml('<kittens fuzzy="yes">I like fuzzy kittens.</kittens>');
```

&emsp;&emsp;我们现在可以实现我们的todoTransfer了，但是在这之前，我们需要先处理一下svg，我们打开设计交付给我们的svg文件，我们发现里面有很多冗余信息，比如里面包含一些譬如id, class, width, height 这些信息我们都不需要，所以我们要将这些无用的信息处理掉，这样可以让我们生成的json更精简。

&emsp;&emsp;所以我们的流程变成了：

```javascript
const { src, dest, series } = require('gulp');
gulp.src('src/svgs/*.svg')
  .pipe(移除冗余信息)
  .pipe(将内容转换成json)
  .pipe(将文件格式转换为js)
  .dest('src/js')
```

&emsp;&emsp;当然去除冗余信息也有工具可以用，我们找到了这个：
```
svgo
```
&emsp;&emsp;这个库可以将为我们去掉svg中冗余信息并且压缩。

&emsp;&emsp;通过上面的处理，我们将svg通过转换成json标记的内容，这样可以在任何前端框架中去转换使用。


> 文中提到的库
- gulp
- svgo
- @rgrove/parse-xml
