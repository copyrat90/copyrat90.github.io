---
layout: post
title: Fight the Lag! The Trick Behind GGPO's Low Latency Netcode (WIP)
tags: [Netcode]
author: Tony Cannon
last_modified_at: 2025-04-10T14:06:00+09:00
---

본 포스트는 [Tony Cannon의 *Fight the Lag! The Trick Behind GGPO's Low Latency Netcode*](https://drive.google.com/open?id=1cV0fY8e_SC1hIFF5E1rT8XRVRzPjU8W9)를 한국어로 번역한 것입니다.\
This post is a Korean translation of the [*Fight the Lag! The Trick Behind GGPO's Low Latency Netcode* by Tony Cannon](https://drive.google.com/open?id=1cV0fY8e_SC1hIFF5E1rT8XRVRzPjU8W9).

> 콘솔/PC에서 판매중인 명작 멀티플레이어 아케이드 게임들 몇몇은 온라인으로 플레이하면 랙 때문에 엉망이 된다.
> 그 이유는 간단하다. 정밀한 타이밍과 컨트롤이 중요한 게임을 하는 경우 (예를 들어 벨트스크롤 액션 게임<sub>brawlers</sub>, 슈팅 게임<sub>shoot'em-ups</sub>, 격투 게임<sub>fighting games</sub>), 최대한 랙이 없는 환경에서 하지 않는다면, 다들 자기 생각대로 제대로 플레이가 안 된다고 느낄 것이기 때문이다.
> 이 글에서는, 필자가 GGPO 넷코드<sub>netcode</sub> ("Good Game, Peace Out"의 약자) 를 어떻게 설계했는지를 다룰 것이다.
> (GGPO는 *스컬걸즈*, *스트리트 파이터 3 서드 스트라이크 온라인 에디션*과, FinalBurn Alpha 에뮬레이터로 구동되는 수많은 아케이드 게임에 사용되었다.)
> 온라인 멀티플레이어에서의 랙을 숨기고 최소화하려는 개발자들에게 유용할 것이다. 특히 실력이 타이밍으로 좌우되는 게임에 말이다.
>
> GGPO는 이하 조건을 만족하는 모든 아케이드 스타일 게임에 적용할 수 있다: 1) 모든 게임 시뮬레이션 상태의 업데이트는 오로지 이전 시뮬레이션 상태와 플레이어 입력에 의해 결정론적<sub>deterministically</sub>으로 이루어져야 하고, 2) 컨트롤러 입력, 화면 렌더링, 오디오와는 별개로 시뮬레이션 상태를 업데이트할 수 있어야 하며, 3) 필요시 시뮬레이션 상태를 저장하고 복원할 수 있어야 한다.
> 이 모든 조건을 만족한다면 나머지 세부 사항은 GGPO가 책임진다.
> 게임 개발자는 그저 게임 루프를 수정해 추측 기반 실행<sub>speculative execution</sub>을 허용하기만 하면 된다.

# Identifying the checkpoint

멀티플레이어 게임에 온라인 지원을 추가하는 제일 흔한 방법은 각 기기에서 시뮬레이션을 돌리되, 각 시뮬레이션이 완전히 똑같은 입력을 받도록 하여 동기화를 유지하는 방법이다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig1.svg" width="802px"/>
<figcaption aria-hidden="true">그림 1: 전통적인 프레임 지연<sub>frame-delay</sub> 방식의 네트워크 플레이 구현.</figcaption>
</figure>

시뮬레이션이 오로지 입력에 의해서만 결정<sub>determined</sub>된다면, 양쪽 기기에서 같은 입력을 하는 것으로 같은 결과를 재현할 수 있다.
이 방법은 장점이 많다.
컴퓨터는 태생적으로 결정론적이고 대부분의 아케이드 게임이 이미 정수 연산만을 수행한다. (정수 연산은 서로 다른 프로세서 간에도 보통 같은 결과를 도출한다.)
이를 통해 개발자는 네트워크 구현의 세부 사항으로부터 완전히 분리될 수 있다.
게임이 결정론적으로 수행되도록 하는 문제는 제쳐두면, 게임플레이를 구현하는 엔지니어와 디자이너들은 네트워크 엔진의 세부 사항과는 분리되어 작업할 수 있다.
이 말은 이미 출시된 게임에 온라인 멀티플레이어를 차후에 추가<sub>retroactively add</sub>할 수 있다는 말이다.
(게임이 작성되었던 원본 하드웨어의 에뮬레이터를 제공한다면, 심지어 게임을 재컴파일 할 필요도 없다!)
마지막 장점은, 이 방식은 단순한 매치메이킹<sub>matchmaking</sub> 서비스를 제외하면 서버를 운영할 필요가 없다는 점이다.
그리고 보통 이런 서비스는 콘솔 제조사에서 제공한다.

안타깝게도, 이 방식은 큰 단점<sub>major drawback</sub>도 하나 있다.
보통 이런 게임은 각 시뮬레이션 업데이트 전에 컨트롤러 입력을 읽어오도록 설계되어 있다는 점이다.
모든 원격<sub>remote</sub> 플레이어의 입력을 받아오기 전까지는 1프레임의 시뮬레이션을 수행할 수 없다.
실제 게임에서, 이는 한 기기에서 다른 기기로 패킷<sub>packet</sub>을 전송하는 데 걸리는 시간 만큼의 입력 지연<sub>input delay</sub>의 모습으로 나타난다.
한마디로, 게임이 랙 걸린다.
게임이 의도한 빠릿한 반응속도는 느릿하고 지척거리는<sub>sluggish and squishy</sub> 조작감으로 대체되어, 플레이어의 경험을 망가뜨린다.

많은 경우, 네트워크 계층에 의해 발생한 지연은 게임의 느낌을 완전히 바꿔버리고 만다.
*스트리트 파이터* 팬들이 흔히 "랙 전략<sub>lag tactics</sub>"이라 부르는, 상대가 내 기술을 볼 시점이면 반응하기엔 늦었다는 점을 이용한 플레이가 가능하다.
나를 포함해 많은 사람에게, 랙 전략은 온라인 대전 경험<sub>competitive experience</sub>에 지대한 영향을 미친다.

필자는 이 방식의 장점은 그대로 취하면서, 랙 문제를 더 잘 해결하기 위해 GGPO를 작성했다.
시뮬레이션 전체에 입력 지연을 추가하는 대신, GGPO는 각 원격 플레이어의 행동 시작 선딜<sub>start-up of each remote player's action</sub>에 연결의 지연 시간을 감춤으로써 로컬 플레이어 아바타의 즉각적인 반응을 보장한다.
이 방식이 온라인 매치에서 오프라인의 경험을 충실히 재현하는데 더 성공적이다.

# The Solution: GGPO

GGPO는 로컬 플레이어가 느끼는<sub>perceived</sub> 입력 지연을 추측 기반 실행<sub>speculative execution</sub>을 이용해 제거한다.
플레이어의 시뮬레이션 프레임을 실행하기 전에 모든 입력이 도착하기를 기다리는 대신, GGPO는 기존 행동을 토대로 원격 플레이어가 무엇을 할지를 예측한다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig2.svg" width="802px"/>
<figcaption aria-hidden="true">그림 2: GGPO의 예측 메커니즘 뜯어보기</figcaption>
</figure>

전통적인 프레임 지연<sub>frame-delay</sub> 방식을 사용했을 때 로컬 플레이어가 느끼는 랙이 위 방법으로 제거된다.
로컬 플레이어 아바타의 반응성이 오프라인 플레이와 똑같아진다<sub>just as responsive</sub>.
비록 입력이 도착할 때까지 다른 플레이어의 행동을 알 수는 없지만, GGPO의 예측 메커니즘이 게임 시뮬레이션이 대부분의 시간 동안 올바른 것을 보장한다.

GGPO가 네트워크에서 원격 입력을 받으면, 예측된 입력과 실제 입력을 비교한다.
불일치<sub>discrepancy</sub>가 발견되면, GGPO는 시뮬레이션을 최초로 틀린 프레임으로 되감고<sub>rewind back</sub>, 업데이트된 입력 스트림을 토대로 각 플레이어의 입력을 재예측한 후, 새로운 예측을 토대로 현재 프레임까지 시뮬레이션을 진행시킨다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig3.svg" width="826px"/>
<figcaption aria-hidden="true">그림 3: GGPO가 동작하는 모습. 1프레임에서의 롤백<sub>rollback</sub>에 유의할 것.</figcaption>
</figure>

# Hiding the lag

GGPO에서는, 로컬 플레이어의 행동은 항상 즉각적으로 적용되고, 언제나 올바르다.
이는 좋은 특성인데, 컨트롤러 입력에 대한 아바타의 반응성이 보통 온라인 플레이를 즐기는 데 있어서 가장 중요한 측면이기 때문이다.

더군다나, 이미 수신된 원격 플레이어 입력으로 인한 장기적인 효과도 올바르다.
예를 들어, *스트리트 파이터*에서 상대가 몇 프레임 전에 파동권<sub>fireball</sub>을 썼다면, 그 기탄<sub>fireball</sub>의 움직임은 완전히 결정론적이고 상대 플레이어의 후속 입력에 의해 전혀 영향받지 않는다.
그러므로, 로컬 시뮬레이션 상태상 플레이어가 파동권을 쐈다고 확정되는<sub>determined</sub> 그 직후부터 기탄의 움직임은 항상 올바르게 보일 것이다.
그에 따라 파동권 타이밍과 그것에 대응하는 느낌<sub>experience</sub>이 온라인이건 오프라인이건 똑같아진다.
이는 중요한데, 파동권에 대응하는 것이 *스트리트 파이터*의 주된 플레이 요소<sub>major part</sub> 중 하나이기 때문이다!

비슷하게, 적은 보통 점프 이후 그 궤적<sub>arc</sub>을 바꿀 수 없으므로, 적이 점프 후 하강할 때, 이를테면 잘 잰 승룡권<sub>well-paced Dragon Punch</sub>을 꽂아주는 등의 대응도 온·오프라인 막론하고 똑같을 것이다.
그런데 모든 게 항상 잘 작동하는 것처럼 보인다면, 지연 시간<sub>latency</sub>은 어디로 간 걸까?

지연 시간<sub>latency</sub>은 적이 행동을 개시하는 시점과 로컬 시뮬레이션이 그것을 인지하는 시점 사이에 숨겨져 있다.
그 사이 간격<sub>window</sub>의 손실된 시간은 시뮬레이션에서 건너뛰어진 셈<sub>effectively skipped</sub>이다.
예를 들어, 당신과 적이 패킷을 보내는 데 60ms가 걸리는 네트워크 상에서 *스트리트 파이터*를 하고 있다고 해 보자.
적이 기술을 사용한 시점에, 적의 시뮬레이션은 컨트롤러 입력을 즉시 처리한다.
적에게는, 기술이 바로 나오는 것으로 보인다. 로컬 입력은 시뮬레이션으로 즉시 보내지고 항상 올바르기 때문이다.
하지만, 당신 시뮬레이션 측에서는, 해당 입력을 담은 패킷이 도착하는 다음 60ms 동안 적이 기술을 사용했다는 것을 알 방법이 없다.
패킷이 도착하면, GGPO는 게임을 60ms 되감고, 적의 입력을 보정한 후, 게임 시뮬레이션을 60ms 빨리감기<sub>fast-forward</sub>해 현재 시간으로 되돌려놓는다.
그 결과, 당신의 기기에서는 적 행동 첫 60ms 동안의 애니메이션을 볼 수 없다.
당신 관점에서는 마치 적 애니메이션이 이미 60ms 재생된 시점부터 시작하는 걸로 보인다.

이는 이상적이지는 않지만, 다른 대안은 로컬 입력을 포함해 시뮬레이션 전체를 60ms 지연시키는 것뿐이다.
실제로 적용해 보면, 60ms의 애니메이션 손실이 훨씬 더 나은<sub>greatly preferable</sub> 사용자 경험을 낳는다.
이것은 부분적으로는 로컬 입력 반응성이 훨씬 좋아진 덕분이기도 하지만, 대부분의 경우 60ms가 그저 중요하지 않기 때문이기도 하다.
이를 설명하기 위해, 구체적인 사례를 들어 살펴보자.

*스트리트 파이터*의 공격 대부분은 3단계<sub>phases</sub>로 이루어진다: 선딜<sub>start-up</sub>, 공격<sub>execution</sub>, 그리고 후딜<sub>recovery</sub>이 그것이다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig4.svg" width="852px"/>
<figcaption aria-hidden="true">그림 4: 대부분의 광대역<sub>broadband</sub> 연결에서, GGPO는 스트리트 파이터 기술의 랙을 선딜 단계<sub>start-up phase</sub>에 숨길<sub>mask</sub> 수 있다.</figcaption>
</figure>

기술의 선딜<sub>start-up</sub>은 유저가 버튼을 누른 후 실제 공격 판정이 생기기까지 걸리는 시간을 말한다.
선딜<sub>start-up</sub> 단계에서도 보통 애니메이션은 있지만, 기술이 실제로 데미지를 입히지는 않는다.
진짜 중요한 일<sub>serious business</sub>은 공격 단계<sub>execution window</sub>에서 벌어진다.
공격 단계에서 적이 당신 공격의 판정 범위<sub>active region</sub>와 겹쳐 있다면, 게임 시뮬레이션이 타격<sub>hit</sub>했다고 판정<sub>register</sub>할 것이다.
타격이 발생하면 시뮬레이션은 새로운 애니메이션과 효과음을 재생하고, 적의 체력을 일정량 깎는 등의 여러 효과를 적용시킨다.
시뮬레이션 상태에 관한 한 이는 대단히 중요한 일이다.
후딜<sub>Recovery</sub>은 공격 단계 이후 다음 기술을 쓸 수 있을 때까지의 시간 간격이다.

이 수치들은 보통 프레임<sub>frame</sub> 단위로 측정되는데, 신작이 출시되면 프로 *스트리트 파이터* 플레이어들<sub>competitive players</sub>이 가장 먼저 하는 일은 전술 연구를 위해 모든 기술의 프레임 데이터를 추출<sub>mine</sub>하는 일이다.
*스트리트 파이터* 시리즈 중 가장 사랑받고 또 빠른 *슈퍼 스트리트 파이터 2 터보*의 [프레임 데이터](https://zachd.com/nki/ST/flame.html)를 보면, 대부분의 기술들이 최소한 4프레임(66ms)의 선딜<sub>start-up</sub>이 있음을 볼 수 있다.
이 사실이 굉장히 중요하다.
왜냐면 이 말은 120ms 핑<sub>ping</sub>짜리 연결에서, 거의 모든 기술의 롤백이 공격 단계<sub>execution</sub>에 들어가기 전에 해결될 수 있다는 말이고, 이는 잘못된 예측으로 인한 그래픽과 사운드 글리치<sub>visual and audio glitches</sub>가 거의 항상 선딜<sub>start-up</sub> 애니메이션에서만 발생하고, 적을 때려서 발생하는 경우는 거의 없다는 말이다.

현대의 로스앤젤레스와 뉴욕을 잇는 광대역<sub>broadband</sub> 연결은 일반적으로 120ms보다는 빠른 핑<sub>ping</sub>을 갖는다.
사실, GGPO의 테크닉이 세계를 잇는 광대역 연결에서의 지연까지도 잘 커버한다는<sub>scale well up to</sub> 실사례 증거<sub>anecdotal evidence</sub>도 있다.
**그림 5**를 보면 [GGPO.net](https://www.ggpo.net/) 운영 당시 전형적이었던 저녁 시간대 테스트 서버 활동량 시각화 이미지<sub>snapshot</sub>를 볼 수 있다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig5.svg" width="852px"/>
<figcaption aria-hidden="true">그림 5: <a href="https://www.ggpo.net/">GGPO.net</a> 테스트 서버 활동량 시각화 이미지. 보다시피, 대륙간 플레이도 문제가 되지 않았었다.</figcaption>
</figure>

# Integrating GGPO

GGPO는 개발자가 네트워크 세부 사항으로부터 최대한 격리될 수 있도록 작성되었다.
**그림 6**은 아케이드 게임에서의 단순화된 게임 루프를 표현한 것이다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig6.svg" width="306px"/>
<figcaption aria-hidden="true">그림 6: 전형적인 아케이드 게임 루프.</figcaption>
</figure>

게임 루프는 컨트롤러로부터 샘플링<sub>samples</sub>을 수행해 다음 게임 시뮬레이션 상태에 사용할 입력을 생성한다.
그 입력은 시뮬레이션 엔진으로 전달돼 게임을 다음 프레임으로 업데이트하고 각 시뮬레이션 단계의 결과를 렌더링<sub>render</sub>한다.
타이밍이 중요한<sub>timing-sensitive</sub> 많은 아케이드 게임은 게임 루프를 고정된 속도<sub>fixed rate</sub>로 (예: 60hz) 돌리는데<sub>drive their game-loop</sub>, 그런 경우 다음 프레임을 위해 다음 컨트롤러 상태를 샘플링하기 전, 대기<sub>idle</sub>하는 것이 필요할 수 있다.
이 게임 루프에는 다양한 변형<sub>many variations</sub>이 존재한다.
예를 들어, 어떤 게임은 컨트롤러 샘플링과 게임 업데이트를 30hz로 수행하지만, 최근 2개의 게임 상태를 보간<sub>interpolating</sub>하여 120hz로 렌더링될 수 있다.
GGPO는 이런 상황에서도 마찬가지로 적용할 수 있다.

<figure style=";padding:1em;border-radius:1em;display:table;text-align:center;margin:auto">
<img src="/assets/img/posts/2025-04-09-ggpo-gdmag-article/fig7.svg" width="639px"/>
<figcaption aria-hidden="true">그림 7: GGPO를 쓰는 아케이드 게임 루프. GGPO와 통합하기 위해 추가/수정된 단계는 회색 블록으로 표현하였다.</figcaption>
</figure>

**그림 7**은 GGPO를 통합<sub>incorporate</sub>하기 위해 변형된 게임 루프를 보여준다.
컨트롤러에서 샘플링을 수행한 다음, 개발자는 그 입력을 `ggpo_synchronize_inputs` 함수를 통해 GGPO에 전달해야 한다.
`ggpo_synchronize_inputs`은 모든 로컬 입력을 원격 플레이어에게 전송할 것이다.
또한 이 함수는 모든 원격 플레이어의 예측된 입력과 실제 입력을 병합하여 입력 스트림에 기록한다.
결과물은 각 플레이어의 입력으로 이루어진 튜플<sub>tuple</sub>로, 이를 로컬 게임 엔진에 전달할 수 있다.

개발자는 게임 엔진의 대부분<sub>bulk</sub>을 수정하지 않은 채로 놔둘 수 있다.
이후에 언급할 잠재적 주의사항<sub>potential caveats</sub> 몇개만 빼면 말이다.
게임 루프의 처음으로 돌아가기 전에 대기<sub>idling</sub>하는 대신, 개발자는 `ggpo_advance_frame` 함수를 호출해야 한다.
이 함수는 모든 원격 플레이어로부터 받은 입력을 예측한 값과 비교한다.
불일치<sub>discrepancy</sub>가 발견되면, `ggpo_advance_frame`은 마지막으로 올바르게 예측한 프레임을 불러온 후, 올바르게 고친<sub>corrected</sub> 입력을 가지고 게임의 상태 업데이트 함수를 반복 호출하여 게임 시뮬레이션을 현재 프레임까지 빨리감기<sub>fast-forward</sub>한다.
로드<sub>load</sub>, 저장<sub>save</sub>, 실행<sub>execute</sub> 함수들은 GGPO 초기화 시 개발자가 제공한다.

# Synchronizing the clock and inputs

GGPO는 세션 내 플레이어 간 입력을 동기화하는데<sub>synchronizing inputs</sub> 단순하고 효율적인 프로토콜을 사용한다.

```cpp
struct packet {
    struct {
        uint16  magic;
        uint8   type;
    } hdr;
    union {
        struct {
            uint32  start_frame;
            uint32  ack_frame;
            uint16  num_bits;
            uint8   bits[MAX_COMPRESSED_BITS];
        } input;
        struct {
            int8    frame_advantage;
            uint32  ping;
        } quality_report;
        struct {
            uint32  pong;
        } quality_reply;
    } u;
};
```
> 코드 1: GGPO가 사용하는 UDP 패킷 페이로드<sub>payload</sub>를 단순화한 버전<sub>simplified description</sub>.\
> (역자 주: 실제 버전은 [GGPO 소스 코드의 `udp_msg.h`](https://github.com/pond3r/ggpo/blob/master/src/lib/ggpo/network/udp_msg.h)를 참고할 것.)

각 패킷의 헤더에는 16비트 세션별 식별자<sub>session-specific identifier</sub>가 들어있고, 그 뒤에 전달할 데이터의 타입<sub>type of the payload</sub>이 붙는다.
3가지의 주요 패킷 타입은 `quality_report`, `quality_reply`와, `input`이다.

매번 `ggpo_synchronize_inputs`를 호출할 때마다 입력 페이로드 1개가 전송된다.
가상 버튼을 껐다 켰다<sub>toggle</sub>하는 상태 기계<sub>state machine</sub>를 이용해 입력이 압축된다.
상태 기계는 0프레임에는 비어있는 입력 벡터<sub>empty input vector</sub>로 시작한다.
각 프레임마다 이전 프레임과 달라진 버튼 입력을 비트 배열<sub>bits array</sub> 안에 인코딩하는데, 아래의 꽤나 원시적인 인코딩 방식<sub>fairly trivial coding system</sub>을 사용한다:
```
1 + <N의 5비트> : 이전 입력의 버튼 N을 toggle (반전)
0 : 입력 끝
```
스트림 내의 거의 모든 입력이 한개의 비트 (0) 로 압축된다.
초당 10회 넘는 입력이 있는 극도로 까딱거리는<sub>twitchy</sub> 게임에서조차 초당 300비트 정도로 압축된다.
이는 충분히 작기 때문에 이전에 승인된<sub>acknowledged</sub> 프레임의 모든 입력<sub>all inputs</sub>을 모든 패킷<sub>every packet</sub>에 인코딩한다.
이렇게 하면 유실<sub>dropped</sub>되거나 순서가 뒤섞인<sub>out-of-order</sub> 패킷에 대한 처리가 아주 간단해진다.
GGPO는 원격 플레이어로부터 받은 마지막 x번째까지의 입력 이력<sub>history of the last x inputs</sub>을 보관하고 있는데, 여기서 x는 예측 배리어<sub>prediction barrier</sub>보다 더 큰 범위<sub>window</sub>이다.
원격 플레이어로부터 입력을 수신할 때마다, 이 범위<sub>window</sub>로부터 `start_frame`의 입력을 불러온 후 비트 상태 기계를 실행해 모든 후속 입력을 복원한다.
우리 범위<sub>window</sub>를 넘어서는 입력들을 생성하는 경우, 그 입력들은 예측 시스템<sub>prediction subsystem</sub>으로 보내져 검증<sub>verification</sub>을 거친 후 범위<sub>window</sub>에 저장된다.
끝으로, 마지막으로 생성된 입력의 수를 기억함으로써, 그것을 다음 입력 패킷의 `ack_frame` 필드에 포함해 상대에게 보낼 수 있도록 한다.

GGPO는 주기적으로 `quality_report` 패킷을 보내 연결의 공정성<sub>fairness</sub>을 측정한다.
매 품질 측정<sub>quality-report</sub> 패킷은 현재 기기에서 생성한 타임스탬프<sub>timestamp</sub>와, 현재 피어<sub>peer</sub> 자신의 로컬 "프레임 어드밴티지<sub>frame advantage</sub>" 측정값을 포함한다.
상대방<sub>peer</sub>이 `quality_report` 메시지를 받으면, 받은 즉시 report 내에 있는 `ping` 값을 복사해 `quality_reply` 응답을 보낸다.
본래 발신했던 피어<sub>originating peer</sub>는 현재 시간에서 마지막 `quality_reply` 패킷의 `pong` 값을 빼서 왕복 시간<sub>round-trip time</sub>을 측정한다.

GGPO에서 "프레임 어드밴티지"란 로컬 플레이어가 시간 왜곡<sub>wall clock skew</sub>으로 인해 몇 프레임의 어드밴티지를 갖는지를 나타낸다.
이는 다음 공식으로 계산된다:
```
frame_advantage = (last_remote_frame + (ping * frame_frequency / 2)) - last_local_frame
프레임_어드밴티지 = (마지막_원격_프레임 + (핑 * 프레임_빈도 / 2)) - 마지막_로컬_프레임
```
다시 말해, 상대가 현재 렌더링하는 프레임을 우리가 마지막으로 받은 패킷에 왕복 시간<sub>round-trip time</sub>의 절반을 더하고, 거기서 우리가 렌더링하는 프레임을 빼서 추정하는 것이다.
그 결과는 상대방이 우리보다 앞서 있는 프레임의 수다.
예를 들어, 우리가 `frame_advantage`를 2라고 계산했다 해보자.
이 말은 두 게임으로부터 동일한 거리만큼 떨어진<sub>equidistant</sub> 제3자<sub>neutral observer</sub>가 보기에 내 게임이 20프레임을 렌더링중이라면 같은 순간 상대방은 22프레임을 렌더링중이라는 말이다.
GGPO는 이것을 "유리한 것<sub>advantage</sub>"으로 간주하는데 그 이유는 우리 시뮬레이션은 상대에 비해 2프레임 덜 롤백해도 되기 때문이다.
그 말은 상대 게임 세계가 이쪽보다 평균적으로 더 많이 만료됐다는<sub>out-of-date</sub> 의미이다.
프레임 어드밴티지를 계산함에 있어서 오차를 유발할 가능성이 있는 것들이 많기는 하다.
이를테면 비대칭적 패킷 전송 시간<sub>asymmetric packet transmit times</sub>이나, 간헐적인 연결 문제<sub>intermittent connection issues</sub> 등이 있을 수 있다.
하지만 실제로 써보면 위 공식으로도 잘 작동하는 것으로 보인다.

GGPO 세션에서의 각 피어는 언제나 자신의 로컬 프레임 어드밴티지를 알고 있으며 상대방이 계산한 프레임 어드밴티지도 주기적으로 수신한다.
GGPO는 로컬과 원격의 프레임 어드밴티지를 1프레임 내의 허용 오차<sub>tolerance</sub>로 조정하여<sub>agree to</sub> 게임을 가능한 한 "공정하게<sub>fair</sub>" 유지하려고 한다.
예를 들어, GGPO 종단<sub>endpoint</sub>이 로컬 프레임 어드밴티지를 계속 5라고 계산하는데, 상대방의 로컬 프레임 어드밴티지는 계속 3이라면, GGPO는 로컬 종단의 실행을 1프레임 늦춤으로써 이 차이<sub>disparity</sub>를 평준화<sub>equalize</sub>하려고 할 것이다.
그 결과 로컬 단말<sub>local end</sub>은 1프레임을 잃고 원격 단말<sub>remote end</sub>은 1프레임을 얻어, 양측의 프레임 어드밴티지가 4프레임으로 평준화된다.
로컬 종단을 어떻게 느리게 만드는지는 게임 개발자가 제공하는 콜백<sub>callback</sub> 구현에 따라 달라진다.

# Separating Game State from Rendering

이제 왜 GGPO가 게임 로직과 렌더링을 분리할 것을 요구하는지 명확해졌으리라 생각한다.
각 장면<sub>video frame</sub>의 렌더링마다, 잘못 예측한 원격 입력 탓에 현재 상태를 재평가하느라 게임 상태를 여러번 업데이트해야 할 수도 있기 때문이다.
만일 게임 엔진이 게임 상태 재평가에 더해 그 중간 장면까지 렌더링해야 한다면, GGPO의 롤백 테크닉은 감히 엄두도 못 낼 정도로 오래 걸리는<sub>prohibitively expensive</sub> 작업일 것이다.
예를 들어, 비디오 렌더러<sub>video renderer</sub>는 온갖 역운동학<sub>inverse kinematics</sub>, 소프트바디 시뮬레이션<sub>soft body simulation</sub>, 그리고 시각효과<sub>visual effect</sub>를 위한 기타 비싼<sub>expensive</sub> 연산을 수행할 지도 모르는데, GGPO를 쓰려면 게임 시뮬레이션시 이런 연산이 필요 없도록 처리해야 할 것이다.

게임 렌더러는 또한 잘못 예측된 이전 프레임에서 자원<sub>assets</sub>을 제거할 수 있어야 한다.
예를 들어, 기탄<sub>fireball</sub>을 발사하며 효과음을 낸 프레임이 상대 입력을 받아보니 올바른 결과가 아닐 수 있다.
이 상황은 그래픽<sub>video</sub>에 대해서는 처리하기 쉽지만, 사운드<sub>audio</sub>에 대해서는 까다로울 수 있다.
오디오를 처리하기 위해 사용된 방법 중 2가지가 성공적이었다.

첫번째는 오디오를 시뮬레이션 상태로 취급하는 방법이다.
게임 상태가 초당 30회 업데이트된다면, 각 채널<sub>channel</sub>마다 33.3ms의 사운드<sub>audio</sub> 샘플을 포함해 게임 상태와 같이 재생<sub>render</sub>하면 된다.
효과음<sub>audio</sub>을 재생할 때, 전체를 재생하는 게 아니라 33.3ms만을 재생한다.
이 방법은 비교적 구현하기 쉽지만, 긴 롤백 도중 터지는 소리<sub>audio popping</sub>를 유발할 수 있다.

두번째 방법은 게임 상태 안에 이전에 재생한 모든 효과음과 재생한 시점을 기억해 놓는 방법이다.
저장 간격<sub>window</sub>은 GGPO에 설정된 최대 버퍼 간격<sub>maximum buffer window</sub>만큼만 크면 된다.
효과음<sub>audio</sub>을 재생할 때, 게임 상태 내의 효과음 리스트를 현재 오디오 장치에서 대기 중인<sub>queued</sub> 효과음들과 비교한다.
게임 상태에는 있으나 대기 중<sub>queued</sub>이지 않은 효과음들은 오디오 장치로 전송돼야 한다.
효과음이 롤백 도중에 일어났다면 몇 밀리초 이후 시점부터 재생해야 할 수도 있다.
(이런 경우 볼륨을 0%부터 시작해 서서히 100%로 높이는 것이 터지는 소리<sub>popping effect</sub>를 완화<sub>alleviate</sub>하는 데 도움이 될 수 있다.)
오디오 대기열<sub>audio queue</sub>에는 있으나, 시뮬레이션 상태에는 없는 효과음들의 경우는, 롤백의 결과로 취소된<sub>revoked</sub> 것들이다.
애초에 재생한 것이 잘못이므로 멈춰야 하는데, 갑작스러운 변화<sub>popping</sub>를 막기 위해 볼륨을 몇 프레임에 걸쳐 0%까지 감쇠<sub>attenuating</sub>시켜야 할 수도 있다.

# Getting It All Right the First Time

GGPO SDK는 아주 사용하기 쉽지만, 매우 복잡한 게임이나, GGPO를 고려하지 않고 설계·작성된 게임은, 여러 장벽<sub>stumbling blocks</sub>에 봉착할 수 있다.
첫번째 문제는 금방 다뤘다: 그래픽과 사운드 렌더링<sub>video and audio renderers</sub>이 게임 시뮬레이션 상태와는 분리되어야<sub>divorced</sub> 한다는 점이다.
두번째는, 각 프레임마다 시뮬레이션 업데이트 단계를 여러번 수행하기에 충분한 CPU 시간<sub>budget</sub>이 있음을 게임 개발자가 보장해야 한다는 것이다.
이상적으로는, 게임 시뮬레이션이 프레임당 4~5번 돌 수 있도록 충분히 빠르면 좋은데, 이러면 대략 80ms의 지연 시간을 감출 수 있다.
마지막으로, 게임 시뮬레이션이 완전히 결정론적이고, 오로지 플레이어 입력에 의해서만 결정되어야 한다는 점이다.

로컬 멀티플레이어의 경우 모든 입력이 100% 올바르고 로컬 컨트롤러에서 즉각 들어오므로 롤백을 할 이유가 전혀 없다.
그 결과, 이 3가지 기능은 거의 항상 게임의 네트워킹 부분을 테스트할 때나 구현할 때만 테스트된다.
이는 보통 게임에 투자되는 전체 테스트 시간 대비 아주 작은 시간<sub>small fraction</sub>에 불과하다.

GGPO로 게임을 포팅<sub>port</sub>하려는 개발자들을 돕기 위해, GGPO에는 "동기화 테스트<sub>sync test</sub>"라 불리는 특수한 디버깅 모드가 포함되어 있다.
GGPO를 동기화 테스트<sub>sync test</sub> 모드로 초기화하면, SDK는 모든 게임 프레임을 원격 입력이 잘못 예측된 프레임으로 간주한다. 이는 로컬 플레이를 하더라도 마찬가지로 적용된다.
`ggpo_advance_frame`을 호출할 때마다 항상 프레임이 불러와지고, 설정된 수만큼 다시 실행되며, 게임 상태도 저장될 것이다.
이를 이용하면 가장 어려운 부분의 성능<sub>performance</sub>과 정확성<sub>correctness</sub>이 네트워크나 원격 세션 없이 구현·디버깅될 수 있으므로, GGPO를 통합하는 개발 및 테스트 시간이 현격히 줄어들 것이다.

랙이라는 몬스터를 잡는<sub>slaying the lag monster</sub> 완벽한 해결책<sub>silver bullet</sub>은 없지만, GGPO는 게임 개발자들에게 그것과 씨름해 항복을 받아낼<sub>wrestling it into submission</sub> 또 다른 도구를 제공한다.
GGPO와 함께라면, 개발자들은 게임 루프나 엔진 설계를 복잡하게 만들지 않고도 게임의 입력 지연을 한쪽으로 패킷을 전송하는 시간<sub>one-way packet transmission time</sub> 미만으로 줄일 수 있다.
아케이드 게임 문화에 깊은 뿌리를 둔 사람의 입장으로서 말하건데, 세상의 개발자들이 GGPO나 이 글에서 설명한 테크닉을 가지고 랙 없는 온라인 게임 타이틀을 만들어가길 희망한다.

-----

*Tony Cannon은 1995년 Stanford University에서 컴퓨터 과학에 대한 학사 학위를 취득하며 졸업했다.*
*그는 VXtreme Inc., Microsoft에서 일했었고, 현재는 VMware와 Radiant Entertainment에서 일하고 있다.*
*Tony는 격투 게임에 대한 열정이 있고 Evolution Tournament Series의 공동 설립자이자 토너먼트 디렉터이기도 하다.*

-----

# 부록

## Testing GGPO With Emulators

(WIP)

## GGPO's prediction algorithm

(WIP)


*마지막 수정 : {{ page.last_modified_at }}*
