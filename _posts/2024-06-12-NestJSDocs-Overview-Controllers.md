---
title: "[NestJS | Docs | Overview] Controllers"
categories: [NestJS]
tags: [NestJS]
image: nestJSLogo.png
---


## Controller

---

![Desktop View](/2024-06-12-NestJSDocs-Overview-Controllers/controller.png){: width="600"  }

- `Controller`의 목적은 요청을 받고 응답하는 것
- `Routing mechanism`이란 특정요청에 대해 어떤 contorller가 응답할지 제어하는 것
- Controller는 하나이상의 Route를 가지며 , 각 route는 다르게 정의할 수 있다
- Controller를 만들기위해 class와 decorator를 사용한다
  - Decorator는 class와 metadata를 연결하고 nest가 routing map을 만들 수 있게한다.
- CLI's [**CRUD generator**](https://docs.nestjs.com/recipes/crud-generator#crud-generator)
  - `nest g resource [name]`

## Routing

---

```ts
//cats.controller.ts
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get() 
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

- To create a controller using the CLI
  - `$ nest g controller [name]`
- endpoint
  - Get /cats
  - `the (optional) prefix declared for the controller` +  <br>`any path specified in the method's decorator.`
- method name(`findAll`)은 중요하지 않음

### Nest employs two **different** options for manipulating responses

1. Standard (recommended)
    - return!!
    - primitive type(e.g., `string`, `number`, `boolean`)은 있는 그대로 return됨
    - reference type(e.g `object`,`array`)is  **automatically** serialized to JSON
    - default status code : 200 , except for POST requests which use 201.
    - `@HttpCode(...)` decorator를 이용해 status code 바꿀 수 있음
2. Library-specific(e.g., Express) [**response object**](https://expressjs.com/en/api.html#res)
    - `@Res()` 데코레이터 사용

    ```ts
    //cats.controller.ts
    import { Controller, Get, Res, HttpStatus } from '@nestjs/common';
    import { Response } from 'express';
    
    @Controller('cats')
    export class CatsController {
      @Get()
      findAll(@Res() res: Response) {
         res.status(HttpStatus.OK).json([]);
      }
    }
    ```

    - 주의사항

        `@Res()` , 또는 `@Next()` 를 사용하는경우

        - Standard approach is **automatically disabled**
        - To use both approaches at the same time
        (for example, by injecting the response object to only set cookies/headers but still leave the rest to the framework)
        `@Res({ passthrough: true })`

## Request object

---

```ts
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}

```

- In order to take advantage of `express` typings (as in the `request: Request` parameter example above), install `@types/express` package.
- list of the provided decorators

| --- | --- |
| @Request(), @Req() | req |
| @Response(), @Res()* | res |
| @Next() | next |
| @Session() | req.session |
| @Param(key?: string) | req.params / req.params[key] |
| @Body(key?: string) | req.body / req.body[key] |
| @Query(key?: string) | req.query / req.query[key] |
| @Headers(name?: string) | req.headers / req.headers[name] |
| @Ip() | req.ip |
| @HostParam() | req.hosts |

- when you inject either `@Res()` or `@Response()` in a method handler,<br>
you put Nest into **Library-specific mode** for that handler,<br>
and you become responsible for managing the response.
you must issue some kind of response by making a call on the `response` object.<br>
(e.g., `res.json(...)` or `res.send(...)`), or the HTTP server will hang.

## Resources

---

Nest provides decorators for all of the standard HTTP methods:

 `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`, `@Options()`, and `@Head()`. In addition, `@All()`

```ts
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

## Route wildcards

---

```ts
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

- Pattern based routes are supported as well
- A wildcard in the middle of the route is only supported by express.

## Status code

We can easily change this behavior by adding the `@HttpCode(...)` decorator at a handler level.

```ts
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

## Headers

```ts
@Post()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat';
}
```

## Redirection

`@Redirect()` takes two arguments, `url` and `statusCode`, both are optional.<br>
 The default value of `statusCode` is `302` (`Found`) if omitted.

```ts
@Get()
@Redirect('https://nestjs.com', 301)
```

`@Redirect()` 데코레이터는 기본적으로 지정된 URL과 상태 코드를 사용하여 리디렉션을 수행한다.
메서드가 반환하는 값이 있으면, 그 값이 `@Redirect()` 데코레이터의 기본 값을 덮어씁니다.

```ts
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

## Route parameters

```ts
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

```ts
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}

```

## Sub-Domain Routing

- match되는 도메인을 domain을 캡쳐할 수 있다.

```ts
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

- route path의 param처럼  동적으로 사용 가능하다

```ts
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}

```

## Scopes

- NestJS 애플리케이션에서는 대부분의 객체(예: 서비스, Controller)가 싱글톤으로 관리되고
모든 요청에 대해 동일한 인스턴스를 공유한다.
- Node js는 싱글 스레드 이벤트 루프를 사용하여 요청을 처리함(스레드를 여러개 사용하지 않음) 때문에 state가 모두 공유되고 이는 데이터 경합이나 동시성 문제를 일으키지 않기때문에
싱글톤 인스턴스를 사용하는것은 대게 안전하다.
- 그러나 요청기반의 생명주기 (request-based lifetime) 가 필요할 수 있다
  - 테스트 케이스
  - per-request caching in GraphQL applications
  - request tracking or multi-tenancy
  - NestJS에서 제공하는 스코프 제어 기능을 사용할 수 있다.

## Asynchronicity

아래처럼 fulfilled data로 resolve하여 client에게 반환하는것은 자동으로 처리해준다.

```ts
@Get()
async findAll(): Promise<any[]> {
  return [];
}
```

```ts
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
```

Both of the above approaches work

## Request payloads

- 타입스크립트를 사용한다면 DTO(Data Transfer Object) schema를 이용하자
- `interface`도 가능하고 `class`도 가능하지만 class를 권장한다( interface는 컴파일이후에 사라지니까)

```ts
// path : cats/dto/create-cat.dto.ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

- `validationPipe`와 함께 사용할수도 있다

## Full resource sample

- `Bind()`의 사용 방법

```ts
import { Controller, Get, Query, Post, Body, Put, Param, Delete, Bind } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Body())
  create(createCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  @Bind(Query())
  findAll(query) {
    console.log(query);
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  @Bind(Param('id'))
  findOne(id) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  @Bind(Param('id'), Body())
  update(id, updateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  @Bind(Param('id'))
  remove(id) {
    return `This action removes a #${id} cat`;
  }
}
```

## Library-specific approach

- 응답객체에대해 완벽히 제어할 수 있는 장점이 있지만 플랫폼에 종속되고 테스트가 어려워짐 <br>
(응답객체  모킹)
- `interceptors`,`@HttpCode()` / `@Header()` decorators와 같은 nest기능 사용 못함

```ts
import { Controller, Get, Post, Bind, Res, Body, HttpStatus } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  @Bind(Res(), Body())
  create(res, createCatDto) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  @Bind(Res())
  findAll(res) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

- `passthrough` option can resolve

```ts
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
```
