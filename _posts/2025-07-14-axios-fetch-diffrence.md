---
title: "Fetch vs Axios — 언제, 왜, 어떤 걸 써야 할까?"
author: tjddnr9553
date: 2025-07-14 09:00:00 +0900
categories: [Frontend, JavaScript]
tags: [fetch, axios, HTTP, API, Network]

---

# 서론

JavaScript에서 HTTP 요청을 보내는 방법은 다양하지만, 대부분의 개발자는 두 가지를 많이 사용합니다:

* **`fetch`**: 브라우저에 기본 내장된 표준 API
* **`axios`**: 풍부한 기능을 갖춘 오픈소스 HTTP 클라이언트 라이브러리

그렇다면 이 둘의 **차이점은 무엇이고**, **언제 어떤 것을 사용하는 것이 적합할까요?**

이번 글에서는 `fetch`와 `axios`의 **기능, 사용성, 에러 처리, 인터셉터, 브라우저 호환성** 등 다양한 관점에서 비교하고, **실무에서 어떤 선택이 더 나은지** 고민해보겠습니다.

---

## 1. fetch

### 개요

`fetch`는 ES6 이후 브라우저에 내장된 네이티브 API로, **HTTP 요청을 보내는 표준 방식**입니다.

```js
fetch('https://api.example.com/data')
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err));
```

### 장점

* ✅ **브라우저 내장** — 별도 설치 없이 즉시 사용 가능
* ✅ **Promise 기반**으로 비동기 흐름이 직관적
* ✅ **Stream, Blob 등 다양한 응답 타입 처리 가능**
* ✅ 최신 브라우저에서 널리 지원됨

### 단점

* ❌ **응답 상태 코드가 404/500이어도 에러로 인식되지 않음**
* ❌ `timeout`, `upload progress`, `cancel token` 기능 없음
* ❌ `application/x-www-form-urlencoded` 전송 시 직접 설정 필요
* ❌ 응답 내용을 추출하기 위해 `res.json()` 또는 `res.text()`를 항상 호출해야 함

---

## 2. axios

### 개요

`axios`는 `XMLHttpRequest` 기반의 **Promise 기반 HTTP 클라이언트**로, `fetch`보다 많은 기능을 기본 제공합니다.

```js
import axios from 'axios';

axios.get('https://api.example.com/data')
  .then(res => console.log(res.data))
  .catch(err => console.error(err));
```

### 장점

* ✅ **자동 JSON 변환** — 응답이 자동으로 `res.data`에 저장됨
* ✅ **인터셉터(Interceptors)** — 요청/응답 가로채기 기능으로 인증 토큰 등 처리 가능
* ✅ **타임아웃 설정**, **요청 취소**, **진행률 이벤트** 지원
* ✅ 브라우저와 Node.js 모두 사용 가능

### 단점

* ❌ **패키지 설치 필요** (`npm install axios`)
* ❌ **번들 사이즈**가 fetch에 비해 다소 큼
* ❌ `fetch`보다 내부적으로 복잡한 동작 (예: 자동 헤더 설정 등)

---

## 기능 비교

| 항목              | fetch                  | axios                                   |
| --------------- | ---------------------- | --------------------------------------- |
| 설치 필요 여부        | ❌ (브라우저 기본 내장)         | ⭕ `npm install axios` 필요                |
| 응답 자동 파싱        | ❌ (수동 `res.json()` 호출) | ⭕ (`res.data`에 자동 저장)                   |
| 에러 처리 방식        | ❌ HTTP 오류 코드도 성공 처리    | ⭕ HTTP 오류 시 자동 reject                   |
| 요청 취소           | ❌ 기본 지원 없음             | ⭕ `AbortController` 또는 `CancelToken` 지원 |
| 인터셉터 지원         | ❌ 없음                   | ⭕ 요청 및 응답 가로채기 가능                       |
| 응답 타입 자동 처리     | ❌ 수동 지정                | ⭕ 자동으로 Content-Type 인식                  |
| 업로드/다운로드 진행률 추적 | ❌ 지원 안함                | ⭕ `onUploadProgress` 가능                 |
| 동작 환경           | 브라우저                   | 브라우저 + Node.js (SSR 가능)                 |
| 타임아웃 설정         | ❌ 없음                   | ⭕ 가능 (`timeout: 5000`)                  |

---

## 실제 사용 예: POST 요청

### fetch

```js
fetch('/api/user', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Alice' })
})
  .then(res => {
    if (!res.ok) throw new Error('서버 에러');
    return res.json();
  })
  .then(data => console.log(data));
```

### axios

```js
axios.post('/api/user', { name: 'Alice' })
  .then(res => console.log(res.data))
  .catch(err => console.error(err));
```

---

## 언제 어떤 걸 사용할까?

| 상황                                  | 추천        |
| ----------------------------------- | --------- |
| 번들 사이즈를 최소화하고 싶을 때                  | **fetch** |
| Node.js에서도 요청을 보내야 할 때              | **axios** |
| 요청/응답을 일괄적으로 가로채고 싶을 때 (ex. JWT 토큰) | **axios** |
| 요청 실패 시 강력한 에러 핸들링이 필요할 때           | **axios** |
| 단순한 GET/POST 요청만 수행할 때              | **fetch** |
| 요청 취소, 타임아웃, 진행률 추적이 필요한 경우         | **axios** |

---

## 결론

* **간단한 요청**, **작은 프로젝트**: `fetch`로도 충분
* **복잡한 인증 처리**, **공통 응답 로직**, **SSR, 대규모 프로젝트**: `axios`가 더 유리

| 기준                         | 추천 도구   |
| -------------------------- | ------- |
| 브라우저 환경 + 간단한 API 호출       | `fetch` |
| 요청/응답 가로채기 및 에러 제어가 필요한 경우 | `axios` |
| Node.js 또는 SSR 지원 필요       | `axios` |

> ✅ **결론**: 작은 프로젝트나 단순 요청은 `fetch`,
> 복잡한 에러 처리나 인증, SSR까지 고려한다면 `axios`가 더 실용적입니다.

---
