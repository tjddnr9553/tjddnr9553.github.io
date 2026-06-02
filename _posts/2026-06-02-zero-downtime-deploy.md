---
title: "NCP Container Registry로 구현한 무중단 배포 (feat. Auto Scaling)"
author: tjddnr9553
date: 2026-06-02 09:00:00 +0900
categories: [DevOps, Trouble Shooting]
tags: [NCP, Next.js, Docker, Container Registry, Load Balancer, Auto Scaling, DevOps]
---

## 서론

[지난 글](/posts/trouble-shooting/)에서 다중 서버 환경의 `BUILD_ID` 불일치 문제를 다뤘다. 당시 결론은 "Object Storage + CDN을 적용하고, Blue-Green 배포까지 가서 무중단 배포 환경을 구축하겠다"였다.

이번 글은 그 후속편이다. 다만 접근 방식이 조금 바뀌었다. **NAS에 빌드 결과물을 공유하던 구조 자체를 버리고, 이미지를 Container Registry로 관리하는 방식으로 전환**했다.

핵심은 단순하다.

- 빌드는 **로컬 테스트 서버**에서 한 번만 한다.
- 그 결과물을 **이미지로 굳혀서** NCP Container Registry에 올린다.
- 각 서버는 빌드하지 않고 **완성된 이미지를 받아서 실행**만 한다.

지난번 문제의 근본 원인이 "여러 서버가 같은 NAS에서 동시에 빌드하다 충돌"이었으니, 빌드를 한 곳으로 몰고 결과물을 불변(Immutable) 이미지로 만들면 그 문제는 원천 차단된다.

---

## 기존 문제 복습

지난 글에서 겪었던 핵심 문제를 한 줄로 요약하면:

> 두 서버가 같은 NAS의 `.next` 폴더를 동시에 덮어쓰면서 `BUILD_ID`가 깨졌다.

그리고 빌드 타이밍이 서버마다 어긋나면, 한 서버는 신버전 HTML을 주고 다른 서버는 구버전 JS를 주면서 404가 터졌다.

이 문제의 본질은 **"빌드 결과물이 가변(mutable)이고, 서버마다 따로 빌드한다"**는 데 있었다. 그렇다면 빌드를 한 번만 하고, 그 결과물을 모든 서버가 똑같이 받아쓰면 된다. 바로 **컨테이너 이미지**가 그 역할을 한다.

---

## 새로운 배포 구조

### 전체 흐름

```
[로컬 테스트 서버]
    │ 1. docker build (이미지 빌드)
    │ 2. docker push
    ▼
┌─────────────────────────────┐
│   NCP Container Registry    │  ← 완성된 이미지 저장소
│   myapp/frontend:latest     │
└──────────────┬──────────────┘
               │ 3. docker login + docker pull
               ▼
        ┌──────────────┐
        │   메인 서버    │  ← 최신 이미지 pull 후 교체
        └──────────────┘
```

빌드는 로컬 테스트 서버에서 단 한 번. 그 결과물을 이미지로 만들어 Registry에 올리고, 실제 서버들은 그 이미지를 받아 실행만 한다. 이제 어떤 서버에서도 `npm run build`를 돌리지 않으니, 동시 빌드로 인한 충돌은 발생할 수 없다.

### 1단계: 로컬에서 빌드 후 Push

```bash
#!/bin/bash
# build-and-push.sh (로컬 테스트 서버)

# 이미지 태그 (커밋 해시 또는 타임스탬프)
TAG=$(git rev-parse --short HEAD)
REGISTRY=myregistry.kr.ncr.ntruss.com

# 1. 이미지 빌드
docker build -t $REGISTRY/frontend:$TAG -t $REGISTRY/frontend:latest .

# 2. Registry 로그인
docker login $REGISTRY -u $NCP_ACCESS_KEY -p $NCP_SECRET_KEY

# 3. Push
docker push $REGISTRY/frontend:$TAG
docker push $REGISTRY/frontend:latest

echo "✅ Pushed frontend:$TAG to NCP Container Registry"
```

### 2단계: 메인 서버에서 Pull 후 교체

```bash
#!/bin/bash
# deploy.sh (메인 서버)

REGISTRY=myregistry.kr.ncr.ntruss.com

# 1. Registry 로그인
docker login $REGISTRY -u $NCP_ACCESS_KEY --password-stdin $NCP_SECRET_KEY

# 2. 최신 이미지 받기
docker pull $REGISTRY/frontend:latest

# 3. 프론트 컨테이너 교체 (아래 무중단 전략 참고)
docker compose up -d frontend
```

---

## 메인 서버 무중단 교체 전략

여기서부터가 이번 작업의 핵심이다. 메인 서버의 프론트 컨테이너를 새 이미지로 교체하는 동안 서비스가 끊기면 안 된다. 이때 **오토스케일링 서버**와 **로드밸런서의 헬스 체크**를 활용했다.

### 시나리오

전제: 평소에는 메인 서버 + 오토스케일링 서버(최소 1대)가 함께 트래픽을 받고 있다.

```
[배포 전: 정상 상태]
                ┌─────────────┐
                │     LB      │
                └──────┬──────┘
            ┌──────────┴──────────┐
            ▼                     ▼
    ┌──────────────┐      ┌──────────────┐
    │   메인 서버    │      │   오토스케일링   │
    │  (healthy)   │      │  (healthy)   │
    └──────────────┘      └──────────────┘
       양쪽 모두 트래픽 수신
```

**1) 메인 서버 프론트 컨테이너 stop**

```
[메인 서버 교체 시작]
                ┌─────────────┐
                │     LB      │
                └──────┬──────┘
            ┌──────────┴──────────┐
            ✕                     ▼
    ┌──────────────┐      ┌──────────────┐
    │   메인 서버    │      │   오토스케일링  │
    │ (unhealthy)  │      │  (healthy)   │ ← 모든 트래픽 여기로
    └──────────────┘      └──────────────┘
```

메인 서버에서 프론트 컨테이너를 stop하면, LB가 헬스 체크 실패를 감지하고 **트래픽을 오토스케일링 서버로만** 보낸다. 이 짧은 순간 동안 오토스케일링 서버가 모든 트래픽을 감당한다.

**2) 메인 서버에서 새 이미지로 Compose up**

```
[메인 서버 재기동]
    ┌──────────────┐
    │   메인 서버    │  ← docker compose up
    │ (starting…)  │     새 이미지로 빠르게 기동
    └──────────────┘
         │
         ▼ healthy 판정
    ┌──────────────┐
    │   메인 서버    │
    │  (healthy)   │ ← LB가 다시 트래픽 분배
    └──────────────┘
```

stop과 동시에 새 이미지로 빠르게 `compose up`을 한다. 컨테이너가 뜨고 헬스 체크를 통과하면, LB는 다시 양쪽으로 트래픽을 보낸다.

여기까지만 보면 깔끔하다. 하지만 문제가 하나 남아있었다.

---

## 새로운 문제: 버전 불일치

메인 서버는 방금 **최신 이미지**로 교체했다. 그런데 오토스케일링 서버는?

```
[배포 직후]
    ┌──────────────┐      ┌──────────────┐
    │    메인 서버   │      │    오토스케일링  │
    │  frontend    │      │   frontend   │
    │   v2 (최신)   │      │    v1 (구버전) │ ← 그대로!
    └──────────────┘      └──────────────┘
```

오토스케일링 서버는 **이미 떠 있던 구버전 이미지 그대로** 돌아가고 있다. 결국 LB 뒤에 **서로 다른 버전의 프론트가 2개** 존재하게 된다. 지난 글에서 겪었던 그 버전 불일치 문제가, 형태만 바뀌어 다시 나타난 것이다.

```
사용자 A → 메인 서버 (v2 HTML, v2 청크 참조)
사용자 A → 오토스케일링 서버 (v2 청크 요청) → v1만 있음 → 404
```

---

## 해결: Launch Configuration 갱신 + 강제 재기동

오토스케일링 서버는 직접 `docker pull`을 해주기 애매하다. 언제 스케일 인/아웃 될지 모르고, 새로 뜨는 서버는 **Launch Configuration의 init script**를 따라 기동되기 때문이다. 그래서 두 가지를 같이 했다.

### 1) init script / Launch Configuration 수정

오토스케일링 그룹의 **Launch Configuration**에 들어가는 init script를 수정해서, 새 서버가 뜰 때 항상 최신 이미지를 받아서 올라오도록 했다.

```bash
#!/bin/bash
# Auto Scaling - Launch Configuration init script

REGISTRY=myregistry.kr.ncr.ntruss.com

# 1. Registry 로그인
docker login $REGISTRY -u $NCP_ACCESS_KEY --password-stdin $NCP_SECRET_KEY

# 2. 최신 이미지 pull
docker pull $REGISTRY/frontend:latest

# 3. 새 이미지로 컨테이너 기동
cd /app
docker compose up -d
```

이제 **앞으로 새로 뜨는 오토스케일링 서버는 자동으로 최신 이미지**로 올라온다.

### 2) 실행 중인 오토스케일링 서버 강제 중지

문제는 **이미 떠 있는** 구버전 오토스케일링 서버다. init script는 새로 뜨는 서버에만 적용되니, 기존 서버는 여전히 구버전이다.

그래서 실행 중이던 오토스케일링 서버를 **강제로 중지**했다. 우리는 오토스케일링 그룹이 **최소 1대를 항상 유지**하도록 설정해뒀기 때문에, 강제 중지하면 그룹이 곧바로 새 서버를 하나 띄운다. 그리고 그 새 서버는 방금 수정한 init script를 따라 **최신 이미지**로 올라온다.

```
[강제 중지 → 자동 재기동]

   강제 중지            최소 1대 유지 정책
┌──────────────┐         ┌──────────────┐
│   오토스케일링  │     ✕    │  오토스케일링  │
│  v1 (구버전)   │ ──────▶ │   (재기동…)   │
└──────────────┘         └──────┬───────┘
                                │ init script
                                ▼
                         ┌──────────────┐
                         │   오토스케일링  │
                         │   v2 (최신)   │
                         └──────────────┘

   ※ 이 동안 메인 서버(v2)가 트래픽을 감당
```

새 오토스케일링 서버가 뜨는 동안에는 **이미 최신 버전으로 교체된 메인 서버**가 트래픽을 받아준다. 그리고 새 서버가 healthy해지면, 이제 **양쪽 모두 최신 버전(v2)**으로 서비스하게 된다.

```
[최종 상태: 버전 일치]
    ┌──────────────┐      ┌──────────────┐
    │    메인 서버   │      │   오토스케일링   │
    │  frontend    │      │  frontend    │
    │   v2 (최신)   │      │   v2 (최신)   │ ← 일치!
    └──────────────┘      └──────────────┘
```

---

## 전체 배포 순서 정리

```
1. [로컬]   docker build → Container Registry로 push
2. [메인]   docker login → docker pull (최신 이미지)
3. [메인]   프론트 컨테이너 stop
              → LB가 unhealthy 감지
              → 트래픽을 오토스케일링 서버로
4. [메인]   새 이미지로 compose up
              → healthy 되면 LB가 다시 양쪽 분배
5. [ASG]    Launch Configuration / init script 수정
              (docker login + pull + compose up)
6. [ASG]    실행 중인 오토스케일링 서버 강제 중지
              → 최소 1대 유지 정책으로 자동 재기동
              → 새 init script로 최신 이미지 기동
              → (이 동안 메인 서버가 트래픽 감당)
7. [완료]   메인 + 오토스케일링 모두 최신 버전으로 서비스
```

---

## 마치며

솔직히 이건 아직 **완벽한 무중단 배포는 아니다.**

- 메인 서버 프론트를 stop하는 짧은 순간, 트래픽이 오토스케일링 서버 한쪽으로 쏠린다.
- 오토스케일링 서버를 강제 중지하고 새로 뜨는 동안은 메인 서버 한쪽만 트래픽을 받는다.
- 이 과정의 타이밍이 어긋나면 잠깐의 지연이나 버전 불일치가 노출될 수 있다.

그럼에도 의미가 있었던 건, **사용자에게 "점검 중" 페이지를 최대한 보여주지 않게 됐다**는 점이다. 예전처럼 배포할 때마다 흰 화면이 뜨거나 서비스가 통째로 죽는 일은 사라졌고, 적어도 한쪽 서버는 항상 살아서 트래픽을 받아주는 구조가 됐다.

빌드를 한 곳으로 몰고 이미지를 불변 단위로 다루기 시작하니, 지난 글에서 그렇게 골치 아프던 `BUILD_ID` 불일치 문제도 자연스럽게 풀렸다. 남은 버전 불일치는 "최신 이미지를 모든 서버에 어떻게 동시에 반영하느냐"의 문제였고, init script 갱신 + 강제 재기동으로 일단 메꿨다.

다음 목표는 이 수동 순서를 **CI/CD 파이프라인으로 자동화**하고, 진짜 **Rolling Update / Blue-Green**으로 발전시켜서 트래픽 쏠림 구간까지 없애는 것이다. 그 과정도 정리해서 또 글로 남길 예정이다.

---

## 참고 자료

- [NCP Container Registry 사용 가이드](https://guide.ncloud-docs.com/docs/container-registry-overview)
- [NCP Auto Scaling - Launch Configuration](https://guide.ncloud-docs.com/docs/compute-autoscaling-overview)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [NCP Load Balancer - Health Check](https://guide.ncloud-docs.com/docs/networking-loadbalancer-lbtargetgroup)