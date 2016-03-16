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

注意组件不会直接与服务器打交道！组件不知道也不关注我们是怎么获得数据的。那些细节委派给了`heroService`类（我们待会会讲到）。有个指导原则：通过委派一个支持的服务类来获取数据。

虽然组件应该立即请求heroes,但我们不会再组件的constructor里拿方法去调用服务的，而是在生命周期的`ngOnInit`钩子里调用，并依靠Angular在它实例化这个组件时调用`ngOnInit`。

> 这是一个“最佳实践”。当组件的构造器都是简单的，真正起作用（尤其是调用远程服务器）是在一个分离的方法中进行操作时，组件会变得更容易测试和调试。

服务的获取跟`addHero`方法都会返回一个我们订阅的HTTP响应的可观察对象(Observable)，并指定一个方法成功或者失败对应的行为。我们待会还将讲到可观察对象(Observables)和订阅(subscription)。

在我们关于整理组件的基本直觉下，我们能够转变为使用后端数据源以及和它打交道的客户端侧的`HeroService`的开发方式。

### 获取数据

在我们之前很多例子中，我们在一个服务中通过返回模拟的heroes来伪造与服务器交互的情况，如同下面这个例子：

```js
import {Hero} from './hero';
import {HEROES} from './mock-heroes';
import {Injectable} from 'angular2/core';

@Injectable()
export class HeroService {
  getHeroes() {
    return Promise.resolve(HEROES);
  }
}

```

在这一节，我们使用 Angular自身的HTTP客户端服务从服务器上获取heroes。下面是新的`HeroService`:

app/toh/hero.service.ts
```js
import {Injectable}     from 'angular2/core';
import {Http, Response} from 'angular2/http';
import {Hero}           from './hero';
import {Observable}     from 'rxjs/Observable';

@Injectable()
export class HeroService {
  constructor (private http: Http) {}

  private _heroesUrl = 'app/heroes';  // URL to web api

  getHeroes () {
    return this.http.get(this._heroesUrl)
                    .map(res => <Hero[]> res.json().data)
                    .catch(this.handleError);
  }
  private handleError (error: Response) {
    // in a real world app, we may send the error to some remote logging infrastructure
    // instead of just logging it to the console
    console.error(error);
    return Observable.throw(error.json().error || 'Server error');
  }
}
```
我们从导入Angular的Http客户端服务开始，然后把它注入到`HeroService`的constructor中。

Http不是 Angular core里面的一部分。它是在自己的`angular2/http`库中的一个可选服务。此外，这个库甚至不是主要的Angular脚本文件。它是有自己的脚本文件（包含在 Angular npm bundles里面），所以我们必须在`index.html`中加载它。

index.html
```html
<script src="node_modules/angular2/bundles/http.dev.js"></script>
```

再观察一下我们是怎么调用`http.get`的：

app/toh/hero.service.ts (http.get)
```js
return this.http.get(this._heroesUrl)
                .map(res => <Hero[]> res.json().data)
                .catch(this.handleError);
```
我们将资源的URL传递过去，它请求了应该返回heroes的服务器。

> 一旦我们像下面附录描述的配置好`in-memory` web api。


> Alternatively, we can (temporarily) target a JSON file by changing the endpoint URL:


> private _heroesUrl = 'app/heroes.json'; // URL to JSON file

The return value may surprise us. Many of us would expect a promise. We'd expect to chain a call to then() and extract the heroes. Instead we're calling a map() method. Clearly this is not a promise.

In fact, the http.get method returns an Observable of HTTP Responses (Observable<Response>) from the RxJS library and map is one of the RxJS operators.

HTTP GET DELAYED

> The http.get does not send the request just yet! This observable is cold which means the request won't go out until something subscribes to the observable. That something is the HeroListComponent.

### RxJS Library
