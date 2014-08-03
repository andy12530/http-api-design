# HTTP API 设计指南

## 简介

这是一份关于 HTPP+JSON API设计的最佳实践指南，是在[Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference) 基础上改进的。

我们的目标是在业务逻辑中尽可能保持一致和专注，同时避免设计中的脱节。我们需要一个 *良好的*，*具有一致性*，*文档齐备*的 API 设计，并不是*一成不变*的。

我们假设你已经熟悉了HTPP+JSON API，因此这份文档不会覆盖这些基础知识。

欢迎大家对这份指南做出改进贡献[contributions]

## 内容

* [基础](#foundations)
  *  [是否 TLS](#require-tls)
  *  [HTTP版本和头部](#version-with-accepts-header)
  *  [支持缓存中的Etags](#support-caching-with-etags)
  *  [根据Request-Ids跟踪请求](#trace-requests-with-request-ids)
  *  [根据ranges分段](#paginate-with-ranges)
* [请求](#requests)
  *  [返回正确的状态码](#return-appropriate-status-codes)
  *  [尽可能的提供完整的资源](#provide-full-resources-where-available)
  *  [接受请求实体中序列化的 JSON](#accept-serialized-json-in-request-bodies)
  *  [使用一至的路径格式](#use-consistent-path-formats)
  *  [小写路径和属性](#downcase-paths-and-attributes)
  *  [支持非 ID的提取](#support-non-id-dereferencing-for-convenience)
  *  [最小化的路径嵌套](#minimize-path-nesting)
* [响应](#responses)
  *  [提供资源(uu)ids](#provide-resource-uuids)
  *  [提供标准的时间戳](#provide-standard-timestamps)
  *  [使用ISO8601 UTC 时间格式 ](#use-utc-times-formatted-in-iso8601)
  *  [嵌套外键关系](#nest-foreign-key-relations)
  *  [通用的错误格式](#generate-structured-errors)
  *  [显示限速状态](#show-rate-limit-status)
  *  [默认美化 JSON 输出](#pretty-print-json-by-default)
* [参阅](#artifacts)
  *  [提供机器可读的 JSON 模式](#provide-machine-readable-json-schema)
  *  [提供人类可读的文档](#provide-human-readable-docs)
  *  [提供可执行的例子](#provide-executable-examples)
  *  [描述稳定性](#describe-stability)

### Foundations

#### Require TLS

Require TLS to access the API, without exception. It’s not worth trying
to figure out or explain when it is OK to use TLS and when it’s not.
Just require TLS for everything.

对于不支持 TLS 的请求返回`403 Forbidden`。

Respond to non-TLS requests with `403 Forbidden`.  Redirects are 
discouraged since they allow sloppy/bad client behaviour without 
providing any clear gain.  Clients that rely on redirects double up on 
server traffic and render TLS useless since sensitive data will already
 have been exposed during the first call.

#### HTTP 版本和头部

版本是这个 API 可以支持的最低，使用`Accepts`来传送版本，一个简单的鸽子：
```
Accept: application/vnd.heroku+json; version=3
```
宁可不提供默认的版本，也不要客户端盯着某个特定的版本来使用。


#### 支持缓存中的 Etags

在所有请求的响应头部提供`Etags`，标识出返回资源的特征。用户就可以在随后的请求中提供`If-None-Match`头部来确定资源内容是否过期。

#### 根据Request-Ids跟踪请求

在每个API 的响应头部包含`Request-Id`，目前流行使用 UUID 。如果服务端和客户端记录了这些值，它会对跟踪和调试请求非常有用。

#### 根据ranges分段

当响应包含大的数据时，使用头部的`Content-Range`来进行分段。参考[Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges) 详细列出了请求和响应的头部，状态码，限制，以及页面流。

### 请求

#### 返回正确的状态码

对每个响应返回正确的HTTP状态码。

成功的响应应该这样的指南编码：

* `200`: `GET`请求调用成功, 以及 `DELETE` 或者 `PATCH` 请求调用同步完成
* `201`: `POST`请求同步调用完成
* `202`: `POST`, `DELETE`, or `PATCH` 调用已经接受，并且会进行异步处理

* `206`: `GET`请求成功, 但是只有部分响应返回: 参见 [上述关于分段的描述](#paginate-with-ranges)

注意用户认证以及认证错误的状态码：

* `401 Unauthorized`: 请求失败是因为用户未认证
* `403 Forbidden`: 请求失败是因为用户没有被许可访问特定的资源

当发生错误时返回合适的状态码可以提供额外的信息：

* `422 Unprocessable Entity`: 请求已经被理解，但是包含错误的参数
* `429 Too Many Requests`: 请求超过限制，稍后重试
* `500 Internal Server Error`: 服务器内部错误，检查站点状态或者报告错误
参考阅读关于用户和服务器错误的状态码文档[HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) 


#### 尽可能的提供完整的资源

尽可能在响应中提供完整的资源描述（例如，对象的所有属性）。在201和200请求中总是提供完整的资源，包括`PUT`/`PATCH` 和 `DELETE`，例如：

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

202 响应不会提供完整的资源，例如：

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### 接受请求实体中序列化的 JSON

在`PUT`/`PATCH`/`POST` 请求实体中接受序列化的 JSON, 这可以作为表单数据的替代或者补充。这样就可以创建对称的JSON 序列化响应, 例如：

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### 使用一致的路径格式

##### 资源名称
对于资源名称使用复数，除非这个资源是系统里一个单独的（例如：在多数系统中，一个确定的用户只对应一个账户）。这将保证你指向特定的资源的方式是一致的。

##### 动作

Prefer endpoint layouts that don’t need any special actions for
individual resources. In cases where special actions are needed, place them under a standard `actions` prefix, to clearly delineate them:

```
/resources/:resource/actions/:action
```

e.g.

```
/runs/{run_id}/actions/stop
```

#### 小写路径和属性

使用小写和中划线分割的路径名，这样可以和主机名保持一致，例如：

```
service-api.com/users
service-api.com/app-setups
```

这样虽然好，但是使用中划线分割可以让属性名称在JavaScript里使用时，不用再用括号包起来，例如：
```
service_class: "first"
```

#### 支持非 ID的提取

某些情况下，终端用户提供 IDs来标识资源可能不太方便。例如：一个用户想获得Heroku应用程序的名称，但是这个应用程序是用 UUID 来标识的。这种情况下可以考虑接受 ID 和名称，例如：

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

不要只接受名称而忘记了 IDs

#### 最小化的路径嵌套
在嵌套资源父/子关系的数据模型中，路径可能存在深度嵌套，例如：

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```
限制嵌套深度而不要使用根资源定位。使用嵌套来表示范围的集合。例如：上面的例子是dyno属于app，而app属于org。
```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### 响应

#### 提供资源(uu)ids

默认给每个资源一个`id`属性。使用 UUIDs除非你有更好的理由。不要使用IDs这样不能跨越实例和服务的，尤其是自增 IDs。

使用小写的 UUIDs格式`8-4-4-4-12`，例如：

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### 提供标准的时间戳
默认为资源提供`created_at` 和 `updated_at`时间戳。例如：

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```
这些时间戳可能对某些资源没有意义，那么它是可以被省略的。

#### 使用ISO8601 UTC 时间格式

只接受和返回 UTC时间。渲染时间使用 ISO8601的格式。例如：

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### 嵌套外键关系
在嵌套的对象中使用序列化外键引用，例如：

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```

代替，例如：

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```
这样的好处在于，内联有关信息或资源变动，而无需改变响应的结构或者引进更多高级别的响应字段，例如：

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

#### 通用的错误格式

在响应正方中使用通知，一致的错误格式。包括一个机器可读的错误`id`，人类可读的错误`message`，可选的`url`向客户端指出关于错误的更多信息以及如何解决它。例如：


```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

记录下客户可以遇到的错误格式以及可能的错误`id`。

#### 显示限速状态

限制请求的速度可以保护服务的健康，并且可以保证向客户端提供高质量的服务。你可以使用[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket)  算法来量化请求限制数量。

在咋就的头部`RateLimit-Remaining` 显示请求tokens的剩余数量。

#### 默认美化 JSON 输出

第一次用户使用 API 可能是在命令行中使用curl。如果你使用了美化输出，那么 API 的
返回信息在命令行中会非常容易理解。为了方便开发人员，在响应中美化 JSON 输出，例如：

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

替代下面的例子：

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

确保包括一个换行符，这样可以确保用户终端不受阻碍。

对于大多数 API 来说漂亮的输入是非常有必要的。但对于某些性能敏感的 API 的关键点（例如：非常高的流量）或者客户端不关心的地方，可以不使用美化的输出。

### 参阅

#### 提供机器可读的 JSON 模式

提供机器可读模式，以精确地指定的API。使用 [PRMD]()https://github.com/interagent/prmd)来管理你的的应用，并确保它通过`PRMD验证`。

#### 提供人类可读的文档

提供人类可读的文档，客户端开发人员可以根据它来了解你的API。 

如上所述如果你使用PRMD创建了API，`prmd doc`可以很容易地生成Markdown文档。

除了关键点的细节，提供了一个API概述和相关信息：

* 认证，包括获得和使用的认证tokens
* API的稳定性和版本，包括如何选择合适的 API 版本
* 常用的请求和响应头部
* 不同语言的 API 调用示例

#### 提供可执行的例子
提供一个可执行的例子，这样用户就可以直接在终端里调用 API 工作。要尽最大可能保证例子可以一字不差的执行，这样就可以减少用户需要做尝试的 API。例如：

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```
如果你使用[prmd](https://github.com/interagent/prmd) 来生成Markdown文档的话，它会自己生成每个端点（endpoint）的例子。

#### 描述稳定性

描述 API 的稳定性或者各关键点（endpoint）的稳定性和成熟度。例如：使用 原型/开发/生产 来标识。

参见[Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
，一个可能的稳定和变化管理方法。

一旦 API被确定为生产环境准备或者稳定，不要在当前版本做出任何可能造成冲突的更改。如果必须要做出这样的更改
 增加版本号创建一个新的 API。

