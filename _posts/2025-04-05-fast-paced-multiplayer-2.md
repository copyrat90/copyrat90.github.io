---
layout: post
title: Fast-Paced Multiplayer (Part II) - Client-Side Prediction and Server Reconciliation
tags: [Netcode]
author: Gabriel Gambetta
last_modified_at: 2025-04-05T21:38:00+09:00
---

본 포스트는 [Gabriel Gambetta의 *Fast-Paced Multiplayer (Part II): Client-Side Prediction and Server Reconciliation*](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)를 한국어로 번역한 것입니다.\
This post is a Korean translation of the [*Fast-Paced Multiplayer (Part II): Client-Side Prediction and Server Reconciliation* by Gabriel Gambetta](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html).

시리즈의 다른 글 링크:
1. [Client-Server Game Architecture](/2025/04/05/fast-paced-multiplayer-1)
2. Client-Side Prediction and Server Reconciliation (현재 글)
3. [Entity Interpolation](/2025/04/05/fast-paced-multiplayer-3)
4. [Lag Compensation](/2025/04/05/fast-paced-multiplayer-4)
5. [Live Demo](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)

# Introduction

시리즈의 [첫번째 글](/2025/04/05/fast-paced-multiplayer-1)에서, authoritative server와 dumb client를 사용한 클라이언트-서버 모델을 살펴보았다.
이는 클라이언트는 단순히 입력만을 서버에 전송하고, 서버로부터 업데이트된 게임 상태를 받으면 렌더링만을<sub>render</sub> 하는 방식이다.

이 방식을 순진하게<sub>naive</sub> 구현하면 유저의 명령과 화면의 변화 사이에 지연 시간을 야기한다.
예를 들어, 플레이어가 오른쪽 방향키를 누르면, 캐릭터가 이동을 시작하는 데까지 0.5초가 걸릴 수 있다.
그 이유는 클라이언트의 입력이 서버까지 전송되고, 서버가 입력을 처리하고 새로운 게임 상태를 연산한 후, 업데이트된 게임 상태가 클라이언트까지 돌아와야 하기 때문이다.

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm2-01.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>네트워크 지연의 결과.</i></figcaption>
</figure>

인터넷과 같이 수십~수백 밀리초의 지연이 발생할 수 있는 네트워크 환경에서, 이 방식은 좋은 경우에도 반응성이 낮게 느껴지며, 최악의 경우에는 전혀 플레이가 불가능 할 수 있다.
이번 글에서는, 이 문제를 최소화하거나 아예 없애버리는 방법을 찾아보도록 하겠다.

# Client-side prediction

비록 치터들<sub>cheating players</sub>이 있긴 하지만, 대부분의 시간 동안 게임 서버는 정상적인 요청을 처리한다 (정상 클라이언트의 모든 요청과 치터가 치트를 안 쓰는 때의 요청).
이는 들어오는 입력 대부분이 정상이고 게임 상태를 예측한 대로 업데이트 시킨다는 말이다.
그 말은, 캐릭터가 (10,10)에 있는 상태에서 오른쪽 방향키를 눌렀다면, (11,10)으로 이동할 것이라는 의미다.

이 점을 이용할 수 있다.
게임 월드가 충분히 *결정론적<sub>deterministic</sub>*이라면 (다시 말해, 주어진 게임 상태와 입력에 대해, 결과를 완벽히 예측할 수 있다면),
입력을 서버에 전송하는 *즉시* 클라이언트에서 입력을 처리할 수 있다.
이 말은, 서버가 입력을 처리하여 만들 게임 상태를 클라이언트 측에서 *예측*한다는 말이다.
이를 통해 입력을 하고 나서 그 효과를 렌더링하기까지의 지연 시간을 제거할 수 있다.
게다가, 대부분의 경우 이 예측은 정확할 것이므로, 서버가 업데이트된 게임 상태를 전송하고 나서도 시각적인 불일치<sub>visible mismatch</sub>가 발생하는 경우는 없을 것이다.

예를 들어 100 ms 랙<sub>lag</sub>이 있고, 캐릭터가 다음 칸으로 이동하는 애니메이션이 100 ms 가 걸린다고 해보자.
순진한<sub>naive</sub> 구현에서, 전체 동작은 200 ms 가 소요될 것이다:

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm2-02.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>네트워크 지연 + 애니메이션.</i></figcaption>
</figure>

월드가 결정론적이므로, 서버에 전송한 입력이 성공적으로 실행될 거라 가정할 수 있다.
이 가정 하에서, 클라이언트는 입력이 처리된 뒤의 게임 상태를 예측할 수 있고, 대부분의 경우 이 예측은 맞아떨어질 것이다.

입력을 보낸 후 게임 상태가 오길 기다렸다가 렌더링하는 대신, 입력을 보내자마자 성공했다고 가정하고 결과를 렌더링할 수 있다.
그 사이에 "진짜<sub>true</sub>" 게임 상태가 도착하고, 보통의 경우, 로컬 측에서 계산한 결과와 일치할 것이다:

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm2-03.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>서버가 행동을 확인하는 동안 애니메이션이 재생된다.</i></figcaption>
</figure>

이제 플레이어의 행동과 화면상 결과 사이에 지연이 전혀 없으면서, 서버는 여전히 authoritative하다.
(만일 조작된 클라이언트가 비정상적인 입력을 보낸다면, 자기 화면에는 자기 마음대로 렌더링하겠지만,
서버의 상태에는 영향을 미치지 못할 것이며, 다른 플레이어에게는 보이지 않을 것이다.)

# Synchronization issues

위 예시에서, 필자는 모든 게 잘 작동하는 수치를 세심하게 골랐다.
그러나, 약간 변형된 시나리오를 생각해보자.
서버와의 사이에 250 ms 랙이 있고, 다음 칸으로 이동하는 데 100 ms 가 걸린다고 해보자.
그리고 플레이어가 오른쪽 방향키를 2번 연속 눌러, 오른쪽으로 2칸 이동하려 한다 해보자.

지금까지의 테크닉을 사용한다면, 이런 일이 발생할 것이다:

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm2-04.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>예측한 상태와 authoritative<sub>승인된</sub> 상태 간 불일치.</i></figcaption>
</figure>

새로운 게임 상태가 도착한 **t = 250 ms** 시점에, 흥미로운 문제가 발생했다.
클라이언트가 예측한 상태는 **x = 12**인데, 서버는 새로운 게임 상태가 **x = 11**이라고 말한다.
서버가 최종 결정권을 가지므로<sub>authoritative</sub>, 클라이언트는 캐릭터 위치를 **x = 11**로 되돌려야 한다.
그런데 그 후, **t = 350**에 새로운 서버 상태가 도착하고, **x = 12**라고 하므로, 캐릭터가 이번엔 앞으로 워프를 한다.

플레이어의 관점에서 보면, 오른쪽 방향키를 2번 누른 이후, 캐릭터가 오른쪽으로 2칸 이동했고,
거기 50 ms 동안 서 있다가, 갑자기 왼쪽으로 1칸 워프했다가, 100 ms 동안 서 있다가, 오른쪽으로 1칸 워프한다.
당연히 이대로 놔둘 수는 없다.

# Server reconciliation

이 문제를 해결하려면, 클라이언트는 게임 월드의 *현재 시간*을 보지만, 서버에서 도착하는 업데이트는 랙 때문에 *과거 시점*이라는 점을 이해해야 한다.
서버가 업데이트된 게임 상태를 보내는 시점에는, 클라이언트의 모든 명령을 처리한 게 아니다.

이 문제를 회피하는 게 그리 어려운 건 아니다.
우선, 클라이언트는 각 요청에 sequence number<sub>시퀀스 번호</sub>를 추가한다.
우리 예시에서는, 첫번째 키 입력은 요청 #1, 두번째 키 입력은 요청 #2 이다.
그 다음, 서버가 응답을 할 때, 마지막으로 처리한 입력의 sequence number를 포함해 보낸다:

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm2-05.png" style="background-color:#777777"/>
<figcaption aria-hidden="true"><i>클라이언트 측 예측 + server reconciliation<sub>서버 조율</sub>.</i></figcaption>
</figure>

이제, **t = 250** 시점에, 서버가 **"네 요청 #1 까지 본 결과, 네 위치는 x = 11"**이라고 말한 것이 된다.
서버가 권위적이므로<sub>authoritative</sub>, 캐릭터 위치는 **x = 11**로 설정된다.
이번엔 클라이언트가 서버로 보낸 각 요청을 저장해둔다고 하자.
새로 도착한 게임 상태를 보면, 서버가 이미 요청 #1 을 처리했다는 걸 알 수 있으므로, 그 요청은 이제 버려도 된다.
하지만 아직 요청 #2 에 대한 결과는 서버로부터 오지 않았다는 것도 알 수 있다.
그러므로 클라이언트 측 예측을 다시 적용하여, 클라이언트는 서버가 보낸 마지막 authoritative 상태에서부터, 서버에서 아직 처리되지 않은 입력을 더하여 "현재" 상태를 추정할 수 있다.

**t = 250** 시점에, 클라이언트는 **"네 요청 #1 까지 본 결과, 네 위치는 x = 11"** 메시지를 받는다.
클라이언트는 #1 까지의 전송했던 입력을 버리지만, 아직 서버에서 승인<sub>acknowledge</sub>받지 않은 #2 는 유지한다.
클라이언트는 내부 게임 상태를 서버가 보낸 **x = 11**까지 업데이트하고, 아직 서버에서 확인하지 않은 모든 입력을 적용한다.
이 경우, 입력 #2, **"오른쪽으로 1칸 이동"**을 적용한다.
결과는 **x = 12**이고, 올바르다.

예제를 계속하면, **t = 350** 시점에 서버로부터 **"x = 12, 마지막으로 처리된 요청 = #2"** 메시지가 도착한다.
이 시점에, 클라이언트는 #2 까지의 입력을 모두 버리고, 상태를 **x = 12**로 업데이트한다.
더 이상 다시 실행할<sub>replay</sub> 미처리 입력이 없으므로, 처리는 거기서 끝나고, 결과는 올바르다.

# Odds and ends

위에서 다룬 예제는 이동 처리를 다루지만, 같은 원리를 거의 모든 것에 적용할 수 있다.
예를 들어, 턴제 대전 게임에서, 플레이어가 다른 캐릭터를 공격할 때,
유혈이나 데미지 수치를 보여줄 수 있지만, 서버가 대답하기 전까지는 실제 캐릭터의 체력을 업데이트 해서는 안 된다.

쉽게 되돌릴 수 없는 복잡한 게임 상태의 경우, HP가 0이라고 하더라도 서버에서 결과가 올 때까지 캐릭터를 죽이는 것을 피해야 할 수도 있다.
(만일 상대 캐릭터가 죽기 직전 구급상자를 사용했는데, 서버가 아직 그걸 알려주지 않았다면 어쩔 것인가?)

이건 흥미로운 점이다.
월드가 완벽히 결정론적이고 치터가 전혀 없어도, 클라이언트가 예측한 상태와 서버에서 도착한 상태가 reconciliation<sub>조율</sub> 이후엔 맞지 않을 수 있다.
싱글 플레이어에서는 불가능한 시나리오이지만, 서버에 여러 명의 플레이어가 있는 상황에서는 흔히 나타날 수 있는 상황이다.
다음 글에서는 이에 대해 다루겠다.

## Summary

Authoritative server를 사용할 때, 플레이어에겐 반응성이라는 환상<sub>illusion of responsiveness</sub>을 보여 주되, 서버에서 실제로 입력이 처리되기를 기다려야 한다.
그러기 위해, 클라이언트는 입력의 결과를 시뮬레이션<sub>simulates</sub>한다.
업데이트된 서버 상태가 도착하면, 예측했던 클라이언트 상태는 업데이트된 상태와 전송했으나 아직 서버가 승인<sub>acknowledge</sub>하지 않은 입력으로부터 재계산된다<sub>recomputed</sub>.

[<< Part I: Client-Server Game Architecture](/2025/04/05/fast-paced-multiplayer-1) · **[Part III: Entity Interpolation >>](/2025/04/05/fast-paced-multiplayer-3)**

*마지막 수정 : {{ page.last_modified_at }}*
