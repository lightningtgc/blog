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

> 或者，我们能(短暂地)通过改变请求的URL来指定到一个JSON文件：

> private _heroesUrl = 'app/heroes.json'; // JSON 文件URL

返回的值可能让我们吃惊.我们大多数以为是个promise对象。我们以为要链式地调用`then()`并获取heroes。而实际上我们是调用了`map()`方法。很明显这不是一个promise。

实际上，http.get方法从RxJS库里返回一个HTTP响应的可观察对象(Observable<Response>)，还有map是RxJS的一个操作方法。

HTTP的延迟

> http.get暂时还没有发送请求！这个可观察对象是具有冻结性的，意味着请求不会发出去除非有东西订阅了这个对象。这个东西就是HeroListComponent。


### RxJS 库

RxJS ("Reactive Extensions") 是一个Angular支持的第三方的库，可以实现异步观察者模式。

我们全部的开发者指南示例都安装了RxJS的npm包，以及在`index.html`里加载了RxJS脚本，这么做是因为可观察模式在Angular程序里使用得很广泛。

index.html
```html
<script src="node_modules/rxjs/bundles/Rx.js"></script>
```
当用到HTTP客户端时，我们现在当然需要它。我们必须采用一个额外的关键步骤来使得RxJS可观察模式可用。


### 开启RxJS 操作

The RxJS library is quite large. Size matters when we build a production application and deploy it to mobile devices. We should include only those features that we actually need.

Accordingly, Angular exposes a stripped down version of Observable in the rxjs/Observable module, a version that lacks almost all operators including the ones we'd like to use here such as the map method we called above in getHeroes.

It's up to us to add the operators we need. We could add each operator, one-by-one, until we had a custom Observable implementation tuned precisely to our requirements.

That would be a distraction today. We're learning HTTP, not counting bytes. So we'll make it easy on ourselves and enrich Observable with the full set of operators. It only takes one import statement. It's best to add that statement early when we're bootstrapping the application. :

app/main.ts (import rxjs)
```js
// Add all operators to Observable
import 'rxjs/Rx';
```

### Map the response object

Let's come back to the HeroService and look at the http.get call again to see why we needed map()

app/toh/hero.service.ts (http.get)
```js
return this.http.get(this._heroesUrl)
                .map(res => <Hero[]> res.json().data)
                .catch(this.handleError);
```

The response object does not hold our data in a form we can use directly. It takes an additional step — calling response.json() — to transform the bytes from the server into a JSON object.

> This is not Angular's own design. The Angular HTTP client follows the ES2015 specification for the response object returned by the Fetch function. That spec defines a json() method that parses the response body into a JavaScript object.

> We shouldn't expect json() to return the heroes array directly. The server we're calling always wraps JSON results in an object with a data property. We have to unwrap it to get the heroes. This is conventional web api behavior, driven by security concerns.

** Make no assumptions about the server API. Not all servers return an object with a data property. **

#### Do not return the response object

Our `getHeroes()` could have returned the `Observable<Response>`.

Bad idea! The point of a data service is to hide the server interaction details from consumers. The component that calls the `HeroService` wants heroes. It has no interest in what we do to get them. It doesn't care where they come from. And it certainly doesn't want to deal with a response object.

#### Always handle errors

The eagle-eyed reader may have spotted our use of the catch operator in conjunction with a handleError method. We haven't discussed so far how that actually works. Whenever we deal with I/O we must be prepared for something to go wrong as it surely will.

We should catch errors in the HeroService and do something with them. We may also pass an error message back to the component for presentation to the user but only if we can say something the user can understand and act upon.

In this simple app we provide rudimentary error handling in both the service and the component.

We use the Observable catch operator on the service level. It takes an error handling function with the failed Response object as the argument. Our service handler, errorHandler, logs the response to the console, transforms the error into a user-friendly message, and returns the message in a new, failed observable via Observable.throw.

app/toh/hero.service.ts
```js
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
```

### Subscribe in the HeroListComponent

Back in the HeroListComponent, where we called heroService.get, we supply the subscribe function with a second function to handle the error message. It sets an errorMessage variable which we've bound conditionally in the template.

app/toh/hero-list.component.ts (getHeroes)
```js
getHeroes() {
  this._heroService.getHeroes()
                   .subscribe(
                     heroes => this.heroes = heroes,
                     error =>  this.errorMessage = <any>error);
}
```

> Want to see it fail? Reset the api endpoint in the HeroService to a bad value. Remember to restore it!

#### Peek at results in the console

During development we're often curious about the data returned by the server. Logging to console without disrupting the flow would be nice.

The Observable do operator is perfect for the job. It passes the input through to the output while we do something with a useful side-effect such as writing to console. Slip it into the pipeline between map and catch like this.

app/toh/hero.service.ts
```js
    return this.http.get(this._heroesUrl)
                    .map(res => <Hero[]> res.json().data)
                    .do(data => console.log(data)) // eyeball results in the console
                    .catch(this.handleError);
```
Remember to comment it out before going to production!

### Send data to the server
