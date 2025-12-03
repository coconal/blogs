---
title: axios 学习源码-Axios类(1)
date: 2025-12-03 13:00:00
categories:
  - 学习源码
tags:
  - axios
slug: axios-source-1
description: axios 学习源码(1)
# cover: /images/cover/my-first-post.jpg
---

今天带来 axios 学习源码(1)，主要分析 Axios 类的实现。

## 整体结构

在学习 axios 源码时，我们先从 Axios 的整个项目结构开始
下面是 axios 主要目录结构

```
axios
├── lib // 核心源码（ESM），包含请求管线、适配器、默认配置与工具
│   ├── core // 核心流程与类：请求派发、拦截器、配置合并、数据与头部处理
│   ├── adapters // 环境适配层：Node(http)、浏览器(xhr)、Fetch；含适配器解析与选择
│   ├── defaults // 全局默认配置与请求/响应转换策略、XSRF、超时等
│   ├── helpers // 通用工具：URL/参数序列化、FormData 转换、Cookies、节流测速等
│   ├── cancel // 取消机制：CancelToken/CanceledError/isCancel
│   ├── platform // 跨环境抽象：浏览器/Node 的 FormData、Blob、URLSearchParams等
│   ├── env // 环境与版本数据
│   ├── axios.js // 默认实例构造、静态属性挂载与对外导出
│   └── utils.js // 基础类型判断与通用工具库
├── index.js // 包根入口：解构并具名导出 axios 及静态工具，兼容 ESM/CJS
├── index.d.ts // TypeScript 类型定义（同时有 cts 变体）
├── test // 测试用例：单元/集成/模块兼容与回归
├── examples // 使用示例：GET/POST、上传、响应转换等
├── sandbox // 本地调试沙箱：简单页面与脚本
├── .github // 维护与自动化：CI、Issue/PR 模板、工作流配置
├── templates // 维护脚本使用的模板文件
├── bin // 项目维护脚本：发布、标签、贡献者、CI 辅助
├── package.json // 包配置、依赖与脚本
├── rollup.config.js // 打包配置（Rollup）
├── webpack.config.js // 打包配置（Webpack，备用/历史）
├── tsconfig.json // TypeScript 编译配置
├── README.md // 项目说明与用法
└── LICENSE // 许可证
```

我们先从 `lib/core/Axios.js` 开始分析
Axios 类是 axios 的核心类，负责处理请求和响应。它包含了请求管线、拦截器、配置合并、数据与头部处理等功能。

构造函数

```javascript
  constructor(instanceConfig) {
    this.defaults = instanceConfig || {};
    this.interceptors = {
      request: new InterceptorManager(),
      response: new InterceptorManager()
    };
  }
```

Axios 类的构造函数主要负责初始化 Axios 实例的默认配置和拦截器管理器。
`defaults` 属性存储了 Axios 实例的默认配置，包括请求/响应转换策略、XSRF、超时等。
`interceptors` 属性包含了请求拦截器和响应拦截器的管理器

```javascript
  /**
   * Dispatch a request
   *
   * @param {String|Object} configOrUrl The config specific for this request (merged with this.defaults)
   * @param {?Object} config
   *
   * @returns {Promise} The Promise to be fulfilled
   */
  async request(configOrUrl, config) {
    try {
      return await this._request(configOrUrl, config);
    } catch (err) {
      if (err instanceof Error) {
        // *** 错误处理
      }
    }
  }
```

`request` 是 Axios 类的async方法, 内部调用 `_request` 方法处理请求。
捕获错误后，会判断错误是否为 Error 实例。如果是，会进行错误处理。

接下来重点介绍`_request(configOrUrl, config)` 方法
一步步来

## 主方法 _request(configOrUrl, config)

### 合并配置

在进行其他操作前，会先合并配置 `configOrUrl` 和 `config`。

```javascript
    if (typeof configOrUrl === 'string') {
            config = config || {};
            config.url = configOrUrl;
        } else {
            config = configOrUrl || {};
        }

    config = mergeConfig(this.defaults, config);
```

这一步其实很简单，就是合并默认配置和传入的配置。
如果传入的配置是字符串，会将其作为 URL 赋值给 `config.url`。
否则，会将传入的配置合并到默认配置中。
然后使用 `mergeConfig` 方法合并默认配置和传入的配置。 具体实现可以参考 `lib/core/mergeConfig.js` 文件。

### 处理拦截器

获取 `config` 配置以后，将解构出
`const {transitional, paramsSerializer, headers} = config;`
transitional 是一个对象，包含了过渡性的配置项。
paramsSerializer 是一个函数，用于序列化请求参数。
headers 是一个对象，包含了请求头信息。

```javascript
   if (transitional !== undefined) {
      validator.assertOptions(transitional, {
        silentJSONParsing: validators.transitional(validators.boolean),
        forcedJSONParsing: validators.transitional(validators.boolean),
        clarifyTimeoutError: validators.transitional(validators.boolean)
      }, false);
    },

    if (paramsSerializer != null) {
      if (utils.isFunction(paramsSerializer)) {
        config.paramsSerializer = {
            serialize: paramsSerializer
        }
      } else {
        validator.assertOptions(paramsSerializer, {
            encode: validators.function,
            serialize: validators.function,
        }, true);
      }
    }
```

结构出后就对 `transitional` 进行校验。
如果 `transitional` 不是 undefined，校验 transitional 三个布尔项（静默 JSON、强制 JSON、明确超时错误）
如果 `paramsSerializer` 不是 null，会校验其是否为函数或符合参数序列化配置项的格式。 是函数，会将其赋值给 `config.paramsSerializer.serialize`。
否则，会校验 `paramsSerializer` encode、serialize 是否符合参数序列化配置项的格式。

### 检验绝对URL和拼写提示

`allowAbsoluteUrls` 填充和继承 `defaults.allowAbsoluteUrls`
然后对配置键做拼写提示（如 baseUrl → baseURL ）这个不多说

```javascript
    // Set config.allowAbsoluteUrls
    if (config.allowAbsoluteUrls !== undefined) {
    // do nothing
    } else if (this.defaults.allowAbsoluteUrls !== undefined) {
        config.allowAbsoluteUrls = this.defaults.allowAbsoluteUrls;
    } else {
        config.allowAbsoluteUrls = true;
    }

    validator.assertOptions(config, {
        baseUrl: validators.spelling('baseURL'),
        withXsrfToken: validators.spelling('withXSRFToken')
    }, true);
```

### 方法与头部处理

```javascript
    // Set config.method
    config.method = (config.method || this.defaults.method || 'get').toLowerCase(); // 转换为小写，保证后面查询方法为小写

    // Flatten headers header扁平化，合并上下文头和请求头
    let contextHeaders = headers && utils.merge(
        headers.common,
        headers[config.method]
    );

    headers && utils.forEach(
        ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
        (method) => {
            delete headers[method];
        }
    );

    config.headers = AxiosHeaders.concat(contextHeaders, headers);
```

先将 `headers.common` 和 `headers[config.method]` 合并到 `contextHeaders` 中。
然后删除 `headers` 中的 `common` 以及删除 `['delete', 'get', 'head', 'post', 'put', 'patch', 'common']` 遍历后对应的`headers[method]`。
最后将 `contextHeaders` 和 `headers` 合并到 `config.headers` 中。

例如：

```javascript
    headers = {
        common: {
            Accept: 'application/json'
        },
        get: {
            'Cache-Control': 'no-cache'
        },
        post: { 
            'X-POST-DEFAULT': '1', 'Content-Type': 'application/x-www-form-urlencoded' 
        },
        a: '123'
    };
    config = {
        method: 'post',
        headers: {
            Authorization: 'Bearer abc123',
            'Content-Type': 'application/json'
        }
    }

    // utils.merge 相当于以下操作
    const mergedHeaders = Object.assign({}, headers.common, headers[config.method]);

    /**合并后的 headers 为
     * 
     *     
    {
        'Accept': 'application/json' ,
        'Authorization': 'Bearer abc123',
        'X-POST-DEFAULT': '1',
        'Content-Type': 'application/json'
    }
     */
    
    // delete headers 后

    /**
     * 删除后的 headers 为
     * 
     *     
    {
        a: '123'
    }
     */

    /**
     * 合并后的 headers 为
     * 
     *     
    {
        'Accept': 'application/json' ,
        'Authorization': 'Bearer abc123',
        'X-POST-DEFAULT': '1',
        'Content-Type': 'application/json'
        a: '123'
    }
     */
```

### 收集拦截器

#### 收集请求拦截器

```javascript
    // filter out skipped interceptors
    const requestInterceptorChain = [];
    let synchronousRequestInterceptors = true;
    this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
      if (typeof interceptor.runWhen === 'function' && interceptor.runWhen(config) === false) {
        return;
      }

      synchronousRequestInterceptors = synchronousRequestInterceptors && interceptor.synchronous;

      requestInterceptorChain.unshift(interceptor.fulfilled, interceptor.rejected);
    });
```

上面这部分主要是添加请求拦截器到 `requestInterceptorChain` 中。
逐步遍历配置的request拦截器，判断 `runWhen` 是否为函数且返回值为 `false`，如果是，则跳过该拦截器。
否则，将该拦截器的 `fulfilled` 和 `rejected` 方法添加到 `requestInterceptorChain` 中。
最后，判断所有拦截器是否为同步拦截器，将结果赋值给 `synchronousRequestInterceptors`。
如果任何一个拦截器 `synchronous=false` ，整体走异步链；否则走同步链
`runWhen` 和 `synchronous` 详细见 [axios 拦截器](https://axios-http.com/docs/interceptors)
请求拦截器以 unshift 的方式入栈，形成“后添加先执行”的顺序

#### 收集响应拦截器

```javascript
    // filter out skipped interceptors
    const responseInterceptorChain = [];
    this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
      responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
    });
```

响应拦截器以 push 的方式入栈，形成“先添加先执行”的顺序。

#### 执行异步拦截器链

```javascript
    let promise;
    let i = 0;
    let len;

    if (!synchronousRequestInterceptors) {
        const chain = [dispatchRequest.bind(this), undefined];
        chain.unshift(...requestInterceptorChain);
        chain.push(...responseInterceptorChain);
        len = chain.length;

        promise = Promise.resolve(config);

        while (i < len) {
            promise = promise.then(chain[i++], chain[i++]);
        }

        return promise;
    }
```

synchronousRequestInterceptors 为 `true` 时，执行同步拦截器链。 直接跳过异步拦截器链。
异步拦截器链的执行顺序为：

1. 先执行所有请求拦截器，按照添加顺序执行。
2. 执行 `dispatchRequest` 函数，发送请求。
3. 执行所有响应拦截器，按照添加顺序执行。

那么，异步拦截器链的执行顺序为：

1. 先执行所有请求拦截器，按照添加顺序执行。
2. 执行 `dispatchRequest` 函数，发送请求。
3. 执行所有响应拦截器，按照添加顺序执行。

**值得注意的是，请求拦截器的执行顺序是相反的。先添加的请求拦截器最后执行。**

#### 执行同步拦截器链

```javascript
    len = requestInterceptorChain.length;

    let newConfig = config;

    while (i < len) {
      const onFulfilled = requestInterceptorChain[i++];
      const onRejected = requestInterceptorChain[i++];
      try {
        newConfig = onFulfilled(newConfig);
      } catch (error) {
        onRejected.call(this, error);
        break;
      }
    }
```

如果没有执行异步拦截链，则会执行以上的同步执行请求拦截器链。
同步执行请求拦截器链，按照添加顺序执行。执行顺序同上
最终形成promise链

#### 执行请求以及响应拦截器链

```javascript
    try {
        promise = dispatchRequest.call(this, newConfig);
    } catch (error) {
        return Promise.reject(error);
    }

    i = 0;
    len = responseInterceptorChain.length;

    while (i < len) {
        promise = promise.then(responseInterceptorChain[i++], responseInterceptorChain[i++]);
    }

    return promise;
```

这里就好理解多了。首先执行 `dispatchRequest` 函数，发送请求。
如果请求成功，执行所有响应拦截器，按照添加顺序执行。
如果请求失败，执行所有响应拦截器，按照添加顺序执行。

如果没有失败，再执行所有响应拦截器，按照添加顺序执行。

最后返回promise链
