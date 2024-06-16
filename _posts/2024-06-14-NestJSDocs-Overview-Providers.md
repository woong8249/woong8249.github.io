---
title: "[NestJS | Docs | Overview] Providers"
categories: [NestJS]
tags: [NestJS]
image: nestJSLogo.png
---


## **Providers**

---

<u>Providers are a fundamental concept in Nest.</u>

Nest classes may be treated as a provider (ex `services`,`repositories`,`factories`,`helper`.. so on) , 프로바이더는 의존성으로서 주입될 수 있다. 때문에 객체간 서로 다양한 관계뢰 연결 될 수있으며  객체 주입은 Nest런타임 시스템에게 위임 될수 있다.
![https://docs.nestjs.com/assets/Components_1.png](https://docs.nestjs.com/assets/Components_1.png)

위 그림의 `Component`를  `class`, 화살표를 의존성으로 매칭해 생각하면 된다. controller가 http request,response에 관련된 것을 다루고 ,복잡한 task는 Providers(service)에게 위임하듯
의존성 주입이 가능한 형태로 역할을 분담하는데 이것이 `Provider`의 목적이다(관심사 분리).

## **Services**

---

- To create a service using the CLI,
  - `$ nest g service cats`
- `@Injectable()` 데코레이터가 붙은 class는  Nest의 `Ioc continer`에 의해 관리된다.
- 아래는 CatService라는 프로바이더를 Contorller에 의존성 주입해주는 예시다.

```ts
// interfaces/cat.interface.ts
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

```ts
//cats.service.ts
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

```ts
// cats.controller.ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

The `CatsService` is **injected** through the class constructor.  Notice the use of the `private` syntax in constructor. This shorthand allows us to both declare and initialize the `catsService` member immediately in the same location.

## **Dependency injection**

---

Nest는 `Dependency injection`라 불리는 디자인 패턴으로 지어진다.

```ts
constructor(private catsService: CatsService) {}
```

Nest will resolve the `catsService` by creating and returning an instance of `CatsService`(or, in the normal case of a singleton, returning the existing instance if it has already been requested elsewhere)

@Injectable 데코가 붙은 class CatService를 @Controller 데코가 붙은 class CatsController constructor에 주입해줌

## **Scopes**

---

When the application is bootstrapped, every dependency must be resolved, and therefore every provider has to be instantiated. Similarly,

However, there are ways to make your provider lifetime **request-scoped** as well. You can read more about these techniques [**here**](https://docs.nestjs.com/fundamentals/injection-scopes).

## **Custom providers**

---

nest는 프로바이더간 관계를 해결해주는 빌트인IOC(inversion of control)를 가지고있다. <br>
정의하는 다양한 방법이 있다.[**Custom Provider**](/posts/NestJSDocs-Fundamentals-Custom-Provider/)

## **Optional providers**

---

가끔, 반드시 해결해야 할 필요는 없는 종속성이 있을 수 있다.

특정 class가 `configuration object`에대 의존성을 가진지만, `configuration object`가 pass되지 않은때는  `default value`가 사용되는경우, 해당경우는 configuration provider가 없어도 오류가 발생하지 않으므로 dependency는 optional이 된다.

To indicate a provider is optional, use the `@Optional()` decorator in the constructor's signature.

```ts
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

- **@Injectable() 데코레이터**

    `@Injectable()` 데코레이터가 있으니 , 이 클래스는 Nest의 IoC (Inversion of Control) 컨테이너에 의해 관리된다. 모듈의 프로바이더로 등록되어 있다면, Nest는 `HttpService`를 의존성으로 사용하는 곳에 `HttpService`의 인스턴스를 주입해준다.

- **@Inject('HTTP_OPTIONS')와 @Optional() 데코레이터**

    `@Inject('HTTP_OPTIONS')`는 커스텀 토큰 `HTTP_OPTIONS`로 식별되는 의존성을 주입하기 위해 사용되다. `HttpService` 클래스가 인스턴스화될 때, Nest는 `httpClient`에 `HTTP_OPTIONS`로 등록된 의존성을 주입합니다. 이 경우, `@Optional()` 데코레이터가 있기 때문에 `HTTP_OPTIONS`가 제공되지 않아도 `HttpService` 인스턴스 생성이 가능하며, `httpClient`는 `undefined`가 될 수 있다.

## **Property-based injection**

---

지금까지 사용해온 기술은 **`constructor-based injection`** 이라고한다.
매우 드물게 `property-based injection`이 유용할 수도 있다.

만약  여러단계의 상속계층을 가지는 `class`들 이있고, Top-Level class에서 여러 의존성을 가진다면 이를 `super()`로 매번 전달해줘야한다.<br>
In order to avoid this, you can use the `@Inject()` decorator at the property level.

```ts
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

If your class doesn't extend another class, you should always prefer using **constructor-based** injection.

생성자에 정의하는게 의존성에대해 명시적이며, 가시성이 좋고 SingleTone이 아니게되는경우에 사이드이펙트에서도 자유롭다.

## **Provider registration**

---

1. defined a provider (Catservice with `@injectable`)
2. defined  a consumer(CatsController using **`constructor-based injection` or `property-based injection`** )
3. we need to register the service with Nest so that it can perform the injection.

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

This is how our directory structure should look now:

```bash
src
├── cats
│   ├── dto
│   │   └── create-cat.dto.ts
│   ├── interfaces
│   │   └── cat.interface.ts
│   ├── cats.controller.ts
│   └── cats.service.ts
├── app.module.ts
└── main.ts

```
