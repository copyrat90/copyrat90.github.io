---
layout: post
title: (C++20) Coroutines (WIP)
tags: [C++]
author: copyrat90
last_modified_at: 2025-10-13T17:33:00+09:00
---

C++20 표준에 도입된 코루틴은 대체 뭐고, 어떻게 쓰는가?

본 글에서는, 우리가 일반적으로 쓰던 함수를 서브루틴<sub>Subroutines</sub>이라고 부르고, 이 글에서 다루고자 하는 대상을 코루틴<sub>Coroutines</sub>,\
둘 모두를 아우르거나 보다 포괄적으로 말할 때는 함수<sub>Function</sub>라는 표현을 쓰도록 하겠다.

내용은 [Lewis Baker - Asymmetric Transfer](https://lewissbaker.github.io) 블로그 글들을 그대로 옮기거나 요약한 내용이다.

* TOC
{:toc}


# 코루틴 이론

우선 코루틴이 뭔지부터 정리해보자.

코루틴<sub>Coroutines</sub>은, 서브루틴<sub>Subroutines</sub>이 지원하는 호출<sub>Call</sub>과 반환<sub>Return</sub> 기능 뿐만 아니라,\
추가로 중단<sub>Suspend</sub>, 재개<sub>Resume</sub>, 파괴<sub>Destroy</sub>를 지원하는 좀 더 일반화된 함수라고 할 수 있다.

다시 말해, 코루틴은 자기가 원하는 시점에 함수를 일시 정지한 후, 호출자<sub>Caller</sub>(또는 재개자<sub>Resumer</sub>)에게 제어를 돌려주는<sub>transfers execution back</sub> 기능을 추가로 제공하는 함수이다.


## 활성 프레임<sub>Activation frame</sub>

활성 프레임<sub>Activation frame</sub>이란, 함수의 현재 실행 상태를 저장하고 있는 메모리이다.\
대표적으로, 함수의 매개 변수<sub>parameters</sub>와 지역 변수<sub>local variables</sub>, 반환 주소<sub>return-address</sub>가 여기에 저장된다.

서브루틴에서는, 피호출자<sub>Callee</sub>가 호출자<sub>Caller</sub>에게 제어를 돌려주려면 함수를 끝내고 반환<sub>Return</sub>하는 방법 밖에 없다.\
이 말은, 피호출자<sub>Callee</sub> 활성 프레임의 생명 주기<sub>lifetime</sub>는 호출자<sub>Caller</sub>에 완벽히 종속된다는 의미이다.\
그래서 서브루틴의 활성 프레임은 LIFO 구조의 스택에 온전히 저장할 수 있다.

하지만 코루틴에서는, 피호출자가 자신의 실행을 일시 중단하고 호출자(또는 재개자)에게 제어를 돌려줄 수 있다.\
이 말은, 피호출자가 중단되더라도 그 활성 프레임은 살아있어야 한다는 의미이다. (경우에 따라 호출자가 반환된 이후까지 살아있을 수도 있음)\
그래서 코루틴의 활성 프레임은 스택에 온전히 저장될 수는 없다.

활성 프레임이 스택에 저장될 때, 이를 흔히 스택 프레임<sub>Stack frame</sub>이라고 부른다.\
스택 프레임을 할당/해제하는 방법은 간단하다. 그저 CPU 내 스택 포인터<sub>stack pointer</sub> 레지스터를 프레임 크기만큼 올리고/내리고 하면 된다.

코루틴의 경우, 활성 프레임은 두 부분으로 나뉘는데:
1. **스택 프레임**: 재개 이후에는 재사용되지 않아서 중단 시 제거되어도 상관없는 부분
1. **코루틴 프레임**: 재개 이후에 재사용돼서 중단 시에도 살아있어야 하는 부분

적어도 후자는 `operator new`로 할당된 공간에 저장된다.\
(테스트해봤는데 GCC 기준으로는 전자도 `operator new`로 할당된 공간에 몰아서 저장되는 것으로 보임)\
즉 기본적으로 스택에 저장되지 않는다는 것이며, 이를 stackless 코루틴이라고 부른다.


## 서브루틴의 호출<sub>Call</sub> 명령

함수가 다른 서브루틴을 호출할 때, 호출자는 자기 자신을 정지시킬 준비를 해야 한다.

우선 현재 CPU 레지스터의 값들을 나중에 복원하기 위해 메모리에 저장해야 한다.\
[호출 규약<sub>Calling convention</sub>](https://en.wikipedia.org/wiki/Calling_convention)에 따라 이걸 호출자가 할지, 피호출자가 할지가 달라진다.

그리고 호출자는 매개변수들을 새로운 활성 프레임에 저장해, 피호출자가 접근할 수 있도록 한다.\
또한 호출자는 자신이 재개될 주소를 새 활성 프레임에 저장하고, 제어를 피호출자 측으로 넘긴다.

## 서브루틴의 반환<sub>Return</sub> 명령

피호출자가 `return`문으로 반환되면, 피호출자는 우선 반환값<sub>return value</sub>를 호출자가 접근할 수 있는 곳에 저장한다.\
이는 호출자의 활성 프레임일 수도 있고, 피호출자의 활성 프레임일 수도 있다.\
(매개변수나 반환값은 두 활성 프레임의 경계에 걸쳐 있어서 어디라고 딱 집어 말하기 좀 애매한 구석이 있다.)

그리고 피호출자는 활성 프레임을 아래 과정을 통해 소멸<sub>destroy</sub>시킨다.
* 반환점<sub>return-point</sub> 기준으로 살아있는<sub>in-scope</sub> 지역 변수들 소멸<sub>destroy</sub>
* 매개변수들 소멸<sub>destroy</sub>
* 활성 프레임이 사용하는 메모리를 해제<sub>free</sub>

마지막으로, 아래 과정을 통해 호출자로 제어를 되돌려놓는다.
* 스택 포인터를 포함해 레지스터 값들을 복원
    * 이를 통해 현재 활성 프레임이 호출자의 것으로 되돌아오게 됨
* 호출자의 재개점<sub>resume-point</sub>으로 점프<sub>jump</sub>

[호출 규약<sub>Calling convention</sub>](https://en.wikipedia.org/wiki/Calling_convention)에 따라 반환 작업을 호출자, 피호출자가 나눠 할 수 있다.

## 중단<sub>Suspend</sub> 명령

중단<sub>Suspend</sub> 명령은 코루틴의 중간 지점에서 실행을 중단하고 호출자(또는 재개자)에게 제어를 돌려주는 일을 한다.

코루틴의 몸체<sub>body</sub> 내부에 중단 지점<sub>suspend-points</sub>을 정의할 수 있다.\
C++20 코루틴에서는 `co_await`이나 `co_yield` 키워드를 사용한 지점이 중단 지점이 된다.

코루틴이 아무 중단 지점<sub>suspend-points</sub>에나 도달하면, 아래 과정을 통해 재개 가능토록 준비를 한다.
* 레지스터 값들을 코루틴 프레임에 저장
* 어느 중단점<sub>suspend-point</sub>에서 멈췄는지를 나타내는 값 저장. 이를 통해:
    * 어디서 재개<sub>Resume</sub>해야 할지를 알 수 있음
    * 어느 변수들이 살아있는지 알 수 있어, 파괴<sub>Destroy</sub> 명령에서 제거해야 하는 대상을 알 수 있음

코루틴이 재개 가능토록 준비가 되면, 코루틴은 '중단된' 것으로 취급된다.

그 후, 코루틴은 호출자/재개자로 반환되기 전에 추가 로직을 수행할 수 있다.\
이 추가 로직에서는 코루틴 프레임의 핸들<sub>handle</sub>에 접근할 수 있는데, 이를 통해 코루틴을 재개/파괴할 수 있다.\
거기서 코루틴이 미래에 재개되도록 스케줄링<sub>schedule</sub>하면,
코루틴의 재개<sub>Resume</sub>가 자기 자신의 중단<sub>Suspend</sub>과 경쟁 상태<sub>race</sub>에 놓이는 것을 동기화<sub>synchronisation</sub> 없이 방지할 수 있다.

그 후, 코루틴은 즉시 실행을 재개할 것인지, 아니면 호출자/재개자에게 제어를 되돌려줄지를 선택할 수 있다.

호출자/재개자에게 제어가 되돌아간다면, 코루틴의 스택 프레임 부분은 스택에서 제거된다<sub>popped off the stack</sub>.

## 재개<sub>Resume</sub> 명령

재개<sub>Resume</sub> 명령은 현재 '중단' 상태인 코루틴에 대해서만 쓸 수 있다.

어떤 함수가 코루틴을 재개하려면, 특정 코루틴 호출의 중간 부분으로 '호출'해 들어갈 수 있어야 한다.\
재개하고자 하는 특정한 호출<sub>particular invocation</sub>로 들어가기 위해,
재개자는 해당 호출의 중단<sub>Suspend</sub> 명령에서 제공됐던 코루틴 프레임 핸들을 가지고 `void resume()` 메서드를 호출한다.

일반 함수 호출과 마찬가지로, `resume()` 호출에서도 피호출자에게 제어를 넘기기 전에
새 스택 프레임을 할당하고 호출자의 반환 주소<sub>return-address</sub>를 스택 프레임에 저장해둔다.

하지만, 제어 흐름을 코루틴의 시작 부분으로 옮기는 대신, 코루틴이 마지막으로 실행 중이었던 위치로 옮긴다.\
코루틴 프레임에서 재개점<sub>resume-point</sub>을 불러와 거기로 점프하는 방식이다.

코루틴이 다음번에 중단되거나 끝까지 실행<sub>runs to completion</sub>되면 `resume()` 호출이 반환되고 호출자에게 제어가 되돌아간다.

## 파괴<sub>Destroy</sub> 명령

파괴<sub>Destroy</sub> 명령은 코루틴 재개 없이 코루틴 프레임을 제거<sub>destroys</sub>한다.\
현재 '중단' 상태인 코루틴에 대해서만 쓸 수 있다.

파괴 명령은 재개 명령과 마찬가지로 새 스택 프레임을 할당하고 호출자의 반환 주소를 저장하여, 코루틴의 활성 프레임을 재활성화한다.\
하지만, 코루틴 몸체의 마지막 중단점으로 제어를 옮기는 대신,
별도의 코드 위치<sub>alternative code-path</sub>로 옮겨 중단점 내 지역 변수들을 정리하고 코루틴 프레임의 메모리를 해제<sub>free</sub>한다.

재개 명령과 비슷하게, 파괴 명령은 특정 호출의 중단<sub>Suspend</sub> 명령에서 제공됐던 코루틴 프레임 핸들을 가지고 `void destroy()` 메서드를 호출하여 이뤄진다.

## 코루틴의 호출<sub>Call</sub> 명령

코루틴의 호출<sub>Call</sub> 명령은 서브루틴의 그것과 똑같다.\
사실, 호출자 측에서는 차이점이 아예 없다.

하지만, 제어가 호출자 측으로 되돌아오는 이유가 코루틴이 끝까지 실행<sub>runs to completion</sub>되어서일 수도 있고,
첫번째 중단점<sub>first suspend-point</sub>에서 중단되어서일 수도 있다는 점이 다르다.

코루틴을 호출할 때, 호출자는 새로운 스택 프레임을 할당하고, 매개변수와 반환 주소를 스택 프레임에 쓰고, 제어를 코루틴에게 넘긴다.\
이건 서브루틴 호출과 완전히 똑같다.

그 후 코루틴이 하는 첫번째 작업은 힙<sub>heap</sub>에 코루틴 프레임을 할당하고, 매개변수들을 스택 프레임 쪽에서 코루틴 프레임 쪽으로 복사/이동시키는 일이다.
복사/이동을 통해, 매개변수들은 첫번째 중단점 이후에도 살아남아 있을 수 있게 된다.

## 코루틴의 반환<sub>Return</sub> 명령

코루틴의 반환<sub>Return</sub> 명령은 서브루틴과는 약간 다르다.

코루틴이 `co_return`문을 실행하면, 반환값을 어딘가에 저장하고(어디인지는 코루틴이 커스터마이징 할 수 있다),
범위 내<sub>in-scope</sub> 지역 변수들을 제거<sub>destructs</sub>한다.\
(하지만 매개변수는 제거하지 않는다.)

그 후, 코루틴은 호출자/재개자로 반환되기 전에 추가 로직을 수행할 수 있다.\
이 추가 로직은 마음대로 커스터마이징이 가능하다.\
예를 들어 반환값을 내보내기<sub>publish</sub> 위한 일을 수행할 수도, 결과를 기다리던 다른 코루틴을 재개할 수도 있다.

그 후, 코루틴은 아래 둘 중 하나의 명령을 수행한다:
1. 중단<sub>Suspend</sub> 명령: 코루틴 프레임을 살려놓는다.
1. 파괴<sub>Destroy</sub> 명령: 코루틴 프레임을 제거<sub>destroy</sub>한다.

그 후, 제어는 중단/파괴 명령의 동작 방식<sub>semantics</sub>에 따라 호출자/재개자에게 되돌아가며, 스택 프레임 부분을 제거한다.

반환<sub>Return</sub> 명령에 전달되는 반환값<sub>return-value</sub>은 호출<sub>Call</sub> 명령에서 반환받는 값과는 다르다는 점을 놓치지 말자.\
최초 호출 명령 이후, 한참 나중에야 재개된 코루틴이 반환 명령을 수행할 수도 있기 때문이다.


## 그림으로 이해하기

이 부분은 번역을 생략.\
[원본 글의 An illustration 단락](https://lewissbaker.github.io/2017/09/25/coroutine-theory#an-illustration)을 참조하라.


# `co_await` 연산자 이해하기

`co_await` 연산자의 작동 원리를 이해하면 코루틴이 어떻게 중단·재개되는지 그 베일을 벗기는 데<sub>demystify</sub> 도움이 될 수 있다.

하지만 우선 C++20 코루틴 TS<sub>기술 명세, Technical Specifications</sub>를 가볍게 살펴볼 필요가 있다.


## 코루틴 TS가 제공하는 것들

* 새로운 언어 키워드: [`co_await`](https://en.cppreference.com/w/cpp/keyword/co_await.html), [`co_yield`](https://en.cppreference.com/w/cpp/keyword/co_yield.html), [`co_return`](https://en.cppreference.com/w/cpp/keyword/co_return.html)
* [`<coroutine>` 헤더에 추가된 여러 타입들](https://en.cppreference.com/w/cpp/coroutine.html)
    * [`std::coroutine_handle`](https://en.cppreference.com/w/cpp/coroutine/coroutine_handle.html)
    * [`std::coroutine_traits`](https://en.cppreference.com/w/cpp/coroutine/coroutine_traits.html)
    * [`std::suspend_always`](https://en.cppreference.com/w/cpp/coroutine/suspend_always.html)
    * [`std::suspend_never`](https://en.cppreference.com/w/cpp/coroutine/suspend_never.html)
* 라이브러리 작성자가 코루틴을 커스터마이징하는데 쓸 수 있는 일반적 메커니즘
* 비동기<sub>asynchronous</sub> 코드 작성을 쉽게 만들어주는 언어 기능<sub>language facility</sub>

코루틴 TS가 제공하는 언어 설비<sub>facilities</sub>는 코루틴판 *저수준 어셈블리 언어*라고 생각할 수 있다.\
이 설비<sub>facilities</sub>는 직접 안전하게 쓰기는 어렵고, 어플리케이션 개발자들이 안전하게 쓸 수 있도록 라이브러리 작성자들에게 고수준 추상화 계층<sub>higher-level abstractions</sub>을 만들어 배포하라는 의도로 나온 것에 가깝다.


## 컴파일러 <-> 라이브러리 상호작용

재미있게도, 코루틴 TS는 코루틴의 동작 방식<sub>semantics</sub>을 정의하지 않는다.\
호출자에게 전달할 값을 어떻게 생성해야 하는지 정의하지 않는다.\
`co_return`문에 전달된 반환값으로 무엇을 해야 하는지도, 코루틴 바깥으로 전파<sub>propagates</sub>된 예외를 어떻게 처리해야 하는지도 정의하지 않는다.\
코루틴이 어떤 스레드에서 재개되어야 하는지도 정의하지 않는다.

그 대신, 라이브러리 코드가 코루틴의 동작<sub>behaviour</sub>을 커스터마이징할 수 있도록 특정 인터페이스를 따르는 타입을 구현할 것을 요구한다.\
그러면 컴파일러가 라이브러리가 제공한 타입의 객체<sub>instance</sub>를 만들어, 그것의 메서드<sub>method</sub>를 호출하는 코드를 생성한다.\
이 방식<sub>approach</sub>은 라이브러리 작성자가 `iterator` 타입과 `begin()`/`end()`를 정의하여 범위 기반 for 반복문<sub>range-based for-loop</sub>을 커스터마이징 할 수 있는 것과 흡사하다.

코루틴 TS가 특정한 동작 방식을 강제하지 않는 덕분에 C++20 코루틴은 강력하다.\
이 덕에 라이브러리 작성자는 온갖 목적으로 서로 다른 다양한 코루틴을 정의할 수 있게 되었다.

예를 들어, 값 하나를 비동기적으로 생성하는<sub>produces a single value asynchronously</sub> 코루틴이나,\
여러 값을 게으르게 생성<sub>produces a sequence of values lazily</sub>하는 코루틴,\
`std::optional<T>` 값을 보고 `std::nullopt`이면 얼리 리턴<sub>early-exiting</sub>하여 제어 흐름<sub>control-flow</sub>을 단순화하는 코루틴 등을 정의할 수 있다.

코루틴 TS는 아래 2종류의 인터페이스를 정의한다:
* **Promise** 인터페이스
* **Awaitable** 인터페이스

**Promise** 인터페이스는 코루틴 그 자체의 동작<sub>behaviour</sub>을 커스터마이징하는 메서드를 기술<sub>specifies</sub>한다.\
라이브러리 작성자는 코루틴이 호출될 때 무엇이 일어날지, 반환될 때 무엇이 일어날지 (일반적인 반환이나 처리되지 않은 예외),
해당 코루틴 내 `co_await`나 `co_yield` 표현식의 동작을 커스터마이징 할 수 있다.

**Awaitable** 인터페이스는 `co_await` 표현식의 동작 방식을 제어하는<sub>control the semantics of a `co_await` expression</sub> 메서드를 기술한다.\
값이 `co_await`되면, awaitable 객체에 대해 다음과 같은 일련의 호출을 하는 코드로 변환된다:
* 코루틴을 중단할지 말지를 판단
* 중단된 후 실행되어 나중에 재개되도록 스케줄링하는 로직을 수행
* 코루틴이 재개된 후 `co_await` 표현식의 결과값을 생성하는 로직을 수행


## Awaiter와 Awaitable: `operator co_await` 설명

`co_await` 연산자는 새로운 단항 연산자로 하나의 값에 적용될 수 있다. (e.g. `co_await someValue`)

`co_await` 연산자는 코루틴 내에서만 쓸 수 있다.
그런데 사실 이 설명은 다소 동어반복적<sub>tautology</sub>이다.
왜냐면 정의에 따르면 `co_await` 연산자를 사용한 함수 몸체는 코루틴으로 컴파일되기 때문이다.

`co_await` 연산자를 지원하는 타입을 **Awaitable** 타입이라고 부른다.

`co_await` 연산자가 어떤 타입에 적용될 수 있는지는 `co_await` 표현식이 쓰인 문맥에 따라 다르다.\
코루틴에 쓰인 promise 타입의 `await_transform` 메서드가 `co_await`의 의미를 바꿔버릴 수 있기 때문이다.

필요한 경우 필자는 promise 타입에 `await_transform` 멤버가 없는 코루틴 문맥에서 `co_await` 연산자가 지원되는 경우 **Normally Awaitable**하다고 부른다.\
또한 필자는 promise 타입에 적절한 `await_transform`이 있는 특정 타입의 코루틴 문맥에서만 `co_await` 연산자가 지원되는 경우 **Contextually Awaitable**하다고 부른다.

**Awaiter** 타입은 `co_await` 표현식에서 호출되는 3개의 특별한 메서드를 구현하는 타입이다:
* `await_ready`
* `await_suspend`
* `await_resume`

사실 'Awaiter'라는 용어는 C#의 `async` 키워드의 메커니즘에서 필자가 빌려온 것이다.\
C#의 `async` 키워드는 `GetAwaiter()` 메서드로 C++에서의 **Awaiter**와 기이하게 비스무리한<sub>eerily similar</sub> 인터페이스를 갖는 객체를 반환하도록 구현된다.\
C# awaiter가 궁금하다면 [이 글](https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-2-awaitable-awaiter-pattern)을 읽어보라.\
(역자 주: 그러고보니 이전에 [C#에서 custom awaitable을 작성해본 적이 있는데](/2025/03/23/csharp-custom-awaitable), C++ 코루틴과 꽤나 흡사한 측면이 있다.)

타입은 **Awaitable**이면서 동시에 **Awaiter**일 수 있다.

컴파일러가 `co_await <expr>` 표현식을 변환할 시, 관련 타입들에 의해 여러가지 방식으로 변환될 수 있다.


### Awaiter를 가져오기<sub>obtaining</sub>

await되는 값에 대해 컴파일러가 제일 처음 하는 일은 그것의 **Awaiter** 객체를 가져오는<sub>obtain</sub> 일이다.\
awaiter 객체를 가져오기 위해 여러 단계를 거치는데, [N4680](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf)의 5.3.8(3)에 정리되어 있다.

await되려는 코루틴이 `P` 타입의 promise 객체를 가지고, `promise`는 그 객체의 좌측값 참조<sub>l-value reference</sub>라고 해보자.

promise 타입 `P`가 `await_transform` 멤버를 가지면 `<expr>`은 먼저 `promise.await_transform(<expr>)` 호출로 전달되어 **Awaitable** 값 `awaitable`을 가져온다.\
그렇지 않은, promise 타입에 `await_transform` 멤버가 없는 경우에는 `<expr>`을 있는 그대로 평가해 **Awaitable** 객체 `awaitable`로 사용한다.

그 후, **Awaitable** 객체 `awaitable`에 적용 가능한 `operator co_await()` 오버로드가 있는 경우, 이를 호출해 **Awaiter** 객체를 얻어온다.\
그렇지 않으면, `awaitable` 객체 그대로를 awaiter 객체로 사용한다.

위 규칙을 `get_awaitable()`과 `get_awaiter()`라는 함수의 형태로 나타낸다면, 이런 느낌일 것이다:
```cpp
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
  if constexpr (has_any_await_transform_member_v<P>)
    return promise.await_transform(static_cast<T&&>(expr));
  else
    return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
  if constexpr (has_member_operator_co_await_v<Awaitable>)
    return static_cast<Awaitable&&>(awaitable).operator co_await();
  else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
    return operator co_await(static_cast<Awaitable&&>(awaitable));
  else
    return static_cast<Awaitable&&>(awaitable);
}
```


### Awaiter를 대기하기<sub>awaiting</sub>

`<expr>` 결과를 **Awaiter**로 변환하는 로직을 위 함수들과 같이 캡슐화했다 치면
`co_await <expr>`의 의미<sub>semantics</sub>는 (대강) 이렇게 해석될 수 있다:
```cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(promise)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>)
    {
      awaiter.await_suspend(handle_t::from_promise(promise));
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(promise)))
      {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
```
(역자 주: 위 코드와 아래 설명에는 `std::coroutine_handle`을 반환하는 버전의 `await_suspend()`가 빠져 있다.
해당 글이 작성된 2017년엔 코루틴 TS에 아직 symmetric transfer가 추가되기 전이어서 그런 듯하다. 이에 대한 내용은 한참 아래에서 다룬다.)

`void`를 반환하는 버전의 `await_suspend()`의 경우 `await_suspend()`가 반환하면 무조건 코루틴의 호출자/재개자에게 제어를 돌려주고,\
`bool`을 반환하는 버전은 awaiter 객체가 조건부로 호출자/재개자에게 돌려주지 않고 코루틴을 즉시 재개하는 것을 허용한다.

`bool`을 반환하는 버전의 `await_suspend()`가 유용한 경우가 있는데, 이를테면 awaiter가 시작한 비동기 명령<sub>async operation</sub>이 어떨 때는 동기적으로 완료될 수 있는 경우와 같은 상황에서 그러하다.\
동기적으로 완료되는 경우, `await_suspend()`가 `false`를 반환하여 코루틴이 즉시 재개되어 실행을 계속해야 한다고 명시할<sub>indicate</sub> 수 있다.

컴파일러는 `<suspend-coroutine>` 지점에 현재 코루틴의 상태를 저장하고 재개를 준비하는 코드를 생성한다.\
이는 레지스터 값들과 `<resume-point>`의 위치를 코루틴 프레임 메모리에 저장하는 과정을 포함한다.

현재 코루틴은 `<suspend-coroutine>` 명령이 완료되면 중단된 것으로 취급된다.\
중단된 코루틴을 우리측에서 처음으로 관찰할 수 있는 지점은 `await_suspend()` 호출 내부에서이다.\
코루틴이 중단되면 재개<sub>resumed</sub>되거나 파괴<sub>destroyed</sub>될 수 있다.

코루틴이 재개(혹은 파괴)되도록 스케줄링<sub>schedule</sub>하는 것은 `await_suspend()`의 책임이다.\
`await_suspend()`에서 `false`를 반환하는 것은 현재 스레드에서 코루틴을 즉시 재개하겠다고 스케줄링<sub>scheduling</sub>하는 것으로 취급된다<sub>counted as</sub>.

`await_ready()` 메서드의 목적은 명령이 대기 없이 동기적으로 완료될 수 있는 경우 `<suspend-coroutine>`의 비용을 피하도록 하는 것이다.

`<return-to-caller-or-resumer>` 지점에서 제어는 호출자나 재개자에게 되돌아가는데, 로컬 스택 프레임은 제거<sub>popping</sub>되지만 코루틴 프레임은 살아있다.

중단된 코루틴이 재개된다면, 그 때 수행 흐름<sub>execution</sub>은 `<resume-point>`로부터 재개된다.\
달리 말해, 결과를 얻기 위해 `await_resume()` 메서드가 호출되기 직전 시점부터 재개된다는 말이다.

`await_resume()` 메서드의 반환값은 `co_await` 표현식의 결과가 된다.\
`await_resume()` 메서드에서 예외를 던질 수도 있는데 그 경우 예외는 `co_await` 표현식 바깥으로 전파된다.

예외가 `await_suspend()`의 바깥으로 전파되는 경우엔 코루틴이 `await_resume()` 호출 없이 자동으로 재개되고
예외는 `co_await` 표현식 바깥으로 전파된다는 점을 유념하라.


## 코루틴 핸들<sub>Coroutine Handles</sub>

`co_await` 표현식의 `await_suspend()` 호출에서 매개변수로 `std::coroutine_handle<P>`가 전달된다는 것을 눈치챘을지도 모르겠다.

이 타입은 코루틴 프레임을 참조하는 소유권이 없는<sub>non-owning</sub> 핸들을 나타낸다.
이 핸들로 코루틴을 재개하거나 코루틴 프레임을 파괴할 수 있다.
또한 이 핸들을 통해 코루틴의 promise 객체에 접근할 수 있다.

`std::coroutine_handle`은 (축약하면) 아래와 같은 인터페이스를 갖는다:
```cpp
namespace std
{
  template<typename Promise>
  struct coroutine_handle;

  template<>
  struct coroutine_handle<void>
  {
    bool done() const;

    void resume();
    void destroy();

    void* address() const;
    static coroutine_handle from_address(void* address);
  };

  template<typename Promise>
  struct coroutine_handle : coroutine_handle<void>
  {
    Promise& promise() const;
    static coroutine_handle from_promise(Promise& promise);

    static coroutine_handle from_address(void* address);
  };
}
```

**Awaitable** 타입을 구현할 때, `coroutine_handle`에서 핵심적으로 사용할 메서드는 `.resume()`일 것이다.
명령이 완료된 이후 대기하는<sub>awaiting</sub> 코루틴을 재개하고 싶을 때 호출하면 된다.\
`coroutine_handle`의 `.resume()`을 호출하면 중단된 코루틴이 `<resume-point>`로부터 재활성화된다.\
`.resume()` 호출은 다음 번 `<return-to-caller-or-resumer>`를 만나는 지점에서 반환될 것이다.

`.destroy()` 메서드는 코루틴 프레임을 파괴시킨다.
스코프 내 변수들의 소멸자를 호출하고 코루틴 프레임이 사용하는 메모리를 해제한다.\
당신이 코루틴 promise 타입을 구현하는 라이브러리 작성자가 아니라면, 일반적으로는 `.destroy()`를 직접 호출할 필요는 없다. (사실 호출을 피해야 한다.)\
보통은, 코루틴 프레임은 코루틴이 반환하는 어떤 RAII 타입이 소유하도록 구현될 것이다.\
그러니 RAII 객체의 협력<sub>cooperation</sub> 없이 `.destroy()`를 호출하는 것은 이중 파괴 버그<sub>double-destruction bug</sub>를 야기할 것이다.

`.promise()` 메서드는 코루틴 promise 객체의 참조를 반환한다.\
하지만 보통은, `.destroy()`와 마찬가지로, 당신이 promise 타입의 작성자일 때만 쓸모가 있다.\
코루틴의 promise 객체를 코루틴의 내부 구현 사항<sub>internal implementation detail</sub>으로 감추는 것을 고려하라.\
대부분의 **Normally Awaitable** 타입들에서 `await_suspend()` 메서드의 매개변수 타입으로 `coroutine_handle<void>`를 사용해야 한다. `coroutine_handle<Promise>` 대신.

`coroutine_handle<P>::from_promise(P& promise)` 함수는 코루틴 promise 객체의 참조로부터 코루틴 핸들을 재생성<sub>reconstructing</sub>하는 것을 가능케 한다.\
타입 `P`가 코루틴 프레임에 사용된 구체<sub>concrete</sub> promise 타입과 정확히 일치해야<sub>exactly matches</sub> 한다는 점을 유념하라.\
구체 promise 타입이 `Derived`인데 `coroutine_handle<Base>`를 생성하려고 하면 정의되지 않은 동작<sub>undefined behaviour</sub>이 될 수 있다.

`.address()`/`from_address()` 함수들은 코루틴 핸들과 `void*` 포인터 사이의 변환을 가능케 한다.\
이건 '컨텍스트<sub>context</sub>' 변수를 C-스타일 API에 넘기는 경우에 쓰라고 의도한 것이다.
그러니 어떤 상황에서는 **Awaitable** 타입을 구현하는 데 유용할 수도 있다.\
하지만, 필자 경험상 대부분의 경우 콜백 '컨텍스트' 매개변수에 추가 정보를 넘겨야 했어서
보통 `coroutine_handle`을 구조체에 넣은 다음 그 구조체의 포인터를 '컨텍스트' 매개변수로 넘겼지, `.address()` 반환값을 사용하진 않았다.


## 동기화가 필요없는<sub>Synchronisation-free</sub> 비동기 코드

(WIP)


### Stackful 코루틴과의 비교

(WIP)


## 메모리 할당 피하기

(WIP)


## 간단한 스레드-동기화 객체<sub>thread-synchronisation primitive</sub> 예제

이 부분은 번역을 생략.\
[원본 글의 An example: Implementing a simple thread-synchronisation primitive 단락](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await#an-example-implementing-a-simple-thread-synchronisation-primitive)을 참조하라.


# promise 타입 이해하기

코루틴 TS는 `co_await`, `co_yield`, `co_return`의 3가지 키워드를 추가한다.\
이 코루틴 키워드 중 아무 것이나 함수 몸체에 사용하면 컴파일러는 그 함수를 서브루틴이 아닌 코루틴으로 컴파일한다.

컴파일러는 우리가 작성한 코드를 꽤 기계적인 변환<sub>fairly mechanical transformations</sub>을 거쳐
함수 내 특정 위치에서 실행을 중단, 나중에 재개할 수 있는 상태 기계<sub>state-machine</sub>로 바꾸어놓는다.

코루틴 TS에서 도입한 2번째 인터페이스인 **Promise** 인터페이스는 이 코드 변환에 중요한 역할을 한다.

**Promise** 인터페이스는 코루틴 그 자체의 동작<sub>behaviour</sub>을 커스터마이징하는 메서드를 기술<sub>specifies</sub>한다.\
라이브러리 작성자는 코루틴이 호출될 때 무엇이 일어날지, 반환될 때 무엇이 일어날지 (일반적인 반환이나 처리되지 않은 예외),
해당 코루틴 내 `co_await`나 `co_yield` 표현식의 동작을 커스터마이징 할 수 있다.

## Promise 객체

**Promise** 객체는 코루틴 실행 중 특정 지점에 호출되는 메서드들을 구현함으로써 코루틴 그 자체의 동작을 정의하고 제어한다.

> 계속하기에 앞서, "promise"가 무엇인지에 관한 모든 선입견을 내려놓기를 바란다.\
> 비록 어떤 용례에서는 코루틴 promise 객체가 실제로 `std::future` 짝<sub>pair</sub>의 `std::promise` 부분과 비슷한 역할을 하기도 하지만,
> 다른 용례에서는 이 유사점이 과장되는 측면이 있다<sub>the analogy is somewhat stretched</sub>.\
> 코루틴의 promise 객체를 코루틴의 동작을 제어하고 그 상태를 추적하는 "코루틴 상태 컨트롤러<sub>coroutine state controller</sub>" 객체라고 여기는 편이 더 쉬울 수도 있다.

promise 객체의 인스턴스<sub>instance</sub>는 코루틴 함수가 호출될 때 코루틴 프레임 내부에 생성된다.

컴파일러는 코루틴 실행 중 중요한 지점들에 promise 객체의 특정 메서드들을 호출하는 코드를 생성한다.

아래 예제들에서, 특정 코루틴 호출의 코루틴 프레임 내에 생성된 promise 객체를 `promise`라고 가정하겠다.

`<body-statements>`라는 몸체를 가지는 코루틴 함수를 작성했고, 그 안에 코루틴 키워드(`co_return`, `co_await`, `co_yield`)가 존재한다면
코루틴 몸체는 (대강) 아래와 같은 코드로 변환된다:
```cpp
{
  co_await promise.initial_suspend();
  try
  {
    <body-statements>
  }
  catch (...)
  {
    promise.unhandled_exception();
  }
FinalSuspend:
  co_await promise.final_suspend();
}
```

코루틴 함수가 호출될 때, 코루틴 몸체의 코드를 실행하기에 앞서 수행되는 여러 단계가 있는데, 이는 일반적인 서브루틴의 동작과는 약간 다르다.

이 단계를 요약하면 다음과 같다 (각 단계는 아래에서 더 상세히 다룬다).
1. `operator new`를 이용해 코루틴 프레임을 할당한다 (선택).
1. 함수 매개변수를 코루틴 프레임에 복사한다.
    * 값 매개변수는 복사/이동되고, 참조는 참조로 남는다.
        * 고로 참조하는 객체의 수명이 끝났다면 참조는 dangling이 된다.
1. `P` 타입 promise 객체의 생성자를 호출한다.
    * promise 타입이 모든 코루틴 매개변수를 받는 생성자를 갖고 있다면, 복사가 완료된 코루틴 실인자<sub>arguments</sub>들로 그 생성자를 호출한다.
    * 그렇지 않다면, 기본 생성자가 호출된다.
1. `promise.get_return_object()` 메서드를 호출해 그 결과를 지역 변수로 들고 있는다. 이는 코루틴이 처음 중단될 때 호출자에게 반환된다.
    * 이 단계를 포함, 그 전에 예외가 던져진다면 예외는 호출자에게 전파되지, promise에 저장되지 않는다.
1. `promise.initial_suspend()` 메서드를 호출해 그 결과를 `co_await`한다.
    * Promise 타입은 흔히 `std::suspend_always`를 반환해 게으른 시작<sub>lazily-started</sub>을 하거나, `std::suspend_never`로 즉시 시작<sub>eagerly-started</sub>한다.
1. `co_await promise.initial_suspend()` 표현식이 (즉시든 비동기든) 재개되면, 당신이 작성한 코루틴 몸체의 수행을 시작한다.

(역자 주: [cppreference 코루틴 문서](https://en.cppreference.com/w/cpp/language/coroutines.html#Execution)에서는
코루틴 프레임<sub>coroutine frame</sub> 대신 코루틴 상태<sub>coroutine state</sub>라는 표현을 사용한다.\
또한 코루틴 상태의 할당/해제가 선택사항이라는 언급이 없는데, 확정된 표준은 선택권을 주지 않는 것일지도 모르겠다.\
제대로 확인하려면 확정된 표준 문서를 읽어봐야하겠는데 귀찮은 관계로 생략...)

`co_return`문에 도달하면 아래 동작이 수행된다:
1. `promise.return_void()` 또는 `promise.return_value(<expr>)`를 수행한다.
    * `<expr>`이 `void`로 평가되면 전자를 수행, 아니면 후자
1. automatic storage duration 생명주기를 갖는 모든 변수를 생성된 것의 역순으로 소멸시킨다.
1. `promise.final_suspend()`를 호출해 그 결과를 `co_await`한다.

그 대신, 코루틴이 처리되지 않는 예외로 인해 `<body-statements>`를 빠져나간다면:
1. 예외를 잡아 catch-block 내에서 `promise.unhandled_exception()`을 호출한다.
1. `promise.final_suspend()`를 호출해 그 결과를 `co_await`한다.
    * 예를 들어 continuation을 재개하거나 결과를 내보내기<sub>publish</sub> 위해 쓰일 수 있다.
    * 이 시점 이후로 코루틴을 재개하면 undefined behavior이다.

`co_return`이든, 처리되지 않은 예외이든, 핸들을 통해 파괴되든 간에, 코루틴 상태가 파괴되면 다음 동작을 수행한다:
1. promise 객체의 소멸자를 호출한다.
1. 함수 매개변수 복사본들의 소멸자를 호출한다.
1. 코루틴 상태가 사용하던 메모리를 `operator delete`로 해제한다 (선택).
1. 호출자/재개자에게 제어를 돌려준다.

수행 흐름<sub>execution</sub>이 `co_await` 표현식 내의 `<return-to-caller-or-resumer>` 지점에 처음 도달하면,
또는 코루틴이 `<return-to-caller-or-resumer>`를 한번도 만나지 않고 끝까지 수행되면,
코루틴은 중단되거나 파괴되고, 이전에 `promise.get_return_object()` 호출로 반환됐던 반환 객체가 코루틴의 호출자에게 반환된다.


### 코루틴 프레임 할당

(WIP)

#### 코루틴 프레임 메모리 할당 커스터마이징

(WIP)

### 코루틴 프레임으로 매개변수 복사

(WIP)

### promise 객체 생성

(WIP)

### 반환 객체 얻어오기

(WIP)

### 최초 중단점<sub>The initial-suspend point</sub>

(WIP)

### 호출자에게 반환

(WIP)

### `co_return`으로 코루틴 반환

(WIP)

### 코루틴 몸체 바깥으로 전파된 예외 처리

(WIP)

### 최종 중단점<sub>The final-suspend point</sub>

코루틴 몸체의 사용자가 정의한 부분의 수행이 끝나고 결과가 `return_void()`, `return_value()`, `unhandled_exception()` 호출로 잡히고<sub>captured</sub> 모든 지역 변수가 소멸된 후에,
코루틴은 호출자/재개자로 반환되기 전에 추가 로직을 수행할 수 있다.

코루틴은 `co_await promise.final_suspend();` 문을 수행한다.

이 덕에 코루틴은 결과를 내보내거나<sub>publish</sub>, 완료를 통지하거나<sub>signalling completion</sub>, continuation을 재개하는 등의 로직을 수행할 수 있다.\
또한 이 덕에 원한다면 코루틴이 끝까지 실행되어 코루틴 프레임이 파괴되기 전에 코루틴을 즉시 중단시킬 수 있다.

`final_suspend` 시점에서 중단된 코루틴을 `resume()`하는 것은 정의되지 않은 동작<sub>undefined behaviour</sub>이라는 점을 유념하라.\
여기서 중단된 코루틴에 할 수 있는 것은 `destroy()` 뿐이다.

이 제한사항의 이유<sub>rationale</sub>는, Gor Nishanov에 따르면,
표현돼야 하는 중단 상태와 필요한 분기의 개수를 줄임으로써 컴파일러가 최적화할 여지가 많아지는 효과가 있기 때문이다.

코루틴이 `final_suspend` 지점에서 중단되지 않는 것이 허용되기는 하지만, 가능하면 **당신 코루틴이 `final_suspend`에서 중단되도록 코루틴을 구조화하는 것이 추천된다**는 것을 유념하자.\
왜냐하면 이렇게 하면 `.destroy()` 호출을 코루틴 바깥에서 하는 것이 강제되어 (보통은 어떤 RAII 소멸자에서 담당할 것이다)
컴파일러가 코루틴 프레임의 생명 주기가 호출자에게 종속되는지<sub>nested</sub> 판단하기가 훨씬 용이해지기 때문이다.\
그렇게 되면 컴파일러가 코루틴 프레임의 메모리 할당을 생략<sub>elide</sub>할 가능성이 훨씬 높아진다.


### 컴파일러는 promise 타입을 어떻게 선택하는가

(WIP)

### 특정 코루틴 활성 프레임을 구분하기<sub>identifying</sub>

(WIP)

### `co_await`의 동작 커스터마이징

promise 타입은 원한다면 코루틴 몸체에 있는 모든 `co_await` 표현식의 동작을 커스터마이징 할 수 있다.

promise 타입에 `await_transform()`이라는 이름의 메서드를 정의하기만 하면,
컴파일러는 코루틴 몸체의 모든 `co_await <expr>`을 `co_await promise.await_transform(<expr>)`로 바꿔놓는다.

여기엔 여러 중요하고 강력한 용도들이 있다:

**1. Normally Awaitable하지 않은 타입을 awaiting할 수 있게 된다.**

예를 들어, `std::optional<T>`를 반환하는 코루틴의 promise 타입은,
`std::optional<U>`를 받아 `U`타입 값을 반환하거나 `std::nullopt`의 경우엔 코루틴을 중단시키는
`await_transform()` 오버로드를 제공할 수 있다.

```cpp
template<typename T>
class optional_promise
{
  ...

  template<typename U>
  auto await_transform(std::optional<U>& value)
  {
    class awaiter
    {
      std::optional<U>& value;
    public:
      explicit awaiter(std::optional<U>& x) noexcept : value(x) {}
      bool await_ready() noexcept { return value.has_value(); }
      void await_suspend(std::coroutine_handle<>) noexcept {}
      U& await_resume() noexcept { return *value; }
    };
    return awaiter{ value };
  }
};
```

**2. 특정 타입의 `await_transform` 오버로드를 delete로 선언하여 그 타입들이 awaiting되는 걸 막을 수 있다.**

예를 들어, `std::generator<T>`의 promise 타입은 아무 타입이나 받을 수 있는 `await_transform()` 템플릿 멤버 함수를 delete로 선언할 수 있다.\
이렇게 하면 코루틴 내에서 `co_await`의 사용을 막는 효과가 있다.

```cpp
template<typename T>
class generator_promise
{
  ...

  // 이 코루틴 타입에 대해서 co_await 사용을 방지한다.
  template<typename U>
  std::suspend_never await_transform(U&&) = delete;

};
```

**3. Normally Awaitable한 값의 동작을 변경<sub>adapt and change</sub>할 수 있다.**

예를 들어, 모든 `co_await` 표현식이 특정 executor에서 재개되도록 하는 종류의 코루틴을 정의할 수 있다.
이는 awaitable을 `resume_on()` 연산자로 감싸는<sub>wrapping</sub> 식으로 구현될 수 있다 (`cppcoro::resume_on()`을 보라).

```cpp
template<typename T, typename Executor>
class executor_task_promise
{
  Executor executor;

public:

  template<typename Awaitable>
  auto await_transform(Awaitable&& awaitable)
  {
    using cppcoro::resume_on;
    return resume_on(this->executor, std::forward<Awaitable>(awaitable));
  }
};
```

마지막으로 `await_transform()`에 대해 첨언하면, promise 타입에 대해 `await_transform()` 멤버를 *하나라도* 정의하는 순간
컴파일러는 *모든* `co_await` 표현식을 `promise.await_transform()` 호출로 바꿔놓는다는 점을 유념하는 게 중요하다.\
이 말은 일부 타입에 대해서만 `co_await`의 동작을 커스터마이징하고 싶다면
단순히 인자를 포워딩만 하는 fallback `await_transform()` 오버로드를 추가로 제공해야 한다는 말이다.


### `co_yield`의 동작 커스터마이징

마지막으로 promise 타입을 이용해 커스터마이징 할 수 있는 것은 `co_yield` 키워드의 동작이다.

`co_yield` 키워드가 코루틴에서 나타나면 컴파일러는 `co_yield <expr>` 표현식을 `co_await promise.yield_value(<expr>)` 표현식으로 번역한다.\
고로 promise 객체가 쓸 `yield_value()` 메서드를 하나 이상 정의함으로써 `co_yield` 키워드의 동작을 커스터마이징 할 수 있다.

`await_transform`과는 달리, `yield_value()` 메서드가 정의되어 있지 않으면 `co_yield` 키워드의 기본 동작이 존재하지 않는다는 점을 유념하라.\
고로 promise 타입에서 `co_await` 지원을 제거하려면 명시적으로 `await_transform()`을 delete로 선언해야 하는 반면<sub>opt-out</sub>,
`co_yield`를 지원하려면 명시적으로 `yield_value()`를 정의해야 한다<sub>opt-in</sub>.

promise 타입이 `yield_value()` 메서드를 지원하는 전형적인 예는 `generator<T>` 타입의 경우이다:
```cpp
template<typename T>
class generator_promise
{
  T* valuePtr;
public:
  ...

  std::suspend_always yield_value(T& value) noexcept
  {
    // yield된 값의 주소를 저장한 후 co_yield 표현식에서 코루틴을 중단시키는
    // awaitable을 반환한다. 그러면 제어는 generator<T>::begin() 이나
    // generator<T>::iterator::operator++() 안에서 호출된
    // coroutine_handle<>::resume() 호출로 되돌아갈 것이다.
    valuePtr = std::addressof(value);
    return {};
  }
};
```


# Symmetric Transfer 이해하기

(WIP)

## task 코루틴 작동방식 배경 설명

(WIP)

## `task` 구현 개요<sub>Outline</sub>

(WIP)

## `task::promise_type` 구현하기

(WIP)

## `task::operator co_await()` 구현하기

(WIP)

## 스택 오버플로 문제

(WIP)

## 코루틴 TS 해결책

(WIP)

## 여전한 문제점

(WIP)

## "symmetric transfer" 입장!

Gor Nishanov가 2018년에 작성한 [P0913R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0913r0.html) 제안서<sub>paper</sub> "Add symmetric coroutine control transfer"는
한 코루틴이 중단될 때 추가적인 스택 공간 사용 없이 다른 코루틴이 대칭적으로<sub>symmetrically</sub> 재개되도록 하는 기능<sub>facility</sub>을 제공하도록 하는 이 문제의 해결책을 제안했다.

이 제안서<sub>paper</sub>는 2가지 핵심 변경사항을 제안했다:
* `await_suspend()`에서 `std::coroutine_handle<T>`를 반환하는 것을 허용한다.
  그게 반환되면 실행 흐름이 반환된 핸들로 나타내어지는 코루틴으로 대칭적으로 전달<sub>symmetrically transferred</sub>되어야 함을 의미한다.
* `std::noop_coroutine()` 함수를 추가한다.
  이 함수는 `await_suspend()`에서 반환될 수 있는 특수한 `std::coroutine_handle`을 가져오는데,
  이걸 반환하면 다른 코루틴으로 실행 흐름을 전달하는 대신 현재 코루틴을 중단시킨 후 `.resume()` 호출을 반환시킨다.

그래서, "대칭적으로 전달<sub>symmetric transfer</sub>"한다는 게 무슨 의미일까?

코루틴을 `std::coroutine_handle`의 `.resume()` 호출로 재개하면 코루틴이 실행되는 동안 `.resume()` 호출자의 스택이 활성화된 상태로 남아있게 된다.\
이 코루틴이 다음번에 중단되고 그 중단점의 `await_suspend()` 호출이 `void`를 반환하거나 (무조건 중단) `true`를 반환해야 (조건부 중단) 그제야 `.resume()` 호출이 반환된다.

이것은 "비대칭적인 전달<sub>asymmetric transfer</sub>"이라고 여겨질 수 있고, 서브루틴 호출과 똑같이 동작한다.\
`.resume()`의 호출자는 아무 함수나 될 수 있다 (코루틴일 수도 아닐 수도 있다).

코루틴을 `.resume()` 호출로 재개할 때마다 그 코루틴을 실행하기 위한 새로운 스택 프레임을 생성하는 것이다.

하지만, "symmetric transfer"에서는 그저 한 코루틴을 중단시키고 다른 코루틴을 재개할 뿐이다.\
두 코루틴 사이에는 암시적인 호출자/피호출자 관계가 없다.
코루틴이 중단되면 제어를 아무 코루틴에게나(자기 자신 포함) 전달할 수 있지, 중단/완료되었을 때 제어를 꼭 이전 코루틴에게 전달할 필요는 없다.

symmetric-transfer를 사용할 때 컴파일러가 `co_await` 표현식을 어떻게 lower하는지 보자:
```cpp
{
  decltype(auto) value = <expr>;
  decltype(auto) awaitable =
      get_awaitable(promise, static_cast<decltype(value)&&>(value));
  decltype(auto) awaiter =
      get_awaiter(static_cast<decltype(awaitable)&&>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::coroutine_handle<P>;

    //<suspend-coroutine>

    auto h = awaiter.await_suspend(handle_t::from_promise(p));
    h.resume();
    //<return-to-caller-or-resumer>

    //<resume-point>
  }

  return awaiter.await_resume();
}
```

다른 `co_await` 형태와는 다른 중요한 부분에 주목해보겠다:

```cpp
auto h = awaiter.await_suspend(handle_t::from_promise(p));
h.resume();
//<return-to-caller-or-resumer>
```

상태 기계가 lowered되면 (나중에 다룰 예정), `<return-to-caller-or-resumer>` 부분이 `return;` 문이 되어
마지막으로 코루틴을 재개한 `.resume()` 호출이 호출자에게 반환된다.

이 말은 현재 함수의 몸체가 `std::coroutine_handle::resume()`인데 그 안에서
동일 시그니처의 또 다른 `std::coroutine_handle::resume()` 호출 직후에 `return;`을 하는 상황이라는 것이다.

일부 컴파일러들은, 최적화가 켜져 있을 때, 몇몇 조건이 만족되면 tail-position(반환 직전)에 있는 다른 함수 호출을 꼬리 호출<sub>tail-calls</sub>로 바꾸는 최적화를 수행할 수 있다.

우연히도 우리가 하고자 하는 스택 오버플로 문제의 해결을 이 꼬리 호출 최적화<sub>tail-call optimisation</sub>가 해줄 수 있다.\
하지만 최적화가 되길 간절히 바라는 대신, 우리는 최적화가 꺼져 있더라고 이 꼬리 호출 변환이 항상 일어나는 게 보장되길 원한다.

하지만 우선 꼬리 호출<sub>tail-calls</sub>이 뭔지부터 파보자.


## 꼬리 호출<sub>Tail-calls</sub>

(WIP)

## `task` 재작성하기<sub>revisited</sub>

새로운 "symmetric transfer" 기능<sub>capability</sub>을 머릿속에 넣었다면<sub>under our belt</sub> 우리의 이전 `task` 타입 구현을 고칠 차례다.

그러려면 두 `await_suspend()` 메서드의 구현을 고쳐야 한다:
1. task를 await할 때 해당 task의 코루틴을 symmetric-transfer로 재개하도록
1. task의 코루틴이 완료됐을 때 대기하는 코루틴을 symmetric-transfer로 재개하도록

await되는 방향을 나타내려면<sub>address</sub> `task::awaiter`의 메서드를 이것에서:
```cpp
void task::awaiter::await_suspend(
    std::coroutine_handle<> continuation) noexcept {
  // continuation을 task의 promise에 저장해놓음으로써 task가 완료됐을 때
  // final_suspend()가 이 코루틴을 재개해야 한다는 것을 알게 한다.
  coro_.promise().continuation = continuation;

  // 그 후 현재 initial-suspend-point(중괄호 오픈)에 중단되어 있는
  // task의 코루틴을 재개한다.
  coro_.resume();
}
```
이 모습으로 수정해야 한다:
```cpp
std::coroutine_handle<> task::awaiter::await_suspend(
    std::coroutine_handle<> continuation) noexcept {
  // continuation을 task의 promise에 저장해놓음으로써 task가 완료됐을 때
  // final_suspend()가 이 코루틴을 재개해야 한다는 것을 알게 한다.
  coro_.promise().continuation = continuation;

  // 그 후 현재 initial-suspend-point(중괄호 오픈)에 중단되어 있는
  // task의 코루틴을, 핸들을 반환함으로써 꼬리-재개한다.
  return coro_;
}
```

그리고 return-path를 나타내려면<sub>address</sub> `task::promise_type::final_awaiter` 메서드를 이것에서:
```cpp
void task::promise_type::final_awaiter::await_suspend(
    std::coroutine_handle<promise_type> h) noexcept {
  // 이제 코루틴이 final-suspend 위치에서 중단되었다.
  // promise에서 continuation을 찾아 재개한다.
  h.promise().continuation.resume();
}
```
이 모습으로 수정해야 한다:
```cpp
std::coroutine_handle<> task::promise_type::final_awaiter::await_suspend(
    std::coroutine_handle<promise_type> h) noexcept {
  // 이제 코루틴이 final-suspend 위치에서 중단되었다.
  // promise에서 continuation을 찾아 symmetrical하게 재개한다.
  return h.promise().continuation;
}
```

이제 우리 `task` 구현은 `void`를 반환하는 `await_suspend` 방식<sub>flavour</sub>에서 있었던 스택 오버플로 문제에서 자유로우며,
`bool`을 반환하는 `await_suspend` 방식<sub>flavour</sub>에서 있었던 비결정론적인<sub>non-deterministic</sub> 재개 컨텍스트 문제에서 자유롭다.


### 스택 시각화하기

(WIP)

symmetric-transfer 버전의 `task` 타입 예제를 보고 싶다면 이 Compiler Explorer 링크를 참조하라: <https://godbolt.org/z/9baieF>


## `await_suspend`의 보편적 형태<sub>the Universal Form</sub>로서의 Symmetric Transfer

(WIP)

### 재귀를 종료하기<sub>Terminating</sub>

(WIP)

### 다른 모습<sub>flavours</sub>의 `await_suspend()`를 표현하기

(WIP)

### 그럼 왜 3개의 모습<sub>flavours</sub>을 둬야 하나?

(WIP)


# 참고자료

* [Lewis Baker - Asymmetric Transfer](https://lewissbaker.github.io)
    * [cppcoro](https://github.com/lewissbaker/cppcoro) 개발자의 코루틴 설명 블로그.
* [cppreference.com - Coroutines](https://en.cppreference.com/w/cpp/language/coroutines.html)


*마지막 수정 : {{ page.last_modified_at }}*
