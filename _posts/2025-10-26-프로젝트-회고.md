---
layout: post
title:  "Potenup 1st project-retrospective"
date:   2025-10-26 19:00:00 +0900
categories: "project"
---
# Poten-up LMS 프로젝트 회고

## 개요
- 2025-09-15일에 Poten-up backend 과정 시작
- 최종 목표인 LXP 솔루션 개발을 위한 미니프로젝트 첫번째
- 일단 LXP(Learning Experience Platform) 프로젝트 전에 LXP라는 개념이 일상에서 익숙하지 않다고 생각해서, 우리에게 익숙한 LMS(Learning Management System) 부터 구축

## 설계
1. 요구 사항 명세 : 기존 존재하는 LMS 사이트를 참고해서 선정
   - Category : 강의의 종류 / 카테고리 목록을 조회
   - Course : 강의 / 조회한 카테고리에서 강의를 CRUD
     - Section : 강의 내부의 각각의 파트 / 강의 내에서 CRUD
       - Content : 강의 영상이나 강의자료 등의 요소들 / 강의 내에 종속된 section에 종속되어 CRUD
         - Asset : Content에 속하는 파일 형식의 요소 / 강의 생성시 파일을 업로드(파일이기때문에 비동기처리)
     - notice : 강의의 공지사항 / 강의에 대한 공지사항 CRUD
   - user : 강의를 작성하는 강의 제작자 / 강의 생성

2. DB 설계
   1. Category : name / parent(소속 상위 카테고리)
   2. Course : title / user(instructor?) / create_at / updated_at / summary / detail / sections
      1. Section : name / seq / contents
         1. content : name / seq / assets
            1. asset : mime_type / path / original_filename / converted_filename
   3. notice : body / created_at / updated_at
   4. user : name

3. 아키텍쳐 설계 
   1. Course가 생성되어야 Section이 생성가능하고 , Section이 생성되어야 Content가 생성가능
   2. Course가 생성될때 section과 content가 같이 생성되어야하고, section은 Course를 통해 추가하면서 생성되어야하고, content는 section을 통해 추가하며넛 생성되어야한다.
   3. notice 는 Couse와 연관되어있지만, 꼭 Course와 함께 생성할 필요가 없다고 판단
   4. Asset은 Content가 생성되면서, Asset을 비동기로 저장을 넘기는 방식(파일 시스템을 비동기 처리하지않을경우 실패가능성 + 리소스 낭비)이 맞다고 판단
   > 따라서 Course를 Aggregate로 판단 + Course 가 MVP 면서 기능이 가장 많아 Domain을 repository/domain/contoller/service로 분리해서 업무 담당 & 외의 Asset 담당

## 시스템 아키텍처

### 데이터 모델 계층 구조
```
Course (Aggregate Root)
 ├─ Section (1:N)
    └─ Content (1:N)
       └─ Asset (1:N, 비동기 처리)
 ├─ Notice (1:N, 독립적 생성)
 └─ User (강의 제작자)
```

### 비동기 업로드 흐름
```
1. Content 생성 요청
   ↓
2. Asset DTO 생성 및 BlockingQueue에 추가 (PENDING 상태)
   ↓
3. 워커 스레드가 Queue에서 가져옴 (UPLOADING 상태)
   ↓
4. 파일 I/O 시뮬레이션 (sleep)
   ↓
5. 성공 → SUCCESS / 실패 → FAILED
   ↓
6. 실패 시 최대 3회 재시도 (RETRY 상태)
   ↓
7. DB 상태 업데이트
```

### 아키텍처 설계의 핵심 결정
- **Aggregate 패턴**: Course를 Aggregate Root로 설정하여 계층 구조 보장
- **비동기 처리**: Asset 업로드는 메인 스레드 블로킹 방지를 위해 별도 처리
- **도메인 분리**: Course 도메인은 repository/domain/controller/service로 분리, Asset은 별도 관리

## 나의 담당 기능: Asset 비동기 업로드 시스템

### 문제 인식
- **기존 문제점**: 파일 크기가 클 경우 동기식 업로드로 인한 타임아웃 위험
- **리소스 낭비**: 동기 방식의 파일 I/O는 스레드를 블로킹
- **사용자 경험**: 대용량 파일 업로드 시 응답 지연

### 구현 설계
- **Consumer/Producer 패턴** 기반 설계
  - Producer: Asset 업로드 요청을 Queue에 추가
  - Consumer: 워커 스레드가 Queue에서 요청을 가져와 처리
- **BlockingQueue + ExecutorService** 채택
  - Java 기본 동시성 API에 집중
  - CompletableFuture는 더 적합할수 있지만, 기본을 위해 채택하지 않음
- **상태**: `PENDING → UPLOADING → SUCCESS/FAILED → RETRY`

### 핵심 구현 로직

> pseudo코드 시뮬레이션
> 1. Content 생성 시 Asset DTO 생성
> 2. AssetService가 DTO를 사용해 Asset BlockingQueue에 추가
> 3. 워커 스레드 풀(ExecutorService)이 Queue에서 가져와 처리
> 4. sleep()으로 파일 I/O 시뮬레이션 (실제 파일시스템 구현 X)
> 5. 상태 DB 업데이트 : `ACCEPTED(PENDING/UPLOADING/RETRY) -> SUCCESS/FAILED`

### 재시도 로직
- 실패 시 최대 3회 자동 재시도
- 최종 실패 시 DB에 상태 업데이트
- 사용자가 수동으로 재시도 요청 가능

> **Note**: 실제 파일시스템까지는 구현하지 않고, 비동기 처리 구조와 상태 관리를 중점으로 

## 기술 선택의 트레이드오프

### BlockingQueue + ExecutorService vs CompletableFuture

**선택**: BlockingQueue + ExecutorService

**선택 이유**:
- Java 기본 동시성 API에 집중하여 기본기를 탄탄히 하고 싶었음
- Producer-Consumer 패턴을 직접 구현하며 이해하고 싶었음
- BlockingQueue는 명시적이고 제어 가능한 대기열 관리

**CompletableFuture의 장점**:
- 더 현대적이고 풍부한 API
- 함수형 스타일로 구현 가능
- 체이닝과 조합이 용이

**다음에는**: CompletableFuture도 시도해보고 싶음

### 비동기 처리 구현 방식

**선택**: sleep() 기반 시뮬레이션

**이유**: 
- Java 초기 단계에서 파일시스템 구현보다 비동기 구조에 집중
- 상태 관리와 재시도 로직에 더 관심

**한계점**:
- 실제 파일 I/O를 구현하지 않아 실제 성능 측정 불가
- 실제 환경에서의 파일 처리 예외 케이스 미검증

**개선 방향**: 다음 프로젝트에서는 실제 파일 시스템과 함께 구현

## 학습 내용

### Python vs Java 멀티스레딩

**기존 경험 (Python)**:
- GIL(Global Interpreter Lock) 때문에 멀티스레딩 불가
- CPU-bound 작업은 multiprocessing 사용

**Java 멀티스레딩**:
- JVM의 GIL 없이 진정한 멀티스레딩 가능
- 여러 스레드가 동시에 CPU 집약적 작업 수행 가능

**구현 시 관찰**:
- 공유 자원 접근이 없는 영역에서는 별도 동기화 불필요
- 각 워커 스레드가 독립적으로 Asset 처리
- BlockingQueue가 내부적으로 동기화 처리

### 코드 품질 향상

리뷰를 통해 배운 내용:

```java
// 기존: 모호한 함수명
retryAble()  // 재시도 가능성 판단인데 의미가 불명확
toDb()       // toString 같은 표준 메서드와 혼동

// 개선: 명확한 의도 전달
canRetry()      // boolean 반환
forDatabase()   // DB용으로 변환 표현
```

**핵심 학습**:
- 함수명은 반환 타입과 역할을 명확히 표현해야 함
- 기존 Java 표준 메서드 네이밍과 겹치지 않게 주의
- Magic Number 대신 상수 사용

### Aggregate 패턴 이해

**처음 접한 개념**: DDD(Domain-Driven Design)의 Aggregate

**이 프로젝트에서**:
- Course를 Aggregate Root로 설정
- Section, Content, Asset이 Course의 일관성 경계 안에 있음
- Course 생성 없이는 하위 엔티티 생성 불가

**배운 점**:
- 도메인 모델링 시 일관성 경계 정의의 중요성
- Aggregate Root를 통한 데이터 무결성 보장  

## 어려웠던 점

### 1. 팀 협업의 어색함
- 기존에 혼자 AI를 통해 개발하거나 주먹구구식으로 개발하는 것이 많아서 팀 단위 협업이 익숙하지 않았음
- 내가 만든 기능을 다른 팀원이 모르는 상황 발생
- 내 기능이 필요한 시점에서 내 기능을 위한 parameter가 없어 구현 불가능
- 서로의 작업 진행 상황을 파악하기 어려웠음

### 2. 개념 이해의 부족
- 웹 개발자를 목표로 하면서, 웹개발에 대한 DDD 같은 개념(like Aggregate) 등을 학습하지않음
- "Course를 Aggregate로 해야 한다"는 대화를 했지만, 처음에는 그 이유를 명확히 이해하지 못함
- 도메인 설계 시 일관성 경계의 중요성을 경험을 통해서야 깨달음

### 3. 소통의 공백
- 회의를 같이 했어도 각자의 영역에 집중하다 보면 필요한 부분을 망각
- 내가 Asset 업로드 기능을 완성했지만, 다른 팀원이 이를 활용하는 방법을 모름
- 내쪽에서 API 사용 방법을 명시하지 않아 사용되지 않음
- 확인도 하지 않고, 요청도 하지 않아서 기능이 사장됨

### 4. 구체화의 미흡
- 미리 어떻게 할지 더 자세히 구체화했어야 했는데, 애매한 부분을 넘어감
- "비동기로 처리한다"는 결정만 하고, 상태 관리나 에러 처리에 대한 상세 논의 부족
- 예외 케이스에 대한 정의가 명확하지 않음
- 어느 정도까지 구현해야 하는지 불명확

## 후회하는 점 + 구체적 개선 계획

### 1. 적극적인 프로젝트 태도와 공유

**문제**: 내 기능에 대해 팀에 충분히 공유하지 않아 사용되지 않음

**구체적 해결책**:
- 기능 완성 시점에 README나 API 문서 작성
- 사용 예시 코드 포함하여 PR에 첨부
- 기능 완성 알림을 슬랙/팀 채널에 명시적으로 전달
- 주간 회고에서 내가 완성한 기능과 사용 방법 공유

### 2. 상세한 사전 구체화

**문제**: 애매한 부분을 넘어가며 추측하며 개발

**구체적 개선책**:
- 설계 단계에서 모든 시나리오 작성
- 예외 케이스 정의 문서화
  - 예: "재시도 3회 실패 시 어떻게 할지?"
  - 예: "동시에 같은 Asset 업로드 시?"
- 구현 전에 팀원들과 검토하며 합의
- 중간 체크포인트에서 미흡한 부분 재구체화

### 3. 지속적이고 인지 가능한 소통

**문제**: PR 리뷰 수정 후 리뷰어에게 알리지 않음

**구체적 개선책**:
- 리뷰 코멘트에 "수정 완료했습니다" 라고 명시적 답변
- 큰 변경사항이 있으면 별도 메시지 전달
- 팀 소통 채널에서 진행 상황 공유
- 블로킹 이슈는 즉시 공유

### 4. 개념 이해의 선제적 학습

**문제**: 프로젝트 중에 개념을 처음 배워 적용이 늦음

**구체적 개선책**:
- 프로젝트 시작 전, 사용할 주요 패턴/개념 학습
  - 예: DDD, Aggregate 패턴
  - 예: 비동기 처리 패턴
- 기술 선택 시 이론부터 학습 후 구현
- 모르는 개념은 정리하고 블로깅 하는 습관에 대한 필요성

## 다음 프로젝트 시 목표

### 협업 측면
1. **지속적 소통**: 주 2~3회 이상 진행 상황 공유
2. **적극적 태도**: 내 기능 완성 시 팀 공유 필수
3. **명시적 요청**: 필요한 부분은 망설이지 말고 요청

### 기술 측면
1. **상세한 구체화**: 설계 단계에서 시나리오 완전성 확보
2. **선제적 학습**: 프로젝트 시작 전 핵심 개념 학습
3. **단계적 구현**: 기본 구현 → 테스트 → 개선의 사이클

### 코드 품질
1. **명확한 네이밍**: 리뷰 피드백을 사전에 고려
2. **문서화**: 기능 완성 시 README/API 문서 작성
3. **테스트**: 주요 로직은 최소한 단위 테스트 작성
4. **건설적인 코드 리뷰**: 
   - 내가 받은 코드 리뷰의 값진 경험을 바탕으로 팀원들에게도 동일하게 제공
   - 피드백이 단순 비판이 아닌, 성장을 위한 제안이 되도록
   - 리뷰를 잘 하기 위해 더 많이 알고, 더 자세히 알아야 겠다는 생각

## 프로젝트 성과

- **구현 기간**: 약 3주
- **핵심 성과**: Asset 비동기 업로드 시스템 구현
  - BlockingQueue + ExecutorService 기반 Producer-Consumer 패턴
  - 상태 전이 관리 (PENDING → UPLOADING → SUCCESS/FAILED → RETRY)
  - 최대 3회 자동 재시도 로직
- **학습 성과**: Python vs Java 멀티스레딩 차이, DDD Aggregate 패턴 이해
- **코드 리뷰**: 5건 이상 피드백 반영 (함수명, Magic Number 등)

