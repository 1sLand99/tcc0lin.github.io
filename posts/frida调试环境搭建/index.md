# Frida调试环境搭建


## 一、背景
从上手难度上来说frida可以说是开箱即用，几乎没有学习成本，对于大多数使用者来说，任何一个App都可以直接使用一个js脚本搞定。但是当面对繁多的App，如何将这种分析工具进行模块化拆分是需要好好思考思考的，直接使用js来做不太现实，而解决方案是ts&#43;npm，优势在于
- 静态类型系统：提供静态类型检查，编译时即可发现类型错误，提升代码健壮性和可维护性
- 模块化管理：通过类型系统实现模块间的契约式开发，适用于大型项目管理
## 二、frida ts项目配置
frida接入ts开发可以参考[frida-agent-example](https://github.com/oleavr/frida-agent-example)，使用方式也很简单，参考
```shell
$ git clone git://github.com/oleavr/frida-agent-example.git
$ cd frida-agent-example/
$ npm install
$ frida -U -f com.example.android --no-pause -l _agent.js
```
实时编译可以直接调用`npm run watch`，如下
![](https://github.com/tcc0lin/picx-images-hosting/raw/master/WX20250221-190405@2x.7zqk9ymbhn.webp)
原理可以看`package.json`配置文件
```json
{
  &#34;name&#34;: &#34;frida-agent-example&#34;,
  &#34;version&#34;: &#34;1.0.0&#34;,
  &#34;description&#34;: &#34;Example Frida agent written in TypeScript&#34;,
  &#34;private&#34;: true,
  &#34;main&#34;: &#34;agent/index.ts&#34;,
  &#34;scripts&#34;: {
    &#34;prepare&#34;: &#34;npm run build&#34;,
    &#34;build&#34;: &#34;frida-compile agent/index.ts -o _agent.js -c&#34;,
    &#34;watch&#34;: &#34;frida-compile agent/index.ts -o _agent.js -w&#34;
  },
  &#34;devDependencies&#34;: {
    &#34;@types/frida-gum&#34;: &#34;^18.5.1&#34;,
    &#34;@types/node&#34;: &#34;^18.19.3&#34;,
    &#34;frida-compile&#34;: &#34;^16.4.1&#34;
  }
}
```
调用的是`frida-compile agent/index.ts -o _agent.js -w`来编译

修改index.ts代码可以动态编译
![](https://github.com/tcc0lin/picx-images-hosting/raw/master/WX20250221-190714@2x.2ks1rj7649.webp)

frida ts环境配置完成后需要主动暴露端口接入chrome调试，如下
```ts
frida -U -f com.tencent.wework -l _agent.js --debug --runtime=v8
```
![](https://github.com/tcc0lin/picx-images-hosting/raw/master/WX20250221-190512@2x.6wquz2qhm6.webp)
默认端口为9229

## 三、chrome调试环境配置
chrome打开标签`chrome://inspect`，点击`Open dedicated DevTools for Node`来新打开`DevTools`
![](https://github.com/tcc0lin/picx-images-hosting/raw/master/WX20250221-190548@2x.9dd3dzxdip.webp)
进入`DevTools`后切换到Source可以看到该端口对应的文件
![](https://github.com/tcc0lin/picx-images-hosting/raw/master/WX20250221-190620@2x.1ap4l7p6t0.webp)
单步调试
![](https://github.com/tcc0lin/picx-images-hosting/raw/master/WX20250221-190811@2x.8vn1pevzy5.webp)

---

> 作者: tcc0lin  
> URL: http://localhost:1313/posts/frida%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/  

