---
title: "React 모달이 화면 중앙에서 이동하는 문제 해결하기 — React Portal 활용법"
author: tjddnr9553
date: 2025-09-08 10:00:00 +0900
categories: [Frontend, React]
tags: [React, Modal, Portal, UI, CSS]
---

# 서론

React 프로젝트에서 모달을 구현하다가 이상한 현상을 발견했습니다.  
분명 CSS로 `fixed` 포지션과 `flexbox`를 사용해 중앙 정렬했는데, **모달이 나타날 때 화면 중앙이 아닌 다른 위치에서 시작해 중앙으로 이동하는 듯한 깜빡임**이 발생했습니다.

처음엔 단순한 CSS 문제라고 생각했지만, 알고 보니 React의 **렌더링 순서**와 **DOM 트리 구조**에 관련된 더 깊은 문제였습니다.

이번 글에서는 **왜 React 모달이 예상치 못한 위치에서 렌더링되는지**, 그리고 **React Portal을 사용해 이 문제를 깔끔하게 해결하는 방법**을 정리해보겠습니다.

---

## 문제 상황: 모달이 중앙에서 벗어나는 이유

### 초기 코드

```jsx
// Project/page.tsx
{showCreateModal && (
  <div className="fixed inset-0 bg-black/50 flex justify-center items-center z-50 p-4">
    <div className="bg-white p-6 rounded-xl shadow-xl w-full max-w-md">
      <h2>새 프로젝트 생성</h2>
      {/* 모달 내용 */}
    </div>
  </div>
)}
```

위 코드는 언뜻 보면 문제없어 보입니다. `fixed` 포지션으로 화면 전체를 덮고, `flex`로 중앙 정렬했으니까요.

### 발생한 문제

하지만 실제로 모달을 열면 다음과 같은 현상이 발생했습니다:

1. 모달이 화면 중앙이 아닌 **부모 컴포넌트 위치**에서 렌더링
2. 잠시 후 CSS가 적용되며 중앙으로 이동
3. 사용자에게는 **깜빡임이나 위치 이동**으로 보임

### 원인 분석

이 문제의 주요 원인은 다음과 같습니다:

| 원인 | 설명 |
|-----|------|
| **DOM 트리 위치** | 모달이 부모 컴포넌트 내부에 렌더링되어 초기 위치가 부모에 의존 |
| **CSS 적용 타이밍** | React가 DOM을 생성한 후 CSS가 적용되는 시차 발생 |
| **부모 스타일 간섭** | 부모 컴포넌트의 `transform`, `position` 등이 `fixed` 동작에 영향 |
| **리플로우(Reflow)** | 모달 렌더링 시 전체 레이아웃 재계산으로 인한 성능 저하 |

특히 부모 컴포넌트에 `transform`이나 `filter` 같은 CSS 속성이 있으면, **`fixed` 포지션이 viewport가 아닌 부모 요소를 기준으로 동작**하게 됩니다.

---

## 해결 방법: React Portal 사용하기

### React Portal이란?

React Portal은 **부모 컴포넌트의 DOM 계층을 벗어나** 다른 DOM 노드에 자식을 렌더링할 수 있게 해주는 기능입니다.

```jsx
import { createPortal } from 'react-dom';

// 일반 렌더링: 부모 내부에 렌더링
<div className="parent">
  <Modal />  {/* parent 내부에 렌더링 */}
</div>

// Portal 사용: body에 직접 렌더링
<div className="parent">
  {createPortal(<Modal />, document.body)}  {/* body에 렌더링 */}
</div>
```

### Portal을 사용한 해결 코드

```jsx
import { createPortal } from 'react-dom';

// 수정된 모달 코드
{showCreateModal && typeof document !== 'undefined' && createPortal(
  <div className="fixed inset-0 bg-black/50 flex justify-center items-center z-[9999] p-4 animate-fadeIn">
    <div className="bg-white p-6 rounded-xl shadow-xl w-full max-w-md animate-scaleIn">
      <h2>새 프로젝트 생성</h2>
      {/* 모달 내용 */}
    </div>
  </div>,
  document.body  // body에 직접 렌더링
)}
```

### 애니메이션 추가

부드러운 모달 등장을 위해 CSS 애니메이션도 추가했습니다:

```scss
// animation.scss
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.animate-fadeIn {
  animation: fadeIn 0.2s ease-out;
}

.animate-scaleIn {
  animation: scaleIn 0.2s ease-out;
}
```

---

## Portal 사용의 이점

| 이점 | 설명 |
|-----|------|
| **독립적인 DOM 위치** | 부모 컴포넌트의 스타일에 영향받지 않음 |
| **z-index 관리 용이** | 최상위 레벨에서 렌더링되어 다른 요소 위에 확실히 표시 |
| **성능 최적화** | 부모 컴포넌트 리렌더링 시 모달이 영향받지 않음 |
| **SSR 호환성** | `typeof document !== 'undefined'` 체크로 서버 사이드 렌더링 지원 |

---

## 실제 적용 시 주의사항

### 1. SSR(Server-Side Rendering) 환경

Next.js 같은 SSR 환경에서는 `document` 객체가 서버에서 존재하지 않으므로 체크가 필요합니다:

```jsx
{showModal && typeof document !== 'undefined' && createPortal(
  <Modal />,
  document.body
)}
```

### 2. 이벤트 버블링

Portal로 렌더링된 컴포넌트도 **React 이벤트는 원래 부모로 버블링**됩니다:

```jsx
// 부모 컴포넌트
<div onClick={() => console.log('부모 클릭')}>
  {createPortal(
    <button onClick={() => console.log('자식 클릭')}>
      클릭
    </button>,
    document.body
  )}
</div>
// 버튼 클릭 시: "자식 클릭" → "부모 클릭" 순서로 출력
```

### 3. 접근성 고려

모달 사용 시 접근성을 위해 다음 속성들을 추가하는 것이 좋습니다:

```jsx
<div 
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  className="fixed inset-0..."
>
  <h2 id="modal-title">모달 제목</h2>
</div>
```

---

## Portal vs 일반 렌더링 비교

| 항목 | 일반 렌더링 | Portal 렌더링 |
|-----|------------|--------------|
| DOM 위치 | 부모 컴포넌트 내부 | 지정한 DOM 노드 (예: body) |
| 부모 스타일 영향 | 받음 | 받지 않음 |
| z-index 관리 | 복잡함 | 간단함 |
| 이벤트 버블링 | 일반적 | React 이벤트만 버블링 |
| 사용 복잡도 | 단순 | 약간 복잡 |

---

## 결론

React에서 모달이 예상치 못한 위치에 렌더링되는 문제는 **DOM 트리 구조와 CSS 상속** 때문에 발생합니다.

**React Portal**을 사용하면:
- ✅ 모달이 부모 컴포넌트의 스타일 영향을 받지 않음
- ✅ 화면 중앙에 정확히 위치
- ✅ z-index 관리가 간편해짐
- ✅ 깜빡임 없이 부드럽게 나타남

> 💡 **핵심 팁**: 모달, 툴팁, 드롭다운 같은 **오버레이 UI**는 Portal을 사용해 document.body에 렌더링하는 것이 가장 안전하고 예측 가능한 방법입니다.

모달 위치 문제로 고민 중이라면, Portal을 한 번 시도해보세요. 단 몇 줄의 코드 변경으로 훨씬 안정적인 UI를 구현할 수 있습니다.

---

### 참고 자료
- [React 공식 문서 - Portals](https://react.dev/reference/react-dom/createPortal)
- [MDN - CSS position](https://developer.mozilla.org/en-US/docs/Web/CSS/position)
- [CSS Tricks - Fixed Positioning](https://css-tricks.com/absolute-relative-fixed-positioining-how-do-they-differ/)