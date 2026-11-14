---
title: "기다리는 동안 다른 일을 하는 법, 코루틴"
description: "Kotlin 코루틴의 기본 개념과 사용 예제, 주의할 점, 그리고 Java 가상 스레드와의 차이를 식당 비유로 설명합니다."
pubDatetime: 2026-11-14T10:00:00+09:00
tags:
  - 개발
  - Kotlin
  - 코루틴
---

서비스를 만들다 보면 코드가 하는 일의 상당 부분이 **기다림**입니다. DB에 물어보고 답을 기다리고, 다른 서버에 요청을 보내고 답을 기다립니다. 문제는 기다리는 동안 프로그램이 아무것도 못 하고 멈춰 있으면 사용자도 같이 기다려야 한다는 점입니다.

이 글에서는 이 기다림을 효율적으로 다루는 Kotlin의 도구, **코루틴**(Coroutine)을 소개합니다. 개발자가 아니어도 읽을 수 있도록 식당 비유를 곁들이고, 마지막에는 Java 21에 들어온 **가상 스레드**와 어떻게 다른지도 비교해 보겠습니다.

## 왜 코루틴을 쓸까?

**1. 왜 코루틴을 쓰나요?**
네트워크·DB 응답처럼 **기다리는 시간이 긴 작업을, 적은 자원으로 동시에 많이** 처리하고 싶어서입니다.

**2. 왜 동시에 많이 처리하는 것이 어려웠나요?**
전통적인 방식에서는 "동시에 하나의 일"이 곧 **스레드 하나**였기 때문입니다. 스레드는 운영체제가 관리하는 일꾼인데, 만들 때마다 메모리가 들고 개수에도 한계가 있습니다. 요청 1만 개를 동시에 기다리려면 스레드 1만 개가 필요한 셈이었습니다.

**3. 그럼 스레드는 왜 기다리는 동안에도 자리를 차지하나요?**
스레드가 응답을 기다리는 동안 **그 스레드는 다른 일을 할 수 없는 채로 붙잡혀 있기** 때문입니다. 일은 하지 않고 자원만 쓰는 낭비입니다.

**4. 그래서 이전에는 어떻게 해결했나요? 거기에는 한계가 없었나요?**
콜백이나 `CompletableFuture`, Rx 같은 **비동기 프로그래밍**으로 "응답이 오면 이 함수를 실행해 줘"라고 맡기는 방식을 썼습니다. 스레드는 아끼지만 코드가 "일이 끝나면 호출될 함수"들로 쪼개져서 **읽기도, 예외 처리도, 디버깅도 어려워졌습니다.** 위에서 아래로 읽히는 평범한 코드의 장점을 잃은 것입니다.

**5. 그 한계를 코루틴은 어떻게 극복했나요?**
코루틴은 **기다리는 지점에서 스레드를 붙잡지 않고 잠시 양보**하는 방식입니다. 개발자는 콜백 없이 위에서 아래로 읽히는 평범한 순차 코드를 쓰는데, 실행될 때는 비동기처럼 스레드를 아낍니다. 코루틴 하나는 스레드 하나보다 훨씬 가벼워서 수만 개를 띄워도 부담이 적습니다.

> 🎯 **사용하는 이유:** 기다림이 많은 작업을 적은 스레드로 동시에 처리해, 서비스가 더 많은 요청을 감당하게 하기 위해서입니다.
>
> 🎯 **극복한 문제:** "스레드를 아끼려면 읽기 어려운 비동기 코드를 써야 한다"는 딜레마를, 순차 코드처럼 쓰면서 비동기처럼 동작하게 만들어 해결했습니다.

## 기본 개념

### 스레드와 코루틴

> 🍳 **비유: 주방 인력 문제**
>
> 식당 주방에 요리사(스레드)가 3명 있다고 해 봅시다. 손님이 파스타를 주문하면 면이 삶아지는 10분 동안 요리사가 냄비 앞에 멍하니 서서 기다린다면 어떨까요? 주문이 3건만 들어와도 주방은 마비됩니다.
>
> 일 잘하는 요리사는 면을 올려놓고 **타이머를 맞춘 뒤 다른 요리를 하러 갑니다.** 타이머가 울리면 돌아와 이어서 합니다. 코루틴이 바로 이 방식입니다. 요리사 3명이 수십 개의 주문을 동시에 돌릴 수 있습니다.

이 글에서는 이 식당 비유를 끝까지 이어서 쓰겠습니다.

아래 그림이 두 방식의 차이입니다. 요리사 한 명이 주문 A, B를 처리할 때, 위쪽은 면이 삶아지는 동안 냄비 앞에서 기다리고, 아래쪽은 타이머를 맞춰 두고 다른 주문을 처리합니다. 같은 일을 하는데 걸리는 시간이 달라집니다.

<figure class="dg">
<svg viewBox="0 0 650 262" role="img" aria-label="요리사 한 명이 주문 A와 B를 처리하는 시간표. 위쪽은 기다리는 동안 요리사가 멈춰 있어 12칸이 걸리고, 아래쪽은 기다리는 동안 다른 주문을 처리해 7칸이 걸립니다.">
<defs><pattern id="dg-hatch" width="8" height="8" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><line class="dg-hatch-line" x1="0" y1="0" x2="0" y2="8"/></pattern></defs>
<text class="dg-title" x="8" y="22">① 스레드 방식: 기다리는 동안 요리사가 붙잡혀 있습니다</text>
<text class="dg-title" x="8" y="104">② 코루틴 방식: 기다리는 동안 다른 주문을 처리합니다</text>
<text class="dg-label" x="8" y="51.0" dominant-baseline="central">요리사</text>
<rect class="dg-a" x="100" y="34" width="40" height="34" rx="5"/><text class="dg-center dg-a-text" x="120.0" y="51.0">A</text>
<rect class="dg-idle" x="140" y="34" width="160" height="34" rx="5"/><text class="dg-center dg-idle-text" x="220.0" y="51.0">멍하니 기다림</text>
<rect class="dg-a" x="300" y="34" width="40" height="34" rx="5"/><text class="dg-center dg-a-text" x="320.0" y="51.0">A</text>
<rect class="dg-b" x="340" y="34" width="40" height="34" rx="5"/><text class="dg-center dg-b-text" x="360.0" y="51.0">B</text>
<rect class="dg-idle" x="380" y="34" width="160" height="34" rx="5"/><text class="dg-center dg-idle-text" x="460.0" y="51.0">멍하니 기다림</text>
<rect class="dg-b" x="540" y="34" width="40" height="34" rx="5"/><text class="dg-center dg-b-text" x="560.0" y="51.0">B</text>
<text class="dg-strong" x="588" y="51.0" dominant-baseline="central" style="fill:var(--dg-wait)">총 12칸</text>
<text class="dg-label" x="8" y="133.0" dominant-baseline="central">요리사</text>
<rect class="dg-a" x="100" y="116" width="40" height="34" rx="5"/><text class="dg-center dg-a-text" x="120.0" y="133.0">A</text>
<rect class="dg-b" x="140" y="116" width="40" height="34" rx="5"/><text class="dg-center dg-b-text" x="160.0" y="133.0">B</text>
<rect class="dg-free" x="180" y="116" width="120" height="34" rx="5"/><text class="dg-center dg-free-text" x="240.0" y="133.0">다른 일 하는 중</text>
<rect class="dg-a" x="300" y="116" width="40" height="34" rx="5"/><text class="dg-center dg-a-text" x="320.0" y="133.0">A</text>
<rect class="dg-b" x="340" y="116" width="40" height="34" rx="5"/><text class="dg-center dg-b-text" x="360.0" y="133.0">B</text>
<text class="dg-strong" x="388" y="133.0" dominant-baseline="central" style="fill:var(--dg-b)">총 7칸</text>
<text class="dg-label" x="8" y="167" dominant-baseline="central">A 냄비</text>
<rect class="dg-timer-a" x="140" y="156" width="160" height="22" rx="5"/><text class="dg-center dg-label" x="220.0" y="167.0">A 면 삶는 중 (타이머)</text>
<text class="dg-label" x="8" y="195" dominant-baseline="central">B 냄비</text>
<rect class="dg-timer-b" x="180" y="184" width="160" height="22" rx="5"/><text class="dg-center dg-label" x="260.0" y="195.0">B 면 삶는 중 (타이머)</text>
<line class="dg-axis" x1="100" y1="232" x2="580" y2="232"/>
<text class="dg-label" x="588" y="232" dominant-baseline="central">시간 →</text>
<text class="dg-label" x="100" y="252">한 칸은 같은 길이의 시간입니다.</text>
<g class="dg-cover"><rect x="100" y="28" width="545" height="46"/><rect x="100" y="110" width="545" height="110"/></g>
<line class="dg-playhead" x1="100" y1="28" x2="100" y2="222"/>
</svg>
<figcaption>그림 1. 같은 요리사 한 명이 주문 두 건을 처리하는 시간표. 색이 칠해진 칸은 요리사가 직접 일하는 시간입니다.</figcaption>
</figure>


| 용어 | 식당 비유 | 설명 |
| --- | --- | --- |
| 스레드 | 요리사 | 실제로 일하는 일꾼. 운영체제가 관리하며 개수가 한정적입니다 |
| 코루틴 | 주문서 한 장 | "이 주문을 처음부터 끝까지 만든다"는 작업 단위. 요리사 사이를 옮겨 다닐 수 있습니다 |
| `suspend` 함수 | "기다리는 동안 자리를 비켜도 되는 작업" | 기다리는 지점에서 스레드를 놓아주는 함수 |
| 디스패처 | 주방장 | 어떤 주문서를 어느 요리사에게 맡길지 정합니다 |
| 스코프 | 한 테이블의 주문 묶음 | 같이 시작하고 같이 끝나며, 문제가 생기면 함께 정리되는 단위 |

### 동작 순서

1. 코루틴이 시작되어 요리사(스레드) 한 명이 일을 맡습니다.
2. 응답을 기다려야 하는 지점(`suspend` 함수)을 만납니다.
3. 코루틴은 **일시 중단**되고, 요리사는 풀려나서 다른 주문서를 처리합니다.
4. 응답이 오면 코루틴이 **재개**되어, 그때 놀고 있는 요리사가 이어서 처리합니다.

중요한 점은 3번에서 스레드가 **블로킹(멈춰 서서 기다림)되지 않는다**는 것입니다. 중단된 코루틴은 어느 스레드도 잡고 있지 않습니다.

### 구조적 동시성: 부모와 자식

코루틴을 쓸 때 가장 중요한 개념을 꼽으라면 **구조적 동시성**입니다. 코루틴은 아무 곳에서나 제멋대로 시작하지 않고, **다른 코루틴(부모) 안에서 시작해 부모에 소속(자식)** 되어야 한다는 원칙입니다. 코루틴들이 나무처럼 부모-자식 구조를 이루고, 이 구조를 따라 완료·실패·취소가 전달됩니다.

> 🍳 **비유: 한 테이블의 코스 주문**
>
> 한 테이블이 코스 요리(샐러드, 파스타, 스테이크)를 주문하면, 주문서 한 묶음이 만들어집니다. 이 묶음에는 세 가지 규칙이 따라붙습니다.
>
> 1. **모든 요리가 나와야 그 테이블의 주문이 끝납니다.** 샐러드만 나왔다고 "주문 완료"라고 하지 않습니다.
> 2. **한 요리에 문제가 생기면 나머지 요리도 접습니다.** 메인 재료가 떨어졌다면 샐러드를 계속 만들 이유가 없습니다.
> 3. **손님이 주문을 취소하면 모든 요리가 함께 취소됩니다.** 켜 놓은 불도 끄고 재료도 정리합니다.

| 규칙 | 코루틴에서는 | 비유 |
| --- | --- | --- |
| 완료 | 부모는 **모든 자식이 끝날 때까지** 끝나지 않습니다 | 모든 요리가 나와야 주문 종료 |
| 실패 | 자식이 실패하면 **부모에게 예외가 전달되고, 형제 코루틴도 취소**됩니다 | 한 요리가 막히면 나머지도 접기 |
| 취소 | 부모가 취소되면 **모든 자식도 취소**됩니다 | 주문 취소 시 전체 요리 중단 |

그림으로 보면 신호가 부모와 자식 사이를 오가는 방향이 분명해집니다.

<figure class="dg">
<svg viewBox="0 0 640 525" role="img" aria-label="구조적 동시성의 세 가지 규칙. 완료: 자식이 모두 끝나야 부모가 끝납니다. 실패: 자식 하나가 실패하면 부모에게 전달되고 나머지 자식도 취소됩니다. 취소: 부모를 취소하면 모든 자식이 취소됩니다.">
<text class="dg-title" x="8" y="20">① 완료: 자식이 모두 끝나야 부모도 끝납니다</text>
<path class="dg-line" d="M186,97 H238 V53 H290"/>
<path class="dg-line" d="M186,97 H238 V97 H290"/>
<path class="dg-line" d="M186,97 H238 V141 H290"/>
<text class="dg-label" x="101.0" y="62" text-anchor="middle">부모 코루틴</text>
<rect class="dg-surface" x="16" y="73" width="170" height="48" rx="8"/>
<text class="dg-center dg-strong" x="101.0" y="97">주문 진행 중</text>
<rect class="dg-surface" x="290" y="36" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="53">샐러드 조리 중…</text>
<rect class="dg-surface" x="290" y="80" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="97">파스타 조리 중…</text>
<rect class="dg-surface" x="290" y="124" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="141">스테이크 조리 중…</text>
<g class="dg-on dg-on-25"><rect class="dg-b" x="290" y="36" width="330" height="34" rx="8"/><text class="dg-center dg-b-text" x="455.0" y="53.0">샐러드 ✓ 완료</text></g>
<g class="dg-on dg-on-45"><rect class="dg-b" x="290" y="80" width="330" height="34" rx="8"/><text class="dg-center dg-b-text" x="455.0" y="97.0">파스타 ✓ 완료</text></g>
<g class="dg-on dg-on-65"><rect class="dg-b" x="290" y="124" width="330" height="34" rx="8"/><text class="dg-center dg-b-text" x="455.0" y="141.0">스테이크 ✓ 완료</text></g>
<g class="dg-on dg-on-85"><rect class="dg-b" x="16" y="73" width="170" height="48" rx="8"/><text class="dg-center dg-b-text" x="101.0" y="97.0">✓ 주문 종료</text></g>
<line class="dg-sep" x1="0" y1="172" x2="640" y2="172"/>
<text class="dg-title" x="8" y="195">② 실패: 한 자식이 실패하면 형제도 함께 취소됩니다</text>
<path class="dg-line" d="M186,272 H238 V228 H290"/>
<path class="dg-line" d="M186,272 H238 V272 H290"/>
<path class="dg-line" d="M186,272 H238 V316 H290"/>
<text class="dg-label" x="101.0" y="237" text-anchor="middle">부모 코루틴</text>
<rect class="dg-surface" x="16" y="248" width="170" height="48" rx="8"/>
<text class="dg-center dg-strong" x="101.0" y="272">주문 진행 중</text>
<rect class="dg-surface" x="290" y="211" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="228">샐러드 조리 중…</text>
<rect class="dg-surface" x="290" y="255" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="272">파스타 조리 중…</text>
<rect class="dg-surface" x="290" y="299" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="316">스테이크 조리 중…</text>
<g class="dg-on dg-on-25"><rect class="dg-bad" x="290" y="255" width="330" height="34" rx="8"/><text class="dg-center dg-bad-text" x="455.0" y="272.0">파스타 ✗ 실패 (재료 소진)</text></g>
<path class="dg-on dg-on-35 dg-line-bad" d="M290,272 H238 V272 H186"/>
<g class="dg-on dg-on-45"><rect class="dg-bad" x="16" y="248" width="170" height="48" rx="8"/><text class="dg-center dg-bad-text" x="101.0" y="272.0">전체 취소 결정</text></g>
<path class="dg-on dg-on-55 dg-line-cancel" d="M186,272 H238 V228 H290"/>
<path class="dg-on dg-on-55 dg-line-cancel" d="M186,272 H238 V316 H290"/>
<g class="dg-on dg-on-65"><rect class="dg-cancel" x="290" y="211" width="330" height="34" rx="8"/><text class="dg-center dg-cancel-text" x="455.0" y="228.0">샐러드 취소됨</text></g>
<g class="dg-on dg-on-65"><rect class="dg-cancel" x="290" y="299" width="330" height="34" rx="8"/><text class="dg-center dg-cancel-text" x="455.0" y="316.0">스테이크 취소됨</text></g>
<line class="dg-sep" x1="0" y1="347" x2="640" y2="347"/>
<text class="dg-title" x="8" y="370">③ 취소: 부모를 취소하면 자식도 모두 취소됩니다</text>
<path class="dg-line" d="M186,447 H238 V403 H290"/>
<path class="dg-line" d="M186,447 H238 V447 H290"/>
<path class="dg-line" d="M186,447 H238 V491 H290"/>
<text class="dg-label" x="101.0" y="412" text-anchor="middle">부모 코루틴</text>
<rect class="dg-surface" x="16" y="423" width="170" height="48" rx="8"/>
<text class="dg-center dg-strong" x="101.0" y="447">주문 진행 중</text>
<rect class="dg-surface" x="290" y="386" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="403">샐러드 조리 중…</text>
<rect class="dg-surface" x="290" y="430" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="447">파스타 조리 중…</text>
<rect class="dg-surface" x="290" y="474" width="330" height="34" rx="8"/>
<text class="dg-center" x="455.0" y="491">스테이크 조리 중…</text>
<g class="dg-on dg-on-25"><rect class="dg-cancel" x="16" y="423" width="170" height="48" rx="8"/><text class="dg-center dg-cancel-text" x="101.0" y="447.0">주문 취소 요청</text></g>
<path class="dg-on dg-on-35 dg-line-cancel" d="M186,447 H238 V403 H290"/>
<path class="dg-on dg-on-35 dg-line-cancel" d="M186,447 H238 V447 H290"/>
<path class="dg-on dg-on-35 dg-line-cancel" d="M186,447 H238 V491 H290"/>
<g class="dg-on dg-on-45"><rect class="dg-cancel" x="290" y="386" width="330" height="34" rx="8"/><text class="dg-center dg-cancel-text" x="455.0" y="403.0">샐러드 취소됨 (정리 완료)</text></g>
<g class="dg-on dg-on-55"><rect class="dg-cancel" x="290" y="430" width="330" height="34" rx="8"/><text class="dg-center dg-cancel-text" x="455.0" y="447.0">파스타 취소됨 (정리 완료)</text></g>
<g class="dg-on dg-on-65"><rect class="dg-cancel" x="290" y="474" width="330" height="34" rx="8"/><text class="dg-center dg-cancel-text" x="455.0" y="491.0">스테이크 취소됨 (정리 완료)</text></g>
</svg>
<figcaption>그림 2. 구조적 동시성의 세 가지 규칙. 점선이 움직이는 방향이 신호가 전달되는 방향입니다.</figcaption>
</figure>

이 원칙이 없으면 어떻게 될까요? 구조 없이 시작한 코루틴(`GlobalScope.launch`)은 **영수증 없이 주방에 들어간 주문서**와 같습니다. 주문한 손님이 가게를 나가도 요리는 계속되고, 실패해도 아무도 모르고, 끝났는지도 알 수 없습니다. 구조적 동시성은 "시작한 작업은 반드시 끝을 책임지는 곳이 있다"는 보장이며, 스레드로 직접 동시성을 다룰 때 흔한 **작업 누수**(아무도 모르게 계속 도는 작업)를 구조적으로 막아 줍니다.

## 다른 방식과 비교

| 방식 | 비유 | 장점 | 한계 |
| --- | --- | --- | --- |
| 스레드 직접 사용 | 주문마다 요리사를 한 명씩 고용 | 코드가 단순함 | 인건비(메모리)가 크고 인원 한계가 있음 |
| 콜백, `CompletableFuture` | 요리사에게 "끝나면 이 쪽지대로 해 줘"라고 쪽지를 남김 | 스레드를 아낌 | 쪽지가 쌓이면 흐름을 따라가기 어려움 |
| 코루틴 | 타이머 맞추고 다른 요리 하러 가기 | 스레드를 아끼면서 코드는 순차적 | `suspend` 개념을 익혀야 함 |

## 사용 예제

이제부터는 개발자를 위한 실전 코드입니다. Kotlin 2.4, kotlinx-coroutines 1.11.0, JDK 21 기준입니다.

### 준비물

`build.gradle.kts`에 의존성을 추가합니다.

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
}
```

### 가장 기본: `launch`

```kotlin
import kotlinx.coroutines.*

fun main() {
    runBlocking {
        launch {
            delay(1000)
            println("주문 접수 완료")
        }
        println("주문을 받는 중...")
    }
}
```

실행 결과입니다.

```text
주문을 받는 중...
주문 접수 완료
```

`launch`는 "이 주문서를 주방에 넘겨 두고 나는 내 할 일을 계속한다"는 뜻입니다. `delay`는 `Thread.sleep`과 달리 스레드를 붙잡지 않고 **코루틴만 잠시 쉬게** 합니다. `runBlocking`은 일반 코드와 코루틴 세계를 이어 주는 다리로, 안의 코루틴이 모두 끝날 때까지 현재 스레드를 막습니다. 그래서 `main`이나 테스트에서만 씁니다.

| 빌더 | 하는 일 | 비유 |
| --- | --- | --- |
| `launch` | 결과값 없이 작업을 시작합니다 (`Job` 반환) | 설거지를 맡겨 두기 |
| `async` | 결과값이 있는 작업을 시작합니다 (`Deferred` 반환) | 요리를 시키고 나중에 접시를 받기 |
| `runBlocking` | 코루틴이 끝날 때까지 현재 스레드를 막습니다 | 가게 문을 열고 마감까지 지키기 |
| `coroutineScope` | 안의 코루틴이 모두 끝나야 반환합니다 | 한 테이블의 주문이 다 나올 때까지 확인하기 |

### 여러 작업을 동시에: `async`와 `await`

```kotlin
suspend fun fetchUser(): String { delay(1000); return "철수" }
suspend fun fetchOrders(): List<String> { delay(1000); return listOf("피자", "콜라") }

fun main() {
    runBlocking {
        val user = async { fetchUser() }
        val orders = async { fetchOrders() }
        println("${user.await()}님의 주문: ${orders.await()}")
    }
}
```

두 작업이 각각 1초씩 걸리지만 동시에 출발하므로, 직접 실행해 보면 전체는 **약 1초**(1046ms)에 끝납니다. 순서대로 호출했다면 2초가 걸렸을 것입니다. `suspend` 키워드는 "이 함수는 기다리는 동안 자리를 비울 수 있다"는 표시이고, `suspend` 함수는 다른 `suspend` 함수나 코루틴 안에서만 부를 수 있습니다.

### 구조적 동시성 직접 확인하기

앞의 세 가지 규칙을 코드로 확인해 보겠습니다. `coroutineScope`는 "테이블 하나의 주문 묶음"을 만드는 함수입니다.

**1. 부모는 자식이 끝날 때까지 기다립니다**

```kotlin
runBlocking {
    coroutineScope {
        launch { delay(300); println("샐러드 완성") }
        launch { delay(600); println("스테이크 완성") }
        println("주문 접수 끝, 요리를 기다리는 중")
    }
    println("코스 요리 모두 완성, 서빙 시작")
}
```

```text
주문 접수 끝, 요리를 기다리는 중
샐러드 완성
스테이크 완성
코스 요리 모두 완성, 서빙 시작
```

`coroutineScope`의 블록 자체는 `launch`를 던져 놓고 바로 끝나지만, `coroutineScope`는 **자식이 모두 끝날 때까지 반환하지 않습니다.** 그래서 `서빙 시작`은 항상 마지막에 출력됩니다.

**2. 자식이 실패하면 형제도 취소되고, 예외는 밖으로 전달됩니다**

```kotlin
runBlocking {
    try {
        coroutineScope {
            launch { delay(5000); println("이 줄은 출력되지 않습니다") }
            launch { delay(100); throw IllegalStateException("재료 소진") }
        }
    } catch (e: IllegalStateException) {
        println("잡았다: ${e.message}")
    }
}
```

실행하면 `잡았다: 재료 소진`만 출력됩니다. 5초짜리 형제 코루틴은 자동으로 취소되었고, 예외는 `coroutineScope` 밖의 `try-catch`에서 잡혔습니다.

**3. 부모를 취소하면 자식도 모두 취소됩니다**

```kotlin
runBlocking {
    val order = launch {
        launch {
            try { delay(5000); println("파스타 완성") }
            finally { println("파스타 주문 취소: 불 끄고 재료 정리") }
        }
        launch {
            try { delay(5000); println("스테이크 완성") }
            finally { println("스테이크 주문 취소: 불 끄고 재료 정리") }
        }
    }
    delay(100)
    order.cancelAndJoin()
    println("테이블 주문 전체 취소됨")
}
```

```text
파스타 주문 취소: 불 끄고 재료 정리
스테이크 주문 취소: 불 끄고 재료 정리
테이블 주문 전체 취소됨
```

부모(`order`) 하나를 취소했을 뿐인데 자식 둘이 모두 멈추고, `finally` 블록에서 **정리 작업까지** 실행됩니다. 파일이나 연결을 닫는 코드를 `finally`에 두면 취소될 때도 안전하게 정리됩니다.

**4. 실패를 전파하고 싶지 않다면: `supervisorScope`**

기본 규칙은 "하나가 실패하면 모두 접는다"이지만, 서로 **독립적인 작업**이라면 하나의 실패가 다른 작업까지 멈추게 하고 싶지 않을 수 있습니다. 그럴 때 `supervisorScope`를 씁니다.

```kotlin
runBlocking {
    supervisorScope {
        launch(CoroutineExceptionHandler { _, e -> println("실패 기록: ${e.message}") }) {
            delay(100)
            throw IllegalStateException("스테이크 재료 소진")
        }
        launch { delay(300); println("샐러드는 정상 완성") }
    }
    println("supervisorScope 종료")
}
```

```text
실패 기록: 스테이크 재료 소진
샐러드는 정상 완성
supervisorScope 종료
```

> 🍳 **비유: 따로 계산하는 별도 주문**
>
> 코스 요리가 아니라 각자 시킨 단품 주문이라면, 한 사람의 요리가 막혔다고 옆 사람의 요리를 접을 필요가 없습니다. `supervisorScope`는 이렇게 **자식끼리 서로의 실패에 영향을 주지 않는** 묶음입니다. 대신 실패한 자식의 예외는 부모로 올라가지 않으므로, 위 예제처럼 `CoroutineExceptionHandler` 등으로 직접 기록하거나 처리해야 합니다.

| 스코프 | 자식이 실패하면 | 어울리는 상황 | 비유 |
| --- | --- | --- | --- |
| `coroutineScope` | 형제도 취소하고 예외를 밖으로 전달 | 전부 성공해야 의미가 있는 작업 (화면 하나를 구성하는 여러 API 호출 등) | 코스 요리 |
| `supervisorScope` | 실패한 자식만 끝나고 나머지는 계속 | 서로 독립적인 작업 (알림 여러 건 발송 등) | 각자 시킨 단품 주문 |

### 블로킹 작업은 전용 디스패처로: `withContext`

파일 읽기나 오래된 JDBC 드라이버처럼 **어쩔 수 없이 스레드를 막는 코드**는 블로킹 작업용 디스패처로 보냅니다.

```kotlin
val result = withContext(Dispatchers.IO) {
    Thread.sleep(100) // 블로킹 호출이라고 가정
    "파일 읽기 완료"
}
```

| 디스패처 | 하는 일 | 비유 |
| --- | --- | --- |
| `Dispatchers.Default` | 계산 위주 작업. CPU 코어 수에 맞춘 스레드를 씁니다 | 손이 바쁜 조리 담당 |
| `Dispatchers.IO` | 블로킹 입출력 작업. 더 많은 스레드를 씁니다 | 냉동창고에 다녀오는 심부름 담당 |
| `Dispatchers.Main` | 화면(UI)을 그리는 작업. Android 등에서 씁니다 | 손님 앞에 서는 홀 담당 |

## 사용 시 주의할 점

### 1. 코루틴 안에서 블로킹 호출을 하지 마세요

```kotlin
launch {
    Thread.sleep(1000) // 나쁨: 요리사가 냄비 앞에서 멍하니 서 있음
    delay(1000)        // 좋음: 타이머를 맞추고 자리를 비움
}
```

`Dispatchers.Default`는 스레드가 CPU 코어 수만큼만 있어서, 블로킹 호출이 몇 개만 겹쳐도 **모든 요리사가 붙잡혀 주방 전체가 멈춥니다.** 블로킹이 불가피하다면 앞에서 본 `withContext(Dispatchers.IO)`로 감싸세요. Dispatcher와 스레드 풀의 구조는 [코루틴 Dispatcher 글](/posts/kotlin-dispatchers/)에서 자세히 다룹니다.

### 2. 취소 예외를 삼키지 마세요

코루틴의 취소는 `CancellationException`이라는 예외가 던져지는 방식으로 전달됩니다. 그런데 `catch (e: Exception)`으로 모든 예외를 잡으면 이 신호까지 같이 잡혀 버립니다.

```kotlin
val job = launch {
    try {
        delay(5000)
    } catch (e: Exception) {            // 취소 신호까지 삼킴
        println("삼킴: ${e::class.simpleName}")
    }
    println("취소됐는데도 계속 실행")
}
delay(100)
job.cancelAndJoin()
```

직접 실행하면 `삼킴: JobCancellationException`에 이어 `취소됐는데도 계속 실행`이 출력됩니다. 주문이 취소됐다는 연락을 받고도 요리를 계속하는 셈입니다. `CancellationException`은 다시 던지거나, 잡지 않도록 구체적인 예외 타입만 잡는 것이 안전합니다.

또 취소는 **협력적**입니다. `delay` 같은 `suspend` 함수를 만나야 취소를 확인하므로, 중단 지점 없이 계산만 도는 반복문은 취소되지 않습니다. 이런 경우 `ensureActive()`나 `isActive`로 직접 확인해야 합니다.

### 3. `launch` 바깥의 `try-catch`로는 예외가 잡히지 않습니다

구조적 동시성 때문에 자식의 예외는 호출한 쪽이 아니라 **부모 코루틴으로 전파**됩니다.

```kotlin
runBlocking {
    try {
        launch { throw IllegalStateException("boom") }  // 예외는 부모로 전파됨
    } catch (e: Exception) {
        println("잡았다")  // 실행되지 않음
    }
}
```

직접 실행하면 `catch`는 실행되지 않고 예외가 그대로 터져 프로그램이 종료됩니다. `launch`는 코루틴을 **시작만 하고 바로 돌아오기** 때문에, `try-catch`가 감싸고 있는 범위에서는 아직 예외가 일어나지 않았습니다. 해결 방법은 두 가지입니다.

- `launch` **안쪽**에서 `try-catch`로 잡습니다.
- 앞에서 본 것처럼 `coroutineScope { ... }` 전체를 `try-catch`로 감쌉니다. 이 경우 자식의 예외가 `coroutineScope` 밖으로 전달되므로 잡을 수 있습니다.

### 4. `GlobalScope`와 부모 없는 코루틴을 피하세요

`GlobalScope.launch`는 어떤 스코프에도 속하지 않아서, 호출한 쪽이 끝나도 계속 돌고, 실패해도 아무도 모릅니다. 영수증 없이 들어간 주문서처럼 추적이 안 됩니다. 항상 `coroutineScope`, `viewModelScope`처럼 **수명이 정해진 스코프**에서 시작해 구조적 동시성의 보호를 받으세요.

### 5. `runBlocking`은 진입점에서만

`runBlocking`은 스레드를 막습니다. 서버의 요청 처리 코드 안에서 쓰면 코루틴을 쓰는 의미가 사라집니다. `main`이나 테스트 같은 경계에서만 사용하세요.

### 6. 코루틴은 CPU 작업을 빠르게 해 주지 않습니다

코루틴이 아끼는 것은 **기다리는 시간의 낭비**이지 계산 시간이 아닙니다. 요리사가 손으로 하는 조리 시간은 그대로이고, 요리사 수(코어 수)보다 많은 계산 작업을 동시에 돌려도 빨라지지 않습니다.

## Java 가상 스레드와의 비교

Java 21부터 **가상 스레드**(Virtual Thread)가 정식으로 들어왔습니다. 목표가 "기다림이 많은 작업을 적은 자원으로 동시에 처리한다"로 같아서 자주 비교됩니다.

> 🍳 **비유: 두 가지 해결책**
>
> 코루틴은 요리사가 **"여기서 타이머를 맞추고 자리를 비우자"고 스스로 의식하며** 일하는 방식입니다. `suspend`가 그 표시입니다.
>
> 가상 스레드는 주방을 개조해서 **주문마다 가상의 요리사 한 명을 붙여 주는 방식**입니다. 요리사는 평소처럼 면 앞에서 "기다리면" 되고, 그러면 주방 시스템이 알아서 그 가상 요리사를 치우고 진짜 요리사를 다른 주문에 보냅니다. 요리사는 자리를 비운다는 사실을 의식하지 않습니다.

| 항목 | Kotlin 코루틴 | Java 가상 스레드 |
| --- | --- | --- |
| 동작 주체 | 라이브러리와 컴파일러 (`suspend`를 변환) | JVM 자체 |
| 코드 작성 방식 | `suspend` 함수를 따로 만들어야 함 | 기존 블로킹 코드를 그대로 사용 |
| 기존 블로킹 라이브러리 | `withContext(Dispatchers.IO)`로 감싸야 함 | 그대로 써도 기다릴 때 스레드가 풀림 |
| 취소와 예외 전파 | 구조적 동시성이 기본 제공됨 | 직접 챙겨야 함 (구조적 동시성 API는 별도) |
| 동시성 도구 | `Flow`, `Channel`, `select` 등 풍부함 | 기존 Java 동시성 도구 (`ExecutorService` 등) |
| 실행 환경 | JVM, Android, JS, Native (멀티플랫폼) | JDK 21 이상의 JVM |
| 학습 비용 | `suspend`, 스코프, 디스패처 학습 필요 | 기존 스레드 지식 그대로 |

### 가상 스레드 코드

같은 Kotlin 프로젝트에서도 가상 스레드를 바로 쓸 수 있습니다.

```kotlin
Executors.newVirtualThreadPerTaskExecutor().use { executor ->
    repeat(3) { i ->
        executor.submit {
            Thread.sleep(100) // 블로킹처럼 보이지만 가상 스레드라 스레드를 붙잡지 않음
            println("작업 $i, 가상 스레드: ${Thread.currentThread().isVirtual}")
        }
    }
}
```

`Thread.sleep`이 평범한 블로킹 호출인데도 문제가 없다는 점이 코루틴과의 가장 큰 차이입니다. 다만 Kotlin에서는 `suspend` 함수 안에서 `Thread.sleep`을 부르면 앞서 본 것처럼 스레드가 막힙니다. 두 모델의 규칙은 서로 다릅니다.

### 함께 쓸 수도 있습니다

코루틴이 돌아갈 스레드를 가상 스레드로 지정할 수도 있습니다.

```kotlin
Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher().use { dispatcher ->
    runBlocking(dispatcher) {
        println("가상 스레드 위에서 실행 중: ${Thread.currentThread().isVirtual}") // true
    }
}
```

`asCoroutineDispatcher()`는 `ExecutorService`를 코루틴 디스패처로 바꿔 줍니다. 블로킹 라이브러리가 많은 곳에서 `Dispatchers.IO` 대신 가상 스레드를 쓰는 식으로 응용할 수 있습니다.

### 그래서 무엇을 쓸까요?

| 상황 | 추천 |
| --- | --- |
| Kotlin 프로젝트에서 새로 동시성 코드를 작성 | 코루틴 (취소·구조적 동시성·`Flow`가 강점) |
| 기존 Java 블로킹 코드(Spring MVC 등)의 처리량을 높이고 싶음 | 가상 스레드 (코드를 거의 안 바꿔도 됨) |
| Android, 멀티플랫폼 | 코루틴 (가상 스레드는 JDK 21 이상 필요) |
| 데이터 스트림을 다루는 작업 | 코루틴 `Flow` |

Spring Boot 프로젝트에서는 설정 한 줄(`spring.threads.virtual.enabled=true`)로 요청 처리에 가상 스레드를 쓸 수 있고, 코루틴은 WebFlux 등에서 `suspend` 컨트롤러로 이어서 쓸 수 있습니다. **둘은 경쟁 관계라기보다 서로 다른 문제를 푸는 도구**에 가깝습니다.

## 알아둘 점

> 💡 가상 스레드는 JDK 버전에 따라 한계가 있습니다. Java 21에서는 `synchronized` 블록 안에서 기다리면 가상 스레드가 진짜 스레드를 붙잡는 **고정(pinning)** 문제가 있었고, 이후 JDK 버전에서 개선되었습니다. 사용하는 JDK 버전의 릴리스 노트를 확인하세요.

- **`suspend`는 호출하는 쪽까지 전파됩니다.** `suspend` 함수는 일반 함수에서 바로 부를 수 없어서, 기존 코드에 도입하면 호출 경로 전체가 영향을 받습니다. 이 점이 가상 스레드와의 가장 큰 차이입니다.
- **디버깅 방법이 다릅니다.** 코루틴은 스레드 사이를 옮겨 다녀서 스택 트레이스가 낯설 수 있습니다. 로그에 코루틴 이름(`CoroutineName`)을 붙이면 추적이 쉽습니다.
- **이 글의 코드는 직접 실행해서 출력을 확인했습니다.** Kotlin 2.4.21, kotlinx-coroutines 1.11.0, JDK 21 환경이며, 시간 측정값은 실행 환경에 따라 달라질 수 있습니다.

## 정리

코루틴은 **"기다리는 동안 스레드를 놓아주는" 가벼운 작업 단위**입니다. 순차 코드처럼 읽히면서도 적은 스레드로 많은 일을 동시에 처리할 수 있습니다.

- **기본 개념:** `suspend`로 기다리는 지점을 표시하고, **구조적 동시성**으로 부모-자식 단위로 완료·실패·취소를 함께 다룹니다.
- **사용 예제:** `launch`, `async`, `coroutineScope`, `withContext`로 대부분의 상황을 다룹니다.
- **주의할 점:** 블로킹 호출, 취소 예외 삼키기, `launch` 바깥의 `try-catch`, `GlobalScope`, `runBlocking` 남용을 피합니다.
- **가상 스레드와의 비교:** 코루틴은 "개발자가 명시하는 중단", 가상 스레드는 "JVM이 알아서 해 주는 중단"입니다. 목적은 비슷하지만 접근이 다르고, 함께 쓸 수도 있습니다.

> 🎯 **사용하는 이유:** 기다림이 많은 작업을 적은 스레드로 동시에 처리해, 같은 서버로 더 많은 요청을 감당하기 위해서입니다.
>
> 🎯 **극복한 문제:** 스레드를 아끼려면 콜백으로 쪼개진 읽기 어려운 코드를 써야 했던 문제를, 순차 코드 모양 그대로 비동기처럼 동작하게 만들어 해결했습니다.

더 나아가고 싶다면 코루틴이 실행되는 스레드를 정하는 [코루틴 Dispatcher 글](/posts/kotlin-dispatchers/), 비동기 데이터 흐름을 다루는 [코루틴 Flow 글](/posts/kotlin-flow/), 코루틴 사이에 데이터를 주고받는 **`Channel`**, 그리고 Spring WebFlux와 코루틴을 결합하는 방법을 살펴보세요.
