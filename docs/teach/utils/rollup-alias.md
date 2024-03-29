最近在用 uniapp 开发小程序, 因为封装强似组件库的组件, 所以我脑洞大开想为这个组件库配置一个自定义别名, 以方便导入, 更彰显我对这个组件库的宠爱。 
这是我想达到的效果

![图 1](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/55f755bb66d33494c94bb94e7ec4ac9c4281d90b204d920743589a2b16840599.png)  



这是我的配置项
```ts
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
      '@components/': resolve(__dirname, './src/components/hex-ui'),
    },
  },
```

在我配置之后, 我肆无忌惮的开始修改组件内部的路径, 然而一切都大失所望, 就好像一巴掌打在自己的脸上。我讨厌红色。

![图 2](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/abc7b5a9b55d99897bbfc7ef2da80ea38a038a56bb2542a76c1c16498d731c6e.png)  


随后，我开始疯狂 google，对于 baidu 我深恶痛绝，结果搜到都是以下内容

![图 3](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/4b359ff72aa1443dc16cbb4ab7d2f68285c15e5c9def7a2510151ea7a3e76512.png)  


我随便点开一个

![图 4](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/c34a97556978cb2b89a9d772ffecd645eee898a22f6c4258bbc3ce6fcf618642.png)  


这都是神魔？ google 你是不是收了 csdn 的 money。 fuck, 这跟 vite 有关系吗? 这tmd 是 ts 的 路径联想没配置好。在寻找答案的过程中，我发现好多人对于 ts alias 和 vite alias 没有弄清楚。 其实两者是没有关系的, ts 的 alias 只对里的编辑器的提示有效, 而 vite alias 是对编译生效的, 就好像这次我没有配对一直报路径错误，反而ts可以正常提示。废话不多说继续寻找答案！ [对于没搞懂两者关系的可以看看这篇, 来源思否](滑动验证页面)

这个时候, 我已经不相信搜索引擎了。就在我一筹莫展的时候，我莫名其妙的去修改了一下 alias 的 key。成功了，卧槽，竟然成功了。事情并没有到此结束，本着程序员的好奇心，我想搞明白这个别名插件的处理逻辑。首先我想到这个 diao 插件是不是给我识别成正则了,导致我的路径没有匹配到。
```ts
'@components/': resolve(__dirname, './src/components/hex-ui'),
// 修改后
'@components': resolve(__dirname, './src/components/hex-ui'),
```

我开始问 chatgpt, 这个 alias 插件的源码在哪, 果不其然, 它又一次欺骗了我。vite 里面根本没有对应的文件, 倒是 plugin 里面有一个 resolver 文件.


![图 5](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/e3c20facff86b385d1cb95f2bc45825e434c9f51cafc52614cb4e09a4967f7ab.png)  


我继续问它, 还好这次没有失望.


![图 6](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/8d7db3001866ea131fcc144c2f66516343063588b00f7b3e35c359ce660f4177.png)  



于是我开始码了, 因为 vite 是使用了rollup生态, so 我直接使用 rollup @rollup/plugin-alias 开始了我的征程。

首先按照 chatgpt 流程初始化一个 rollup 项目，如下所示，这就是 @rollup/plugin-alias 的源码了。这里因为在 page.josn 文件设置了 ` "type": "module",` 所以选择了符合 esmodule 的文件。


![图 7](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/5fff061b172d02c7883387475b8a24b9a988b1387972ed73048ffd48f480b147.png)  


文件提取出来之后，就可以作为插件使用了。其实一个插件的本质就是一个函数，它接受来自 rollup 给它传递的参数，经过它自己的处理再返回出去。

打个比方：就好像有一堆土，rollp 把它处理成了一块黏土，而插件就好像一道道工序，a 工序把它打造成杯子的形状然后传给 b 工序，b 工序给它上了一层釉，传递给 c，c工序烧制成型，返回 roullp 这个工厂做出了成品。

![图 8](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/1482f6d75aa90fa0b44c8191acda4338868280afe8d836c373c5e1a07cc135cf.png)  



根绝插件的入口，我们很轻松的找到了匹配函数

![图 9](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/fe2f345cc8687318e85845d55a4080e6adb899096f287092cad23c3d9cfe2b5d.png)  


```js
function matches(pattern, importee) {
    if (pattern instanceof RegExp) {
        return pattern.test(importee);
    }
    if (importee.length < pattern.length) {
        return false;
    }
    if (importee === pattern) {
        return true;
    }
    // eslint-disable-next-line prefer-template
    return importee.startsWith(pattern + '/');
}
```

可以看到这个匹配函数接收两个参数，一个是匹配规则，一个是路径。这里对匹配规则做了判断，匹配规则可以使正则也可以是字符串，再次也验证了前面的猜想。有可能是正则的问题，在这里我们开始寻找答案。

在这里我们进行了 debug，错误很明显，原来插件会自动拼接一个斜杠这时候参数就变成 "@mylib//",
那么 `"@mylib/module.js".startsWith("@mylib//")` 自然而然就是false了。

![图 10](https://cdn.jsdelivr.net/gh/Journey98/A-week-to-learn@assert/image/cbb1ab3f3e5ebd83fd813008af8e4abe949a01b98244256c81fc7f16c29c110a.png)  


就此真相大白了。一个小小的问题引发的一次探索。

[代码地址](https://github.com/Journey98/roullp-alias)