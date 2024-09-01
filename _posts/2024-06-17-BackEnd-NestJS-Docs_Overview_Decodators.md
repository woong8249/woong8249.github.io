---
title: "[NestJS | Docs | Overview] Custom Decorators"
categories: [BackEnd,NestJS]
tags: [NestJS]
image: nestJSLogo.png
---

Nest is built around a language feature called **decorators**.

Decorators are a well-known concept in a lot of commonly used programming languages, but in the JavaScript world, they're still relatively new. Here's a simple definition:

>
>📌 An ES2016 decorator is an expression which returns a function and can take a target, name and property descriptor as arguments. You apply it by prefixing the decorator with an `@` character and placing this at the very top of what you are trying to decorate. Decorators can be defined for either a class, a method or a property.

decorator는

- 함수를 리턴하는 표현이며  `target`,`name` ,`property descriptor as arguments` 를 가진다.
- `@chracter` 와 함께 장식하고하는것의 바로위에 놓음으로 적용시킬수 있다.
- `class`, `method`, `property`에 적용될 수 있다.

## Param decorators

Nest provides a set of useful **param decorators** that you can use together with the HTTP route handlers.

| @Request(), @Req() | req |
| --- | --- |
| @Response(), @Res() | res |
| @Next() | next |
| @Session() | req.session |
| @Param(param?: string) | req.params / req.params[param] |
| @Body(param?: string) | req.body / req.body[param] |
| @Query(param?: string) | req.query / req.query[param] |
| @Headers(param?: string) | req.headers / req.headers[param] |
| @Ip() | req.ip |
| @HostParam() | req.hosts |

Additionally, you can create your own **custom decorators**. Why is this useful?

nodejs에서 아래와같이 properties를 `request object`에서 추출하는것은 흔한일이다.

이는 결국 `request object` 에 직접 접근을함을 의미한다

```tsx
const user = req.user;
```

you can create a `@User()` decorator and reuse it across all of your controllers.

```tsx
//user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

Then, you can simply use it wherever it fits your requirements.

```tsx
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}

```

만약 위 코드가 특정 `Provider`였다면 관심사분리를 생각해 `req`,`res` 객체에 접근하지 않는게 합리적이다. 또한 `Controller`였다 하더라도이는 해당객체의 접근은 플랫폼의 제한을 둘수 있기때문에 위와같이 `custom decorator` 사용하는것이 좋다.

## Passing data

---

아래 예제처럼 데코레이터의 인자를통해  createParamDecorator의 첫번째 인자로 전달받을 수 있다

```tsx
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

```json

{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

```tsx
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

You can use this same decorator with different keys to access different properties.

`createParamDecorator<T>()` is a generic.

`createParamDecorator<string>((data, ctx) => ...)`

`createParamDecorator((data: string, ctx) => ...)` 와같이 typing을 사용하자

```ts
export declare function createParamDecorator<FactoryData = any, FactoryInput = any, FactoryOutput = any>(factory: CustomParamFactory<FactoryData, FactoryInput, FactoryOutput>, enhancers?: ParamDecoratorEnhancer[]): (...dataOrPipes: (Type<PipeTransform> | PipeTransform | FactoryData)[]) => ParameterDecorator;

export type CustomParamFactory<TData = any, TInput = any, TOutput = any> = (data: TData, input: TInput) => TOutput;
```

## Working with pipes

---

nest의 built-in decorator와 custom decorator의 처리방식이 같기떄문에  buit-in decorator에서 파이프를 사용하듯이 custom decorator에서도 pipe를 사용할 수 있다.(decorator의 인자에 파이프사용하듯)

Moreover, you can apply the pipe directly to the custom decorator:

```tsx

@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
```

Note that `validateCustomDecorators` option must be set to true. `ValidationPipe` does not validate arguments annotated with the custom decorators by default.

## Decorator composition

---

Nest provides a helper method to compose multiple decorators.

For example, suppose you want to combine all decorators related to authentication into a single decorator. This could be done with the following construction:

```tsx
//auth.decorator.ts
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

You can then use this custom `@Auth()` decorator as follows:

```tsx
@Get('users')
@Auth('admin')
findAllUsers() {}

```

4개의 decorator를 하나로 만든셈이다

Auth = SetMetadata + UseGuards + ApiBearerAuth+ ApiUnauthorizedResponse

- The `@ApiHideProperty()` decorator from the `@nestjs/swagger` package is not composable and won't work properly with the `applyDecorators` function.
