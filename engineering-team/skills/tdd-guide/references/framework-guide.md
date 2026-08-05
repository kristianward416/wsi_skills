# Framework Guide

This reference documents the concrete, idiomatic conventions
`framework_adapter.py` needs to produce for each supported testing
framework: Jest, Vitest, Pytest, JUnit, and Mocha. Use it to check that
generated or adapted tests actually look like tests a developer on that
stack would write by hand — correct structure, correct mocking API,
correct assertion style.

## Jest (JavaScript/TypeScript)

### Structure: describe/it

```javascript
// orderCalculator.test.js
const { OrderCalculator } = require('./orderCalculator');

describe('OrderCalculator', () => {
  describe('calculate()', () => {
    it('returns the sum of line item totals', () => {
      const calc = new OrderCalculator([{ price: 10, qty: 2 }, { price: 5, qty: 1 }]);
      expect(calc.calculate()).toBe(25);
    });

    it('returns 0 for an empty order', () => {
      const calc = new OrderCalculator([]);
      expect(calc.calculate()).toBe(0);
    });

    it('throws for a negative quantity', () => {
      expect(() => new OrderCalculator([{ price: 10, qty: -1 }]).calculate())
        .toThrow('quantity must be non-negative');
    });
  });
});
```

### Mocking with `jest.mock`

```javascript
// paymentService.test.js
jest.mock('./paymentGateway');
const { charge } = require('./paymentGateway');
const { PaymentService } = require('./paymentService');

test('charges the gateway with the order total', async () => {
  charge.mockResolvedValue({ status: 'ok', id: 'ch_123' });

  const result = await new PaymentService().pay({ total: 42 });

  expect(charge).toHaveBeenCalledWith(42);
  expect(result.id).toBe('ch_123');
});

test('surfaces a gateway failure', async () => {
  charge.mockRejectedValue(new Error('card_declined'));

  await expect(new PaymentService().pay({ total: 42 }))
    .rejects.toThrow('card_declined');
});
```

Spying on a method of a real object instead of mocking a whole module:

```javascript
test('logs a warning on retry', () => {
  const warnSpy = jest.spyOn(console, 'warn').mockImplementation(() => {});
  retryableCall();
  expect(warnSpy).toHaveBeenCalledWith(expect.stringContaining('retrying'));
  warnSpy.mockRestore();
});
```

### Conventions checklist for the adapter
- File name: `<subject>.test.js` or `<subject>.spec.js`, colocated with source.
- One top-level `describe` per class/module; nested `describe` per method.
- Use `beforeEach`/`afterEach` for setup/teardown, not shared mutable
  module-level state.
- Async tests use `async/await` with `expect(...).resolves`/`.rejects`,
  not manual `.then()` chains.

## Vitest (JavaScript/TypeScript)

Vitest's API mirrors Jest's almost exactly, but imports are explicit and
mocking uses `vi` instead of `jest`.

```typescript
// orderCalculator.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { OrderCalculator } from './orderCalculator';
import * as paymentGateway from './paymentGateway';

describe('OrderCalculator', () => {
  it('returns the sum of line item totals', () => {
    const calc = new OrderCalculator([{ price: 10, qty: 2 }]);
    expect(calc.calculate()).toBe(20);
  });
});

describe('PaymentService', () => {
  beforeEach(() => {
    vi.spyOn(paymentGateway, 'charge').mockResolvedValue({ status: 'ok', id: 'ch_1' });
  });

  it('charges the gateway with the order total', async () => {
    const result = await new PaymentService().pay({ total: 42 });
    expect(paymentGateway.charge).toHaveBeenCalledWith(42);
    expect(result.id).toBe('ch_1');
  });
});
```

Module-level auto-mocking:

```typescript
vi.mock('./paymentGateway', () => ({
  charge: vi.fn().mockResolvedValue({ status: 'ok', id: 'ch_1' }),
}));
```

### Conventions checklist for the adapter
- Explicit `import { describe, it, expect, vi } from 'vitest'` — Vitest
  does not inject Jest-style globals unless `globals: true` is set in
  config; the adapter should not assume globals are available.
- Use `vi.fn()`, `vi.spyOn()`, `vi.mock()` — never emit `jest.*` calls into
  a Vitest file.
- `vi.mock()` calls are hoisted, same as Jest's `jest.mock()` — factory
  functions must not reference out-of-scope variables that aren't
  themselves mocked.

## Pytest (Python)

### Structure and fixtures

```python
# test_order_calculator.py
import pytest
from order_calculator import OrderCalculator

@pytest.fixture
def empty_order():
    return OrderCalculator([])

@pytest.fixture
def two_item_order():
    return OrderCalculator([{"price": 10, "qty": 2}, {"price": 5, "qty": 1}])

def test_calculate_sums_line_items(two_item_order):
    assert two_item_order.calculate() == 25

def test_calculate_empty_order_is_zero(empty_order):
    assert empty_order.calculate() == 0

def test_calculate_rejects_negative_quantity():
    with pytest.raises(ValueError, match="quantity must be non-negative"):
        OrderCalculator([{"price": 10, "qty": -1}]).calculate()
```

### `parametrize` for triangulated cases

```python
@pytest.mark.parametrize("qty,price,expected", [
    (0, 10, 0),
    (1, 10, 10),
    (3, 10, 30),
    (2, 0, 0),
])
def test_calculate_various_quantities(qty, price, expected):
    calc = OrderCalculator([{"price": price, "qty": qty}])
    assert calc.calculate() == expected
```

### Mocking with `monkeypatch` / `unittest.mock`

```python
def test_pay_charges_gateway_with_total(monkeypatch):
    calls = {}
    def fake_charge(amount):
        calls["amount"] = amount
        return {"status": "ok", "id": "ch_1"}

    monkeypatch.setattr("payment_gateway.charge", fake_charge)

    result = PaymentService().pay({"total": 42})

    assert calls["amount"] == 42
    assert result["id"] == "ch_1"
```

### Conventions checklist for the adapter
- File name: `test_<subject>.py`, function name: `test_<behavior>`.
- Shared setup goes in `@pytest.fixture`, not `setUp`-style classes,
  unless the project already uses `unittest.TestCase`.
- Use `pytest.raises` as a context manager for exception assertions, with
  `match=` for the message when it matters.
- Prefer `parametrize` over copy-pasted near-identical test functions.

## JUnit (Java)

### Structure and assertions (JUnit 5 / Jupiter)

```java
// OrderCalculatorTest.java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.junit.jupiter.api.Assertions.*;

class OrderCalculatorTest {

    private OrderCalculator calc;

    @BeforeEach
    void setUp() {
        calc = new OrderCalculator();
    }

    @Test
    void calculateSumsLineItems() {
        calc.addItem(10.0, 2);
        calc.addItem(5.0, 1);
        assertEquals(25.0, calc.calculate());
    }

    @Test
    void calculateEmptyOrderIsZero() {
        assertEquals(0.0, calc.calculate());
    }

    @Test
    void calculateRejectsNegativeQuantity() {
        IllegalArgumentException ex = assertThrows(
            IllegalArgumentException.class,
            () -> calc.addItem(10.0, -1)
        );
        assertTrue(ex.getMessage().contains("quantity must be non-negative"));
    }

    @ParameterizedTest
    @CsvSource({"0,10,0", "1,10,10", "3,10,30", "2,0,0"})
    void calculateVariousQuantities(int qty, double price, double expected) {
        calc.addItem(price, qty);
        assertEquals(expected, calc.calculate());
    }
}
```

### Mocking with Mockito

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentGateway gateway;

    @Test
    void paysChargesGatewayWithTotal() {
        when(gateway.charge(42.0)).thenReturn(new ChargeResult("ok", "ch_1"));

        PaymentService service = new PaymentService(gateway);
        ChargeResult result = service.pay(new Order(42.0));

        verify(gateway).charge(42.0);
        assertEquals("ch_1", result.getId());
    }
}
```

### Conventions checklist for the adapter
- File name: `<Subject>Test.java`, one class per subject class.
- Use `@BeforeEach`/`@AfterEach` for setup/teardown, `@Test` per behavior.
- Use `assertThrows` for exceptions, not try/catch with a fail() fallback.
- Mock only injected collaborators (constructor/setter injected), verify
  interactions with `verify()` only when the interaction itself is the
  contract being tested.

## Mocha (JavaScript) + Chai

### Structure: describe/it with Chai assertions

```javascript
// orderCalculator.test.js
const { expect } = require('chai');
const { OrderCalculator } = require('./orderCalculator');

describe('OrderCalculator', () => {
  describe('calculate()', () => {
    it('returns the sum of line item totals', () => {
      const calc = new OrderCalculator([{ price: 10, qty: 2 }, { price: 5, qty: 1 }]);
      expect(calc.calculate()).to.equal(25);
    });

    it('returns 0 for an empty order', () => {
      expect(new OrderCalculator([]).calculate()).to.equal(0);
    });

    it('throws for a negative quantity', () => {
      expect(() => new OrderCalculator([{ price: 10, qty: -1 }]).calculate())
        .to.throw('quantity must be non-negative');
    });
  });
});
```

### Mocking with Sinon (Mocha has no built-in mocking)

```javascript
const sinon = require('sinon');
const { expect } = require('chai');
const paymentGateway = require('./paymentGateway');
const { PaymentService } = require('./paymentService');

describe('PaymentService', () => {
  afterEach(() => sinon.restore());

  it('charges the gateway with the order total', async () => {
    const chargeStub = sinon.stub(paymentGateway, 'charge')
      .resolves({ status: 'ok', id: 'ch_1' });

    const result = await new PaymentService().pay({ total: 42 });

    expect(chargeStub.calledWith(42)).to.be.true;
    expect(result.id).to.equal('ch_1');
  });
});
```

### Conventions checklist for the adapter
- File name: `<subject>.test.js`, run via `mocha` (Mocha itself provides
  only `describe`/`it`/hooks — no assertion or mocking library).
- Assertion library is Chai (`expect` or `assert` style) unless the
  project already uses Node's built-in `assert`.
- Mocking library is typically Sinon — always `sinon.restore()` in
  `afterEach` to avoid stubs leaking across tests.
- Async tests return a promise or use `async` functions — Mocha does not
  auto-await `async/await` returns the same way but `describe`/`it`
  callbacks that are `async` functions work correctly since Mocha awaits
  the returned promise.

## Quick reference: how `framework_adapter.py` should route

| Detected framework | Test runner API | Mocking API | Assertion style |
|---|---|---|---|
| Jest | `describe`/`it`/`test` (global) | `jest.mock`, `jest.fn`, `jest.spyOn` | `expect(...).toBe/toEqual/toThrow` |
| Vitest | `describe`/`it`/`test` (import from `vitest`) | `vi.mock`, `vi.fn`, `vi.spyOn` | `expect(...).toBe/toEqual/toThrow` |
| Pytest | plain functions + fixtures | `monkeypatch`, `unittest.mock` | `assert`, `pytest.raises` |
| JUnit | `@Test` methods in a class | Mockito `@Mock`/`when`/`verify` | `assertEquals`, `assertThrows` |
| Mocha | `describe`/`it` (global) | Sinon `stub`/`spy` | Chai `expect(...).to.equal/.to.throw` |
