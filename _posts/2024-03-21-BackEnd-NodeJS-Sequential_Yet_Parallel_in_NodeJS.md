---
title: "Sequential Yet Parallel: Enhancing Async Processing in Node.js"
categories: [BackEnd,NodeJS]
tags: [NodeJS]
image: node_logo.png 
---

## Synopsis

---

[Node JS 공식문서(Asynchronous flow control)](https://nodejs.org/en/learn/asynchronous-work/asynchronous-flow-control)에 여러작업을 비동기적으로 처리하는 3가지 패턴이 소개되어있다.

> You will be able to perform almost all of your operations with the following 3 patterns:
>
1. **In series:** functions will be executed in a strict sequential order, this one is most similar to **`for`** loops.
2. **Full parallel:** when ordering is not an issue, such as emailing a list of 1,000,000 email recipients.
3. **Limited parallel:** parallel with limit, such as successfully emailing 1,000,000 recipients from a list of 10E7 users.

이중 1번과 2번을 합쳐 순차적이되 병렬적으로 처리함으로 처리속도을 개선한 패턴을 끄적여보았다.

- 원문코드(공식문서 코드)의 비동기작업은  의존성 없이 실행하기위해  `new Promise`와 `setTimeout`,`Math.ramdon`으로 대체 했습니다.
- top level await를 사용하기도합니다.

## In series

---

아래 코드는 비동기 작업의 순서를 보장하기위해 콜백 패턴을 사용한 예시입니다.

```js
async function operation(num) {
  const waitTime = 100 * Math.floor(Math.random() * 10);
  await new Promise((res) => setTimeout(res, waitTime));
  return num;
}

function print(num) {
  console.log(`${num} is done!`);
}

function executeFunctionWithArgs(operation, callback) {
  const { args, func } = operation;
  func(args).then(print).then(callback);
}

function serialProcedure(operation) {
  if (!operation) process.exit(0); // finished
  executeFunctionWithArgs(operation, () => {
    serialProcedure(operations.shift());
  });
}

const operations = new Array(100)
  .fill(null)
  .map((_, index) => ({ func: operation, args: index }));

serialProcedure(operations.shift());
```

보다 간단히  `for` 이용해도 되겠죠?

```js
async function operation(num) {
  const waitTime = 100 * Math.floor(Math.random() * 10);
  await new Promise((res) => setTimeout(res, waitTime));
  return num;
}

function print(num) {
  console.log(`${num} is done!`);
}

const operations = new Array(100)
  .fill(null)
  .map((_, index) => ({ func: operation, args: index }));

for (const op of operations) {
  await op.func(op.args).then(print);
}

```

주의할 점은 아래와같이 `forEach`안에 async를 작성하더라도 이는 **Sequential하게 실행되지 않습니다.**
여기서 `async`와 `awiat`의 적용대상은 forEach 첫번재 인자로 전달된 함수내부입니다.
지금 상황에서 `async`키워드는 있으나 마나죠!

```js
operations.forEach(async (_operation) => {
  await _operation.func(_operation.args).then(print);
});
```

하나의 작업이 끝날때까지 이벤트 루프가 block되기때문에 시간적으로는 너무 손해를 보는 느낌입니다.

## Full parallel

---

순서를 보장하지 않지만 작업을 병렬적으로 처리할 수 있습니다.

위에 언급된 `forEach`를 이용해 작성해보겠습니다.

```js
async function operation(num) {
  const waitTime = 100 * Math.floor(Math.random() * 10);
  await new Promise((res) => setTimeout(res, waitTime));
  return num;
}

function print(num) {
  console.log(`${num} is done!`);
}

const operations = new Array(100)
  .fill(null)
  .map((_, index) => ({ func: operation, args: index }));

operations.forEach((op) => {
  const { args, func } = op;
  func(args).then(print);
});

```

순서가 지켜지지 않지만 일처리 속도는 매우빠르겠죠!

## Sequential Yet Parallel

---

두 패턴의 장점 순서보장, 병렬처리를 동시에 가져가는 패턴을 만들어봅시다

처음 떠오른 방법은 `Promise.all` 이였습니다.

```js
async function operation(num) {
  const waitTime = 100 * Math.floor(Math.random() * 10);
  await new Promise((res) => setTimeout(res, waitTime));
  return num;
}

function print(num) {
  console.log(`${num} is done!`);
}

const operations = new Array(100)
  .fill(null)
  .map((_, index) => ({ func: operation, args: index }));

const promises = operations.map((op) => op.func(op.args));
await Promise.all(promises).then((allDone) =>
  allDone.map((item) => print(item))
);

```

이는 모든 작업이 완료될 때까지 기다려야 한다는 단점이 있습니다. 이에 대한 대안으로, 각 작업이 완료되는 대로 처리할 수 있는 두 번째 방법을 제시합니다.

```js
async function operation(num) {
  const waitTime = 100 * Math.floor(Math.random() * 10);
  await new Promise((res) => setTimeout(res, waitTime));
  return num;
}

function print(num) {
  console.log(`${num} is done!`);
}

const operations = new Array(100)
  .fill(null)
  .map((_, index) => ({ func: operation, args: index }));

const promises = operations.map((op) => op.func(op.args));

// for (const operation of promises) {
//   await operation;
//   operation.then(print);
// }

for await (const operation of promises) {
  print(operation);
}

```
