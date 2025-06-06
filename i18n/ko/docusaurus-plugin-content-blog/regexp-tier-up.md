---
title: "V8 정규 표현식 개선"
author: "정규 표현식에 대한 의견을 자주 표현하는 Patrick Thier와 Ana Peško"
avatars: 
  - "patrick-thier"
  - "ana-pesko"
date: "2019-10-04 15:24:16"
tags: 
  - internals
  - RegExp
description: "이 블로그 포스트에서는 정규 표현식을 해석의 장점을 활용하고 단점을 완화하는 방법을 설명합니다."
tweet: "1180131710568030208"
---
기본 설정에서 V8은 정규 표현식을 처음 실행할 때 네이티브 코드로 컴파일합니다. [JIT-less V8](/blog/jitless) 작업의 일부로 우리는 정규 표현식을 위한 인터프리터를 도입했습니다. 정규 표현식을 해석하면 더 적은 메모리를 사용하는 장점이 있지만, 성능 상의 단점도 동반됩니다. 이 블로그 게시물에서는 정규 표현식을 해석하는 장점을 활용하고 단점을 완화하는 방법을 설명합니다.

<!--truncate-->
## RegExp를 위한 단계적 전략

우리는 정규 표현식에서 '최상의 조합'을 사용하고 싶습니다. 이를 위해 모든 정규 표현식을 처음에는 바이트코드로 컴파일한 후 해석합니다. 이렇게 하면 많은 메모리를 절약할 수 있고, 새로운 더 빠른 인터프리터와 함께 성능 손실은 수용 가능한 수준으로 유지됩니다. 동일한 패턴의 정규 표현식이 다시 사용되면 이를 '뜨거운' 것으로 간주하여 네이티브 코드로 다시 컴파일합니다. 그 이후에는 가능한 한 빠르게 실행을 계속합니다.

V8의 정규 표현식 코드에는 호출된 메소드와 글로벌인지 비글로벌인지, 빠른 경로인지 느린 경로인지에 따라 많은 다른 경로가 있습니다. 하지만 우리는 단계적 결정이 가능한 한 중앙 집중화되기를 원합니다. 실행 시 특정 값으로 초기화되는 V8의 RegExp 객체에 ticks 필드를 추가했습니다. 이 값은 컴파일러로 단계적 전환하기 전에 정규 표현식이 해석될 횟수를 나타냅니다. 정규 표현식이 해석될 때마다 ticks 필드를 1씩 감소시킵니다. [CodeStubAssembler](/blog/csa)로 작성된 내장 기능에서 모든 정규 표현식에 대해 호출되며 실행 시 ticks 플래그를 확인합니다. ticks가 0에 도달하면 정규 표현식을 네이티브 코드로 다시 컴파일해야 함을 알고 런타임으로 전환하여 이를 처리합니다.

정규 표현식에는 서로 다른 실행 경로가 있을 수 있다는 것을 언급했습니다. 매개변수로 함수와 함께 글로벌 대체를 수행하는 경우 네이티브 코드와 바이트코드 구현이 다릅니다. 네이티브 코드는 모든 일치를 미리 저장할 배열을 기대하며, 바이트코드는 한 번에 하나씩 일치시킵니다. 이러한 이유로 이 사용 사례에서는 항상 네이티브 코드로 빠르게 단계적 전환하기로 결정했습니다.

## RegExp 인터프리터 속도 향상

### 런타임 오버헤드 제거

정규 표현식이 실행될 때 [CodeStubAssembler](/blog/csa)로 작성된 내장이 호출됩니다. 이 내장은 이전에 JSRegExp 객체의 코드 필드에 JIT된 네이티브 코드가 포함되어 직접 실행할 수 있는지 확인한 후 그렇지 않으면 런타임 메소드를 호출하여 RegExp를 컴파일(또는 JIT-less 모드에서 해석)했습니다. JIT-less 모드에서는 모든 정규 표현식 실행이 V8 런타임을 통해 이루어졌는데, 이는 실행 스택에서 JavaScript와 C++ 코드 간의 전환 때문에 비용이 많이 듭니다.

V8 v7.8부터 RegExp 컴파일러가 정규 표현식을 해석하기 위해 바이트코드를 생성할 때 생성된 바이트코드와 함께 RegExp 인터프리터로의 트램펄린을 JSRegExp 객체의 코드 필드에 저장하게 되었습니다. 이를 통해 인터프리터는 이제 런타임을 거치지 않고 내장에서 직접 호출됩니다.

### 새로운 디스패치 방법

이전의 RegExp 인터프리터는 간단한 `switch`-기반 디스패치 방법을 사용했습니다. 이 방법의 주요 단점은 CPU가 다음 실행할 바이트코드를 예측하는 데 어려움을 겪어 많은 브랜치 예측 실패를 초래하여 실행 속도가 느려진다는 것입니다.

우리는 V8 v7.8에서 디스패치 방법을 스레드 코드를 사용하도록 변경했습니다. 이 방법을 사용하면 현재 실행 중인 바이트코드를 기반으로 CPU의 브랜치 예측기가 다음 바이트코드를 예측할 수 있어 예측 실패가 줄어들게 됩니다. 더 자세히는, 디스패치 테이블을 사용하여 각 바이트코드 ID와 바이트코드를 구현하는 핸들러의 주소 간의 매핑을 저장합니다. V8의 인터프리터 [Ignition](/docs/ignition)도 이 접근법을 사용합니다. 하지만 Ignition과 RegExp 인터프리터의 큰 차이는 Ignition의 바이트코드 핸들러는 [CodeStubAssembler](/blog/csa)로 작성된 반면, RegExp 인터프리터 전체는 C++로 작성되어 [computed `goto`s](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)(clang에서도 지원되는 GNU 확장)를 사용하며, 이는 CSA보다 읽기 및 유지 관리가 더 쉽습니다. computed gotos를 지원하지 않는 컴파일러의 경우 이전의 `switch`-기반 디스패치 방법으로 대체됩니다.

### 바이트코드 피홀 최적화

바이트코드 피폴 최적화에 대해 이야기하기 전에, 동기 부여의 한 예를 살펴보겠습니다.

```js
const re = /[^_]*/;
const str = 'a0b*c_ef';
re.exec(str);
// → 매칭 결과 'a0b*c'
```

이 간단한 패턴에 대해 정규 표현식 컴파일러는 모든 문자에 대해 실행되는 3개의 바이트코드를 생성합니다. 높은 수준에서 이는 다음과 같습니다:

1. 현재 문자를 로드합니다.
2. 문자가 `'_'`와 같은지 확인합니다.
3. 그렇지 않으면, 대상 문자열에서 현재 위치를 이동하고 `1번`으로 돌아갑니다.

우리의 대상 문자열에 대해, 우리는 비일치 문자를 찾을 때까지 17개의 바이트코드를 해석합니다. 피폴 최적화의 아이디어는 여러 바이트코드의 기능을 결합한 새롭게 최적화된 바이트코드로 바이트코드 시퀀스를 대체하는 것입니다. 우리의 예에서는 `goto`로 생성된 암시적 루프를 새로운 바이트코드에서 명시적으로 처리할 수 있습니다. 따라서 단일 바이트코드는 모든 매칭 문자를 처리하여 16번의 디스패치를 절약합니다.

이 예는 만들어진 것이지만, 여기에서 설명된 바이트코드 시퀀스는 실제 웹사이트에서 자주 발생합니다. 우리는 [실제 웹사이트](/blog/real-world-performance)를 분석하고 우리가 발견한 가장 빈번한 바이트코드 시퀀스에 대해 새로운 최적화된 바이트코드를 생성했습니다.

## 결과

![그림 1: 다양한 티어 업 값에 따른 메모리 절약 효과](/_img/regexp-tier-up/results-memory.svg)

그림 1은 Facebook, Reddit, Twitter 및 Tumblr 탐색 사례에 대한 다양한 티어 업 전략이 메모리에 미치는 영향을 보여줍니다. 기본값은 JIT 코드 크기이며, 이후에는 우리가 사용하는 정규표현식 코드 크기(티어 업을 하지 않을 경우 바이트코드 크기, 티어 업을 할 경우 네이티브 코드 크기)가 1, 10, 100으로 초기화된 틱 값에 대한 크기입니다. 마지막으로, 모든 정규 표현식을 해석하는 경우 정규표현식 코드 크기를 보여줍니다. 우리는 이러한 결과와 다른 벤치마크를 사용하여 티어 업을 활성화하고 틱 값을 1로 초기화하기로 결정했습니다. 즉, 정규 표현식을 한 번 해석한 후 티어 업합니다.

이 티어 업 전략을 적용하면 실제 사이트에서 V8의 힙 코드 크기를 4%에서 7%까지, V8의 실질적인 크기를 1%에서 2%까지 줄였습니다.

![그림 2: 정규표현식 성능 비교](/_img/regexp-tier-up/results-speed.svg)

그림 2는 이 블로그 게시물[^strict-bounds]에서 설명된 모든 개선사항에 대해 RexBench 벤치마크 스위트에서 정규표현식 인터프리터의 성능에 미치는 영향을 보여줍니다. 참고로, JIT 컴파일된 정규표현식의 성능도 표시됩니다(네이티브).

[^strict-bounds]: 여기에서 보여지는 결과는 [V8 v7.8 릴리즈 노트](/blog/v8-release-78#faster-regexp-match-failures)에서 이미 설명된 정규표현식의 개선사항을 포함합니다.

새로운 인터프리터는 이전보다 최대 2배 빠르고 평균적으로 약 1.45배 빠릅니다. 우리는 대부분의 벤치마크에서 JIT된 정규표현식의 성능에 매우 근접합니다. 유일한 예외는 Regex DNA입니다. 이 벤치마크에서 인터프리터 정규표현식이 JIT된 정규표현식보다 훨씬 느린 이유는 사용된 긴 대상 문자열(~300,000 문자) 때문입니다. 우리가 디스패치 오버헤드를 최소화했음에도 불구하고, 1,000자를 초과하는 문자열에서는 오버헤드가 누적되어 느린 실행이 발생합니다. 긴 문자열에서 인터프리터가 훨씬 느리므로 이러한 문자열에 대해 적극적으로 티어 업을 수행하는 휴리스틱을 추가했습니다.

## 결론

V8 v7.9(Chrome 79)부터, 우리는 정규표현식을 즉시 컴파일하는 대신 티어 업합니다. 따라서 이전에 JIT이 없는 V8에서만 사용되었던 인터프리터가 이제 어디에서나 사용됩니다. 결과적으로 우리는 메모리를 절약합니다. 우리는 이를 실행 가능한 것으로 만들기 위해 인터프리터 속도를 높였습니다. 하지만 이것은 끝이 아니며, 앞으로도 더 많은 개선 사항이 기대됩니다.

이번 기회를 이용해 우리의 인턴십 동안 지원해준 V8 팀의 모든 분들께 감사드리고 싶습니다. 정말 멋진 경험이었습니다!
