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

​		axios是通过createInstance方法传入默认config创建出来的一个对象，axios上还有一个create方法，这个方法内部也是调用createInstance创建对象。好处就是可以再传入一个自定义config，跟默认config合并后，再传入createInstance方法。

​		这里的设计可以这么理解：如果没有特殊需求，可以拿来axios直接用，但是如果有一些预制config，可以通过为axios.create方法传入预制的config，来生成一个有一点特殊的对象来用。并且还可以在真正使用的时候再次传入每次请求时的特殊config，实现最大的灵活性。

​		下面再来说createInstance生成的'instance'，从命名上来看，应该是通过某个构造函new出来的实例，但并不是，这只是一个普通的函数。当然js中函数也是对象，构造函数也不是什么特殊的函数，只是通过new的方式来使用，行为跟普通函数调用有区别而已。

## createInstance		

接下来看下这个createInstance函数：

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  // 这里主要研究这个
  var instance = bind(Axios.prototype.request, context);
  
  // 这些暂时先不管
  // Copy axios.prototype to instance
  utils.extend(instance, Axios.prototype, context);
  // Copy context to instance
  utils.extend(instance, context);
  
  return instance;
}
```

暂时忽略utils.extend相关的操作，因为从字面上看他们只是对instance的一些扩展，不会改变instance的本质；最后return的这个instance不是new Axios出来的那个东西，而是bind函数的返回值：

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

也就是这个wrap函数： axios和axios.create的返回值都是这个wrap函数。



那么通过一个常用case来看看到底发生了什么：

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

上面的axios就是wrap函数，所以wrap中的arguments = [param]; （arguments并不是真正的数组，这里是示意性表示）。结合上面的bind调用，这里的fn就是Axios.prototype.request，thisArg就是context，axios(param)就是Axios.prototype.request.apply(context, [param])，等同于Axios.prototype.request(param).bind(context);

​		感觉上Axios.prototype.request方法应该是个纯函数，也就是没有副作用，不依赖this等。but, 他内部依赖了this，不是纯函数，如果不通过Axios的实例来调用，那就得绑定this来使用。

​		到这里为止，大概脉络基本清楚了。接下来考虑这个问题，上面的context是什么，起什么作用。其实context才是Axios这个构造函数new出来的实例，他主要就是起到提供上下文给bind函数使用（从名字上也可以看出这点）。

## 简单总结

​		简单总结一下：Axios构造函数new出来的实例并不是直接用来发请求的，他只是一个context，真正发请求的是Axios.prototype.request这个方法，但是他不是一个纯函数，依赖了this，所以需要new Axios出来的实例提供context。

​		每次调用createInstance方法（可以通过axios.create使用）生成的实例都是在全新的context下执行的。

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

注，defaults中的配置不是axios这个库中的defaultConfig，而是合并了库的defaultConfig和createInstance时传入的config之后的一个合并的config；



## extend

我们知道instance就是wrap函数，接着看看两个extend都干了什么

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
    // 这一堆代码就是把arguments转换成一个真数组args
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```

这里稍微研究下源码中套了好几层的逻辑，就是把context上的defaults和interceptors属性以及request和getUri以及一堆http methods（delete, get, head, options, post, put, patch），都挂在wrap函数上。



## 再次简单总结

在这之前都是处于设置阶段，也就是设置axios，还没真正发起请求。

最终通过createInstance拿到的对象如下：

**let ret = bind(Axios.prototype.request, context)**

ret.defaults = context.defaults

ret.interceptors = context.interceptors

**ret.request = bind(Axios.prototype.request, context)**

ret.getUri = bind(Axios.prototype.getUri, context)

ret.get = bind(Axios.prototype.get, context)

上面加粗的两个对象用===判断是不同的，但是使用起来效果却是一样的（因为是同一个context）；



## 核心：request函数的逻辑

这里开始进入执行阶段了，真正发起请求了。

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

这里mergeConfig的实现还是蛮复杂的，不过我们不研究，暂且就简单理解为合并两个对象吧。

前面一半都是在处理config，略过不看。



后面开始处理interceptors了，问题是这些interceptors是哪里来的？

当然如果你不设置，就没有，那么我们可以在哪里设置，或者说在哪个阶段设置？



这里首先要明确一个事情，代码执行到这里意味着已经开始真正发起请求了，而在这之前我们还有机会做设置，比如设置interceptors。



## interceptors

再次回到设置阶段：

```js
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
```

这里的interceptors分了两种：request和response，他们不是简单的数组，而是提供了一个InterceptorManager来管理对他们的操作，但内部还是得有一个数组来存放我们传入的interceptors

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
```

我们看到构造函数内部还是有一个数组，符合我们的猜想。

这里的实现类似node中的eventEmitter，下面主要看use这个方法：

接受两个函数，类似promise的用法，然后每个handler对象有两个key：fulfilled和rejected

使用case如下：

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



## 继续request的逻辑

回到执行阶段：

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

利用了Promise的特性，让config、response、error在多个interceptors之间传递；

## dispatchRequest

依然还在执行阶段

```js
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);
    // Ensure headers exist
  config.headers = config.headers || {};

  // Transform request data
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );
  
	。。。
  
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
};
```

发现一个奇怪的东西：throwIfCancellationRequested，从字面上看是跟错误处理以及cancellation相关的逻辑。

那么请求可以cancel吗？

答案是可以，那么如何实现呢？

这里先埋一个伏笔，等把request的执行逻辑研究完再来继续这个话题。

```js
  // Transform request data
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );
```

这里可以把config传入的data做统一的处理，比如调整格式之类的。

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

这里通过使用不同的adapter实现了针对不同环境（浏览器、node）的适配。

这里再次出现throwIfCancellationRequested，都先忽略。

```js
    // Transform response data
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );
```

同样这里可以对response中的data做统一处理，然后就是直接return response了，交给后面的responseInterceptors来处理了，如果没有的话，就进入到使用者写的then或者catch中了。



## adapter与cancel

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

​		在----------------request的分割线-------------------之前都是简化了细节的对XMLHttpRequest的封装：

​		逻辑也很简单，new XMLHttpRequest出来，open之后注册各种事件响应函数：onreadystatechange，onabort，onerror，ontimeout，上传和下载的progress，然后再send。

​		这其中有很多细节，比如设置http basic authentication的支持，封装response，封装error，添加xsrf支持，设置其他hearders，responseType的设置，withCredentidals设置，可以单独写个文章来总结一下。



​	之后和-----------------cancelation的分割线---------------------之前都是对cancelation的封装

​		因为new出来的request被封装在xhr adapter之内了，普通用户没法接触到他，不能直接执行request.abort()。但是如果想cancal请求，必须要执行这个方法。

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

​		初一看很绕，new CancelToken的时候，接受一个executer；接着在构造函数内部生成了一个promise，把这个promise的resolve函数的引用赋给一个promise之外的变量resolvePromise。

​		接着在执行executer的时候，给这个外部传入的executer函数传入了一个cancel函数。这个cancel函数是延迟执行的，也就是不在生成cancelToken的时候执行，那么可以通过在提供executer的时候，拿到内部这个cancel函数。这个跟之前拿到Promise中的resolve函数的技巧是一样的。

​		如果把代码改写一下就容易理解了：

```js
let cancel = function cancel(message) {
  if (token.reason) {
    // Cancellation has already been requested
    return;
  }

  token.reason = new Cancel(message);
  resolvePromise(token.reason);
}
executor(cancel);
```

​		拿到这个cancel函数有什么用呢？可以abort请求吗？猜测应该是可以的，我们来看看这个cancel函数的实现：

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

​		我们发现cancel函数内并没有直接abort请求，执行cancel函数的时候只是resolve了cancelToken内部的promise，所以猜测应该是在这个promise的then中来处理的：

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

source是CancelToken构造函数上的静态方法，这里也可以看做直接使用构造函数的一个示例。

## http

// todo











