---
name: tdd-bdd
description: >
  Enforces strict Test-Driven Development (TDD) with a BDD-style structure and mocked
  dependencies. Use this skill whenever the user asks to write tests, write a function
  with tests, follow TDD, write unit tests, mock dependencies, or asks for testable code.
  Also trigger when the user asks for a service, repository, or handler and mentions
  testing, coverage, or mocks in the same request. Core languages are Java and Golang.
  Extends cleanly to Python, Flutter/Dart, or any other language when the user specifies one.
  If tests are involved in any way, this skill applies.
---

# TDD / BDD Skill

Write tests first. Always. Then write the minimal implementation that makes those tests pass.

---

## Workflow (always follow this order)

### Step 1: Understand the Problem
- Identify inputs, outputs, and constraints
- Identify every external dependency that needs mocking (HTTP, DB, time, randomness, filesystem)
- If anything is ambiguous, ask before writing tests

### Step 2: Write Tests First
Structure every test suite with at minimum:
- **Happy path** - valid input produces the expected output
- **Edge case** - boundary, null, empty, or unusual input
- **Failure case** - error handling and exception behaviour

Do not write implementation yet.

### Step 3: Confirm the Red Phase
State explicitly which tests would fail and why. There is no implementation to pass them.

### Step 4: Write Minimal Implementation
Only write what is needed to pass the tests. No speculative code.

### Step 5: Refactor (optional)
Improve structure or clarity without breaking tests.

---

## Output Format

Always structure responses in this order:

```
### Step 1: Test Cases
<tests>

### Step 2: Implementation
<code>

### Step 3: Notes (optional)
<brief explanation of mocking strategy or any non-obvious decisions>
```

---

## What NOT to Test

Avoid these anti-patterns:

- Testing private methods directly. Test behaviour through the public interface.
- Asserting on log output. Logs are not contracts.
- Testing framework internals. If you did not write it, do not test it.
- Testing getters and setters in isolation. They prove nothing.
- Brittle tests that couple to implementation details rather than behaviour.
- Tests that pass because an exception was silently swallowed.

---

## Mocking Rules

Always mock:
- HTTP and external API calls
- Database queries and writes
- Filesystem reads and writes
- System time (`LocalDateTime.now()`, `time.Now()`, `DateTime.now`, etc.)
- Random number or UUID generation

Never use:
- Real network calls in unit tests
- Shared mutable state between tests
- Hidden or ambient dependencies that are not injected

---

## Core Language Reference

### Java + JUnit 5 + Mockito

BDD structure uses `@Nested` classes and `@DisplayName` to mirror `describe/it` semantics.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @InjectMocks
    private OrderService orderService;

    @Nested
    @DisplayName("processOrder")
    class ProcessOrder {

        @Test
        @DisplayName("should return confirmed status when payment succeeds")
        void shouldReturnConfirmedStatusWhenPaymentSucceeds() {
            Order pendingOrder = Order.builder()
                .id(1L)
                .totalAmount(new BigDecimal("49.99"))
                .build();

            when(orderRepository.findById(1L)).thenReturn(Optional.of(pendingOrder));
            when(paymentGateway.charge(pendingOrder.getTotalAmount())).thenReturn(true);

            OrderResult result = orderService.processOrder(1L);

            assertThat(result.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
            verify(paymentGateway).charge(pendingOrder.getTotalAmount());
        }

        @Test
        @DisplayName("should throw OrderNotFoundException when order does not exist")
        void shouldThrowWhenOrderDoesNotExist() {
            when(orderRepository.findById(99L)).thenReturn(Optional.empty());

            assertThatThrownBy(() -> orderService.processOrder(99L))
                .isInstanceOf(OrderNotFoundException.class)
                .hasMessageContaining("99");
        }

        @Test
        @DisplayName("should return failed status when payment is declined")
        void shouldReturnFailedStatusWhenPaymentIsDeclined() {
            Order pendingOrder = Order.builder()
                .id(1L)
                .totalAmount(new BigDecimal("49.99"))
                .build();

            when(orderRepository.findById(1L)).thenReturn(Optional.of(pendingOrder));
            when(paymentGateway.charge(pendingOrder.getTotalAmount())).thenReturn(false);

            OrderResult result = orderService.processOrder(1L);

            assertThat(result.getStatus()).isEqualTo(OrderStatus.FAILED);
        }
    }
}
```

Minimal implementation:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public OrderResult processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));

        boolean paymentSucceeded = paymentGateway.charge(order.getTotalAmount());

        OrderStatus status = paymentSucceeded ? OrderStatus.CONFIRMED : OrderStatus.FAILED;

        return OrderResult.builder().status(status).build();
    }
}
```

### Golang + testing + testify

Structure tests using subtests via `t.Run` to mirror BDD grouping. Inject dependencies
via interfaces so they can be replaced with fakes or mocks in tests.

```go
type UserRepository interface {
    FindByID(ctx context.Context, userID int64) (*User, error)
}

func TestGetUserFullName(t *testing.T) {
    t.Run("returns full name when user exists with first and last name", func(t *testing.T) {
        mockRepo := &MockUserRepository{
            user: &User{FirstName: "John", LastName: "Doe"},
        }

        result, err := getUserFullName(context.Background(), mockRepo, 1)

        assert.NoError(t, err)
        assert.Equal(t, "John Doe", result)
    })

    t.Run("returns first name only when last name is empty", func(t *testing.T) {
        mockRepo := &MockUserRepository{
            user: &User{FirstName: "John", LastName: ""},
        }

        result, err := getUserFullName(context.Background(), mockRepo, 1)

        assert.NoError(t, err)
        assert.Equal(t, "John", result)
    })

    t.Run("returns error when user is not found", func(t *testing.T) {
        mockRepo := &MockUserRepository{err: ErrUserNotFound}

        _, err := getUserFullName(context.Background(), mockRepo, 99)

        assert.ErrorIs(t, err, ErrUserNotFound)
    })
}

type MockUserRepository struct {
    user *User
    err  error
}

func (m *MockUserRepository) FindByID(ctx context.Context, userID int64) (*User, error) {
    return m.user, m.err
}
```

Minimal implementation:

```go
func getUserFullName(ctx context.Context, userRepo UserRepository, userID int64) (string, error) {
    user, err := userRepo.FindByID(ctx, userID)
    if err != nil {
        return "", err
    }

    if user.LastName == "" {
        return user.FirstName, nil
    }

    return user.FirstName + " " + user.LastName, nil
}
```

---

## Extending to Other Languages

When the user specifies a language outside the core stack, apply the same TDD workflow
using the idiomatic framework for that language. Map the concepts as follows:

| Concept         | Java              | Golang            | Python            | Flutter/Dart       |
|----------------|-------------------|-------------------|-------------------|--------------------|
| Test runner     | JUnit 5           | testing           | pytest            | flutter_test       |
| Mocking         | Mockito           | interface fakes   | unittest.mock     | mocktail           |
| Assertions      | AssertJ           | testify           | assert / pytest   | expect()           |
| BDD grouping    | @Nested @DisplayName | t.Run          | class Test / describe (pytest-describe) | group() / test() |

### Python example structure

```python
from unittest.mock import MagicMock, patch
import pytest
from user_service import get_user_full_name

class TestGetUserFullName:
    def test_returns_full_name_when_both_names_present(self, mock_user_repository):
        mock_user_repository.find_by_id.return_value = User(first_name="John", last_name="Doe")

        result = get_user_full_name(user_repository=mock_user_repository, user_id=1)

        assert result == "John Doe"

    def test_returns_first_name_only_when_last_name_is_missing(self, mock_user_repository):
        mock_user_repository.find_by_id.return_value = User(first_name="John", last_name=None)

        result = get_user_full_name(user_repository=mock_user_repository, user_id=1)

        assert result == "John"

    def test_raises_when_user_is_not_found(self, mock_user_repository):
        mock_user_repository.find_by_id.side_effect = UserNotFoundException("User not found: 99")

        with pytest.raises(UserNotFoundException):
            get_user_full_name(user_repository=mock_user_repository, user_id=99)

@pytest.fixture
def mock_user_repository():
    return MagicMock()
```

### Flutter/Dart example structure

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late MockUserRepository mockUserRepository;
  late UserService userService;

  setUp(() {
    mockUserRepository = MockUserRepository();
    userService = UserService(userRepository: mockUserRepository);
  });

  group('getUserFullName', () {
    test('returns full name when both first and last name are present', () async {
      when(() => mockUserRepository.findById(1))
          .thenAnswer((_) async => User(firstName: 'John', lastName: 'Doe'));

      final result = await userService.getUserFullName(1);

      expect(result, equals('John Doe'));
    });

    test('returns first name only when last name is null', () async {
      when(() => mockUserRepository.findById(1))
          .thenAnswer((_) async => User(firstName: 'John', lastName: null));

      final result = await userService.getUserFullName(1);

      expect(result, equals('John'));
    });

    test('throws UserNotFoundException when user does not exist', () async {
      when(() => mockUserRepository.findById(99))
          .thenThrow(UserNotFoundException('User not found: 99'));

      expect(
        () => userService.getUserFullName(99),
        throwsA(isA<UserNotFoundException>()),
      );
    });
  });
}
```

For any other language not listed here, apply the same principles: inject dependencies,
mock at the boundary, structure tests in happy/edge/failure groups, and write tests before
any implementation exists.

---

## Strict Mode

If the user says "strict TDD" or "wait for my signal", stop after Step 2 and output only
the tests. Do not write implementation until the user explicitly says "continue" or
"now implement". Confirm this mode is active at the top of your response.
