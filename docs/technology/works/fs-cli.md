---
title: Node文件下载上传CLI工具实现
date: 2022-11-09
tags:
 - 技术笔记
 - 个人作品
categories:
 - 技术笔记
---
# Node文件下载上传CLI工具实现

## 前言
在日常学习/生活中，下载资源时，大部分情况是通过别人分享的资源站点，找到下载入口然后触发下载。

当资源通过url传播的时候，一般也是直接打开，通过浏览器触发下载。

资深的冲浪选手，一般会用一些客户端工具（还记得Win上的各种下载器），Mac上笔者有时候会使用 [NeatDownloadManager](https://www.neatdownloadmanager.com/index.php/en/)，无 🪜 时也能拥有不错的下载速度

**Coder**们用命令行下载文件的方式就很多了，比如最常使用的内置库 [curl](https://github.com/curl/curl)

下面是最常用的拉取资源的例子
```sh
# 链接是第三方服务缩短后的
# -L 参数表明自动对资源进行重定向
curl -L http://mtw.so/6647Rc -o 码上掘金logo.image

# 通过管道
curl -L http://mtw.so/6647Rc >码上掘金logo.image

# 原图链接 https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/759e2aa805c0461b840e0f0f09ed05fa~tplv-k3u1fbpfcp-zoom-1.image
```

当然 **curl** 也支持上传下载，以及多种传输协议，具体用法这里就不展开了，感兴趣的读者可以前往[Quick Reference: Curl 备忘清单](https://wangchujiang.com/reference/docs/curl.html) 进一步了解。

本文针对常见的文件下载&上传场景做一些定制。

定制场景包含`url文件下载`，`GitHub/Gitee Release资源简化下载`，`七牛OSS文件直传`等。

知识点包含`Node实现下载文件`，`Proxy 网络代理`，`CLI配置存储`等

下面是简单演示
TODO：

## url资源下载
先是纯 **url资源下载** 的场景，本小节将详细展开相关小功能的实现

### Node原生实现
基于`读写流`操作，可以看到代码还是十分的简洁
```ts
import https from 'https'
import fs from 'fs'
import path from 'path'
function downloadByUrl(url: string, filename?: string) {
  const filepath = path.resolve(filename || randomName())
  https.get(url, (response) => {
    // 创建1个可写流
    const writeStream = fs.createWriteStream(filepath)
    response.pipe(writeStream).on('close', () => {
      console.log(`file save to ${filepath}`)
    })
  })
}

// sourceUrl 为前面的原图链接
downloadByUrl(sourceUrl,'test.image')
```

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODI2MjI2ODMxNQ==668262268315)

### 下载进度获取
大一点的文件肯定无法实现秒下载，需要获取一下进度，了解现在下载了多少

资源的总大小可以一般可以通过`response headers`中的`content-length`字段获取
```ts
const sumSize = +response.headers['content-length']
```

流的传输进度可以通过`on data`事件间接获取

在不通过`response.setEncoding(BufferEncoding)`修改的编码时，`chunk`默认是`Buffer`类型

```ts
let receive = 0
response.on('data', (chunk: Buffer) => {
  receive += chunk.length
  const percentage = receive / sumSize
})
```
到此进度`percentage`就可以获取到了

对上面的方法进行稍加改造通过链式调用增加`progress`，`end`两个方法

```ts
function downloadByUrl(url: string, filename?: string) {
  let receive = 0

  // 支持链式调用相关逻辑
  let progressFn: (cur: number, rec: number, sum: number) => void
  let endFn: (filepath: string) => void

  const thisArg = {
    progress: (fn: typeof progressFn) => {
      progressFn = fn
      return thisArg
    },
    end: (fn: typeof endFn) => {
      endFn = fn
      return thisArg
    }
  }

  https.get(url, (response) => {
    // 输出文件路径
    const filepath = path.resolve(filename || randomName())
    // 创建一个可写流
    const writeStream = fs.createWriteStream(filepath)

    const sumSize = +response.headers['content-length']! || 0
    response.on('data', (chunk: Buffer) => {
      receive += chunk.length
      progressFn && progressFn(chunk.length, receive, sumSize)
    })

    response.pipe(writeStream).on('close', () => {
      endFn && endFn(filepath)
    })
  })
  return thisArg
}

// 调用示例
downloadByUrl(sourceUrl, 'test.image')
  .progress((current, receive, sum) => {
    console.log(receive, ((receive / sum) * 100).toFixed(2), '%')
  })
  .end((filepath) => {
    console.log('file save:', filepath)
  })
```
![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODI2MzA2OTAyMQ==668263069021)

### 重定向处理
部分资源在对外直接暴露时，可能是一个短链，此时就需要做重定向处理

重定向的状态码常见`301`和`302`，当然还有其它的3开头的这里不赘述

除了状态码，重定向的目标url由`response.headers.location`表示

这里稍微改造一下之前的代码，添加一个重定向逻辑即可
```ts
// 通过url 简单区分一下 资源是 https 还是 http 
const _http = url.startsWith('https') ? https : http
_http.get(
  url,
  {
    // 添加一个UA，避免404
    // 部分短链服务网站没有UA会响应404
    headers: {
      'User-Agent': 'node http module'
    }
  },
  (response) => {
    const { statusCode } = response
    // 判断状态码是否3开头
    if (Math.floor(statusCode! / 100) === 3) {
      // 且存在 location
      if (response.headers.location) {
        // 递归
        downloadByUrl(response.headers.location, filename)
          // 透传事件
          .progress(progressFn)
          .end(endFn)
        return
      }
      // 不存在抛出错误
      throw new Error(
        `url:${url} status ${statusCode} without location header`
      )
    }
  }
)
```

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODI2NjY3NjE0MA==668266676140)

为了防止无限重定向，还需要加个次数限制，再简单改造一下上述代码，添加一个配置属性作为入参

```ts
interface Options {
  filename: string
  maxRedirects: number
}
function downloadByUrl(url: string, option?: Partial<Options>) {
  const ops: Options = { filename: randomName(), maxRedirects: 10, ...option }
  // 省略一些重复代码
  _http.get(
    url,
    (response) => {
      const { statusCode } = response
      if (Math.floor(statusCode! / 100) === 3 && ops.maxRedirects) {
        ops.maxRedirects -= 1
        // 递归调用
        if (response.headers.location) {
          downloadByUrl(response.headers.location, ops)
          return
        }
      }
    }
  )
  return thisArg
}
```

### 请求超时
部分资源由于网络原因可能出现超时，为了避免长时间无反馈等待，可以设置超时时间

`http`模块支持`timeout`属性设置

```ts
// 接着之前的例子修改部分代码即可
const request = _http.get(
  url,
  {
    // 设置超时时间，单位ms
    timeout: ops.timeout || 300000,
  },
  (response) => {
    // 省略response 逻辑
  }
)
request.on('timeout', () => {
  // 中断请求，输出错误
  request.destroy()
  console.error(`http request timeout url:${url}`)
})
```
下面是请求 goggle logo 失败示例

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODMyOTcwMTAwMA==668329701000)

### Proxy
部分资源访问不顺畅的时候，通常会走服务代理（🪜）

以谷歌的`logo`资源链接`https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_92x30dp.png`

要让前面的方法`downloadByUrl`顺利执行，就需要其走代理服务

为`http`模块添加代理也非常简单，原生提供了一个`agent`参数，可用于设置代理

```ts
import http from 'http'

const request = http.get(url,{
  agent: Agent,
})
```

这个`Agent`的构造可以直接用社区已经封装好的[http-proxy-agent](https://www.npmjs.com/package/http-proxy-agent)

```ts
const HttpProxyAgent = require('http-proxy-agent')

const proxy = new HttpProxyAgent('http://127.0.0.1:7890')
```

在调用时只需将这个`proxy`实例传入即可

```ts
http.get(url, {
  agent: proxy
})
```

原有的方法只需要添加一个`proxy`入参即可，
```ts
const request = _http.get(url, {
  agent: ops.proxy ? new HttpProxyAgent(ops.proxy) : undefined,
})
```

下面是使用代理成功请求的示例

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODYwNTcyODQ3NA==668605728474)

### 自行实现proxyAgent

TODO:

### 合法文件名生成
文件下载到本地肯定需要有个名字，如果用随机的或者用户手动输入那肯定体验较差

最常见的就是通过`url`的`pathname`生成

比如上面的谷歌图片资源，咱们使用`URL`构造出一个示例，查看url的构成

```ts
new URL(sourceUrl)
```

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODY5MzA0OTM4Mg==668693049382)

文件名就可以取`pathname`最后一截，通过`path.basename`即可获取

```ts
import path from 'path'

const url = new URL('http://www.google.com/images/googlelogo_color_92x30dp.png')
const filename = path.basename(url.pathname) // googlelogo_color_92x30dp.png
```
当然文件名也可能会重复，再非覆盖写入的前提下，通过会在文件名后添加"分隔符+数字"，比如`x.png`,`x_1.png`,`x 1.png`

提取文件名与后缀可以用`path.parse`直接获取
```ts
import path from 'path'

// { ext: '.png', name: 'google' }
path.parse('google.png')

// { ext: '', name: 'hashname' }
path.parse('hashname')

// { ext: '.ts', name: 'index.d' }
path.parse('index.d.ts')

// { ext: '.', name: 'index' }
path.parse('index.')

// { ext: '', name: '.gitkeep' }
path.parse('.gitkeep')
```
但是针对带有多个 **.** 的文件名不太友好，比如`.d.ts`是期望被当做完整的`ext`处理

所以咱们可以对其简单递归包装一下实现1个`nameParse`，确保最后`parse(input).name === input`即可
```ts
function nameParse(filename: string, suffix = '') {
  const { name, ext } = path.parse(filename)
  if (name === filename) {
    return { name, ext: ext + suffix }
  }
  return nameParse(name, ext + suffix)
}
```
下面是运行示例

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODY5NzM4MDAzNw==668697380037)

到此完成了`name`和`ext`的分离

文件名分离后简单进行一下`name`的合法性替换，避免出现操作系统不支持的字符

>正则来自于Google

```ts
function normalizeFilename(name: string) {
  return name.replace(/[\\/:*?"<>|]/g, '')
}
```

再做文件名去重只需要给`name`添加后缀数字即可

`url`上的内容还可能存在`encode`的情况，比如`掘金.png` => encode => `%E6%8E%98%E9%87%91.png`

因此咱们在处理从`pathname`提取的`filename`前先进行必要的`decode`

```ts
decodeURIComponent('%E6%8E%98%E9%87%91.png') // 掘金.png
```

有了前面的准备工作咱们就可以组装出一个从`url`提取合法可用的文件名的方法嘞

```ts
function getValidFilenameByUrl(url: string) {
  const urlInstance = new URL(url)
  return decodeURIComponent(path.basename(urlInstance.pathname))
}
getValidFilenameByUrl('http://a/b/c.png?width=100&height') // c.png
```

然后是获取不重复的文件路径
```ts
function getNoRepeatFilepath(filename: string, dir = process.cwd()) {
  const { name, ext } = nameParse(filename)
  let i = 0
  let filepath = ''
  do {
    filepath = path.join(dir, `${name}${i ? ` ${i}` : ''}${ext}`)
    i += 1
  } while (fs.existsSync(filepath))
  return filepath
}
```

最后集成到`downloadByUrl`方法中，使输出的文件名可控

```ts
// ...code
const filename = normalizeFilename(
  ops.filename || getValidFilenameByUrl(url) || randomName()
)
const filepath = ops.override
  ? path.resolve(filename)
  : getNoRepeatFilepath(filename)

const writeStream = fs.createWriteStream(filepath)

// ...code
```

测试案例运行结果如下

![图片](https://img.cdn.sugarat.top/mdImg/MTY2ODc4Njc0NzcwMg==668786747702)

### 相关三方库

## 本地配置存储

## Releases资源下载

<comment/>
