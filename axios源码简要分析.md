# Axios库源码简要分析

## 入口

入口文件：module.exports = require('./lib/axios');

重点分析这个文件：

核心代码：

```js
/**
 * Create an instance of Axios
 *
 * @param {Object} defaultConfig The default config for the instance
 * @return {Axios} A new instance of Axios
 */
function createInstance(defaultConfig) {
	...
}

// Create the default instance to be exported
var axios = createInstance(defaults);

// Expose Axios class to allow class inheritance
axios.Axios = Axios;

// Factory for creating new instances
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

module.exports = axios;

// Allow use of default import syntax in TypeScript
module.exports.default = axios;
```



## 认识axios

​		axios本身就是通过createInstance创建出来的一个instance，axios上还有一个create方法，通过这个方法还能创建一个跟axios差不多的instance，区别就是前者使用了defaultConfig，create方法可以传入自定义的config，然后在create方法内部合并后再创建实例。

​		其实这里的操作有点迷，因为这些实例在使用的时候调用的都是Axios.prototype.request方法，而这个方法也是可以传入config，并在内部合并的，感觉是引入了不必要的复杂性。我理解这里主要是想突出这种设计：axios本身可以拿来直接用，如果有自定义需求，又可以用axios.create方法定制一个instance来用。

​		下面再说下这个一直提到的instance，从命名上来看，应该是通过某个构造函new出来的实例，但并不是，这只是一个普通的函数。当然js中函数也是对象，构造函数其实也不是什么特殊的函数，只是通过new的方式来使用，行为跟普通函数调用有区别而已。



## instance		

接下来重点研究下这个createInstance函数：

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  // 这里主要研究这个
  var instance = bind(Axios.prototype.request, context);
	。。。
  return instance;
}
```

暂时忽略utils.extend相关的操作，因为从字面上看他们只是对instance的一些扩展，不会改变instance的本质；最终的这个instance不是new Axios出来的那个东西，而是bind函数的返回值：

```js
module.exports = function bind(fn, thisArg) {
  return function wrap() {
    // 这一堆代码就是把arguments转换成一个真数组，
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```

也就是这个wrap函数： axios、axios.create出来的实例本质上都是这个wrap函数而已。

那么通过一个常用case来看看使用过程发生了什么：

```js
let param = {
    baseURL: process.env.VUE_APP_SOMEPREFIX,
    url: '/info/get/someDetail',
    method: 'post',
  }
export function getSth() {
  return axios(param);
}
```

上面的axios就是wrap函数，所以wrap中的arguments = [param]; （arguments并不是真正的数组，这里是示意性表示）

结合上面的bind调用，这里的fn就是Axios.prototype.request，return axios(param)其实就是return fn.apply(thisArg, args)

fn简写为request，thisArg是context，args就是[param]，也就是：

Axios.prototype.request.apply(context, [param])  ==  Axios.prototype.request(param).bind(context);

​		从感觉上Axios.prototype.request方法应该是个纯函数，也就是没有副作用，不依赖this等，但他不是，内部依赖了this，所以如果不是通过Axios的实例来调用，那就得绑定this来使用。

​		到这里为止，很多东西都清楚了，但是还有个问题，上面的context到底是什么，起什么作用。其实context才是Axios这个构造函数new出来的实例，他主要就是起到提供上下文给bind函数使用（从名字上也可以看出这点）。

## 简单总结：

​		简单总结一下：Axios构造函数new出来的实例并不是直接用来发请求的，他只是一个context，真正发请求的是Axios.prototype.request这个方法，但是他不是一个纯函数，依赖了this，所以需要new Axios出来的实例提供context。

每次调用createInstance方法（可以通过axios.create使用）生成的实例都是在全新的context下执行的。



## context

还是从createInstance函数里的代码来研究：

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);

  // Copy context to instance
  utils.extend(instance, context);

  return instance;
}
```

既然context是new Axios出来的实例，再来看下Axios的代码：

```js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```

主要带了defaults和interceptors两个属性。

注，defaults中的配置不是真的axios这个库中的defaults，而是合并了库的defaults和createInstance时传入的config之后的一个defaultConfig；



## extend

然后我们知道instance就是wrap函数，然后看看通过两个utils.extend都干了什么。

```js
  utils.extend(instance, Axios.prototype, context);
  utils.extend(instance, context);
```

```js
function extend(a, b, thisArg) {
  forEach(b, function assignValue(val, key) {
    if (thisArg && typeof val === 'function') {
      a[key] = bind(val, thisArg);
    } else {
      a[key] = val;
    }
  });
  return a;
}
```

```js
module.exports = function bind(fn, thisArg) {
  return function wrap() {
    // 这一堆代码就是把arguments转换成一个真数组args，
    return fn.apply(thisArg, args);
  };
};
```

这里稍微研究下源码的套了好几层的逻辑，就是把context上的defaults和interceptors属性以及request和getUri以及一对http methods（delete, get, head, options, post, put, patch），都挂在wrap函数上。



## 再次简单总结：在这之前都是设置阶段

最终通过createInstance拿到的对象如下：

let ret = bind(Axios.prototype.request, context)  <- 1

ret.defaults = context.defaults

ret.interceptors = context.interceptors

ret.request = bind(Axios.prototype.request, context) <- 2

ret.getUri = bind(Axios.prototype.getUri, context)

ret.get = bind(Axios.prototype.get, context)

上面1、2是同一个东西，当然用===判断是不一样的，但是执行起来效果一样（因为是同一个context）；



## 核心：request函数的逻辑，这里进入执行阶段了

```js
Axios.prototype.request = function request(config) {
  // Allow for axios('example/url'[, config]) a la fetch API
  if (typeof config === 'string') {
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }

  config = mergeConfig(this.defaults, config);

  // Set config.method
  if (config.method) {
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) {
    config.method = this.defaults.method.toLowerCase();
  } else {
    config.method = 'get';
  }
  
  // ------------处理config的分界线-----------------

  // Hook up interceptors middleware
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);

  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

这里mergeConfig的实现还是蛮复杂的，不过我们先不研究，暂且就简单理解为合并两个对象吧。

前面一半都是在处理config，比较简单。



后面开始处理interceptors了，问题是这些interceptors是哪里来的？

当然如果你不设置，就没有，那么我们可以在哪里设置，或者说在哪个阶段设置？



这里首先要明确一个事情，代码执行到这里意味着已经开始真正执行请求的逻辑了，而在这之前我们其实可以做很多设置的，

比如设置interceptors。



## 打断一下，先研究interceptors，再次回到设置阶段

```js
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
```

这里的interceptors分了两种：request和response，他们也不是简单的数组，而是提供了一个InterceptorManager来管理对他们的操作，但内部还是得有一个数组来存放我们传入的interceptors

```js
function InterceptorManager() {
  this.handlers = [];
}
/**
 * Add a new interceptor to the stack
 */
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};

/**
 * Remove an interceptor from the stack
 */
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

/**
 * Iterate over all the registered interceptors
 * This method is particularly useful for skipping over any
 * interceptors that may have become `null` calling `eject`.
 */
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};

module.exports = InterceptorManager;

```

我们看到构造函数内部还是有一个数组，符合我们的猜想。

这里的实现类似eventEmitter，主要看use这个方法：

接受两个函数，类似promise的用法，然后每个handler对象有两个key：fulfilled和rejected

所以使用case如下：

```JS
axios.interceptors.request.use(
  config => {
    return config;
  },
  error => {
    return Promise.reject(error);
  },
);
```

```js

service.interceptors.response.use(
  response => {
    if (// 正常请求) {
      return response;
    } else {
      return Promise.reject(response);
    }
  },
  error => {
    return Promise.reject(error);
  },
);
```

其中的config、response、error会在多个interceptors之间传递，具体如何实现需要在request函数中看。



## 继续request的逻辑，回到执行阶段

```js
Axios.prototype.request = function request(config) {
	。。。  
  // ------------处理config的分界线-----------------

  // Hook up interceptors middleware
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);

  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }

  return promise;
};
```

这里会构造一个chain数组，结构为[requestInterceptors, ..., dispatchRequest, undefined, ...., responseInterceptors ]

核心的执行逻辑就是:

```js
var promise = Promise.resolve(config);
while (chain.length) {
  promise = promise.then(chain.shift(), chain.shift());
}
```

这里就实现了先执行requestInterceptors，然后通过dispatchRequest真正发起请求，然后是responseInterceptors。

利用了Promise的特性，让config、response、error会在多个interceptors之间传递；



## dispatchRequest，继续研究执行阶段

```js
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);
	。。。
};
```

发现一个奇怪的东西：throwIfCancellationRequested，从字面上看是跟错误处理以及cancellation相关的逻辑。

那么请求可以cancel吗？

答案是可以，那么如何实现呢？

这里先埋一个伏笔，等把request的执行逻辑演技完再来看这个话题。

```js
  // Transform request data
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );
```

这里可以把config传入的data做统一的处理，比如格式什么的。

接下来是重点：

```js
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(function onAdapterResolution(response) {
    throwIfCancellationRequested(config);

    // Transform response data
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);

      // Transform response data
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }

    return Promise.reject(reason);
  });
```

这里通过使用不同的adapter实现了针对不同环境的（浏览器、node）适配。

这里再次出现throwIfCancellationRequested，都先忽略。

```js
    // Transform response data
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );
```

同样这里可以对response中的data做统一处理，然后就是直接return response了，交给后面的responseInterceptors来处理了，如果没有的话，就进入到库的使用者写的then或者catch中了。



## adapter的实现与request cancelation

```js
var adapter = config.adapter || defaults.adapter;
```

```js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // For browsers use XHR adapter
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
    // For node use HTTP adapter
    adapter = require('./adapters/http');
  }
  return adapter;
}
```

浏览器使用xhr，node环境使用http

### xhr

```js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestData = config.data;
    var request = new XMLHttpRequest();

    var fullPath = buildFullPath(config.baseURL, config.url);
    request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true);

    request.onreadystatechange = function handleLoad() {
			。。。
      settle(resolve, reject, response);
      request = null;
    };
    request.onabort = function handleAbort() {
      reject(createError('Request aborted', config, 'ECONNABORTED', request));
      request = null;
    };
    request.onerror = function handleError() {
      reject(createError('Network Error', config, null, request));
      request = null;
    };
    request.ontimeout = function handleTimeout() {
      reject(createError(timeoutErrorMessage, config, 'ECONNABORTED',
        request));
      request = null;
    };
    if (typeof config.onDownloadProgress === 'function') {
      request.addEventListener('progress', config.onDownloadProgress);
    }
    if (typeof config.onUploadProgress === 'function' && request.upload) {
      request.upload.addEventListener('progress', config.onUploadProgress);
    }
		// ----------------request的分割线-------------------
    if (config.cancelToken) {
      // Handle cancellation
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }

        request.abort();
        reject(cancel);
        // Clean up request
        request = null;
      });
    }
		//-----------------cancelation的分割线---------------------
    // Send the request
    request.send(requestData);
  });
};
```

在----------------request的分割线-------------------之前都是简化了细节的对XMLHttpRequest的封装：

​		逻辑也很简单，new XMLHttpRequest出来，open之后注册各种事件响应函数：

onreadystatechange

onabort

onerror

ontimeout

上传和下载的progress

然后再send

​		这其中有很多细节，比如设置http basic authentication的支持，封装response，封装error，添加xsrf支持，设置其他hearders，responseType的设置，withCredentidals设置，可以单独写个文章来总结一下。



之后和-----------------cancelation的分割线---------------------之前都是对cancelation的封装

​		因为new XMLHttpRequest被封装在xhr adapter之内了，普通用户没法接触到他，不能直接执行request.abort()。但是如果想cancal请求，必须要执行这个方法。

​		axios给出的方案是通过在config中提供一个cancalToken（因为config是贯穿整个请求始终的，很多地方的设计都利用了这一点），通过cancelToken提供的能力来实现cancelation。



### CancelToken

```js
'use strict';

var Cancel = require('./Cancel');

/**
 * A `CancelToken` is an object that can be used to request cancellation of an operation.
 */
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}

/**
 * Throws a `Cancel` if cancellation has been requested.
 */
CancelToken.prototype.throwIfRequested = function throwIfRequested() {
  if (this.reason) {
    throw this.reason;
  }
};
module.exports = CancelToken;

```

​		初一看很绕，new CancelToken的时候，接受一个executer；接着在构造函数内部生成一个new Promise中，把resolve函数的引用给一个外面的变量resolvePromise；然后在执行executer的时候又给这个外部传入的executer函数传入了一个cancel函数，这个cancel函数是延迟执行的，也就是不在生成cancelToken的时候执行，那么可以通过在提供executer的时候，拿到内部这个cancel函数。这个跟之前拿到Promise中的resolve函数的技巧是一样的。

​		拿到这个cancel函数有什么用呢？可以abort请求吗？猜测应该是可以的，但是函数内部并不能直接abort，我们来看看axios是如何实现的：

```js
function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  }
```

执行cancel函数的时候只是resolve了cancelToken内部的Promise，所以应该是在这个Promise的then中来处理的：

```js
    if (config.cancelToken) {
      // Handle cancellation
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }

        request.abort();
        reject(cancel);
        // Clean up request
        request = null;
      });
    }
```

这里把这个then的逻辑挂在cancelToken内的Promise上，之后任何时候执行cancel函数，就会进入这个then的逻辑中，然后检测是否还有request对象，如果有，就abort，至此完美实现cancelation。

​		至于为什么要检测request是否还存在，因为前面的onreadystatechange，onabort，onerror，ontimeout事件都会把request置为null，同时也说明request已经完成或者出错，不需要abort了。

​	cancelToken可以手动new出来使用，同时也提供了工厂方法：

```js
/**
 * Returns an object that contains a new `CancelToken` and a function that, when called,
 * cancels the `CancelToken`.
 */
CancelToken.source = function source() {
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};
```

是CancelToken构造函数上的静态方法，其实也可以看做直接使用构造函数的一个示例。



## http

// todo











