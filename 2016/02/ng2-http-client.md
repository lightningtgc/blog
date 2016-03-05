# HTTP 客户端

> 本文讲述一个用 Angular Http 客户端的远程服务

[Http](https://tools.ietf.org/html/rfc2616)是浏览器/服务器通信最主要的协议。

> [WebSocket](https://tools.ietf.org/html/rfc6455) 协议是另一个重要的通信技术；但在这一章中我们不会讲到它。

现代浏览器支持两种基于HTTP的API：[XMLHttpRequest (XHR)](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) 和 [JSONP](https://en.wikipedia.org/wiki/JSONP)。少数浏览器也支持[Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)。

Angular HTTP 客户端库简化了我们接下来将学到的`XHR`和`JSONP` API们的程序编码：
* Http 客户端概述
* 使用 http.get 获取数据
* HTTP 响应中得 RxJS Observable
* 使用 RxJS map 提取JSON数据
* 错误处理
* 打印结果日志
* 发送数据到服务器
* 添加 Headers
* 使用 Promises 代替 observables
* JSONP
* 设置查询字符串参数
* 搜索项输入防抖
* 附录：内存 web api 服务

我们通过代码来阐述这些主题，你可以[在浏览器中运行这些例子](https://angular.io/resources/live-examples/server-communication/ts/plnkr.html)。

### Http 客户端示例

我们使用 Angular Http 客户端通过`XMLHttpRequest(XHR)`去通信。

我们将展示一个迷你版本的 “Tour of Heroes” (ToH)应用程序的[教程](https://angular.io/docs/ts/latest/tutorial/)。这个版本会从服务器获取一些`hero`，将它们展示在一个列表中，并让我们新增 `hero` 还有将它们保存到服务器中。

程序运行效果如下：

![http-toh](https://raw.githubusercontent.com/lightningtgc/blog/master/2016/02/assets/http-toh.gif)

这是由两个组件实现的 - 一个父组件 `TohComponent`和子组件 `HeroListComponent`。我们已经从许多其他文档例子中知道这些类型的组件。那让我们看看它们是怎么变得可以支持与服务器通信的吧。

> 我们创建了两个只针对一个小例子的组件，有点将“关系分离”做得过头了。我们提出一个关于当程序发展时更容易调整的程序结构的观点。

这是父组件`TohComponent shell`：

app/toh.component.ts
```ts
import {Component}         from 'angular2/core';
import {HTTP_PROVIDERS}    from 'angular2/http';
import {Hero}              from './hero';
import {HeroListComponent} from './hero-list.component';
import {HeroService}       from './hero.service';
@Component({
  selector: 'my-toh',
  template: `
  <h1>Tour of Heroes</h1>
  <hero-list></hero-list>
  `,
  directives:[HeroListComponent],
  providers: [
    HTTP_PROVIDERS,
    HeroService,
  ]
})
export class TohComponent { }
```
通常我们会导入自己需要的内容。这次新导入的是`HTTP_PROVIDERS`,它是一个从 Angular HTTP库中引入的服务供给的数组。我们将使用那个库来进入服务器。我们也导入一个`HeroService`,待会会提到它。

组件指定了`HTTP_PROVIDERS`和`HeroService`到元数据供给的数组中，这使得它们能够成为这个“Tour of Heroes”程序的子组件。

> 学习关于`providers`的内容请参看[依赖注入](https://github.com/gf-rd/blog/issues/12)的章节。

这个例子只有一个子组件，`HeroListComponent`全部展示在这里：

app/toh/hero-list.component.ts
```ts 
import {Component, OnInit} from 'angular2/core';
import {Hero}              from './hero';
import {HeroService}       from './hero.service';
@Component({
  selector: 'hero-list',
  template: `
  <h3>Heroes:</h3>
  <ul>
    <li *ngFor="#hero of heroes">
      {{ hero.name }}
    </li>
  </ul>
  New Hero:
  <input #newHero />
  <button (click)="addHero(newHero.value); newHero.value=''">
    Add Hero
  </button>
  <div class="error" *ngIf="errorMessage">{{errorMessage}}</div>
  `,
  styles: ['.error {color:red;}']
})
export class HeroListComponent implements OnInit {
  constructor (private _heroService: HeroService) {}
  errorMessage: string;
  heroes:Hero[];
  ngOnInit() { this.getHeroes(); }
  getHeroes() {
    this._heroService.getHeroes()
                     .subscribe(
                       heroes => this.heroes = heroes,
                       error =>  this.errorMessage = <any>error);
  }
  addHero (name: string) {
    if (!name) {return;}
    this._heroService.addHero(name)
                     .subscribe(
                       hero  => this.heroes.push(hero),
                       error =>  this.errorMessage = <any>error);
  }
}
```

组件模板展示了一个用到`NgFor`重复功能的指令的heroes列表。

在heroes列表下方是一个输入框和一个添加hero的按钮可供我们输入新hero的名字并把加到数据库的。我们使用一个[本地模板变量](https://angular.io/docs/ts/latest/guide/template-syntax.html#local-vars)`newHero`通过(click)事件绑定来获得输入框的值。当用户点击这个按钮，我们将那个值传递到组件中的`addHero`方法，接着清除它为下个新的hero名字做准备。

在那个按钮下方是一个可能出现的错误信息。

### HeroListComponent 类

我们将`HeroService`注入到`constructor`中。这是我们在父组件 `TohComponent`提供的`HeroService`实例。

Notice that the component does not talk to the server directly! The component doesn't know or care how we get the data. Those details it delegates to the heroService class (which we'll get to in a moment). This is a golden rule: always delegate data access to a supporting service class.

Although the component should request heroes immediately, we do not call the service get method in the component's constructor. We call it inside the ngOnInit lifecycle hook instead and count on Angular to call ngOnInit when it instantiates this component.

> This is a "best practice". Components are easier to test and debug when their constructors are simple and all real work (especially calling a remote server) is handled in a separate method.

The service get and addHero methods return an Observable of HTTP Responses to which we subscribe, specifying the actions to take if a method succeeds or fails. We'll get to observables and subscription shortly.

With our basic intuitions about the component squared away, we can turn to development of the backend data source and the client-side HeroService that talks to it.

### Fetch data
