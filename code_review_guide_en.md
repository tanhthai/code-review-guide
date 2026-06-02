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
9. [Type Checking](#type-checking)
10. [Infrastructure & Cost Impact](#infrastructure--cost-impact-advanced-review)
11. [Style & Formatting](#style--formatting-last)
12. [Implementation Intent & Alternative Analysis](#implementation-intent--alternative-analysis)
13. [Reliability](#reliability)
14. [Operational Excellence](#operational-excellence)
15. [Cost Optimization](#cost-optimization)
16. [Sustainability](#sustainability)

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

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Does it solve the correct problem?</summary>

**Requirement:** Apply 20% discount for premium users, 10% for regular users.

❌ Bad — solves the wrong problem:

```java
public double calculateDiscount(User user, double price) {
    return price * 0.10; // Always 10% — ignores user type entirely
}
```

✅ Good — solves the correct problem:

```java
public double calculateDiscount(User user, double price) {
    if (user.isPremium()) {
        return price * 0.20;
    }
    return price * 0.10;
}
```

The bad version compiles and looks clean — but it silently violates the business requirement.

**Tip — scenario matrix approach:** Before reviewing the code, list every scenario implied by the requirement, then walk through the code for each one.

| Scenario | Expected | Does the code handle it? |
|---|---|---|
| Premium user with valid price | 20% discount | ✅ |
| Regular user with valid price | 10% discount | ✅ |
| Premium user with price = 0 | 0 | ? |
| Null user | Error / handled | ? |
| Negative price | Error / handled | ? |

Any cell that is "?" becomes a review comment. This turns a vague "does it solve the correct problem?" into a concrete checklist.

</details>

<details>
<summary>Are acceptance criteria fully satisfied?</summary>

**Ticket AC:** "Show an error message when login fails."

The PR adds login logic and returns a `401` status on failure — but no user-facing error message is rendered anywhere. The code is technically correct but the AC is only half-met. A reviewer who only reads the backend logic will miss this.

Check every AC line by line against the actual change.

</details>

<details>
<summary>Does the change match the PR scope?</summary>

**Ticket:** "Fix null pointer exception on the user profile page."

The PR fixes the bug — but also refactors the authentication service "while they were in there." The auth refactor is unreviewed, unticketed, and adds risk to an unrelated area. This is scope creep.

The reverse is also true: if the ticket requires updating both the API and the UI, but only the API is changed, that's missing work.

</details>

<details>
<summary>Are edge cases and failure paths explicitly covered?</summary>

**Requirement:** Process an order and calculate its total.

❌ Bad — ignores null and empty input:

```java
public void processOrder(List<Item> items) {
    double total = 0;
    for (Item item : items) {
        total += item.getPrice(); // NullPointerException if items is null
    }
    checkout(total); // Called even for empty orders
}
```

✅ Good — guards edge cases explicitly:

```java
public void processOrder(List<Item> items) {
    if (items == null || items.isEmpty()) {
        throw new IllegalArgumentException("Order must contain at least one item");
    }
    double total = 0;
    for (Item item : items) {
        if (item.getPrice() < 0) {
            throw new IllegalArgumentException("Item price cannot be negative: " + item.getName());
        }
        total += item.getPrice();
    }
    checkout(total);
}
```

Common categories to check:

- null / empty inputs
- zero, negative, or boundary values
- max size / very large inputs
- duplicate or repeated values
- concurrent access to shared state
- external dependency unavailable (timeout, error)

</details>

<details>
<summary>Does it follow business/domain rules?</summary>

**Requirement:** Users may withdraw funds, but a minimum balance of $10 must always remain.

❌ Bad — only checks available balance, misses the domain rule:

```java
public void withdraw(Account account, double amount) {
    if (amount > account.getBalance()) {
        throw new InsufficientFundsException();
    }
    account.debit(amount);
}
```

✅ Good — enforces the minimum balance rule:

```java
public void withdraw(Account account, double amount) {
    if (amount <= 0) {
        throw new IllegalArgumentException("Withdrawal amount must be positive");
    }
    if (account.getBalance() - amount < 10.0) {
        throw new InsufficientFundsException("Minimum balance of $10 must be maintained");
    }
    account.debit(amount);
}
```

Domain rules are often implicit — reviewers must know the spec, not just read the code.

</details>

<details>
<summary>Are SLAs, compliance constraints, or data volume expectations respected?</summary>

**Spec:** "The search API must return results in under 300ms for up to 100,000 records."

The PR adds a new filter that works fine in dev with 500 records — but runs a full table scan with no index. At 100,000 records in production, it will blow the SLA.

Check the spec for performance budgets, data volume targets, retention rules, or compliance requirements (GDPR, PCI, HIPAA) and verify the code respects them.

</details>

<details>
<summary>Are there regressions to existing behavior?</summary>

**Scenario:** A shared utility method is updated to support a new optional parameter with a default value. A caller in another module relied on the previous default behavior — it now silently receives wrong output with no compile error and no failing test.

Ask: could this change break something that currently works, even outside the files modified in this PR?

</details>
</blockquote>

</details>

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
- Rate limiting
- Dependency/config risks

**Security must be reviewed early — not as an afterthought.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Input validation + output escaping (XSS)</summary>

**Input validation** — reject bad input before it enters your system:

❌ Bad — no validation, accepts anything:

```java
@PostMapping("/comments")
public void submitComment(String comment) {
    commentRepository.save(new Comment(comment)); // null, blank, or 10,000-char input accepted
}
```

✅ Good — validate before processing:

```java
@PostMapping("/comments")
public void submitComment(String comment) {
    if (comment == null || comment.isBlank()) {
        throw new IllegalArgumentException("Comment must not be empty");
    }
    if (comment.length() > 500) {
        throw new IllegalArgumentException("Comment must not exceed 500 characters");
    }
    commentRepository.save(new Comment(comment));
}
```

Common validation checks:

- null / blank
- length (min / max)
- format (email, phone, UUID, date)
- numeric range (min / max value)
- allowlist (only accepted values, e.g. enum, country code)
- file type + size
- no dangerous patterns (SQL keywords, script tags, path traversal `../`)

**Output escaping** — encode before rendering to prevent XSS:

❌ Bad — raw user input injected into HTML:

```java
public String renderComment(String comment) {
    return "<p>" + comment + "</p>"; // XSS: attacker submits <script>stealCookies()</script>
}
```

✅ Good — escape before rendering:

```java
public String renderComment(String comment) {
    return "<p>" + HtmlUtils.htmlEscape(comment) + "</p>";
}
```

</details>

<details>
<summary>AuthN + AuthZ (incl. IDOR)</summary>

- **AuthN (Authentication)** — verifying *who* the user is (login, token validation)
- **AuthZ (Authorization)** — verifying *what* the user is allowed to do
- **IDOR (Insecure Direct Object Reference)** — an AuthZ failure where a user accesses another user's resource by manipulating an ID in the request

**Requirement:** Allow users to view their own orders.

❌ Bad — no ownership check (IDOR):

```java
// GET /api/orders/{orderId}
public Order getOrder(Long orderId) {
    return orderRepository.findById(orderId); // Any user can access any order
}
```

✅ Good — verify the resource belongs to the caller:

```java
public Order getOrder(Long orderId, User currentUser) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new NotFoundException("Order not found"));
    if (!order.getOwnerId().equals(currentUser.getId())) {
        throw new ForbiddenException("Access denied");
    }
    return order;
}
```

</details>

<details>
<summary>Injection: SQL + command + NoSQL</summary>

**SQL injection** — user input concatenated into a query string:

❌ Bad — string concatenation opens SQL injection:

```java
public User findByUsername(String username) {
    String query = "SELECT * FROM users WHERE username = '" + username + "'";
    return jdbcTemplate.queryForObject(query, userRowMapper);
}
```

✅ Good — parameterized query:

```java
public User findByUsername(String username) {
    return jdbcTemplate.queryForObject(
        "SELECT * FROM users WHERE username = ?",
        userRowMapper,
        username
    );
}
```

**Command injection** — user input passed directly to a shell command:

❌ Bad — attacker can append arbitrary shell commands:

```java
public String ping(String host) throws IOException {
    Process process = Runtime.getRuntime().exec("ping -c 1 " + host); // attacker passes: google.com; rm -rf /
    return new String(process.getInputStream().readAllBytes());
}
```

✅ Good — validate input and pass arguments as an array, bypassing shell interpretation:

```java
public String ping(String host) throws IOException {
    if (!host.matches("^[a-zA-Z0-9.-]+$")) {
        throw new IllegalArgumentException("Invalid host");
    }
    Process process = new ProcessBuilder("ping", "-c", "1", host).start();
    return new String(process.getInputStream().readAllBytes());
}
```

**NoSQL injection** — user input embedded in a raw query document:

❌ Bad — attacker passes `{"$gt": ""}` to bypass the username match:

```java
public Document findUser(String username) {
    String jsonQuery = "{\"username\": \"" + username + "\"}";
    return collection.find(Document.parse(jsonQuery)).first(); // operator injection
}
```

✅ Good — use a typed query builder; input is always treated as a value, never as an operator:

```java
public Document findUser(String username) {
    return collection.find(Filters.eq("username", username)).first();
}
```

</details>

<details>
<summary>CSRF (when cookie-based auth)</summary>

**Scenario:** A new POST /transfer endpoint is added with no CSRF token validation. The app uses cookie-based auth.

An attacker embeds a hidden auto-submitting form on a malicious site. When a logged-in user visits that page, their browser sends the request with their cookie — and the transfer executes without their knowledge.

Check: is the endpoint state-changing? Does the app use cookie auth? If yes, CSRF protection is required.

Popular protection methods:

- **CSRF token** — server generates a per-session token, embedded in forms/headers (`X-CSRF-Token`), validated on every state-changing request (default in Spring Security)
- **SameSite cookie** — set `SameSite=Strict` or `SameSite=Lax` on the session cookie so browsers block it from cross-origin requests
- **Custom request header** — require a header (e.g. `X-Requested-With: XMLHttpRequest`) that browsers cannot add automatically in cross-origin form posts
- **Double submit cookie** — send the CSRF token as both a cookie and a request parameter; validate they match server-side

</details>

<details>
<summary>Sensitive data + secrets + safe logging</summary>

**Requirement:** Log authentication attempts for auditing.

❌ Bad — credentials appear in logs:

```java
public void authenticate(String username, String password) {
    log.info("Login attempt: user={}, password={}", username, password); // Password exposed in log files
}
```

✅ Good — log only what is safe:

```java
public void authenticate(String username, String password) {
    log.info("Login attempt: user={}", username); // Never log passwords, tokens, or secrets
}
```

Never log or expose:

- passwords / passphrases
- API keys / secret tokens / JWT secrets
- session IDs / OAuth tokens / refresh tokens
- credit card numbers / CVV / bank account numbers
- SSN / national ID / passport numbers
- private keys / certificates
- PII in high-risk contexts (health records, government IDs)

</details>

<details>
<summary>SSRF / outbound URL fetch safety</summary>

- **SSRF (Server-Side Request Forgery)** — an attack where the attacker tricks your server into making HTTP requests on their behalf, often targeting internal services unreachable from the internet
- **Outbound URL fetch safety** — the practice of validating any URL your server fetches before making the request, to prevent SSRF and related abuses

**Requirement:** Fetch a URL provided by the user to generate a link preview.

❌ Bad — fetches any URL including internal services:

```java
public String fetchPreview(String url) throws IOException {
    return new URL(url).openStream().toString(); // SSRF: attacker passes http://169.254.169.254/latest/meta-data/
}
```

✅ Good — validate scheme and block private addresses:

```java
public String fetchPreview(String url) throws IOException {
    URI uri = URI.create(url);
    if (!List.of("http", "https").contains(uri.getScheme()) || isPrivateAddress(uri.getHost())) {
        throw new IllegalArgumentException("URL not allowed");
    }
    return httpClient.get(url);
}
```

The example blocks private addresses, but a production-safe implementation should also check:

- **scheme** — only allow `http` / `https`; reject `file://`, `ftp://`, `gopher://`, etc.
- **redirects** — re-validate the destination after each redirect; a public URL can redirect to an internal address
- **response size** — enforce a max bytes limit to prevent memory exhaustion from large payloads
- **timeout** — set a short connect + read timeout to prevent slow-response abuse
- **domain allowlist** — if the use case permits, restrict to a known set of trusted domains entirely

</details>

<details>
<summary>File upload safety</summary>

**Requirement:** Allow users to upload a profile picture.

❌ Bad — no type check, no size limit, path traversal risk:

- **Path traversal** — an attack where a user-supplied filename contains `../` sequences to escape the intended directory and write files to arbitrary locations on the server (e.g. `../../etc/cron.d/backdoor`)

```java
public void uploadFile(MultipartFile file) throws IOException {
    Path path = Paths.get("/uploads/" + file.getOriginalFilename()); // Path traversal: ../../../etc/passwd
    Files.write(path, file.getBytes()); // No type or size validation
}
```

✅ Good — validate type, size, and use a safe filename:

```java
private static final Set<String> ALLOWED_TYPES = Set.of("image/jpeg", "image/png");
private static final long MAX_SIZE = 5 * 1024 * 1024; // 5MB

public void uploadFile(MultipartFile file) throws IOException {
    if (!ALLOWED_TYPES.contains(file.getContentType())) {
        throw new IllegalArgumentException("File type not allowed");
    }
    if (file.getSize() > MAX_SIZE) {
        throw new IllegalArgumentException("File too large");
    }
    String safeFilename = UUID.randomUUID() + getExtension(file.getOriginalFilename());
    Files.write(Paths.get("/uploads/", safeFilename), file.getBytes());
}
```

</details>

<details>
<summary>Rate limiting</summary>

**Scenario:** The PR adds a new POST /api/send-email endpoint backed by a third-party email provider. No rate limiting is applied.

An attacker can call it thousands of times per minute — racking up costs, burning through provider quotas, and potentially using the endpoint for spam.

Check: is this a new public or unauthenticated endpoint? If yes, rate limiting is required.

</details>

<details>
<summary>Dependency/config risks</summary>

- **CVE (Common Vulnerabilities and Exposures)** — a public registry of known security flaws in software libraries; a library with a CVE may allow an attacker to exploit your app through that dependency

**Scenario:** The PR adds a new library `com.example:imageparser:2.1.0`. That version has a known remote code execution CVE. The build passes, the feature works — but the app is now vulnerable.

Check: are new dependencies pinned to a specific version and free of known CVEs? (For Java/Maven projects, scan with `mvn dependency-check`.) Are credentials or API keys stored in environment variables or a secrets manager — not hardcoded in source files or committed to the repo?

</details>
</blockquote>

</details>

---

## Performance & System Risk

> Before discussing style, check whether this can break production under load.

- Any O(n²) logic unintentionally?
- N+1 query risk?
- Heavy computation inside request thread?
- Large data loads?
- Blocking calls in high-throughput endpoints?

**If performance is wrong, the system suffers.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Any O(n²) logic unintentionally?</summary>

**Requirement:** Find duplicate emails in a list.

❌ Bad — nested loop, O(n²):

```java
public List<String> findDuplicates(List<String> emails) {
    List<String> duplicates = new ArrayList<>();
    for (int i = 0; i < emails.size(); i++) {
        for (int j = i + 1; j < emails.size(); j++) {
            if (emails.get(i).equals(emails.get(j))) {
                duplicates.add(emails.get(i));
            }
        }
    }
    return duplicates;
}
```

✅ Good — HashSet lookup, O(n):

```java
public List<String> findDuplicates(List<String> emails) {
    Set<String> seen = new HashSet<>();
    List<String> duplicates = new ArrayList<>();
    for (String email : emails) {
        if (!seen.add(email)) { // add() returns false if already present
            duplicates.add(email);
        }
    }
    return duplicates;
}
```

</details>

<details>
<summary>N+1 query risk?</summary>

**Requirement:** Load all orders with their customer names.

❌ Bad — 1 query for orders + N queries for customers:

```java
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order order : orders) {
    String name = customerRepository.findById(order.getCustomerId()).getName(); // N queries
    order.setCustomerName(name);
}
```

✅ Good — single JOIN query:

```java
List<OrderWithCustomer> orders = orderRepository.findAllWithCustomer(); // 1 query with JOIN
```

</details>

<details>
<summary>Heavy computation inside request thread?</summary>

**Requirement:** Generate a PDF report on demand.

❌ Bad — blocks the request thread for the full duration:

```java
@PostMapping("/reports/generate")
public ResponseEntity<byte[]> generateReport(@RequestBody ReportRequest request) {
    byte[] report = reportService.generatePdfReport(request); // 10+ seconds — thread is blocked
    return ResponseEntity.ok(report);
}
```

✅ Good — offload to async job, return immediately:

```java
@PostMapping("/reports/generate")
public ResponseEntity<String> generateReport(@RequestBody ReportRequest request) {
    String jobId = reportService.enqueueReportGeneration(request); // queued for background processing — returns immediately
    return ResponseEntity.accepted().body(jobId); // 202 Accepted: caller polls with jobId to check when ready
}
```

</details>

<details>
<summary>Large data loads?</summary>

**Requirement:** Return a user's transaction history.

❌ Bad — loads entire table into memory:

```java
public List<Transaction> getHistory(Long userId) {
    return transactionRepository.findAllByUserId(userId); // Could be millions of rows
}
```

✅ Good — paginate:

```java
public Page<Transaction> getHistory(Long userId, Pageable pageable) {
    return transactionRepository.findAllByUserId(userId, pageable);
}
```

</details>

<details>
<summary>Blocking calls in high-throughput endpoints?</summary>

**Requirement:** Return a user's personalised feed.

❌ Bad — synchronous call blocks a thread pool thread:

```java
@GetMapping("/feed")
public List<Post> getFeed(Long userId) {
    return externalFeedService.fetch(userId); // Blocks thread until external service responds
}
```

✅ Good — non-blocking async response:

```java
@GetMapping("/feed")
public CompletableFuture<List<Post>> getFeed(Long userId) {
    return externalFeedService.fetchAsync(userId);
}
```

</details>
</blockquote>

</details>

---

## Scalability & Concurrency

> Now think at system level.

- Stateless design?
- Any shared mutable state?
- Thread-safe?
- Safe for horizontal scaling?
- Async needed?
- Any race condition risk?
- Safe to retry (idempotent)?

**Especially important in backend / distributed systems.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Stateless design?</summary>

**Scenario:** The PR stores active user sessions in a `HashMap` field on a Spring `@Service` bean.

This works on a single instance but breaks when scaled horizontally — instance A holds sessions that instance B knows nothing about. Users get randomly logged out depending on which node handles their request.

Stateful in-memory caches must be externalised (Redis, DB) for multi-instance safety.

</details>

<details>
<summary>Any shared mutable state? / Thread-safe?</summary>

**1. Lost update** — two threads read and write the same variable simultaneously; one update is overwritten:

❌ Bad — `count++` is three steps (read → add → write) and can be interrupted in between:

```java
@Service
public class RequestCounter {
    private int count = 0;

    public void increment() { count++; } // Thread A and B both read 5, both write 6 — one increment lost
}
```

✅ Good — `AtomicInteger` makes the entire read-add-write unbreakable:

```java
@Service
public class RequestCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); } // single atomic operation
}
```

**2. Visibility** — one thread writes a value, another thread never sees the update due to CPU cache:

❌ Bad — `running = false` may stay in CPU cache; the loop in another thread never stops:

```java
private boolean running = true;

public void stop() { running = false; }         // Thread A writes
public void run()  { while (running) { ... } }  // Thread B may read stale cached value
```

✅ Good — `volatile` forces every read to go directly to main memory:

```java
private volatile boolean running = true;

public void stop() { running = false; }
public void run()  { while (running) { ... } }
```

**3. Non-thread-safe collections** — `HashMap` and `ArrayList` are not safe for concurrent reads and writes:

❌ Bad — concurrent writes corrupt entries or throw `ConcurrentModificationException`:

```java
private final Map<String, User> cache = new HashMap<>();

public void addToCache(String key, User user) {
    cache.put(key, user); // Two threads writing simultaneously → corrupted state
}
```

✅ Good — `ConcurrentHashMap` handles concurrent access safely:

```java
private final Map<String, User> cache = new ConcurrentHashMap<>();

public void addToCache(String key, User user) {
    cache.put(key, user); // thread-safe
}
```

For check-then-act race conditions involving database state, see **Any race condition risk?** below.

</details>

<details>
<summary>Any race condition risk?</summary>

**Requirement:** Deduct a balance only if funds are sufficient.

❌ Bad — check-then-act without a lock allows double-spend:

```java
public void deductBalance(Long userId, double amount) {
    User user = userRepository.findById(userId);
    if (user.getBalance() >= amount) { // Two threads can both pass this check simultaneously
        user.setBalance(user.getBalance() - amount); // Both deduct — account goes negative
        userRepository.save(user);
    }
}
```

✅ Good — lock the row before reading:

```java
@Transactional
public void deductBalance(Long userId, double amount) {
    User user = userRepository.findByIdWithLock(userId); // Pessimistic lock (SELECT FOR UPDATE)
    if (user.getBalance() < amount) throw new InsufficientFundsException();
    user.setBalance(user.getBalance() - amount);
    userRepository.save(user);
}
```

</details>

<details>
<summary>Safe for horizontal scaling?</summary>

**Scenario:** The PR uses a JVM `synchronized` block to prevent concurrent access to a shared resource.

This works on a single node — but on two nodes, each JVM has its own lock. Both nodes can enter the block simultaneously, defeating the protection entirely.

For multi-instance coordination, use distributed locking (Redis `SETNX`, DB advisory locks) instead of JVM-level synchronisation.

</details>

<details>
<summary>Async needed?</summary>

**Scenario:** A POST /checkout endpoint synchronously sends a confirmation email before returning a response.

If the email service is slow or temporarily down, the checkout hangs — or fails entirely. The user's payment went through, but they see an error.

Confirmation emails are fire-and-forget. Publish a domain event or push to a message queue and return the checkout result immediately.

</details>

<details>
<summary>Safe to retry (idempotent)?</summary>

Message brokers like AWS SQS use **at-least-once delivery** — if a handler crashes or times out before the message is deleted, SQS redelivers it. The handler runs again from the beginning, including any side effects that already completed in the first attempt.

**Scenario:** A handler saves an order to the DB, then calls the payment gateway. The payment call throws a timeout exception. SQS redelivers the message. On retry, the DB insert runs again — creating a duplicate order — before a second charge attempt is made.

❌ Bad — no guard against re-execution; duplicate records on retry:

```java
@SqsListener("order-events")
public void handleOrderEvent(OrderMessage message) {
    orderRepository.save(new Order(message.getOrderId(), message.getAmount())); // runs again on retry → duplicate
    paymentService.charge(message.getOrderId(), message.getAmount());           // may double-charge
}
```

✅ Good — deduplicate using the message ID before doing any work:

```java
@SqsListener("order-events")
public void handleOrderEvent(OrderMessage message) {
    if (orderRepository.existsByMessageId(message.getMessageId())) {
        return; // already processed — safe to skip
    }
    orderRepository.save(new Order(message.getOrderId(), message.getAmount(), message.getMessageId()));
    paymentService.charge(message.getOrderId(), message.getAmount());
}
```

Or use an **upsert** (`INSERT ... ON CONFLICT DO NOTHING` keyed on `messageId`) so the DB itself rejects duplicates atomically.

Common patterns to make logic retry-safe:

- **Idempotency key check** — store the message/request ID on first success; skip if already seen
- **Upsert instead of insert** — `INSERT ... ON CONFLICT DO NOTHING / DO UPDATE` prevents duplicate rows
- **Conditional updates** — only update if the record is still in the expected state (`WHERE status = 'PENDING'`)
- **External API guard** — check if the external call already succeeded before calling again (e.g. look up the charge by order ID before creating a new one)

Ask: if this handler is called twice with the same input, does the outcome change?

</details>
</blockquote>

</details>

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

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>No business logic in controller?</summary>

❌ Bad — discount logic and total calculation live in the controller:

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
    double total = request.getItems().stream()
        .mapToDouble(i -> i.getPrice() * i.getQuantity()).sum();
    if (total > 10000) total = total * 0.9; // Business rule in controller
    return ResponseEntity.ok(orderRepository.save(new Order(request.getUserId(), total)));
}
```

✅ Good — controller delegates, service owns the logic:

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
    return ResponseEntity.ok(orderService.createOrder(request));
}
```

</details>

<details>
<summary>No infra logic in domain?</summary>

❌ Bad — domain entity talks directly to the database:

```java
public class User {
    public void save() {
        DataSource ds = DataSourceConfig.getInstance(); // Infra concern in domain
        ds.execute("INSERT INTO users ...");
    }
}
```

✅ Good — domain entity is pure; insertion is delegated to the service layer via a repository:

```java
public class User {
    private Long id;
    private String email;
    // domain behaviour only — no DB, no HTTP, no I/O
}

public interface UserRepository extends JpaRepository<User, Long> {}

@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(User user) {
        return userRepository.save(user); // persistence handled here, not inside User
    }
}
```

</details>

<details>
<summary>No tight coupling introduced?</summary>

❌ Bad — service instantiates its own dependency:

```java
public class OrderService {
    private final MySQLOrderRepository repository = new MySQLOrderRepository(); // Hard-coded
}
```

✅ Good — depends on an abstraction, injected from outside:

```java
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

</details>

<details>
<summary>Single Responsibility Principle respected?</summary>

❌ Bad — one service handles user creation, emailing, logging, and reporting:

```java
public class UserService {
    public User createUser(UserRequest request) { ... }
    public void sendWelcomeEmail(User user) { ... }
    public void logUserCreation(User user) { ... }
    public byte[] generateUserReport(Long userId) { ... }
}
```

✅ Good — each class has one reason to change:

```java
public class UserService    { public User createUser(UserRequest request) { ... } }
public class EmailService   { public void sendWelcomeEmail(User user) { ... } }
public class ReportService  { public byte[] generateUserReport(Long userId) { ... } }
```

</details>

<details>
<summary>Dependency direction correct?</summary>

**Scenario:** The domain `Order` class directly imports and calls `EmailNotificationService` (an infrastructure concern) after being saved.

This means the domain layer depends on the infrastructure layer — the wrong direction. The domain should emit a domain event (`OrderPlaced`); infrastructure listens and reacts. Dependencies always point inward, toward the domain.

</details>

<details>
<summary>No duplication?</summary>

**Scenario:** The PR adds a `calculateTax()` method in `InvoiceService` that is nearly identical to one already in `OrderService`.

When tax rules change, both copies must be updated — and one will inevitably be missed, causing inconsistent behaviour. Extract the shared logic into a `TaxCalculator` component owned by the domain layer.

</details>
</blockquote>

</details>

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

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Dependencies injectable?</summary>

❌ Bad — hard-coded dependency, impossible to mock:

```java
public class OrderService {
    private final OrderRepository repository = new JpaOrderRepository(); // Cannot be replaced in tests

    public Order createOrder(OrderRequest request) {
        return repository.save(new Order(request));
    }
}
```

✅ Good — injected dependency, easily mocked:

```java
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository; // Pass a mock in tests
    }
}
```

</details>

<details>
<summary>Business logic separated? / Pure logic extractable?</summary>

**Scenario:** A controller method fetches data, applies pricing rules, formats the response, and sends an audit event — all inline. There is no way to test the pricing logic without spinning up the full HTTP stack.

When business logic is embedded inside framework callbacks (controllers, listeners, batch processors), it cannot be unit-tested in isolation. Extract it into a plain service or domain method that takes inputs and returns outputs, with no framework dependencies.

</details>

<details>
<summary>No hidden side effects?</summary>

❌ Bad — calculation method secretly writes to cache and audit log:

```java
public double calculateTotal(List<Item> items) {
    double total = items.stream().mapToDouble(Item::getPrice).sum();
    auditLog.record("Total: " + total); // Hidden side effect
    cache.put("lastTotal", total);      // Another hidden side effect
    return total;
}
```

✅ Good — pure function, predictable and easy to test in isolation:

```java
public double calculateTotal(List<Item> items) {
    return items.stream().mapToDouble(Item::getPrice).sum();
}
```

</details>

<details>
<summary>Unit-testable without DB?</summary>

❌ Bad — business logic buried behind a repository call; tests need a real DB:

```java
public class DiscountService {
    @Autowired
    private UserRepository userRepository;

    public double getDiscount(Long userId) {
        User user = userRepository.findById(userId); // Forces DB in every unit test
        return user.isPremium() ? 0.20 : 0.10;
    }
}
```

✅ Good — accept the domain object; no DB needed in unit tests:

```java
public class DiscountService {
    public double getDiscount(User user) {
        return user.isPremium() ? 0.20 : 0.10; // Test with new User(isPremium=true)
    }
}
```

</details>

<details>
<summary>Mocking straightforward?</summary>

**Scenario:** A service depends on `new EmailClient("smtp.internal", 587, true, "user", "pass")` constructed inline. To test the service, you'd need a real SMTP server.

If mocking a dependency requires more setup than the test itself, the dependency is too tightly bound. Depend on an interface (`EmailSender`), inject it, and mock the interface in tests with a single line.

</details>

<details>
<summary>⊕ Bonus — Recommended code organisation for testability (Spring web app)</summary>

> This section is supplementary to the checklist above. It provides a concrete reference architecture for Spring web apps that makes each layer independently testable.

Structure code into 4 layers. Each layer has a single concern — making each independently testable.

```
src/
└── main/java/com/yourapp/
    ├── controller/                    ← HTTP in/out only
    │   └── OrderController.java
    ├── application/                   ← use-case orchestration
    │   └── PlaceOrderUseCase.java
    ├── domain/                        ← pure business logic, no framework
    │   ├── Order.java
    │   ├── OrderRepository.java       ← interface (domain-owned)
    │   └── OrderPlacedEvent.java
    └── infrastructure/                ← DB, messaging, external APIs
        ├── JpaOrderRepository.java    ← implements OrderRepository
        └── SnsOrderEventPublisher.java
```

**What each layer does and how to test it:**

| Layer | Responsibility | Depends on | Test type | Setup needed |
|---|---|---|---|---|
| `domain/` | Business rules, pure logic | nothing | Unit test | `new Order(...)` — no mocks, no Spring |
| `application/` | Orchestrate: load → execute → save → publish | `domain/` interfaces only | Unit test with mocks | Mock `OrderRepository`, mock publisher |
| `controller/` | Parse HTTP request, return response | `application/` | `@WebMvcTest` slice | Mock `PlaceOrderUseCase` only |
| `infrastructure/` | DB queries, SNS, SES, external APIs | `domain/` (implements its interfaces) | Integration test | Real DB via Testcontainers |

> **Dependency rule:** `application/` never imports `infrastructure/` classes directly — it depends on the repository *interface* defined in `domain/`, and Spring injects the infrastructure implementation at runtime. This keeps the application layer testable without a real DB.

**The key rule:** the further inward the layer, the cheaper the test.

```java
// domain/ — zero dependencies, tested with plain objects
@Test
void order_shouldApplyBulkDiscount_whenTotalExceeds10000() {
    Order order = new Order(List.of(new Item("A", 12000.0)));
    assertThat(order.getFinalTotal()).isEqualTo(10800.0); // no Spring, no mock
}

// application/ — mock the boundaries, test the orchestration
@Test
void placeOrder_shouldSaveAndPublishEvent() {
    when(orderRepository.save(any())).thenReturn(savedOrder);
    useCase.execute(request);
    verify(eventPublisher).publish(any(OrderPlacedEvent.class));
}

// controller/ — mock the use case, test HTTP mapping only
@Test
void POST_orders_shouldReturn201_onSuccess() throws Exception {
    when(placeOrderUseCase.execute(any())).thenReturn(orderId);
    mockMvc.perform(post("/orders").contentType(APPLICATION_JSON).content(body))
           .andExpect(status().isCreated());
}
```

</details>
</blockquote>

</details>

---

## Test Coverage & Test Quality

> Now check safety net.

- Main flows covered?
- Edge cases tested?
- Failure cases tested?
- Tests validate behavior (not implementation details)?
- Integration tests needed?

**Tests protect future refactoring.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Main flows covered?</summary>

**Scenario:** A checkout feature is merged with tests only for the pricing logic. The full flow — add to cart → apply coupon → pay → receive confirmation — has no end-to-end or integration test.

If a main user journey has no test coverage, a future change anywhere in that path can break it silently.

</details>

<details>
<summary>Edge cases tested?</summary>

**Scenario:** A `calculateDiscount()` method is tested with a typical order of $50. But there are no tests for a $0 order, a negative price, or an empty item list.

Ask: what are the boundary values? What happens at zero, null, empty, max, and min?

</details>

<details>
<summary>Failure cases tested?</summary>

**Scenario:** The PR adds tests for the happy path — a successful payment. But there are no tests for what happens when the payment gateway is down, when the card is declined, or when the request times out.

Failure paths are where bugs cause the most damage. For every success test, ask: what is the corresponding failure test?

</details>

<details>
<summary>Tests validate behavior (not implementation details)?</summary>

❌ Bad — test verifies a method was called, not what the user experiences:

```java
@Test
void createOrder_shouldCallRepositorySave() {
    orderService.createOrder(request);
    verify(orderRepository, times(1)).save(any()); // Breaks on any refactor, even correct ones
}
```

✅ Good — test verifies the observable outcome:

```java
@Test
void createOrder_shouldReturnOrderWithCorrectTotal() {
    Order order = orderService.createOrder(request);
    assertThat(order.getTotal()).isEqualTo(expectedTotal);
}
```

</details>

<details>
<summary>Integration tests needed?</summary>

**Scenario:** Each service method is unit-tested in isolation with mocks. But when the real DB schema is used, a query fails because a column was renamed in a migration that the mocks never reflected.

Unit tests alone cannot catch integration failures. Ask: is there a layer boundary (DB, HTTP, message queue) that needs a real integration test?

</details>
</blockquote>

</details>

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

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Code understandable in 6 months?</summary>

This is the overarching goal of every point in this section. The "6 months" heuristic captures the reality that even the original author forgets the context behind decisions they made after a few months. If understanding the code requires knowledge that only existed in someone's head at the time of writing, it becomes a maintenance burden.

Ask: could a developer — including future-you — read this without needing to ask anyone what it does or why?

The number is illustrative, not exact. Some teams say "a new hire tomorrow", others say "a stranger next year". The point is the same: **code is read far more than it is written**, so clarity must survive beyond the moment of authorship. Every other point in this section — naming, method length, nesting depth, dead code, comments — is a specific way to answer this question with "yes".

</details>

<details>
<summary>Naming meaningful and domain-consistent?</summary>

❌ Bad — abbreviated names with no domain meaning:

```java
public double calc(User u, List<Item> l, boolean f) {
    double t = 0;
    for (Item i : l) t += i.getP() * i.getQ();
    return f ? t * 0.9 : t;
}
```

✅ Good — names communicate intent:

```java
public double calculateOrderTotal(User customer, List<Item> items, boolean isPremiumDiscount) {
    double subtotal = items.stream()
        .mapToDouble(item -> item.getPrice() * item.getQuantity())
        .sum();
    return isPremiumDiscount ? subtotal * 0.9 : subtotal;
}
```

</details>

<details>
<summary>Methods not too long?</summary>

**Scenario:** A `processCheckout()` method is 200 lines long. It validates input, applies coupons, calculates tax, charges the card, sends an email, and updates inventory — all inline.

A method should do one thing. If you need to scroll to read it, or if you need a comment to mark sections ("// step 1", "// step 2"), it should be split. Each extracted method becomes independently readable and testable.

</details>

<details>
<summary>Reasonable nesting depth?</summary>

❌ Bad — four levels of nesting make the happy path hard to follow:

```java
public void processPayment(Payment payment) {
    if (payment != null) {
        if (payment.getAmount() > 0) {
            if (paymentGateway.isAvailable()) {
                if (accountService.hasSufficientFunds(payment)) {
                    paymentGateway.process(payment);
                }
            }
        }
    }
}
```

✅ Good — guard clauses flatten the structure:

```java
public void processPayment(Payment payment) {
    if (payment == null || payment.getAmount() <= 0) throw new IllegalArgumentException();
    if (!paymentGateway.isAvailable()) throw new ServiceUnavailableException();
    if (!accountService.hasSufficientFunds(payment)) throw new InsufficientFundsException();
    paymentGateway.process(payment);
}
```

</details>

<details>
<summary>No dead code?</summary>

**Scenario:** The PR contains a `legacyCalculateShipping()` method that is never called anywhere. It was replaced three months ago but never deleted.

Dead code creates confusion — future readers wonder if it's intentional, if it's needed, or if removing it will break something. If it is unused, delete it. Version control preserves history.

</details>

<details>
<summary>Complex logic explained?</summary>

❌ Bad — the magic number and formula are unexplained:

```java
double adjustedScore = rawScore * 0.85 + (completionRate * 15);
```

✅ Good — a comment explains the why, not just the what:

```java
// Score formula: 85% weight on raw accuracy + 15% weight on task completion rate
// Defined in product spec v2.3 — do not change without updating the spec
double adjustedScore = rawScore * 0.85 + (completionRate * 15);
```

</details>
</blockquote>

</details>

---

## Type Checking

> The type system is your first line of defence against a whole class of bugs — at zero runtime cost.

- Primitive obsession? Use domain types, not raw `String` / `long` / `int`
- Using `Object`, raw `Map`, or `Any` where a specific type exists?
- Null handled safely? (`Optional`, nullability annotations, or null-safe wrappers)
- Unchecked casts present and unexplained?
- Closed value sets defined as `enum`, not magic strings or integer constants?
- **TypeScript?** `any` where `unknown` fits? Unsafe `as` casts instead of type guards? Missing discriminants on union types? Branded types missing where value mix-ups are possible?

**A well-typed API makes entire categories of bugs unrepresentable before the code runs.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Primitive obsession?</summary>

**Requirement:** Process a payment using an order ID and a customer ID.

❌ Bad — raw primitives carry no type information; wrong argument order compiles silently:

```java
public void processPayment(long orderId, long customerId, double amount) { ... }

// Caller passes arguments in the wrong order — no compile error:
processPayment(customerId, orderId, amount);
```

✅ Good — domain types make wrong argument order a compile error:

```java
public record OrderId(long value) {}
public record CustomerId(long value) {}
public record Money(double amount, Currency currency) {}

public void processPayment(OrderId orderId, CustomerId customerId, Money amount) { ... }

// Now this is a compile error — types do not match:
processPayment(customerId, orderId, amount); // ❌ CustomerId is not an OrderId
```

Domain types also carry business rules: a `Money` type can enforce non-negative values; a plain `double` cannot.

</details>

<details>
<summary>Using `Object`, raw `Map`, or `Any` where a specific type exists?</summary>

❌ Bad — caller has no idea what keys or value types to expect:

```java
public Map<String, Object> getUserProfile(Long userId) {
    Map<String, Object> profile = new HashMap<>();
    profile.put("name", "Jane");
    profile.put("age", 30);
    profile.put("premium", true);
    return profile;
}
```

✅ Good — typed return communicates the contract explicitly:

```java
public record UserProfile(String name, int age, boolean isPremium) {}

public UserProfile getUserProfile(Long userId) { ... }
```

Callers get compile-time field names and types. Renaming a field propagates through the compiler — instead of causing a runtime `null` or `ClassCastException`.

</details>

<details>
<summary>Null handled safely?</summary>

**Requirement:** Return a user's optional display name.

❌ Bad — nullable return leaks silently; callers forget to null-check:

```java
public String getDisplayName(Long userId) {
    return userRepository.findById(userId).getDisplayName(); // may return null
}
```

✅ Good — `Optional` signals the absence contract explicitly:

```java
public Optional<String> getDisplayName(Long userId) {
    return Optional.ofNullable(
        userRepository.findById(userId).getDisplayName()
    );
}
```

When `Optional` is not available (e.g. for performance-sensitive code or collection element types), use `@Nullable` / `@NonNull` annotations so static analysis tools and callers know what to expect.

</details>

<details>
<summary>Unchecked casts present and unexplained?</summary>

❌ Bad — unchecked cast suppressed silently; fails at runtime if the assumption ever breaks:

```java
@SuppressWarnings("unchecked")
public List<Order> getOrders(Object raw) {
    return (List<Order>) raw; // ClassCastException if raw contains something else
}
```

✅ Good — validate before casting, or redesign to avoid the cast entirely:

```java
public List<Order> getOrders(List<?> raw) {
    return raw.stream()
        .filter(item -> item instanceof Order)
        .map(item -> (Order) item)
        .collect(Collectors.toList());
}
```

If suppression is truly unavoidable, add a comment explaining exactly why the cast is safe and which invariant guarantees it.

</details>

<details>
<summary>Closed value sets defined as `enum`?</summary>

**Requirement:** Represent an order status of `PENDING`, `PROCESSING`, or `COMPLETED`.

❌ Bad — magic strings; typos compile, exhaustiveness is unenforced:

```java
public void updateStatus(String status) {
    if (status.equals("PENDNG")) { ... } // typo — compiles, silently does nothing
}
```

✅ Good — `enum` makes invalid values unrepresentable and switch exhaustiveness checkable:

```java
public enum OrderStatus { PENDING, PROCESSING, COMPLETED }

public void updateStatus(OrderStatus status) {
    switch (status) {
        case PENDING     -> ...;
        case PROCESSING  -> ...;
        case COMPLETED   -> ...;
        // compiler warns if a new enum value is added and not handled here
    }
}
```

</details>

<details>
<summary>TypeScript: advanced type system checks</summary>

TypeScript's type system is far more expressive than Java's — but that expressiveness can be misused. These checks apply on top of the general ones above.

### `any` vs `unknown`

❌ Bad — `any` silences all type checks; errors are deferred to runtime:

```typescript
function parseConfig(raw: any) {
    return raw.timeout * 1000; // no error even if raw is null or a string
}
```

✅ Good — `unknown` forces the caller to narrow before use:

```typescript
function parseConfig(raw: unknown) {
    if (typeof raw !== 'object' || raw === null || !('timeout' in raw)) {
        throw new Error('Invalid config');
    }
    return (raw as { timeout: number }).timeout * 1000;
}
```

### Discriminated unions and exhaustiveness

❌ Bad — no discriminant; branches require unsafe casts:

```typescript
type Shape = { width: number; height: number } | { radius: number };

function area(shape: Shape) {
    return (shape as any).radius
        ? Math.PI * (shape as any).radius ** 2
        : (shape as any).width * (shape as any).height;
}
```

✅ Good — discriminant field makes each branch type-safe; `never` catches unhandled variants at compile time:

```typescript
type Shape =
    | { kind: 'rect';   width: number; height: number }
    | { kind: 'circle'; radius: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case 'rect':   return shape.width * shape.height;
        case 'circle': return Math.PI * shape.radius ** 2;
        default: {
            const _exhaustive: never = shape; // compile error if a new variant is added and not handled here
            throw new Error(`Unhandled shape: ${_exhaustive}`);
        }
    }
}
```

### `as` casts vs type guards

❌ Bad — `as` bypasses the type system; wrong at runtime if the assumption breaks:

```typescript
function getUser(data: unknown): User {
    return data as User; // no structural check performed
}
```

✅ Good — a type guard validates first; the narrowed type inside is safe:

```typescript
function isUser(data: unknown): data is User {
    return typeof data === 'object' && data !== null && 'id' in data && 'email' in data;
}

function getUser(data: unknown): User {
    if (!isUser(data)) throw new Error('Invalid user payload');
    return data; // narrowed to User — no cast needed
}
```

### Branded / nominal types

❌ Bad — structural aliases are interchangeable; wrong IDs can be passed silently:

```typescript
type UserId    = string;
type ProductId = string;

function getProduct(id: ProductId) { ... }

const userId: UserId = 'usr_123';
getProduct(userId); // no error — UserId and ProductId are structurally identical
```

✅ Good — branded types prevent cross-domain value mix-ups at compile time:

```typescript
type UserId    = string & { readonly __brand: 'UserId' };
type ProductId = string & { readonly __brand: 'ProductId' };

function getProduct(id: ProductId) { ... }

const userId = 'usr_123' as UserId;
getProduct(userId); // ❌ compile error — UserId is not assignable to ProductId
```

### Generic constraints

❌ Bad — unconstrained `T`; the function must cast to access any property:

```typescript
function getLabel<T>(item: T): string {
    return (item as any).label; // forced cast — no compiler help
}
```

✅ Good — `T extends { label: string }` gives the compiler enough information to type-check the body:

```typescript
function getLabel<T extends { label: string }>(item: T): string {
    return item.label; // safe — T is guaranteed to have .label
}
```

</details>
</blockquote>

</details>

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

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>DB load increased?</summary>

**Scenario:** The PR adds a new background job that runs every 5 minutes and queries the `orders` table for all records modified in the last 24 hours — joining three tables with no pagination.

This query runs fine in staging with 10,000 rows. In production with 50 million rows, it locks rows, spikes CPU, and degrades latency for all other queries. Ask: does this add a new query pattern? What is the data volume? Does it need batching?

</details>

<details>
<summary>Index needed?</summary>

**Requirement:** Filter orders by status for an admin dashboard.

❌ Bad — full table scan on every request:

```java
public List<Order> findPendingOrders() {
    return orderRepository.findByStatus("PENDING"); // No index on 'status' — full scan at scale
}
```

✅ Good — back the query with an index via migration:

```sql
-- Add to your DB migration
CREATE INDEX idx_orders_status ON orders(status);
```

Any new column used in a `WHERE`, `JOIN`, or `ORDER BY` clause should be evaluated for an index.

</details>

<details>
<summary>Cache invalidation correct?</summary>

**Scenario:** The PR adds caching for a user's profile using `userId` as the cache key. But when a user updates their email, the cache is never invalidated. Users see stale data until the TTL expires.

Check: when data is mutated, is the corresponding cache entry evicted or updated? Are cache keys specific enough to avoid cross-user collisions?

</details>

<details>
<summary>Message queue impact?</summary>

**Scenario:** The PR publishes an event to a queue on every user action, including mouseover events captured from the frontend. At 100,000 concurrent users, this floods the queue with millions of low-value messages per minute, crowding out critical order events.

Ask: what is the publish rate at production scale? Are consumer groups correctly isolated? Is there a dead-letter queue for failed messages?

</details>

<details>
<summary>Cloud cost increased?</summary>

**Scenario:** The PR stores a full JSON snapshot of the user object in a DynamoDB record on every login. The object is 50KB and users log in frequently. At scale, this multiplies read/write costs significantly compared to storing only the fields actually needed.

Ask: does this change increase storage, read/write units, egress, or compute in a way that compounds at scale?

</details>

<details>
<summary>Monitoring/logging sufficient?</summary>

❌ Bad — failure is silent, no observability:

```java
public void processPayment(PaymentRequest request) {
    try {
        paymentGateway.charge(request);
    } catch (Exception e) {
        // swallowed — nobody knows this failed
    }
}
```

✅ Good — log with enough context to diagnose:

```java
public void processPayment(PaymentRequest request) {
    try {
        paymentGateway.charge(request);
        log.info("Payment processed: orderId={}, amount={}", request.getOrderId(), request.getAmount());
    } catch (Exception e) {
        log.error("Payment failed: orderId={}, reason={}", request.getOrderId(), e.getMessage(), e);
        throw e;
    }
}
```

</details>
</blockquote>

</details>

---

## Style & Formatting (Last)

> Only after everything else:

- Formatting
- Lint warnings
- Minor naming improvements

These should ideally be automated (lint, formatter, CI).
Humans should focus on thinking-level issues.

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Formatting</summary>

**Scenario:** A PR touches 3 lines of logic but shows 200 lines of diff because the author's IDE reformatted the entire file on save.

Formatting noise buries the real change and makes review harder. Enforce a shared formatter (e.g. Spotless, google-java-format) in CI so formatting is never a human concern in review. If a PR contains unrelated formatting changes, ask the author to separate them.

</details>

<details>
<summary>Lint warnings</summary>

**Scenario:** The PR introduces two new `@SuppressWarnings("unchecked")` annotations without explanation. The build passes, but the suppressed warnings hide potentially unsafe casts.

Lint warnings should be fixed, not suppressed. If suppression is truly necessary, it must be accompanied by a comment explaining why it is safe. Unaddressed lint warnings accumulate into technical debt.

</details>

<details>
<summary>Minor naming improvements</summary>

**Scenario:** A variable is named `data` in a method that processes payment responses. Renaming it to `paymentResponse` costs nothing and makes the code immediately clearer.

Minor naming improvements are worth a comment — but they should never block a PR. Suggest, don't require. Reserve blocking feedback for correctness, security, and architecture issues.

</details>
</blockquote>

</details>

---

## Implementation Intent & Alternative Analysis

> Step back from *how* and ask *why*.

- Why is the implementation this way?
- Why is the code placed here (this layer / class / module)?
- Why does this approach actually fulfill the goal?
- Self-investigate those questions before commenting
- Propose an alternative only when a concrete benefit exists

**Questioning intent surfaces hidden assumptions and overlooked solutions.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Why this approach?</summary>

Before accepting a solution, ask whether the chosen mechanism is the right tool for the problem. This is not about style — it is about whether there is a fundamentally better path.

Questions to form:

- Is this approach the simplest one that satisfies the requirement?
- Does it introduce accidental complexity (custom logic, extra abstractions) that a standard library or pattern would eliminate?
- Does it solve the problem in the right place, or is it working around a deeper issue?

**Scenario:** A PR adds a scheduled job that polls the `notifications` table every 10 seconds and sends pending emails.

*Why questions:* Why poll on a schedule? Why not react to the event that creates the notification row?

*Self-investigation:* The polling approach works but adds constant DB load and introduces latency proportional to the interval. The notification row is created as a result of a domain event — that event already exists in the system.

*Alternative:* Publish a `NotificationCreated` event when the row is inserted and consume it in an async listener. Emails are sent immediately, DB polling is eliminated, and the coupling between the scheduler and the table disappears.

❌ Original — polling approach:

```java
@Scheduled(fixedDelay = 10_000)
public void sendPendingNotifications() {
    List<Notification> pending = notificationRepository.findByStatus("PENDING");
    pending.forEach(n -> {
        emailService.send(n);
        n.setStatus("SENT");
        notificationRepository.save(n);
    });
}
```

✅ Alternative — event-driven:

```java
@TransactionalEventListener
public void onNotificationCreated(NotificationCreatedEvent event) {
    emailService.send(event.getNotification());
}
```

The alternative eliminates the polling loop, reduces DB load, and sends emails within milliseconds of the triggering action.

</details>

<details>
<summary>Why is it placed here?</summary>

Location carries meaning. A method in the wrong layer, class, or module misleads future readers about ownership and makes the code harder to test and reuse.

Questions to form:

- Why does this class own this responsibility?
- Why is this logic in this layer (controller / service / domain / infrastructure)?
- Why is this utility method duplicated here instead of living in a shared location?

**Scenario:** A PR adds a `calculateAge(LocalDate birthDate)` method directly inside `UserController`.

*Why questions:* Why is an age calculation — a pure domain computation — inside the HTTP layer? Who else might need this?

*Self-investigation:* Age calculation has no dependency on HTTP. It is pure business logic that could apply to patients, employees, or any person entity across multiple features. Placing it in the controller makes it unreachable without going through HTTP and invisible to other callers.

*Alternative:* Move it to the `User` domain object or a `DateUtils` shared utility.

❌ Original — logic buried in controller:

```java
@RestController
public class UserController {
    @GetMapping("/users/{id}/age")
    public int getUserAge(@PathVariable Long id) {
        User user = userService.findById(id);
        return Period.between(user.getBirthDate(), LocalDate.now()).getYears(); // domain logic in controller
    }
}
```

✅ Alternative — logic on the domain object:

```java
public class User {
    public int getAge() {
        return Period.between(this.birthDate, LocalDate.now()).getYears();
    }
}

@RestController
public class UserController {
    @GetMapping("/users/{id}/age")
    public int getUserAge(@PathVariable Long id) {
        return userService.findById(id).getAge(); // controller delegates
    }
}
```

Now the calculation is testable without the HTTP layer and reusable across the codebase.

</details>

<details>
<summary>Why does it fulfill the goal?</summary>

An implementation can be syntactically correct and architecturally sound, yet still fail to accomplish what the feature actually needs. This question closes the gap between intent and effect.

Questions to form:

- Does this implementation actually achieve the stated goal, or does it only appear to?
- Is the benefit delivered directly, or does it depend on assumptions that may not hold?
- Could the goal be achieved with a simpler mechanism?

**Scenario:** A PR adds an in-memory `HashMap` cache on `ProductService.getProduct()` to "speed up product lookups."

*Why questions:* Why does caching here help? Where is the actual bottleneck? The goal is "faster product lookups" — is this the right lever?

*Self-investigation:* `ProductService` is a Spring `@Service` (singleton). Its `HashMap` cache lives in one JVM instance. In a multi-instance deployment, each instance builds its own cache independently and cache hits are split across nodes. More critically, when a product is updated via a different endpoint, this cache is never invalidated — callers will see stale data indefinitely.

*Alternative:* Use a shared, TTL-based cache (Redis or Spring's `@Cacheable` with an eviction policy) so all instances share the same data and stale entries expire predictably.

❌ Original — unshared, never-invalidated cache:

```java
@Service
public class ProductService {
    private final Map<Long, Product> cache = new HashMap<>();

    public Product getProduct(Long id) {
        return cache.computeIfAbsent(id, productRepository::findById);
    }

    public void updateProduct(Product product) {
        productRepository.save(product); // cache is never cleared
    }
}
```

✅ Alternative — shared cache with automatic eviction:

```java
@Service
public class ProductService {
    @Cacheable("products")
    public Product getProduct(Long id) {
        return productRepository.findById(id);
    }

    @CacheEvict(value = "products", key = "#product.id")
    public void updateProduct(Product product) {
        productRepository.save(product);
    }
}
```

The alternative achieves the goal (faster lookups), works across instances, and keeps data consistent after updates.

</details>

<details>
<summary>How to self-investigate before commenting</summary>

Raising a "why" question as a review comment without investigating it first shifts work to the author without adding value. The reviewer's job is to investigate, not just interrogate.

**The process:**

1. **Form the question** — articulate the specific "why" clearly: *why this mechanism, why this location, why this fulfills the goal*.
2. **Investigate constraints** — check the PR description, linked ticket, and surrounding code. The author may have had a good reason: a framework limitation, a deadline, a dependency on another team's API.
3. **Identify the alternative** — if a better path exists, describe it concretely: what it looks like, what benefit it delivers, what trade-off it carries.
4. **Comment only if there is a finding** — if the investigation reveals the original choice was well-reasoned given the constraints, drop the question. If a genuine improvement exists, raise it with the alternative already sketched out.

**What a good alternative-based comment looks like:**

> I noticed `processOrder` polls the status table every 5 seconds. Given that status changes are triggered by an existing `OrderStatusChanged` event, we could consume that event directly and eliminate the poll. That would reduce DB load and cut the latency from ~5s to near-zero. Constraints I'm not aware of (e.g. the event not being reliable yet) might make polling safer for now — worth discussing?

This comment:

- States the observation
- Proposes a concrete alternative
- Acknowledges possible constraints
- Invites discussion rather than demanding a change

</details>
</blockquote>

</details>

---

## Reliability

> A reliable system continues to function correctly even when dependencies fail.

- Timeouts set on all external calls?
- Retry logic uses exponential backoff?
- Circuit breaker protects unstable dependencies?
- Graceful degradation when a dependency is unavailable?
- Health check endpoints accurately reflect dependency status?

**Without explicit fault tolerance, a single slow dependency can bring down the entire service.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Timeouts set on all external calls?</summary>

Every external call (HTTP, DB, cache, message broker) needs a timeout. A missing timeout means one slow downstream can exhaust the thread pool and freeze the entire service.

❌ Bad — no timeout; a slow downstream blocks threads indefinitely:

```java
RestTemplate restTemplate = new RestTemplate();
PaymentResponse response = restTemplate.postForObject(
    paymentUrl, request, PaymentResponse.class // No timeout — thread hangs if service is slow
);
```

✅ Good — connection and read timeouts configured:

```java
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
factory.setConnectTimeout(2_000); // 2s to establish connection
factory.setReadTimeout(5_000);    // 5s to receive full response
RestTemplate restTemplate = new RestTemplate(factory);
```

</details>

<details>
<summary>Retry logic uses exponential backoff?</summary>

**Scenario:** An API call to a third-party service occasionally fails with transient 503 errors. The PR retries three times in a tight loop with no delay. Under load, all callers retry simultaneously — creating a thundering herd that overwhelms the already-struggling service and prevents it from recovering.

❌ Bad — immediate retry with no delay; all callers hammer the service at once:

```java
for (int i = 0; i < 3; i++) {
    try {
        return externalService.call(request);
    } catch (TransientException e) {
        // retry immediately — thundering herd
    }
}
throw new ServiceUnavailableException();
```

✅ Good — exponential backoff with jitter (Spring Retry):

```java
@Retryable(
    value = TransientException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2, random = true) // 500ms → ~1s → ~1.5s with jitter
)
public Response callExternalService(Request request) {
    return externalService.call(request);
}
```

Jitter (random added to each delay) prevents synchronized retries across many instances — each caller backs off by a slightly different amount.

</details>

<details>
<summary>Circuit breaker protects unstable dependencies?</summary>

**Scenario:** A product page calls a recommendations service that starts failing. With no circuit breaker, every product page request waits for the full 5-second timeout before failing. Threads pile up. The product page — whose core functionality does not depend on recommendations — becomes completely unavailable.

❌ Bad — no circuit protection; every call waits for the full timeout:

```java
public List<Product> getRecommendations(Long userId) {
    return recommendationService.fetch(userId); // 5s timeout × every request → thread pool exhaustion
}
```

✅ Good — circuit opens after failure threshold; fallback returns immediately:

```java
@CircuitBreaker(name = "recommendations", fallbackMethod = "defaultRecommendations")
public List<Product> getRecommendations(Long userId) {
    return recommendationService.fetch(userId);
}

private List<Product> defaultRecommendations(Long userId, Throwable t) {
    return Collections.emptyList(); // fast fallback — page still loads without recommendations
}
```

After a configured failure threshold, the circuit opens and all calls return the fallback immediately — no timeouts, no thread waste. The circuit closes again after a cooldown period.

</details>

<details>
<summary>Graceful degradation when a dependency is unavailable?</summary>

**Scenario:** A PR adds a call to an inventory service inside the product listing endpoint. When the inventory service is down, the exception propagates and the entire product listing returns 500 — even though showing inventory status is supplemental, not essential to the feature.

Ask: if this dependency is unavailable, what is the minimum useful response the system can still provide? Return cached data, defaults, or a partial response rather than failing the whole request.

Non-essential dependencies should never take down a core user flow.

</details>

<details>
<summary>Health check endpoints accurately reflect dependency status?</summary>

**Scenario:** The service exposes `/actuator/health` which always returns `{"status": "UP"}`. But the database connection pool is exhausted and no requests can actually be served. Kubernetes keeps routing traffic to the failing pod.

Check: does the health endpoint verify real dependencies (DB connectivity, cache availability, critical downstream services)? A health endpoint that always returns `UP` is worse than no health check — it hides failures from orchestrators and on-call engineers.

In Spring Boot, configure dependency health indicators so `DOWN` propagates correctly:

```yaml
management:
  health:
    db:
      enabled: true
    redis:
      enabled: true
```

</details>
</blockquote>

</details>

---

## Operational Excellence

> Code that cannot be safely deployed, observed, or diagnosed under failure is not production-ready.

- Structured logs with correlation IDs?
- Distributed trace spans added for new cross-service calls?
- Alerting or SLO defined for new critical endpoints?
- API changes backward compatible or versioned?
- New failure modes identified?

**Operational excellence determines how quickly you detect and recover — not whether failures happen.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Structured logs with correlation IDs?</summary>

In a multi-threaded or multi-service system, a single user request produces log lines across many classes and threads. Without a correlation ID, diagnosing a production failure means manually sifting thousands of interleaved entries.

❌ Bad — unstructured string, no request context:

```java
log.info("Processing order for user " + userId + ", amount: " + amount);
// 200 threads log simultaneously — no way to find all lines from one request
```

✅ Good — structured fields + correlation ID via MDC:

```java
MDC.put("correlationId", request.getCorrelationId());
log.info("Processing order: userId={}, amount={}", userId, amount);
// Every subsequent log line in this request thread automatically carries correlationId
// Filterable in Elasticsearch or CloudWatch by correlationId field
```

Key properties of production-ready logs:

- **Structured** (key=value or JSON) — filterable by field, not just full-text search
- **Correlation ID** — links all log lines from one request across services
- **Correct log level** — `ERROR` for actionable failures, `WARN` for recoverable issues, `INFO` for key business events, `DEBUG` for troubleshooting detail

</details>

<details>
<summary>Distributed trace spans added for new cross-service calls?</summary>

**Scenario:** A PR adds a call to an inventory service inside the checkout flow. No trace span wraps the call. In production, checkout latency spikes. The Jaeger/Zipkin trace shows the checkout endpoint taking 3s total — but no breakdown of where the time is spent. The inventory call is invisible.

Check: for each new cross-service call (HTTP, gRPC, message publish), is a trace span created? With Spring Cloud Sleuth or Micrometer Tracing, HTTP client calls are often instrumented automatically — but custom async tasks or thread hand-offs may require explicit spans:

```java
Span span = tracer.nextSpan().name("inventory-check").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    return inventoryService.checkAvailability(productId);
} finally {
    span.end();
}
```

</details>

<details>
<summary>Alerting or SLO defined for new critical endpoints?</summary>

**Scenario:** A new POST /payments endpoint is deployed with no error-rate alert. Three days later, a payment gateway config change causes 30% of payment requests to fail silently. No alert fires. The engineering team learns about it from customer complaints 4 hours later.

Ask: is this endpoint on a critical user path? If yes, before deploy:

- Define an error rate alert (e.g., >1% 5xx over 5 minutes → page on-call)
- Define a latency SLO (e.g., p99 < 2s)
- Add a dashboard panel for error rate and latency

The PR itself may not configure alerts, but reviewers should flag the absence when the stakes are high.

</details>

<details>
<summary>API changes backward compatible or versioned?</summary>

**Scenario:** A field in a response JSON is renamed from `userName` to `username`. The change is deployed. Existing mobile clients — not force-updated — start receiving `null` for the field they rely on.

❌ Bad — field renamed; existing clients break immediately:

```java
// Before: { "userName": "john_doe" }
public record UserResponse(String username) {} // renamed — old clients receive null for "userName"
```

✅ Good — old field kept; clients migrate on their own schedule:

```java
public record UserResponse(
    String username,             // new canonical name
    @Deprecated String userName  // kept for backward compatibility until clients migrate
) {}
```

Ask:

- Does this rename or remove a request/response field?
- Does it change the semantics of an existing field?
- Does it add a required field where one was previously optional?

Any yes requires a versioning strategy: field aliasing, API version suffix (`/v2/`), or a migration period with both names present.

</details>

<details>
<summary>New failure modes identified?</summary>

**Scenario:** A PR adds a background job that refunds orders. The job updates the DB first, then calls the bank API. If the bank call fails after the DB is updated, the order is marked refunded but no money was actually sent. This is a new failure mode — partial state — that did not exist before.

When a PR introduces a new failure mode, especially one that can cause data inconsistency or silent data loss, reviewers should ask:

- How will this be detected? (error log, alert, reconciliation job)
- How will it be remediated? (manual re-run, automated retry, compensating transaction)

If the PR has no answer, raise it: *"If the bank call fails after the DB update, how do we detect and reconcile the inconsistency?"*

</details>
</blockquote>

</details>

---

## Cost Optimization

> Wasteful patterns compound at scale — a slow query or over-fetched payload that seems harmless at low traffic can generate significant cloud spend at production volume.

- Only the columns/fields actually needed are fetched?
- Data transfer and egress costs considered?
- Compute right-sized for the workload pattern?
- Redundant downstream calls reduced through caching?
- Batch operations used instead of per-record API calls?

**Cost is a non-functional requirement. At scale, the wrong access pattern is both a performance and a budget problem.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Only the columns/fields actually needed are fetched?</summary>

**Requirement:** Send a welcome email to all users.

❌ Bad — fetches the entire entity including large blobs, when only two fields are needed:

```java
List<User> users = userRepository.findAll(); // SELECT * — fetches avatar, preferences, audit history
for (User user : users) {
    emailService.sendWelcome(user.getEmail(), user.getName()); // only uses 2 fields
}
```

✅ Good — projection fetches only the needed columns:

```java
public interface UserEmailView {
    String getName();
    String getEmail();
}

List<UserEmailView> users = userRepository.findAllProjectedBy(); // SELECT name, email FROM users
```

Each extra column adds database I/O, network bytes between DB and app server, and JVM heap pressure. At millions of rows, the difference in latency, memory, and cost is significant.

</details>

<details>
<summary>Data transfer and egress costs considered?</summary>

**Scenario:** A new list endpoint embeds the full product catalog as a nested JSON array in each order response — sometimes 500+ products per response. The endpoint is called from a mobile app that only displays the order status and total.

At 10,000 daily requests, each returning 200KB of unused nested data, that is 2GB of egress per day billed at cloud provider rates.

Ask:

- What fields does the consumer actually display or use?
- Are large nested objects included by default when callers could opt in instead?
- Is the response payload proportional to what the caller needs?

Consider sparse fieldsets, projection parameters, or separate endpoints for summary vs. detail views.

</details>

<details>
<summary>Compute right-sized for the workload pattern?</summary>

**Scenario:** A reconciliation job runs once per hour for about 30 seconds. It is implemented as a long-running thread in the main application service — consuming memory and a thread slot continuously, even during the 59 minutes it does nothing.

Match the compute model to the workload:

| Workload | Better fit |
| --- | --- |
| Triggered by event, short-lived | Lambda / Fargate task |
| Scheduled, short duration | Scheduled Lambda / Kubernetes CronJob |
| High-throughput, continuous | Always-on service (EC2, ECS) |
| Heavy batch, periodic | AWS Batch / Kubernetes Job |

Embedding infrequent jobs in an always-on service wastes compute. Conversely, using Lambda for a 10,000-TPS hot path introduces cold-start latency and per-invocation costs that may exceed a dedicated instance.

</details>

<details>
<summary>Redundant downstream calls reduced through caching?</summary>

**Scenario:** A product details endpoint is called 100 times per second. For each call, it fetches the seller profile from an external Seller Service — despite seller profiles changing at most a few times per day. No cache is in place.

At 100 RPS, that is 8,640,000 external calls per day for data that almost never changes.

Ask: is this data read far more often than it changes? If yes, even a short TTL (60 seconds) reduces external calls by 99.9% with negligible staleness risk.

```java
@Cacheable(value = "sellerProfiles", key = "#sellerId")
public SellerProfile getSellerProfile(Long sellerId) {
    return sellerService.fetchProfile(sellerId); // cached for 60s; 1 call per minute per seller
}
```

</details>

<details>
<summary>Batch operations used instead of per-record API calls?</summary>

**Scenario:** A PR adds a notification feature that calls the push notification service once per user in a loop.

❌ Bad — N individual HTTP calls for N users:

```java
for (User user : usersToNotify) {
    notificationService.sendPush(user.getId(), message); // 1 HTTP call per user
}
```

✅ Good — single batch call:

```java
List<Long> userIds = usersToNotify.stream().map(User::getId).collect(Collectors.toList());
notificationService.sendPushBatch(userIds, message); // 1 HTTP call for all users
```

This is the external-API equivalent of the N+1 DB query. Each unnecessary call adds network latency, connection overhead, and often API rate-limit or per-call billing pressure.

</details>
</blockquote>

</details>

---

## Sustainability

> Sustainable software does more with less — fewer CPU cycles, less memory, fewer network round trips, and resources released promptly when no longer needed.

- Resources (connections, streams, threads) explicitly released after use?
- No redundant computation inside hot loops?
- Efficient data structures chosen for the access pattern?
- Batch API calls used instead of per-record requests?
- Background jobs scoped to run only as long as needed?

**Sustainable code reduces cloud cost, carbon footprint, and system load simultaneously.**

<details>
<summary>Further details</summary>

<blockquote>
<details>
<summary>Resources explicitly released after use?</summary>

**Requirement:** Read a configuration file on startup.

❌ Bad — stream never closed; OS file handles leak under repeated calls:

```java
public String readConfig(String path) throws IOException {
    InputStream stream = new FileInputStream(path);
    return new String(stream.readAllBytes()); // stream never closed
}
```

✅ Good — try-with-resources guarantees cleanup on exit or exception:

```java
public String readConfig(String path) throws IOException {
    try (InputStream stream = new FileInputStream(path)) {
        return new String(stream.readAllBytes());
    }
}
```

Resource leaks compound under load: each leaked DB connection reduces pool availability until new requests cannot obtain a connection and the service stalls. Apply the same pattern to DB connections, HTTP clients, file handles, and thread pool executors.

</details>

<details>
<summary>No redundant computation inside hot loops?</summary>

**Requirement:** Filter orders that meet the active promotion's minimum order value.

❌ Bad — expensive DB call made on every iteration even though the result does not change:

```java
public List<Order> filterByActivePromotion(List<Order> orders) {
    List<Order> result = new ArrayList<>();
    for (Order order : orders) {
        Promotion active = promotionService.getActivePromotion(); // DB call on every iteration
        if (order.getTotal() >= active.getMinimumOrderValue()) {
            result.add(order);
        }
    }
    return result;
}
```

✅ Good — fetch once outside the loop, reuse:

```java
public List<Order> filterByActivePromotion(List<Order> orders) {
    Promotion active = promotionService.getActivePromotion(); // fetched once
    return orders.stream()
        .filter(order -> order.getTotal() >= active.getMinimumOrderValue())
        .collect(Collectors.toList());
}
```

A single extra DB call per iteration is negligible at 10 items — at 10,000 items and 100 RPS, it becomes 1,000,000 unnecessary DB calls per second.

</details>

<details>
<summary>Efficient data structures chosen for the access pattern?</summary>

**Requirement:** Filter out blacklisted products from a list.

❌ Bad — `List.contains()` is O(n) per check; total complexity O(n²):

```java
public List<Product> filterBlacklisted(List<Product> products, List<Long> blacklistedIds) {
    return products.stream()
        .filter(p -> !blacklistedIds.contains(p.getId())) // O(n) contains inside O(n) stream
        .collect(Collectors.toList());
}
```

✅ Good — convert to `Set` once; each lookup is O(1):

```java
public List<Product> filterBlacklisted(List<Product> products, List<Long> blacklistedIds) {
    Set<Long> blacklistSet = new HashSet<>(blacklistedIds); // convert once — O(n)
    return products.stream()
        .filter(p -> !blacklistSet.contains(p.getId()))     // O(1) per lookup
        .collect(Collectors.toList());
}
```

Inefficient data structures waste CPU cycles, which translates directly to compute cost and energy consumption at scale.

</details>

<details>
<summary>Background jobs scoped to run only as long as needed?</summary>

**Scenario:** A PR adds a `@Scheduled` job that processes a queue every 5 minutes. After processing, the job does nothing for the remaining 5 minutes — but still holds a thread and a DB connection open between runs.

Ask:

- Does this job hold resources between executions?
- Could it be triggered on-demand (event-driven) instead of on a schedule?
- If scheduled, does it release all resources immediately after completing its work?

An event-driven approach (consuming from a queue only when messages exist) avoids idle resource consumption entirely. If polling is necessary, acquire resources at the start of each execution and release them at the end — not held open between runs.

</details>
</blockquote>

</details>
