# 自定义参数序列化

## 需求分析

在之前的章节，我们对请求的 url 参数做了处理，我们会解析传入的 params 对象，根据一定的规则把它解析成字符串，然后添加在 url 后面。在解析的过程中，我们会对字符串 encode，但是对于一些特殊字符比如 `@`、`+` 等却不转义，这是 axios 库的默认解析规则。当然，我们也希望自己定义解析规则，于是我们希望 `ts-axios` 能在请求配置中允许我们配置一个 `paramsSerializer` 函数来自定义参数的解析规则，该函数接受 `params` 参数，返回值作为解析后的结果，如下：

```typescript
axios.get('/more/get', {
  params: {
    a: 1,
    b: 2,
    c: ['a', 'b', 'c']
  },
  paramsSerializer(params) {
    return qs.stringify(params, { arrayFormat: 'brackets' })
  }
}).then(res => {
  console.log(res)
})
```

## 代码实现

首先修改一下类型定义。

`types/index.ts`：

```typescript
export interface AxiosRequestConfig {
  // ...
  paramsSerializer?: (params: any) => string
}
```

然后修改 `buildURL` 函数的实现。

`helpers/url.ts`：

```typescript
/**
 * @description 把 params 拼接到 url 上
 * @example
 * axios({
 *   method: 'get',
 *   url: '/base/get',
 *   params: {
 *     a: 1,
 *     b: 2
 *   }
 * })
 * // 拼接为
 * /base/get?a=1&b=2
 *
 * { foo: ['bar', 'baz'] }
 * // 数组需要拼接为
 * /base/get?foo[]=bar&foo[]=baz'
 *
 * { foo: { bar: 'baz' } }
 * // 对象需要拼接为（encode 之后的结果）
 * /base/get?foo=%7B%22bar%22:%22baz%22%7D
 *
 * const date = new Date()
 * { date }
 * // 日期需要拼接为 date.toISOString()
 * /base/get?date=2023-05-09T03:37:20.151Z
 *
 * { foo: '@:$, ' }
 * // 允许特殊字符 \@、:、$、,、、[、] 出现在 url 中，而不被 encode。注意：空格会被转为 +
 * // 上一行的 \at 符号实际不需要前面的反斜杠，at 符号在注释中有特殊含义，需要加个反斜杠转义一下
 * /base/get?foo=@:$+
 *
 * { foo: 'bar', baz: null }
 * // 忽略空值 null、undefined
 * /base/get?foo=bar
 *
 * axios({
 *   method: 'get',
 *   url: '/base/get#hash',
 *   params: {
 *     foo: 'bar'
 *   }
 * })
 * // 丢弃 url 中的哈希标记
 * /base/get?foo=bar
 *
 * axios({
 *   method: 'get',
 *   url: '/base/get?foo=bar',
 *   params: {
 *     bar: 'baz'
 *   }
 * })
 * // 保留 url 中已存在的参数
 * /base/get?foo=bar&bar=baz
 */
export function buildURL(
  url: string,
  params?: any,
  paramsSerializer?: (params: any) => string
): string {
  if (!params) {
    return url
  }

  let serializedParams

  if (paramsSerializer) {
    serializedParams = paramsSerializer(params)
  } else if (isURLSearchParams(params)) {
    serializedParams = params.toString()
  } else {
    const parts: string[] = []

    Object.keys(params).forEach(key => {
      const val = params[key]
      // 忽略空值
      if (val === null || typeof val === 'undefined') {
        return
      }
      let values = []
      // 处理数组
      if (Array.isArray(val)) {
        values = val
        key += '[]'
      } else {
        values = [val]
      }
      values.forEach(val => {
        // 处理 Date 对象
        if (isDate(val)) {
          val = val.toISOString()
        } else if (isPlainObject(val)) {
          // 处理对象
          val = JSON.stringify(val)
        }
        parts.push(`${encode(key)}=${encode(val)}`)
      })
    })

    // 多个参数时需要通过 & 拼接
    serializedParams = parts.join('&')
  }

  if (serializedParams) {
    const markIndex = url.indexOf('#')
    if (markIndex !== -1) {
      // 丢弃 url 中的哈希标记
      url = url.slice(0, markIndex)
    }

    // 保留 url 中已存在的参数
    url += (url.indexOf('?') === -1 ? '?' : '&') + serializedParams
  }

  return url
}
```

这里我们给 `buildURL` 函数新增了 `paramsSerializer` 可选参数，另外我们还新增了对 `params` 类型判断，如果它是一个 `URLSearchParams` 对象实例的话，我们直接返回它 `toString` 后的结果。

`helpers/util.ts`：

```typescript
export function isURLSearchParams(val: any): val is URLSearchParams {
  return typeof val !== 'undefined' && val instanceof URLSearchParams
}
```

最后我们要修改 `buildURL` 调用的逻辑。

`core/dispatchRequest.ts`：

```typescript
function transformURL(config: AxiosRequestConfig): string {
  const { url, params, paramsSerializer } = config
  return buildURL(url!, params, paramsSerializer)
}
```

## demo 编写

`examples/more-params-serializer/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>More example</title>
  </head>
  <body>
    <script src="/__build__/more-params-serializer.js"></script>
  </body>
</html>
```

`examples/more-params-serializer/app.ts`

```typescript
import axios from '../../src/index'
import qs from 'qs'
import createButton from '../create-button'

createButton('params: new URLSearchParams()', () => {
  axios.get('/more/get', {
    params: new URLSearchParams('a=b&c=d')
  }).then(res => {
    console.log(res)
  })
})

createButton('使用默认处理', () => {
  axios.get('/more/get', {
    params: {
      a: 1,
      b: 2,
      c: ['a', 'b', 'c']
    }
  }).then(res => {
    console.log(res)
  })
})

createButton('自定义 paramsSerializer', () => {
  const instance = axios.create({
    paramsSerializer(params) {
      console.log('自定义 ', qs.stringify(params, { arrayFormat: 'brackets' }));

      return qs.stringify(params, { arrayFormat: 'brackets' })
    }
  })

  instance.get('/more/get', {
    params: {
      a: 1,
      b: 2,
      c: ['a', 'b', 'c']
    }
  }).then(res => {
    console.log(res)
  })
})
```

我们编写了 3 种情况的请求，第一种满足请求的 params 参数是 `URLSearchParams` 对象类型的。后两种请求的结果主要区别在于前者并没有对 `[]` 转义，而后者会转义。

默认处理：会保留 `[` `]`

```http
http://localhost:3000/more/get?a=1&b=2&c[]=a&c[]=b&c[]=c
```

自定义参数序列化，使用 `qs.stringify`：会转义 `[` -> `%5B`，转义 `]` -> `%5D`

```http
http://localhost:3000/more/get?a=1&b=2&c%5B%5D=a&c%5B%5D=b&c%5B%5D=c
```

`examples/server.js` 响应内容追加 `query`，方便验证

```js
  router.get('/more/get', (req, res) => {
    res.json({
      cookies: req.cookies,
      query: req.query,
    })
  })
```

`examples/index.html`

```html
      <li><a href="more-params-serializer">More: paramsSerializer</a></li>
```

至此，`ts-axios` 实现了自定义参数序列化功能，用户可以配置 `paramsSerializer` 自定义参数序列化规则。下一节课我们来实现 `ts-axios` 对 `baseURL` 的支持。
