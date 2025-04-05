---
layout: post
title: Fast-Paced Multiplayer (Part I) - Client-Server Game Architecture
tags: [Netcode]
author: Gabriel Gambetta
last_modified_at: 2025-04-05T14:49:00+09:00
---

본 포스트는 [Gabriel Gambetta의 *Fast-Paced Multiplayer (Part I): Client-Server Game Architecture*](https://www.gabrielgambetta.com/client-server-game-architecture.html)를 한국어로 번역한 것입니다.\
This post is a Korean translation of the [*Fast-Paced Multiplayer (Part I): Client-Server Game Architecture* by Gabriel Gambetta](https://www.gabrielgambetta.com/client-server-game-architecture.html).

시리즈의 다른 글 링크:
1. Client-Server Game Architecture (현재 글)
2. [Client-Side Prediction and Server Reconciliation](/2025/04/05/fast-paced-multiplayer-2)
3. [Entity Interpolation](/2025/04/05/fast-paced-multiplayer-3)
4. [Lag Compensation](/2025/04/05/fast-paced-multiplayer-4)
5. [Live Demo](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)

# Introduction

이 글은 빠른 호흡의 멀티플레이어 게임을 가능토록 하는 테크닉과 알고리즘을 알아보는 시리즈의 첫번째 글이다.
멀티플레이어 게임의 기본 개념에 익숙하다면, 다음 글로 바로 넘어가도 된다. 이번 글은 서론에 해당한다.

게임을 만드는 것 자체가 쉽지 않은 일이다.
허나, 멀티플레이어 게임을 만드려면 완전히 새로운 차원의 문제를 풀어야 한다.
흥미롭게도, 문제의 핵심은 인간 본성과 물리학에 있다.

# The problem of cheating

모든 게 치트 문제<sub>cheating</sub>로부터 시작된다.

게임 개발자로서, 싱글 플레이어 게임에서의 치트<sub>cheat</sub>는 큰 관심거리가 아니다.
치터<sub>cheating player</sub> 자기 혼자에게만 적용되고, 다른 사람에게는 영향이 없으므로.
물론 치터는 게임을 의도된 대로 경험하진 못할 수 있다.
그러나 치터 자신의 게임이므로, 자기가 원하는 방식으로 즐길 권리가 있는 것이다.

하지만 멀티플레이어 게임의 경우는 다르다.
어떤 대전 게임이든, 치터는 자기 게임을 좋게 만드는 것을 넘어, 다른 유저의 게임을 나쁘게 만든다.
치팅은 플레이어가 게임을 떠나게 만드므로, 개발자는 이를 막을 대책을 강구해야 할 것이다.

치트를 막기 위한 수많은 방법이 있으나, 가장 중요한 방법 (그리고 거의 유일하게 의미 있는 방법) 은 단순하다.
*플레이어를 믿지 마라.*
항상 최악의 상황을 상정하라.
즉, 플레이어들은 *치팅을 시도할 것*이라 생각하라.

# Authoritative servers and dumb clients

이는 언뜻 보기에 간단한 해법으로 연결된다.
게임 내 모든 행동을 우리가 관리하는 중앙 서버에서만 일어나도록 하고, 클라이언트들은 그저 게임의 관찰자로 만드는 해법이 그것이다.
달리 말해, 클라이언트는 입력 (버튼 눌림, 명령 등) 만을 서버에 전송하고, 서버가 게임을 돌리고, 그 결과를 클라이언트에게 돌려주는 것이다.
이는 보통 *authoritative server<sub>권위 있는 서버</sub>* 방식이라 부르는데, 월드에서 일어나는 모든 일에 대한 최종 승인권을 서버가 갖기 때문이다.

물론, 서버 취약점으로 인한 치트가 발생할 수 있으나, 그건 본 시리즈의 범위를 벗어난다.
그래도, authoritative server를 사용하면 수많은 핵을 막을 수 있다.
예를 들어, 플레이어의 HP를 판단할 때 클라이언트를 믿지 않는다.
조작된 클라이언트는 자기 플레이어의 HP값을 10000%라고 변조할 수 있지만, 서버는 사실 10%밖에 없음을 *알고 있다*.
조작된 클라이언트가 뭐라 생각하든, 플레이어는 피격당하면 바로 죽을 것이다.

서버는 또한 플레이어의 월드상 위치도 믿지 않는다.
조작된 클라이언트는 서버에게 **"내 위치는 (10,10)"**이라 말하고 1초 뒤에 **"내 위치는 (20,10)"**이라 말할 수 있다.
이걸 믿었다면, 벽을 뚫거나 다른 플레이어보다 빠르게 이동해버릴 수 있다.
그 대신, 서버는 플레이어가 (10,10)에 있음을 *알고 있고*, 클라이언트가 서버에게 **"나 오른쪽으로 1칸 이동하고 싶어"**라고 말하게 한다.
서버는 내부 상태를 업데이트해 플레이어 위치를 (11,10)으로 만들고, 플레이어에게 **"네 위치는 (11,10)"**이라고 회신한다.

<figure style="background-color:#777777;padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="https://www.gabrielgambetta.com/img/fpm1-01.png" style="background-color:#777777"/>
<figcaption aria-hidden="true">단순한 클라이언트-서버 상호작용.</figcaption>
</figure>

요약하면, 게임 상태는 서버만이 유일하게 관리한다.
클라이언트는 자신의 행동을 서버에 전송한다.
서버는 게임 상태를 주기적으로 업데이트하며, 새로운 게임 상태를 클라이언트에게 보내고, 클라이언트는 이를 단순히 화면에 렌더링만<sub>render</sub> 한다.

# Dealing with networks

위에서 다룬 dumb client<sub>멍청한 클라이언트</sub> 방식은 전략 게임이나 포커 같은 느린 턴제 게임에 대해서는 잘 동작한다.
LAN 환경에서도 동작할 것이다. 통신이 즉각적이므로.
하지만 네트워크를 경유하는 빠른 호흡의 게임에 이 방식을 적용하려고 하면 동작하지 않을 것이다.

물리학에 대해 이야기해보자. 샌프란시스코에서 뉴욕에 있는 서버에 접속한다고 해 보자. 이는 대략 4,000 km 이다. (참고로 서울-부산 사이 거리가 400 km 정도 된다.)
그 어떤 것도 빛보다 빠르게 이동할 수 없고, 인터넷을 통한 바이트 전송도 예외는 아니다. (통신도 빛의 pulse, 회선 내의 전자<sub>electron</sub>, 혹은 전자기파로 이루어지는 것이다.)
빛은 대략 시속 300,000 km 이므로, 4,000 km를 이동하는 데는 13 ms 가 소요된다.

꽤 빠르다고 생각할 수 있으나, 이건 실제로는 아주 낙관적인 상황의 예측이다.
데이터가 빛의 속도로 직진한다는 가정이고, 보통의 경우 그렇지 않을 것이다.
현실에서는, 데이터가 라우터와 라우터를 거치며 여러 번의 점프 (네트워크 용어로는 hop이라고 부른다) 를 한다.
이는 빛의 속도로 이루어지지 않는다.
라우터에서 데이터가 복사, 검사, 재라우팅 되기 때문에 다소간 지연이 발생한다.

논의를 위해, 클라이언트에서 서버로 전송되기까지 50 ms 가 걸린다고 해 보자.
이것은 거의 최선의 상황이다.
뉴욕에서 도쿄로 연결한다면 어떨까?
모종의 이유로 네트워크가 혼잡하다면?
100 ms, 200 ms, 심지어 500 ms 의 지연도 발생하는 경우가 있다.

우리 예시로 돌아와서, 클라이언트가 서버에게 **"나 오른쪽 방향키 눌렀어"**라고 보낸다 하자.
서버는 이걸 50 ms 뒤에 받는다.
서버가 이 요청을 받는 즉시 처리해서 업데이트된 상태를 보낸다 치자.
클라이언트가 새로운 게임 상태 메시지 **"네 위치는 (1,0)"**을 받는데 50 ms 가 추가로 소요된다.

플레이어 관점에서 보면, 오른쪽 방향키를 눌렀는데 0.1초 동안 아무런 반응이 없는 것으로 보인다.
그런 이후에야 캐릭터가 오른쪽으로 1칸 이동한다.
이 *랙<sub>lag</sub>*이 얼마 안 돼 보일지 모르지만, 생각보다 알아차리기 쉽다.
그리고 당연히, 0.5초의 랙은 알아채기 쉬운 정도가 아니라, 아예 게임 플레이를 불가능하게 만든다.

# Summary

네트워크 멀티플레이어 게임은 아주 재밌지만, 완전히 새로운 차원의 문제를 야기한다.
Authoritative server 구조는 대부분의 치트를 방지하지만, 이걸 그대로 구현하면 게임의 반응성이 떨어진다.

앞으로의 글에서는, authoritative server를 기반으로 하되, 플레이어가 느끼는 지연을 최소화하여,
로컬 멀티플레이나 싱글 플레이와 구분이 거의 불가능할 정도의 시스템을 만드는 방법을 살펴보도록 하겠다.

**[Part II: Client-Side Prediction and Server Reconciliation >>](/2025/04/05/fast-paced-multiplayer-2)**

*마지막 수정 : {{ page.last_modified_at }}*
