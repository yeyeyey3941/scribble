---
layout: post
title:  "FastAPI의 의존성 주입(DI)과 미들웨어 패턴"
date:   2025-08-06 20:00:00 +0900
categories: python, fastapi
---
# 의존성 주입(Dependency Injection, DI)

## 목차
1. [의존성 주입이란?](#의존성-주입이란)
2. [Django vs FastAPI의 DI 차이점](#django-vs-fastapi의-di-차이점)
3. [코드 예시 비교](#코드-예시-비교)
4. [싱글톤 패턴 vs DI vs FastAPI Depends](#싱글톤-패턴-vs-di-vs-fastapi-depends)
5. [미들웨어 패턴과 DI](#미들웨어-패턴과-di)
6. [실제 사용 사례](#실제-사용-사례)
7. [정리 및 결론](#정리-및-결론)

## 의존성 주입이란?

### 기본 개념
**의존성 주입(Dependency Injection)**은 객체가 필요한 의존성을 **직접 생성하지 않고**, **가져오는** 패턴입니다.

- **의존성**: A가 동작하기 위해 B가 **반드시 필요한** 관계
- **주입**: 외부에서 필요한 의존성을 **제공하는** 행위

### 핵심 목적
- **결합도 감소**: 클래스 간의 강한 연결을 느슨하게 만듦
- **재사용성 향상**: 동일한 코드를 다양한 상황에서 활용
- **테스트 용이성**: Mock 객체 주입으로 격리된 테스트 가능
- **유지보수성**: 구현체 변경 시 최소한의 코드 수정

## Django vs FastAPI의 DI 차이점

### Django의 접근 방식
```python
# Django - 프레임워크 중심, DI 기본 지원 없음
def create_post(request):
    svc = PostCreationService()  # 직접 생성
    svc.create_post()
```

**특징:**
- **전역 객체와 모듈 임포트** 방식
- **미들웨어 시스템**으로 request 객체에 정보 첨부

### FastAPI의 접근 방식
```python
# FastAPI - DI를 핵심 기능으로 제공
@app.get("/items/")
async def read_items(commons: Annotated[dict, Depends(common_parameters)]):
    return commons
```

**특징:**
- **의존성 주입을 핵심 기능**으로 제공
- **요청 단위 싱글톤** 자동 관리
- **의존성 체인** 자동 해결
- **타입 힌팅** 완전 지원

## 코드 예시 비교

### 의존성 주입 없는 코드 (결합도 높음)

```python
class NotificationService:
    def __init__(self):
        self.service = EmailService()  # 직접 객체 생성 (강한 결합)

    def notify(self, message):
        self.service.send(message)

# 사용
manager = NotificationService()
manager.notify("서비스에 가입해주셔서 감사합니다.")
```

**문제점:**
- `NotificationService`가 `EmailService`에 강하게 묶여 있음
- 다른 구현체(SMS, Slack 등)로 교체하려면 클래스 내부를 수정해야 함
- 테스트 시 mocking이 어렵고 유연성 부족

### 의존성 주입 사용한 코드 (결합도 낮음)

```python
class NotificationService:
    def __init__(self, service):
        self.service = service  # 외부에서 주입된 의존성

    def notify(self, message):
        self.service.send(message)

# 사용
email_service = EmailService()
manager = NotificationService(email_service)  # 의존성 주입
manager.notify("가입해주셔서 감사합니다.")

# 쉬운 구현체 교체
sms_service = SMSService()
manager = NotificationService(sms_service)  # DI로 구조 바꾸지 않고 적용
```

**장점:**
- `NotificationService`는 어떤 "알림 서비스"든 받을 수 있음
- 다른 구현체로 쉽게 교체 가능
- 테스트 용이성 (Mock 객체 주입 가능)

## 싱글톤 패턴 vs DI vs FastAPI Depends

### 비교표

| 패턴 | 인스턴스 생성 | 메모리 사용 | 테스트 용이성 | 결합도 | 생명주기 | 
|------|-------------|------------|-------------|-------|---------|
| **싱글톤** | 앱 시작시 1번 | 최소 | 어려움 | 높음 | 앱 전체 | 
| **Not using DI** | 매번 새로 생성 | 많음 | 어려움 | 높음 | 즉시 소멸 | 
| **FastAPI Depends** | 요청당 1번(default) | 중간 | 쉬움 | 낮음 | 요청 범위 | 

> FastAPI Depends는 기본적으로 요청당 1번 인스턴스를 생성 But Depends(use_cache=False) 옵션을 통해 요청당 여러번 인스턴스를 생성 가능

### 싱글톤 패턴 예시
```python
class SingletonService:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

### FastAPI Depends 예시
```python
def get_service():
    return EmailService()

@app.get("/notify")
def notify(service: EmailService = Depends(get_service)):
    service.send("환영합니다!")
    return {"message": "알림 전송 완료"}
```

### 핵심 차이점
- **싱글톤**: 앱 전체에서 하나의 인스턴스 공유 (상태 공유 위험)
- **FastAPI Depends**: 요청-범위 캐싱 자동 관리 (안전한 격리)

## 미들웨어 패턴과 DI

### 미들웨어에서 DI가 필수인 이유

#### 1. **미들웨어 체인의 복잡성**
```
Request → [로깅] → [인증] → [캐싱] → [비즈니스 로직] → Response
```
- 여러 미들웨어가 같은 서비스(Logger, DB, Cache 등)를 사용
- 각각 개별 생성하면 메모리 낭비 + 일관성 문제

#### 2. **횡단 관심사(Cross-cutting Concerns)**
- 로깅, 인증, 캐싱 등은 모든 미들웨어에서 필요
- DI 없이는 각 미들웨어마다 동일한 코드 중복

### DI 없는 미들웨어 (문제점)
```python
class AuthMiddleware:
    def __init__(self):
        self.logger = Logger()          # 직접 생성
        self.db = Database()           # 직접 생성

class LoggingMiddleware:
    def __init__(self):
        self.logger = Logger()          # 또 다른 인스턴스!
        self.db = Database()           # 또 다른 연결!
```

### DI 적용한 미들웨어 (해결책)
```python
# DI 컨테이너
container = DIContainer()
logger = ConsoleLogger()  # 하나의 Logger 인스턴스만 생성
container.register(ILogger, logger)

# 미들웨어에 의존성 주입
auth_middleware = AuthMiddleware(
    logger=container.get(ILogger),
    authenticator=container.get(IAuthenticator)
)

logging_middleware = LoggingMiddleware(
    logger=container.get(ILogger)  # 같은 인스턴스 공유
)
```

## 실제 사용 사례

### 1. 데이터베이스 연결 관리
```python
# FastAPI에서 DB 연결 DI
def get_db():
    db = SessionLocal()
    try:
        yield db  # 요청 동안만 유지
    finally:
        db.close()  # 자동 정리

@app.get("/users")
def get_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

### 2. 인증 서비스
```python
def verify_token(token: str = Depends(oauth2_scheme)):
    # 토큰 검증 로직
    return user_id

@app.get("/protected")
async def protected_route(user_id: str = Depends(verify_token)):
    return {"user": user_id}
```

### 3. 환경별 설정
```python
def get_email_service():
    if os.getenv("ENV") == "production":
        return RealEmailService()
    else:
        return FakeEmailService()  # 개발환경에서는 가짜 서비스
```

## FastAPI Depends의 특별한 기능

### 1. 요청 범위 싱글톤
```python
# 같은 요청 내에서는 하나의 인스턴스만 생성됨
@app.get("/complex")
def complex_endpoint(
    service1: EmailService = Depends(get_email_service),
    service2: EmailService = Depends(get_email_service)  # 같은 인스턴스!
):
    print(f"같은 객체? {service1 is service2}")  # True!
```

### 2. 의존성 체인 자동 해결
```python
def get_database():
    return Database()

def get_user_repository(db: Database = Depends(get_database)):
    return UserRepository(db)

def get_user_service(repo: UserRepository = Depends(get_user_repository)):
    return UserService(repo)

@app.get("/users/{user_id}")
def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)  # 자동으로 db → repo → service 생성
):
    return service.get_user(user_id)
```

### 3. 테스트에서 Mock 주입
```python
# 테스트 코드
class MockEmailService:
    def send(self, message):
        print(f"[MOCK] {message}")

def mock_email_service():
    return MockEmailService()

# 테스트 시 의존성 교체
app.dependency_overrides[get_email_service] = mock_email_service
```

## 정리 및 결론

### 의존성 주입의 핵심 가치

#### 1. **아키텍처 설계 도구**
- 단순한 재사용성을 넘어서 전체 시스템 설계에 영향
- 객체 생명주기 관리, 리소스 효율성, 테스트 용이성, 설정 유연성 제공

#### 2. **복잡성 관리**
- 특히 미들웨어 패턴같이 **여러 컴포넌트가 연쇄적으로 실행**되는 상황에서 필수
- 횡단 관심사를 효율적으로 처리

#### 3. **현대적 프레임워크의 필수 요소**
- Django: 전통적 방식, 필요시 별도 라이브러리 사용
- FastAPI: DI를 핵심 기능으로 제공, 현대적 API 개발에 최적화

### 언제 사용해야 할까?

| 상황 | 권장 패턴 |
|------|----------|
| **간단한 유틸리티, 상태 없는 함수** | 일반 함수/클래스 |
| **설정, 캐시, 연결 풀 등 전역적 리소스** | 싱글톤 패턴 |
| **요청별 격리가 필요한 서비스** | FastAPI Depends |
| **복잡한 미들웨어 체인** | DI 패턴 필수 |

### 최종 결론

**의존성 주입**은 현대적 소프트웨어 개발에서 **필수적인 설계 패턴**입니다. 특히:

- **유지보수 가능한 코드** 작성을 위해 필수
- **테스트 주도 개발(TDD)**을 위해 필요
- **마이크로서비스 아키텍처**에서 핵심적 역할
- **FastAPI 같은 현대적 프레임워크**의 강력함을 최대한 활용하기 위해 필요

> DI는 "객체를 외부에서 주입받는다"는 단순한 개념을 넘어서, **전체 애플리케이션의 결합도를 낮추고 유연성을 높이는 아키텍처 설계 철학**입니다.

