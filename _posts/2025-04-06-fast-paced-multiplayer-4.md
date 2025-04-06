---
layout: post
title: Fast-Paced Multiplayer (Part IV) - Lag Compensation
tags: [Netcode]
author: Gabriel Gambetta
last_modified_at: 2025-04-06T10:41:00+09:00
---

본 포스트는 [Gabriel Gambetta의 *Fast-Paced Multiplayer (Part IV): Lag Compensation*](https://www.gabrielgambetta.com/lag-compensation.html)를 한국어로 번역한 것입니다.\
This post is a Korean translation of the [*Fast-Paced Multiplayer (Part IV): Lag Compensation* by Gabriel Gambetta](https://www.gabrielgambetta.com/lag-compensation.html).

시리즈의 다른 글 링크:
1. [Client-Server Game Architecture](/2025/04/05/fast-paced-multiplayer-1)
2. [Client-Side Prediction and Server Reconciliation](/2025/04/05/fast-paced-multiplayer-2)
3. [Entity Interpolation](/2025/04/05/fast-paced-multiplayer-3)
4. Lag Compensation (현재 글)
5. [Live Demo](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)

# Introduction

이전 3개 글에서 아래와 같이 요약할 수 있는 클라이언트-서버 게임 구조에 대해 다뤘다.
* 서버는 모든 클라이언트로부터 시간 정보<sub>timestamp</sub>를 포함한 입력을 받는다
* 서버는 입력을 처리하고, 월드 상태를 업데이트한다
* 서버는 모든 클라이언트에게 평범한 월드 스냅샷<sub>regular world snapshots</sub>을 보낸다
* 클라이언트는 입력을 보내고 그 효과를 로컬에서 시뮬레이션한다
* 클라이언트가 월드 업데이트를 받으면
  * 예측된 상태를 승인된<sub>authoritative</sub> 상태로 동기화한다<sub>syncs</sub>
  * 다른 개체들에 대해서는, 알려진 과거 상태들을 가지고 보간<sub>interpolates</sub>한다

플레이어의 관점에서, 이는 2개의 중요한 결과를 야기한다:
* 플레이어는 **자기 자신**을 **현재 모습**으로 본다
* 플레이어는 **다른 개체**들을 **과거 모습**으로 본다

이런 상황은 일반적으로 문제가 없지만, 시간, 공간적으로 민감한 사건에 대해서는 꽤 골치아픈 문제가 생긴다.
예를 들어, 적의 머리를 사격하는 상황에서 말이다!

# Lag Compensation

저격총으로 목표물의 머리를 완벽히 정조준하고 있다.
탕! 절대 빗나갈 수가 없는 한발이다.

그런데 빗나간다.

왜 빗나갔을까?

전에 설명했던 클라이언트-서버 아키텍처 때문에, 적 머리의 *현 위치*가 아닌, 100 ms *이전*에 있었던 위치에 대고 조준하고 있는 것이다!

어찌 보면, 게임을 빛의 속도가 아주 느린 우주에서 플레이하고 있다고 볼 수 있다.
적의 과거 위치를 보면서 조준하고 있으나, 방아쇠를 당기기 오래 전에 적은 이미 다른 곳으로 이동한 것이다.

다행히, *대부분의* 플레이어가 *대체로* 만족하는 비교적 간단한 해결책이 있다.
(만족하지 않는 1가지 사례는 아래에서 살펴볼 것이다.)

원리를 설명하자면 이렇다:
* 총을 발사할 때, 클라이언트는 시간 정보<sub>timestamp</sub>, 조준 위치를 포함한 전체 이벤트 정보를 서버로 송신한다.
* **가장 중요한 단계다.** 서버가 모든 입력을 시간 정보와 함께 받으므로, 과거 어떤 시점의 월드든 정확하게<sub>authoritatively</sub> 재현할 수 있다.
  구체적으로 말하면, 특정 클라이언트가 어떤 시점에 보는 월드 상태를 정확하게 재현할 수 있다.
* 이 말은, 서버가 유저가 총을 쏘던 시점에 시야에 있던 목표물을 정확히 알 수 있다는 말이다.
  목표물은 적 머리의 *과거* 위치였지만, 서버는 그게 *해당 유저* 입장에서 현재 위치였음을 알 수 있다.
* 서버가 사격을 *그 시점 기준으로* 처리하고, 클라이언트를 업데이트한다.

이제 모두가 행복해졌다!

서버는 서버이기 때문에 언제나 그렇듯 행복하다.

총을 쏜 유저는 적의 머리를 조준, 사격하고, 짜릿한<sub>rewarding</sub> 헤드샷을 맞춰서 행복하다.

적은 그닥 행복하지 않은 유일한 사람이다.
가만히 있다가 맞았다면, 자기 잘못이겠지?
움직이고 있다가 맞았으면... 우와, 총 쏜 사람이 대단한 저격수였나보네.

하지만 적이 노출된 위치에 있다가, 벽 뒤로 안전하게 숨고, *그 뒤에* 총에 맞는다면 어떨까?

그런 일이 실제로 발생할 수 있다.
그것이 이 해결책을 사용한 대가<sub>tradeoff</sub>이다.
과거의 적을 사격할 수 있게 되었기 때문에, 은폐를 했다고 해도 몇 밀리초 동안은 총에 맞을 가능성이 생긴다.

약간의 부조리는 있지만, 그래도 이것이 모두가 동의할 만한 가장 합리적인 해결책이다.
빗나갈 수가 없는 사격이 빗나간다면 그게 훨씬 나쁠 테니까!

# Conclusion

빠른 호흡의 멀티플레이어 게임에 관한 시리즈는 이걸로 끝이다.
이런 것은 확실히 제대로 만들기 쉽지 않지만, 정확한 개념적 이해가 있다면, 불가능할 정도로 어려운 건 아니다.

작성한 글의 대상 독자는 게임 개발자들이었지만, 게이머들도 흥미롭게 읽을 수 있겠다!
게이머의 관점에서 왜 어떤 일이 그렇게 벌어지는 지를 이해하는 것은 흥미로울 테니.

## Further Reading

여기서 다뤘던 테크닉들은 꽤나 영리하지만, 필자가 만든 것이 아니다.
필자가 작성한 글은 다른 글, 소스 코드와 같은 외부 매체와 약간의 실험을 통해 필자가 배운 것을 그저 이해하기 쉽게 정리한 가이드에 불과하다.

이 주제에 관련된 읽어볼만한 글은 [What Every Programmer Needs to Know About Game Networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/)과 [Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)이 있다.

[<< Part III: Entity Interpolation](/2025/04/05/fast-paced-multiplayer-3) · **[Live Demo >>](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)**

*마지막 수정 : {{ page.last_modified_at }}*
