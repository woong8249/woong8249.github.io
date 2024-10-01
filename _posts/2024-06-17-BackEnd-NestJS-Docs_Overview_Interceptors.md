---
title: "[NestJS | Docs | Overview] Interceptors"
categories: [BackEnd,NestJS]
tags: [nest-js]
image: nestJSLogo.png
---


`interceptor` =   `@Injectable()` + `NestInterceptor interface`

![https://docs.nestjs.com/assets/Interceptors_1.png](https://docs.nestjs.com/assets/Interceptors_1.png)

Interceptors have a set of useful capabilities which are inspired by the [**Aspect Oriented Programming**](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) technique.

- bind extra logic before / after method execution

    ⇒ `handler` 호출 전,후로 추가로직을 바인딩수 있음

- transform the result returned from a function

    ⇒ `handler` 호출후 리턴된 result를 변형할 수 있다

- transform the exception thrown from a function

    ⇒ `handler` 에서 던져진 exception을 바꿀수 있다.

- extend the basic function behavior

    ⇒ `hanlder` 의 기본 동작을 확장할수 있다.

- completely override a function depending on specific conditions (e.g., for caching purposes)

    ⇒ `hanlder`를 특정 조건에따라 완전히 덮어쓸 수있다

즉 `Route handler`의 호출 전,후로 끼어들어 조작이 가능하다 .

- AOP

    `Aspect-Oriented Programming (AOP)`은 프로그램의 기능을 모듈화하는 프로그래밍 패러다임 중 하나로 . 특히, 특정한 기능(`횡단 관심사`)을 애플리케이션의 핵심 비즈니스 로직과 분리하여 코드의 모듈성을 향상시키고 유지보수성을 높이는 데 중점을 둡니다.

    자세한건 링크 [**Aspect Oriented**](https://en.wikipedia.org/wiki/Aspect-oriented_programming)

## Basics

---

```tsx
export interface NestInterceptor<T = any, R = any> {
    /**
     * Method to implement a custom interceptor.
     *
     * @param context an `ExecutionContext` object providing methods to access the
     * route handler and class about to be invoked.
     * @param next a reference to the `CallHandler`, which provides access to an
     * `Observable` representing the response stream from the route handler.
     */
    intercept(context: ExecutionContext, next: CallHandler<T>): Observable<R> | Promise<Observable<R>>;
}
```

Each interceptor implements the `intercept()` method, which takes two arguments.

1. `ExecutionContext` instance  (exactly the same object as for [**guards**](https://docs.nestjs.com/guards))

    `ExecutinCOntext`는 `ArgumentHost` 를 상속한다

    We saw `ArgumentsHost` before in the exception filters chapter.

    There, we saw that it's a wrapper around arguments that have been passed to the original handler, and contains different arguments arrays based on the type of the application.

    - **Original Handler**:  Controller method
    - Arguments : Controller method에 전달되는 인수 (`request`,`response`…)
    - Wrapper : 감싼다 ⇒ 포함한다.
    - type of application ⇒ 어떤 프로토콜을 사용하는지에따라

## **Execution context**

---

`ArgumentsHost`는 원래의 핸들러에 전달된 인수들을 감싸는 객체입니다.

다음과 같은 주요 메서드를 제공합니다:

- `getArgs()`: 모든 인수들을 배열로 반환합니다.
- `getArgByIndex(index: number)`: 특정 인덱스의 인수를 반환합니다.
- `switchToHttp()`: HTTP 요청 및 응답 객체에 접근할 수 있는 객체를 반환합니다.
- `switchToRpc()`: RPC 데이터에 접근할 수 있는 객체를 반환합니다.
- `switchToWs()`: WebSocket 데이터에 접근할 수 있는 객체를 반환합니다.
- `getType<TContext extends string = ContextType>(): TContext`: 애플리케이션의 타입을 반환합니다 (예: 'http', 'ws', 'rpc').

By extending `ArgumentsHost`, `ExecutionContext` also adds several new helper methods that provide additional details about the current execution process.

주로 가드(Guards)와 인터셉터(Interceptors)에서 사용됩니다.

- `getClass<T = any>(): Type<T>`: 현재 실행 중인 컨트롤러 클래스를 반환합니다.
- `getHandler(): Function`: 현재 실행 중인 핸들러(메서드)를 반환합니다.

Learn more about `ExecutionContext`[**here**](https://docs.nestjs.com/fundamentals/execution-context).

## Call handler

---

```tsx
export interface CallHandler<T = any> {
    /**
     * Returns an `Observable` representing the response stream from the route
     * handler.
     */
    handle(): Observable<T>;
}
```

The second argument is a `CallHandler`.

 The `CallHandler` interface implements the `handle()` method,

`interceptor`안에서 `handle()`을 통해 route handler method를 호출할 수 있다

handle을 호출하지 않으면 ,the route handler method won't be executed at all.

- `intercept()` method effectively **wraps** the request/response stream.
- As a result, you may implement custom logic **both before and after** the execution of the final route handler.
- you can write code in your `intercept()` method that executes **before** calling `handle()`
- Because the `handle()` method returns an `Observable`, we can use powerful [**RxJS**](https://github.com/ReactiveX/rxjs) operators to further manipulate the response.

     the response stream is received via the `Observable`, additional operations can be performed on the stream, and a final result returned to the caller.

`AOP`(Aspect oriented programming)에서  hndle()을 호출하는것이 `pointcut` 이라 불린다.

이는 추가적인 로직이 더해지는 포인트를 의미한다.

## Aspect interception

---

The first use case :

 to log user interaction, We show a simple `LoggingInterceptor` below:

```tsx
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}

```

- Interceptors, like controllers, providers, guards, and so on, can **inject dependencies** through their `constructor`.

## Binding interceptors

---

In order to set up the interceptor, we use the `@UseInterceptors()` decorator imported from the `@nestjs/common` package.

Like [**pipes**](https://docs.nestjs.com/pipes) and [**guards**](https://docs.nestjs.com/guards), interceptors can be controller-scoped, method-scoped, or global-scoped.

```tsx
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

When someone calls the `GET /cats` endpoint, you'll see the following output in your standard output:

```bash
Before...
After... 1ms
```

Note that we passed the `LoggingInterceptor` class

we can also pass an in-place instance:

```tsx
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

If we want to restrict the interceptor's scope to a single method, we simply apply the decorator at the **method level**.

In order to set up a global interceptor, we use the `useGlobalInterceptors()` method of the Nest application instance:

```tsx
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

global interceptors registered from outside of any module (with `useGlobalInterceptors()`, as in the example above) cannot inject dependencies since this is done outside the context of any module.

 In order to solve this issue, you can set up an interceptor **directly from any module** using the following construction:

```tsx

import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

 regardless of the module where this construction is employed, the interceptor is, in fact, global.

## Response mapping

we can easily mutate it using RxJS's `map()` operator.

The response mapping feature doesn't work with the library-specific response strategy (using the `@Res()` object directly is forbidden).

```tsx

//transform.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

Nest interceptors work with both synchronous and asynchronous `intercept()` methods.

imagine we need to transform each occurrence of a `null` value to an empty string `''`. We can do it

```tsx

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}

```

## **Exception mapping**

Another interesting use-case is to take advantage of RxJS's `catchError()` operator to override thrown exceptions:

```tsx

import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}

```

## Stream overriding

---

There are several reasons why we may sometimes want to completely prevent calling the handler and return a different value instead.

ex 1.  to implement a cache to improve response time

Let's take a look at a simple **cache interceptor** that returns its response from a cache.

```tsx
//cache.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

The key point to note is that we return a new stream here, created by the RxJS `of()` operator, therefore the route handler **won't be called** at all.

In order to create a generic solution, you can take advantage of `Reflector` and create a custom decorator. The `Reflector` is well described in the [**guards**](https://docs.nestjs.com/guards) chapter

## More operators

---

Imagine you would like to handle **timeouts** on route requests.

```tsx
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```
