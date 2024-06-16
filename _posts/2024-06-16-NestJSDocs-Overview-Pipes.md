---
title: "[NestJS | Docs | Overview] Pipes"
categories: [NestJS]
tags: [NestJS]
image: nestJSLogo.png
---

## Pipes

---

A pipe is a class annotated with the `@Injectable()` decorator, which implements the `PipeTransform` interface.

![https://docs.nestjs.com/assets/Pipe_1.png](https://docs.nestjs.com/assets/Pipe_1.png)

Pipes have two typical use cases:

- `transformation` : input data를 원하는 form으로 변경 (e.g., from string to integer)
- `validation` : 유효한 input data인지 평가, 유효하면 pass it through unchanged, ; otherwise, throw an exception

두경우 모두 pipe는 `controller route handler`(method)에 인자(arg)를 대상으로 동작한다.

Nest는 route handler가 호출되기 직전에 pipe를 삽입해 method로 향하는 `argument`를 받아 작업(validation or transformation)을 수행하고, 그후에 router handler가 호출된다. (with 작업이 수행된 argument)

Nest comes with a number of built-in pipes that you can use out-of-the-box.
You can also build your own custom pipes.

파이프는 exceptions zone 안에서 동작한다. 파이프 작업이 예외를 던지는경우, exception layer(어떤 스코프이든 상관없이)내에서 처리되고,controller method는 실행되지 않는다.

## Build-in Pipes

---

Nest comes with nine pipes available out-of-the-box:  from the `@nestjs/common` package.

| 파이프 이름 | 설명 |
| --- | --- |
| ValidationPipe | 입력 데이터의 유효성을 검사하여 검증합니다. |
| ParseIntPipe | 입력 값을 정수(integer)로 변환합니다. |
| ParseFloatPipe | 입력 값을 부동 소수점 수(float)로 변환합니다. |
| ParseBoolPipe | 입력 값을 불리언(boolean)으로 변환합니다. |
| ParseArrayPipe | 입력 값을 배열(array)로 변환합니다. |
| ParseUUIDPipe | 입력 값을 UUID 형식으로 변환합니다. |
| ParseEnumPipe | 입력 값을 열거형(enum) 값으로 변환합니다. |
| DefaultValuePipe | 입력 값이 없을 때 기본값을 설정합니다. |
| ParseFilePipe | 파일 입력을 처리하고 변환합니다. |

## Binding pipes

---

To use a pipe, we need to bind an instance of the pipe class to the appropriate context.

we'll refer to as binding the pipe at the method parameter level:

```ts
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

위 코드는 `findOne`의 `parameter`로 `number`가 들어오는 것과 그렇지 않으면 `fineOne`이 호출되지 않음을 보장한다.(예외 발생)

For example, assume the route is called like:

`GET localhost:3000/abc`

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}

```

In the example above, we pass a class (`ParseIntPipe`)

 we can instead pass an in-place instance.

Passing an in-place instance is useful if we want to customize the built-in pipe's behavior by passing options:

```ts
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

Binding the other transformation pipes (all of the **Parse*** pipes) works similarly.

```ts
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Here's an example of using the `ParseUUIDPipe` to parse a string parameter and validate if it is a UUID.

```ts
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

Binding validation pipes is a little bit different; we'll discuss that in the following section.

## Custom pipes

---

let's build simple custom versions of each from scratch to see how custom pipes are constructed.

입력 값을 받고 즉시 동일한 값을 반환하도록 해보자

```ts

import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

- The generic interface `PipeTransform<T, R>`
  - `T` :  type of the input value
  - `R` :  type of the return value

- transform(value, metadata)
  - `value` : currently processed method argument
  - `metadata`:  currently processed method argument's metadata.

  ```ts
        export interface ArgumentMetadata {
          type: 'body' | 'query' | 'param' | 'custom';
          metatype?: Type<unknown>;
          data?: string;
        }
  ```

  - **`type`** :  from where(body, query, parm, custom)
  - **`metatype`**:<br>
      the metatype of the argument, for example, `String`.
      `route handler method signature`에 type을 작성하지 않거나 `vanilla JavaScript`를 사용하는 경우 `undefined`이다.

  - **`data`** :<br>
      `decorator`에 전달된 문자열. decorator괄호를 비워둔경우 `undefiend`이다.

TypeScript interfaces disappear during transpilation. Thus, if a method parameter's type is declared as an interface instead of a class, the `metatype` value will be `Object`.

## Schema based validation

---

Let's make our validation pipe a little more useful.

we probably would like to ensure that the post body object is valid before attempting to run our service method.

```ts

@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Let's focus in on the `createCatDto` body parameter. Its type is `CreateCatDto`:

```ts
//create-cat.dto.ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

`create`method로 들어오는 요청이 유효한지(`createCatDto`와 같은지) 검증하는 방법은 여러가지이다.

1. `route handler method` 안에서 이를 수행할수 있지만 이는 단일 책임 원칙(SPR)에 위반된다
2. `validator class` 를 만들어 이를 위임할 수 있다
3. `validator middleware`를 만들어 이를 위임할 수 있다. (하지만 호출될 핸들러와 파라미터에대해 접근이 제한적이다)
4. This is, of course, exactly the `use case for which pipes are designed`. So let's go ahead and refine our validation pipe.

## Object schema validation

---

One common approach is to use **schema-based** validation.

The [**Zod**](https://zod.dev/) library allows you to create schemas in a straightforward way, with a readable API.  Let's build a validation pipe that makes use of Zod-based schemas.

```bash
npm install --save zod
```

we create a simple class that takes a schema as a `constructor` argument.

```ts
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

In the next section, you'll see how we supply the appropriate schema for a given controller method using the `@UsePipes()` decorator.

## Binding validation pipes

---

In this case, we want to bind the pipe at the method call level.

we need to do the following to use the `ZodValidationPipe`:

1. Create an instance of the `ZodValidationPipe`
2. Pass the context-specific Zod schema in the class constructor of the pipe

    ```ts
    import { z } from 'zod';
    
    export const createCatSchema = z
      .object({
        name: z.string(),
        age: z.number(),
        breed: z.string(),
      })
      .required();
    
    export type CreateCatDto = z.infer<typeof createCatSchema>;
    ```

3. Bind the pipe to the method

    ```ts
    @Post()
    @UsePipes(new ZodValidationPipe(createCatSchema))
    async create(@Body() createCatDto: CreateCatDto) {
      this.catsService.create(createCatDto);
    }
    ```

    `zod` library requires the `strictNullChecks` configuration to be enabled in your `tsconfig.json` file.

## Class validator

---

The techniques in this section require TypeScript and are not available if your app is written using vanilla JavaScript.

Nest works well with the [**class-validator**](https://github.com/typestack/class-validator) library. This powerful library allows you to use decorator-based validation.(especially when combined with Nest's **Pipe** capabilities since we have access to the `metatype`)

```bash
npm i --save class-validator class-transformer
```

significant advantage of this technique :

별도의 유효성검사 `class`를 생성할 필요없이 `CreateCatDto class`에 몇가지 데코레이터를 추가해주기만 하면 된다.

```ts

import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

Now we can create a `ValidationPipe` class that uses these annotations.

```ts
//validation.pipe.ts
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

참고로 해당 예제코드처럼 직접 `ValidationPipe`를 구성할 필요도 없다. (nest  built-in `ValidationPipe`가 이미 있고 더많은 기능을 제공한다)
해당 샘플은 `CustomPipe` 메커니즘을 설명하기위한것이다.

You can find full details, along with lots of examples [**here**](https://docs.nestjs.com/techniques/validation).
Let's go through this code.

1. `transform()` method is marked as `async`.

    Nest가  synchronous and **asynchronous** pipes 모두 지원하기때문에 가능하다.

2. we are using destructuring to extract the `metatype` field.
3. the helper function `toValidate()`.

    처리중인 현재 argument가 `javascript`인 경우 유효성 검사 단계를 우회하는일을 담당.

4. we use the class-transformer function `plainToInstance()`

    `JavaScript argument object`를 validation에 적용할수 있는 `typed object`로 변환하기 위해,
    `deserialized network request`(역직렬화 된 네트워크 요청)으로부터 들어오는 `post body object`는
    type 정보를 가지고 있지 않다.(`express`가 이와같은식으로 동작한다)
    `Class-validator` 가 앞서 정의했던 DTO를 유효성검사해야하기떄문에 ,vanilla object를 적절한 decorated object로 바꿔주는 것이다.

5. `return` or `throw exception`

The last step is to bind the `ValidationPipe`

 Pipes can be `parameter-scoped`, `method-scoped`, `controller-scoped`, or `global-scoped`.

```ts
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

## Global scoped pipes

---

`ValidationPipe` 가 generic하게 작성되었기때문에 전역적으로 사용해도 된다

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

In the case of [**hybrid apps**](https://docs.nestjs.com/faq/hybrid-application) the `useGlobalPipes()` method doesn't set up pipes for gateways and micro services.  For "standard" (non-hybrid) microservice apps, `useGlobalPipes()` does mount pipes globally.

- hybrid apps : 한 애플리케이션 내에서 두 가지 이상의 프로토콜을 동시에 사용하는 경우
- gateways : sokect 통신을 하는경우
- micro services : `TCP`, `gRPC`,`Redis`,…

Global pipes are used across the whole application, for every controller and every route handler.

의존성 주입 측면에서, global pipes는 모든 모듈 밖에서 등록되기때문에 (with `useGlobalPipes()`) 의존성주입이 불가하다.  아래와같이 작성해 해당 issue를 해결할 수 있다

```ts

import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

위같은 방식을 사용하는경우, 모듈에 관계없이 해당 파이프는 실제로 global하다.

Choose the module where the pipe is defined.

`useClass` is not the only way of dealing with custom provider registration.

## Transformation use case

---

파이프의 use case는 validation만 있는게 아니다.파이프는 input data를 원하는 format으로  transform하는 기능도 가능하다.
클라이언트로부터 전달되는 데이터가 적절하게 처리되어야하는경우(numeric한 문자열을 정수로 변환, 소문자화 ,일부 필수 데이터누락시 기본값 사용 등) 유용하다.

**Transformation pipes** can perform these functions by interposing a processing function between the client request and the request handler.

Here's a simple `ParseIntPipe` which is responsible for parsing a string into an integer value. (As noted above, Nest has a built-in `ParseIntPipe` that is more sophisticated; we include this as a simple example of a custom transformation pipe).

```ts
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

We can then bind this pipe to the selected param as shown below:

```ts
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

Another useful transformation case : bring  **existing user** entity  from db using an id supplied in the request:

```ts
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

- input : id
- output: UserEntity

## Providing defaults

---

`Parse*` 는 파라미터의 값을 전제한다.

`null`이나 `undefined` 를 받으면 exception을 일으킴

`DefaultValuePipe` 를 통해 기본값을 설정할 수 있음.

```ts
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```
