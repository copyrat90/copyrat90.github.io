---
layout: post
title: Fast-Paced Multiplayer (Part III) - Entity Interpolation
tags: [Netcode]
author: Gabriel Gambetta
last_modified_at: 2025-04-05T21:34:00+09:00
---

본 포스트는 [Gabriel Gambetta의 *Fast-Paced Multiplayer (Part III): Entity Interpolation*](https://www.gabrielgambetta.com/entity-interpolation.html)를 한국어로 번역한 것입니다.\
This post is a Korean translation of the [*Fast-Paced Multiplayer (Part III): Entity Interpolation* by Gabriel Gambetta](https://www.gabrielgambetta.com/entity-interpolation.html).

시리즈의 다른 글 링크:
1. [Client-Server Game Architecture](/2025/04/05/fast-paced-multiplayer-1)
2. [Client-Side Prediction and Server Reconciliation](/2025/04/05/fast-paced-multiplayer-2)
3. Entity Interpolation (현재 글)
4. [Lag Compensation](/2025/04/05/fast-paced-multiplayer-4)
5. [Live Demo](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)

# Introduction

시리즈의 [첫번째 글](/2025/04/05/fast-paced-multiplayer-1)에서, *authoritative server*와 그것이 클라이언트 치트를 막는데 효과적임을 소개했다.
그러나, 이 테크닉을 순진하게<sub>naively</sub> 적용하면 플레이 가능성<sub>playability</sub>과 반응성<sub>responsiveness</sub>을 가로막는 잠재적 방해 요인<sub>showstopper</sub>이 될 수 있다.
[두번째 글](/2025/04/05/fast-paced-multiplayer-2)에서는, 이런 한계를 극복하는 방법으로 *클라이언트 측 예측<sub>client-side prediction</sub>*을 제안했다.

두 글의 내용<sub>net result</sub>은 전송 지연이 있는 authoritative server와 연결되어도, 플레이어가 게임 내 캐릭터를 싱글 플레이어와 똑같은 느낌으로 조작할 수 있게 하는 개념과 테크닉이었다.

이 글에서는, 다른 플레이어가 조작하는 캐릭터가 동일한 서버에 연결된 경우 나타날 수 있는 일에 대해 다룬다.

# Server time step

이전 글에서, 우리가 묘사한 서버의 동작은 꽤 단순했다.
클라이언트의 입력을 읽고, 게임 상태를 업데이트 한 후, 클라이언트에게 되돌려주는 것이었다.
그러나, 1명 이상의 클라이언트가 연결되는 경우, 메인 서버 루프는 다소 달라지게 된다.

그런 경우, 여러 클라이언트가 빠른 주기로 동시에 입력을 보낼 수 있다.
(플레이어가 방향키를 누르든, 마우스를 움직이든, 클릭을 하든 명령을 생성하는 주기로 보낼 것이다.)
매번 입력을 받을 때마다 게임 월드를 업데이트하고 게임 상태를 전파<sub>broadcast</sub>하는 것은 너무 많은 CPU 시간과 대역폭<sub>bandwidth</sub>을 낭비할 것이다.

더 나은 방법론은 클라이언트의 입력을 받을 때, 어떤 처리도 하지 않고 대기열에 넣는<sub>queue</sub> 것이다.
그 대신, 게임 월드는 예를 들어 1초에 10번 정도의 낮은 주기로 업데이트된다.
매 업데이트 사이의 지연을 (이 경우 100 ms) 일컬어 time step<sub>단위 시간</sub>이라 한다.
매 업데이트 루프 반복마다, 처리되지 않은 클라이언트 입력 전체가 적용되고 (물리를 더 예측 가능하게 만들기 위해, time step보다 작은 단위의 시간 증가값으로 처리될 수 있다), 새로운 게임 상태가 클라이언트들에게 전파된다.

요약하면, 게임 월드는 클라이언트 입력 여부와 그 양에 관계 없이, 예측 가능한 빈도<sub>predictable rate</sub>로 업데이트된다.

# Dealing with low-frequency updates

클라이언트의 관점에서, 이 방식은 기존과 같이 부드럽게<sub>smoothly</sub> 동작한다.
클라이언트 측 예측은 업데이트 지연과 독립적으로 작동하므로, 예측 가능하지만 다소 드문 빈도<sub>relatively infrequent</sub>의 상태 업데이트에 대해서도 잘 동작할 것이다.
그러나, 게임 상태가 낮은 빈도로 전파되므로 (기존 예시를 계속하면, 매 100 ms 마다), 클라이언트는 월드를 돌아다니는 다른 개체들<sub>entities</sub>에 대해 너무 듬성듬성<sub>sparse</sub>한 정보만을 얻게 된다.

첫번째 구현은 상태 업데이트를 받을 때에 다른 캐릭터의 위치를 업데이트하는 것이다.
이는 곧바로 뚝뚝 끊어지는<sub>choppy</sub> 움직임을 야기하는데, 그 말은, 부드럽게 움직이는 대신 100 ms 마다 띄엄띄엄<sub>discrete</sub> 점프를 한다는 의미다.

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm3-01.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>클라이언트 2가 바라본 클라이언트 1.</i></figcaption>
</figure>

이 문제에 대한 대응책은 개발하고 있는 게임의 종류에 따라 여러 가지로 나뉜다.
일반적으로는, 게임 개체들<sub>entities</sub>이 예측 가능할수록, 올바르게 처리하기 쉽다.

# Dead reckoning

자동차 레이싱 게임을 만들고 있다고 해보자.
아주 빠르게 달리는 자동차의 움직임은 꽤 예측이 쉽다.
예를 들어, 차가 1초에 100m 를 움직인다면, 1초 뒤 시작 지점부터 대략 100m 앞에 있을 것으로 예측할 수 있다.

왜 "대략"인가?
그 1초 사이에 자동차가 약간 가속하거나 감속했을 수 있고, 좌측이나 우측으로 약간 회전했을 수 있다.
여기서의 핵심 단어는 "약간"이다.
고속에서는 자동차의 기동성<sub>maneuverability</sub> 때문에, 플레이어의 실제 행동을 고려하더라도, 특정 시점의 위치는 직전 위치, 속력 및 방향에 상당히 의존적이다.
다른 말로 하면, 레이싱 카는 한 순간에 180° 회전을 할 수 없다.

이 사실을 100 ms 마다 업데이트를 보내는 서버와의 통신에서 어떻게 이용할 수 있을까?
클라이언트는 각 상대 차량에 대한 권위 있는<sub>authoritative</sub> 속력과 방향<sub>heading</sub> 정보를 받는다.
다음 100 ms 동안은 새로운 정보를 받을 수 없지만, 여전히 화면에는 다른 차들이 달리는 모습을 보여줘야 한다.
가장 간단한 방법은 차의 방향과 가속이 그 100 ms 동안 일정할 것이라 가정하고, 그 차의 물리를 로컬에서 연산하는 것이다.
그 후, 100 ms 가 지나 서버의 업데이트가 도착하면, 차의 위치가 조정된다<sub>corrected</sub>.

이 조정<sub>correction</sub>은 여러 요인에 따라 클 수도 작을 수도 있다.
만약 플레이어가 실제로 차를 일직선상에서 속력 변화 없이 몰았다면, 예측된 위치는 조정할 위치와 정확히 같을 것이다.
반면에, 플레이어가 뭔가와 부딪혔다면, 예측한 위치는 완전히 틀렸을 것이다.

Dead reckoning<sub>추측 항법</sub>은 전함과 같은 느린 속력의 상황에서도 쓸 수 있다는 점에 주목하라.
사실, "dead reckoning<sub>추측 항법</sub>"이라는 용어 자체가 해양 항해<sub>marine navigation</sub>에서 온 말이다.

# Entity interpolation

상황에 따라 dead reckoning을 전혀 적용할 수 없을 수도 있다.
특히, 플레이어의 방향과 속력이 즉각적으로 바뀔 수 있는 모든 상황에서 그러하다.
예를 들어, 3D 슈팅 게임에서, 플레이어들은 아주 빠른 속도로 달리고, 멈추고, 직각으로 꺾는데<sub>turn corners</sub>,
이는 dead reckoning을 본질적으로 쓸모 없게 만든다.
새로운 위치와 속력이 이전 데이터로부터 예측될 수 없기 때문이다.

서버가 권위적인<sub>authoritative</sub> 데이터를 보낼 때에만 플레이어 위치를 업데이트 할 수는 없다.
그렇게 되면 플레이어들이 100 ms 마다 짧은 거리를 순간이동하게 되고, 그래서는 게임을 플레이 할 수가 없다.

우리가 가진 것은 매 100 ms 마다 받는 권위적인<sub>authoritative</sub> 위치 데이터이다.
비결<sub>trick</sub>은 플레이어에게 그 중간에 일어나는 일을 어떻게 보여주느냐다.
해결책의 핵심은 유저의 플레이어의 비해 다른 플레이어들은 *과거의 모습으로* 보여주는 것이다.

**t = 1000**에 위치 정보를 받았다고 해보자.
이미 **t = 900**에 위치 정보를 받았으므로, 다른 플레이어가 **t = 900**과 **t = 1000**에 어디에 있었는지는 알고 있다.
그러므로, **t = 1000**에서 **t = 1100** 동안, 다른 플레이어가 **t = 900**에서 **t = 1000** 사이에 한 일을 보여주면 된다.
이 방법대로면 항상 유저에게 *실제 움직임 데이터*를 보여줄 수 있다.
다만 이를 100 ms "늦게" 보여줄 뿐이다.

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm3-02.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>클라이언트 2는 클라이언트 1의 알려진 위치 사이를 보간하여, "과거를 렌더링"한다.</i></figcaption>
</figure>

**t = 900**에서 **t = 1000** 사이를 보간<sub>interpolate</sub>하기 위해 사용하는 위치 데이터는 게임에 따라 달라진다.
보통은 보간<sub>interpolation</sub>이면 충분하다.
그렇지 못한 경우, 서버에서 매 업데이트마다 더 디테일한 이동 데이터를 보내도록 할 수 있다.
예를 들어, 플레이어 정보 뒤에 직진한 구간의 목록<sub>sequence of straight segments</sub>을 붙여 보내거나, 10 ms 마다 샘플링한 위치 정보를 보내 보간을 더 보기 좋게 만들 수도 있다.
(10배나 더 많은 데이터를 보낼 필요는 없다. 작은 움직임의 변화값들<sub>deltas</sub>만을 보내는 것이므로, 통신에 사용할 포맷<sub>format on the wire</sub>은 이를 고려해 특수한 최적화를 하면 된다.)

이 테크닉을 사용하겠다면, 모든 플레이어가 서로 약간 다른 게임 월드를 보게 된다는 점을 기억해야 한다.
왜냐하면 각 플레이어는 자기 자신은 *현재 모습*을 보지만, 다른 개체들<sub>entities</sub>은 *과거 모습*으로 보기 때문이다.
그러나, 호흡이 빠른 게임이라 할 지라도, 다른 개체들이 100 ms 지연되어 보이는 정도는 알아채기 어렵다.

예외는 있다.
플레이어가 다른 플레이어를 사격하는 등 시간, 공간적으로 정밀해야 하는 경우이다.
다른 플레이어는 과거의 모습으로 보고 있으므로, 100 ms 의 지연 시간을 갖고 조준하고 있는 것이다.
다시 말해, 목표물의 100 ms 이전 위치에 대고 사격하고 있는 것이다!
이걸 처리하는 법은 다음 글에서 다루도록 하자.

# Summary

드문 빈도<sub>infrequent</sub>의 업데이트와 네트워크 지연이 있는 Authoritative server를 사용하는 클라이언트-서버 환경에서도, 여전히 플레이어에게 연속성의 환상<sub>illusion of continuity</sub>과 부드러운 움직임을 제공해야 한다.
[시리즈의 2편](/2025/04/05/fast-paced-multiplayer-2)에서 유저가 조작하는 플레이어의 이동을 실시간으로 보여 주기 위해, 클라이언트 측 예측과 서버 조율<sub>server reconciliation</sub>을 활용하였다.
이는 사용자 입력이 로컬 플레이어에게 즉시 적용되도록 하여, 게임플레이를 불가능하게 만드는 렌더링 지연을 제거했다.

그러나, 다른 개체들은 여전히 문제였다.
이번 글에서 이를 해결하기 위한 2가지 다른 방법을 살펴보았다.

첫번째 방법은, dead reckoning<sub>추측 항법</sub>으로, 위치, 속도와 가속 등 이전 데이터로부터 개체의 현재 위치를 잘 추정할 수 있는 종류의 시뮬레이션에 적용할 수 있는 방법론이다.
이 방법론은 추측이 어려운 상황에서는 쓸 수 없다.

두번째 방법은, entity interpolation<sub>개체 보간</sub>으로, 미래를 전혀 예측하지 않는다.
오로지 서버로부터 받은 실제 개체 데이터만을 사용하므로, 다른 개체들이 살짝 지연되어 보이게 된다.

결론적인 효과<sub>net effect</sub>는 유저의 플레이어는 *현재 모습*으로 보이지만, 다른 개체들은 *과거 모습*으로 보인다는 것이다.
이는 보통은 아주 매끄러운<sub>seamless</sub> 경험을 선사한다.

그러나, 다른 방책이 없으면, 이동하는 목표물을 사격하는 등 시공간적으로 정밀해야 하는 경우 이런 환상은 무너지게 된다.
클라이언트 2에서 렌더링하는 클라이언트 1의 위치는, 서버에서의 위치와도, 클라이언트 1에서의 위치와도 맞지 않으며, 헤드샷을 불가능하게 만든다!
헤드샷이 없으면 성립하는 게임은 없으므로, 다음 글에서 이 문제를 다뤄보도록 하겠다.

[<< Part II: Client-Side Prediction and Server Reconciliation](/2025/04/05/fast-paced-multiplayer-2) · **[Part IV: Lag Compensation >>](/2025/04/05/fast-paced-multiplayer-4)**

*마지막 수정 : {{ page.last_modified_at }}*
