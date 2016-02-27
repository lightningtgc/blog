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
As usual, we import the symbols we need. The newcomer is HTTP_PROVIDERS, an array of service providers from the Angular HTTP library. We'll be using that library to access the server. We also import a HeroService that we'll look at shortly.

The component specifies both the `HTTP_PROVIDERS and the HeroService in the metadata providers array, making them available to the child components of this "Tour of Heroes" application.

> 学习关于`providers`的内容请参看[依赖注入](https://github.com/gf-rd/blog/issues/12)的章节。

This sample only has one child, the HeroListComponent shown here in full:

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

The component template displays a list of heroes with the NgFor repeater directive.

Beneath the heroes is an input box and an Add Hero button where we can enter the names of new heroes and add them to the database. We use a local template variable, newHero, to access the value of the input box in the (click) event binding. When the user clicks the button, we pass that value to the component's addHero method and then clear it to make ready for a new hero name.

Below the button is an optional error message.

### The HeroListComponent class
