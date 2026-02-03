---
title: "NCP 다중 서버 환경에서 Next.js 빌드 충돌 해결기"
author: tjddnr9553
date: 2026-02-03 10:00:00 +0900
categories: [DevOps, Trouble Shooting]
tags: [NCP, Next.js, Docker, Load Balancer, Auto Scaling, DevOps]
---

## 서론

서비스가 성장하면서 서버 한 대로는 감당이 안 되는 시점이 온다. 드디어 그 시점을 맞이했고, NCP(Naver Cloud Platform)에서 서버를 1대에서 2대로 늘리고 로드밸런서를 붙이고, 오토스케일링까지 적용하는 작업을 진행했다.

그런데... 예상치 못한 곳에서 서비스가 완전히 멈춰버렸다.

---

## 기존 인프라 구조

먼저 우리 서비스의 구조를 간단히 설명하면:

```
NAS (공유 스토리지)
├── /code
│   ├── frontend/        # Next.js 프로젝트
│   │   ├── .next/       # 빌드 결과물
│   │   └── ...
│   └── backend/
└── ...

서버 1 ──┬── Docker Container (Frontend)
         └── Docker Container (Backend)

서버 2 ──┬── Docker Container (Frontend)  <- 새로 추가
         └── Docker Container (Backend)
```

GitHub 코드와 빌드 결과물은 NAS에 저장하고, 각 서버에서 Docker 컨테이너로 마운트해서 사용하는 구조였다. 서버가 1대일 때는 이 구조가 아무 문제 없이 잘 동작했다.

---

## 문제 발생: 동시 빌드로 인한 서비스 중단

서버를 2대로 늘리고 로드밸런서를 연결한 직후, 프론트엔드 배포를 진행했다.

**그 순간 서비스가 완전히 죽었다.**

### 원인 분석

문제의 원인은 생각보다 단순했다:

1. **서버 1**에서 `npm run build` 실행 → `.next` 폴더에 빌드 결과물 생성 중
2. **서버 2**에서도 같은 타이밍에 `npm run build` 실행
3. **두 서버가 같은 NAS의 `.next` 폴더를 동시에 덮어쓰기**
4. 빌드 중인 `.next` 폴더의 파일들이 서로 충돌
5. 기존 서비스도, 새 빌드도 모두 깨져버림

```
[에러 상황]
서버 1: .next/static/chunks/main-abc123.js 생성 중
서버 2: .next/static/chunks/main-def456.js로 덮어쓰기
서버 1: main-abc123.js를 참조하려는데 파일이 없음 → 500 에러
```

Next.js는 빌드할 때마다 고유한 `BUILD_ID`를 생성하고, 이에 맞는 청크 파일들을 만든다. 두 서버의 `BUILD_ID`가 다르니 서로의 파일을 참조할 수 없는 상황이 된 것이다.

---

## 첫 번째 시도: Sticky Session 활성화

급한 불을 끄기 위해 NCP 로드밸런서에서 **Sticky Session**을 활성화했다.

### Sticky Session이란?

Sticky Session(세션 어피니티)은 **한 번 연결된 클라이언트를 계속 같은 서버로 보내는 기능**이다.

```
[Sticky Session OFF]
사용자 A → 요청 1 → 서버 1
사용자 A → 요청 2 → 서버 2  (다른 서버로 갈 수 있음)
사용자 A → 요청 3 → 서버 1

[Sticky Session ON]
사용자 A → 요청 1 → 서버 1
사용자 A → 요청 2 → 서버 1  (항상 같은 서버)
사용자 A → 요청 3 → 서버 1
```

보통 쿠키 기반으로 동작하며, 클라이언트가 처음 접속할 때 어떤 서버에 연결되었는지 기록해두고 이후 요청은 같은 서버로 라우팅한다.

### 결과

Sticky Session을 켜니 문제는 해결됐다. 각 사용자가 하나의 서버에 고정되니 `BUILD_ID` 불일치 문제가 발생하지 않았다.

**하지만 이건 근본적인 해결책이 아니었다.**

| 문제점 | 설명 |
|--------|------|
| 로드밸런싱 효과 감소 | 트래픽이 균등하게 분산되지 않을 수 있음 |
| 서버 장애 시 문제 | 고정된 서버가 죽으면 해당 사용자들은 세션 유실 |
| 오토스케일링과 충돌 | 새 서버가 추가되어도 기존 사용자는 기존 서버에 고정 |
| 배포 시 복잡성 | 특정 서버만 새 빌드를 서빙하는 상황 발생 |

로드밸런서를 달아놓고 Sticky Session으로 고정해버리면, 로드밸런서를 쓰는 의미가 크게 퇴색된다.

---

## 두 번째 시도: 임시 폴더로 빌드 후 교체

Sticky Session 없이 해결하기 위해 빌드 전략을 바꿔봤다.

```bash
# 기존 방식
npm run build  # .next 폴더에 직접 빌드

# 변경된 방식
NEXT_BUILD_DIR=.next-new npm run build  # .next-new에 빌드
rm -rf .next.old
mv .next .next.old
mv .next-new .next  # 빠르게 교체
```

이렇게 하면:
1. 빌드하는 동안 기존 `.next`는 그대로 서비스 중
2. 빌드 완료 후 순식간에 교체
3. 다운타임 최소화

### 결과

**여전히 실패.** Sticky Session을 해제하자마자 다시 흰 화면이 나왔다.

원인은 **서버 간 빌드 타이밍 불일치** 문제였다:

```
시간 T1: 서버 1 - 새 빌드(.next) 적용 완료, BUILD_ID = abc123
시간 T1: 서버 2 - 아직 빌드 중, BUILD_ID = old456

사용자 요청:
1. 서버 1에서 HTML 받음 (BUILD_ID abc123의 JS 파일 참조)
2. 서버 2로 JS 파일 요청 라우팅됨
3. 서버 2에는 abc123 빌드가 없음 → 404
```

폴더 이름 교체 방식은 **단일 서버** 환경에서는 유효하지만, **다중 서버** 환경에서는 서버 간 동기화 문제가 여전히 남아있었다.

---

## 근본적인 해결 방안들

이 문제를 제대로 해결하려면 아키텍처 자체를 다시 생각해야 한다.

### 방안 1: Blue-Green 배포

두 개의 환경(Blue/Green)을 번갈아가며 사용하는 방식이다. 현재 우리처럼 로컬에서 `git push` → 서버에서 `git pull` → `npm run build` 하는 수동 배포 환경에서도 적용 가능하다.

```
[Blue-Green 배포 개념]
                    ┌─────────────┐
                    │ Load        │
                    │ Balancer    │
                    └──────┬──────┘
                           │
              현재 트래픽 →│
                           ▼
    ┌─────────────────────────────────────────┐
    │              Blue 환경                   │
    │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │ ← 현재 서비스 중
    │  │ 서버 1  │  │ 서버 2  │  │ 서버 3  │  │
    │  │(v1.0.0) │  │(v1.0.0) │  │(v1.0.0) │  │
    │  └─────────┘  └─────────┘  └─────────┘  │
    └─────────────────────────────────────────┘

    ┌─────────────────────────────────────────┐
    │             Green 환경                   │
    │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │ ← 새 버전 배포 및 테스트 중
    │  │ 서버 1  │  │ 서버 2  │  │ 서버 3  │  │
    │  │(v1.1.0) │  │(v1.1.0) │  │(v1.1.0) │  │
    │  └─────────┘  └─────────┘  └─────────┘  │
    └─────────────────────────────────────────┘

    → Green 환경 준비 완료 후 로드밸런서가 Green으로 전환
    → 문제 발생 시 즉시 Blue로 롤백
```

#### 우리 환경에서 적용하는 방법

NAS에 두 개의 폴더를 만들어서 Blue-Green을 구현할 수 있다:

```
NAS (공유 스토리지)
├── /code
│   ├── frontend-blue/     # Blue 환경 (현재 서비스 중)
│   │   ├── .next/
│   │   └── ...
│   ├── frontend-green/    # Green 환경 (새 버전 배포용)
│   │   ├── .next/
│   │   └── ...
│   └── frontend -> frontend-blue  # 심볼릭 링크
```

```bash
# 배포 스크립트 예시 (deploy.sh)
#!/bin/bash
CURRENT=$(readlink /nas/code/frontend)
if [ "$CURRENT" = "/nas/code/frontend-blue" ]; then
    TARGET="frontend-green"
else
    TARGET="frontend-blue"
fi

# 1. 비활성 환경에서 git pull & build
cd /nas/code/$TARGET
git pull origin main
npm run build

# 2. 빌드 성공 확인 후 심볼릭 링크 교체
ln -sfn /nas/code/$TARGET /nas/code/frontend

# 3. 모든 서버의 컨테이너 재시작 (혹은 graceful reload)
docker restart frontend-container

echo "Deployed to $TARGET"
```

| 장점 | 설명 |
|------|------|
| 무중단 배포 | 새 환경이 완전히 준비된 후 전환 |
| 즉시 롤백 | 문제 시 심볼릭 링크만 되돌리면 됨 |
| 빌드 충돌 없음 | 각 환경이 독립적인 폴더 사용 |
| 테스트 가능 | 전환 전 Green 환경에서 테스트 가능 |

---

### 방안 2: Object Storage + CDN 활용 (추천)

정적 파일(JS, CSS, 이미지)을 NCP Object Storage에 올리고 CDN으로 서빙하는 방식이다. **이 방식이 `BUILD_ID` 불일치 문제를 가장 확실하게 해결한다.**

```
[기존 문제 상황]
사용자 → 서버 1 (HTML 응답, BUILD_ID: abc123)
       → 서버 2 (JS 요청, BUILD_ID: xyz789) → 404 에러!

[Object Storage 적용 후]
사용자 → 서버 1 (HTML 응답)
       → CDN (JS/CSS 요청) → 항상 같은 빌드 파일 응답!
```

#### 구현 방법

**1. next.config.js 설정**

```javascript
// next.config.js
const BUILD_ID = process.env.BUILD_ID || Date.now().toString();

module.exports = {
  generateBuildId: async () => {
    // 모든 서버가 같은 BUILD_ID를 사용하도록 환경변수로 지정
    return BUILD_ID;
  },
  assetPrefix: process.env.NODE_ENV === 'production'
    ? `https://cdn.example.com/static/${BUILD_ID}`
    : '',
  // CDN에서 정적 파일을 가져오도록 설정
}
```

**2. 빌드 및 업로드 스크립트**

```bash
#!/bin/bash
# build-and-upload.sh

# 1. BUILD_ID 생성 (타임스탬프 또는 git commit hash)
export BUILD_ID=$(date +%Y%m%d%H%M%S)
# 또는: export BUILD_ID=$(git rev-parse --short HEAD)

# 2. 빌드 실행
npm run build

# 3. 정적 파일을 Object Storage에 업로드
# NCP CLI 사용 예시
ncloud storage cp -r .next/static s3://my-bucket/static/$BUILD_ID/

# 4. BUILD_ID를 파일로 저장 (서버들이 참조)
echo $BUILD_ID > /nas/code/frontend/CURRENT_BUILD_ID
```

**3. 서버 시작 시 BUILD_ID 로드**

```javascript
// server.js 또는 시작 스크립트
const fs = require('fs');
const BUILD_ID = fs.readFileSync('/nas/code/frontend/CURRENT_BUILD_ID', 'utf8').trim();
process.env.BUILD_ID = BUILD_ID;
```

#### 아키텍처

```
[Object Storage + CDN 아키텍처]

                    ┌─────────────┐
                    │ Load        │
                    │ Balancer    │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ 서버 1  │       │ 서버 2  │       │ 서버 3  │
    │ (SSR)   │       │ (SSR)   │       │ (SSR)   │
    └─────────┘       └─────────┘       └─────────┘
         │                 │                 │
         └─────────────────┴─────────────────┘
                           │
                    HTML만 응답
                    (JS/CSS는 CDN URL 참조)
                           │
                           ▼
                    ┌─────────────┐
                    │     CDN     │
                    │ (정적 파일) │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   Object    │
                    │   Storage   │
                    │ /static/    │
                    │   ├─ v1/    │
                    │   ├─ v2/    │
                    │   └─ v3/    │
                    └─────────────┘
```

| 장점 | 설명 |
|------|------|
| BUILD_ID 문제 완전 해결 | 정적 파일이 서버와 분리됨 |
| CDN 캐싱 | 전 세계 엣지에서 빠른 응답 |
| 서버 부하 감소 | 정적 파일 요청이 서버로 안 감 |
| 버전 관리 용이 | 이전 버전 파일도 보존 |
| 롤백 간편 | CURRENT_BUILD_ID만 변경하면 됨 |

---

### 방안 3: Blue-Green + Object Storage 조합 (최적)

두 가지를 조합하면 가장 안정적인 배포가 가능하다:

```bash
#!/bin/bash
# 최종 배포 스크립트

# 1. BUILD_ID 생성
export BUILD_ID=$(date +%Y%m%d%H%M%S)

# 2. 비활성 환경 결정
CURRENT=$(readlink /nas/code/frontend)
if [ "$CURRENT" = "/nas/code/frontend-blue" ]; then
    TARGET="frontend-green"
else
    TARGET="frontend-blue"
fi

# 3. 비활성 환경에서 빌드
cd /nas/code/$TARGET
git pull origin main
npm run build

# 4. 정적 파일 Object Storage 업로드
ncloud storage cp -r .next/static s3://my-bucket/static/$BUILD_ID/

# 5. BUILD_ID 파일 업데이트
echo $BUILD_ID > /nas/code/$TARGET/CURRENT_BUILD_ID

# 6. 환경 전환
ln -sfn /nas/code/$TARGET /nas/code/frontend

# 7. 컨테이너 재시작
docker restart frontend-container

echo "✅ Deployed to $TARGET with BUILD_ID: $BUILD_ID"
```

---

## 각 방안 비교

| 방안 | 복잡도 | 비용 | 다운타임 | 롤백 | BUILD_ID 문제 해결 |
|------|--------|------|----------|------|-------------------|
| Blue-Green | 낮음 | 저장공간 2배 | 거의 없음 | 매우 쉬움 | 부분 해결 |
| Object Storage | 중간 | CDN 비용 발생 | 없음 | 쉬움 | 완전 해결 |
| 조합 | 중간 | CDN + 저장공간 | 없음 | 매우 쉬움 | 완전 해결 |

---

## 결론 및 다음 단계

이번 트러블 슈팅을 통해 배운 것들:

1. **단일 서버에서 잘 되던 것이 다중 서버에서는 안 될 수 있다** - 공유 리소스에 대한 동시 접근 문제를 항상 고려해야 한다.

2. **Sticky Session은 임시방편** - 근본적인 아키텍처 문제를 가리는 것일 뿐, 해결책이 아니다.

3. **빌드 결과물은 불변(Immutable)해야 한다** - 여러 서버가 같은 빌드 결과물을 공유하되, 빌드 중에 변경되면 안 된다.

4. **정적 파일 분리가 핵심** - 서버와 정적 파일을 분리하면 `BUILD_ID` 불일치 문제를 원천 차단할 수 있다.

다음 스텝은 **Object Storage + CDN**을 먼저 적용해서 `BUILD_ID` 문제를 완전히 해결하고, 이후 **Blue-Green 배포**까지 적용해서 무중단 배포 환경을 구축하는 것이다. 이 과정도 정리해서 후속 글로 작성할 예정이다.

---

## 참고 자료

- [Next.js Deployment Documentation](https://nextjs.org/docs/deployment)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [NCP Load Balancer - Sticky Session](https://guide.ncloud-docs.com/docs/networking-loadbalancer-classiclbconsole)
- [Blue-Green Deployment Pattern](https://martinfowler.com/bliki/BlueGreenDeployment.html)
