# 6. 영속성 어댑터 구현하기

> 데이터베이스 주도 설계를 피하기 위해 영속성 계층을 앱 계층의 플러그인으로 만들어보자.

## 의존성 역전

![image](https://user-images.githubusercontent.com/6725753/157010412-76f77116-e272-4ecc-a26f-04f4ea052cdd.png)

- 앱은 영속성 기능을 사용하기 위해 포트 호출. 이 포트는 영속성 어댑터에 의해 구현.(DIP)
- 헥사고날 아키텍처에서 영속성 어댑터는 앱에서 호출하기 때문에 `outgoing` 어댑터
- 포트는 앱과 영속성 사이의 간접적 계층
    - 영속성 문제에 신경쓰지 않고 도메인 코드 개발 즉,코드 의존성을 없앤다
    - 영속성 코드를 변경하더라도 코어 코드에 영향이 없다
- DIP로 인해 정적인 상황에서는 의존성이 역전되었지만, 동적인 타임에는 여전히 앱이 영속성 코드에 의존하고 있다.
- 하지만 인터페이스 계약을 만족하는 한 영속성 코드 수정은 문제가 없다.

<br>

## 영속성 어댑터의 책임

> 영속성 어댑터가 일반적으로 하는 일.

1. 입력을 받는다
    - 입력 모델은 인터페이스의 도메인 엔티티나 특정 데이터베이스 연산 전용 객체
2. 입력을 데이터베이스 포맷으로 매핑한다
    - JPA인 경우 JPA 엔티티 객체로 매핑
    - 맥락에 따라 매핑이 필요하지 않는 경우도 있음
    - JPA가 아닌 어떤 기술도 상관없음
    - **핵심은 영속성 어댑터의 입력 모델이 어댑터가 아닌 코어에 존재. 이로인해 어댑터 변경이 코어에 영향을 주지 않는다**
3. 입력을 데이터베이스로 보낸다
4. 데이터베이스 출력을 애플리케이션 포맷으로 매핑한다
    - `출력 모델도 코어에 위치`
5. 출력을 반환한다

<br>

## 포트 인터페이스 나누기

> 그렇다면 영속성을 담당하는 포트 인터페이스를 어떻게 나눠야 할까?

![image](https://user-images.githubusercontent.com/6725753/157163384-5df55945-db1e-4859-85f1-ffeb17735123.png)

- 위 그림처럼 하나에 리포지터리 인터페이스에 담아 놓는게 일반적이다.
- 하지만 이럴경우 `넓은` 포트 인터페이스 문제점을 갖게 된다. (`SRP 위배`)
- 이로 인해 불필요한 의존이 생기고 테스트를 어렵게 한다
    - 위 그림에서 RegisterAccountService를 테스트 한다면 AccountRepository 모킹을 해야할 때 어떤 메서드를 모킹해야 하는지 일일이 찾아봐야한다.
    - 또한, 다음에 이 테스트에 작업하는 사람은 인터페이스 전체가 모킹 되었다고 오해를 할 수도 있다.
- 따라서 `ISP(Interface Segregation Principle 인터페이스 분리 원칙)`을 적용해야한다.

![image](https://user-images.githubusercontent.com/6725753/157164231-df353c57-52c7-435b-84e0-b6364d6a26c0.png)

<br>

## 영속 어댑터 나누기

> 지금까지 어댑터 하나로 해결했으나 이제 나눠보자.

![image](https://user-images.githubusercontent.com/6725753/157164823-8331bbf5-4108-48ff-85f7-39063a5575d9.png)

- 영속성 연산이 필요한 도메인 클래스(`aggregate`) 당 하나의 어댑터 구현
- 물론 어댑터를 필요에 따라 훨씬 더 쪼갤 수도 있다. 예) JPA 외에 SQL을 바로 쓰기 위한 포트를 구현하는 경우 등
- 이는 여러 개의 바운디드 컨텍스트의 영속성 요구사항을 분리하기 위한 좋은 토대

  ![image](https://user-images.githubusercontent.com/6725753/157165183-87647a3c-b631-4a00-8a11-6ac481b66c64.png)


- 각 바운디드 컨텍스트는 영속성 어댑터를 따로 가지고 있고
- 바운디드 컨텍스트??? 는 서로 분리되어 의존성이 없다
- 만약 서로 접근할 필요가 있다면 영속성 어댑터에 바로 접근하지 않고 인커밍 포트를 통해서 접근하게 된다

<br>

## 스프링 데이터 JPA 예제

```java

@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Account {

    private final AccountId id;
    @Getter
    private final Money baselineBalance;
    @Getter
    private final ActivityWindow activityWindow;

    // 유효한 상태의 Account 엔티티만 생성하도록 팩토리 메서드 제공
    public static Account withoutId(
            Money baselineBalance,
            ActivityWindow activityWindow) {
        return new Account(null, baselineBalance, activityWindow);
    }

    public static Account withId(
            AccountId accountId,
            Money baselineBalance,
            ActivityWindow activityWindow) {
        return new Account(accountId, baselineBalance, activityWindow);
    }
}
```

```java

@Entity
@Table(name = "account")
@Data
@AllArgsConstructor
@NoArgsConstructor
class AccountJpaEntity {
    // 영속성 어댑터를 위한 Account 엔티티 따로 정의

    @Id
    @GeneratedValue
    private Long id;

}
```

```java

@Entity
@Table(name = "activity")
@Data
@AllArgsConstructor
@NoArgsConstructor
class ActivityJpaEntity {
    // 영속성 어댑터에서 사용할 Activity 엔티티 정의

    @Id
    @GeneratedValue
    private Long id;

    @Column
    private LocalDateTime timestamp;

    @Column
    private long ownerAccountId;

    @Column
    private long sourceAccountId;

    @Column
    private long targetAccountId;

    @Column
    private long amount;

}
```

```java
interface ActivityRepository extends JpaRepository<ActivityJpaEntity, Long> {
    // ActivityRepository는 스프링에 의해서 자동으로 구현체가 생성된다
}
```

```java

@RequiredArgsConstructor
@PersistenceAdapter
class AccountPersistenceAdapter implements
        LoadAccountPort,
        UpdateAccountStatePort { // 실제 영속성 포트

    private final SpringDataAccountRepository accountRepository;
    private final ActivityRepository activityRepository;
    private final AccountMapper accountMapper;
    // LoadAccountPort, UpdateAccountStatePort 두 개의 포트를 구현한다

    @Override
    public Account loadAccount(
            AccountId accountId,
            LocalDateTime baselineDate) {

        // Account를 디비에서 불러오고
        AccountJpaEntity account =
                accountRepository.findById(accountId.getValue())
                        .orElseThrow(EntityNotFoundException::new);

        // 베이스 날짜 이후 Activity를 가져오고
        List<ActivityJpaEntity> activities =
                activityRepository.findByOwnerSince(
                        accountId.getValue(),
                        baselineDate);

        // 베이스 잔고를 구하기 위해 베이스 날짜 전까지의 입금 / 출금 활동을 디비에서 가져와서
        Long withdrawalBalance = activityRepository
                .getWithdrawalBalanceUntil(
                        accountId.getValue(),
                        baselineDate)
                .orElse(0L);

        Long depositBalance = activityRepository
                .getDepositBalanceUntil(
                        accountId.getValue(),
                        baselineDate)
                .orElse(0L);

        // Account Entity로 변경시 베이스 잔고를 계산한다
        return accountMapper.mapToDomainEntity(
                account,
                activities,
                withdrawalBalance,
                depositBalance);

    }

    @Override
    public void updateActivities(Account account) {
        for (Activity activity : account.getActivityWindow().getActivities()) {
            if (activity.getId() == null) {
                activityRepository.save(accountMapper.mapToJpaEntity(activity));
            }
        }
    }

}

```

```java

@Component
class AccountMapper {
    // 맵퍼는 도메인 엔티티와 영속성 엔티티 간에 쌍으로 존재한다.

    Account mapToDomainEntity(
            AccountJpaEntity account,
            List<ActivityJpaEntity> activities,
            Long withdrawalBalance,
            Long depositBalance) {

        Money baselineBalance = Money.subtract(
                Money.of(depositBalance),
                Money.of(withdrawalBalance));

        return Account.withId(
                new AccountId(account.getId()),
                baselineBalance,
                mapToActivityWindow(activities));

    }

    ActivityJpaEntity mapToJpaEntity(Activity activity) {
        return new ActivityJpaEntity(
                activity.getId() == null ? null : activity.getId().getValue(),
                activity.getTimestamp(),
                activity.getOwnerAccountId().getValue(),
                activity.getSourceAccountId().getValue(),
                activity.getTargetAccountId().getValue(),
                activity.getMoney().getAmount().longValue());
    }

}

```

- 쌍으로 엔티티를 만들지 않고 '매핑하지 않기' 전략이 유효할 수도 있다
- 하지만 이런 경우 JPA를 위해서 도메인 모델을 타협할 수 밖에 없다(기본 생성자라든가...)
- 즉, 영속성 측면에 타협하지 않을 때 좀 더 풍부한 도메인 모델응 생성할 수 있다.

<br>

## 데이터베이스 트랜잭션은 어떻게 해야 할까?

> 트랜잭션의 시작은 어디에 위치해야 할까?

- 트랜잭션은 특정 유스케이스에 대해서 모든 쓰기 작업에 걸쳐 있어야한다. 하나라도 실패하면 다 같이 롤백되어야 하기 때문
- 어댑터 같은 경우 어떤 유스케이스에 포함되는지 알지 못하기때문에 탈락
- **트랜잭션의 시작은 서비스**
- 스프링에서 가장 쉬운 방법은 Service 클래스에 붙여서 모든 public 메서드를 트랜잭션으로 감싸는 것이다
- 만약 서비스가 @transactional 에너테이션으로 오염되지 않고 깔끔하게 유지되길 원한 다면 AspectJ 같은 도구를 이용해 AOP으로 트랜잭션 경계를 코드에 위빙할 수 있다.

<br>

## 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

- 도메인 코드에 플로그인처럼 동작하는 영속성 어댑터는 도메인이 영속성에 분리되어 풍부한 도메인 모델이 가능하다
- 좁은 포트 인터페이스는 포트마다 다른 방식으로 구현 가능하기 때문에 유연하다
- 포트의 인터페이스만 지킨다면 영속성 기술 선택에 자유로워진다.

