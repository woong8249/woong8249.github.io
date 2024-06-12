---
title: "[NestJS | Docs] Overview-Exception Filter"
categories: [NestJS]
tags: [NestJS]
image: nestJSLogo.png
---


## Exception filters

---

Nest에는 어플리케이션에 걸쳐  다루어지지않은 모든 예외를 처리하는 **exceptions filter** 내장되어 있다.
예외처리가 되지않으면 `exceptions filter`에서 이를 포착하고 사용자에게 적절한 응답을 보낸다.
이는 `HttpException` type의 예외를 다루는 `built-in global exception filter`에 의해 수행된다.<br>
예외가 인식되지 않는경우(`HttpException`이 아니거나 이를 상속하지 않은 class인경우 )
`built-in exception filter` 는 아래와같은 default JSON response를 반환한다

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

![https://docs.nestjs.com/assets/Filter_1.png](https://docs.nestjs.com/assets/Filter_1.png)

`global exception filter`는 `http-errors` 라이브러리의 예외를 부분적으로 지원한다.<br>
던져진 예외가 `statusCode`와 `message` 속성을 포함하면, 해당 예외는 제대로 처리되어 응답으로,
인식되지 않은 예외는 기본적으로 `InternalServerErrorException`으로 처리된다.

## Throwing standard exceptions

---

Nest provides a built-in `HttpException` class, exposed from the `@nestjs/common` package.<br>
Let's assume that this route handler throws an exception for some reason.

```ts
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

We used the `HttpStatus` here. This is a helper enum imported from the `@nestjs/common` package.
When the client calls this endpoint, the response looks like this:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException` constructor takes two required arguments

1. `response` argument
    - It defines the JSON response body.
    -  It can be a `string` or an `object`
2. The `status` argument
    - It defines the [**HTTP status code**](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status).

기본적으로 `JSON response body`는 두속성을 포함한다

1. statuscode (should be a valid HTTP status code)
    - Best practice is to use the `HttpStatus` enum imported from `@nestjs/common`
2. message
    - JSON response body의 message를 덮어쓰려면 response argument에 string을 pass하면 된다.
    - JSON response body의 전체를 덮어쓰려면  pass an object in the `response` argument.
3. options(optinal)
    -  that can be used to provide an error [**cause**](https://nodejs.org/en/blog/release/v16.9.0/#error-cause).<br>
    This `cause` object is not serialized into the response object

아래는 response body 전체를 덮어쓰고 error cause를 제공하는 예시다

```ts
// cats.controller.ts
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}

```

Using the above, this is how the response would look:

```json
{
  "status": 403,
  "error": "This is a custom message"
}

```

## Custom exceptions

---

In many cases, you will not need to write custom exceptions, and can use the built-in Nest HTTP exception.<br>
하지만 사용자 정의 예외를 작성해야 하는 경우에는 **기본 HttpException 클래스를 상속**하는 자체적인 exception 계층을 만드는 것이 좋다.<br>
 Let's implement such a custom exception:

```ts
// forbidden.exception.ts
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

## Built-in HTTP exceptions

---

Nest provides a set of standard exceptions that inherit from the base `HttpException`.

| exceptions | status code |
| --- | --- |
| BadRequestException | 400 |
| UnauthorizedException | 401 |
| NotFoundException | 404 |
| ForbiddenException | 403 |
| NotAcceptableException | 406 |
| RequestTimeoutException | 408 |
| ConflictException | 409 |
| GoneException | 410 |
| HttpVersionNotSupportedException | 505 |
| PayloadTooLargeException | 413 |
| UnsupportedMediaTypeException | 415 |
| UnprocessableEntityException | 422 |
| InternalServerErrorException | 500 |
| NotImplementedException | 501 |
| ImATeapotException | 418 |
| MethodNotAllowedException | 405 |
| BadGatewayException | 502 |
| ServiceUnavailableException | 503 |
| GatewayTimeoutException | 504 |
| PreconditionFailedException | 412 |

아래와 같이 사용가능

```ts
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

Using the above, this is how the response would look:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400,
}
```

## Exception filters

---

you may want **full control over the exceptions layer**.<br>
It  can control the exact flow of control and the content of the response sent back to the client.<br>
Let's create an exception filter that is responsible for catching exceptions which are an instance of the `HttpException` class, and implementing custom response logic for them.<br>
Request 객체을 통해 원본 ***url***을 가져와 이를 response정보에 포함시키고, Response 객체를 통해**response.json()** 메서드를 사용하여 응답해 Exception을 제어해보자

```ts
//http-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

All exception filters should implement the generic `ExceptionFilter<T>` interface.<br>
@Catch데코레이터는 필수 메타데이터를 exception filter에 바인드해 Nest가 해당 예외만 찾도록 만든다.<br>
단일 또는 쉼표로 구분된 리스트 형태의 매개변수를 받을 수 있고, 이를 통해 한 번에 여러 유형의 예외에 대한 필터를 설정할 수 있다.<br>

### Arguments host

Let's look at the parameters of the `catch()` method.

1. exception : 현재 처리되는 exception
2. host : `ArgumentsHost object` ( we'll examine further in the [**execution context chapter**](https://docs.nestjs.com/fundamentals/execution-context))

- 객체로, 요청(Request) 및 응답(Response) 객체에 접근할 수 있게 한다.
- HTTP, 마이크로서비스, WebSocket 등 모든 실행 컨텍스트에서 작동한다.

## Binding filters

---

Let's tie our new `HttpExceptionFilter` to the `CatsController`'s `create()` method.

```ts
//cats.controller.ts
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

`@UseFilters()` decorator can take a single filter instance, or a comma-separated list of filter instances.

예시는 new와 함께 ForbiddenException 인스턴스를 throw했지만 ForbiddenException class자체도 가능하며 class 를 throw한경우 nest가 의존성을 주입해준다.

Prefer applying filters by using classes instead of instances when possible. It reduces **memory usage** since Nest can easily reuse instances of the same class across your entire module.

Exception filters can be scoped at different levels:

1. method-scoped of the controller/resolver/gateway
    - controller ⇒ Http
    - resolver ⇒ GraphQL
    - gateway ⇒ Socket
2. controller-scoped
3. global-scoped

to set up a filter as controller-scoped, you would do the following:

```ts
//cats.controller.ts
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

이 또한 class를 패스해도된다

To create a global-scoped filter, you would do the following:

```ts
//main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

The `useGlobalFilters()` method does not set up filters for gateways or hybrid applications

- gateways ⇒ socket
- hybrid applications ⇒ 다양한 통신 프로토콜을 혼합하여 사용하는 애플리케이션

의존성 주입측면에서 모든 모듈밖에서 등록된 전역필터는 모든 모듈 컨텍스트 밖이기때문에 nest에의해서 의존성이 주입될수없다

위 문제를 해결하기위해 아래와 같이 모듈에서 직접 전역범위 필터를 등록 할 수있다.<br>
(main에 작성했던   `app.useGlobalFilters(new HttpExceptionFilter());`은빼고 )

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

You can add as many filters with this technique as needed; simply add each to the providers array.

- **부연설명**

    Also, `useClass` is not the only way of dealing with custom provider registration. Learn more [**here**](https://docs.nestjs.com/fundamentals/custom-providers).

    하지만 위코드 예제는 `HttpExceptionFilter`에 아무것도 주입해주지 않는다.<br>
    아래는 내가 작성해본 예시코드이다.

    ```ts
    // app.module.ts
    import {
      MiddlewareConsumer,
      Module,
      NestModule,
      RequestMethod,
    } from '@nestjs/common';
    import { LoggerMiddleware } from './common/middleware/logger.middleware';
    import { AppController } from './app.controller';
    import { AppService } from './app.service';
    import { DogsModule } from './dogs/dogs.module';
    import { HttpExceptionFilter } from './common/exception/http-exception.filter';
    import { APP_FILTER } from '@nestjs/core';
    
    @Module({
      imports: [DogsModule],
      controllers: [AppController],
      providers: [
        AppService,
        {
          provide: APP_FILTER,
          useFactory: () => new HttpExceptionFilter(String(Math.random() * 10)),
        },
      ],
    })
    export class AppModule implements NestModule {
      configure(consumer: MiddlewareConsumer) {
        consumer
          .apply(LoggerMiddleware)
          .forRoutes({ path: 'cats', method: RequestMethod.GET });
      }
    }
    
    ```

    ```ts
    // http-exception.filter.ts
    import {
      ExceptionFilter,
      Catch,
      ArgumentsHost,
      HttpException,
    } from '@nestjs/common';
    import { Request, Response } from 'express';
    
    @Catch(HttpException)
    export class HttpExceptionFilter implements ExceptionFilter {
      constructor(private readonly token: string) {}
      catch(exception: HttpException, host: ArgumentsHost) {
        console.log(this.token);
        const ctx = host.switchToHttp();
        const response = ctx.getResponse<Response>();
        const request = ctx.getRequest<Request>();
        const status = exception.getStatus();
        const res = {
          statusCode: status,
          timestamp: new Date().toISOString(),
          path: request.url,
        };
        console.log(res);
        response.status(status).json(res);
      }
    }
    
    ```

    `provide: APP_FILTER`는 `HttpExceptionFilter` 클래스가 전역 필터로 사용될 수 있도록 설정하는 데 필요한 식별자를 제공하는 것이다.

## Catch everything

---

예외유형에 상관없이 모든 예외를 잡으려면 `@catch`  데코레이터를 비우면 된다.

아래코드는 [**HTTP adapter](https://docs.nestjs.com/faq/http-adapter)를 사용하기때문에 플랫폼에 종속되지 않는다**

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // In certain situations `httpAdapter` might not be available in the
    // constructor method, thus we should resolve it here.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

When combining an exception filter that catches everything with a filter that is bound to a specific type, the "Catch anything" filter should be declared first to allow the specific filter to correctly handle the bound type.

위 문장을 읽어보니 정의된 순서에 따른 exception Filter의 실행순서에대해 이것저것 실험해본 결과 먼가 수상해서 “인것같다”와 같은 추측성 말투로 글을 작성한다.

특정 A라는 예외에 대해 포착할수있는 두 종류의 `exception Filter`가 있다면, 미들웨어에 호출순서가 있는 것 처럼 exception Filter 또한 선언된 순서에 따라 어떤 `Filter`가 먼저 호출될지를 제어할 수 있는 것 같다.

1. 예상과는 다르게 선언되거나 배열안에 넣는 순서의 역순으로 Filter가 진행되었다
    - 전역모듈에서 의존성을 주입하는 방식과 아닌경우 모두 테스트 해봄
2. express의 에러 미들웨어처럼 next()를 이용할수는 없는 것 같다
    - NestJS의 에러 필터(`ExceptionFilter`)는 데코레이터를 통해 특정에러를 포착하기에
    if, else와 같은 구분이 필요없다. 때문에 응답까지 반드시 마무리 짓게끔 설게된것이 아닌가 싶다

    ```ts
    // http-exception.filter.ts
    import {
      ExceptionFilter,
      Catch,
      ArgumentsHost,
      HttpException,
    } from '@nestjs/common';
    import { Request, Response } from 'express';
    
    @Catch(HttpException)
    export class HttpExceptionFilter implements ExceptionFilter {
      constructor(private readonly token: string) {
        console.log(this.token);
      }
      catch(exception: HttpException, host: ArgumentsHost) {
        const ctx = host.switchToHttp();
    
        const response = ctx.getResponse<Response>();
        const request = ctx.getRequest<Request>();
        const status = exception.getStatus();
        const res = {
          statusCode: status,
          timestamp: new Date().toISOString(),
          path: request.url,
        };
        console.log(res);
        response.status(status).json(res);
      }
    }
    
    // all-exceptions.filter.ts
    import {
      ExceptionFilter,
      Catch,
      ArgumentsHost,
      HttpException,
      HttpStatus,
    } from '@nestjs/common';
    import { HttpAdapterHost } from '@nestjs/core';
    
    @Catch()
    export class AllExceptionsFilter1 implements ExceptionFilter {
      constructor(private readonly httpAdapterHost: HttpAdapterHost) {}
    
      catch(exception: unknown, host: ArgumentsHost): void {
        console.log(1111);
        // In certain situations `httpAdapter` might not be available in the
        // constructor method, thus we should resolve it here.
        const { httpAdapter } = this.httpAdapterHost;
        const ctx = host.switchToHttp();
    
        const httpStatus =
          exception instanceof HttpException
            ? exception.getStatus()
            : HttpStatus.INTERNAL_SERVER_ERROR;
    
        const responseBody = {
          statusCode: httpStatus,
          timestamp: new Date().toISOString(),
          path: httpAdapter.getRequestUrl(ctx.getRequest()),
        };
    
        httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
      }
    }
    
    @Catch()
    export class AllExceptionsFilter2 implements ExceptionFilter {
      constructor(private readonly httpAdapterHost: HttpAdapterHost) {}
    
      catch(exception: unknown, host: ArgumentsHost): void {
        console.log(222);
        // In certain situations `httpAdapter` might not be available in the
        // constructor method, thus we should resolve it here.
        const { httpAdapter } = this.httpAdapterHost;
        const ctx = host.switchToHttp();
        const httpStatus =
          exception instanceof HttpException
            ? exception.getStatus()
            : HttpStatus.INTERNAL_SERVER_ERROR;
    
        const responseBody = {
          statusCode: httpStatus,
          timestamp: new Date().toISOString(),
          path: httpAdapter.getRequestUrl(ctx.getRequest()),
        };
    
        httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
      }
    }
    
    ```

    ```ts
    //main.ts
    import { NestFactory } from '@nestjs/core';
    import { AppModule } from './app.module';
    import { AllExceptionsFilter } from './common/exception/all-exceptions.filter';
    
    async function bootstrap() {
      const app = await NestFactory.create(AppModule);
      app.useGlobalFilters(new AllExceptionsFilter(1));
      app.useGlobalFilters(new AllExceptionsFilter(2));
      app.useGlobalFilters(new AllExceptionsFilter(3));
      await app.listen(3000);
    }
    bootstrap();
    
    // all-exceptions.filter.ts
    import {
      ExceptionFilter,
      Catch,
      ArgumentsHost,
      HttpException,
      HttpStatus,
    } from '@nestjs/common';
    
    import { Request, Response } from 'express';
    
    @Catch()
    export class AllExceptionsFilter implements ExceptionFilter {
      constructor(private id: number) {}
      catch(exception: unknown, host: ArgumentsHost): void {
        console.log(this.id);
        // In certain situations `httpAdapter` might not be available in the
        // constructor method, thus we should resolve it here.
        const ctx = host.switchToHttp();
        const request = ctx.getRequest<Request>();
        const response = ctx.getResponse<Response>();
        // eslint-disable-next-line @typescript-eslint/ban-ts-comment
        //@ts-ignore
        const status = exception.getStatus();
        const httpStatus =
          exception instanceof HttpException
            ? exception.getStatus()
            : HttpStatus.INTERNAL_SERVER_ERROR;
    
        const responseBody = {
          statusCode: httpStatus,
          timestamp: new Date().toISOString(),
          path: request.url,
        };
    
        response.status(status).json(responseBody);
      }
    }
    
    ```

## Inheritance

---

there might be use-cases when you would like to simply extend the built-in default **global exception filter**, and override the behavior based on certain factors.

In order to delegate exception processing to the base filter, you need to extend `BaseExceptionFilter` and call the inherited `catch()` method.

```ts
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

`BaseExceptionFilter` 를 상속하는 `Method-scoped` and `Controller-scoped` filters는 new 와함께 인스턴화되어선 안된다. nest가 자동으로하게 두어라

Global filters **can** extend the base filter. This can be done in either of two ways.

- to inject the `HttpAdapter` reference when instantiating the custom global filter:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

- 위에 언급했던 전역으로만드는 다른방법.
