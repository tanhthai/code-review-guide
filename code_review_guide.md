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
<summary>Business logic separated? / Pure logic extractable?</summary>

**Scenario:** A controller method fetches data, applies pricing rules, formats the response, and sends an audit event — all inline. There is no way to test the pricing logic without spinning up the full HTTP stack.

When business logic is embedded inside framework callbacks (controllers, listeners, batch processors), it cannot be unit-tested in isolation. Extract it into a plain service or domain method that takes inputs and returns outputs, with no framework dependencies.

</details>

<details>
<summary>Mocking straightforward?</summary>

**Scenario:** A service depends on `new EmailClient("smtp.internal", 587, true, "user", "pass")` constructed inline. To test the service, you'd need a real SMTP server.

If mocking a dependency requires more setup than the test itself, the dependency is too tightly bound. Depend on an interface (`EmailSender`), inject it, and mock the interface in tests with a single line.

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
<summary>Failure cases tested?</summary>

**Scenario:** The PR adds tests for the happy path — a successful payment. But there are no tests for what happens when the payment gateway is down, when the card is declined, or when the request times out.

Failure paths are where bugs cause the most damage. For every success test, ask: what is the corresponding failure test?

</details>

<details>
<summary>Edge cases tested?</summary>

**Scenario:** A `calculateDiscount()` method is tested with a typical order of $50. But there are no tests for a $0 order, a negative price, or an empty item list.

Ask: what are the boundary values? What happens at zero, null, empty, max, and min?

</details>

<details>
<summary>Main flows covered?</summary>

**Scenario:** A checkout feature is merged with tests only for the pricing logic. The full flow — add to cart → apply coupon → pay → receive confirmation — has no end-to-end or integration test.

If a main user journey has no test coverage, a future change anywhere in that path can break it silently.

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

<details>
<summary>No dead code?</summary>

**Scenario:** The PR contains a `legacyCalculateShipping()` method that is never called anywhere. It was replaced three months ago but never deleted.

Dead code creates confusion — future readers wonder if it's intentional, if it's needed, or if removing it will break something. If it is unused, delete it. Version control preserves history.

</details>

<details>
<summary>Methods not too long?</summary>

**Scenario:** A `processCheckout()` method is 200 lines long. It validates input, applies coupons, calculates tax, charges the card, sends an email, and updates inventory — all inline.

A method should do one thing. If you need to scroll to read it, or if you need a comment to mark sections ("// step 1", "// step 2"), it should be split. Each extracted method becomes independently readable and testable.

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

<details>
<summary>DB load increased?</summary>

**Scenario:** The PR adds a new background job that runs every 5 minutes and queries the `orders` table for all records modified in the last 24 hours — joining three tables with no pagination.

This query runs fine in staging with 10,000 rows. In production with 50 million rows, it locks rows, spikes CPU, and degrades latency for all other queries. Ask: does this add a new query pattern? What is the data volume? Does it need batching?

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
