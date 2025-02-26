---
layout: post
title: GameNetworkingSockets 분석 (2) - 메시지 동적 할당
tags: [C#]
author: copyrat90
last_modified_at: 2025-02-26T14:09:00+09:00
---

[ValveSoftware/GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets) 분석 제 2편.

한 줄 요약: 기본적으로 메시지는 동적 할당되고, pooling 하려면 따로 함수 포인터를 등록해야 한다.

# 메시지 기본 할당 전략

메시지를 API 한번 호출로 한꺼번에 전송하고 싶으면, [`ISteamNetworkingSockets::SendMessages()`](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#SendMessages)를 쓸 수 있다.

그런데, 링크한 API 주석을 보면, 우선 [`ISteamNetworkingUtils::AllocateMessage()`](https://partner.steamgames.com/doc/api/ISteamNetworkingUtils#AllocateMessage)로 메시지를 할당해서 쓰라고 한다.\
(괄호 안엔 아예 직접 할당하지 말라고까지 적혀 있다.)\
그렇다면, 메시지 객체가 pooling이 되고 있는 걸까?

```cpp
SteamNetworkingMessage_t *CSteamNetworkingUtils::AllocateMessage( int cbAllocateBuffer ) {
    return CSteamNetworkingMessage::New( cbAllocateBuffer );
}
```

`CSteamNetworkingMessage::New(size)`는 길이가 좀 되므로 나눠서 분석해보자.

```cpp
CSteamNetworkingMessage *CSteamNetworkingMessage::New( uint32 cbSize ) {
    // FIXME Should avoid this dynamic memory call with some sort of pooling
    CSteamNetworkingMessage *pMsg = new CSteamNetworkingMessage;
```
일단 `CSteamNetworkingMessage` 자체는 `new`로 동적 할당...\
주석에도 나중에 pooling을 하는 코드로 고쳐야한다고 써놨다.

```cpp
    // Allocate buffer if requested
    if ( cbSize ) {
        pMsg->m_pData = malloc( cbSize );
        if ( pMsg->m_pData == nullptr ) {
            delete pMsg;
            SpewError( "Failed to allocate %d-byte message buffer", cbSize );
            return nullptr;
        }
        pMsg->m_cbSize = cbSize;
        pMsg->m_pfnFreeData = CSteamNetworkingMessage::DefaultFreeData;
    } else {
        pMsg->m_cbSize = 0;
        pMsg->m_pData = nullptr;
        pMsg->m_pfnFreeData = nullptr;
    }
    ...
```
`cbSize`가 `0`이 아니라면, `malloc(cbSize)`로 `m_pData`에 내부 payload를 저장할 공간을 동적 할당한다.\
그리고, 해제 시 호출될 함수 포인터 `m_pfnFreeData`에 `CSteamNetworkingMessage::DefaultFreeData`를 세팅하고 있다.

```cpp
void CSteamNetworkingMessage::DefaultFreeData( SteamNetworkingMessage_t *pMsg ) {
    free( pMsg->m_pData );
}
```
`DefaultFreeData()`는 그냥 `free()`다. 글자 그대로 해제만을 전담.

반대로, 매개변수로 받은 `cbSize`가 `0`이었으면, 공간을 할당하지 않고, `m_pfnFreeData`가 `nullptr`로 세팅된다.\
이걸 이용해 pooling 하겠다면, `m_pfnFreeData`에 payload 공간을 반환하는 함수를 넣으면 될 것이다.\
(당연히 `m_pData`와 `m_cbSize`도 직접 설정해야겠고.)

```cpp
    // Clear identity
    pMsg->m_conn = k_HSteamNetConnection_Invalid;
    pMsg->m_identityPeer.m_eType = k_ESteamNetworkingIdentityType_Invalid;
    pMsg->m_identityPeer.m_cbSize = 0;

    // Set the release function
    pMsg->m_pfnRelease = ReleaseFunc;

    // Clear these fields
    pMsg->m_nConnUserData = 0;
    pMsg->m_usecTimeReceived = 0;
    pMsg->m_nMessageNumber = 0;
    pMsg->m_nChannel = -1;
    pMsg->m_nFlags = 0;
    pMsg->m_idxLane = 0;
    pMsg->m_links.Clear();
    pMsg->m_linksSecondaryQueue.Clear();

    return pMsg;
}
```

몇몇 필드를 기본값으로 초기화하고, `m_pfnRelease`를 `CSteamNetworkingMessage::ReleaseFunc`로 세팅한다.\
이게 메시지 객체 자체를 해제할 때 불리는 함수다.\
메시지는 `new`로 할당했었으니 당연히...
```cpp
void CSteamNetworkingMessage::ReleaseFunc( SteamNetworkingMessage_t *pIMsg ) {
    CSteamNetworkingMessage *pMsg = static_cast<CSteamNetworkingMessage *>( pIMsg );

    // Free up the buffer, if we have one
    if ( pMsg->m_pData && pMsg->m_pfnFreeData )
        (*pMsg->m_pfnFreeData)( pMsg );
    pMsg->m_pData = nullptr; // Just for grins

    // We must not currently be in any queue.  In fact, our parent
    // might have been destroyed.
    Assert( !pMsg->m_links.m_pQueue );
    Assert( !pMsg->m_links.m_pPrev );
    Assert( !pMsg->m_links.m_pNext );
    Assert( !pMsg->m_linksSecondaryQueue.m_pQueue );
    Assert( !pMsg->m_linksSecondaryQueue.m_pPrev );
    Assert( !pMsg->m_linksSecondaryQueue.m_pNext );

    // Self destruct
    // FIXME Should avoid this dynamic memory call with some sort of pooling
    delete pMsg;
}
```
...`delete`로 해제한다.

이 메시지 자체의 동적 할당까지 피하고 싶다면, API 권장사항인 `AllocateMessage()` 호출을 피해야 한다.\
그리고 거의 같은 기능을 하지만, pool에서 할당받는 것으로 대체한 버전의 `MyAllocateMessage()`라던가 만들고,\
`m_pfnRelease`도 `ReleaseFunc()` 유사하지만 pool에 반환하는 `MyReleaseFunc()`로 바꿔야 할 것이다.

# C#에서의 pooling 처리

C# P/Invoke를 쓰는 입장에서, 이 할당/해제 함수들을 C# 측에서 `Marshal.GetFunctionPointerForDelegate()`로 세팅한다면?\
메시지 하나 할당 및 해제할 때마다 managed -> unmanaged -> managed 를 왔다갔다 해야한다.\
이는 성능 문제를 야기할 것으로 보이므로, C++ 측에서 pooling 처리하는 함수를 만들고,\
C# 측에서는 내가 만든 함수를 P/Invoke로 호출하도록 짜야 할 것이다.

*마지막 수정 : {{ page.last_modified_at }}*
