---
title: "[NestJS | Docs | Overview] Modules"
categories: [NestJS]
tags: [NestJS]
image: nestJSLogo.png
---

## Modules

---

![Desktop View](/2024-06-11-NestJSDocs-Overview-Modules/module.png){: width="400"  }

Nest에서 module이란 `@Module()` 데코레이터가 붙은 클래스를 말한다.<br>
`@Module()` 데코레이터가 제공하는 metadata 사용해 nest는 어플리케이션의구조를 조직화한다.<br>
각 애플리케이션은 적어도 하나의 module(root)을 가지며,  **`application graph`**의 시작점이 된다.

>📌 **`application graph`** <br>
> `the internal data structure Nest uses to resolve module and provider relationships and dependencies.`<br>
> 모듈과 프로바이더의 관계(relationships) 또 의존성(dependency)을 해결하는  내부 데이터

`@Module()` 데코레이터는 아래 property들을 가진 단일 객체를 사용한다

- `providers` : nest injector의해 인스턴스화 되고, 적어도 이 모듈 전체에 공유되는 프로바이더
- `controllers` :  모듈내에서 정의되는 컨트롤러의 집합
- `imports` : 해당모듈에서 필요로하는 프로바이더를 export하는 타모듈의 모음
- `exports` : 해당 모듈에서 제공하고 다른 모듈에서 필요로 하는 프로바이더의 집합. <br>
   > `You can use either the provider itself or just its token (provide value).`<br>
   > 프로바이더를 export할 때 프로바이더 자체 또는 프로바이더를 식별하는 토큰을 내보낼 수 있다. <br>
   > 토큰은 일반적으로 클래스 이름이지만, 문자열이나 심볼(symbol)로도 정의될 수 있다. <br>
   > import해온 module을 그대로 export 할 수 있고 이는 import해온 모듈의 모든 provider를 export 하는것과 같다.
<br>

모듈은 기본적으로 프로바이더를 캡슐화한다.
현재모듈의 프로바이더가 아니거나, 다른 모듈에서 export되어 현재 모듈로 import되지 않은 프로바이더를 주입하는 것은 불가능하다.
export된 provider는 모듈의 public interface or API가 된다.

## Feature modules

---

`CatsController`와 `CatsService`는 같은 애플리케이션 도메인에 속한다.
별도의 `feature module(기능 모듈)`을 만들어 그곳으로 옮기는 것은 매우 합리적이다.

- To create a module using the CLI, `$ nest g module cats`

```ts
// cats/cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

```ts
// app.module.ts
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

![Desktop View](/2024-06-11-NestJSDocs-Overview-Modules/directory-structure.png){: width="400" }

## Shared modules

---

![Desktop View](/2024-06-11-NestJSDocs-Overview-Modules/shared-module.png){: width="500" }

`Nest`에서 모듈은 기본적으로 singleton이기에, <br>
여러모듈에서 같은 프로바이더 인스턴스를 공유할 수 있다

1. we first need to **export** the `CatsService` provider by adding it to the module's `exports` array,
2. Now any module that imports the `CatsModule` has access to the `CatsService` and will share the same instance with all other modules that import it as well.

```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

- **부연설명**

    모듈은 특정 기능이나 기능집합(`Provider`)을 캡슐화하며,<br>
    내부 프로바이더를 노출하려면 반드시 `exports` 배열에 포함시켜야한다.<br>
    그후 해당 모듈을 import해  `Provider`에 접근 가능하게 한다.

    1. some.module  ⇒ export 프로바이더
    2. current.module ⇒ import some module
    3. then dependency injection is available
  
  <br>

- **주의사항**

    한 상황을 가정해보자

    1. some.module  ⇒ export 프로바이더 (이미 provider배열에 들어간)
    2. current.module ⇒ import some module
    3. current.module ⇒ insert some.provider into  providers array again

    위와 같이 특정 프로바이더를 프로바이더배열에 여러번 추가하면, <br>
    이는 **`singleton`**이 깨지는 상황을 초례한다. (여러 인스턴스가 생김) <br>
    즉 provider배열에 들어가는 것은 한번이어야 한다!<br>
    (항상그렇다고는 말못하겠음 아직 배우는 중이라)

    <br>

- **Why import the whole module instead of just the provider?**

    이는 개인적으로 궁금증으로, 프로바이더또한 클래스인데 <br>
    "Nest는 왜 모듈단위의 import를 강요했을까?"에 대한 의문에서 시작해 정리해보았다.
    1. **의존성 문제**

        모듈은 종종 여러 프로바이더와 함께 작동한다.<br>
        특정 프로바이더만 가져올 경우 해당 프로바이더가 의존하고 있는 다른 프로바이더가 누락될 수 있고,
        이는 의존성 주입 실패로 이어질 수 있다.<br>
        (프로바이더의 제어권은 모듈에게있다.)

    2. **모듈 초기화 로직 누락**

        모듈에는 초기화 로직이 포함될 수 있고, 이 로직은 모듈이 처음 로드될 때 실행된다.<br>
        특정 프로바이더만 가져오면 모듈 초기화 로직이 실행되지 않은채로 가져와 질 수 있다.<br>
        (물론 부트스트랩시에 어떤 모듈이 초기화되느냐의 순서에따라 다를 수 있을 거같다.)

    3. **캡슐화 위반**

        모듈은 프로바이더를 캡슐화하여 내부 구현 세부 사항을 숨기도록 설계되었다.<br>
        특정 프로바이더만 가져오면 모듈의 캡슐화 원칙이 깨질 수 있다.
        캡슐화는 내부 구현 세부 사항을 숨기고, 외부에서 접근할 수 있는 인터페이스만 노출하여 추상화하고
        오용을 방지하는 개념이다.<br>
        프로바이더만 import해서 사용하는 것은 캡슐화를 무시하고,
        모듈이 의도한 방식대로 사용하지 않는다는 의미로 해석할 수 있다.

    4. **유지보수 어려움**

        모듈 단위로 의존성을 관리하지 않으면 코드베이스가 커짐에 따라 유지보수가 어려워질 수 있다.
        모듈은 의존성을 한 곳에 모아서 관리하도록 설계되어 있다.<br>
        여러 모듈에서 여러 프로바이더를 개별적으로 가져오면 의존성 구조가 복잡해지고,
        변경 사항이 있을 때마다 여러 곳을 수정해야 할 수 있다.

## Module re-exporting

---

import한 모듈을 또 다시 다른 모듈로 내보내는 것도 가능하다.

```ts
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}

```

- **부연설명**

    모듈을 `exports` 배열에 넣으면 해당 모듈이 가지고 있는 모든 프로바이더를 다시 전부 익스포트하게 된다.
    이를 통해 모듈 간의 종속성을 효율적으로 관리할 수 있다.

- **내가 한번 만들어본 예시**

    `DogsModule`이 `CatsModule`을 사용하고, 이를 다시 익스포트하면, `AppModule`은 `DogsModule`만 임포트하면 된다.
     이는 모듈 간의 의존성을 더 깔끔하게 관리할 수 있게한다.

    ```ts
    // app.module.ts
    import { Module } from '@nestjs/common';
    
    import { AppController } from './app.controller';
    import { AppService } from './app.service';
    import { DogsModule } from './dogs/dogs.module';
    
    @Module({
      imports: [DogsModule],
      controllers: [AppController],
      providers: [AppService],
    })
    export class AppModule {}
    
    ```

    ```ts
    // app.controller.ts
    import { Controller, Get } from '@nestjs/common';
    import { AppService } from './app.service';
    import { CatsService } from './cats/cats.service';
    
    @Controller()
    export class AppController {
      constructor(
        private readonly appService: AppService,
        private readonly catService: CatsService,
      ) {}
    
      @Get()
      getHello(): string {
        const tmpCatTest = this.catService.findAll();
        console.log(tmpCatTest);
        return this.appService.getHello();
      }
    }
    
    ```

    ```ts
    // dogs.module.ts
    import { Module } from '@nestjs/common';
    import { CatsModule } from 'src/cats/cats.module';
    import { DogsController } from './dogs.controller';
    import { DogsService } from './dogs.service';
    
    @Module({
      imports: [CatsModule],
      providers: [DogsService],
      controllers: [DogsController],
      exports: [CatsModule],
    })
    export class DogsModule {}
    
    ```

    ```ts
    //dog.controller.ts
    import { Controller } from '@nestjs/common';
    import { CatsService } from 'src/cats/cats.service';
    import { DogsService } from './dogs.service';
    
    @Controller('dogs')
    export class DogsController {
      constructor(
        private catsService: CatsService,
        private dogsService: DogsService,
      ) {}
    }
    
    ```

    ```ts
    // cats.modules.ts
    import { Module } from '@nestjs/common';
    import { CatsController } from './cats.controller';
    import { CatsService } from './cats.service';
    
    @Module({
      controllers: [CatsController],
      providers: [CatsService],
      exports: [CatsService],
    })
    export class CatsModule {}
    
    ```

## Dependency injection

---

controller에 service(provider)를 주입했듯이 모듈도 클래스이므로 provider를 주입 할 수 있다.
모듈에 모듈을 주입하는것은  [**circular dependency**](https://docs.nestjs.com/fundamentals/circular-dependency) 이기때문에 안됨

```ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

- **부연설명**

    여기서 내가 헷갈렷던점은
    1. `@Module` 데코레이터의 `Providers`에 `CatService`를 넣는것과
    2. `module class`의 `constructor`에 dependency를 주입하는것

    두가지의 차이가 와닿지 않았다. 둘다 클래스를 만들고 할당하는 과정이 반복되는 것이아닌가 하고 착각했다.<br>
    부트스트랩과정에서  Nest는 프로바이더들을 인스턴스화하고 모듈의 컨텍스트에 등록한다.
    또한  `constructor(private catsService: CatsService) {}`와같이 의존성을 주입할 곳을 찾아
    인스턴스화했던 클래스를 주입해 준다.
    즉 인스턴스가 만들어지는 과정은 한번이고 인스턴스를 만들고 의존성으로 주입해주는 것또한 nest가 해주는것이다.

## Global modules

---

만약 같은 모듈을 모든 곳에서 불러와야 한다면 ??
**`@Global()`** 데코레이터를 사용하여 모듈을 **전역적으로** 설정하는 것이 가능하다

```ts
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

- The `@Global()` decorator makes the module global-scoped.
- Global modules should be registered **only once**, generally by the root or core module.<br>
  => root module에 등록 꼭해야함(import)<br>
`CatsModule`은 전역 모듈로 , `CatsService provider`를 필요로 하는 모듈은  CatsModule을 import 하지 않아도 된다.

- Making everything global is not a good design decision.
- Global modules are available to reduce the amount of necessary boilerplate.

## Dynamic modules

---

> easily create customizable modules that can register and configure providers dynamically<br>
`Dynamic modules`이란 프로바이더를 동적으로 등록하고 설정할 수 있는 커스터마이징 가능한 모듈을 말한다

- Dynamic modules are covered extensively [**here**](https://docs.nestjs.com/fundamentals/dynamic-modules).
- In this chapter, we'll give a brief overview to complete the introduction to modules.

`DatabaseModule` 예시를 살펴보자

```ts
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}

```

`forRoot()` 는 동기 또는 비동기로 dynamic module을 리턴한다.(이 때문에 프로바이더가 동적으로 등록 할 수 있다는 의미인거 같다)
해당 모듈은  default로 Connection provider를 정의하지만,(in the `@Module()` decorator metadata)
forRoot() 메서드에 전달된 *entities,* *options*에 따라 repository(collection of providers) 제공한다.<br>

`DynamicModule`에 의해 반환되는 properties는 `@Module()`데코레이터에 정의되는 기본 모듈의 메타데이터를 확장`extend`한다(`override X`).

 해당 `DynamicModule` 을 전역으로 등록하고 싶다면 아래와 같이 작성할 수 있다(root module에 따로 import 안해줘도 됨)

```ts
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

다른방법

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

If you want to in turn re-export a dynamic module,

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```
