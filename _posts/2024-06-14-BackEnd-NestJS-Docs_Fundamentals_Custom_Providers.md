---
title: "[NestJS | Docs | Fundamental] Custom providers"
categories: [BackEnd,NestJS]
tags: [NestJS]
image: nestJSLogo.png
---
So far, we've only explored one main pattern(Constructor based). As your application grows more complex, you may need to take advantage of the full features of the DI system, so let's explore them in more detail.

## **DI fundamentals**

---

의존성 주입은 의존성의 인스턴스화를 `Nest js runtime system`에 위임하는 `inversion of control(IoC) technique`이다. 아래 예시에서 어떤 일이 일어나는지 자세들여다 보자

1. `provider`를 정의한다. `@Injectable()`데코레이터는 `Catservice class`를 `provider`로 만든다.

    ```tsx
    //cat.service.ts
    import { Injectable } from '@nestjs/common';
    import { Cat } from './interfaces/cat.interface';
    
    @Injectable()
    export class CatsService {
      private readonly cats: Cat[] = [];
    
      findAll(): Cat[] {
        return this.cats;
      }
    }
    ```

2. Nest가 `provider(Catservice)`를 `Controller class`에 주입하기를 요청한다.(constructor)

    ```tsx
    // cat.controller.ts
    import { Controller, Get } from '@nestjs/common';
    import { CatsService } from './cats.service';
    import { Cat } from './interfaces/cat.interface';
    
    @Controller('cats')
    export class CatsController {
      constructor(private catsService: CatsService) {}
    
      @Get()
      async findAll(): Promise<Cat[]> {
        return this.catsService.findAll();
      }
    }
    ```

3. 프로바이더를 `Nest Ioc container`에 등록,연결한다.

    ```tsx
    //cat.module.ts
    import { Module } from '@nestjs/common';
    import { CatsController } from './cats/cats.controller';
    import { CatsService } from './cats/cats.service';
    
    @Module({
      controllers: [CatsController],
      providers: [CatsService],
    })
    export class AppModule {}
    ```

There are three key steps in the process:

1. `cats.service.ts`에,  `@Injectable()` decorator가 `CatsService` class는  Nest IoC container에의해 관리된다고 선언(declare)한다.
2. `cats.controller.ts`의 `CatsController`는 constructor주입과함께 Catservice토큰에 대한 종속성을  선언한다
3. `app.module.ts` 에서, `token Catservice` 로 class CatseSevice(cat.service.ts)를 등록했다
    - NestJS에서 `token`은 DI (Dependency Injection) 시스템에서 의존성을 식별하는 데 사용되는 키 또는 식별자이다.

`Nest Ioc container`가 `CatsController`를 초기화 할때 첫째로 의존성을 먼저  찾는다. `CatsService` 종속성을 찾으면 등록단계(3)에 따라 `CatsService class`를 반환하는 `CatsService` 토큰에 대한 조회를 수행한다. `Singletone(default scope`을 가정하면 Nest는 CatsServie의 인스턴스를 생성하고 캐시한 후 반환하거나 이미 캐시된 경우 기존 인스턴스를 반환한다.

One key feature is that dependency analysis (or `creating the dependency graph`), is **transitive**.<br>
( **transitive**전이적이다⇒의존성 분석이 직접적읜 의존성에만 국한되지 않고, 간접적인 의존성까지도 포함한다는 의미 )

즉 CatsService자체에 종속성이 있어도 종속성은 해결된다. `dependency graph`는 종속성이 올바른 순서(bottom up: 상향식)로 해결되도록 보장한다. 이 메커니즘을 사용하면 개발자가 복잡한 종속성 그래프를 관리할 필요가 없다.

## **Standard providers**

---

Let's take a closer look at the `@Module()` decorator. In `app.module`, we declare:

```tsx
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

the syntax `providers: [CatsService]` is short-hand for the more complete syntax:

```tsx
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

## **Custom providers**

---

when your requirements go beyond those offered by *Standard providers??*

- You want to create a custom instance instead of having Nest instantiate (or return a cached instance of) a class
- You want to re-use an existing class in a second dependency
- You want to override a class with a mock version for testing

Nest allows you to define Custom providers to handle these cases.

## **Value providers: `useValue`**

---

아래와 같은경우 사용 할 수 있다.

1. injecting a constant value
2. putting an external library into the Nest container
3. replacing a real implementation with a mock object for test

3번째를 목적 예시를 살펴보자

```tsx
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}

```

the `CatsService` token will resolve to the `mockCatsService` mock object.   you can use any object that has a compatible interface, including a literal object or a class instance instantiated with `new`.

## **Non-class-based provider tokens**

---

So far, we've used class names as our provider token(Provider배열에서).

아래처럼 provider token으로 JS의 `string`이나 `symbol`을 ,Ts의 `esnums`사용할수도 있다

```tsx
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

- `string-valued token`  :  `‘CONNECTION’`
- `useValue` : pre-existing `connection` object we've imported from an external file

위처럼  `string-valued token`인 `‘CONNECTION’`을 사용할때 어떻게 프로바이더를 주입하는지 살펴보자

```tsx
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

위와 같은 방식으로 `Nest`가 어디에 주입해야하는지 알려주는 셈이다. 위의 예에서는 직접적으로 문자열을 `@Inject()`안에 넣었지만, 깔끔한 코드를 위해 `constant.ts` 와같은 파일을 만들어 변수를 주입해주는게 좋다.

## **Class providers: `useClass`**

---

```tsx
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

we have used the `ConfigService` class name as our token. For any class that depends on `ConfigService`,  Nest will inject an instance of the provided class (`DevelopmentConfigService` or `ProductionConfigService`)

- **dynamically determine a class**

    위의 코드 예시는 특정 `consumer`가 의존성으로 `ConfigService class`를  가지는경우 선택적인 주입이 가능해진다.

- **위 코드예제를 그대로 적용하면 의존성이 해결되지 않는다. 왜인지 정확히 모르겠다**

    ```tsx
    // dogs.module.ts
    import { Injectable, Module } from '@nestjs/common';
    import { CatsModule } from 'src/cats/cats.module';
    import { DogsController } from './dogs.controller';
    import { DogsService } from './dogs.service';
    
    const dogCustomProvider = {
      provide: 'DOG_OPTION',
      useValue: 'this is dog option',
    };
    
    @Injectable()
    export abstract class ConfigService {
      abstract name: string;
    }
    
    class DevelopmentConfigService extends ConfigService {
      name = 'DevelopmentConfigService';
    }
    
    class ProductionConfigService extends ConfigService {
      name = 'ProductionConfigService';
    }
    
    const configServiceProvider = {
      provide: ConfigService,
      useClass:
        process.env.NODE_ENV === 'development'
          ? DevelopmentConfigService
          : ProductionConfigService,
    };
    
    @Module({
      imports: [CatsModule],
      providers: [DogsService, dogCustomProvider, configServiceProvider],
      controllers: [DogsController],
      exports: [CatsModule],
    })
    export class DogsModule {}
    
    ```

    ```tsx
    //dog.controller.ts
    import { Controller, Inject, Optional } from '@nestjs/common';
    import { CatsService } from 'src/cats/cats.service';
    import { DogsService } from './dogs.service';
    import { ConfigService } from './dogs.module';
    
    @Controller('dogs')
    export class DogsController {
      constructor(
        private catsService: CatsService,
        private dogsService: DogsService,
        private configService: ConfigService,
        @Optional() @Inject('DOG_OPTION') private dogOption: string,
      ) {
        console.log(this.dogOption);
        console.log(this.configService.name);
      }
    }
    ```

    `abstract class`가 아닌 실제 `class`로 바꿔줘도 의존성이 해결되지않는다

    의존성을 해결하기위해 아래와같이  `dogs.module.ts`에서`provide` 에 token으로 `string`으로 주고 ,   `dog.controller.ts`에 `@Inject('ConfigService')` 와함께 명시적으로 주입할 의존성에대해 작성하니 의존성이 주입되는것을 확인하긴했다.

    ```tsx
    // dogs.module.ts
    const configServiceProvider = {
      provide: 'ConfigService',
      useClass:
        process.env.NODE_ENV === 'development'
          ? DevelopmentConfigService
          : ProductionConfigService,
    };
    
    //dog.controller.ts
    @Controller('dogs')
    export class DogsController {
      constructor(
        private catsService: CatsService,
        private dogsService: DogsService,
        @Inject('ConfigService') private configService: ConfigService,
        @Optional() @Inject('DOG_OPTION') private dogOption: string,
      ) {
        console.log(this.dogOption);
        console.log(this.configService.name);
      }
    }
    
    ```

    현재로서는 내가 공식문서를 잘못 이해한것인지, 이렇게 쓰는게 맞는건지 잘 모르겠다

    (추후 다시 확인필요)

## **Factory providers: `useFactory`**

---

The `useFactory` syntax allows for creating providers **dynamically**.

The actual provider will be supplied by the value returned from a factory function.

프로바이더 자기자신이 의존성을 가지는경우는 조금 복잡해진다

아래 예시를 살펴보자

`useFactory` 는 어떤 인자를 받고 어떤걸 리턴하는지 설명하고 있다

`inject` 는  어떤것을 주입할건지에 대해 설명하고있다

`useFactory` 함수의 인자와 `inject`배열에 들어가는 provider는 1대1 대응되도록 작성해준다

아래 코드에서는 `useFactory`가 실행되기위해서 첫번째 프로바이더는 필수이고

두번째 프로바이더는 없어도 문제될것 없게 작성되어있다

`@Module`쪽 token을 살펴보면 `connectionProvider`, `OptionsProvider`, 주석 이 작성되어있는데

`connectionProvider` 는 팩토리함수가 리턴하는 프로바이더를 의미하고

OptionsProvider는 팩토리함수가 주입받아야하는 프로바이더를 의미한다.

```tsx

const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \_____________/            \__________________/
  //        This provider              The provider with this
  //        is mandatory.              token can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}

```

## **Alias providers: `useExisting`**

---

`useExisting`을 이용하면 기존 존재하는 프로바이더에 별칭을 만들어 같은 프로바이더에 접근하는 방식을 두개만들 수 있다.

 If both dependencies are specified with `SINGLETON` scope, they'll both resolve to the same instance.

```tsx
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}

```

## **Non-service based providers**

---

providers는 종종 service로 적용되지만, 해당용도에만 국한되지는 않는다.

```tsx
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}

```

## **Export custom provider**

다른 프로바이더와 마찬가지로 , custom provier또한 선언된 모듈에 제한된다.

다른 모듈에서 사용되려면 export되어야하고  두가지 방식 모두 가능하다 (export부분 token 정의방식이 다름)

```tsx
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

```tsx
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```
