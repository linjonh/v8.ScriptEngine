---
title: "디자인 리뷰 가이드라인"
description: "이 문서는 V8 프로젝트의 디자인 리뷰 가이드라인을 설명합니다."
---
적용 가능한 경우 아래의 가이드라인을 반드시 따르십시오.

V8의 디자인 리뷰를 공식화하는 여러 가지 이유가 있습니다:

1. 개별 기여자(IC)에게 의사결정자가 누구인지 분명히 하고 기술적 의견 차이로 인해 프로젝트가 진행되지 않을 경우 앞으로 나아가는 경로를 강조
1. 간단한 디자인 토론을 할 수 있는 포럼 생성
1. V8 기술 리드(TL)이 모든 중대한 변경 사항을 인지하고 TL 레이어에서 의견을 제시할 기회를 제공
1. 전 세계의 모든 V8 기여자들의 참여 증대

## 개요

![V8의 디자인 리뷰 가이드라인 한눈에 보기](/_img/docs/design-review-guidelines/design-review-guidelines.svg)

중요:

1. 선한 의도를 가정하십시오.
1. 친절하고 문명적으로 행동하십시오.
1. 실용적이어야 합니다.

제안된 솔루션은 다음과 같은 가정/기둥에 기반을 둡니다:

1. 제안된 워크플로우는 개인 기여자(IC)가 주도합니다. 그들은 프로세스를 조정하는 사람입니다.
1. 그들의 안내 TL은 영토를 탐색하고 적합한 LGTM 제공자를 찾을 수 있도록 돕는 임무를 맡습니다.
1. 기능이 논란이 없다면 거의 아무런 부담도 추가되지 않아야 합니다.
1. 논란이 많은 경우, 기능은 V8 Eng Review Owners 회의를 통해 '상승'하여 추가 단계가 결정될 수 있습니다.

## 역할

### 개인 기여자(IC)

LGTM: 해당 없음
이 사람은 기능의 작성자이자 디자인 문서의 작성자입니다.

### IC의 기술 리드(TL)

LGTM: 반드시 필요
이 사람은 특정 프로젝트나 구성 요소의 TL입니다. 이는 여러분의 기능이 영향을 미칠 주요 구성 요소의 소유자가 될 가능성이 높습니다. TL이 누구인지 명확하지 않다면 v8-eng-review-owners@googlegroups.com을 통해 V8 Eng Review Owners에게 질문하십시오. TL은 적절한 경우 추가적으로 필요한 LGTM 제공자를 목록에 추가할 책임이 있습니다.

### LGTM 제공자

LGTM: 반드시 필요
이 사람은 LGTM을 제공해야 하는 사람입니다. IC일 수도 있고 TL(M)일 수도 있습니다.

### 문서의 “임의” 리뷰어(RRotD)

LGTM: 필요하지 않음
이 사람은 단순히 제안을 검토하고 의견을 남깁니다. 이들의 의견은 고려되어야 하지만 LGTM은 필요하지 않습니다.

### V8 Eng Review Owners

LGTM: 필요하지 않음
막힌 제안은 [v8-eng-review-owners@googlegroups.com](mailto:v8-eng-review-owners@googlegroups.com)을 통해 V8 Eng Review Owners에 '상승'할 수 있습니다. 이러한 상승의 잠재적 사용 사례:

- LGTM 제공자가 응답하지 않는 경우
- 디자인에 대해 합의에 도달할 수 없는 경우

V8 Eng Review Owners는 LGTM을 제공하지 않은 사람 또는 LGTM을 제공한 사람의 의견을 무효화할 수 있습니다.

## 구체적인 워크플로우

![V8의 디자인 리뷰 가이드라인 한눈에 보기](/_img/docs/design-review-guidelines/design-review-guidelines.svg)

1. 시작: IC가 기능 작업을 결정하거나 기능을 할당받음
1. IC가 초기 디자인 문서/설명/한 장짜리를 몇몇 RRotD에게 보냄
    1. 프로토타입은 "디자인 문서"의 일부로 간주됩니다.
1. IC는 LGTM을 제공해야 할 사람들을 LGTM 제공자 목록에 추가합니다. TL은 반드시 목록에 포함되어야 합니다.
1. IC는 피드백을 반영합니다.
1. TL이 LGTM 제공자 목록에 더 많은 사람을 추가합니다.
1. IC는 초기 디자인 문서/설명/한 장짜리를 [v8-dev+design@googlegroups.com](mailto:v8-dev+design@googlegroups.com)에 보냅니다.
1. IC가 LGTM을 수집합니다. TL이 돕습니다.
    1. LGTM 제공자는 문서를 검토하고 댓글을 추가하며 문서 시작 부분에서 LGTM 또는 LGTM이 아닌지를 제공합니다. LGTM이 아닌 경우, 이유를 반드시 나열해야 합니다.
    1. 선택사항: LGTM 제공자는 자신을 LGTM 제공자 목록에서 제거하거나 다른 LGTM 제공자를 제안할 수 있습니다.
    1. IC와 TL은 해결되지 않은 문제를 해결하려고 노력합니다.
    1. 모든 LGTM이 수집되면 기존 스레드를 핑하여 v8-dev@googlegroups.com에 이메일을 보내 구현을 발표하십시오.
1. 선택사항: IC와 TL이 차단되거나 더 광범위한 토론을 원할 경우 문제를 V8 Eng Review Owners에 상정할 수 있습니다.
    1. IC가 v8-eng-review-owners@googlegroups.com에 이메일을 보냅니다.
        1. TL을 CC에 추가
        1. 메일 안에 디자인 문서 링크 첨부
    1. 모든 V8 Eng Review Owners 구성원이 문서를 검토하고 선택적으로 자신을 LGTM 제공자 목록에 추가해야 합니다.
    1. 기능을 해제할 다음 단계가 결정됩니다.
    1. 이후에도 차단이 해결되지 않거나 새로운 해결 불가능한 차단이 발견되면, 8번으로 이동하십시오.
1. 선택사항: 기능이 이미 승인된 후 LGTM이 아닌 의견이 추가되면 이를 정상적인 해결되지 않은 문제로 처리해야 합니다.
    1. IC와 TL은 해결되지 않은 문제를 해결하기 위해 노력합니다.
1. 종료: IC가 기능을 진행합니다.

그리고 항상 기억하십시오:

1. 선한 의도를 가정하십시오.
1. 친절하고 문명적으로 행동하십시오.
1. 실용적이어야 합니다.

## FAQ

### 이 기능이 디자인 문서를 작성할 가치가 있는지 어떻게 결정합니까?

디자인 문서가 적절한 경우에 대한 몇 가지 힌트:

- 최소 두 개 이상의 구성 요소에 영향을 미침
- 비-V8 프로젝트 예: 디버거(Debugger), Blink와의 조정 필요
- 구현에 1주 이상의 노력이 소요됨
- 언어 기능임
- 플랫폼 특정 코드를 다루게 됨
- 사용자에게 직접적인 영향을 미치는 변경사항
- 특별한 보안 고려 사항이 있거나 보안 영향이 명확하지 않음

의심스러우면 TL에게 물어보세요.

### LGTM 제공자 목록에 누구를 추가할지 결정하는 방법은?

다음은 사람들이 LGTM 제공자 목록에 추가되어야 할 때에 대한 몇 가지 포인터입니다:

- 소스 파일/디렉토리의 OWNER
- 예상되는 주요 컴포넌트 전문가
- 변경 사항의 다운스트림 소비자 (예: API를 변경할 때)

### “내” TL은 누구인가요?

아마도 당신의 기능이 영향을 줄 주요 컴포넌트의 OWNER 중 한 명일 가능성이 높습니다. TL이 누구인지 명확하지 않다면 v8-eng-review-owners@googlegroups.com를 통해 V8 엔지니어링 리뷰 OWNER들에게 문의하세요.

### 디자인 문서에 대한 템플릿은 어디에서 찾을 수 있나요?

[여기](https://docs.google.com/document/d/1CWNKvxOYXGMHepW31hPwaFz9mOqffaXnuGqhMqcyFYo/template/preview)에서 확인할 수 있습니다.

### 큰 변화가 생기면 어떻게 해야 하나요?

예를 들어 LGTM 제공자들에게 명확하고 합리적인 기한을 제시하면서 반복적으로 연락하여 여전히 LGTM을 받았는지 확인하세요.

### LGTM 제공자들이 내 문서에 댓글을 달지 않을 때, 나는 무엇을 해야 하나요?

이 경우에는 다음과 같은 경로를 따라 점진적으로 문제를 해결할 수 있습니다:

- 이메일, 행아웃 또는 문서 내 댓글/지정을 통해 직접 연락하여 명시적으로 LGTM 또는 비-LGTM을 추가해달라고 요청하세요.
- 당신의 TL에게 도움을 요청하세요.
- v8-eng-review-owners@googlegroups.com로 문제를 에스컬레이션하세요.

### 누군가가 나를 LGTM 제공자로 문서에 추가했어요. 내가 해야 할 일은?

V8은 더 투명한 의사결정과 더 간단한 에스컬레이션을 목표로 합니다. 디자인이 충분히 좋고 실행해야 한다고 생각된다면, 테이블의 본인 이름 옆 셀에 “LGTM”을 추가하세요.

차단할 우려나 의견이 있다면, 테이블의 본인 이름 옆 셀에 “Not LGTM, because \<이유>”를 작성하세요. 추가 리뷰 요청을 받을 준비를 해두세요.

### 이 프로세스는 Blink Intents 프로세스와 어떻게 작동하나요?

V8 디자인 리뷰 가이드라인은 [V8의 Blink Intent+Errata 프로세스](/docs/feature-launch-process)를 보완합니다. WebAssembly나 JavaScript 언어 기능을 새로 출시하려는 경우, 꼭 V8의 Blink Intent+Errata 프로세스와 V8 디자인 리뷰 가이드라인을 따르세요. Intent to Implement를 보낼 시점에 모든 LGTM을 취합하는 것이 합리적일 가능성이 높습니다.
