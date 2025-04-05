---
layout: post
title: GameNetworkingSockets 분석 (2) - 메시지 동적 할당
tags: [GameNetworkingSockets]
author: copyrat90
last_modified_at: 2025-04-05T22:08:00+09:00
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
그리고 거의 같은 기능을 하지만, pool에서 할당받는 것으로 대체한 버전의 `my_allocate_message()`라던가 만들고,\
`m_pfnRelease`도 `ReleaseFunc()` 유사하지만 pool에 반환하는 `MyReleaseFunc()`로 바꿔야 할 것이다.

수정 1: 이거 해보니까 불가능하다. 이유는 `CSteamNetworkingMessage`가 `SteamNetworkingMessage_t`를 상속받아,\
`m_links`와 `m_linksSecondaryQueue`라는 private field를 추가하기 때문이다.
```cpp
pMsg->m_links.Clear();
pMsg->m_linksSecondaryQueue.Clear();
```
이걸 내 함수에서 초기화 할 수가 없다.

GameNetworkingSockets GitHub repo에 [Issue](https://github.com/ValveSoftware/GameNetworkingSockets/issues/369)로 올려놓음.

수정 2: 꼼수가 있다.\
`CSteamNetworkingMessage` API가 노출이 안 돼 있을 뿐이므로, 그냥 내 라이브러리 쪽으로 `CSteamNetworkingMessage` 선언부를 복붙하면 된다.\
이러면 근데 GNS 소스 코드가 수정되면 내 라이브러리도 수정해야 하니까 제대로 된 해결법은 아니긴 하다.

그리고 pooling을 통해 성능을 올리려면 thread-local한 pool을 구현해야 할 것 같은데...\
시간이 남거나 bottleneck이 되면 그 때 가서 고민해야겠다.

# Pooling 처리

## 고민 1

C# P/Invoke를 쓰는 입장에서, 이 할당/해제 함수들을 C# 측에서 `Marshal.GetFunctionPointerForDelegate()`로 세팅한다면?\
메시지 하나 할당 및 해제할 때마다 managed -> unmanaged -> managed 를 왔다갔다 해야한다.

이는 성능 문제를 야기할 것으로 보이므로, C++ 측에서 pooling 처리하는 함수를 만들고,\
C# 측에서는 내가 만든 함수를 P/Invoke로 호출하도록 짜야 할 것이다.

## 고민 2

다시 생각해보니, 어차피 managed -> unmanaged 로 payload 복사를 피하기 위해서는 [`SendMessages()`](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#SendMessages)를 쓸 수 밖에 없고,\
그러려면 [`AllocateMessage()`](https://partner.steamgames.com/doc/api/ISteamNetworkingUtils#AllocateMessage)든 직접 만든 `my_allocate_message()`든 호출해서 unmanaged `CSteamNetworkingMessage`를 매번 받아야 한다.\
결국 C++ 측에서 pooling 하도록 처리해도, 메시지 하나 할당할 때마다 managed -> unmanaged 쪽으로 P/Invoke 호출이 필요하다는 말이다.

그러면 차라리 C# 쪽에서 unmanaged 메모리를 pooling하도록 처리하고, 해제 함수만 `Marshal.GetFunctionPointerForDelegate()`로 하면 어떨까?\
이러면 할당 대신, 메시지 하나 해제할 때마다 unmanaged -> managed 쪽으로 콜백이 일어날 것이다.

뭐가 더 나을진 모르겠으나, 어쨌든 1회의 switching이니까 아주 큰 차이는 없을 것으로 보인다.\
그리고 C++ 코드 추가해서 DLL을 추가로 불러오는 건 여간 귀찮은 게 아니니... C# 쪽에서 unmanaged 메모리를 pooling할까?

## Shared Payload

같은 payload를 여러 메시지가 공유하는 경우가 많을 것이다.\
(특정 캐릭터가 주변 캐릭터 모두에게 이동 메시지를 보내는 등)\
이걸 payload는 1번만 할당하고, 여러 메시지가 그 payload를 공유하도록 하면 좋을 것이다.

이러려면, `m_pfnFreeData`에서 payload가 가리키는 메모리를 바로 해제시키는 게 아니라,\
그 payload의 reference count를 둬서 그걸 감소시키고 0이 되면 반환하도록 처리해야한다.

문제는 `m_pData`는 전송할 payload 데이터만 포함하고 있어야 하니, reference count를 저장할 공간을 어떻게 두냐는 건데...\
그냥 앞 4바이트에 ref count를 두고, `m_pData`에는 실제 전송할 payload인 뒤 4바이트부터의 `IntPtr`을 저장해놓으면 된다.\
ref count의 alignment를 맞추면서 할당하려면, [`NativeMemory.AlignedAlloc()`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.nativememory.alignedalloc?view=net-8.0#system-runtime-interopservices-nativememory-alignedalloc(system-uintptr-system-uintptr))을 쓰면 되겠다.

## 고민 3

하지만 payload 앞에 붙은 ref count를 C#에서 [`Interlocked.Increment()`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.interlocked.increment?view=net-9.0#system-threading-interlocked-increment(system-int32@))할 수가 없다... unmanaged 메모리라 `ref int`가 안 된다.\
다시 C++ 코드를 추가하는 방향으로 회귀해서 생각해보자...

결국 위에서 말한 걸 하려면 C++ 측에서 아래 함수들을 제공해야 한다.
1. `allocate_shared_payload()` : ref count를 앞에 숨겨놓은 payload의 pointer를 반환해주는 함수.
1. `add_shared_payload_to_message()` : `CSteamNetworkingMessage`에 payload를 추가하는 함수. 여기서 `++ref_count`와 `m_pfnFreeData` 함수 포인터 세팅도 수행.
1. `remove_shared_payload_from_message()` : `--ref_count`후 0이면 해제하는 함수. 이게 바로 `m_pfnFreeData`에 세팅될 함수.
1. `force_deallocate_shared_payload()` : 만일 예외 상황이 터져서 메시지에 payload를 추가 못하거나, 전송 못하면, payload를 수동으로 해제해야 하는데, 그 때 쓰일 함수.

그리고 이걸 P/Invoke로 호출할 수 있어야 하므로 `extern "C" __declspec(dllexport)`를 붙여 노출시키면 될 것이다.

## Shared Payload 구현

그리 길지 않으니 그냥 전체를 실어 놓는다.

```cpp
using ref_count_t = std::atomic_int32_t;

/// @brief Allocates the shared payload with a hidden reference count.
/// @param size Size to allocate space.
/// @return Allocated space if it succeeded, otherwise `nullptr`.
GNS_PRAC_INTERFACE void* gns_prac_allocate_shared_payload(std::int32_t size) {
    if (size <= 0)
        return nullptr;

    // allocate space for (ref count + payload size)
    void* ptr = GNS_PRAC_ALIGNED_ALLOC(alignof(ref_count_t), sizeof(ref_count_t) + size);
    if (!ptr)
        return nullptr;

    // use the front space as a ref count
    ref_count_t* ref_count = ::new (static_cast<void*>(ptr)) ref_count_t;
    ;
#if __cplusplus < 202002L // explicit zero init required before C++20
    ref_count->store(0, std::memory_order_relaxed);
#else
    ((void)ref_count); // suppress unused variable warning
#endif

    // return the payload space (i.e. ref count is hidden)
    return (std::byte*)ptr + sizeof(ref_count_t);
}

/// @brief Adds the shared payload to the message.
///
/// This increases the reference count of the payload.
///
/// You MUST use the shared payload allocated with `allocate_shared_payload()`, nothing else.
/// @param msg Message to add the payload to.
/// @param payload Payload to add to.
/// @param size Size of the payload.
GNS_PRAC_INTERFACE void gns_prac_add_shared_payload_to_message(SteamNetworkingMessage_t* msg, void* payload,
                                                               std::int32_t size) {
    // ref count should exist before the payload
    ref_count_t* ref_count = reinterpret_cast<ref_count_t*>((std::byte*)payload - sizeof(ref_count_t));

    // increase the ref count
    ref_count->fetch_add(1, std::memory_order_relaxed);

    // add the payload to the message
    msg->m_pData = payload;
    msg->m_cbSize = size;
    msg->m_pfnFreeData = gns_prac_remove_shared_payload_from_message;
}

/// @brief Removes the shared payload from the message.
///
/// This decreases the reference count of the payload, and deallocates the payload if the ref count reaches zero.
///
/// This is a callback function which automatically set when you call `add_shared_payload_to_message()`,
/// so you don't need to use this function directly.
/// @param msg Message to remove the payload from.
GNS_PRAC_INTERFACE void gns_prac_remove_shared_payload_from_message(SteamNetworkingMessage_t* msg) {
    // ref count should exist before the payload
    ref_count_t* ref_count = reinterpret_cast<ref_count_t*>((std::byte*)msg->m_pData - sizeof(ref_count_t));

    // if this was the last shared reference
    if (1 == ref_count->fetch_sub(1, std::memory_order_relaxed))
    {
        // destroy the ref count
        ref_count->~ref_count_t();

        // free the space (ref count is the alloc address)
        GNS_PRAC_ALIGNED_FREE(ref_count);
    }
}

/// @brief Force deallocate the shared payload.
///
/// This is only necessary if you have an exception in your program
/// which prevents sending the message with already allocated shared payload.
/// @param payload Payload to deallocate.
GNS_PRAC_INTERFACE void gns_prac_force_deallocate_shared_payload(void* payload) {
    // ref count should exist before the payload
    ref_count_t* ref_count = reinterpret_cast<ref_count_t*>((std::byte*)payload - sizeof(ref_count_t));

    // destroy the ref count
    ref_count->~ref_count_t();

    // free the space (ref count is the alloc address)
    GNS_PRAC_ALIGNED_FREE(ref_count);
}
```

*마지막 수정 : {{ page.last_modified_at }}*
