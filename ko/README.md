# HTTP API 설계 가이드

## 주의
초벌 번역으로 오역이 존재합니다. 읽을 때 참고하세요.

## 소개

이 가이드에서는 [Heroku 플랫폼 API](https://devcenter.heroku.com/articles/platform-api-reference)를 만들 때의 경험을 바탕으로 HTTP+JSON API를 설계하는 방법에 대해 설명한다.

이 가이드에는 API에 추가되어야 하는 내용이 포함되어 있으며
Heroku의 새로운 내부 API를 만들 때도 이 가이드를 활용한다.
이 가이드가 Heroku 외부의 API 설계자들에게도 흥미로운 내용이 되길 바란다.

이 가이드에서 달성하고자 하는 목표는 설계의 낭비를 피하면서 일관성을 유지하고 비즈니스 로직에 집중하는 것이다.
우리는 _훌륭하고, 일관적이고 문서화가 잘 된_ API를 설계하는 방법을 찾는 것이며
_유일한/이상적인 방법_을 찾아야만 하는 것은 아니다.

이 가이드는 HTTP+JSON API의 기초를 알고 있는 독자를 대상으로 하고 있으므로
이 가이드에서 HTTP+JSON API의 기초적인 내용을 모두 다루지는 않을 것이다.

이 가이드에 [기여](../CONTRIBUTING.md)하는 것은 언제나 환영이다.

## 차례

*  [알맞은 상태 코드를 반환하라](#return-appropriate-status-codes)
*  [전체 리소스를 제공 가능한 곳에서는 전체 리소스를 제공하라](#provide-full-resources-where-available)
*  [요청의 본문에 직렬화된 JSON을 포함시켜라](#accept-serialized-json-in-request-bodies)
*  [리소스의 (UU)ID를 제공하라](#provide-resource-uuids)
*  [표준 타임스탬프를 제공하라](#provide-standard-timestamps)
*  [ISO8601에 정의된 포맷으로 UTC 시간을 사용하라](#use-utc-times-formatted-in-iso8601)
*  [일관된 패스 형태를 사용하라](#use-consistent-path-formats)
*  [패스와 속성은 소문자로 만들어라](#downcase-paths-and-attributes)
*  [외래키 관계는 중첩시켜라](#nest-foreign-key-relations)
*  [편의를 위해 id없는 역참조?(dereferencing)을 지원하라](#support-non-id-dereferencing-for-convenience)
*  [에러를 구조적으로 만들어라](#generate-structured-errors)
*  [Etags로 캐시할 수 있도록 지원하라](#support-caching-with-etags)
*  [Request Id로 요청을 추적하라](#trace-requests-with-request-ids)
*  [범위별로 페이지를 나눠라?](#paginate-with-ranges)
*  [사용량의 상태를 보여줘라?](#show-rate-limit-status)
*  [Accepts 헤더에 버전을 부여하라?](#version-with-accepts-header)
*  [경로의 중첩을 최소화하라](#minimize-path-nesting)
*  [기계가 읽을 수 있는 JSON 스키마를 제공하라](#provide-machine-readable-json-schema)
*  [사람이 읽을 수 있는 문서를 제공하라](#provide-human-readable-docs)
*  [실행 가능한 예제를 제공하라](#provide-executable-examples)
*  [안정성을 표현하라?](#describe-stability)
*  [TLS를 요구하라](#require-tls)
*  [기본적으로 예쁜 모양의 JSON을 제공하라](#pretty-print-json-by-default)

### 알맞은 상태 코드를 반환하라

각 응답마다 적절한 HTTP 상태 코드를 반환하라. 성공적인 응답의 경우 다음의 가이드에 따라 코드를 지정하라.

* `200`: `GET` 호출, 동기적으로 완료되는 `DELETE`나 `PATCH` 호출 요청이 성공했을 때
* `201`: 동기적으로 완료되는 `POST` 호출 요청이 성공했을 때
* `202`: 비동기적으로 처리될 `POST`, `DELETE`, `PATCH` 호출을 받았을 때
* `206`: `GET` 호출 요청이 성공했으나 부분적인 응답이 반환될 때. [범위별로 페이지를 나눠라?](#paginate-with-ranges) 참조

사용자 에러와 서버 에러의 경우에 해당하는 상태 코드에 대한 가이드는 [HTTP 응답 코드 스펙](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)을 참고하라.

### 전체 리소스를 제공할 수 있는 곳에서는 전체 리소스를 제공하라

응답을 보낼 때는 가능한 전체 리소스를 보여주는 결과를 제공하라. (즉, 오브젝트의 모든 속성을 제공하라.)
200과 201응답을 보낼 때는 `PUT`/`PATCH`와 `DELETE` 요청의 경우를 포함하여 전체 리소스를 항상 제공하라.

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

202 응답의 경우에는 다음과 같이 전체 리소스를 보여주는 결과를 제공하지 않는다.

```
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

### Accept serialized JSON in request bodies
### 요청의 본문에 직렬화된 JSON을 포함시켜라

Accept serialized JSON on `PUT`/`PATCH`/`POST` request bodies, either
instead of or in addition to form-encoded data. This creates symmetry
with JSON-serialized response bodies, e.g.:
`PUT`/`PATCH`/`POST` 요청의 본문에 직렬화된 JSON을 사용하라. 폼 데이터를 대체하거나
같이 사용한다. 이렇게 함으로써 응답의 JSON 형태의 본문과 동일한 모습이 될 것이다.

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

### Provide resource (UU)IDs
### 리소스의 (UU)ID를 제공하라

Give each resource an `id` attribute by default. Use UUIDs unless you
have a very good reason not to. Don’t use IDs that won’t be globally
unique across instances of the service or other resources in the
service, especially auto-incrementing IDs.
각 리소스에 기본적으로 `id` 속성을 할당하라. 사용하지 말아야 할 이유가 있지 않다면 UUID를 사용하라.
서비스 인스턴스나 서비스의 다른 리소스 사이에서 유일한 값이 될 수 없는 ID를 사용하면 안된다.
특히 자동으로 증가하는 ID는 사용하지 말것.

Render UUIDs in downcased `8-4-4-4-12` format, e.g.:
UUID는 다음의 예와 같이 소문자로 된 `8-4-4-4-12`와 같은 형태로 표현하라. 
```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

### Provide standard timestamps
### 표준 타임스탬프를 제공하라

Provide `created_at` and `updated_at` timestamps for resources by default,
e.g:
기본적으로 리소스에 대해 `created_at`과 `updated_at` 타임스탬프를 제공하라.

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

These timestamps may not make sense for some resources, in which case
they can be omitted.
이런 타임스탬프는 일부 리소스의 경우에는 의미가 없을 수도 있는데 그런 경우에는 생략해도 된다.

### Use UTC times formatted in ISO8601
### ISO8601에 정의된 포맷으로 UTC 시간을 사용하라

Accept and return times in UTC only. Render times in ISO8601 format,
e.g.:
UTC로만 시간 값을 주고 받아라. 시간은 ISO8601 형태로 표시하라.

```
"finished_at": "2012-01-01T12:00:00Z"
```

### Use consistent path formats
### 일관된 패스 형태를 사용하라

#### Resource names
#### 리소스 이름

Use the plural version of a resource name unless the resource in question is a singleton within the system (for example, in most systems a given user would only ever have one account). This keeps it consistent in the way you refer to particular resources.
질의 중인 리소스가 시스템 내에서 하나만 존재하지는 것이 아니라면 리소스 이름에 복수형을 사용하라.
(예를 들어 대부분의 시스템에서 특정 사용자는 하나의 계정만을 갖는다.) 이러한 방법을 사용하면
특정한 리소스를 참조하는 방법의 일관성을 유지할 수 있다.

#### Actions
#### 액션

Prefer endpoint layouts that don’t need any special actions for
individual resources. In cases where special actions are needed, place
them under a standard `actions` prefix, to clearly delineate them:
개별 리소스에 특별한 액션이 필요하지 않는 종료점 레이아웃을 더 사용하라.(?) 특별한 액션이 필요한 경우
특별한 액션을 일반적인 `actions` 접두 뒤에 두면 된다. 그렇게 함으로써 명확하게 
특별한 액션을 설명할 수 있다.

```
/resources/:resource/actions/:action
```

예제:

```
/runs/{run_id}/actions/stop
```

### Downcase paths and attributes
### 패스와 속성은 소문자로 만들어라

Use downcased and dash-separated path names, for alignment with
hostnames, e.g:
소문자를 사용하고 단어는 대시로 구별하여 패스의 이름을 사용하여 호스트명과 정렬을 맞춘다.

```
service-api.com/users
service-api.com/app-setups
```

Downcase attributes as well, but use underscore separators so that
attribute names can be typed without quotes in JavaScript, e.g.:
속성도 소문자를 사용한다. 그러나 구분자로 언더스코어를 사용하여 속성의 이름이 자바스크립트에서도
따옴표 없이 표현할 수 있도록 한다.

```
service_class: "first"
```

### Nest foreign key relations
### 외래키 관계는 중첩시켜라

Serialize foreign key references with a nested object, e.g.:
외래키로 참조하는 내용은 중첩된 객체로 직렬화한다.

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```
  
Instead of e.g:
다음과 같이 사용하지 않는다.

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

This approach makes it possible to inline more information about the
related resource without having to change the structure of the response
or introduce more top-level response fields, e.g.:
이런 방법을 사용하면 응답의 구조를 변경하거나 최상위 응답 필드를 추가하지 않고도
관련된 리소스에 대한 더 많은 정보를 포함시킬 수 있다.

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

### Support non-id dereferencing for convenience
### 편의를 위해 id없는 역참조?(dereferencing)을 지원하라

In some cases it may be inconvenient for end-users to provide IDs to
identify a resource. For example, a user may think in terms of a Heroku
app name, but that app may be identified by a UUID. In these cases you
may want to accept both an id or name, e.g.:
어떤 경우에는 최종 사용자가 리소스를 식별할 수 있는 ID 정보를 제공하는 일이 불편할 때도 있다.
예를 들어 어떤 사용자가 Heroku 앱 이름을 알고 있지만 앱은 UUID로 구별하는 경우이다.
이런 경우 id나 이름을 모두 받으면 좋을 것이다.

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

Do not accept only names to the exclusion of IDs.
ID를 제외하고 이름만 받지는 말아야 한다.

### Generate structured errors
### 에러를 구조적으로 만들어라

Generate consistent, structured response bodies on errors. Include a
machine-readable error `id`, a human-readable error `message`, and
optionally a `url` pointing the client to further information about the
error and how to resolve it, e.g.:
에러의 경우 일관적이고 구조적인 형태로 응답을 만들어라. 컴퓨터가 읽을 수 있는 에러 `id`와
사람이 읽을 수 있는 에러 `message`는 필수로, 에러에 대한 추가 정보와 에러를 해결하는 방법이 있는
곳을 클라이언트에게 알려주는 `url`은 선택적으로 포함시켜라.

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

Document your error format and the possible error `id`s that clients may
encounter.
에러의 형식과 클라이언트가 맞닥뜨릴 수 있는 발생 가능한 에러 `id`를 문서화하라.

### Support caching with Etags
### Etags로 캐시할 수 있도록 지원하라

Include an `ETag` header in all responses, identifying the specific
version of the returned resource. The user should be able to check for
staleness in their subsequent requests by supplying the value in the
`If-None-Match` header.
`ETag`헤더를 모든 응답에 포함시켜 반환되는 리소스의 버전을 구별하라. 사용자는 이후의 요청에서
`If-None-Match` 헤더에 값을 지정하여 변경 여부를 확인하게 될 것이다.

### Trace requests with Request-Ids
### Request Id로 요청을 추적하라

Include a `Request-Id` header in each API response, populated with a
UUID value. If both the server and client log these values, it will be
helpful for tracing and debugging requests.
UUID의 값을 포함하고 있는 `Request-Id`헤더를 각 API 응답에 포함시켜라.
서버와 클라이언트 양쪽에서 이 값을 기록하면 요청을 추적하고 디버깅할 때 도움이 될 것이다.

### Paginate with Ranges
### 범위별로 페이지를 나눠라?

Paginate any responses that are liable to produce large amounts of data.
Use `Content-Range` headers to convey pagination requests. Follow the
example of the [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)
for the details of request and response headers, status codes, limits,
ordering, and page-walking.
대량의 데이터를 만드는 모든 응답은 페이지를 나눠라. `Content-Range`헤더를 사용하여 
페이지로 구별된 응답을 전달하라. 요청과 응답 헤더, 상태 코드, 숫자 제한(?), 순서, 페이지 워킹(?)에 대한
자세한 내용은 [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)의 예제를 따르라.

### Show rate limit status
### 사용량의 상태를 보여줘라?

Rate limit requests from clients to protect the health of the service
and maintain high service quality for other clients. You can use a
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) to
quantify request limits.
서비스를 안정적으로 유지하기 위해 클라이언트로부터 오는 요청을 제한하고 다른 클라이언트에게 
높은 수준의 서비스를 지속적으로 제공하라. 요청 제한을 측정하기 위해 
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket)을
사용할 수 있다.

Return the remaining number of request tokens with each request in the
`RateLimit-Remaining` response header.
각 요청마다 `RateLimit-Remaining` 응답 헤더에 남은 요청 토큰의 수를 반환하라.

### Version with Accepts header
### Accepts 헤더에 버전을 부여하라?

Version the API from the start. Use the `Accepts` header to communicate
the version, along with a custom content type, e.g.:
시작부터 API에 버전을 부여하라. `Accepts` 헤더를 사용하여 버전을 주고 받고 
커스텀 컨텐트 타입을 함께 사용하라.

```
Accept: application/vnd.heroku+json; version=3
```

Prefer not to have a default version, instead requiring clients to
explicitly peg their usage to a specific version.
기본 버전을 사용하지 않는 편이 좋다. 대신 클라이언트가 명시적으로 특정한 버전을 사용할 것임을
나타내도록 요구하라.

### Minimize path nesting
### 경로의 중첩을 최소화하라

In data models with nested parent/child resource relationships, paths
may become deeply nested, e.g.:
중첩되어 있는 부모/자식의 리소스 관계로 이루어진 데이터 모델의 경우
경로가 깊게 중첩될 수 있다.

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Limit nesting depth by preferring to locate resources at the root
path. Use nesting to indicate scoped collections. For example, for the
case above where a dyno belongs to an app belongs to an org:
루트 패스에서 리소스를 가리키는 방식을 사용하여 중첩되는 깊이를 제한하라.
한정된 범위의 콜렉션을 가리키는데 중첩을 사용하라. 예를 들어 dyno가 org에 포함된
app에 포함되는 위 예제와 같은 경우 다음과 같이 표현하라

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### Provide machine-readable JSON schema
### 기계가 읽을 수 있는 JSON 스키마를 제공하라

Provide a machine-readable schema to exactly specify your API. Use
[prmd](https://github.com/interagent/prmd) to manage your schema, and ensure
it validates with `prmd verify`.
기계가 읽을 수 있는 스키마를 제공하여 API를 정확하게 표현하라.
[prmd](https://github.com/interagent/prmd)를 사용하여 스키마를 관리하고
`prmd verify`로 유효성을 보장하라.

### Provide human-readable docs
### 사람이 읽을 수 있는 문서를 제공하라

Provide human-readable documentation that client developers can use to
understand your API.
사람이 읽을 수 있는 문서를 제공하여 클라이언트 개발자들이 API를 이해하는데 이용할 수 있도록 하라.

If you create a schema with prmd as described above, you can easily
generate Markdown docs for all endpoints with with `prmd doc`.
앞에서 설명한 prmd로 스키마을 만들었다면 `prmd doc` 명령어를 사용하여 
최종 항목에 대한 마크다운 문서를 쉽게 만들 수 있다.

In addition to endpoint details, provide an API overview with
information about:
최종 항목에 대한 상세한 정보와 더불어 다음과 같은 정보가 포함된 API 개요를 제공하라.

* Authentication, including acquiring and using authentication tokens.
* API stability and versioning, including how to select the desired API
  version.
* Common request and response headers.
* Error serialization format.
* Examples of using the API with clients in different languages.
* 인증, 인증 토큰을 획득하고 사용하는 방법이 포함되어야 한다.
* API 안정성과 버전 정책, 필요한 API 버전을 선택하는 방법이 포함되어야 한다.
* 공통으로 사용하는 요청, 응답 헤더
* 다양한 언어로 이루어진 클라이언트로 API를 사용하는 예제

### Provide executable examples
### 실행 가능한 예제를 제공하라

Provide executable examples that users can type directly into their
terminals to see working API calls. To the greatest extent possible,
these examples should be usable verbatim, to minimize the amount of
work a user needs to do to try the API, e.g.:
사용자가 직접 터미널에 입력하여 동작하는 API 호출을 확인할 수 있는 실행 가능한 예제를 제공하라.
이러한 예제는 있는 그대로 베껴서 사용가능하게 만들어  사용자가API를 시험해보기 위해 해야 하는
일의 양을 최소화해야한다.

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

If you use [prmd](https://github.com/interagent/prmd) to generate Markdown
docs, you will get examples for each endpoint for free.
[prmd](https://github.com/interagent/prmd)을 사용하여 마크다운 문서를 만든다면
모든 최종 항목에 대한 예제를 쉽게 만들 수 있을 것이다.

### Describe stability
### 안정성을 표현하라?

Describe the stability of your API or its various endpoints according to
its maturity and stability, e.g. with prototype/development/production
flags.
API의 안정성이나 API의 성숙도와 안정성에 따른 다양한 최종 항목을 표현하라. 예를 들어,
prototype/development/production 플래그를 제공하라.

See the [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
for a possible stability and change management approach.
사용할 수 있는 안정성과 변경 관리 방법은 [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)을 확인하라.

Once your API is declared production-ready and stable, do not make
backwards incompatible changes within that API version. If you need to
make backwards-incompatible changes, create a new API with an
incremented version number.
API가 상용화 준비가 완료되어 있고 안정적이라고 선언한 후에는 해당 API 버전에서는
하위 버전과 호환되지 않는 변경은 하지 말라. 하위 버전과 호환되지 않는 변경을 해야 한다면
버전 숫자를 증가시킨 새로운 API를 만들어라.

### Require TLS
### TLS를 요구하라

Require TLS to access the API, without exception. It’s not worth trying
to figure out or explain when it is OK to use TLS and when it’s not.
Just require TLS for everything.
API에 접근하기 위해 TLS를 요구하라. 예외는 없다. TLS를 사용해도 괜찮을 때와 그렇지 않을 때를
알려고 하거나 설명하려고 노력할 필요는 없다. 모든 작업에 TLS를 요구하라.

### Pretty-print JSON by default
### 기본적으로 예쁜 모양의 JSON을 제공하라

The first time a user sees your API is likely to be at the command line,
using curl. It’s much easier to understand API responses at the
command-line if they are pretty-printed. For the convenience of these
developers, pretty-print JSON responses, e.g.:
처음 사용자가 curl을 사용하여 API를 보는 것은 명령어 라인에 있는 것과 같다.
API가 예쁘게 출력된다면 명령어 라인에서 API 응답을 이해하는 것이 한층 쉬울 것이다.
이런 개발자의 편의를 위해 JSON 응답은 예쁘게 출력하라.

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

Instead of e.g.:
다음과 같이 출력하면 안된다.

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Be sure to include a trailing newline so that the user’s terminal prompt
isn’t obstructed.
마지막에 줄바꿈을 하여 사용자의 터미널 프롬프트가 불편한 곳에 위치하지 않도록 하라.

For most APIs it will be fine performance-wise to pretty-print responses
all the time. You may consider for performance-sensitive APIs not
pretty-printing certain endpoints (e.g. very high traffic ones) or not
doing it for certain clients (e.g. ones known to be used by headless
programs).
대부분의 API의 경우에는 성능이 괜찮으면서도 예쁘게 응답을 출력하면 된다. 
최종 항목을 예쁘게 출력하지 않거나 (아주 높은 트래픽을 처리해야 하는 API의 경우)
특정한 클라이언트에게만 예쁘게 출력하지 않는 (UI가 없는 프로그램에서 사용되는 것으로 알려진 API의 경우)
성능이 중요한 요소인 API를 고려할 수도 있을 것이다
