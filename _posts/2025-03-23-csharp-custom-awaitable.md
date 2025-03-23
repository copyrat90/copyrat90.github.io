---
layout: post
title: (C#) async-await 패턴에 사용할 custom awaitable 작성하기
tags: [C#]
author: copyrat90
last_modified_at: 2025-03-23T15:26:00+09:00
---

C# 에서 async-await 패턴의 작동 방식과, 그를 이용한 custom awaitable 작성하기.

# 배경

최근 2주간 [GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)를 [C#에서 P/Invoke로 감싸는 나만의 라이브러리 GnsSharp](https://github.com/nalchi-net/GnsSharp)을 직접 바닥부터 만들었다.

더 나아가서 아예 Steamworks API 전용 기능 일부까지 binding하고 있었는데, 그중 비동기 [Steam CallResults](https://partner.steamgames.com/doc/sdk/api#callresults)를 C# 스타일에 맞게 async-await으로 만들어야겠다고 생각해서, 한번 조사해봤다.

# async-await의 동작 방식

기본 개념은 아래 이미지 한 장이면 다 설명이 된다. [(출처)](https://vkontech.com/exploring-the-async-await-state-machine-the-awaitable-pattern/)

![](https://i0.wp.com/vkontech.com/wp-content/uploads/2020/11/image.png)

좀 더 구체적으로 코드상으로 어떻게 표현되는지 보자.

C# 에서는 `GetAwaiter()` method를 갖는 클래스는 자동으로 awaitable이 된다. (duck typing?)\
이 때, 반환되는 awaiter는 다음 조건들을 만족시켜야 한다.
* [`INotifyCompletion`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.inotifycompletion?view=net-9.0) 또는 [`ICriticalNotifyCompletion`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.icriticalnotifycompletion?redirectedfrom=MSDN&view=net-9.0) 상속.
    * 이 말은, [`INotifyCompletion.OnCompleted(Action)`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.inotifycompletion.oncompleted?view=net-9.0#system-runtime-compilerservices-inotifycompletion-oncompleted(system-action))을 구현해야 한다는 말이다.
    * 후자를 상속받은 경우, 추가로 [`ICriticalNotifyCompletion.UnsafeOnCompleted(Action)`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.icriticalnotifycompletion.unsafeoncompleted?view=net-9.0#system-runtime-compilerservices-icriticalnotifycompletion-unsafeoncompleted(system-action))도 구현해야 한다.
* `bool IsCompleted { get; }` property를 갖고 있어야 한다.
* `GetResult()` method를 갖고 있어야 한다.
    * 이 녀석의 반환형이 `await`시에 반환되는 type이 된다.

`var outcome = await task;`를 했을 때, 위 멤버들이 실제 작동하는 방식은 아래와 비슷하다. [(출처)](https://www.jacksondunstan.com/articles/4918)
```cs
var awaiter = task.GetAwaiter();
if (awaiter.IsCompleted)
{
    // Remove 'outcome =' if `GetResult` returns void
    outcome = awaiter.GetResult();
}
else
{
    SuspendTheFunction();

    Action continuation = () => {
        ResumeTheFunction();
        // Remove 'outcome =' if `GetResult` returns void
        outcome = awaiter.GetResult();
    };
    var cnc = awaiter as ICriticalNotifyCompletion;
    if (cnc != null)
    {
        cnc.UnsafeOnCompleted(continuation);
    }
    else
    {
        awaiter.OnCompleted(continuation);
    }
}
```

즉, 우선 `GetAwaiter()`로 awaiter를 가져오고:
* `awaiter.IsCompleted`가 `true`이면 이미 비동기 처리가 완료된 것이므로 `awaiter.GetResult()`로 즉시 결과를 가져온다.
    * 이 경우, method의 중단 없이 현재 thread가 계속 후속 method 코드를 실행할 것이다.
* `awaiter.IsCompleted`가 `false`이면 아직 비동기 처리가 완료되지 않은 것이므로:
    1. 현재 실행중인 method를 중단시킨다.
    1. 중단점으로부터 method를 재시작하는 `ResumeTheFunction()`이라는 특수한 함수를 만들고, 그걸 호출 후 결과를 받아오는 `awaiter.GetResult()`를 그 이후에 호출하도록 하는 `continuation` Action을 하나 만든다.
    1. awaiter의 타입에 따라 `OnCompleted()`나 `UnsafeOnCompleted()`에 `continuation`을 매개변수로 넘겨서 method의 재개를 등록하게 한다.
    * 이 경우, 현재 thread와 후속 method 코드를 재개하는 thread가 다를 수도 있다.\
      물론 같을 수도 있다. 그건 `continuation`을 누가 Invoke하는 지에 따라 결정될 것.

위 동작을 나타낸 이미지 [(출처)](https://vkontech.com/exploring-the-async-await-state-machine-the-awaitable-pattern/)

![](https://i0.wp.com/vkontech.com/wp-content/uploads/2020/11/image-1.png)

## 예시 1. `Task`와 `TaskAwaiter`

흔히 자주 쓰이는 `System.Threading.Tasks`의 [`Task`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task-1?view=net-9.0)라고 특별한 건 아니고, 위 패턴을 따라 구현되어 있다.\
[`Task.GetAwaiter()`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.getawaiter?view=net-9.0)는 [`TaskAwaiter`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.taskawaiter?view=net-9.0)를 반환하며, 이는 위 awaiter의 조건들을 만족한다.

작업이 완료되면, C# 런타임의 [`ThreadPool`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool?view=net-9.0) 내의 thread가 `continuation`을 호출해 후속 method 코드를 재개한다.\
(참고로, Windows에서 장치 I/O를 대기하는 경우에 한정해 [기존에는 IOCP worker thread가 관여했는데](https://blog.stephencleary.com/2013/11/there-is-no-thread.html), .NET 7부터는 [managed thread pool에서 batch polling을 통해 처리한다고 한다.](https://github.com/dotnet/runtime/issues/46610#issuecomment-1109014759))

## 예시 2. GodotSharp의 `SignalAwaiter`

Godot Engine에서 [`GodotObject.ToSignal()`](https://github.com/godotengine/godot/blob/2303ce843a362cf5497ca3336451de23793eb711/modules/mono/glue/GodotSharp/GodotSharp/Core/GodotObject.base.cs#L192-L195) method를 호출하면 [`SignalAwaiter`](https://github.com/godotengine/godot/blob/2303ce843a362cf5497ca3336451de23793eb711/modules/mono/glue/GodotSharp/GodotSharp/Core/SignalAwaiter.cs)가 반환되는데, 이 녀석은 awaitable과 awaiter의 역할을 동시에 수행한다.

Signal을 보내면, C++ 엔진 수준에서 [`SignalAwaiter.SignalCallback()`](https://github.com/godotengine/godot/blob/2303ce843a362cf5497ca3336451de23793eb711/modules/mono/glue/GodotSharp/GodotSharp/Core/SignalAwaiter.cs#L33-L65)을 호출해 `continuation`을 수행하는 것으로 보인다.\
기본적으로 게임 루프를 돌리는 main thread가 Signal emission도 담당하는 것으로 보이므로, main thread에서 `await ToSignal()`을 했었다면 후속 처리도 같은 thread일 것이다.

# 내 라이브러리에서의 custom awaitable 구현: `CallTask<T>`

내가 만든 [GnsSharp](https://github.com/nalchi-net/GnsSharp)에서는 [`CallTask<T>`](https://github.com/nalchi-net/GnsSharp/blob/main/GnsSharp/CallTask.cs)가 awaitable과 awaiter의 역할을 동시에 수행한다.

Steamworks API 구현 스타일과 동일하게 [`SteamAPI.RunCallbacks()`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/SteamAPI.cs#L206-L213)를 제공하고, 이걸 유저가 직접 호출해야 한다.\
그러면 [`Dispatcher.RunCallbacks()`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/Dispatcher.cs#L17-L97) -> [`ISteamUtils.OnDispatch()`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/ISteamUtils.cs#L778-L846) -> [`ISteamUtils.HandleCallCompletedResult()`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/ISteamUtils.cs#L848-L870)를 거쳐서 최종적으로 [`CallTask<T>.SetResultFrom()`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/CallTask.cs#L76-L111)이 호출된다.

유저가 [`SteamAPI.RunCallbacks()`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/SteamAPI.cs#L206-L213)를 별도의 thread에서 돌릴 수도 있으므로, 내부에 [`taskLock`](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/CallTask.cs#L17)을 하나 둬서 동기화 문제를 방지한다.

```cs
public void OnCompleted(Action continuation)
{
    bool lockTaken = false;
    try
    {
        Monitor.Enter(this.taskLock, ref lockTaken);

        // 혹시 `taskLock` 걸기 전에 결과를 받은 게 아닌지 double-check
        if (this.IsCompleted)
        {
            // 결과가 왔으면, `taskLock`을 해제하고...
            Monitor.Exit(this.taskLock);
            lockTaken = false;

            // ...`continuation`을 호출해 즉시 재개
            continuation();
        }
        else
        {
            // 결과가 없었으면, 차후에 `RunCallbacks()` 돌리는 thread가 호출하도록
            // `continuation`을 내부에 저장해 둠
            this.continuation = continuation;
        }
    }
    finally
    {
        if (lockTaken)
        {
            Monitor.Exit(this.taskLock);
        }
    }
}
```

`OnCompleted()`가 호출됐는데 (즉 `IsCompleted`가 `false`였는데), `taskLock`을 소유하기 전 틈에 또 `SetResultFrom()`이 먼저 수행되어 결과를 저장할 수 있으므로,\
`taskLock`을 소유한 이후 `IsCompleted`를 double-check 해서, 결과가 그 틈에 들어왔다면 `continuation()`을 호출해 즉시 실행해준다.

결과가 안 들어왔다면 `RunCallbacks()`를 돌리는 thread가 호출하도록 `continuation`을 내부에 저장해둔다.

```cs
public void SetResultFrom(HSteamPipe pipe, ref SteamAPICallCompleted_t callCompleted)
{
    Debug.Assert(T.CallbackParamId == callCompleted.AsyncCallbackId, $"Callback id mismatch (expected {T.CallbackParamId} for {typeof(T)}, got {callCompleted.AsyncCallbackId})");
    unsafe
    {
        Debug.Assert(sizeof(T) == callCompleted.ParamSize, $"Callback param size mismatch (expected {sizeof(T)} for {typeof(T)}, got {callCompleted.ParamSize})");
    }

    bool lockTaken = false;
    try
    {
        Monitor.Enter(this.taskLock, ref lockTaken);

        // Get the call result param directly into `result`
        Span<byte> resultRaw = MemoryMarshal.AsBytes(MemoryMarshal.CreateSpan(ref this.result, 1));
        bool gotResult = Native.SteamAPI_ManualDispatch_GetAPICallResult(pipe, callCompleted.AsyncCall, resultRaw, resultRaw.Length, callCompleted.AsyncCallbackId, out this.isFailed);
        Debug.Assert(gotResult, "There was no call result available");

        this.isCompleted = true;

        // 이미 continuation이 등록된 경우, 재개할 책임은 이쪽에 있다.
        if (this.continuation != null)
        {
            Monitor.Exit(this.taskLock);
            lockTaken = false;

            this.continuation();
        }
    }
    finally
    {
        if (lockTaken)
        {
            Monitor.Exit(this.taskLock);
        }
    }
}
```

`RunCallbacks()`가 결과를 가져와서 `SetResultFrom()`으로 저장할 때는, `taskLock`을 소유한 상태로 `this.result`에 직접 결과를 쓴다.\
이 때, [`MemoryMarshal.AsBytes()`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.memorymarshal.asbytes?view=net-9.0)로 `this.result`를 `Span<byte>` 형식으로 변환하고, 그걸 바로 P/Invoke 함수에 전달하여 불필요한 복사를 최소화한다.\
(이게 어떻게 pinning 되어 P/Invoke 호출이 되는지는 [이전 글](/2025/02/17/nullptr-ref-out-for-interop#설명)에서 설명한 바 있다.)

만일, 이미 `this.continuation`이 등록된 상황이었다면, 재개를 해준다.\
그렇지 않다면 할 일은 없다. `this.isCompleted = true;`로 세팅했으므로, `await` 한 측에서 재개할 것이다.

## 사족: `SteamAPICall_t`로 `CallTask<T>` 찾기

Steamworks API 측에서 돌아오는 CallResult 내부에는, 현재 async request의 핸들인 `SteamAPICall_t`가 같이 저장되어 들어온다. (그냥 `ulong`)\
이걸 이용해 `CallTask<T>`를 찾아야 하는데, 일단 간단하게 [`Dictionary<SteamAPICall_t, ICallTask>` 1개](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/ISteamUtils.cs#L20-L21)에다가 모든 `CallTask<T>`를 저장해두고 처리하고 있다.

이 때, P/Invoke로 비동기 요청을 보내는 함수가 반환하기도 전에 `RunCallbacks()` thread에 호출 결과가 돌아올 가능성도 존재한다.\
그러면 `Dictionary`에 미처 `(SteamAPICall_t, CallTask<T>)`를 집어넣기도 전에 `RunCallbacks()` thread에서 찾으려 들어서 못 찾는 상황도 있을 수 있다.\
이런 불상사를 방지하기 위해, 아래와 같이 미리 `Dictionary`용 `asyncCallTasksLock`을 소유한 다음에야 P/Invoke로 비동기 요청을 보내도록 했다.
```cs
internal CallTask<TResult>? SafeSteamAPICall<T1, T2, TResult>(Func<T1, T2, SteamAPICall_t> nativeCall, T1 param1, T2 param2)
        where TResult : unmanaged, ICallbackParam
        where T1 : allows ref struct
        where T2 : allows ref struct
{
    var task = new CallTask<TResult>();

    // 우선 Dictionary에 대한 lock을 소유하고...
    lock (this.asyncCallTasksLock)
    {
        // ... 그 이후에야 Steam API 비동기 요청을 보냄
        SteamAPICall_t handle = nativeCall(param1, param2);

        if (handle == SteamAPICall_t.Invalid)
        {
            return null;
        }

        // Dictionary에 (handle, task) 쌍을 저장
        this.asyncCallTasks.Add(handle, task);
    }

    return task;
}

private void HandleCallCompletedResult(HSteamPipe pipe, ref CallbackMsg_t msg)
{
    ref var callCompleted = ref msg.GetCallbackParamAs<SteamAPICallCompleted_t>();

    ICallTask? task;

    // Dictionary에 대한 lock을 소유하고...
    lock (this.asyncCallTasksLock)
    {
        // ... 해당하는 `ICallTask`를 찾으려고 시도
        if (this.asyncCallTasks.TryGetValue(callCompleted.AsyncCall, out task))
        {
            // 찾았으면 Dictionary에서 제거
            this.asyncCallTasks.Remove(callCompleted.AsyncCall);
        }
    }

    if (task != null)
    {
        // 찾았으면 `SetResultFrom()` 호출
        task.SetResultFrom(pipe, ref callCompleted);
    }
    else
    {
        Debug.WriteLine($"Got unexpected call result #{callCompleted.AsyncCall}, Id = {callCompleted.AsyncCallbackId}");
    }
}
```

참고로 유명한 Steamworks 바인딩인 [Facepunch.Steamworks](https://github.com/Facepunch/Facepunch.Steamworks)에서는 [이 부분을 lock을 안 걸고 처리](https://github.com/Facepunch/Facepunch.Steamworks/blob/aa87fe557fa0578b4361bf0f23848d64d0c94a79/Facepunch.Steamworks/Classes/Dispatch.cs#L265-L277)하는 것으로 보인다.\
아니 `RunCallbacks()`를 다른 thread에서 돌리면 위험할텐데?

### 사족의 사족: 왜 C#엔 variadic generics가 없는 것인가...

참고로 위 `SafeSteamAPICall<T1, T2, TResult>()`는 그저 매개변수 2개 받는 async Steam API 함수를 호출하는 버전이고,\
실제로는 아래와 같이 [6개까지 overload](https://github.com/nalchi-net/GnsSharp/blob/e2b3db93cc054d880b86d4a813122df9851104c6/GnsSharp/ISteamUtils.cs#L637-L776)가 되어 있다.
```cs
internal CallTask<TResult>? SafeSteamAPICall<T, TResult>(Func<T, SteamAPICall_t> nativeCall, T param);
internal CallTask<TResult>? SafeSteamAPICall<T1, T2, TResult>(Func<T1, T2, SteamAPICall_t> nativeCall, T1 param1, T2 param2);
internal CallTask<TResult>? SafeSteamAPICall<T1, T2, T3, TResult>(Func<T1, T2, T3, SteamAPICall_t> nativeCall, T1 param1, T2 param2, T3 param3);
internal CallTask<TResult>? SafeSteamAPICall<T1, T2, T3, T4, TResult>(Func<T1, T2, T3, T4, SteamAPICall_t> nativeCall, T1 param1, T2 param2, T3 param3, T4 param4);
internal CallTask<TResult>? SafeSteamAPICall<T1, T2, T3, T4, T5, TResult>(Func<T1, T2, T3, T4, T5, SteamAPICall_t> nativeCall, T1 param1, T2 param2, T3 param3, T4 param4, T5 param5);
internal CallTask<TResult>? SafeSteamAPICall<T1, T2, T3, T4, T5, T6, TResult>(Func<T1, T2, T3, T4, T5, T6, SteamAPICall_t> nativeCall, T1 param1, T2 param2, T3 param3, T4 param4, T5 param5, T6 param6);
```

C++ 이었으면 template parameter pack으로 쉽게 처리했을 부분인데, [C# 엔 이게 없는 것으로 보인다.](https://github.com/dotnet/roslyn/issues/5058)\
원래는 그냥 lambda로 매개변수를 죄다 캡쳐했었는데, `Span<T>`는 그게 불가능해서 울며 겨자 먹기로 저 지저분한 overload 방식으로 선회했다. 답이 없나?

질문글을 올리니 누가 [이걸 자동 생성하는 Source Generator](https://github.com/WhiteBlackGoose/InductiveVariadics) 방식을 해법으로 주는데, 그건 좀...

## 사용 예시

GnsSharp의 [`ISteamRemoteStorage`](https://github.com/nalchi-net/GnsSharp/blob/main/GnsSharp/ISteamRemoteStorage.cs)를 이용해 Steam Cloud에서 파일을 읽어오는 예시

```cs
async Task ReadSpacewarCloudFileAsync()
{
    string fileName = "message.dat";

    // Assuming properly initialized, and running callback seperately
    var storage = ISteamRemoteStorage.User!;

    // Get the file size first.
    int size = storage.GetFileSize(fileName);
    if (size == 0)
    {
        Console.WriteLine($"File '{fileName}' doesn't exist");
        return;
    }

    Console.WriteLine($"Size of '{fileName}' was {size}");

    // Start reading file asynchronously.
    CallTask<RemoteStorageFileReadAsyncComplete_t>? readTask
        = storage.FileReadAsync(fileName, 0, (uint)size);

    // Skip if failed to start reading file.
    if (readTask == null)
    {
        Console.WriteLine("File read not initiated");
        return;
    }

    // Await for reading to complete.
    RemoteStorageFileReadAsyncComplete_t? complete = await readTask;

    // Skip if reading failed.
    if (!complete.HasValue)
    {
        Console.WriteLine("File read not complete");
        return;
    }
    if (complete.Value.Result != EResult.OK)
    {
        Console.WriteLine($"File read not complete: {complete.Value.Result}");
        return;
    }

    // Allocate buffer to copy the read bytes.
    Span<byte> raw = stackalloc byte[(int)complete.Value.ReadSize];

    // Copy the result to this buffer.
    if (storage.FileReadAsyncComplete(complete.Value.FileReadAsync, raw))
    {
        // Assuming it's a UTF-8 string, print it.
        string str = Encoding.UTF8.GetString(raw);
        Console.WriteLine($"Read string: {str}");
    }
    else
    {
        Console.WriteLine("FileReadAsyncComplete() failed");
    }
}
```

# 참고 자료

* [Stephen Toub - await anything;](https://devblogs.microsoft.com/dotnet/await-anything/)
* [JacksonDunstan.com - How Async and Await Work](https://www.jacksondunstan.com/articles/4918)
* [Vasil Kosturski - Exploring the async/await State Machine – The Awaitable Pattern](https://vkontech.com/exploring-the-async-await-state-machine-the-awaitable-pattern/)

*마지막 수정 : {{ page.last_modified_at }}*
