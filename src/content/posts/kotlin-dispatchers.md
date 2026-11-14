---
title: "누가 어느 요리사에게 일을 맡길까, 코루틴 Dispatcher"
description: "코루틴 Dispatcher의 종류와 역할, Default와 IO가 스레드 풀을 나눠 쓰는 방식, 사용할 때 주의할 점을 식당 비유와 애니메이션으로 설명합니다."
pubDatetime: 2026-11-14T12:00:00+09:00
tags:
  - 개발
  - Kotlin
  - 코루틴
---

[코루틴 글](/posts/kotlin-coroutines/)에서 `Dispatchers.IO`로 블로킹 작업을 감싸라는 이야기를 했습니다. 하지만 "왜 IO여야 하고, Default는 안 되는지", "둘은 정확히 무엇이 다른지"는 짧게만 넘어갔습니다. 실무에서 코루틴이 느려지거나 멈추는 문제의 상당수가 바로 이 지점에서 나옵니다.

이 글에서는 코루틴이 **어느 스레드에서 실행될지를 정하는 도구, Dispatcher**를 자세히 다룹니다. 코루틴 글의 식당 비유를 그대로 이어서 설명하니, 먼저 읽고 오시면 더 편합니다. Flow도 결국 Dispatcher 위에서 돌기 때문에, 앞서 쓴 [Flow 글](/posts/kotlin-flow/)의 `flowOn`을 이해하는 데도 도움이 됩니다.

## 왜 Dispatcher를 쓸까?

**1. 왜 Dispatcher를 쓰나요?**
코루틴이 **어떤 스레드에서 실행될지** 코드에서 직접 정하기 위해서입니다.

**2. 왜 실행될 스레드를 정해야 하나요?**
작업의 성격에 따라 알맞은 스레드의 종류와 개수가 다르기 때문입니다. 계산이 많은 작업은 CPU 코어 수만큼의 스레드면 충분합니다. 반대로 파일이나 네트워크를 기다리는 작업은 대부분의 시간을 놀고 있어서 스레드가 더 많이 필요합니다. 화면을 그리는 작업은 반드시 정해진 하나의 스레드에서만 해야 합니다.

**3. 그럼 코루틴이 없던 시절에는 어떻게 했나요?**
`ExecutorService`로 스레드 풀을 직접 만들어 작업을 제출했습니다. 계산용 풀, 입출력용 풀, 화면 갱신용 핸들러를 개발자가 각각 만들고 관리했습니다.

**4. 그 방식의 한계는 무엇이었나요?**
풀의 크기를 개발자가 감으로 정해야 했고, 다 쓴 풀을 `shutdown` 하지 않으면 스레드가 새어 나갔습니다. 또 스레드를 옮길 때마다 "이 작업을 저 풀에 제출하고, 끝나면 화면 스레드로 다시 넘기는" 코드를 직접 짜야 해서 **비즈니스 로직 사이에 스레드 전환 코드가 뒤섞였습니다.**

**5. Dispatcher는 이 한계를 어떻게 극복했나요?**
라이브러리가 **미리 튜닝된 공유 스레드 풀**(`Default`, `IO`)을 제공하고, 개발자는 `withContext(Dispatchers.IO) { ... }` **한 줄로 실행 위치만 선언**합니다. 풀의 생성과 관리는 코루틴 라이브러리가 맡고, 작업이 끝나면 원래 위치로 자동으로 돌아옵니다.

> 🎯 **사용하는 이유:** 작업의 성격(계산, 대기, 화면)에 맞는 스레드에서 코루틴을 실행해, 적은 스레드로도 서비스가 멈추지 않게 하기 위해서입니다.
>
> 🎯 **극복한 문제:** 스레드 풀을 직접 만들고 크기를 정하고 정리해야 했던 부담, 그리고 스레드 전환 코드가 로직에 섞이던 문제를 "실행 위치를 한 줄로 선언"하는 방식으로 해결했습니다.

## 기본 개념

> 🍳 **비유: 주방장과 주방 인원**
>
> 코루틴이 주문서라면, 주문서를 받아 "이 주문은 누구에게 맡길지" 정해 주는 사람이 주방장입니다. 그리고 주방장이 쓸 수 있는 요리사들(스레드)의 모임이 **스레드 풀**입니다.
>
> 주방장은 한 명이 아니라 역할별로 있습니다. 불 앞에서 손이 바쁜 조리 담당, 냉동창고를 오가는 심부름 담당, 손님 앞에 서는 홀 담당처럼요.

| 용어 | 식당 비유 | 설명 |
| --- | --- | --- |
| Dispatcher | 주방장 | 코루틴을 어느 스레드에서 실행할지 정합니다 |
| 스레드 풀 | 주방 인원 전체 | 코루틴이 올라탈 스레드들의 모임입니다 |
| `withContext` | 주문서를 다른 담당에게 넘기기 | 실행 위치를 바꿨다가 끝나면 돌아옵니다 |

### Dispatcher의 종류와 역할

| Dispatcher | 하는 일 | 스레드 수 | 비유 |
| --- | --- | --- | --- |
| `Dispatchers.Default` | 계산 위주 작업 (정렬, 변환, 압축 등) | CPU 코어 수만큼 | 손이 바쁜 조리 담당 |
| `Dispatchers.IO` | 블로킹 입출력 (파일, DB, 네트워크) | 기본 최대 64개 (코어 수가 더 많으면 코어 수) | 냉동창고에 다녀오는 심부름 담당 |
| `Dispatchers.Main` | 화면(UI) 갱신 | 화면 전용 스레드 1개 | 손님 앞에 서는 홀 담당 |
| `Dispatchers.Unconfined` | 호출한 스레드에서 바로 시작하고, 재개는 아무 스레드나 | 정해지지 않음 | 주방장 없이 눈앞의 사람이 일단 시작 |
| `limitedParallelism(n)` | 풀 안에서 **동시에 쓸 수 있는 스레드 수를 n개로 제한**한 Dispatcher | 최대 n개 | 이 일에는 요리사 n명까지만 배정 |
| 직접 만든 풀 | `ExecutorService`를 Dispatcher로 변환 | 만든 대로 | 전담 팀을 따로 고용 |

핵심은 두 가지입니다.

- `Default`는 **일하는 스레드**를 위한 곳이고, `IO`는 **기다리는 스레드**를 위한 곳입니다.
- `Main`은 안드로이드나 데스크톱 UI 환경에서만 쓰며, 서버(Spring 등)에서는 쓰지 않습니다.

## 스레드 풀 점유 영역

Default와 IO가 서로 다른 스레드 풀이라고 생각하기 쉽지만 **실제로는 하나의 스레드 풀을 나눠 씁니다.** 그 풀 안에서 코어 수만큼의 영역은 Default가, 필요할 때 늘어나는 여분 영역은 IO가 주로 사용합니다.

> 🍳 **비유: 하나의 주방, 두 가지 근무 구역**
>
> 식당 주방은 하나이고 요리사도 같은 직원 풀에서 나옵니다. 다만 조리대(코어 수만큼)에서 일하는 사람은 정해져 있고, 냉동창고 심부름은 일손이 모자라면 직원을 더 불러서(최대 64명까지) 시킵니다. 심부름 직원은 대부분 창고 앞에서 기다리는 시간이 길기 때문에 인원이 많아도 부담이 적습니다.

아래 그림은 Default 영역에서 블로킹 작업을 하면 벌어지는 일과, IO로 보냈을 때의 차이입니다. (그림에서는 이해를 돕기 위해 슬롯을 4개씩만 그렸습니다.)

<figure class="dg">
<svg viewBox="0 0 640 400" role="img" aria-label="하나의 스레드 풀 안에서 Default 영역과 IO 여분 영역을 보여 주는 그림. 위쪽은 블로킹 작업이 Default 영역을 모두 차지해 새 요청이 기다리고, 아래쪽은 블로킹 작업을 IO 영역으로 보내 새 요청이 바로 처리됩니다.">
<text class="dg-title" x="8" y="20">① Default에서 블로킹하면: 모든 스레드가 붙잡힙니다</text>
<rect class="dg-sep" x="8" y="34" width="624" height="92" rx="8" fill="none"/>
<text class="dg-label" x="16" y="50">Default 영역 (코어 수만큼)</text>
<text class="dg-label" x="340" y="50">IO 여분 영역 (필요할 때 늘어남)</text>
<g class=""><rect class="dg-surface" x="16" y="62" width="68" height="38" rx="6"/><text class="dg-center" x="50.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="340" y="62" width="60" height="38" rx="6"/><text class="dg-center" x="370.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="92" y="62" width="68" height="38" rx="6"/><text class="dg-center" x="126.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="408" y="62" width="60" height="38" rx="6"/><text class="dg-center" x="438.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="168" y="62" width="68" height="38" rx="6"/><text class="dg-center" x="202.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="476" y="62" width="60" height="38" rx="6"/><text class="dg-center" x="506.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="244" y="62" width="68" height="38" rx="6"/><text class="dg-center" x="278.0" y="81.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="544" y="62" width="60" height="38" rx="6"/><text class="dg-center" x="574.0" y="81.0">스레드</text></g>
<g class="dg-on dg-on-45"><rect class="dg-surface" x="16" y="138" width="270" height="38" rx="6"/><text class="dg-center" x="151" y="157">새 요청: 화면 응답 처리</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="16" y="62" width="68" height="38" rx="6"/><text class="dg-center dg-bad-text" x="50.0" y="81.0">블로킹</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="92" y="62" width="68" height="38" rx="6"/><text class="dg-center dg-bad-text" x="126.0" y="81.0">블로킹</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="168" y="62" width="68" height="38" rx="6"/><text class="dg-center dg-bad-text" x="202.0" y="81.0">블로킹</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="244" y="62" width="68" height="38" rx="6"/><text class="dg-center dg-bad-text" x="278.0" y="81.0">블로킹</text></g>
<g class="dg-on dg-on-65"><rect class="dg-bad" x="16" y="138" width="270" height="38" rx="6"/><text class="dg-center dg-bad-text" x="151.0" y="157.0">✗ 빈 스레드가 없어 기다림</text></g>
<line class="dg-sep" x1="0" y1="198" x2="640" y2="198"/>
<text class="dg-title" x="8" y="220">② IO로 보내면: 블로킹은 여분 영역이 맡습니다</text>
<rect class="dg-sep" x="8" y="234" width="624" height="92" rx="8" fill="none"/>
<text class="dg-label" x="16" y="250">Default 영역 (코어 수만큼)</text>
<text class="dg-label" x="340" y="250">IO 여분 영역 (필요할 때 늘어남)</text>
<g class=""><rect class="dg-surface" x="16" y="262" width="68" height="38" rx="6"/><text class="dg-center" x="50.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="340" y="262" width="60" height="38" rx="6"/><text class="dg-center" x="370.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="92" y="262" width="68" height="38" rx="6"/><text class="dg-center" x="126.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="408" y="262" width="60" height="38" rx="6"/><text class="dg-center" x="438.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="168" y="262" width="68" height="38" rx="6"/><text class="dg-center" x="202.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="476" y="262" width="60" height="38" rx="6"/><text class="dg-center" x="506.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="244" y="262" width="68" height="38" rx="6"/><text class="dg-center" x="278.0" y="281.0">스레드</text></g>
<g class=""><rect class="dg-surface" x="544" y="262" width="60" height="38" rx="6"/><text class="dg-center" x="574.0" y="281.0">스레드</text></g>
<g class="dg-on dg-on-45"><rect class="dg-surface" x="16" y="338" width="270" height="38" rx="6"/><text class="dg-center" x="151" y="357">새 요청: 화면 응답 처리</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="340" y="262" width="60" height="38" rx="6"/><text class="dg-center dg-bad-text" x="370.0" y="281.0">블로킹</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="408" y="262" width="60" height="38" rx="6"/><text class="dg-center dg-bad-text" x="438.0" y="281.0">블로킹</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="476" y="262" width="60" height="38" rx="6"/><text class="dg-center dg-bad-text" x="506.0" y="281.0">블로킹</text></g>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="544" y="262" width="60" height="38" rx="6"/><text class="dg-center dg-bad-text" x="574.0" y="281.0">블로킹</text></g>
<g class="dg-on dg-on-65"><rect class="dg-b" x="16" y="262" width="68" height="38" rx="6"/><text class="dg-center dg-b-text" x="50.0" y="281.0">새 요청</text></g>
<g class="dg-on dg-on-65"><rect class="dg-b" x="16" y="338" width="270" height="38" rx="6"/><text class="dg-center dg-b-text" x="151.0" y="357.0">✓ 바로 처리</text></g>
</svg>
<figcaption>그림 3. Default와 IO는 하나의 스레드 풀을 나눠 씁니다. 빨간 칸은 블로킹 작업이 스레드를 붙잡고 있는 모습입니다.</figcaption>
</figure>

직접 실험으로 확인한 결과입니다. 8코어 환경에서 각각 **블로킹 작업(`Thread.sleep`) 200개**를 시킨 결과입니다.

| 실험 | 결과 |
| --- | --- |
| `Default`에서 200개 실행 | 사용한 스레드 **8개** (코어 수) |
| `IO`에서 200개 실행 | 사용한 스레드 **64개** |
| `Default`에서 `IO`로 전환 (`withContext`) | **같은 스레드**(`DefaultDispatcher-worker-16`)에서 계속 실행 |

세 번째 결과가 중요합니다. `Default`에서 `IO`로 `withContext`를 해도 **스레드가 바뀌지 않을 수 있습니다.** 같은 풀 안에서의 이동이라 비용이 매우 작고, 그래서 필요할 때 `withContext`를 아끼지 않고 써도 됩니다.

### `limitedParallelism`: 풀 안에서 쓸 수 있는 몫 정하기

`limitedParallelism(n)`은 새 스레드를 만드는 것이 아니라, 기존 풀 위에 **"동시에 n개까지만"이라는 한도를 건 Dispatcher**를 만듭니다.

| 실험 (8코어 환경) | 걸린 시간 |
| --- | --- |
| `IO`에서 500ms짜리 블로킹 작업 128개 | 약 1013ms (64개씩 두 번) |
| `IO.limitedParallelism(128)`에서 같은 작업 | 약 521ms (128개 한 번에) |
| `Default.limitedParallelism(1)`에서 200ms짜리 작업 3개 | 약 616ms (하나씩 순서대로) |

`IO`는 기본 64개 한도가 있지만, `Dispatchers.IO.limitedParallelism(128)`처럼 쓰면 **그 기본 한도와 별개의 한도**를 얻습니다. 특정 외부 서비스 호출에만 동시 호출 수를 제한하거나(예: 최대 10개), 반대로 느린 블로킹 라이브러리를 더 많이 동시에 돌리고 싶을 때 씁니다.

## 사용 예제

이제부터는 개발자를 위한 실전 코드입니다. Kotlin 2.4, kotlinx-coroutines 1.11.0, JDK 21 기준입니다. 의존성은 [코루틴 글](/posts/kotlin-coroutines/)의 준비물 그대로입니다.

### 기본: `withContext`로 실행 위치 선언

```kotlin
suspend fun readFile(): String = withContext(Dispatchers.IO) {
    Thread.sleep(100)   // 블로킹 입출력이라고 가정
    "파일 읽기 완료"
}

suspend fun summarize(text: String): Int = withContext(Dispatchers.Default) {
    text.length         // 계산 작업이라고 가정
}

fun main() = runBlocking {
    val text = readFile()          // IO 담당이 처리하고 돌아옴
    println(summarize(text))       // Default 담당이 처리하고 돌아옴
}
```

함수 안에서 `withContext`로 자기 일에 맞는 Dispatcher를 선언해 두면, **호출하는 쪽은 어떤 스레드에서 불러도 안전합니다.** 이런 함수를 흔히 "main-safe 하다"고 부릅니다.

### 자식은 부모의 Dispatcher를 물려받습니다

```kotlin
runBlocking {
    launch(Dispatchers.Default) {
        println("부모=${Thread.currentThread().name}")
        launch { println("자식 상속=${Thread.currentThread().name}") }                    // 부모를 따라 Default 풀
        launch(Dispatchers.IO) { println("자식 IO=${Thread.currentThread().name}") }      // 직접 지정하면 그쪽으로
    }.join()
}
```

`launch`에 Dispatcher를 지정하지 않으면 **부모 코루틴의 Dispatcher를 그대로 씁니다.** `runBlocking` 안의 `launch`가 별도 지정 없이 `main` 스레드에서 도는 이유도 이것입니다. 구조적 동시성의 부모-자식 관계가 실행 위치에도 이어지는 셈입니다.

### 동시에 한 개만: `limitedParallelism(1)`

```kotlin
val single = Dispatchers.Default.limitedParallelism(1)

coroutineScope {
    repeat(3) { launch(single) { /* 한 번에 하나씩 순서대로 실행 */ } }
}
```

공유 자원(파일 하나에 쓰기 등)을 여러 코루틴이 동시에 건드리지 않게 하고 싶을 때 씁니다.

### 전담 팀을 따로: 직접 만든 스레드 풀

```kotlin
Executors.newFixedThreadPool(2).asCoroutineDispatcher().use { dispatcher ->
    withContext(dispatcher) {
        println(Thread.currentThread().name)   // pool-1-thread-1
    }
}
```

특정 라이브러리의 호출을 전용 스레드로만 보내야 할 때처럼, 공유 풀과 분리하고 싶을 때 씁니다. 가상 스레드를 Dispatcher로 쓰는 방법은 [코루틴 글](/posts/kotlin-coroutines/)의 가상 스레드 비교에서 다뤘습니다.

> 💡 직접 만든 풀은 `use`나 `close()`로 반드시 닫아야 합니다. 닫지 않으면 스레드가 계속 남아 프로그램이 종료되지 않을 수 있습니다.

## 사용 시 주의할 점

### 1. Default에서 블로킹 호출을 하지 마세요

Default의 스레드는 코어 수만큼밖에 없습니다. 블로킹 호출 몇 개가 자리를 모두 차지하면 **다른 코루틴이 실행될 스레드가 없어서 멈춥니다.**

```kotlin
coroutineScope {
    repeat(cores) { launch(Dispatchers.Default) { Thread.sleep(1000) } }   // 코어 수만큼 자리를 모두 차지
    delay(100)
    val t = measureTimeMillis { withContext(Dispatchers.Default) { } }      // 빈 작업 하나가 시작되기까지의 시간
    println("새 작업 시작까지 ${t}ms")
}
```

8코어 환경에서 직접 실행하면 **새 작업이 시작되기까지 약 895ms**가 걸렸습니다. 빈 작업인데도 1초 가까이 기다린 것입니다. 같은 상황에서 블로킹 작업을 `withContext(Dispatchers.IO)`로 감싸 보내면 새 작업은 **1ms** 만에 시작됐습니다.

### 2. IO에서 계산 작업을 몰아서 하지 마세요

IO는 코어 수보다 훨씬 많은 스레드를 허용합니다. 여기서 CPU를 많이 쓰는 계산을 돌리면 코어 하나를 여러 스레드가 나눠 쓰게 되어 이득이 없고, 정작 필요한 입출력 작업이 느려집니다. 계산은 Default에서 하세요.

### 3. `suspend` 함수는 스스로 실행 위치를 책임지게 하세요

"이 함수는 IO에서 호출해야 한다"는 규칙을 호출하는 쪽에 떠넘기면 실수하기 쉽습니다. 블로킹 코드를 가진 `suspend` 함수는 **함수 안에서 `withContext`로 감싸서** 어디서 불러도 안전하게 만드는 것이 좋습니다. 앞의 `readFile()`이 그 예입니다.

### 4. 자식은 부모의 Dispatcher를 따라갑니다

`runBlocking` 안에서 시작한 `launch`는 별도 지정이 없으면 `main` 스레드에서 실행됩니다. 이 안에서 블로킹 작업을 하면 바로 `main`이 막힙니다. 어느 Dispatcher에서 시작하는지 항상 의식하고, 필요하면 `launch(Dispatchers.Default)`처럼 명시하세요.

### 5. `Unconfined`는 특별한 이유가 없다면 쓰지 마세요

`Unconfined`는 호출한 스레드에서 바로 시작하고, `delay` 같은 중단 지점 뒤에는 **재개시킨 쪽의 스레드에서 이어서 실행됩니다.**

```kotlin
launch(Dispatchers.Unconfined) {
    println("시작=${Thread.currentThread().name}")   // main
    delay(50)
    println("재개=${Thread.currentThread().name}")   // kotlinx.coroutines.DefaultExecutor
}
```

직접 실행하면 시작은 `main`이었지만 재개는 `kotlinx.coroutines.DefaultExecutor`라는 전혀 다른 스레드였습니다. 어디서 실행될지 예측하기 어려우므로 일반 코드에는 맞지 않습니다. 테스트처럼 특수한 상황에서만 씁니다.

### 6. `Dispatchers.Main`은 의존성이 필요합니다

서버나 일반 JVM 프로젝트에서 `Dispatchers.Main`을 쓰면 다음 예외가 납니다.

```text
IllegalStateException: Module with the Main dispatcher is missing. Add dependency providing the Main dispatcher...
```

`Main`은 Android(`kotlinx-coroutines-android`) 등 UI 환경별 모듈이 있어야 동작합니다.

### 7. `limitedParallelism(1)`은 락이 아닙니다

`limitedParallelism(1)`은 "동시에 실행되는 코루틴을 하나로 제한"할 뿐, **중단 지점(`delay` 등)에서는 다른 코루틴이 끼어들 수 있습니다.** 여러 단계에 걸친 작업을 통째로 보호하는 용도로는 적합하지 않으니, 그럴 때는 `Mutex` 같은 도구를 쓰세요.

## 알아둘 점

> 💡 Dispatcher를 고를 때의 기준은 간단합니다. **"이 코드가 CPU를 쓰는가(Default), 기다리는가(IO), 화면인가(Main)?"** 그리고 기다리는 코드는 `suspend` 함수(`delay`, 코루틴을 지원하는 라이브러리)라면 Dispatcher와 상관없이 스레드를 놓아주므로 걱정하지 않아도 됩니다. 문제는 **스레드를 붙잡는 블로킹 호출**입니다.

- **스레드 수는 환경에 따라 다릅니다.** Default는 코어 수에 따라 달라지고, IO의 기본 64개는 코어 수가 64보다 많은 환경에서는 달라질 수 있습니다.
- **가상 스레드와 함께 쓰는 방법도 있습니다.** 블로킹 라이브러리가 많다면 `IO` 대신 가상 스레드 기반 Dispatcher를 쓰는 것도 선택지입니다. 자세한 내용은 [코루틴 글](/posts/kotlin-coroutines/)을 참고하세요.
- **이 글의 코드는 직접 실행해서 결과를 확인했습니다.** Kotlin 2.4.21, kotlinx-coroutines 1.11.0, JDK 21, 8코어 macOS 환경이며, 시간 측정값은 실행 환경에 따라 달라질 수 있습니다. 글에 실은 코드 블록을 그대로 복사해 컴파일해 본 것은 아닙니다.
- **IO에서 계산 작업을 하면 느려진다는 설명(주의할 점 2번)은 실험하지 않고 구조에서 설명한 내용입니다.**

## 정리

Dispatcher는 **코루틴이 어느 스레드에서 실행될지를 정하는 주방장**입니다. Default와 IO는 하나의 스레드 풀을 나눠 쓰며, 계산은 코어 수만큼의 Default 영역에서, 기다림은 여분이 늘어나는 IO 영역에서 처리합니다.

- **종류와 역할:** `Default`는 계산, `IO`는 블로킹 입출력, `Main`은 화면입니다. `limitedParallelism`으로 풀 안에서의 몫을 제한할 수 있습니다.
- **스레드 풀 점유 영역:** Default와 IO는 같은 풀 위의 다른 영역이라 `withContext`로 옮겨도 스레드가 바뀌지 않을 수 있고, 비용이 작습니다.
- **주의할 점:** Default에서의 블로킹 호출은 풀 전체를 멈추게 합니다. `suspend` 함수는 `withContext`로 스스로 실행 위치를 책임지게 하세요.

> 🎯 **사용하는 이유:** 작업의 성격(계산, 대기, 화면)에 맞는 스레드에서 코루틴을 실행해, 적은 스레드로도 서비스가 멈추지 않게 하기 위해서입니다.
>
> 🎯 **극복한 문제:** 스레드 풀을 직접 만들고 크기를 정하고 정리해야 했던 부담, 그리고 스레드 전환 코드가 로직에 섞이던 문제를 "실행 위치를 한 줄로 선언"하는 방식으로 해결했습니다.

코루틴이 처음이라면 [코루틴 글](/posts/kotlin-coroutines/)을, 값의 흐름에서 실행 위치를 바꾸는 `flowOn`이 궁금하다면 [Flow 글](/posts/kotlin-flow/)을 이어서 읽어 보세요.
