---
layout: post
title: Multi Plaza 개발일지 (02) - Snapshot Interpolation 구현
tags: [Devlog]
author: copyrat90
last_modified_at: 2025-06-30T15:39:00+09:00
---

플레이어 위치 동기화를 하기 위해, Snapshot Interpolation을 내 방식으로 구현했다.

이번에도 YouTube 영상부터.

# 테스트 영상

[![](https://img.youtube.com/vi/yw4ub8FqlTc/hqdefault.jpg)](https://www.youtube.com/watch?v=yw4ub8FqlTc)


# 개념

Snapshot Interpolation 개념에 대해서는 인터넷에 좋은 자료가 많다.
* [Fast-Paced Multiplayer (Part III) - Entity Interpolation (내 번역)](/2025/04/05/fast-paced-multiplayer-3#entity-interpolation)
* [Gaffer on Games - Snapshot Interpolation](https://gafferongames.com/post/snapshot_interpolation/)
* [Valve: Source Multiplayer Networking - Entity interpolation](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking#Entity_interpolation)

위 글들 중 Valve가 쓴 글에 있는 이미지를 빌려와서 초간단 요약하자면...
![](https://developer.valvesoftware.com/w/images/thumb/f/ff/Source_Multiplayer_Networking_-_Interpolation.png/750px-Source_Multiplayer_Networking_-_Interpolation.png)


네트워크로부터 다른 플레이어의 위치 정보<sub>Snapshot</sub>가 특정 주기<sub>Snapshot Interval</sub>로 수신된다.\
(e.g. 위 이미지 상으로는 매 0.05초마다 수신)

이를 이용한 가장 간단한 위치 동기화 방법은, 수신된 위치로 즉시 워프하는 것을 반복하는 것이다.\
하지만 그 방식은 문제점이 많다.

* 0.05초마다 받은 위치를 그대로 렌더링하면 20 fps가 되는데, 끊겨 보일것이다.
* 네트워크 환경상 들쭉날쭉<sub>jitter</sub>한 수신은 필연적이라,\
  어떤 때는 순식간에 여러번 워프하고, 어떤 때는 한참 제자리에 있고 할 것이다.
* UDP 송신 시 패킷 순서가 뒤섞이거나, 아예 유실될 수도 있는데,\
  그러면 갑자기 뒤로 이동하거나, 한참 제자리에 있다가 저 멀리로 워프할 것이다.

그러므로, 최소한 위치 정보를 2개는<sub>#340, #342</sub> 받은 시점부터 시작해, 두 위치 정보 사이를 시간에 따라<sub>Current Rendering Time</sub> 보간<sub>Interpolation</sub>해서 렌더링하는 게 기본이다.

여기에 추가로, 들쭉날쭉한/뒤섞인/유실된 수신에 대응하기 위해서, 즉시 렌더링하지 않고 조금 딜레이를 둬서 렌더링을 시작한다.\
(e.g. 위 이미지상으로는 Interpolation Time이 0.1초이므로, 2개의 Snapshot을 받아도 평균 0.05초는 더 기다렸다가 렌더링을 시작하게 된다.)

딜레이를 더 두는 점은, 유튜브의 버퍼링을 생각하면 이해가 쉬울 것이다.\
한번 버퍼링이 걸리면, 추가 영상 데이터를 꽤 많이 버퍼링한 다음에야 재생이 재개되는데, 또 끊기는 걸 최소화하기 위해서다.


# 내 프로토콜의 차이점

보통 구현을 보면, Unreliable한 패킷을 가만히 정지해 있을 때도 계속 보내주는 식으로 구현하는 경우가 꽤 있다.\
예를 들어 [Mirror의 Snapshot Interpolation: Using the algorithm](https://mirror-networking.gitbook.io/docs/manual/components/network-transform/snapshot-interpolation#using-the-algorithm)을 보면, 다음과 같은 경고문이 써있다:

> Note how `NetworkTransform` sends snapshots **every** `sendInterval` over the **unreliable** channel. Do not send **only if changed**, this would require knowledge about the other end's last received snapshot (either over **reliable**, or with a **notify** algorithm).

하지만 가만히 있는데 위치 정보를 주기적으로 보내는 건 너무 낭비가 심한데...

사실, 위 경고문을 토대로 잘 생각해보면, 마지막으로 정지한 위치를 reliable하게 공유하기만 하면 된다는 것을 알 수 있다. 그러면 같은 위치에 멈출 수 있을 테니.\
그래서, 내가 정의한 프로토콜에서는, **이동** 중에 보내는 위치는 **Unreliable**하게, **정지**할 때 보내는 위치는 **Reliable**하게 송신한다.

한참 정지해 있다가 이동을 시작하는 경우는, 다시 유튜브 버퍼링으로 비유하면 데이터가 한참 안 오는 상황일 뿐이다.
그냥 2번째 Snapshot이 올 때까지 기다렸다가, 오고 난 후부터 일정 딜레이를 더 기다렸다가 이동을 재개하면 된다.


# 구현 세부사항

## Payload

일단 네트워크에서 수신한 시간은 믿을 수가 없으므로, 위치를 보내는 측에서 Snapshot에 시퀀스 번호<sub>sequence number</sub>를 넣어서 보내야 한다.\
1초에 10번 보낸다고 치면, 시퀀스 번호가 +1 증가한 Snapshot은 직전 위치 Snapshot으로부터 100 ms 이후의 위치 정보라는 것을 알 수 있다.

시퀀스 번호는 간단히 2 bytes 정수로 하면 될 것 같다.
그런데 overflow가 일어나면, 어느 게 더 최신 Snapshot일지 어떻게 판단할까?

[Gaffer on Games - Handling Sequence Number Wrap-Around](https://gafferongames.com/post/reliability_ordering_and_congestion_avoidance_over_udp/#handling-sequence-number-wrap-around)를 보자:
```c
inline bool sequence_greater_than( uint16_t s1, uint16_t s2 )
{
    return ( ( s1 > s2 ) && ( s1 - s2 <= 32768 ) ) ||
           ( ( s1 < s2 ) && ( s2 - s1  > 32768 ) );
}
```

값이 절반보다 멀리 떨어져 있다면, 한바퀴 돈 것<sub>wrap-around</sub>으로 보고 숫자가 낮은 쪽을 미래 값으로 판정하겠다는 게 핵심이다.\
일반적인 `ushort`와는 정렬 방식이 달라지다보니, 실수를 막기 위해 별도의 `struct SequenceNumber16 : IComparable<SequenceNumber16>, IEquatable<SequenceNumber16>`을 만들어 사용했다.

위치는 그냥 X, Y좌표를 `float` 2개로 넣어서 보내자.\
(나중에 월드 가장자리 좌표를 확정짓고 나면, 고정 소수점으로 좀 더 압축할 수도 있겠지만, 당장은 모르니.)

그리고 당연히 누가 보냈는지를 알아야하므로, 1 byte의 플레이어 ID가 들어간다.\
(현재 프로토콜 상 플레이어는 255명까지만 지원한다. 플레이어 ID 크기를 줄이기 위함도 있고, `Dictionary` 대신 고정 크기 배열을 쓰기 위함도 있다.)

따라서 Payload는 다음과 같이 요약된다.

* User index (8 bits)
* Client-side sequence number (16 bits)
* Position
    * X (float, 32 bits)
    * Y (float, 32 bits)

여기다가 header와 footer를 넣고, [`nalchi::bit_stream_reader` 구현 디테일로 인한 4 bytes ceil을 거치고 나면](/2025/06/13/multi-plaza-devlog-01.html#2-nalchi--nalchisharp-라이브러리-구현), 메시지 크기는 16 bytes 가 된다.


## Snapshot 버퍼

저장하고 있는 Snapshot은 고작해야 서너개에 불과하지만, Unreliable 송신으로 순서가 뒤섞인 패킷을 재정렬할 필요가 있으므로, [`SortedList<SequenceNumber16,Vector2>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.sortedlist-2?view=net-9.0)에 쌓아놓는 게 적절해보인다.


## 시작 딜레이

Interpolation delay는 [Gaffer on Games - Snapshot Interpolation: Handling Real World Conditions](https://gafferongames.com/post/snapshot_interpolation/#handling-real-world-conditions)에 나온 대로, 일단 350 ms를 사용한다.\
2번째 메시지가 온 시점까지 이미 100 ms를 기다린 셈이므로, Start delay는 250 ms가 될 것이다.

딜레이가 좀 길어보이긴 하지만, 내 게임 기획상 월드는 로비에 해당하고, 충돌 판정이나 전투 같은 건 없으니 이대로 진행해도 무방할 것 같다.


# 설계 및 구현

## 설계

지원해야 하는 연산은 아래와 같다.

* 위치 Snapshot을 수신받을 시, `<SequenceNumber16, Vector2>`를 버퍼에 추가한다.
    * Snapshot을 받은 시점에 2개가 됐을 경우, 일정 딜레이 이후에 재생을 시작한다.
      따라서 `현재 시각 + delay`에 1번째 Snapshot의 렌더링을 시작하겠다고 표시한다.
* 현재 시각 기준으로 보간<sub>Interpolation</sub>된 위치를 받아온다.
    * Snapshot이 1개뿐이라면, 해당 위치. (0개일 수는 없음)
    * 아직 1번째 Snapshot의 시작 시각조차 되지 않았다면, 1번째 위치.
    * 나머지 경우는, 1번째와 2번째 위치를 현 시각에 맞는 위치로 보간해서 반환.
        * 만일 2번째 위치 초과한 시각까지 됐다면, 1번째 위치를 삭제하고 렌더링 시작 시각 표시를 `(시퀀스 번호 차이 * 100 ms)`만큼 증가시킨다.
          이후, 남아있는 Snapshot들을 가지고 처음부터 조건 판단을 되풀이한다.
* 유저가 나간 자리에 새로 들어오면, 버퍼를 초기화하고 `<초기 시퀀스 번호, 초기 좌표>`를 버퍼에 추가한다.


## 구현

복잡할 줄 알았는데, 구현해놓고 보니 생각보다 짧다.\
고로 주석 포함해 전문을 싣는다.

참고로 코드가 Godot Engine과 프로젝트 내 다른 코드에 의존적이라서, 복붙해 쓰시려거든 손 좀 봐야 할 것이다.

```cs
namespace MultiPlaza.Utils;

using System;
using System.Collections.Generic;
using Godot;
using MultiPlaza.Services;
using MultiPlaza.Services.Server.Options;

/// <summary>
/// <para>Interpolates the position snapshots.</para>
/// <para>See <a href="https://gafferongames.com/post/snapshot_interpolation/">Gaffer on Games: Snapshot Interpolation</a> for more info.</para>
/// </summary>
public class SnapshotInterpolator
{
    private const ulong SnapshotInterval = 1_000_000 / WorldServerLoopOptions.TicksPerSecond;
    private const ulong InterpolationStartDelay = 250_000;

    private const int DefaultBufferCapacity = 8;

    private readonly SortedList<SequenceNumber16, Vector2> snapshots;

    private ulong firstSnapshotTime;

    /// <summary>
    /// Initializes a new instance of the <see cref="SnapshotInterpolator"/> class.
    /// </summary>
    /// <param name="capacity">Initial buffer capacity.</param>
    public SnapshotInterpolator(int capacity = DefaultBufferCapacity)
    {
        this.snapshots = new(capacity);
    }

    /// <summary>
    /// Gets the number of snapshots in the interpolator.
    /// </summary>
    public int Count => this.snapshots.Count;

    /// <summary>
    /// <para>Resets the interpolator with the given position.</para>
    /// <para>This <b>must</b> be called first before doing anything.</para>
    /// </summary>
    /// <param name="sequence">Initial sequence number.</param>
    /// <param name="initialPosition">Initial position.</param>
    public void Reset(SequenceNumber16 sequence, Vector2 initialPosition)
    {
        this.snapshots.Clear();
        this.snapshots.Add(sequence, initialPosition);
    }

    /// <summary>
    /// Adds a snapshot to the interpolator, using <see cref="Time.GetTicksUsec"/> as a current time.
    /// </summary>
    /// <param name="sequence">Snapshot sequence number.</param>
    /// <param name="position">Snapshot position.</param>
    /// <returns><see cref="bool"/> Whether the snapshot was successfully added, as it was not duplicated.</returns>
    public bool TryAddSnapshot(SequenceNumber16 sequence, Vector2 position)
        => this.TryAddSnapshot(sequence, position, Time.GetTicksUsec());

    /// <summary>
    /// Adds a snapshot to the interpolator.
    /// </summary>
    /// <param name="sequence">Snapshot sequence number.</param>
    /// <param name="position">Snapshot position.</param>
    /// <param name="currentTime">Current time in microseconds.</param>
    /// <returns><see cref="bool"/> Whether the snapshot was successfully added, as it was not duplicated.</returns>
    public bool TryAddSnapshot(SequenceNumber16 sequence, Vector2 position, ulong currentTime)
    {
        this.ThrowIfNotInitialized();

        if (this.snapshots.ContainsKey(sequence))
        {
            return false;
        }

        this.snapshots.Add(sequence, position);

        // If just added snapshot is the second one
        if (this.snapshots.Count == 2)
        {
            // Starts interpolation with a delay
            this.firstSnapshotTime = currentTime + InterpolationStartDelay;
        }

        return true;
    }

    /// <summary>
    /// Gets the interpolated position between snapshots for the current time, using <see cref="Time.GetTicksUsec"/> as a current time.
    /// </summary>
    /// <returns><see cref="Vector2"/> Interpolated position.</returns>
    public Vector2 GetInterpolatedPosition()
        => this.GetInterpolatedPosition(Time.GetTicksUsec());

    /// <summary>
    /// Gets the interpolated position between snapshots for the current time.
    /// </summary>
    /// <param name="currentTime">Current time in microseconds.</param>
    /// <returns><see cref="Vector2"/> Interpolated position.</returns>
    public Vector2 GetInterpolatedPosition(ulong currentTime)
    {
        this.ThrowIfNotInitialized();

        while (true)
        {
            if (this.snapshots.Count == 1)
            {
                // If there's only 1 snapshot, use its position as-is.
                return this.snapshots.GetValueAtIndex(0);
            }
            else if (currentTime <= this.firstSnapshotTime)
            {
                // Interpolation is currently delayed, use the first position.
                return this.snapshots.GetValueAtIndex(0);
            }
            else
            {
                ulong secondSnapshotTime = this.GetSecondSnapshotTime();

                // If overshoot the second,
                if (currentTime > secondSnapshotTime)
                {
                    // Remove the first snapshot, as it's no longer needed
                    this.snapshots.RemoveAt(0);
                    this.firstSnapshotTime = secondSnapshotTime;

                    // We have to check conditions from the start
                    continue;
                }
                else
                {
                    // Calculate the weight of the interpolation
                    float weight = (float)((currentTime - this.firstSnapshotTime) / (double)(secondSnapshotTime - this.firstSnapshotTime));

                    Vector2 pos1 = this.snapshots.GetValueAtIndex(0);
                    Vector2 pos2 = this.snapshots.GetValueAtIndex(1);

                    // Interpolate between 1st and 2nd positions
                    return pos1.Lerp(pos2, weight);
                }
            }
        }
    }

    private ulong GetSecondSnapshotTime()
    {
        if (this.snapshots.Count <= 1)
        {
            throw new InvalidOperationException("Second snapshot doesn't exist");
        }

        SequenceNumber16 seq1 = this.snapshots.GetKeyAtIndex(0);
        SequenceNumber16 seq2 = this.snapshots.GetKeyAtIndex(1);

        return this.firstSnapshotTime + ((seq2 - seq1).Number * SnapshotInterval);
    }

    private void ThrowIfNotInitialized()
    {
        if (this.snapshots.Count == 0)
        {
            throw new InvalidOperationException($"SnapshotInterpolator is not initialized");
        }
    }
}
```


*마지막 수정 : {{ page.last_modified_at }}*
