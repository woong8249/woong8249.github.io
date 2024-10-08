---
title: blokcing과 Sync는 결국 같은거 아니야?
categories: [BackEnd,NodeJS]
tags: [node-js]
toc: true
image: /blocking.png 
---


## Intro

---

여지껏 `Blocking === Sync`,  `Non-Blocking === Async` 라고 생각하며 개발을 해왔다.
Node JS 공식문서를 읽으며 뭔가 다른거같다는 느낌을 받아 조사하게되고 해당 포스트를 남긴다

필자는 NodeJS에 익숙하기에 NodeJS 공식문서,MDN ([Overview of Blocking vs Non-Blocking,](https://nodejs.org/en/learn/asynchronous-work/overview-of-blocking-vs-non-blocking) [JavaScript Asynchronous Programming and Callbacks](https://nodejs.org/en/learn/asynchronous-work/javascript-asynchronous-programming-and-callbacks),**[Asynchronous](https://developer.mozilla.org/en-US/docs/Glossary/Asynchronous#in_software_design)**)를 기준으로 글을 설명하겠음.

또한 소히 Blocking &Non-Blocking ,Sync & Async를 일반화해 다루는 포스트들과 비교하며 유의해야할 점을 짚어보겠음.

## [Blocking & Non-Blocking](https://nodejs.org/en/learn/asynchronous-work/overview-of-blocking-vs-non-blocking) in NodeJS

---

> **Blocking** is when the execution of additional JavaScript in the Node.js process must wait until a non-JavaScript operation completes. This happens because the event loop is unable to continue running JavaScript while a **blocking** operation is occurring.
This happens because the event loop is unable to continue running JavaScript while a **blocking** operation is occurring.
In Node.js, JavaScript that exhibits poor performance due to being CPU intensive rather than waiting on a non-JavaScript operation, such as I/O, isn't typically referred to as **blocking**.
>

<blockquote class="prompt-tip">
즉 NodeJS 공식문서에서 말하는 blocking은 메인스레드의 흐름이 자바스크립트가 아닌 작업에 의해 차단된 것을 말한다. (논외지만 eventLoop가 차단 되었음을말함. 이는 메인스레드가 이벤트루프를 동작시키는 주체임 또는 같은격으로바라 볼 수 있음.)

Blocking / Non-blocking 은 메인스레드(eventLoop)가 자바스크립트외의 작업(I/O)에의해 차단되느냐의 여부이다.
cpu집약적인 수행을 하는 자바스크립트 코드는 (I/O작업같은거 말고) blocking이 아니다.
</blockquote>

> Synchronous methods in the Node.js standard library that use libuv are the most commonly used **blocking** operations. Native modules may also have **blocking** methods.
All of the I/O methods in the Node.js standard library provide asynchronous versions,
which are **non-blocking**, and accept callback functions.Some methods also have **blocking** counterparts, which have names that end with `Sync`.
>

- readFileSync ⇒ blocking native module
- readFile ⇒ non-blocking using call back

## [Sync & Async](https://nodejs.org/en/learn/asynchronous-work/javascript-asynchronous-programming-and-callbacks) in NodeJS

---

> Asynchronous means that things can happen independently of the main program flow.
>

NodeJS 공식문서에서 말하는 Async의 의미는 메인플로우와(메인스레드) 독립적으로 일이 발생할수 있음을 의미한다.
즉 NodeJS에서 이벤트 루프에 의해 백그라운드로 오프로딩되어  병렬적으로 만들어지는 플로우(like readFile(path,callback)는 async인 셈이다

## Sync ===  Blocking? & Async === Non- Blocking?

---

언급된바에 의하면 확실히 readFile API는 non-Blocking,Async API이다.
readFileSync API는  메인스레드의 흐름을 막았기때문 blocking이고 병렬적으로 수행되지 않았기때문에 Sync이다. 여기서  Sync ===  Blocking? & Async === Non- Blocking? 이라는 혼동이 온다.<br>
"또한 Sync하며 non-Blokcing함은 또는 Async하며 Blocking함은 양립할수 없는가?"라는 의문이 생긴다.
이 때문에 여러 포스트를 찾아보았지만 Async의 두가지 컨텍스트에 대해 동시에 답현하는 포스트가 잘 보이지 않더라..

[MDN](https://developer.mozilla.org/en-US/docs/Glossary/Asynchronous#in_software_design)에서 다음과 같이말한다

> In computing, the word "asynchronous" is used in two major contexts.>

Async에는 아래와 같은 두가지 컨텍스트가있다.

> Asynchronous communication is a method of exchanging messages in which the sending, receiving, and processing of each message is not dependent on the sending, receipt, or processing of other messages. In asynchronous communication, each party receives and processes messages when convenient or possible to do so, rather than doing so immediately upon receipt. Additionally, messages may be sent without waiting for acknowledgement, with the understanding that if a problem occurs, the recipient will request corrections or otherwise handle the situation.
>

이는 여러 요청 또는 작업이 완료되는 `순서`가 포커스다.<br>
이 관점에서 async 하다는 여러요청이 순차적으로 처리되지 않을 수 있음을 의미한다.

> Asynchronous software design expands upon the concept by building code that allows a program to ask that a task be performed alongside the original task (or tasks), without stopping to wait for the task to complete. When the secondary task is completed, the original task is notified using an agreed-upon mechanism so that it knows the work is done, and that the result, if any, is available.
>

이는 한 요청에서 테스크를 처리함에 있어 다른 워커에게 테스크를 오퍼로딩함을 의미 즉 병렬이 포커스이다.

다시 돌아와 NodeJS에서 말하는 Async의 컨텍스트는 두번째로 생각된다.
Sync와 Async의 비교대상은 메인스레드와 외부스레드가 비교대상이고
Blocking은 메인스래드 내부에서의 두함수가 비교대상이다.

때문에 병렬이라는 컨텍스트로 해석해 Async를 적용한다면
Sync인 경우에는 항상 Blocking,  Async인 경우에는 항상Non- Blocking이며
Sync하며 non-Blokcing함은 또는 Async하며 Blocking함은 양립할수 없다라고 생각할수 있다.

하지만 Async를 여러요청에대한 순서라는 컨텍스트를 적용하면
Sync하다고 Blocking,  Async라고 Non- Blocking 이라 단정지을  수 없다.
즉 Sync하며 non-Blokcing함은 또는 Async하며 Blocking함은 양립할수 있다.

이때는 Async의 비교대상을  메인 스래드 내부의 네스팅된 두함수를 기준으로 생각해 보면된다.

아래는 Sync하며 non-Blokcing한 함수의 예시 스니핏이다.

```ts
//어떤 operation을 fire하고 특정시간동안만 실행시키고 stop시키려합니다.
//또한 operation중에 외부 컨디션을 계속 확인 해야합니다.
//때문에 delayMS를 await없이 async하게 호출시킨뒤
//await pollUntilConditionIsMet()를 통해 blocking하며
//컨디션을 살피는동시에 특정시간만 동작시키는 코드를 작성했습니다.
//(외부 조건을 만족하지못하는경우 delay와 상관없이 stop가능)
//여기서 delayMS는 sync인동시에 non-blocking입니다(외부조건이 정상 이행 된 경우)
async function delayMS(time: number): Promise<void> {
// time만큼 딜레이합니다
}

async function pollUntilConditionIsMet(
condition: () => boolean,
_conditionCheckCycleTime?: number,
_limitTime?: number
): Promise<boolean> {
// 첫번째인자로 들어온 함수를,
// 두번째로 들어온 사이클로 poll합니다.조건을 만족하면 poll을 종료하고,아니면 계속 poll합니다
}

let condition = false;

await fireSomeOperation();
delayMS(10000).then(() => {
condition = true;
});

await pollUntilConditionIsMet(
() => {
if (condition) {
return true;
}
if (anything /*시간외에 또다른 외부요인확인하고싶어서*/) {
return true;
}
return false
},
300,
15000);

stopSomeOperation();
```

결론은 애초에 Async& sync와 blocking & non-blocking은 뜻하는 바가 다르기도하지만
sync&async의 컨텍스트를 어떻게해석하느냐에따라 또다르게 해석이 달라질수도 있는 것 같다.
다른 포스트들을 살펴볼때 어떤 컨텍스트로 말하느냐를 조금 생각해보자!

## Reference

---

[Node.js — Overview of Blocking vs Non-Blocking](https://nodejs.org/en/learn/asynchronous-work/overview-of-blocking-vs-non-blocking)

[Node.js — JavaScript Asynchronous Programming and Callbacks](https://nodejs.org/en/learn/asynchronous-work/javascript-asynchronous-programming-and-callbacks)

[Asynchronous - MDN Web Docs Glossary: Definitions of Web-related terms](https://developer.mozilla.org/en-US/docs/Glossary/Asynchronous)

[Blocking, Non-blocking, Sync, Async 의 차이](https://jh-7.tistory.com/25)

[완벽히 이해하는 동기/비동기 & 블로킹/논블로킹](https://inpa.tistory.com/entry/👩‍💻-동기비동기-블로킹논블로킹-개념-정리)
