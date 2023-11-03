# 3. 코드 구성하기

## 계층으로 구성하기

```jsx
buckapl
| ---domain
| | -----Account
| | -----Activity
| | -----AccountRepository
| | -----AccountService
| ---persistence
| | -----AccountRepositoryImpl
| ---web
| | -----AccountController
```

### 문제점

1. 계층으로 코드를 구성하면 기능적인 측면들이 섞이기 쉽다.
2. 애플리케이션의 기능 조각(functional slice)이나 특성(feature)을 구분 짓는 패키지 경계가 업다.
    - 서로 연관되지 않은 기능들끼리 예상하지 못한 부수효과를 일으킬 수 있는 클래스들의 엉망진창 묶음으로 변모할 가능성이 크다.
3. 애플리케이션이 어떤 유스케이스들을 제공하는지 파악할 수 없다.
4. 패키지 구조를 통해서는 우리가 목표로 하는 아키텍처를 파악할 수 없다.

<br>

## 기능으로 구성하기

```jsx
buckapl
| ---account
| | -----Account
| | -----Activity
| | -----AccountRepository
| | -----SendMoneyService
| | -----AccountRepositoryImpl
| | -----AccountController
```

### 장점

1. 패키지 경계를 package-private 접근 수준과 결합하면 각 기능 사이의 불필요한 의존성을 방지할 수 있다.

### 문제점

1. 기능을 기준으로 코드를 구성하면 기반 아키텍처가 명확하게 보이지 않는다.

<br>

## 아키텍처적으로 표현력 있는 패키지 구조

```jsx
buckapl
| ---account
| | -----adapter
| | | ----- in
| | | | ----web
| | | | | ----AccountController
| | | -----out
| | | | ----persistence
| | | | | ----AccountPersistenceAdapter
| | | | | ----SpringDataAccountRepository
| | ----domain
| | | -----Account
| | | -----Activity
| | ----application
| | | -----SendMoneyService
| | | -----port
| | | | ---- in
| | | | | ----SendMoneyUseCase
| | | | ----out
| | | | | ----LoadAccountPort
| | | | | ----UpdateAccountStatePort
```

### 핵사고날 아키텍처 패키지 구조

Account와 관련된 유스케이스는 모두 account 패키지 안에 있다.

* domain
    * 도메인 모델 (Account)
* application
    * 도메인 모델을 둘러싼 서비스 계층 (SendMoneyService)
    * 인커밍 포트 인터페이스 (SendMoneyUseCase)
    * 아웃고잉 포트 인터페이스 (LoadAccountPort, UpdateAccountStatePort)
* adapter
    * 어플리케이션 계층의 인커밍 포트를 호출하는 인커밍 어댑터 (Controller)
    * 어플리케이션 계층의 아웃고잉 포트에 대한 구현을 제공하는 아웃고잉 어댑터 (PersistenceAdapter, Repository)

### 핵사고날 아키텍처 패키지 장점

1. 이러한 패키지 구조는 모델-코드 갭(아키텍처-코드 갭)을 효과적으로 다룰 수 있다.
   >
   💡 [모델-코드 갭(model-code gap)](https://www.ben-morris.com/most-architecture-diagrams-are-useless/#:~:text=George%20Fairbanks%20identified%20what%20he,always%20be%20mapped%20into%20code.)
   > 아키텍처 모델에는 항상 코드에 매핑할 수 없는 추상적인 개념, 기술 선택 및 설계 결정이 혼합되어 있다. 최종 결과는 모델이 정한 구성 요소의 배열과 반드시 일치하지 않는 소스 코드가 될 수 있다.

장점2. 패키지간 접근을 제어할 수 있다.

- package-private인 adapter 클래스
    - 모든 클래스는 application 패키지 내의 포트 인터페이스를 통해 바깥에 호출되기 때문에 adapter는 모두 package-private 접근 수준으로 둬도 된다.
    - 어플리케이션 계층에서 어댑터로 향하는 우발적 의존성은 있을 수 없다.
- package-private이어도 되는 서비스 클래스
    - 인커밍 포트 인터페이스 뒤에 숨겨지는 서비스는 public일 필요가 없다. : ex. `GetAccountBalanceService`
- public이어야 하는 application, domain의 일부 클래스
    - application의 port(in, out) : ex. `SendMoneyUseCase`, `LoadAccountPort`, `UpdateAccountStatePort`
    - 도메인 클래스 : ex. `Account`, `Activity`

<br>

## 의존성 주입의 역할

* 클린 아키텍처의 본질

> 어플리케이션이 인커밍/아웃고잉 어댑터에 의존성을 갖지 않아야 한다.

어댑터는 그저 애플리케이션 계층에 위치한 서비스를 호출할 뿐이다. 영속성 어댑터와 같이 아웃고잉 어댑터에 대해서는 제어 흐름의 반대 방향으로 의존성을 돌리기 위해 의존성 역전 원칙을 이용해야 한다. 모든 계층에
의존성을 가진 중립적인 컴포넌트를 하나 도입하면 의존성 주입을 활용할 수 있다. 중립적인 컴포넌트는 아키텍처를 구성하는 대부분의 클래스를 초기화하는 역할을 한다.

### 예시

![image](https://user-images.githubusercontent.com/53366407/151645788-d03b79c4-72cc-43cb-ae06-11ef2c5f54f8.png)

웹 컨트롤러가 서비스에 의해 구현된 인커밍 포트를 호출한다. 서비스는 어댑터에 의해 구현된 아웃고잉 포트를 호출한다.

* AccountController
    * SendMoneyUseCase 인터페이스가 필요하므로 의존성 주입을 통해 SendMoneyService 클래스의 인스턴스를 주입
* SendMoneyService
    * LoadAccount 인터페이스로 가장한 AccountPersistenceAdapter 클래스의 인스턴스 주입


