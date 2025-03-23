---
layout: post
title: GameNetworkingSockets 분석 (1) - Connection queue & PollGroup queue
tags: [GameNetworkingSockets]
author: copyrat90
last_modified_at: 2025-02-25T15:18:00+09:00
---

[ValveSoftware/GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets) 분석 제 1편.

# GameNetworkingSockets

[Valve](https://www.valvesoftware.com/en)에서 오픈소스로 공개한 UDP 소켓을 게임용으로 추상화한 라이브러리.

[GitHub: ValveSoftware/GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)

TCP처럼 Connection 기반으로 돌아가나, 내부 소켓은 UDP를 사용하고, message 단위로 자르는 작업을 알아서 해 준다.\
그리고 Reliable과 Unreliable한 메시지 둘 다 보낼 수 있다.


# Connection vs PollGroup

Connection 기반으로 돌아가니, 각 Connection에서 메시지를 받는 함수가 존재한다.

[`ISteamNetworkingSockets::ReceiveMessagesOnConnection()`](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#ReceiveMessagesOnConnection)은 아래와 같이\
`ppOutMessages`로 여러 개의 메시지를 여러 pointer를 배열에 넣어 주는 식으로 받도록 되어 있고:
```cpp
int ReceiveMessagesOnConnection( HSteamNetConnection hConn, SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages );
```
이렇게 받아온 메시지들은 사용자 측에서 `SteamNetworkingMessage_t::Release()`를 호출해 직접 해제해줘야 한다.

또한, 이 라이브러리에는 Connection 개념과 별도로 PollGroup 개념이 존재한다.\
그래서 같은 PollGroup에 등록된 모든 Connection의 메시지를 지정한 개수만큼 한 번에 꺼내올 수도 있다.

[`ISteamnetworkingSockets::ReceiveMessagesOnPollGroup()`](https://partner.steamgames.com/doc/api/ISteamnetworkingSockets#ReceiveMessagesOnPollGroup)도 API는 거의 흡사하다.
```cpp
int ReceiveMessagesOnPollGroup( HSteamNetPollGroup hPollGroup, SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages );
```
역시 마찬가지로 사용자 측에서 `SteamNetworkingMessage_t::Release()`를 호출해 메시지를 직접 해제해줘야 한다.


# 의문점 1: 섞어 쓰기?

여기서 의문점이 하나 떠오른다.

PollGroup에 등록된 Connection이 받은 메시지는 무조건 [`ReceiveMessagesOnPollGroup()`](https://partner.steamgames.com/doc/api/ISteamnetworkingSockets#ReceiveMessagesOnPollGroup)으로만 꺼내야 하는가?\
혹은 필요하다면 [`ReceiveMessagesOnConnection()`](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#ReceiveMessagesOnConnection)도 섞어 쓸 수 있나?

## 소스 코드 분석

답을 찾기 위해 GameNetworkingSockets의 소스 코드를 훑어보았다.

우선 [`ReceiveMessagesOnConnection()`](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#ReceiveMessagesOnConnection)부터.
```cpp
int CSteamNetworkingSockets::ReceiveMessagesOnConnection( HSteamNetConnection hConn, SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages ) {
    ConnectionScopeLock connectionLock;
    CSteamNetworkConnectionBase *pConn = GetConnectionByHandleForAPI( hConn, connectionLock, "ReceiveMessagesOnConnection" );
    if ( !pConn )
        return -1;
    return pConn->APIReceiveMessages( ppOutMessages, nMaxMessages );
}

int CSteamNetworkConnectionBase::APIReceiveMessages( SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages ) {
    // Connection must be locked, but we don't require the global lock here!
    m_pLock->AssertHeldByCurrentThread();

    g_lockAllRecvMessageQueues.lock();

    int result = m_queueRecvMessages.RemoveMessages( ppOutMessages, nMaxMessages );
    g_lockAllRecvMessageQueues.unlock();

    return result;
}
```
[`ReceiveMessagesOnConnection()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/csteamnetworkingsockets.cpp#L1378-L1386)에서 [`APIReceiveMessages()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_connections.cpp#L2117-L2128)를 호출,\
모든 수신 메시지 큐에 lock을 걸고 [`SteamNetworkingMessageQueue::RemoveMessages()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_connections.cpp#L188-L207)를 호출한다.

```cpp
int SteamNetworkingMessageQueue::RemoveMessages( SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages ) {
    int nMessagesReturned = 0;
    AssertLockHeld();

    while ( !empty() && nMessagesReturned < nMaxMessages ) {
        // Locate message, put into caller's list
        CSteamNetworkingMessage *pMsg = m_pFirst;
        ppOutMessages[nMessagesReturned++] = pMsg;

        // Unlink from all queues
        pMsg->Unlink();

        // That should have unlinked from *us*, so it shouldn't be in our queue anymore
        Assert( m_pFirst != pMsg );
    }

    return nMessagesReturned;
}

void CSteamNetworkingMessage::Unlink() {
    // Unlink from any queues we are in
    UnlinkFromQueue( &CSteamNetworkingMessage::m_links );
    UnlinkFromQueue( &CSteamNetworkingMessage::m_linksSecondaryQueue );
}

inline void CSteamNetworkingMessage::UnlinkFromQueue( CSteamNetworkingMessage::Links CSteamNetworkingMessage::*pMbrLinks );
```
[`SteamNetworkingMessageQueue::RemoveMessages()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_connections.cpp#L188-L207)에서는 다시 [`CSteamNetworkingMessage::Unlink()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_connections.cpp#L160-L165)를 호출.\
그로 인해, 해당 메시지는 자기 자신을 2개 큐에서 모두 삭제시킨다.\
여기서 `m_links`는 Connection의 큐이고, `m_linksSecondaryQueue`는 PollGroup의 큐이다.

이런 삭제가 가능한 것은, [`CSteamNetworkingMessage` 자신이 2개의 intrusive linked list의 node로서 기능](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_snp.h#L152-L166)하기 때문이다.
```cpp
/// Actual implementation of SteamNetworkingMessage_t, which is the API
/// visible type.  Has extra fields needed to put the message into intrusive
/// linked lists.
class CSteamNetworkingMessage : public SteamNetworkingMessage_t {
    ...
    struct Links {
        SteamNetworkingMessageQueue *m_pQueue;
        CSteamNetworkingMessage *m_pPrev;
        CSteamNetworkingMessage *m_pNext;

        inline void Clear() { m_pQueue = nullptr; m_pPrev = nullptr; m_pNext = nullptr; }
    };

    /// Intrusive links for the "primary" list we are in
    Links m_links;

    /// Intrusive links for any secondary list we may be in.  (Same listen socket or
    /// P2P channel, depending on message type)
    Links m_linksSecondaryQueue;
    ...
};
```

한편, [`CSteamNetworkingSockets::ReceiveMessagesOnPollGroup()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/csteamnetworkingsockets.cpp#L1448-L1459) 또한 마찬가지로 [`SteamNetworkingMessageQueue::RemoveMessages()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_connections.cpp#L188-L207)를 호출하므로:
```cpp
int CSteamNetworkingSockets::ReceiveMessagesOnPollGroup( HSteamNetPollGroup hPollGroup, SteamNetworkingMessage_t **ppOutMessages, int nMaxMessages ) {
    PollGroupScopeLock pollGroupLock;
    CSteamNetworkPollGroup *pPollGroup = GetPollGroupByHandle( hPollGroup, pollGroupLock, "ReceiveMessagesOnPollGroup" );
    if ( !pPollGroup )
        return -1;
    g_lockAllRecvMessageQueues.lock();
    int nMessagesReceived = pPollGroup->m_queueRecvMessages.RemoveMessages( ppOutMessages, nMaxMessages );
    g_lockAllRecvMessageQueues.unlock();
    return nMessagesReceived;
}
```
같은 방식으로 메시지가 2개 큐에서 모두 삭제될 것이다.

결론은, [`ReceiveMessagesOnPollGroup()`](https://partner.steamgames.com/doc/api/ISteamnetworkingSockets#ReceiveMessagesOnPollGroup)과 [`ReceiveMessagesOnConnection()`](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets#ReceiveMessagesOnConnection)을 섞어 써도 상관 없다.\
실제로 그렇게 쓸 일이 있을 지는 모르겠지만.


# 의문점 2: PollGroup 변경

PollGroup의 변경 시에 문제가 되는 상황은 없을까?

이를테면, PollGroup A -> PollGroup B로 변경한 Connection에 대해서,\
`ReceiveMessagesOnPollGroup(A)`를 했을 때 그 Connection의 남은 메시지 일부를 A쪽에서 받아버리는 불상사라든가?

[`CSteamNetworkConnectionBase::SetPollGroup()`](https://github.com/ValveSoftware/GameNetworkingSockets/blob/725e273c7442bac7a8bc903c0b210b1c15c34d92/src/steamnetworkingsockets/clientlib/steamnetworkingsockets_connections.cpp#L781-L862)을 보자.
```cpp
void CSteamNetworkConnectionBase::SetPollGroup( CSteamNetworkPollGroup *pPollGroup ) {
    AssertLocksHeldByCurrentThread( "SetPollGroup" );

    // Quick early-out for no change
    if ( m_pPollGroup == pPollGroup )
        return;

    // Clearing it?
    if ( !pPollGroup ) {
        RemoveFromPollGroup();
        return;
    }

    // Grab locks for old and new poll groups.  Remember, we can take multiple locks without
    // worrying about deadlock because we hold the global lock
    PollGroupScopeLock pollGroupLockNew( pPollGroup->m_lock );
    PollGroupScopeLock pollGroupLockOld;
    if ( m_pPollGroup )
        pollGroupLockOld.Lock( m_pPollGroup->m_lock );

    // Scan all messages that are already queued for this connection,
    // and insert them into the poll groups queue in the (approximate)
    // appropriate spot.  Using local timestamps should be really close
    // for ordering messages between different connections.  Remember
    // that the API very clearly does not provide strong guarantees
    // regarding ordering of messages from different connections, and
    // really anybody who is expecting or relying on such guarantees
    // is probably doing something wrong.
    {
        ShortDurationScopeLock lockMessageQueues( g_lockAllRecvMessageQueues );
        CSteamNetworkingMessage *pInsertBefore = pPollGroup->m_queueRecvMessages.m_pFirst;
        for ( CSteamNetworkingMessage *pMsg = m_queueRecvMessages.m_pFirst ; pMsg ; pMsg = pMsg->m_links.m_pNext ) {
            Assert( pMsg->m_links.m_pQueue == &m_queueRecvMessages );

            // Unlink it from existing poll group queue, if any
            if ( pMsg->m_linksSecondaryQueue.m_pQueue ) {
                Assert( m_pPollGroup && pMsg->m_linksSecondaryQueue.m_pQueue == &m_pPollGroup->m_queueRecvMessages );
                pMsg->UnlinkFromQueue( &CSteamNetworkingMessage::m_linksSecondaryQueue );
            }
            else {
                Assert( !m_pPollGroup );
            }

            // Scan forward in the poll group message queue, until we find the insertion point
            for (;;) {
                // End of queue?
                if ( !pInsertBefore )
                {
                    pMsg->LinkToQueueTail( &CSteamNetworkingMessage::m_linksSecondaryQueue, &pPollGroup->m_queueRecvMessages );
                    break;
                }

                Assert( pInsertBefore->m_linksSecondaryQueue.m_pQueue == &pPollGroup->m_queueRecvMessages );
                if ( pInsertBefore->m_usecTimeReceived > pMsg->m_usecTimeReceived )
                {
                    pMsg->LinkBefore( pInsertBefore, &CSteamNetworkingMessage::m_linksSecondaryQueue, &pPollGroup->m_queueRecvMessages );
                    break;
                }

                pInsertBefore = pInsertBefore->m_linksSecondaryQueue.m_pNext;
            }
        }
    }

    // Tell previous poll group, if any, that we are no longer with them
    if ( m_pPollGroup ) {
        DbgVerify( m_pPollGroup->m_vecConnections.FindAndFastRemove( this ) );
    }

    // Link to new poll group
    m_pPollGroup = pPollGroup;
    Assert( !m_pPollGroup->m_vecConnections.HasElement( this ) );
    m_pPollGroup->m_vecConnections.AddToTail( this );
}
```

위 코드를 보면, 기존 Connection 큐에 있던 메시지를 모두 새 PollGroup 큐로 옮기고 있다.\
(심지어, 새 PollGroup 큐에 넣을 때, 대략적인 수신 시간까지 고려해서 적당한 위치에 끼워넣어준다.)\
옮기기 전, 모든 메시지 큐에 대한 lock을 잡았으므로, 그 틈에 다른 메시지가 수신될 위험성도 없다.\
고로 위에서 말했던 불상사는 발생하지 않을 것이다.

-----

물론, 사용자 측에서도 PollGroup마다 별도 thread에서 메시지를 한번에 꺼내 처리하고 있다면,\
`SetPollGroup()`으로 PollGroup을 바꾸기 전에, 잔여 메시지가 없도록 처리를 해야 할 것이다.

따라서, 사용자측에서 한 번에 메시지를 너무 많이 꺼내면, `SetPollGroup()` 호출 전에 모든 꺼낸 메시지가 처리되기를 한참 기다려야 할 수도 있다.\
그렇다고 메시지를 1개씩 꺼내는 건, C#으로 P/Invoke하는 입장에서 호출 횟수 부담이 너무 크고. 적당히 타협해야겠다.


*마지막 수정 : {{ page.last_modified_at }}*
