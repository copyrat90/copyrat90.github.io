---
layout: post
title: Multi Plaza 개발일지 (01)
tags: [Devlog]
author: copyrat90
last_modified_at: 2025-06-13T19:23:00+09:00
---

올해 초부터 지금까지 작업 끝에 [Steam P2P Datagram Relay](https://partner.steamgames.com/doc/features/multiplayer/steamdatagramrelay)를 경유한 채팅 구현까지 성공했다.

클라이언트 엔진은 Godot Engine을 사용 중이지만, 네트워킹 관련 코드는 [ISteamNetworkingSockets](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets) API를 기반으로 바닥부터 만드느라 시간이 꽤 걸렸다.

눈에 보이는 결과가 궁금할테니 일단 더미 테스트 영상부터 보여주고, 구현 내역 세부사항은 아래에 적도록 하겠다.


# 채팅 테스트 영상

[![](https://img.youtube.com/vi/m29XLcMtsJs/hqdefault.jpg)](https://www.youtube.com/watch?v=m29XLcMtsJs)

그리고, 위 유튜브 동영상에는 안 나오지만 아래 이미지를 보면 2가지 추가기능이 있다.

![](/assets/img/posts/2025-06-13-multi-plaza-devlog-01/chat-filter-test.png)

1. 지나치게 빠르게 채팅하면, 30초간 벙어리
    * 기준: 10초 window에 5번 초과로 채팅
1. [Steam Chat Filtering](https://store.steampowered.com/news/app/593110/view/2855803154584367415) 이용한 욕설/비방 검열


## 대역폭 계산

* 위 더미 테스트 영상에서, 각 클라이언트는 약 2초에 1개의 메시지를 보내고, 한번 채팅에 헤더 포함 평균 155 bytes의 메시지가 전송됨.
    * 따라서, 각 클라이언트가 초당 약 77.5 bytes 정도 보낸다고 보면 됨.
* 로컬 클라이언트를 제외한 128명의 더미가, 로컬 클라이언트를 포함한 128명의 사용자에게 메시지를 보냄
    * 따라서, 서버측에서는 초당 128 × 128 × 77.5 = 1,269,760 bytes 로, 1.21 MiB/s 정도의 broadcast가 필요함.

영상을 보면 초기에 안정적인 네트워크 상황일 때는 1.3 ~ 1.4 MiB/s 정도로 예측한 대역폭을 조금 웃돈다.\
하지만 Steam 소켓의 [SNP 헤더](https://github.com/ValveSoftware/GameNetworkingSockets/blob/master/src/steamnetworkingsockets/clientlib/SNP_WIRE_FORMAT.md)와 암호화 오버헤드 등을 고려하면 그럴 수 있다 싶다.

중간에 네트워크 불안정으로 핑이 튀면서 전송량이 늘어나는 데,\
Steam 릴레이 서버 측 문제인지, 내 방이 무선 공유기에서 멀리 있는 탓인지 모르겠다.


# 구현 내역

연초부터 구현했던 내역을 돌아보자.


## 1. [GnsSharp](https://github.com/nalchi-net/GnsSharp) 라이브러리 구현

> 기존 C# Steamworks 바인딩인 [Steamworks.NET](https://github.com/rlabrecque/Steamworks.NET)과 [Facepunch.Steamworks](https://github.com/Facepunch/Facepunch.Steamworks) 둘 다 아쉬워서,\
> 바닥부터 다시 직접 만든 C# Steamworks 바인딩.

* Commit history: 2025.3.11 - 2025.5.3

1. 둘 다 오픈소스 버전 [GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)를 지원하지 않는 게 아쉬움.
    * 내 라이브러리는 오픈소스 버전 GNS와, Steamworks SDK 버전을 모두 지원함.
        * 그런데 현 프로젝트에는 Steamworks SDK 버전을 쓰고 있어서 당장은 무의미한 장점.
1. 둘 다 connection 별로 서로 다른 Connection Status Changed handler를 지정하는 것이 불가능함.
    * 내 라이브러리는 connection 별로 서로 다른 handler 설정하도록 지원.
        * 이게 있어야 서버 측과 클라이언트 측 코드를 깔끔하게 분리할 수 있음.
        * GNS 쪽은 이게 자동으로 되는데 Steamworks SDK 쪽은 이게 기본적으로 안 돼서, 비슷하게 직접 구현함.
            * 사용자가 connection에 `Marshal.GetFunctionPointerForDelegate()`로 등록한 함수 포인터를,\
              [라이브러리 측에서 `Marshal.GetDelegateForFunctionPointer()`로 가져와서 호출하는 식](https://github.com/nalchi-net/GnsSharp/blob/add303d893a5e912e81c7bef5eebe6401501ca63/GnsSharp/ISteamNetworkingSockets.cs#L1918). 너무 hack인가?
1. 둘 다 오래된 라이브러리이다 보니 P/Invoke 인자로 managed 배열을 사용하는 게 아쉬움.
    * 내 라이브러리는 `Span<T>`, `ReadOnlySpan<T>`를 사용하므로, `stackalloc` 배열을 인자로 넘겨 동적 할당을 피할 수 있음.
1. Facepunch.Steamworks는 API가 추상화되어 있어 [공식 Steamworks API Reference](https://partner.steamgames.com/doc/api)를 이용하기 애매하고,\
    Steamworks.NET은 API가 1:1로 대응되나 [Steam Callback과 CallResult](https://partner.steamgames.com/doc/sdk/api#callbacks) 처리가 C# 답지 못해서 아쉬움.
    * 내 라이브러리는 API가 1:1로 대응되고, Steam Callback는 C# event로, Steam CallResult는 [커스텀 awaitable 구현](/2025/03/23/csharp-custom-awaitable)으로 처리함.
        * 내가 만든 커스텀 awaitable은 Facepunch.Steamworks와 달리:
            * lock을 걸고 Steam API call을 호출하여, [비동기 요청이 Dictionary에 등록되기도 전에 반환되어 유실되는 race condition을 방지](/2025/03/23/csharp-custom-awaitable#사족-steamapicall_t로-calltaskt-찾기).
            * [현재 SynchronizationContext를 capture해 `Post()`](https://github.com/nalchi-net/GnsSharp/blob/add303d893a5e912e81c7bef5eebe6401501ca63/GnsSharp/CallTask.cs#L59-L60)하므로, 동기화가 편리함.
                * 그게 필요없는 상황도 있을 테니, 추가로 `ConfigureAwait(false)`도 제공.
1. Facepunch.Steamworks는 poll group 단위로 메시지를 수신하는 게 불가능.
    * 내 라이브러리는 원본 Steamworks SDK와 1:1 대응이므로 당연히 가능함.\
      아마 Steamworks.NET도 가능할 텐데, 확인해보지는 않았다.


## 2. [nalchi](https://github.com/nalchi-net/nalchi) / [NalchiSharp](https://github.com/nalchi-net/NalchiSharp) 라이브러리 구현

> GameNetworkingSockets (Steamworks 소켓 부분) 의 전송 기능을 보강하는 C++23 라이브러리 및 그것의 C# 바인딩.

* Commit history: 2025.3.2 - 2025.6.11

* reference counting 되는 [`shared_payload`](https://nalchi-net.github.io/nalchi/structnalchi_1_1shared__payload.html)를 이용한 [`socket_extensions::multicast()`](https://nalchi-net.github.io/nalchi/classnalchi_1_1socket__extensions.html) 지원.
    * 여러 클라이언트들에게 동일한 payload를 멀티캐스팅 할 시에, payload를 일일히 할당 -> 복사하지 않고 다 같이 공유해 사용하기 위함.
    * payload 포인터 [4바이트 앞 주소에 `std::atomic<int>`를 숨겨놓는 방식으로 구현](/2025/02/24/gns-analysis-2#shared-payload)
        * 이 ref count를 [`SteamNetworkingMessage_t::m_pfnFreeData`에 등록](https://github.com/nalchi-net/nalchi/blob/168d1c5b1e93e0102a7e8c70bcb05b89c0504fe2/src/shared_payload.cpp#L154)한 [커스텀 payload 해제 함수에서 사용해 해제 처리](https://github.com/nalchi-net/nalchi/blob/168d1c5b1e93e0102a7e8c70bcb05b89c0504fe2/src/shared_payload.cpp#L171-L178).
        * 사용자 측 에러로 전송하지 못했을 경우를 대비한 [`shared_payload::force_deallocate()`](https://nalchi-net.github.io/nalchi/structnalchi_1_1shared__payload.html#ae74e9f8d003554126f7f26e798c3e598) 함수도 제공.
* 비트 단위 직렬화 스트림 구현체 [`bit_stream_writer`](https://nalchi-net.github.io/nalchi/classnalchi_1_1bit__stream__writer.html)와 [`bit_stream_reader`](https://nalchi-net.github.io/nalchi/classnalchi_1_1bit__stream__reader.html) 제공.
    * 공간 효율적인 직렬화를 통해 대역폭을 아끼기 위함.
        * 예시: `enum` 값이 8종류 뿐이라면, [`min = 0`, `max = 7`로 지정하면](https://nalchi-net.github.io/nalchi/classnalchi_1_1bit__stream__writer.html#a858b5ad5fa0974786709cbdbad85dc6e) 단 3비트만으로 직렬화가 됨.
    * Gaffer on Games 블로그 글을 참고해 구현하였음.
        * <https://gafferongames.com/post/reading_and_writing_packets/>
        * <https://gafferongames.com/post/serialization_strategies/>
        * 위 글대로 구현하면, 버퍼는 반드시 4바이트 단위로 읽기/쓰기가 가능해야 하므로, alignment를 고려해 `span<std::uint32_t>` 버퍼만 사용 가능하도록 제한.
        * 또한, `unicast()`와 `multicast()`로 `shared_payload`를 전송 시에, payload가 이 `bit_stream_writer`로 데이터를 썼으면,\
          받는 측에서도 4바이트 단위로 읽어야 하므로, `bit_stream_reader`가 범위를 벗어난 읽기를 하는 것을 막기 위해 4의 배수의 byte 크기로 전송함.
            * 이를 위해 `shared_payload`에는 `std::atomic<int>` ref count 말고도, `bit_stream_writer`를 사용했는지 체크하는 flag가 추가로 숨겨져 있음.
    * 추가로, 몇 비트 필요한지 크기를 재기 위한 [`bit_stream_measurer`](https://nalchi-net.github.io/nalchi/classnalchi_1_1bit__stream__measurer.html)도 제공.
* Doxygen 이용한 [API 문서 제공](https://nalchi-net.github.io/nalchi/).

## 3. [Backdash.Gns](https://github.com/nalchi-net/Backdash.Gns) extension 라이브러리 구현

> C# 오픈소스 롤백 넷코드 구현체인 [Backdash](https://github.com/lucasteles/Backdash) 라이브러리를,\
> 내 [GnsSharp](https://github.com/nalchi-net/GnsSharp) Steamworks 바인딩과 같이 쓸 수 있도록, 하부 네트워킹 레이어를 교체하는 extension

* 주요 작업일시: 2025.4.11 - 2025.4.30

* Backdash는 [GGPO](https://github.com/pond3r/ggpo)의 C# 버전 port로 시작했기 때문인지, Connection 개념이 따로 없는 UDP 기반 프로토콜을 사용함.
    * Connection 개념이 없다 => 매번 전송할 주소를 지정하도록 되어 있음. `ISteamNetworkingSockets`를 사용하기엔 다소 부적합.
    * 그래서 좀 더 "UDP 스러운" [`ISteamNetworkingMessages` 인터페이스](https://partner.steamgames.com/doc/api/ISteamNetworkingMessages)를 대신 사용함.
* [Backdash 예제 게임도 fork](https://github.com/nalchi-net/BackdashGnsGodotSample)해서 [Steam Lobby 시스템](https://partner.steamgames.com/doc/features/multiplayer/matchmaking)을 이용하도록 만듦.


## 4. 위 라이브러리들 기반으로 서버와 클라이언트 구현

* 주요 작업일시: 2025.4.21-2025.6.13

* 서버-클라이언트 부분은 별도의 C# 라이브러리로 분리.
    * Godot 프로젝트는 거기서 필요한 API를 사용하는 식.
    * 분리가 되었으므로, 더미 클라이언트는 Godot Engine을 안 쓰고 C# Console App으로 구현할 수 있었다.
* 서버와 클라이언트의 루프 처리 방법이 다름.
    * 서버는 시작 시 10 tps로 도는 game loop용 `Task`를 스스로 생성해 돈다.
        * 서버 라이브러리와 서버 컨텐츠 단을 별도 프로젝트로 분리하지 않았음.\
          기획상 재사용의 여지가 별로 안 보였으므로, 하나로 합쳐서 단순하게 하기로 함.
    * 클라이언트는 사용자 측에서 `WorldClient.PreTick()`과 `WorldClient.PostTick()`을 직접 호출해줘야 함.
        * `PreTick()`에서 메시지 수신이, `PostTick()`에서 Steam socket 쪽에 보내둔 메시지의 실제 송신(flush)이 일어난다.
            * 수동 flush 단계가 따로 있으므로, 아예 Nagle time은 `int.MaxValue`로 설정해놨다.
                * 그런데 Steamworks 로그 찍어보면 20 ms 로 설정된 것 같다. 저게 상한인 듯.
* RPC 계층 구현
    * 사실 그냥 위 nalchi 라이브러리의 bit stream을 이용한 수동 직렬화-역직렬화를 하고 있다.
        * 직렬화 API 초기화 시 header & footer 삽입
            * 자세한 프로토콜은 비밀이지만, 두개 합쳐서 26 bits(무압축) 또는 52 bits(압축)로 8 bytes가 채 안 된다. 그런데 늘어날 여지가 있음.
        * [`BrotliStream`](https://learn.microsoft.com/en-us/dotnet/api/system.io.compression.brotlistream?view=net-9.0) (`CompressionLevel.Fastest`) 이용한 압축 지원.
            * 그런데 막상 써 보니 압축률이 너무 안 좋아서 거의 대부분의 자료를 무압축으로 전송하고 있음.\
              사용자 리스트 직렬화하는데 800 bytes 이하면 압축시 크기가 오히려 불어나고, 2200 bytes 넘어도 압축률 겨우 10%...\
              이미 비트 단위로 직렬화하고 있어서 압축이 더 될 구석이 없는건가?
        * 코드 자동 생성같은 편의 기능은 없음.\
          같은 메시지 type도 중간 데이터에 따라 후속 데이터가 가변으로 달라지는 경우가 있어서, codegen으로 처리하기가 애매한 측면이 있음.
* 기타 스팀 관련 기능
    * 스팀 로비 자동 열기 / 닫기


### 설계도

클래스를 기반으로 한 커뮤니케이션 다이어그램 형태에 가깝다.\
실제 구현은 아래 설계와 살짝 달라진 부분도 있긴 할 듯.

#### `WorldServer`

![](/assets/img/posts/2025-06-13-multi-plaza-devlog-01/WorldServer.svg)

#### `WorldClient`

![](/assets/img/posts/2025-06-13-multi-plaza-devlog-01/WorldClient.svg)


# 여담

이렇게 많은 작업을 했는데 만든 게 겨우 채팅이라니... 혼자 만드려니 해야 할 작업이 정말 정말 많은 것 같다.\
그래도 이제 어느 정도 기반 작업은 마무리됐으니, 컨텐츠 작업에 들어갈 수 있다는 게 다행이다.


*마지막 수정 : {{ page.last_modified_at }}*
