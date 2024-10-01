---
title: "[NestJS | Docs | Overview] Middleware"
categories: [BackEnd,NestJS]
tags: [nest-js]
image: nestJSLogo.png
---

## **Middleware**

---

![https://docs.nestjs.com/assets/Middlewares_1.png](https://docs.nestjs.com/assets/Middlewares_1.png)

미들웨어는  `route handler`이전에 호출되는 함수로 <br>
`request`,`response`(object) ,`next`(middleware function of express)에 접근권을 가지고 있다.<br>
Nest middleware are, by default, equivalent to [**express**](https://expressjs.com/en/guide/using-middleware.html) middleware.

- **official express documentation describes**
  Middleware functions can perform the following tasks:
  - execute any code.
  - make changes to the request and the response objects.
  - end the request-response cycle.
  - call the next middleware function in the stack.
  - 현재 미들웨어가 equest-response cycle을 끝내지 않으면 반드시 next()를 호출해야한다

`@Injectable()` decorator를 이용해 function또는 class로 custom Nest middleware를 구성 가능하다.

- `Express`와 `fastify`는 middleware와 method signatures가 다르게처리함 read more [**here**](https://docs.nestjs.com/techniques/performance#middleware).

```ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}

```

## **Dependency injection**

---

`Providers` 와 `contorllers`처럼 동일한 모듈 내에서 사용 가능한 dependency를 주입(inject)할 수 있다.(생성자에 주입)

## **Applying middleware**

---

module class의 `configure()` 메서드를 설정해주고 `implements` 키워드로  `NestModule` interface를 적용시켜야 한다.

```ts

// app.module.ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

`forRoutest()` method에 path와 request method를 패스해 미들웨어를 특정 request method에 제한할 수도 있다.

```ts
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

When using the `express` adapter,NestJS는 기본적으로 `body-parser` 패키지를 사용하여 요청 본문을 JSON 형식 및 URL 인코딩 형식으로 파싱한다.
해당 미들웨어를 커스터마이징하려면, NestJS 애플리케이션을 생성할 때 `bodyParser` 플래그를 `false`로 설정하여 기본 등록을 비활성화해야 한다.

## **Route wildcards**

---

Pattern based routes are supported as well.

```ts
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

- fastify는 지원안함 .Instead, you must use parameters (e.g., `(.*)`, `:splat*`).

## **Middleware consumer**

---

`MiddlewareConsumer`는 미들웨어를 관리하는 여러 내장 메서드를 제공하는 `helper class`다. (체이닝 스타일 가능)

`forRoutes()` 메서드는 `single string`, `multiple strings`,  `RouteInfo object`,  `a controller class` and `even multiple controller classes`를 인수로 받을 수 있다.

쉼표로 구분하여 전달하면 된다

```ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

## **Excluding routes**

We can easily exclude certain routes with the `exclude()` method.

This method can take a single string, multiple strings, or a `RouteInfo` object identifying routes to be excluded, as shown below:

```ts
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

The `exclude()` method supports wildcard parameters using the [**path-to-regexp**](https://github.com/pillarjs/path-to-regexp#parameters) package.

## **Functional middleware**

---

Let's transform the logger middleware from class-based into functional middleware to illustrate the difference:

```ts
//logger.middleware.ts
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```

And use it within the `AppModule`:

```ts
//app.module.ts
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

미들웨어에 종속성을 주입할 일이 없는경우에는 훨신 간단하게 작성할수 있으니 좋음

## **Multiple middleware**

---

In order to bind multiple middleware that are executed sequentially, simply provide a comma separated list inside the `apply()` method:

!!순차적으로실행됨!!

```ts
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);

```

## **Global middleware**

---

If we want to bind middleware to every registered route at once, we can use the `use()` method that is supplied by the `INestApplication` instance:

```ts
// main.ts
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```

글로벌 미들웨어는 함수형 미들웨어만 가능하며 클래스 미들웨어처럼 특정 라우트나 메소드에 제한을 걸 수 없다

Accessing the DI container in a global middleware is not possible.

You can use a [**functional middleware**](https://docs.nestjs.com/middleware#functional-middleware) instead when using `app.use()`
