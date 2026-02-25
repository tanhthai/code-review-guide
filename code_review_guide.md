# Code Review Guide

When raising review feedback, prioritize issues in this order — from highest business risk to lowest. Style and formatting should never block a correctness or security concern.

1. [Requirement & Business Correctness](#requirement--business-correctness-look-first)
2. [Security](#security-high-risk-area)
3. [Performance & System Risk](#performance--system-risk)
4. [Scalability & Concurrency](#scalability--concurrency)
5. [Architecture & Design](#architecture--design)
6. [Testability](#testability-design-quality-indicator)
7. [Test Coverage & Test Quality](#test-coverage--test-quality)
8. [Readability & Maintainability](#readability--maintainability)
9. [Infrastructure & Cost Impact](#infrastructure--cost-impact-advanced-review)
10. [Style & Formatting](#style--formatting-last)

---

## Requirement & Business Correctness (Look First)

> If this fails, everything else is irrelevant.

- Does it solve the correct problem?
- Are acceptance criteria fully satisfied?
- Does the change match the PR scope? (no missing work, no scope creep)
- Are edge cases and failure paths explicitly covered in the requirements?
- Does it follow business/domain rules?
- Are SLAs, compliance constraints, or data volume expectations from the spec respected?
- Are there regressions to existing behavior?

**Why first?** Because perfect code that solves the wrong problem is still wrong.

### Does it solve the correct problem?

---

## Security (High Risk Area)

> Security issues can cause immediate production damage.

- Input validation + output escaping (XSS)
- AuthN + AuthZ (incl. IDOR)
- Injection: SQL + command + NoSQL
- CSRF (when cookie-based auth)
- Sensitive data + secrets + safe logging
- SSRF / outbound URL fetch safety
- File upload safety
- Dependency/config risks + rate limiting

**Security must be reviewed early — not as an afterthought.**

---

## Performance & System Risk

> Before discussing style, check whether this can break production under load.

- Any O(n²) logic unintentionally?
- N+1 query risk?
- Heavy computation inside request thread?
- Large data loads?
- Blocking calls in high-throughput endpoints?

**If performance is wrong, the system suffers.**

---

## Scalability & Concurrency

> Now think at system level.

- Stateless design?
- Any shared mutable state?
- Thread-safe?
- Safe for horizontal scaling?
- Async needed?
- Any race condition risk?

**Especially important in backend / distributed systems.**

---

## Architecture & Design

> Now check structural integrity.

- Proper layer separation?
- No business logic in controller?
- No infra logic in domain?
- Dependency direction correct?
- No tight coupling introduced?
- Single Responsibility Principle respected?
- No duplication?

**Architecture protects long-term maintainability.**

---

## Testability (Design Quality Indicator)

> If code is hard to test, it is usually poorly designed.

- Dependencies injectable?
- Business logic separated?
- Pure logic extractable?
- No hidden side effects?
- Unit-testable without DB?
- Mocking straightforward?

**Testability reveals structural quality.**

---

## Test Coverage & Test Quality

> Now check safety net.

- Main flows covered?
- Edge cases tested?
- Failure cases tested?
- Tests validate behavior (not implementation details)?
- Integration tests needed?

**Tests protect future refactoring.**

---

## Readability & Maintainability

> Now check clarity.

- Code understandable in 6 months?
- Naming meaningful and domain-consistent?
- Methods not too long?
- Reasonable nesting depth?
- No dead code?
- Complex logic explained?

**Clean code reduces future cognitive load.**

---

## Infrastructure & Cost Impact (Advanced Review)

> Senior-level check.

- DB load increased?
- Index needed?
- Cache invalidation correct?
- Message queue impact?
- Cloud cost increased?
- Monitoring/logging sufficient?

**This is platform-thinking territory.**

---

## Style & Formatting (Last)

> Only after everything else:

- Formatting
- Lint warnings
- Minor naming improvements
