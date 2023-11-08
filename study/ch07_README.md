# 7. 아키텍처 요소 테스트하기

## 테스트 피라미드

> 테스트가 여러 단위 및 교차 단위 경계 아키텍처 경계 또는 시스템 경계를 결합하면 빌드 비용이 더 많이 들고 실행 속도가 느려지며 취약해지는 경향이 있다. 피라미드는 이러한 테스트가 더 비싸질수록 테스트가 더
> 많은 범위를 커버하는 것을 목표로 해야함을 알려준다. 그렇지 않으면 새로운 기능 대신 테스트를 구축하는 데 너무 많은 시간을 할애하는 경향이 있다.
>
> 기본적으로 테스트 작성 비용이 저렴하고 유지관리가 쉬운 단위 테스트가 프로젝트에서 가장 많은 테스트를 구성한다. 따라서 테스트 피라미드의 가장 기초가 된다. 일반적으로 단일 클래스를 인스턴스화하고 인터페이스를
> 통해 기능을 테스트한다.

<br>


![](https://user-images.githubusercontent.com/30178507/158181032-672eabb8-a3fb-4874-96d2-f9dc8751e7ec.png)

> 💡 좋은 테스트의 속성(FIRST)
>
> F(Fast) : 빠르게 동작해라. <br>
> I(Isolated) : 고립시켜라. <br>
> R(Repeatable): 반복 가능해야 한다. <br>
> S(Self-validating) : 스스로 검증 가능해야 한다. <br>
> T(Timely): 적시에 사용해야 한다.

### 단위 테스트(Unit)

하나의 클래스를 인스턴스화하고 해당 클래스의 인터페이스를 통해 기능들을 테스트한다.
테스트 중인 클래스가 다른 클래스에 의존한다면 의존되는 클래스들은 mock으로 대체한다.

### 통합테스트(Integration)

연결된 여러 유닛을 인스턴스화하고 시작점이 되는 클래스의 인터페이스로 데이터를 보낸 후 유닛들의 네트워크가 기대한 대로 잘 동작하는지 검증한다.

### 시스템 테스트(E2E)

모든 객체 네트워크를 가동시켜 특정 유스케이스가 전 계층에서 잘 동작하는지 검증

<br>

## 단위 테스트로 도메인 엔티티 테스트하기

* 도메인 객체는 가장 만들기 쉽고 이해하기도 쉬우며 빠르게 실행되는 테스트를 만들 수 있다.
* 다른 클래스에 거의 의존하지 않으므로 단위 테스트 이외에 다른 종류의 테스트는 불필요하다.

```java
class AccountTest {

    private final AccountId accountId = new AccountId(1L);
    private final Account account = defaultAccount()
            .withAccountId(accountId)
            .withBaselineBalance(Money.of(555L))
            .withActivityWindow(new ActivityWindow(
                    defaultActivity()
                            .withTargetAccount(accountId)
                            .withMoney(Money.of(999L)).build(),
                    defaultActivity()
                            .withTargetAccount(accountId)
                            .withMoney(Money.of(1L)).build()))
            .build();

    @Test
    @DisplayName("계좌의 금액 계산하는 메서드")
    void calculateBalance() {
        Money balance = account.calculateBalance();

        assertThat(balance).isEqualTo(Money.of(1555L));
    }

    @Test
    @DisplayName("계좌 송금 성공 테스트")
    void withdrawalSucceeds() {
        boolean success = account.withdraw(Money.of(555L), new AccountId(99L));

        assertAll(
                () -> assertThat(success).isTrue(),
                () -> assertThat(account.getActivities()).hasSize(3),
                () -> assertThat(account.calculateBalance()).isEqualTo(Money.of(1000L))
        );
    }

    @Test
    @DisplayName("계좌 송금 실패 테스트 (초과된 금액)")
    void withdrawalFailure() {
        boolean failure = account.withdraw(Money.of(1556L), new AccountId(99L));

        assertAll(
                () -> assertThat(failure).isFalse(),
                () -> assertThat(account.getActivities()).hasSize(2),
                () -> assertThat(account.calculateBalance()).isEqualTo(Money.of(1555L))
        );
    }

    @Test
    @DisplayName("계좌 예금 성공 테스트")
    void depositSuccess() {
        boolean success = account.deposit(Money.of(445L), new AccountId(99L));

        assertAll(
                () -> assertThat(success).isTrue(),
                () -> assertThat(account.getActivities()).hasSize(3),
                () -> assertThat(account.calculateBalance()).isEqualTo(Money.of(2000L))
        );
    }

}
```

## 단위 테스트로 유스케이스 테스트하기

* 유스케이스에서는 보통 Out-Port를 사용하여 타 시스템 또는 DB에서 데이터를 가져오고 도메인 객체를 사용하여 로직을 구성한다.
* 따라서 유스케이스를 단위테스트로 작성하려면 Mock이 필수적이다. Port를 Mocking 하여 특정 메서드들이 잘 호출이 되었는지와 이들 메서드들을 조합하는 유스케이스의 동작을 테스트할 수 있다.
* 다만, Mock을 사용하게 되면 테스트가 코드의 행동 변경과 리펙터링에 취약해진다. 따라서 중요한 상호작용이 있는 경우에만 테스트를 하거나 아예 유스케이스도 통합테스트로 구성을 하는 것이 맞지 않을까 생각이
  든다.

```java
class SendMoneyServiceTest {
    private final LoadAccountPort loadAccountPort = mock(LoadAccountPort.class);
    private final AccountLock accountLock = mock(AccountLock.class);
    private final UpdateAccountStatePort updateAccountStatePort = mock(UpdateAccountStatePort.class);
    private final SendMoneyService sendMoneyService = new SendMoneyService(loadAccountPort, accountLock,
            updateAccountStatePort, moneyTransferProperties());

    @Test
    @DisplayName("계좌 송금 실패 테스트")
    void givenWithdrawalFails_thenOnlySourceAccountIsLockedAndReleased() {
        AccountId sourceAccountId = new AccountId(41L);
        Account sourceAccount = givenAnAccountWithId(sourceAccountId);

        AccountId targetAccountId = new AccountId(42L);
        Account targetAccount = givenAnAccountWithId(targetAccountId);

        givenWithdrawalWillFail(sourceAccount);
        givenDepositWillSucceed(targetAccount);

        SendMoneyCommand command = new SendMoneyCommand(sourceAccountId, targetAccountId, Money.of(300L));

        boolean success = sendMoneyService.sendMoney(command);

        assertThat(success).isFalse();

        then(accountLock).should().lockAccount(eq(sourceAccountId));
        then(accountLock).should().releaseAccount(eq(sourceAccountId));
        then(accountLock).should(times(0)).lockAccount(eq(targetAccountId));
    }

    @Test
    @DisplayName("계좌송금 성공 테스트")
    void transactionSucceeds() {
        Account sourceAccount = givenSourceAccount();
        Account targetAccount = givenTargetAccount();

        givenWithdrawalWillSucceed(sourceAccount);
        givenDepositWillSucceed(targetAccount);

        Money money = Money.of(500L);

        SendMoneyCommand command = new SendMoneyCommand(
                sourceAccount.getId().get(),
                targetAccount.getId().get(),
                money);

        boolean success = sendMoneyService.sendMoney(command);

        assertThat(success).isTrue();

        AccountId sourceAccountId = sourceAccount.getId().get();
        AccountId targetAccountId = targetAccount.getId().get();

        then(accountLock).should().lockAccount(eq(sourceAccountId));
        then(sourceAccount).should().withdraw(eq(money), eq(targetAccountId));
        then(accountLock).should().releaseAccount(eq(sourceAccountId));

        then(accountLock).should().lockAccount(eq(targetAccountId));
        then(targetAccount).should().deposit(eq(money), eq(sourceAccountId));
        then(accountLock).should().releaseAccount(eq(targetAccountId));

        thenAccountsHaveBeenUpdated(sourceAccountId, targetAccountId);
    }

    private void thenAccountsHaveBeenUpdated(AccountId... accountIds) {
        ArgumentCaptor<Account> accountCaptor = ArgumentCaptor.forClass(Account.class);
        then(updateAccountStatePort).should(times(accountIds.length))
                .updateActivities(accountCaptor.capture());

        List<AccountId> updatedAccountIds = accountCaptor.getAllValues()
                .stream()
                .map(Account::getId)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .collect(Collectors.toList());

        for (AccountId accountId : accountIds) {
            assertThat(updatedAccountIds).contains(accountId);
        }
    }

    private void givenDepositWillSucceed(Account account) {
        given(account.deposit(any(Money.class), any(AccountId.class))).willReturn(true);
    }

    private void givenWithdrawalWillFail(Account account) {
        given(account.withdraw(any(Money.class), any(AccountId.class))).willReturn(false);
    }

    private void givenWithdrawalWillSucceed(Account account) {
        given(account.withdraw(any(Money.class), any(AccountId.class))).willReturn(true);
    }

    private Account givenTargetAccount() {
        return givenAnAccountWithId(new AccountId(42L));
    }

    private Account givenSourceAccount() {
        return givenAnAccountWithId(new AccountId(41L));
    }

    private Account givenAnAccountWithId(AccountId id) {
        Account account = mock(Account.class);

        given(account.getId())
                .willReturn(Optional.of(id));
        given(loadAccountPort.loadAccount(eq(account.getId().get()), any(LocalDateTime.class)))
                .willReturn(account);

        return account;
    }

    private MoneyTransferProperties moneyTransferProperties() {
        return new MoneyTransferProperties(Money.of(Long.MAX_VALUE));
    }

}
```

> 테스트 중인 유스케이스 서비스는 `상태`가 없기 때문에 `then` 섹션에서 특정 상태를 검증할 수 없다. <br>
> 대신 테스트는 서비스가 (모킹된) 의존 대상의 특정 메서드와 상호작용했는지 여부를 검증한다.<br>
> 이는 테스트가 코드의 행동 변경뿐만 아니라 코드의 구조 변경에도 취약해진다는 의미가 된다. 자연스럽게 코드가 리팩터링되면 테스트도 변경될 확률이 높아진다.<br>
> 그렇기 때문에, 테스트에서 `어떤 상호작용`을 검증하고 싶은지 신중하게 생각해야 한다. <br>
> 앞의 예제처럼 모든 동작을 검증하는 대신 중요한 핵심만 골라 집중해서 테스트하는 것이좋다.
> 만약 모든 동작을 검증하려고 하면 클래스가 조금이라도 바뀔 때마다 테스트를 변경해야 한다.
> 이는 테스트의 가치를 떨어뜨리는 일이다.
>

<br>

## 통합 테스트로 웹 어댑터 테스트하기

* 웹 어댑터 테스트는 HTTP를 통해 입력을 받고 입력 유효성 검증, 유스케이스가 사용할 수 있는 포멧으로 변경, 다시 HTTP로 응답이라는 일련의 과정이 테스트 되어야한다.
* 이를 위해서 Spring Boot에서는 `@WebMvcTest`를 사용할 수 있다. 실제 유스케이스는 Mocking 하므로 내부로직을 모두 태울수는 없지만 일련의 HTTP 통신이 잘 이뤄지고 유스케이스가 정의한
  입력
  포멧을 제대로 전달하는 지를 테스트할 수 있다.

```java

@WebMvcTest(controllers = SendMoneyController.class)
class SendMoneyControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private SendMoneyUseCase sendMoneyUseCase;

    @Test
    @DisplayName("송금 테스트")
    void testSendMoney() throws Exception {
        mockMvc.perform(post("/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}", 41L, 42L, 500)
                        .header("Content-Type", "application/json"))
                .andExpect(status().isOk());

        then(sendMoneyUseCase)
                .should()
                .sendMoney(eq(new SendMoneyCommand(new AccountId(41L), new AccountId(42L), Money.of(500L))));
    }

}
```

## 통합 테스트로 영속성 어댑터 테스트하기

* 영속성 어댑터도 데이터 영속화를 위한 컨텍스트를 띄우는 과정이 필요하므로 통합테스트가 필요하다.
* 만약 Spring Boot와 Spring Data의 일부 프로젝트에서는 `@DataJpaTest`, `@DataRedisTest`,`@DataMongoTest`등을 사용하여 도메인 객체가 제대로 데이터베이스에
  영속화되는지, 조회가 되어 도메인 객체로 잘 변환이 되는지 등을 테스트할 수 있다.
* 더미데이터로 테스트 하는 것이 더 효율적이다.

```java

@DataJpaTest
@Import({AccountPersistenceAdapter.class, AccountMapper.class})
class AccountPersistenceAdapterTest {
    @Autowired
    private AccountPersistenceAdapter adapterUnderTest;

    @Autowired
    private ActivityRepository activityRepository;

    @Test
    @Sql("AccountPersistenceAdapterTest.sql")
    void loadsAccount() {
        Account account = adapterUnderTest.loadAccount(new AccountId(1L), LocalDateTime.of(2018, 8, 10, 0, 0));

        assertAll(
                () -> assertThat(account.getActivities()).hasSize(2),
                () -> assertThat(account.calculateBalance()).isEqualTo(Money.of(500))
        );
    }

    @Test
    void updatesActivities() {
        Account account = defaultAccount()
                .withBaselineBalance(Money.of(555L))
                .withActivityWindow(new ActivityWindow(
                        defaultActivity()
                                .withId(null)
                                .withMoney(Money.of(1L)).build()))
                .build();

        adapterUnderTest.updateActivities(account);

        ActivityJpaEntity savedActivity = activityRepository.findAll().get(0);

        assertAll(
                () -> assertThat(activityRepository.count()).isEqualTo(1),
                () -> assertThat(savedActivity.getAmount()).isEqualTo(1L)
        );
    }

}
```

<br>

## 시스템 테스트로 주요 경로 테스트하기

* 시스템 테스트는 애플리케이션을 구성하는 전체 컨텍스트를 띄워 API 요청을 통해 잘 동작하는지를 검증한다. 따라서 전체 컨텍스트를 띄워야하므로 `@SpringBootTest`를 사용하여 테스트를 진행한다.
* 출력 포트도 컨텍스트를 띄워야하기 때문에 필요한 데이터베이스는 `in-memory 데이터베이스`를 활용하거나 실제 환경과 비슷하게 가져가고 싶다면 Docker 컨테이너를 띄워 테스트용으로 사용할 수 있다.

```java

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SendMoneySystemTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private LoadAccountPort loadAccountPort;

    @Test
    @Sql("SendMoneySystemTest.sql")
    void sendMoney() {
        Money initialSourceBalance = sourceAccount().calculateBalance();
        Money initialTargetBalance = targetAccount().calculateBalance();
        ResponseEntity<Object> response = whenSendMoney(sourceAccountId(), targetAccountId(), transferredAmount());

        then(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        then(sourceAccount().calculateBalance()).isEqualTo(initialSourceBalance.minus(transferredAmount()));
        then(targetAccount().calculateBalance()).isEqualTo(initialTargetBalance.plus(transferredAmount()));
    }

    private Account sourceAccount() {
        return loadAccount(sourceAccountId());
    }

    private Account targetAccount() {
        return loadAccount(targetAccountId());
    }

    private Account loadAccount(AccountId accountId) {
        return loadAccountPort.loadAccount(accountId, LocalDateTime.now());
    }

    private ResponseEntity<Object> whenSendMoney(AccountId sourceAccountId, AccountId targetAccountId, Money amount) {
        HttpHeaders headers = new HttpHeaders();
        headers.add("Content-Type", "application/json");
        HttpEntity<Void> request = new HttpEntity<>(null, headers);

        return restTemplate.exchange(
                "/accounts/send/{sourceAccountId}/{targetAccountId}/{amount}",
                HttpMethod.POST,
                request,
                Object.class,
                sourceAccountId.getValue(),
                targetAccountId.getValue(),
                amount.getAmount());
    }

    private Money transferredAmount() {
        return Money.of(500L);
    }

    private AccountId sourceAccountId() {
        return new AccountId(1L);
    }

    private AccountId targetAccountId() {
        return new AccountId(2L);
    }

}
```

<br>

## 얼마만의 테스트가 충분할까?

라인 커버리지는 테스트 성공을 측정하는 데 있어서는 잘못된 지표다.
프로덕션의 버그를 수정하고 이 로부터 배우는 것을 우선순위로 삼으면 제대로 가고 있는 것이다.

- 도메인 엔티티를 구현할 때는 단위 테스트로 커버하자.
- 유스케이스를 구현할 떄는 단위 테스트로 커버하자.
- 어댑터를 구현할 때는 통합 테스트로 커버하자.
- 사용자가 취할 수 있는 중요 애플리케이션 경로는 시스템 테스트로 커버하자.