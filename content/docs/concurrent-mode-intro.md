---
id: concurrent-mode-intro
title: Concurrent모드 소개 (실험 단계)
permalink: docs/concurrent-mode-intro.html
next: concurrent-mode-suspense.html
---

<style>
.scary > blockquote {
  background-color: rgba(237, 51, 21, 0.2);
  border-left-color: #ed3315;
}
</style>

<div class="scary">

>주의사항
>
>이 페이지는 안정된 배포판에서 아직 제공되지 않는 실험적인 기능에 대해 설명합니다. 얼리 어답터와 궁금해하시는 분을 대상으로 합니다.
>
>많은 정보는 구식이며 보관을 목적으로만 유지되고 있습니다. 최신화된 정보는 [React 18 Alpha announcement post](/blog/2021/06/08/the-plan-for-react-18.html)를 참조해주세요.
>
>React 18이 배포되기 전에 해당 페이지를 안정된 문서로 대체할 예정입니다.

</div>

이 페이지는 Concurrent 모드의 이론적인 개요를 설명합니다. **더 실용적인 설명을 위해서는 다음 섹션들을 참고하세요**

* [데이터를 가져오기 위한 Suspense](/docs/concurrent-mode-suspense.html) React 컴포넌트에서 데이터를 가져오기에 대한 새로운 메커니즘을 설명합니다.
* [Concurrent UI 패턴](/docs/concurrent-mode-patterns.html) Concurrent 모드와 Suspense로 만들어진 UI 패턴들을 설명합니다.
* [Concurrent 모드 도입하기](/docs/concurrent-mode-adoption.html) 프로젝트에서 Concurrent 모드 사용방법을 설명합니다.
* [Concurrent 모드 API 참고서](/docs/concurrent-mode-reference.html) 실험적 빌드에서 사용 가능한 새로운 API 문서입니다.

## Concurrent 모드란? {#what-is-concurrent-mode}

Concurrent모드는 React 앱이 빠른 반응속도를 유지하도록 하고 사용자의 장치 기능 및 네트워크 속도에 적절하게 맞추도록 돕는 새로운 기능들의 집합체입니다.

이 기능들은 여전히 실험적이며 변경될 수 있습니다. 아직 안정된 React 배포판에 포함되지 않았지만, 실험 배포판에서 그 기능들을 시도해볼 수는 있습니다.

## 차단 vs 인터럽트 렌더링 {#blocking-vs-interruptible-rendering}

**Concurrent 모드를 설명하기 위해서 버전 관리를 예를 들어 설명할 것입니다** 팀에서 일하고 있다면 Git과 같은 버전 관리 시스템을 사용하며 브랜치로 작업할 것입니다. 브랜치가 준비된다면 다른 사람들이 해당 작업을 가져올 수 있도록 작업을 마스터 브랜치로 병합 할 수 있습니다.

버전 관리가 있기 전에는, 개발 작업의 흐름이 지금과는 많이 달랐습니다. 브랜치라는 개념이 없어서 몇몇 파일을 수정하려면 그 파일들을 수정 작업을 완료하기 전까지 작업하면 안 된다고 모든 사람에게 말해야 했습니다. 심지어는 다른 사람과 동시에 일을 시작할 수도 없었습니다 즉, 문자그대로 이러한 제한사항들에 의해 *차단*되었습니다.

이는 React를 포함한 UI 라이브러리들이 어떻게 오늘날 작용하는지를 설명해줍니다. 업데이트 렌더링(새로운 DOM 노드 생성 및 컴포넌트 내에 있는 코드 실행하는 것을 포함해서)을 시작하면 이 일은 방해받지 않습니다. 이러한 접근을 "렌더링 차단"이라고 합니다.

Concurrent 모드에서는, 렌더링은 차단되지 않으나 인터럽트는 가능합니다. 이는 사용자의 경험을 개선하며 또한 이전에 사용할 수 없었던 기능들을 사용할 수 있도록 만들어줍니다. [다음](https://reactjs.org/docs/concurrent-mode-suspense.html) [챕터](https://reactjs.org/docs/concurrent-mode-patterns.html)에서 구체적인 예시를 살펴보기 전에, 새로운 기능들의 높은 수준의 개요를 해보려고 합니다.

###인터럽트 가능한 렌더링 {#interruptible-rendering}

필터링 가능한 제품 목록을 생각해보세요. 목록 필터에 입력하고 모든 키를 누를 때마다 버벅거림을 느낀 적이 있나요? 제품 목록을 업데이트하는 몇몇 작업은 불가피할 수 있습니다. (예를 들어, 새로운 DOM 노드를 만들거나 레이아웃을 수행하는 브라우저를 만드는 작업) 그러면, *언제* *어떻게*  이런 비중 있는 작업을 수행할까요?

버벅거림을 해결하는 한 가지 방법은 입력을 "디바운싱"해주는 것입니다 디바운싱하면, 사용자가 타이핑을 멈추고 나서만 목록을 업데이트합니다. 하지만, 타이핑하고 있을 때 UI가 업데이트하지 않는 사실이 실망스러울 수 있습니다. 이에 대한 대안으로, 입력을 “스로틀”하고 목록을 특정 최대 빈도수로 업데이트 할 수 있습니다. 그러나 저전력 장치에서는 여전히 버벅거릴 것입니다. 디 바운싱 및 쓰로틀링은 둘 다 최적이 아닌 사용자 환경을 만듭니다.

버벅거리는 이유는 간단합니다 렌더링이 시작되면 중간에 중단될 수 없기 때문입니다. 그래서 그 브라우저는 텍스트 입력을 키를 누르는 즉시 업데이트할 수 없습니다. 벤치마크에서 UI 라이브러리(예를 들어 React)가 아무리 잘 보일지라도, 차단 렌더링을 사용하고 컴포넌트의 일정량 작업을 하면 항상 버벅거림을 초래할 것입니다. 그리고 종종, 이에 대한 쉬운 해결책을 찾기 어렵습니다.

**Concurrent 모드는 렌더링을 인터럽트 가능하도록 만듦으로써 근본적인 문제를 수정합니다** 이러한 사실은 사용자가 다른 키를 누를 때, React는 브라우저에 텍스트 입력을 업데이트하는 것을 차단할 필요가 없음을 의미합니다. 대신에, React는 브라우저가 입력에 대한 업데이트를 paint하고 *메모리 내에* 있는 업데이트 목록을 계속 렌더링할 수 있도록 합니다. 렌더링이 끝나면 React는 DOM을 업데이트하고 변경 사항들을 화면에 반영합니다.

개념상으로, React가 "브랜치에서" 모든 업데이트를 준비하는 것으로 생각할 수 있습니다. 브랜치 내에서 작업을 중지하거나 브랜치 사이에서 전환이 자유로운 것처럼, Concurrent모드 내에 React는 더 중요한 일을 위해 진행 중인 업데이트를 중단할 수 있고 그리고서 이전 작업으로 돌아갈 수도 있습니다. 이 기술은 비디오 게임에서 [이중 버퍼링](https://wiki.osdev.org/Double_Buffering)을 떠오르게끔 할 수도 있습니다.

Concurrent모드 기술은 UI에서 디바운싱과 스로틀링의 필요성을 줄입니다 렌더링은 중단이 가능하기 때문에 버벅거림을 피하고자 일부러 작업을 지연시킬 필요가 없습니다. React는 렌더링을 즉시 시작할 수 있지만, 앱에 즉각적인 반응성이 요구될 때는 이 작업을 중단할 수도 있습니다.

### 의도적인 로딩 시퀀스 {#intentional-loading-sequences}

Concurrent 모드는 "브랜치에서" 하는 React 작업과 같다고 전에 말했습니다. 브랜치는 단기적인 수정을 할 때뿐만 아니라 장기적인 실행 기능에도 유용합니다. 때로는 기능에 대해 작업을 할 수도 있습니다. 하지만 마스터로 병합할 수 있는 "적합한 state"가 되기까지 몇 주가 걸릴 수 있습니다. 버전 관리의 이러한 측면은 렌더링에도 적용됩니다.

한번 앱에서 두 화면 사이를 탐색한다고 가정해보겠습니다. 경우에 따라서, 새 화면에서 사용자에게 "충분히 좋은" 로딩 state를 보여주기 위해 필요한 코드와 데이터를 불러오지 못 할 수 있습니다. 빈 화면이나 큰 스피너로 전환하는 것은 어려운 일이 될 수 있지만 일반적으로 필요한 코드와 데이터를 가져오는 데에 그렇게 많은 시간이 소요되지않습니다. **React가 기존 화면에서 조금 더 오래 유지할 수 있고 새 화면을 보여주기 전에 "안 좋은" 로딩 state를 "건너뛸 수" 있다면 더 좋지 않을까요?**

오늘날 이것이 가능하긴 하지만 조정하기는 어려울 수 있습니다. Concurrent 모드에서는 이 기능이 내장되어 있습니다. React는 먼저 메모리, 비유하자면 "다른 브랜치", 에서 새로운 화면을 준비하기 시작합니다. 그래서 React는 더 많은 콘텐츠를 불러올 수 있도록 DOM을 업데이트하기 전에 기다릴 수 있습니다. Concurrent 모드에서는 React가 인라인 표시기로 완벽하게 상호작용하는 이전 화면을 계속 표시하도록 지시할 수 있습니다.

### 동시성 {#concurrency}

위의 두 가지 예시를 살펴보고 Concurrent 모드가 어떻게 통합되는지 살펴보겠습니다. **Concurrent 모드에서 React는 여러 작업을 *동시에*, 다른 팀원들이 각자 작업을 할 수 있는 브랜치처럼, 진행할 수 있습니다**

* CPU 바운드 업데이트(예를 들어 DOM 노드 만들기 및 컴포넌트 코드 실행)의 경우 Concurrency는 더욱 긴급한 업데이트가 이미 시작한 렌더링을 "중단" 할 수 있음을 의미합니다.
* IO 바운드 업데이트(예를 들어 네트워크에서 코드나 데이터를 가져오는 것)의 경우 Concurrency는 모든 데이터가 도달하기 전에 React가 메모리에서 렌더링을 시작할 수 있으며 빈 로딩 state표시를 무시할 수 있음을 의미합니다.

중요한 것은 React를 사용하는 *방식*이 똑같다는 것입니다. 컴포넌트, props 및 state와 같은 개념은 근본적으로 동일한 방식으로 작동합니다. 화면을 업데이트하려면 state를 설정합니다.

React는 휴리스틱을 사용하여 업데이트의 "급함" 정도를 결정하고 몇 줄의 코드를 수정해서 사용자가 모든 상호작용에 대해 원하는 사용자의 경험을 얻을 수 있도록 합니다..

## 생산에 연구를 투입  {#putting-research-into-production}

Concurrent 모드 기능에 대한 공통 주제들이 있습니다. **이 주제들의 목표는 사람-컴퓨터 간 상호작용에 대한 연구 결과가 실제 UI와 통합되도록 돕는 것입니다.**

예를 들어, 조사에 따르면 화면 간 전환에서 중간 로딩 state를 너무 많이 표시하면 전환이 더 *느리게* 느껴진다고 합니다. 이것이 Concurrent 모드에서 고정된 "스케줄"에 새로운 로드 state를 표시하여 부조화가 발생하는 것 또는 불필요하게 많이 업데이트되는 것을 피하고자 하는 이유입니다.

마찬가지로 조사 결과에 따르면, 호버나 텍스트 입력 같은 상호 작용은 아주 짧은 시간 안에 처리되어야 하지만, 클릭이나 페이지 전환은 조금 더 오래 걸린다고 해도 지루한 느낌을 주지않는다고 합니다. Concurrent 모드 내부적으로 사용하는 다른 "우선순위"는 사람들의 인식에 대한 조사에서의 상호 작용에 대한 부분과 대략적으로 일치합니다.

UX(사용자 경험)에 중점을 둔 팀은 가끔 이러한 비슷한 문제들을 일회성 솔루션으로 해결하곤 합니다. 그러나, 그런 솔루션들은 오래 지속하기가 어렵고 유지하기도 어렵습니다. Concurrent 모드를 통한 목적은 UI 조사 결과를 추상화시키고 그것을 사용할 관용적인 방법을 제공하는 것입니다. UI 라이브러리로서, React는 이러한 일을 할 수 있습니다.

## 다음 단계 {#next-steps}

이제 Concurrent 모드에 대해 모두 알게 되었습니다!

다음 페이지에서는 특정 주제에 대한 자세한 내용을 배웁니다.

* [데이터를 가져오기 위한 Suspense](/docs/concurrent-mode-suspense.html) React 컴포넌트에서 데이터를 가져오기에 대한 새로운 메커니즘을 설명합니다.
* [Concurrent UI 패턴](/docs/concurrent-mode-patterns.html) Concurrent 모드와 Suspense로 만들어진 UI 패턴들을 설명합니다.
* [Concurrent 모드 도입하기](/docs/concurrent-mode-adoption.html) 프로젝트에서 Concurrent 모드 사용방법을 설명합니다.
* [Concurrent 모드 API 참고서](/docs/concurrent-mode-reference.html) 실험적 빌드에서 사용 가능한 새로운 API 문서입니다.