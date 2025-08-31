---
layout: post
title: butano-ldtk 최적화
tags: [Devlog]
author: copyrat90
last_modified_at: 2025-09-01T01:44:00+09:00
---

GBA 엔진용 LDtk 레벨 로더 구현 중 최적화했던 부분.

최근 몇주간 [Butano engine](https://github.com/GValiente/butano)용 [LDtk 레벨 로더](https://github.com/copyrat90/butano-ldtk)를 짜던 중이었다.

대충 원리는 Python으로 `my_project.ldtk` 파일을 적절히 파싱한 후, C++ `constexpr` 객체들을 담은 헤더 파일을 codegen하는 것이다.\
그리고 그걸 이용해서 레벨 배경을 조작하는 `ldtk::level_bgs_ptr` 클래스도 추가로 제공한다.

여튼 그걸 설명하려던 건 아니고, 배경 렌더링 부분을 최적화한 일지를 남기려고 한다.

# 최초 구현 (느려터짐)

[최초 commit](https://github.com/copyrat90/butano-ldtk/commit/7d688b75effc1c91298577e230a462edb0d5ee39)

가장 naive한 첫번째 렌더링 구현.
* 카메라 이동시마다 화면에 보이는 모든 타일을 전부 리로드
* 배경 레이어 당 virtual function call 31(열)×21(행) = 651회
    * GBA 해상도가 240x160이고, tile은 8x8이므로, 나누면 30x20일 것 같지만,\
      대부분의 경우 옆 1줄에 추가로 걸치기 때문에 31x21이 된다.
* div, mod 마구마구 사용

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/naivest.gif)

느릴 거라 예상하긴 했지만, 상상 이상의 CPU 사용률이 나온다.\
GBA 에뮬레이터 [Mesen2](https://github.com/SourMesen/Mesen2)로 profiling을 해보자...

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/naivest_profiler.png)

`bg_t::reset_all_cells()` 함수 안에서 94.5% (inclusive) CPU를 사용중이다.\
그 바로 밑에 `thumb_8000F98` 함수가 19.5% (exclusive)를 먹는데,\
이 녀석의 Call Count를 위 함수의 Call Count로 나누면 650.4589번이다.\
이건 8x8 tile cell마다 호출되는 virtual function일 가능성이 높다.

virtual function을 사용하는 이유는, meta-tile 가짓수에 따라 타일 정보를 저장하는 크기가 달라지는데 (`u8`/`u16`),\
이걸 타입을 모르는 채로 generic하게 받아오기 위함이었다.\
근데 이걸 각 cell마다 매번 virtual function call로 받아오고 있으니 느려터진다.

정확히는 배경 레이어가 3개이므로, 프레임당 3 × 651 = 1953번 호출되고 있다.
1번만 호출 -> 구체 타입 파악 -> `static_cast`를 해서 구체 타입의 함수를 호출하는 최적화가 떠오른다.

# 1차 최적화: 가상함수 호출 횟수 줄이기

[가상 함수 651회 -> 1회 최적화 commit](https://github.com/copyrat90/butano-ldtk/commit/0688150a705b0beb712e8459cac658b4fbb546c4)

성능을 위해 `dynamic_cast`는 비활성화된 상태이므로, 그냥 타입을 알려주는 `tile_grid_t::bloated()` 가상함수를 하나 만들고,\
그걸 호출해 구체 타입을 파악 -> `static_cast`를 해서 구체 타입의 함수를 호출하도록 최적화했다.\
(어찌 보면 직접 빚은 `dynamic_cast`라고 볼 수도 있겠다.)

그리고 가상함수는 초기화시 1회만 호출해 구체 타입을 파악하므로, 프레임당 가상함수 호출은 아예 없어졌다.\
대신에 매번 `grid_bloated` 플래그를 판단하는 것으로 대체되었다.

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/no_virtual.gif)

57%의 CPU를 아꼈다.\
배경 레이어가 3개나 된다는 것치곤 성능 변화가 아주 크진 않다.

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/no_virtual_profiler.png)

다음으로 많아보이는 건 divmod 등 정수 나눗셈 관련 코드다.\
대부분의 나눗셈을 그냥 직전 루프의 누적합으로 대체할 수 있을 것으로 보인다.

# 2차 최적화: divmod 제거

[divmod 호출 제거 commit](https://github.com/copyrat90/butano-ldtk/commit/1b23f55333b395b691b96b319184d690ab519a83)

divmod를 초기에 1회만 나눗셈으로 구하고, 그 이후엔 누적합 -> 자리올림을 직접 하는 코드로 대체했다.\
누적 카운터를 계속 +1 한 후 조건 판단하는 것이, 정수 나눗셈을 매번 반복하는 것보다는 훨씬 싸게 먹힌다.

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/no_divmod.gif)

무려 205%의 CPU를 아꼈다.

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/no_divmod_profiler.png)

이젠 그냥 `reset_all_cells()` 함수 자체가 하는 일이 많은 것 같다.\
애초에 651개의 tile cell 전체를 순회하며 업데이트 하는 것 자체가 느리다.

대부분의 경우, 화면 이동 시에 기존 tile map 전체를 다시 로드할 필요는 없다.\
위 이동의 경우도, 프레임당 1-2줄의 행/열이 추가되는 것을 제외하고는, 기존 타일은 재렌더링 없이 그대로 쓸 수 있다.

즉, 이동 시에는 업데이트된 1-2줄만 다시 렌더링하도록 최적화를 하면 된다.

# 3차 최적화: 변경된 부분만 다시 그리기

[변경된 타일맵만 다시 그리는 commit](https://github.com/copyrat90/butano-ldtk/commit/21ab9bcf241313b1709bcfd538d6945a59bd93be)

이동시 새로 보이기 시작한 범위만 다시 그리도록 재구현했다.

![](/assets/img/posts/2025-09-01-butano-ldtk-bg-optimization/row_col.gif)

이번엔 300%는 아꼈다.\
이제 이동 중에 대략 30%의 CPU만 먹는다.\
이 정도면 당장은 충분할 것 같다.

처음으로 배경을 생성할 때는 300%가 넘긴 하는데,\
대부분의 경우 레벨 변경 시 약간의 딜레이는 큰 문제는 아닐 것이다.

*마지막 수정 : {{ page.last_modified_at }}*
