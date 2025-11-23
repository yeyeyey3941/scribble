---
layout: post
title:  "2차 LXP 프로젝트 회고 - Learning 도메인 구현과 성장의 기록"
date:   2025-11-23 19:00:00 +0900
categories: "project"
---

# 2차 LXP 프로젝트 회고

## 개요

Poten-up LXP(Learning Experience Platform) 프로젝트에서 **Learning 도메인**을 담당하여 설계부터 구현까지 전 과정을 경험했다.

- **담당 범위**: Learning 도메인 구현 + 후반부 통합 작업
- **추가 작업**: Course 도메인 보조 작업 및 마이페이지 구현
- **기간**: 프로젝트 전체 기간 중 주도적 역할 수행

## Learning 도메인의 역할

Learning 도메인은 LXP 시스템에서 **학습자의 수강 이력과 진도를 관리**하는 핵심 도메인이다.

**주요 역할**:
- 사용자의 강좌 수강 신청 및 진도 추적
- Catalog 도메인(Course, Lecture)과 Identity 도메인(User)을 연결하는 중심축
- 학습 경험의 핵심 데이터 제공

## 나의 담당 기능

### 1. Learning 도메인 구현

#### Enrollment (수강 등록) 시스템
- **도메인 모델 설계**
  - `Enrollment` 엔티티: User와 Course를 연결하는 핵심 엔티티
  - `LectureProgress` 엔티티: 개별 강의 완료 이력 추적
  - `EnrollmentStatus` Enum: ACTIVE, COMPLETED, CANCELLED 상태 관리
  - UniqueConstraint 설정: `(user_id, course_id)` - 동일 강좌 중복 등록 방지

- **수강 등록 비즈니스 로직**
  - 중복 등록 방지: 이미 등록된 강좌인지 검증
  - 강좌 상태 검증: `PUBLISHED` 상태의 강좌만 등록 가능
  - 초기 진도율 0%로 설정
  - PaymentItem과 1:1 연결 (결제 후 활성화)


#### 진도 관리 시스템
- **LectureProgress 추적**
  - Enrollment와 1:N 관계
  - 각 강의별 완료 시점 기록
  - UniqueConstraint: `(enrollment_id, lecture_id)` - 중복 완료 방지

- **진도율 자동 계산**
  - 완료한 강의 수 / 전체 강의 수 × 100
  - 강의 완료 시마다 자동으로 업데이트
  - 100% 도달 시 자동으로 COMPLETED 상태로 변경


- **상태 전이 관리**
  - ACTIVE → COMPLETED: 진도율 100% 도달 시 자동 변환
  - completedAt 자동 기록

#### Service Layer 설계
- **LearningService 구현**
  - `registerEnrollment()`: 수강 등록 (결제 후 호출)
  - `completeLecture()`: 강의 완료 처리
  - `getEnrollmentProgress()`: 개별 수강 진도 조회
  - `getUserEnrollmentsWithProgress()`: 사용자의 모든 수강 내역 조회

- **Repository 설계**
  - `EnrollmentRepository`: User와 Course로 조회 가능
  - `findByUserIdAndCourseId()`: 중복 검증 및 진도 조회
  - `findByUserId()`: 사용자의 수강 목록


### 2. 후반부 통합 작업

#### 마이페이지 구현
- **계획 변경 대응**
  - 초기 계획: `/enrollment/my` (Learning 도메인 내부)
  - 변경 후: `/my_page` (공통 컨트롤러로 이동)
  - 이유: 수강 목록뿐 아니라 개설 강좌, 게시글 등 통합 필요

- **마이페이지 기능**
  - 수강 중인 강좌 목록 + 진도율 표시
  - 진도바(Progress Bar) UI로 시각화
  - 각 강좌 클릭 시 강좌 상세 페이지로 이동
  - 내가 개설한 강좌 목록 (Course 도메인과 연동)
  - 내가 작성한 게시글 목록 (Community 도메인과 연동)

- **템플릿 구현**
  - Thymeleaf 기반 동적 렌더링
  - 진도율에 따른 동적 progress bar
  - 빈 목록 처리 및 액션 버튼 제공

#### Course 도메인 보조 작업
- **상황**: 원래 Course와 Learning을 두 명이 함께 담당하기로 했으나, 후반부에 파트너의 다른 업무(보고서 작성)로 인해 Course 작업도 담당

- **Course 상태 관리**
  - `CourseStatus` 개선: DRAFT, PUBLISHED 상태 관리
  - 강좌 개설자의 상태 변경 기능 추가
  - 강좌 활성화(activate)와 공개(publish) 통일

- **강의 개설자 기능**
  - `MyCourseController`: 내가 개설한 강좌 관리
  - 강좌 상태 변경 API 추가
  - 개설자 권한 검증

- **Principal 인증 처리 개선**
  - `CustomUserPrincipal` 활용 일관성 확보
  - `@AuthenticationPrincipal` 적용
  - userId 추출 로직 개선

#### 도메인 간 통합
- **Learning ↔ Catalog 연동**
  - Course 정보 조회를 위한 CourseRepository 의존
  - Lecture 완료 처리 시 Course의 Lecture 목록 검증
  - 진도율 계산 시 Course의 전체 강의 수 활용

- **Learning ↔ Identity 연동**
  - User 정보 조회를 위한 UserRepository 의존
  - Optional User 처리 개선
  - 권한 검증 로직 추가

- **CommonController 구현**
  - 여러 도메인 서비스 통합 호출
  - LearningService, CourseCatalogService, CommunityService 활용
  - 마이페이지에서 통합된 정보 제공

- **URL 매핑 정리**
  - 중복되는 애매한 URL 제거
  - `/enrollment/{enrollmentId}/completeLecture/{lectureId}` 매핑
  - 강의 완료 후 강좌 상세 페이지로 리다이렉트

- **버그 수정**
  - my_page 렌더링 오류: parameter 매핑 네이밍 차이 해결
  - course-detail 페이지 수정 버튼 제거
  - course activate와 전체 코스 통합 (보여주지 않으려는 페이지 숨김)
  - progress 초기화 제거
  - URL 주소 오류 개선

## 핵심 설계 결정

### 1. 도메인 모델 구조

```
Enrollment (Aggregate Root)
├─ User (N:1) - 학습자
├─ Course (N:1) - 수강 강좌
├─ PaymentItem (1:1) - 결제 정보
└─ LectureProgress (1:N) - 강의별 진도
   └─ Lecture (N:1) - 완료한 강의
```

**설계 이유**:
- Enrollment를 Aggregate Root로 설정
- 진도 관리의 일관성 보장: LectureProgress는 반드시 Enrollment를 통해서만 생성
- Cascade.ALL + orphanRemoval로 생명주기 관리

### 2. 비즈니스 로직의 위치

**도메인 엔티티에 위치**:
- `completeLecture()`: Enrollment 엔티티 내부
- `updateProgressPercentage()`: private 메서드로 캡슐화
- `changeComplete()`: 상태 전이 로직

**Service Layer에 위치**:
- 트랜잭션 관리
- 도메인 간 조율 (User, Course 조회)
- 예외 처리 및 검증

**장점**:
- 도메인 로직의 응집도 향상
- 비즈니스 규칙이 엔티티에 명확히 표현됨
- Service는 오케스트레이션 역할에 집중

### 3. 양방향 연관관계 관리

**Enrollment ↔ LectureProgress**:
- `@OneToMany(mappedBy = "enrollment", cascade = CascadeType.ALL, orphanRemoval = true)`
- Enrollment 삭제 시 모든 진도 자동 삭제
- 진도 추가/삭제가 Enrollment를 통해서만 가능

## 학습 내용

### UniqueConstraint를 통한 비즈니스 제약

**Enrollment**: `@UniqueConstraint(columnNames = {"user_id", "course_id"})`
- 동일 사용자가 같은 강좌를 중복 등록할 수 없음
- DB 레벨에서 제약 조건 보장

**LectureProgress**: `@UniqueConstraint(columnNames = {"enrollment_id", "lecture_id"})`
- 동일 강의를 중복 완료할 수 없음
- 진도율 계산의 정확성 보장

**장점**:
- 애플리케이션 레벨 검증 + DB 레벨 보장
- 동시성 이슈에서도 안전
- 비즈니스 규칙이 스키마에 명확히 표현됨

### 도메인 주도 설계(DDD) 실천

**Aggregate 패턴 적용**:
- Enrollment를 Aggregate Root로 설정
- LectureProgress는 Enrollment를 통해서만 접근
- 일관성 경계(Consistency Boundary) 명확화


**배운 점**:
- Aggregate는 트랜잭션 일관성의 단위
- 외부에서는 Aggregate Root를 통해서만 접근
- 비즈니스 불변식(Invariant)을 Aggregate 내부에서 보장


## 개선할 점

### 1. 테스트 작성 부족

**현재 상황**:
- 단위 테스트 거의 없음
- 통합 테스트도 부족
- 수동 테스트에 의존

**개선 방향**:
- 핵심 비즈니스 로직에 대한 단위 테스트 작성
  - `Enrollment.completeLecture()` 테스트
  - `updateProgressPercentage()` 로직 테스트
  - 예외 상황 테스트
- Service Layer 통합 테스트
  - 트랜잭션 롤백 검증
  - 도메인 간 연동 테스트


### 2. 도메인 간 통신 인터페이스 정의

**현재 상황**:
- Repository를 직접 주입하여 사용
- 도메인 간 강한 결합
- 변경 시 여러 곳 수정 필요

**개선 방향**:
- 도메인 서비스 인터페이스 정의
- Repository 대신 Service를 통해 접근
- 각 엔티티의 의존성을 낮추는 방향

### 3. 점진적인 학습과 우선순위 설정

**현재 상황**:
- 기본기보다 겉으로 보이는 형식에 집중
- 문서화의 본질보다 외형에 치중
- 우선순위 판단의 어려움

**개선 방향**:
- 핵심 기능 먼저, 문서화는 점진적으로
- 기초부터 차근차근 학습
- 강사님과 팀원의 피드백 적극 수용

## 잘한 점

### 1. 데이터 무결성 보장

**방법**:
- UniqueConstraint로 중복 방지
- 엔티티 내부에서 검증 로직 수행
- Cascade와 orphanRemoval로 일관성 유지

**구체적 적용**:
- 동일 강좌 중복 등록 불가
- 동일 강의 중복 완료 불가
- 진도율 계산 자동화로 데이터 불일치 방지

**결과**:
- 데이터 정합성 문제 없음
- 예상치 못한 버그 최소화

### 2. 유연한 역할 대응

**상황**:
- Course 도메인 보조 작업 수행
- 마이페이지 계획 변경 대응
- 후반부 통합 작업 주도

**긍정적 결과**:
- 전체 시스템 이해도 향상
- 도메인 간 연동 경험
- 팀 프로젝트 완수

**배운 점**:
- 유연한 마인드의 중요성
- 변화에 대한 적응력
- 팀워크의 의미

## 개인적인 성찰

### 정리와 기록의 중요성

**깨달은 점**:
- 정리와 기록이 협업의 핵심임을 인식
- 팀원들의 다른 생각을 하나로 **정리**하고 **기록**하는 과정의 중요성
- 통일된 방향성이 명확한 프로젝트 진행의 기반

**현실**:
- 정리와 기록에 대한 어려움
- 중요성을 인식하면서도 실천의 부족

### 형식에 치우친 초기 접근

**문제점**:
- 핵심 기능 구현보다 문서 정리에 집중
- 본질보다는 외형에 치중한 문서화
- 기초 없이 좋아 보이는 형식만 모방
- 점진적 학습이 아닌 성급한 적용 시도

**비유**:
- 극한의 개념 없이 미분을 시도하는 것과 같은 접근
- 기반 없는 표면적 학습의 한계

### 피드백을 통한 우선순위 재정립

**계기**:
- 강사님의 우선순위에 대한 피드백
- 내 생각에 갇혀 본질을 놓친 습관 발견

**깨달음**:
- 중요한 것과 급한 것의 구분 필요
- 객관적 시각의 중요성
- 기초부터 차근차근 쌓아가는 학습의 필요성

### 앞으로의 방향

**다짐**:
- 애매한 이해보다 확실한 이해 추구
- 복습과 정리의 반복
- 완벽하지 않더라도 제대로 된 정리 습관화

아직 과정이 끝나지 않았기에, 앞으로도 꾸준히 개선해 나갈 것이다.

## 프로젝트 성과

### 핵심 성과
- **Learning 도메인 완전 구현**
  - Enrollment, LectureProgress 엔티티 설계
  - 수강 등록 및 진도 관리 시스템 구축
  - 자동 진도율 계산 및 상태 전이
  
- **마이페이지 통합**
  - 수강 목록 + 진도율 표시
  - 개설 강좌, 게시글 통합
  - 직관적인 UI 제공

- **도메인 간 연동**
  - Learning - Catalog - Identity 연결
  - 원활한 데이터 흐름 확보

## 다음 프로젝트를 위한 목표

이번 프로젝트의 경험을 바탕으로 다음 프로젝트에서는 다음과 같은 목표를 세웠다.

### 협업 목표
1. **문서화 습관**
   - API 문서 작성
   - 설계 의도 기록
   - README 상세화

2. **적극적 소통**
   - 진행 상황 공유
   - 블로킹 이슈 즉시 알림
   - 코드 리뷰 적극 참여

3. **유연한 대응**
   - 계획 변경 수용
   - 역할 전환 준비
   - 팀 목표 우선

### 학습 목표
1. **DDD 심화**
   - Domain Event 학습
   - Bounded Context 명확화
   - 전술적 패턴 적용

2. **JPA 최적화**
   - Fetch Join 전략
   - Entity Graph 활용
   - 2차 캐시 이해

3. **Spring 심화**
   - AOP 활용
   - Transaction 전파
   - 비동기 처리

## 결론

이번 프로젝트에서 **Learning 도메인을 처음부터 끝까지 설계하고 구현**하는 경험을 했다. 단순히 CRUD를 구현하는 것이 아니라, **비즈니스 로직을 도메인 모델에 담아내고**, **데이터 무결성을 보장하는 구조**를 만드는 것이 얼마나 중요한지 깨달았다.

특히 **Aggregate 패턴**을 실제로 적용하면서, 책에서 배운 이론이 실전에서 어떻게 동작하는지 체감할 수 있었다. Enrollment를 통해서만 LectureProgress를 관리함으로써, 진도율 계산의 정확성과 데이터 일관성을 보장할 수 있었다.

후반부에는 예상치 못한 **역할 변경**과 **계획 수정**을 경험했다. 처음에는 당황스러웠지만, 오히려 **전체 시스템을 이해**하고 **도메인 간 연동**을 직접 경험하는 좋은 기회가 되었다. 유연한 대응 능력의 중요성을 배웠다.

개인적으로는 **정리와 기록의 어려움**을 실감했다. 문서화의 중요성을 알면서도 형식에만 치우쳐 본질을 놓친 부분이 아쉽다. 좋아 보이는 것을 따라하기보다는, **기초부터 차근차근 쌓아가는 학습**이 필요함을 깨달았다. 강사님의 피드백을 통해 **우선순위 판단**의 중요성도 느꼈다.

아쉬운 점은 **테스트 작성 부족**과 **API 문서화 미흡**이다. 시간에 쫓겨 핵심 기능 구현에만 집중하다 보니, 품질 관리 측면에서 부족함이 있었다. 다음 프로젝트에서는 **문서화를 습관화**하고, 적절한 수준의 테스트를 적용하여 더 완성도 높은 코드를 작성하고 싶다.

결과적으로 **동작하는 Learning 도메인**을 완성했고, **실제 사용 가능한 마이페이지**를 구현했다. 무엇보다 **도메인 주도 설계**를 실전에 적용하며 배운 것들이 앞으로의 개발에 큰 자산이 될 것이라 확신한다. 

애매하게 알기보다 확실히 이해하고, 모자라 보이더라도 제대로 정리하면서, 한 걸음씩 성장해 나가겠다.

