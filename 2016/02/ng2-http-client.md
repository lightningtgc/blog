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

RxJS库是相当大的。当我们要构建一个产品程序，并部署到移动设备上时，我们需要考虑它的体积。我们应该只引用那些我们实际上需要的特性。

因此，Angular 在`rxjs/Observable`模块中暴露一个Observable的精简版本，这版本缺少大多数操作方法包括一些我们想在这里用到的，如在`getHeroes`里调用到的map方法。

由我们决定是否添加哪些需要的操作方法。我们可以一个接一个地添加每个操作方法，直到我们有一个刚好符合我们需求的定制的Observable实现。

那样会成为今天分心的事。我们学习HTTP，但不计算字节。所以我们将让它对于我们更加简单，直接让Observable带上整套的操作方法吧。这仅仅用到一个import的导入声明。当我们启动程序时最好尽早地添加那个声明：

app/main.ts (import rxjs)
```js
// Add all operators to Observable
import 'rxjs/Rx';
```

### Map 响应的对象

让我们回到`HeroService`，再观察`http.get`的调用，来看看为什么我们需要用到`map()`。

app/toh/hero.service.ts (http.get)
```js
return this.http.get(this._heroesUrl)
                .map(res => <Hero[]> res.json().data)
                .catch(this.handleError);
```

响应的对象不能从我们直接使用的表单来控制我们的数据。这需要一个额外的步骤 - 调用`response.json()` - 将服务端返回的字节转换为一个JSON对象。

> 这不是Angular自己设计的。Angular HTTP 客户端遵循ES2015的规范通过`Fetch`函数返回响应对象。那个规范定义了一个解析响应体为一个JavaScript对象的`json()`方法。

> 我们不应该认为`json()`就会直接返回heroes数组。我们访问的服务器经常将JSON结果包装成一个有数据属性的对象。我们需要解包这个对象来获取heroes。这是常见的web api行为，初衷是基于安全的考虑。

**不要对服务器API 做任何假设。不是全部的服务器都会返回一个有数据属性的对象的**

#### 不要返回响应的对象

我们的`getHeroes()`可以返回`Observable<Response>`。

但这是糟糕的主意！数据服务的要点是去隐藏服务器跟用户的交互细节。调用`HeroService`的组件是想要获取heroes。它不关注我们怎么获取heroes。它也不关心它们从哪里来得。还有它当然也不想去处理响应的对象。

#### 总是对错误进行处理

眼尖的读者可能发现我们写的catch操作方法加入了一个handleError方法。我们还没讨论那么远关于里面实际上是怎么运作的。无论何时我们处理 I/O ，我们必须准备好假设真的出错后的处理手段。

我们应该在`HeroService`里面捕获错误并对进行相应处理。我们也可能将错误信息传回给组件来展示给用户看，但仅限于我们说的是用户能理解和行动起来的东西。

在这个简单的应用里，我们在服务跟组件中都提供基本的错误处理。

我们在服务层面上使用Observable的`catch`操作方法。它采用一个将失败的响应对象作为参数的错误处理函数。我们通过`Observable.throw`进行服务处理，错误处理,将响应打印到控制台，将错误转化为一个用户友好的信息，以及将信息返回成一个新的失败的可观察对象。

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

### 在HeroListComponent中的订阅模式

回到HeroListComponent中，在我们调用`heroService.get`方法的地方，我们提供了有第二个函数参数来处理错误信息的subscribe函数。它设置了一个我们在模板中有条件地绑定的errorMessage变量。

app/toh/hero-list.component.ts (getHeroes)
```js
getHeroes() {
  this._heroService.getHeroes()
                   .subscribe(
                     heroes => this.heroes = heroes,
                     error =>  this.errorMessage = <any>error);
}
```

> 想看看它失败的情况？可在HeroService里将api返回值重置成一个错误的值。但要记得恢复它。

#### 在控制台中一窥结果

在开发中我们经常对服务器返回的数据感到好奇。将数据打印到控制台又不扰乱开发流将是不错的做法。

Observable的do操作符可以完美胜任这项工作。当我们处理一些跟一个有用的副作用的东西例如写数据到控制台时它会将输入的东西输出出来。把她放在管道中像在`map`和`catch`之间这样。

app/toh/hero.service.ts
```js
    return this.http.get(this._heroesUrl)
                    .map(res => <Hero[]> res.json().data)
                    .do(data => console.log(data)) // eyeball results in the console
                    .catch(this.handleError);
```
记得在发布生产环境是要注释掉它！

### 发送数据到服务器
到现在为止我们已经知道怎么使用Angular的内置Http服务来从一个远程地址获取数据。接下来让我们增加能创建新的heroes和在后端保存它们的能力。

我们将创建一个简单方法给`HeroListComponent`去调用，一个仅拿到新的hero的名字并返回一个observable来保留新保存的hero的`addHero`方法。

```js
addHero (name: string) : Observable<Hero>
```
为了实现这个，我们需要知道一个关于创建heroes的服务器api的细节。

我们的数据服务器遵循经典的REST指导方针。它期望一个POST请求在我们获取heroes的相同端点。它期望新的hero数据到达请求体中，结构如同`Hero`的实体但没有`id`属性。请求体应该看起来像下面的：

```
{ "name": "Windstorm" }
```

服务器会产生`id`并返回包括它产生的`id`的新hero代表的完整JSON。Hero包裹成一个有自己数据属性的响应对象去返回。

现在我们知道API是怎么运作的，我们像下面一样来实现`addHerolike`:

app/toh/hero.service.ts (additional imports)
```js
import {Headers, RequestOptions} from 'angular2/http';
```
app/toh/hero.service.ts (addHero)
```js
  addHero (name: string) : Observable<Hero>  {

    let body = JSON.stringify({ name });
    let headers = new Headers({ 'Content-Type': 'application/json' });
    let options = new RequestOptions({ headers: headers });

    return this.http.post(this._heroesUrl, body, options)
                    .map(res =>  <Hero> res.json().data)
                    .catch(this.handleError)
  }
```

`post`方法的第二个参数body需要一个JSON字符串,所以在发送前我们需要使用`JSON.stringify`来转换hero的内容。

> 在不久的将来我们可能可以不用stringify这个步骤。

#### Headers

服务器要求POST请求体有一个`Content-Type`的头部。`headers`是`RequestOptions`中的一个属性。组成可选对象并以post方法的第三方参数将它传进去。

app/toh/hero.service.ts (headers)
```js
let headers = new Headers({ 'Content-Type': 'application/json' });
let options = new RequestOptions({ headers: headers });

return this.http.post(this._heroesUrl, body, options)
```
#### JSON 结果

正如get请求,我们从有`json()`的响应体中提取数据，并通过数据属性拆解hero。


> 知道服务器返回的数据模型。这个web api返回包装成有数据属性的对象的新的hero。一个不同的api可能只返回我们忽略数据的解除绑定的案例中的hero。

回到`HeroListComponent`中，我们看到它的`addHero`方法订阅通过服务的`addHero`方法返回的observable。当数据到达时，它将新的hero对象push进它的heroes数据中为了展示给用户看。

app/toh/hero-list.component.ts (addHero)
```js
addHero (name: string) {
  if (!name) {return;}
  this._heroService.addHero(name)
                   .subscribe(
                     hero  => this.heroes.push(hero),
                     error =>  this.errorMessage = <any>error);
}
```

### Fall back to Promises
Although the Angular http client API returns an Observable<Response> we can turn it into a Promise if we prefer. It's easy to do and a promise-based version looks much like the observable-based version in simple cases.

> While promises may be more familiar, observables have many advantages. Don't rush to promises until you give observables a chance.

Let's rewrite the HeroService using promises , highlighting just the parts that are different.

app/toh/hero.service.ts (promise-based) 
```js
getHeroes () {
  return this.http.get(this._heroesUrl)
                  .toPromise()
                  .then(res => <Hero[]> res.json().data, this.handleError)
                  .then(data => { console.log(data); return data; }); // eyeball results in the console
}
addHero (name: string) : Promise<Hero> {
  let body = JSON.stringify({ name });
  let headers = new Headers({ 'Content-Type': 'application/json' });
  let options = new RequestOptions({ headers: headers });
  return this.http.post(this._heroesUrl, body, options)
             .toPromise()
             .then(res => <Hero> res.json().data)
             .catch(this.handleError);
}
private handleError (error: any) {
  // in a real world app, we may send the error to some remote logging infrastructure
  // instead of just logging it to the console
  console.error(error);
  return Promise.reject(error.message || error.json().error || 'Server error');
}
```

app/toh/hero.service.ts (observable-based)
```js
  getHeroes () {
    return this.http.get(this._heroesUrl)
                    .map(res => <Hero[]> res.json().data)
                    .do(data => console.log(data)) // eyeball results in the console
                    .catch(this.handleError);
  }
  addHero (name: string) : Observable<Hero>  {
    let body = JSON.stringify({ name });
    let headers = new Headers({ 'Content-Type': 'application/json' });
    let options = new RequestOptions({ headers: headers });
    return this.http.post(this._heroesUrl, body, options)
                    .map(res =>  <Hero> res.json().data)
                    .catch(this.handleError)
  }
  private handleError (error: Response) {
    // in a real world app, we may send the error to some remote logging infrastructure
    // instead of just logging it to the console
    console.error(error);
    return Observable.throw(error.json().error || 'Server error');
  }
```

Converting from an observable to a promise is as simple as calling toPromise(success, fail).

We move the observable's map callback to the first success parameter and its catch callback to the second fail parameter and we're done! Or we can follow the promise then.catch pattern as we do in the second addHero example.

Our errorHandler forwards an error message as a failed promise instead of a failed Observable.

The diagnostic log to console is just one more then in the promise chain.

We have to adjust the calling component to expect a Promise instead of an Observable.

app/toh/hero-list.component.ts (promise-based) 
```js
getHeroes() {
  this._heroService.getHeroes()
                   .then(
                     heroes => this.heroes = heroes,
                     error =>  this.errorMessage = <any>error);
}
addHero (name: string) {
  if (!name) {return;}
  this._heroService.addHero(name)
                   .then(
                     hero  => this.heroes.push(hero),
                     error =>  this.errorMessage = <any>error);
}
```

app/toh/hero-list.component.ts (observable-based)
```js
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
```

唯一明显的不同是我们调用的是返回的promise的`then`方法而不是`subscribe`方法。我们给这两个方法相同的调用参数。

> The less obvious but critical difference is that these two methods return very different results!

> The promise-based then returns another promise. We can keep chaining more then and catch calls, getting a new promise each time.

> The subscribe method returns a Subscription. A Subscription is not another Observable. It's the end of the line for observables. We can't call map on it or call subscribe again. The Subscription object has a different purpose, signified by its primary method, unsubscribe.

> Learn more about observables to understand the implications and consequences of subscriptions

### 通过`JSONP`获取数据

We just learned how to make `XMLHttpRequests` using Angulars built-in `Http` service. This is the most common approach for server communication. It doesn't work in all scenarios.

For security reasons, web browsers block `XHR` calls to a remote server whose origin is different from the origin of the web page. The origin is the combination of URI scheme, hostname and port number. This is called the Same-origin Policy.

> Modern browsers do allow XHR requests to servers from a different origin if the server supports the CORS protocol. If the server requires user credentials, we'll enable them in the request headers.

Some servers do not support CORS but do support an older, read-only alternative called JSONP. Wikipedia is one such server.

> This StackOverflow answer covers many details of JSONP.

#### 搜索wikipedia（维基百科）
维基百科提供一个JSONP的搜索接口。让我们构建一个只要我们在文本框中输入就能从维基百科中展示联想内容的简单搜索功能。


图片


Angular Jsonp服务既为JSONP扩展了Http服务又约束我们只使用GET请求。因为JSONP是一个只读设备，所以其他所有的HTTP方法都会抛出一个的错误。

一如既往，我们将与一个Angular 数据通信客户端服务的交互包装进一个专用的服务里，这里将它命名为`WikipediaService`。

app/wiki/wikipedia.service.ts
```js
import {Injectable} from 'angular2/core';
import {Jsonp, URLSearchParams} from 'angular2/http';
@Injectable()
export class WikipediaService {
  constructor(private jsonp: Jsonp) {}
  search (term: string) {
    let wikiUrl = 'http://en.wikipedia.org/w/api.php';
    var params = new URLSearchParams();
    params.set('search', term); // the user's search value
    params.set('action', 'opensearch');
    params.set('format', 'json');
    params.set('callback', 'JSONP_CALLBACK');
    // TODO: Add error handling
    return this.jsonp
               .get(wikiUrl, { search: params })
               .map(request => <string[]> request.json()[1]);
  }
}
```
The constructor expects Angular to inject its jsonp service. We register that service with JSONP_PROVIDERS in the component below that calls our WikipediaService.

#### 搜索的参数


The Wikipedia 'opensearch' API expects four parameters (key/value pairs) to arrive in the request URL's query string. The keys are search, action, format, and callback. The value of the search key is the user-supplied search term to find in Wikipedia. The other three are the fixed values "opensearch", "json", and "JSONP_CALLBACK" respectively.

> The JSONP technique requires that we pass a callback function name to the server in the query string: callback=JSONP_CALLBACK. The server uses that name to build a JavaScript wrapper function in its response which Angular ultimately calls to extract the data. All of this happens under the hood.

如果我们搜索包含"Angular"文字的文章，我们能手动构造查询字符串并像这样调用jsonp：
```js

let queryString =
  `?search=${term}&action=opensearch&format=json&callback=JSONP_CALLBACK`

return this.jsonp
           .get(wikiUrl + queryString)
           .map(request => <string[]> request.json()[1]);
```
In more parameterized examples we might prefer to build the query string with the Angular URLSearchParams helper as shown here:

app/wiki/wikipedia.service.ts (search parameters)
```js
var params = new URLSearchParams();
params.set('search', term); // the user's search value
params.set('action', 'opensearch');
params.set('format', 'json');
params.set('callback', 'JSONP_CALLBACK');
```

This time we call jsonp with two arguments: the wikiUrl and an options object whose search property is the params object.

app/wiki/wikipedia.service.ts (call jsonp)
```js
// TODO: Add error handling
return this.jsonp
           .get(wikiUrl, { search: params })
           .map(request => <string[]> request.json()[1]);
```

Jsonp flattens the params object into the same query string we saw earlier before putting the request on the wire.

### WikiComponent组件

Now that we have a service that can query the Wikipedia API, we turn to the component that takes user input and displays search results.

app/wiki/wiki.component.ts
```js
import {Component}        from 'angular2/core';
import {JSONP_PROVIDERS}  from 'angular2/http';
import {Observable}       from 'rxjs/Observable';
import {WikipediaService} from './wikipedia.service';
@Component({
  selector: 'my-wiki',
  template: `
    <h1>Wikipedia Demo</h1>
    <p><i>Fetches after each keystroke</i></p>
    <input #term (keyup)="search(term.value)"/>
    <ul>
      <li *ngFor="#item of items | async">{{item}}</li>
    </ul>
  `,
  providers:[JSONP_PROVIDERS, WikipediaService]
})
export class WikiComponent {
  constructor (private _wikipediaService: WikipediaService) {}
  items: Observable<string[]>;
  search (term: string) {
    this.items = this._wikipediaService.search(term);
  }
}
```
The providers array in the component metadata specifies the Angular JSONP_PROVIDERS collection that supports the Jsonp service. We register that collection at the component level to make Jsonp injectable in the WikipediaService.

The component presents an <input> element search box to gather search terms from the user. and calls a search(term) method after each keyup event.

The search(term) method delegates to our WikipediaService which returns an observable array of string results (Observable<string[]). Instead of subscribing to the observable inside the component as we did in the HeroListComponent, we forward the observable result to the template (via items) where the async pipe in the ngFor handles the subscription.

> We often use the async pipe in read-only components where the component has no need to interact with the data. We couldn't use the pipe in the HeroListComponent because the "add hero" feature pushes newly created heroes into the list.

### 我们铺张浪费的应用程序

我们的维基百科搜索制造了太多服务端的回调。这是低效的且对于有限资源的移动设备存在潜在的昂贵性能开销。

##### 1. 等待用户停止输入

在每次点击后
At the moment we call the server after every key stroke. The app should only make requests when the user stops typing . Here's how it should work — and will work — when we're done refactoring:

图

##### 2. 当搜索项改变时再搜索

Suppose the user enters the word angular in the search box and pauses for a while. The application issues a search request for Angular.

Then the user backspaces over the last three letters, lar, and immediately re-types lar before pausing once more. The search term is still "angular". The app shouldn't make another request.

##### 3. 处理无序的响应

The user enters angular, pauses, clears the search box, and enters http. The application issues two search requests, one for angular and one for http.

Which response will arrive first? We can't be sure. A load balancer could dispatch the requests to two different servers with different response times. The results from the first angular request might arrive after the later http results. The user will be confused if we display the angular results to the http query.

When there are multiple requests in-flight, the app should present the responses in the original request order. That won't happen if angular results arrive last.

### 更多Observables的乐趣

We can address these problems and improve our app with the help of some nifty observable operators.

We could make our changes to the WikipediaService. But we sense that our concerns are driven by the user experience so we update the component class instead.

app/wiki/wiki-smart.component.ts
```js
import {Component}        from '@angular/core';
import {JSONP_PROVIDERS}  from '@angular/http';
import {Observable}       from 'rxjs/Observable';
import {Subject}          from 'rxjs/Subject';
import {WikipediaService} from './wikipedia.service';
@Component({
  selector: 'my-wiki-smart',
  template: `
    <h1>Smarter Wikipedia Demo</h1>
    <p><i>Fetches when typing stops</i></p>
    <input #term (keyup)="search(term.value)"/>
    <ul>
      <li *ngFor="let item of items | async">{{item}}</li>
    </ul>
  `,
  providers:[JSONP_PROVIDERS, WikipediaService]
})
export class WikiSmartComponent {
  constructor (private _wikipediaService: WikipediaService) { }
  private _searchTermStream = new Subject<string>();
  search(term:string) { this._searchTermStream.next(term); }
  items:Observable<string[]> = this._searchTermStream
    .debounceTime(300)
    .distinctUntilChanged()
    .switchMap((term:string) => this._wikipediaService.search(term));
}
```
We made no changes to the template or metadata, confining them all to the component class. Let's review those changes.

##### 创建一个搜索项的流

We're binding to the search box keyup event and calling the component's search method after each keystroke.

We turn these events into an observable stream of search terms using a Subject which we import from the RxJS observable library:
```js
import {Subject}          from 'rxjs/Subject';
```

Each search term is a string, so we create a new Subject of type string called _searchTermStream. After every keystroke, the search method adds the search box value to that stream via the subject's next method.

```js
private _searchTermStream = new Subject<string>();

search(term:string) { this._searchTermStream.next(term); }
```

#### 监听搜索项

Earlier, we passed each search term directly to the service and bound the template to the service results. Now we listen to the stream of terms, manipulating the stream before it reaches the WikipediaService.
```js
items:Observable<string[]> = this._searchTermStream
  .debounceTime(300)
  .distinctUntilChanged()
  .switchMap((term:string) => this._wikipediaService.search(term));
```

We wait for the user to stop typing for at least 300 milliseconds (debounce). Only changed search values make it through to the service (distinctUntilChanged).

The WikipediaService returns a separate observable of string arrays (Observable<string[]>) for each request. We could have multiple requests in flight, all awaiting the server's reply, which means multiple observables-of-strings could arrive at any moment in any order.

The switchMap (formerly known as flatMapLatest) returns a new observable that combines these WikipediaService observables, re-arranges them in their original request order, and delivers to subscribers only the most recent search results.

The displayed list of search results stays in sync with the user's sequence of search terms.


## 附录：存放在内存服务器的‘英雄的旅程’

If we only cared to retrieve data, we could tell Angular to get the heroes from a heroes.json file like this one:

app/heroes.json
```js
{
  "data": [
    { "id": "1", "name": "Windstorm" },
    { "id": "2", "name": "Bombasto" },
    { "id": "3", "name": "Magneta" },
    { "id": "4", "name": "Tornado" }
  ]
}
```

> We wrap the heroes array in an object with a data property for the same reason that a data server does: to mitigate the security risk posed by top-level JSON arrays.

我们像这样设置JSON文件的源点：

```js
private _heroesUrl = 'app/heroes.json'; // URL to JSON file
```
The get heroes scenario would work. But we want to save data too. We can't save changes to a JSON file. We need a web api server.

We didn't want the hassle of setting up and maintaining a real server for this chapter. So we turned to an in-memory web api simulator instead. You too can use it in your own development while waiting for a real server to arrive.

首先，通过`npm`安装它：
```
npm install a2-in-memory-web-api --save
```
Then load the script in the index.html below angular:

index.html
```html
BAD FILENAME: ../../../_fragments/server-communication/ts/index-in-mem-web-api.html.md   Current path: docs,ts,latest,guide,server-communication PathToDocs: ../../../
```

The in-memory web api gets its data from a class with a createDb() method that returns a "database" object whose keys are collection names ("heroes") and whose values are arrays of objects in those collections.

Here's the class we created for this sample by copy-and-pasting the JSON data:

app/hero-data.ts
```js
export class HeroData {
  createDb() {
    let heroes = [
      { "id": "1", "name": "Windstorm" },
      { "id": "2", "name": "Bombasto" },
      { "id": "3", "name": "Magneta" },
      { "id": "4", "name": "Tornado" }
    ];
    return {heroes};
  }
}
```

We update the HeroService endpoint to the location of the web api data.

```js
private _heroesUrl = 'app/heroes';  // URL to web api
```

Finally, we tell Angular itself to direct its http requests to the in-memory web api rather than externally to a remote server.

This redirection is easy because Angular's http delegates the client/server communication tasks to a helper service called the XHRBackend.

To enable our server simulation, we replace the default XHRBackend service with the in-memory web api service using standard Angular provider registration in the TohComponent. We initialize the in-memory web api with mock hero data at the same time.

Here are the pertinent details, excerpted from TohComponent, starting with the imports:

toh.component.ts (web api imports)
```js
import { provide }           from '@angular/core';
import { XHRBackend }        from '@angular/http';

// in-memory web api imports
import { InMemoryBackendService,
        SEED_DATA }          from 'angular2-in-memory-web-api/core';
import { HeroData }          from '../hero-data';
```

Then we add the following two provider definitions to the providers array in component metadata:

toh.component.ts (web api providers)
```js
// in-memory web api providers
provide(XHRBackend, { useClass: InMemoryBackendService }), // in-mem server
provide(SEED_DATA,  { useClass: HeroData }) // in-mem server data
```

在这个[在线的例子](https://angular.io/resources/live-examples/server-communication/ts/plnkr.html)中看看全部的源码。

