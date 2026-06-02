# コードレビューガイド

レビューフィードバックを提起する際は、ビジネスリスクが高い順に優先してください。スタイルやフォーマットの問題が、正確性やセキュリティの問題をブロックすることがあってはなりません。

1. [要件とビジネス正確性](#要件とビジネス正確性最初に確認)
2. [セキュリティ](#セキュリティ高リスク領域)
3. [パフォーマンスとシステムリスク](#パフォーマンスとシステムリスク)
4. [スケーラビリティと並行性](#スケーラビリティと並行性)
5. [アーキテクチャと設計](#アーキテクチャと設計)
6. [テスタビリティ](#テスタビリティ設計品質の指標)
7. [テストカバレッジとテスト品質](#テストカバレッジとテスト品質)
8. [可読性と保守性](#可読性と保守性)
9. [型チェック](#型チェック)
10. [インフラとコスト影響](#インフラとコスト影響上級レビュー)
11. [スタイルとフォーマット](#スタイルとフォーマット最後に)
12. [実装意図と代替案の分析](#実装意図と代替案の分析)
13. [信頼性](#信頼性)
14. [オペレーショナルエクセレンス](#オペレーショナルエクセレンス)
15. [コスト最適化](#コスト最適化)
16. [持続可能性](#持続可能性)

---

## 要件とビジネス正確性（最初に確認）

> これが失敗すれば、他はすべて無意味です。

- 正しい問題を解決しているか？
- 受け入れ基準はすべて満たされているか？
- 変更はPRのスコープと一致しているか？（作業漏れなし、スコープクリープなし）
- エッジケースと失敗パスは要件で明示的にカバーされているか？
- ビジネス・ドメインルールに従っているか？
- SLA・コンプライアンス制約・仕様書のデータ量見込みは守られているか？
- 既存の動作への回帰はないか？

**なぜ最初なのか？** 間違った問題を完璧に解決したコードは、依然として間違っているからです。

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>正しい問題を解決しているか？</summary>

**要件:** プレミアムユーザーには20%割引、一般ユーザーには10%割引を適用する。

❌ 悪い例 — 間違った問題を解決している:

```java
public double calculateDiscount(User user, double price) {
    return price * 0.10; // 常に10% — ユーザータイプを完全に無視
}
```

✅ 良い例 — 正しい問題を解決している:

```java
public double calculateDiscount(User user, double price) {
    if (user.isPremium()) {
        return price * 0.20;
    }
    return price * 0.10;
}
```

悪い例はコンパイルが通り、見た目もきれいですが、ビジネス要件を暗黙的に違反しています。

**ヒント — シナリオマトリクスアプローチ:** コードをレビューする前に、要件から想定されるすべてのシナリオを洗い出し、それぞれについてコードを追ってみてください。

| シナリオ | 期待値 | コードは対応しているか？ |
|---|---|---|
| プレミアムユーザー・有効な価格 | 20%割引 | ✅ |
| 一般ユーザー・有効な価格 | 10%割引 | ✅ |
| プレミアムユーザー・価格が0 | 0 | ? |
| nullユーザー | エラー / ハンドリング済み | ? |
| 負の価格 | エラー / ハンドリング済み | ? |

「?」のセルがレビューコメントになります。「正しい問題を解決しているか？」という曖昧な問いを、具体的なチェックリストに変えることができます。

</details>

<details>
<summary>受け入れ基準はすべて満たされているか？</summary>

**チケットAC:** 「ログイン失敗時にエラーメッセージを表示する。」

PRはログインロジックを追加し、失敗時に`401`ステータスを返します。しかし、ユーザー向けエラーメッセージはどこにもレンダリングされていません。コードは技術的に正しいですが、ACは半分しか満たされていません。バックエンドロジックだけを読むレビュアーはこれを見逃します。

各AC行を実際の変更に対して一行ずつ確認してください。

</details>

<details>
<summary>変更はPRのスコープと一致しているか？</summary>

**チケット:** 「ユーザープロフィールページのヌルポインター例外を修正する。」

PRはバグを修正しますが、「ついでに」認証サービスもリファクタリングしています。この認証リファクタリングはレビューされておらず、チケットもなく、無関係な領域にリスクを追加しています。これがスコープクリープです。

逆もあります: チケットがAPIとUIの両方の更新を要求しているのに、APIだけ変更されている場合は作業漏れです。

</details>

<details>
<summary>エッジケースと失敗パスは明示的にカバーされているか？</summary>

**要件:** 注文を処理し、合計金額を計算する。

❌ 悪い例 — nullと空入力を無視している:

```java
public void processOrder(List<Item> items) {
    double total = 0;
    for (Item item : items) {
        total += item.getPrice(); // itemsがnullの場合NullPointerException
    }
    checkout(total); // 空の注文でも呼ばれる
}
```

✅ 良い例 — エッジケースを明示的にガードしている:

```java
public void processOrder(List<Item> items) {
    if (items == null || items.isEmpty()) {
        throw new IllegalArgumentException("注文には少なくとも1つの商品が必要です");
    }
    double total = 0;
    for (Item item : items) {
        if (item.getPrice() < 0) {
            throw new IllegalArgumentException("商品価格は負の値にできません: " + item.getName());
        }
        total += item.getPrice();
    }
    checkout(total);
}
```

確認すべき一般的なカテゴリ:

- null / 空入力
- ゼロ、負の値、または境界値
- 最大サイズ / 非常に大きな入力
- 重複または繰り返し値
- 共有状態への同時アクセス
- 外部依存が使用不能（タイムアウト、エラー）

</details>

<details>
<summary>ビジネス・ドメインルールに従っているか？</summary>

**要件:** ユーザーは資金を引き出せるが、常に$10の最低残高を維持する必要がある。

❌ 悪い例 — 利用可能残高のみ確認し、ドメインルールを見逃している:

```java
public void withdraw(Account account, double amount) {
    if (amount > account.getBalance()) {
        throw new InsufficientFundsException();
    }
    account.debit(amount);
}
```

✅ 良い例 — 最低残高ルールを強制している:

```java
public void withdraw(Account account, double amount) {
    if (amount <= 0) {
        throw new IllegalArgumentException("引き出し金額は正の値でなければなりません");
    }
    if (account.getBalance() - amount < 10.0) {
        throw new InsufficientFundsException("$10の最低残高を維持する必要があります");
    }
    account.debit(amount);
}
```

ドメインルールはしばしば暗黙的です — レビュアーはコードを読むだけでなく、仕様を知っている必要があります。

</details>

<details>
<summary>SLA・コンプライアンス制約・データ量見込みは守られているか？</summary>

**仕様:** 「検索APIは最大100,000件のレコードに対して300ms以内に結果を返す必要がある。」

PRは、500件のレコードがある開発環境では正常に動作する新しいフィルターを追加しますが、インデックスなしでフルテーブルスキャンを実行します。本番環境の100,000件のレコードでは、SLAを超過します。

仕様のパフォーマンス予算、データ量目標、保持ルール、またはコンプライアンス要件（GDPR、PCI、HIPAA）を確認し、コードがそれらを守っているか検証してください。

</details>

<details>
<summary>既存の動作への回帰はないか？</summary>

**シナリオ:** 共有ユーティリティメソッドが、デフォルト値を持つ新しいオプションパラメーターをサポートするように更新されます。別のモジュールの呼び出し元が以前のデフォルト動作に依存していた場合、コンパイルエラーも失敗テストもなく、誤った出力を黙って受け取ります。

問い: この変更は、このPRで変更されたファイル以外の現在動作しているものを壊す可能性があるか？

</details>
</blockquote>

</details>

---

## セキュリティ（高リスク領域）

> セキュリティ問題は即座に本番環境に損害を与える可能性があります。

- 入力バリデーション + 出力エスケープ（XSS）
- 認証（AuthN）+ 認可（AuthZ）（IDORを含む）
- インジェクション: SQL + コマンド + NoSQL
- CSRF（クッキーベース認証の場合）
- 機密データ + シークレット + 安全なログ
- SSRF / アウトバウンドURL取得の安全性
- ファイルアップロードの安全性
- レート制限
- 依存関係・設定のリスク

**セキュリティは早期にレビューする必要があります — 後回しにしてはなりません。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>入力バリデーション + 出力エスケープ（XSS）</summary>

**入力バリデーション** — 不正な入力がシステムに入る前に拒否する:

❌ 悪い例 — バリデーションなし、何でも受け付ける:

```java
@PostMapping("/comments")
public void submitComment(String comment) {
    commentRepository.save(new Comment(comment)); // null、空白、または10,000文字の入力も受け付ける
}
```

✅ 良い例 — 処理前にバリデーションする:

```java
@PostMapping("/comments")
public void submitComment(String comment) {
    if (comment == null || comment.isBlank()) {
        throw new IllegalArgumentException("コメントは空であってはなりません");
    }
    if (comment.length() > 500) {
        throw new IllegalArgumentException("コメントは500文字を超えてはなりません");
    }
    commentRepository.save(new Comment(comment));
}
```

一般的なバリデーションチェック:

- null / 空白
- 長さ（最小 / 最大）
- フォーマット（メール、電話、UUID、日付）
- 数値範囲（最小 / 最大値）
- 許可リスト（許可された値のみ、例: enum、国コード）
- ファイルタイプ + サイズ
- 危険なパターンなし（SQLキーワード、スクリプトタグ、パストラバーサル `../`）

**出力エスケープ** — XSSを防ぐためにレンダリング前にエンコードする:

❌ 悪い例 — 生のユーザー入力をHTMLに埋め込む:

```java
public String renderComment(String comment) {
    return "<p>" + comment + "</p>"; // XSS: 攻撃者が <script>stealCookies()</script> を送信
}
```

✅ 良い例 — レンダリング前にエスケープする:

```java
public String renderComment(String comment) {
    return "<p>" + HtmlUtils.htmlEscape(comment) + "</p>";
}
```

</details>

<details>
<summary>認証（AuthN）+ 認可（AuthZ）（IDORを含む）</summary>

- **AuthN（認証）** — ユーザーが*誰であるか*を確認する（ログイン、トークン検証）
- **AuthZ（認可）** — ユーザーが*何をすることが許可されているか*を確認する
- **IDOR（安全でない直接オブジェクト参照）** — リクエスト内のIDを操作することで、ユーザーが別のユーザーのリソースにアクセスするAuthZの失敗

**要件:** ユーザーが自分の注文を閲覧できるようにする。

❌ 悪い例 — 所有権チェックなし（IDOR）:

```java
// GET /api/orders/{orderId}
public Order getOrder(Long orderId) {
    return orderRepository.findById(orderId); // どのユーザーでもどの注文にもアクセスできる
}
```

✅ 良い例 — リソースが呼び出し元に属することを確認する:

```java
public Order getOrder(Long orderId, User currentUser) {
    Order order = orderRepository.findById(orderId)
        .orElseThrow(() -> new NotFoundException("注文が見つかりません"));
    if (!order.getOwnerId().equals(currentUser.getId())) {
        throw new ForbiddenException("アクセスが拒否されました");
    }
    return order;
}
```

</details>

<details>
<summary>インジェクション: SQL + コマンド + NoSQL</summary>

**SQLインジェクション** — ユーザー入力がクエリ文字列に連結される:

❌ 悪い例 — 文字列の連結がSQLインジェクションを可能にする:

```java
public User findByUsername(String username) {
    String query = "SELECT * FROM users WHERE username = '" + username + "'";
    return jdbcTemplate.queryForObject(query, userRowMapper);
}
```

✅ 良い例 — パラメーター化クエリ:

```java
public User findByUsername(String username) {
    return jdbcTemplate.queryForObject(
        "SELECT * FROM users WHERE username = ?",
        userRowMapper,
        username
    );
}
```

**コマンドインジェクション** — ユーザー入力がシェルコマンドに直接渡される:

❌ 悪い例 — 攻撃者が任意のシェルコマンドを追加できる:

```java
public String ping(String host) throws IOException {
    Process process = Runtime.getRuntime().exec("ping -c 1 " + host); // 攻撃者が渡す: google.com; rm -rf /
    return new String(process.getInputStream().readAllBytes());
}
```

✅ 良い例 — 入力をバリデーションし、シェル解釈を回避するために引数を配列として渡す:

```java
public String ping(String host) throws IOException {
    if (!host.matches("^[a-zA-Z0-9.-]+$")) {
        throw new IllegalArgumentException("無効なホスト");
    }
    Process process = new ProcessBuilder("ping", "-c", "1", host).start();
    return new String(process.getInputStream().readAllBytes());
}
```

**NoSQLインジェクション** — ユーザー入力が生のクエリドキュメントに埋め込まれる:

❌ 悪い例 — 攻撃者が`{"$gt": ""}`を渡してusernameマッチをバイパスする:

```java
public Document findUser(String username) {
    String jsonQuery = "{\"username\": \"" + username + "\"}";
    return collection.find(Document.parse(jsonQuery)).first(); // オペレーターインジェクション
}
```

✅ 良い例 — 型付きクエリビルダーを使用。入力は常に値として扱われ、オペレーターとしては扱われない:

```java
public Document findUser(String username) {
    return collection.find(Filters.eq("username", username)).first();
}
```

</details>

<details>
<summary>CSRF（クッキーベース認証の場合）</summary>

**シナリオ:** CSRFトークン検証なしで新しいPOST /transferエンドポイントが追加されます。アプリはクッキーベース認証を使用しています。

攻撃者は悪意のあるサイトに非表示の自動送信フォームを埋め込みます。ログイン中のユーザーがそのページを訪問すると、ブラウザはクッキーとともにリクエストを送信し、ユーザーの知らないうちに送金が実行されます。

確認: エンドポイントは状態変更を行うか？アプリはクッキー認証を使用しているか？YESなら、CSRF保護が必要です。

一般的な保護方法:

- **CSRFトークン** — サーバーがセッションごとのトークンを生成し、フォーム/ヘッダー（`X-CSRF-Token`）に埋め込み、すべての状態変更リクエストで検証する（Spring Securityのデフォルト）
- **SameSiteクッキー** — セッションクッキーに`SameSite=Strict`または`SameSite=Lax`を設定し、ブラウザがクロスオリジンリクエストをブロックするようにする
- **カスタムリクエストヘッダー** — クロスオリジンフォームPOSTでブラウザが自動的に追加できないヘッダー（例: `X-Requested-With: XMLHttpRequest`）を必須にする
- **ダブルサブミットクッキー** — CSRFトークンをクッキーとリクエストパラメーターの両方として送信し、サーバー側で一致を検証する

</details>

<details>
<summary>機密データ + シークレット + 安全なログ</summary>

**要件:** 監査のために認証試行をログに記録する。

❌ 悪い例 — 認証情報がログに表示される:

```java
public void authenticate(String username, String password) {
    log.info("ログイン試行: user={}, password={}", username, password); // パスワードがログファイルに露出
}
```

✅ 良い例 — 安全なものだけをログに記録する:

```java
public void authenticate(String username, String password) {
    log.info("ログイン試行: user={}", username); // パスワード、トークン、シークレットは絶対にログに記録しない
}
```

ログに記録・露出してはならないもの:

- パスワード / パスフレーズ
- APIキー / シークレットトークン / JWTシークレット
- セッションID / OAuthトークン / リフレッシュトークン
- クレジットカード番号 / CVV / 銀行口座番号
- マイナンバー / 国民ID / パスポート番号
- 秘密鍵 / 証明書
- 高リスクコンテキストのPII（医療記録、政府ID）

</details>

<details>
<summary>SSRF / アウトバウンドURL取得の安全性</summary>

- **SSRF（サーバーサイドリクエストフォージェリ）** — 攻撃者がサーバーを騙して代わりにHTTPリクエストを作成させる攻撃。インターネットからアクセスできない内部サービスを標的にすることが多い
- **アウトバウンドURL取得の安全性** — SSRFや関連する悪用を防ぐために、サーバーが取得するURLをリクエスト前にバリデーションする実践

**要件:** ユーザーが提供したURLを取得してリンクプレビューを生成する。

❌ 悪い例 — 内部サービスを含む任意のURLを取得する:

```java
public String fetchPreview(String url) throws IOException {
    return new URL(url).openStream().toString(); // SSRF: 攻撃者が http://169.254.169.254/latest/meta-data/ を渡す
}
```

✅ 良い例 — スキームを検証し、プライベートアドレスをブロックする:

```java
public String fetchPreview(String url) throws IOException {
    URI uri = URI.create(url);
    if (!List.of("http", "https").contains(uri.getScheme()) || isPrivateAddress(uri.getHost())) {
        throw new IllegalArgumentException("URLは許可されていません");
    }
    return httpClient.get(url);
}
```

この例はプライベートアドレスをブロックしていますが、本番対応の実装では以下も確認する必要があります:

- **スキーム** — `http` / `https`のみを許可。`file://`、`ftp://`、`gopher://`等を拒否
- **リダイレクト** — 各リダイレクト後に宛先を再検証。公開URLが内部アドレスにリダイレクトする可能性がある
- **レスポンスサイズ** — 大きなペイロードによるメモリ枯渇を防ぐために最大バイト制限を強制する
- **タイムアウト** — 遅いレスポンスの悪用を防ぐために短い接続 + 読み取りタイムアウトを設定する
- **ドメイン許可リスト** — ユースケースが許す場合は、既知の信頼されたドメインのセットのみに制限する

</details>

<details>
<summary>ファイルアップロードの安全性</summary>

**要件:** ユーザーがプロフィール画像をアップロードできるようにする。

❌ 悪い例 — タイプチェックなし、サイズ制限なし、パストラバーサルリスク:

- **パストラバーサル** — ユーザー提供のファイル名に`../`シーケンスが含まれ、意図したディレクトリから脱出してサーバーの任意の場所にファイルを書き込む攻撃（例: `../../etc/cron.d/backdoor`）

```java
public void uploadFile(MultipartFile file) throws IOException {
    Path path = Paths.get("/uploads/" + file.getOriginalFilename()); // パストラバーサル: ../../../etc/passwd
    Files.write(path, file.getBytes()); // タイプやサイズのバリデーションなし
}
```

✅ 良い例 — タイプ、サイズを検証し、安全なファイル名を使用する:

```java
private static final Set<String> ALLOWED_TYPES = Set.of("image/jpeg", "image/png");
private static final long MAX_SIZE = 5 * 1024 * 1024; // 5MB

public void uploadFile(MultipartFile file) throws IOException {
    if (!ALLOWED_TYPES.contains(file.getContentType())) {
        throw new IllegalArgumentException("ファイルタイプは許可されていません");
    }
    if (file.getSize() > MAX_SIZE) {
        throw new IllegalArgumentException("ファイルが大きすぎます");
    }
    String safeFilename = UUID.randomUUID() + getExtension(file.getOriginalFilename());
    Files.write(Paths.get("/uploads/", safeFilename), file.getBytes());
}
```

</details>

<details>
<summary>レート制限</summary>

**シナリオ:** PRはサードパーティのメールプロバイダーを使った新しいPOST /api/send-emailエンドポイントを追加します。レート制限は適用されていません。

攻撃者は1分間に何千回も呼び出すことができ、コストをかけ、プロバイダーのクォータを消費し、スパムのためにエンドポイントを使用する可能性があります。

確認: これは新しい公開エンドポイントまたは未認証エンドポイントか？YESなら、レート制限が必要です。

</details>

<details>
<summary>依存関係・設定のリスク</summary>

- **CVE（Common Vulnerabilities and Exposures）** — ソフトウェアライブラリの既知のセキュリティ欠陥の公開レジストリ。CVEのあるライブラリは、その依存関係を通じてアプリを攻撃者に悪用させる可能性がある

**シナリオ:** PRは新しいライブラリ`com.example:imageparser:2.1.0`を追加します。そのバージョンには既知のリモートコード実行CVEがあります。ビルドは通過し、機能は動作しますが、アプリは脆弱になっています。

確認: 新しい依存関係は特定のバージョンに固定され、既知のCVEがないか？（Java/Mavenプロジェクトの場合は`mvn dependency-check`でスキャン。）認証情報やAPIキーは、ソースファイルにハードコードされたりリポジトリにコミットされたりするのではなく、環境変数またはシークレットマネージャーに保存されているか？

</details>
</blockquote>

</details>

---

## パフォーマンスとシステムリスク

> スタイルについて話す前に、負荷がかかった状態で本番環境を壊す可能性があるか確認してください。

- 意図せずO(n²)のロジックはないか？
- N+1クエリのリスクはないか？
- リクエストスレッド内での重い計算はないか？
- 大量のデータロードはないか？
- 高スループットエンドポイントでのブロッキング呼び出しはないか？

**パフォーマンスが間違っていれば、システムは苦しみます。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>意図せずO(n²)のロジックはないか？</summary>

**要件:** リスト内の重複メールを見つける。

❌ 悪い例 — ネストされたループ、O(n²):

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

✅ 良い例 — HashSetルックアップ、O(n):

```java
public List<String> findDuplicates(List<String> emails) {
    Set<String> seen = new HashSet<>();
    List<String> duplicates = new ArrayList<>();
    for (String email : emails) {
        if (!seen.add(email)) { // add()はすでに存在する場合falseを返す
            duplicates.add(email);
        }
    }
    return duplicates;
}
```

</details>

<details>
<summary>N+1クエリのリスクはないか？</summary>

**要件:** すべての注文を顧客名と一緒にロードする。

❌ 悪い例 — 注文に1クエリ + 顧客にNクエリ:

```java
List<Order> orders = orderRepository.findAll(); // 1クエリ
for (Order order : orders) {
    String name = customerRepository.findById(order.getCustomerId()).getName(); // Nクエリ
    order.setCustomerName(name);
}
```

✅ 良い例 — 単一のJOINクエリ:

```java
List<OrderWithCustomer> orders = orderRepository.findAllWithCustomer(); // JOINを使った1クエリ
```

</details>

<details>
<summary>リクエストスレッド内での重い計算はないか？</summary>

**要件:** オンデマンドでPDFレポートを生成する。

❌ 悪い例 — 全時間リクエストスレッドをブロックする:

```java
@PostMapping("/reports/generate")
public ResponseEntity<byte[]> generateReport(@RequestBody ReportRequest request) {
    byte[] report = reportService.generatePdfReport(request); // 10秒以上 — スレッドがブロックされる
    return ResponseEntity.ok(report);
}
```

✅ 良い例 — 非同期ジョブにオフロードし、即座に返す:

```java
@PostMapping("/reports/generate")
public ResponseEntity<String> generateReport(@RequestBody ReportRequest request) {
    String jobId = reportService.enqueueReportGeneration(request); // バックグラウンド処理のためにキューに入れる — 即座に返す
    return ResponseEntity.accepted().body(jobId); // 202 Accepted: 呼び出し元はjobIdで準備完了を確認
}
```

</details>

<details>
<summary>大量のデータロードはないか？</summary>

**要件:** ユーザーの取引履歴を返す。

❌ 悪い例 — テーブル全体をメモリにロードする:

```java
public List<Transaction> getHistory(Long userId) {
    return transactionRepository.findAllByUserId(userId); // 数百万行になる可能性がある
}
```

✅ 良い例 — ページネーションを使用する:

```java
public Page<Transaction> getHistory(Long userId, Pageable pageable) {
    return transactionRepository.findAllByUserId(userId, pageable);
}
```

</details>

<details>
<summary>高スループットエンドポイントでのブロッキング呼び出しはないか？</summary>

**要件:** ユーザーのパーソナライズされたフィードを返す。

❌ 悪い例 — 同期呼び出しがスレッドプールスレッドをブロックする:

```java
@GetMapping("/feed")
public List<Post> getFeed(Long userId) {
    return externalFeedService.fetch(userId); // 外部サービスが応答するまでスレッドをブロック
}
```

✅ 良い例 — ノンブロッキング非同期レスポンス:

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

## スケーラビリティと並行性

> システムレベルで考えてください。

- ステートレスな設計か？
- 共有可変状態はないか？
- スレッドセーフか？
- 水平スケーリングに対応しているか？
- 非同期が必要か？
- レースコンディションのリスクはないか？
- リトライに対して安全か（冪等性）？

**バックエンド / 分散システムでは特に重要です。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>ステートレスな設計か？</summary>

**シナリオ:** PRはSpringの`@Service`ビーンの`HashMap`フィールドにアクティブなユーザーセッションを保存します。

単一インスタンスでは機能しますが、水平スケールすると壊れます — インスタンスAはインスタンスBが知らないセッションを保持しています。リクエストを処理するノードによって、ユーザーはランダムにログアウトされます。

インメモリキャッシュのステートフルなものは、マルチインスタンスの安全性のために外部化（Redis、DB）する必要があります。

</details>

<details>
<summary>共有可変状態はないか？ / スレッドセーフか？</summary>

**1. 更新の消失** — 2つのスレッドが同じ変数を同時に読み書きし、一方の更新が上書きされる:

❌ 悪い例 — `count++`は3ステップ（読み取り → 加算 → 書き込み）で、途中で割り込まれる可能性がある:

```java
@Service
public class RequestCounter {
    private int count = 0;

    public void increment() { count++; } // スレッドAとBの両方が5を読み、両方が6を書く — 1つのインクリメントが失われる
}
```

✅ 良い例 — `AtomicInteger`は読み取り-加算-書き込み全体を分割不可能にする:

```java
@Service
public class RequestCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); } // 単一のアトミック操作
}
```

**2. 可視性** — 一方のスレッドが値を書き込んでも、別のスレッドがCPUキャッシュのために更新を見られない:

❌ 悪い例 — `running = false`がCPUキャッシュに留まる可能性があり、別のスレッドのループが止まらない:

```java
private boolean running = true;

public void stop() { running = false; }         // スレッドAが書き込む
public void run()  { while (running) { ... } }  // スレッドBが古いキャッシュ値を読む可能性がある
```

✅ 良い例 — `volatile`はすべての読み取りをメインメモリに直接向ける:

```java
private volatile boolean running = true;

public void stop() { running = false; }
public void run()  { while (running) { ... } }
```

**3. スレッドセーフでないコレクション** — `HashMap`と`ArrayList`は同時の読み書きに対して安全でない:

❌ 悪い例 — 同時書き込みがエントリを破損するか`ConcurrentModificationException`をスローする:

```java
private final Map<String, User> cache = new HashMap<>();

public void addToCache(String key, User user) {
    cache.put(key, user); // 2つのスレッドが同時に書き込む → 状態が破損
}
```

✅ 良い例 — `ConcurrentHashMap`は同時アクセスを安全に処理する:

```java
private final Map<String, User> cache = new ConcurrentHashMap<>();

public void addToCache(String key, User user) {
    cache.put(key, user); // スレッドセーフ
}
```

データベースの状態を含むcheck-then-actレースコンディションについては、下記の**レースコンディションのリスクはないか？**を参照してください。

</details>

<details>
<summary>レースコンディションのリスクはないか？</summary>

**要件:** 資金が十分な場合のみ残高を引き落とす。

❌ 悪い例 — ロックなしのcheck-then-actにより二重支出が可能:

```java
public void deductBalance(Long userId, double amount) {
    User user = userRepository.findById(userId);
    if (user.getBalance() >= amount) { // 2つのスレッドが同時にこのチェックを通過できる
        user.setBalance(user.getBalance() - amount); // 両方が引き落とし — 口座が負になる
        userRepository.save(user);
    }
}
```

✅ 良い例 — 読み取り前に行をロックする:

```java
@Transactional
public void deductBalance(Long userId, double amount) {
    User user = userRepository.findByIdWithLock(userId); // 悲観的ロック (SELECT FOR UPDATE)
    if (user.getBalance() < amount) throw new InsufficientFundsException();
    user.setBalance(user.getBalance() - amount);
    userRepository.save(user);
}
```

</details>

<details>
<summary>水平スケーリングに対応しているか？</summary>

**シナリオ:** PRは共有リソースへの同時アクセスを防ぐためにJVMの`synchronized`ブロックを使用します。

単一ノードでは機能しますが、2つのノードでは各JVMが独自のロックを持ちます。両方のノードが同時にブロックに入ることができ、保護が完全に無効になります。

マルチインスタンスの協調には、JVMレベルの同期の代わりに分散ロック（Redis `SETNX`、DB アドバイザリロック）を使用してください。

</details>

<details>
<summary>非同期が必要か？</summary>

**シナリオ:** POST /checkoutエンドポイントは、レスポンスを返す前に確認メールを同期的に送信します。

メールサービスが遅い場合や一時的にダウンしている場合、チェックアウトがハングするか、完全に失敗します。ユーザーの支払いは処理されましたが、エラーが表示されます。

確認メールはfire-and-forgetです。ドメインイベントを公開するかメッセージキューにプッシュし、チェックアウト結果をすぐに返してください。

</details>

<details>
<summary>リトライに対して安全か（冪等性）？</summary>

AWS SQSなどのメッセージブローカーは**少なくとも1回の配信（at-least-once delivery）**を保証します。ハンドラーがクラッシュしたりタイムアウトしてメッセージが削除される前に失敗すると、SQSはメッセージを再配信します。ハンドラーは最初から再実行され、最初の試行でも完了した副作用を含めて再実行されます。

**シナリオ:** ハンドラーがDBに注文を保存し、その後支払いゲートウェイを呼び出します。支払い呼び出しがタイムアウト例外をスローします。SQSはメッセージを再配信します。リトライ時にDB挿入が再実行され、重複注文が作成されてから2回目の請求が試みられます。

❌ 悪い例 — 再実行に対するガードなし。リトライ時に重複レコードが発生:

```java
@SqsListener("order-events")
public void handleOrderEvent(OrderMessage message) {
    orderRepository.save(new Order(message.getOrderId(), message.getAmount())); // リトライ時に再実行 → 重複
    paymentService.charge(message.getOrderId(), message.getAmount());           // 二重請求の可能性
}
```

✅ 良い例 — 処理前にメッセージIDで重複チェック:

```java
@SqsListener("order-events")
public void handleOrderEvent(OrderMessage message) {
    if (orderRepository.existsByMessageId(message.getMessageId())) {
        return; // 処理済み — スキップしても安全
    }
    orderRepository.save(new Order(message.getOrderId(), message.getAmount(), message.getMessageId()));
    paymentService.charge(message.getOrderId(), message.getAmount());
}
```

または**upsert**（`messageId`をキーとした`INSERT ... ON CONFLICT DO NOTHING`）を使用して、DB自体がアトミックに重複を拒否するようにします。

ロジックをリトライ安全にするための一般的なパターン:

- **冪等性キーチェック** — 初回成功時にメッセージ/リクエストIDを保存し、既に処理済みの場合はスキップ
- **upsertの使用** — `INSERT ... ON CONFLICT DO NOTHING / DO UPDATE`で重複行を防ぐ
- **条件付き更新** — レコードが期待される状態の場合のみ更新（例: `WHERE status = 'PENDING'`）
- **外部APIガード** — 新しい呼び出しを行う前に外部呼び出しが既に成功したか確認（例: 新規作成前に注文IDで請求を検索）

問い: このハンドラーが同じ入力で2回呼ばれた場合、結果は変わるか？

</details>
</blockquote>

</details>

---

## アーキテクチャと設計

> 構造的な整合性を確認してください。

- 適切なレイヤー分離はあるか？
- コントローラーにビジネスロジックはないか？
- ドメインにインフラロジックはないか？
- 依存関係の方向は正しいか？
- 密結合は導入されていないか？
- 単一責任の原則は守られているか？
- 重複はないか？

**アーキテクチャは長期的な保守性を守ります。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>コントローラーにビジネスロジックはないか？</summary>

❌ 悪い例 — 割引ロジックと合計計算がコントローラーにある:

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
    double total = request.getItems().stream()
        .mapToDouble(i -> i.getPrice() * i.getQuantity()).sum();
    if (total > 10000) total = total * 0.9; // ビジネスルールがコントローラーにある
    return ResponseEntity.ok(orderRepository.save(new Order(request.getUserId(), total)));
}
```

✅ 良い例 — コントローラーは委譲し、サービスがロジックを持つ:

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
    return ResponseEntity.ok(orderService.createOrder(request));
}
```

</details>

<details>
<summary>ドメインにインフラロジックはないか？</summary>

❌ 悪い例 — ドメインエンティティが直接データベースと通信する:

```java
public class User {
    public void save() {
        DataSource ds = DataSourceConfig.getInstance(); // ドメインにインフラの関心事
        ds.execute("INSERT INTO users ...");
    }
}
```

✅ 良い例 — ドメインエンティティは純粋。挿入はリポジトリを通じてサービスレイヤーに委譲される:

```java
public class User {
    private Long id;
    private String email;
    // ドメイン動作のみ — DB、HTTP、I/Oなし
}

public interface UserRepository extends JpaRepository<User, Long> {}

@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User createUser(User user) {
        return userRepository.save(user); // 永続化はここで処理、User内ではない
    }
}
```

</details>

<details>
<summary>密結合は導入されていないか？</summary>

❌ 悪い例 — サービスが自分の依存関係をインスタンス化する:

```java
public class OrderService {
    private final MySQLOrderRepository repository = new MySQLOrderRepository(); // ハードコード
}
```

✅ 良い例 — 抽象化に依存し、外部から注入される:

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
<summary>単一責任の原則は守られているか？</summary>

❌ 悪い例 — 1つのサービスがユーザー作成、メール送信、ログ記録、レポートを処理する:

```java
public class UserService {
    public User createUser(UserRequest request) { ... }
    public void sendWelcomeEmail(User user) { ... }
    public void logUserCreation(User user) { ... }
    public byte[] generateUserReport(Long userId) { ... }
}
```

✅ 良い例 — 各クラスは変更する理由が1つだけ:

```java
public class UserService    { public User createUser(UserRequest request) { ... } }
public class EmailService   { public void sendWelcomeEmail(User user) { ... } }
public class ReportService  { public byte[] generateUserReport(Long userId) { ... } }
```

</details>

<details>
<summary>依存関係の方向は正しいか？</summary>

**シナリオ:** ドメインの`Order`クラスが保存後に直接`EmailNotificationService`（インフラの関心事）をインポートして呼び出します。

これはドメインレイヤーがインフラレイヤーに依存することを意味し、間違った方向です。ドメインはドメインイベント（`OrderPlaced`）を発行すべきで、インフラはそれを受け取って反応します。依存関係は常にドメインに向かって内側を指します。

</details>

<details>
<summary>重複はないか？</summary>

**シナリオ:** PRは`InvoiceService`に`calculateTax()`メソッドを追加しますが、`OrderService`に既にほぼ同一のものがあります。

税ルールが変更されると、両方のコピーを更新する必要がありますが、必ず一方が見逃され、動作が不一致になります。共有ロジックをドメインレイヤーが所有する`TaxCalculator`コンポーネントに抽出してください。

</details>
</blockquote>

</details>

---

## テスタビリティ（設計品質の指標）

> コードがテストしにくい場合、通常は設計が不十分です。

- 依存関係は注入可能か？
- ビジネスロジックは分離されているか？
- 純粋なロジックは抽出可能か？
- 隠れた副作用はないか？
- DBなしでユニットテスト可能か？
- モッキングは簡単か？

**テスタビリティは構造的品質を明らかにします。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>依存関係は注入可能か？</summary>

❌ 悪い例 — ハードコードされた依存関係、モック不可能:

```java
public class OrderService {
    private final OrderRepository repository = new JpaOrderRepository(); // テストで置き換えられない

    public Order createOrder(OrderRequest request) {
        return repository.save(new Order(request));
    }
}
```

✅ 良い例 — 注入された依存関係、簡単にモック可能:

```java
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository; // テストでモックを渡す
    }
}
```

</details>

<details>
<summary>ビジネスロジックは分離されているか？ / 純粋なロジックは抽出可能か？</summary>

**シナリオ:** コントローラーメソッドがデータの取得、価格ルールの適用、レスポンスのフォーマット、監査イベントの送信をすべてインラインで行います。フルHTTPスタックを起動せずに価格ロジックをテストする方法がありません。

ビジネスロジックがフレームワークコールバック（コントローラー、リスナー、バッチプロセッサー）内に埋め込まれている場合、独立してユニットテストできません。フレームワークの依存関係なしに入力を受け取り出力を返す、プレーンなサービスまたはドメインメソッドに抽出してください。

</details>

<details>
<summary>隠れた副作用はないか？</summary>

❌ 悪い例 — 計算メソッドが密かにキャッシュと監査ログに書き込む:

```java
public double calculateTotal(List<Item> items) {
    double total = items.stream().mapToDouble(Item::getPrice).sum();
    auditLog.record("合計: " + total); // 隠れた副作用
    cache.put("lastTotal", total);     // 別の隠れた副作用
    return total;
}
```

✅ 良い例 — 純粋な関数、予測可能で独立してテスト可能:

```java
public double calculateTotal(List<Item> items) {
    return items.stream().mapToDouble(Item::getPrice).sum();
}
```

</details>

<details>
<summary>DBなしでユニットテスト可能か？</summary>

❌ 悪い例 — リポジトリ呼び出しの背後にビジネスロジックが埋め込まれ、テストに実際のDBが必要:

```java
public class DiscountService {
    @Autowired
    private UserRepository userRepository;

    public double getDiscount(Long userId) {
        User user = userRepository.findById(userId); // すべてのユニットテストでDBを強制する
        return user.isPremium() ? 0.20 : 0.10;
    }
}
```

✅ 良い例 — ドメインオブジェクトを受け取る。ユニットテストにDBは不要:

```java
public class DiscountService {
    public double getDiscount(User user) {
        return user.isPremium() ? 0.20 : 0.10; // new User(isPremium=true)でテスト
    }
}
```

</details>

<details>
<summary>モッキングは簡単か？</summary>

**シナリオ:** サービスがインラインで構築された`new EmailClient("smtp.internal", 587, true, "user", "pass")`に依存しています。サービスをテストするには実際のSMTPサーバーが必要です。

依存関係をモックするのにテスト自体より多くのセットアップが必要な場合、依存関係が過度に密結合しています。インターフェース（`EmailSender`）に依存し、注入し、テストで1行でインターフェースをモックしてください。

</details>

<details>
<summary>⊕ ボーナス — テスタビリティのための推奨コード構成（Spring Webアプリ）</summary>

> このセクションは上記チェックリストの補足です。各レイヤーを独立してテスト可能にするSpring Webアプリの具体的な参照アーキテクチャを提供します。

コードを4つのレイヤーに構造化します。各レイヤーは1つの関心事を持ち、それぞれ独立してテスト可能です。

```
src/
└── main/java/com/yourapp/
    ├── controller/                    ← HTTPの入出力のみ
    │   └── OrderController.java
    ├── application/                   ← ユースケースのオーケストレーション
    │   └── PlaceOrderUseCase.java
    ├── domain/                        ← 純粋なビジネスロジック、フレームワークなし
    │   ├── Order.java
    │   ├── OrderRepository.java       ← インターフェース（ドメイン所有）
    │   └── OrderPlacedEvent.java
    └── infrastructure/                ← DB、メッセージング、外部API
        ├── JpaOrderRepository.java    ← OrderRepositoryを実装
        └── SnsOrderEventPublisher.java
```

**各レイヤーの役割とテスト方法:**

| レイヤー | 責任 | 依存先 | テストタイプ | 必要なセットアップ |
|---|---|---|---|---|
| `domain/` | ビジネスルール、純粋なロジック | なし | ユニットテスト | `new Order(...)` — モックなし、Springなし |
| `application/` | オーケストレーション: ロード → 実行 → 保存 → 公開 | `domain/`インターフェースのみ | モックを使ったユニットテスト | `OrderRepository`をモック、パブリッシャーをモック |
| `controller/` | HTTPリクエストの解析、レスポンスの返却 | `application/` | `@WebMvcTest`スライス | `PlaceOrderUseCase`のみモック |
| `infrastructure/` | DBクエリ、SNS、SES、外部API | `domain/`（そのインターフェースを実装） | 統合テスト | TestcontainersによるリアルDB |

> **依存関係ルール:** `application/`は`infrastructure/`クラスを直接インポートしません — `domain/`で定義されたリポジトリ*インターフェース*に依存し、SpringがランタイムにインフラImplementationを注入します。これにより、アプリケーションレイヤーはリアルDBなしでテスト可能になります。

**基本ルール:** レイヤーが内側であるほど、テストコストが低くなります。

```java
// domain/ — 依存関係なし、プレーンオブジェクトでテスト
@Test
void order_shouldApplyBulkDiscount_whenTotalExceeds10000() {
    Order order = new Order(List.of(new Item("A", 12000.0)));
    assertThat(order.getFinalTotal()).isEqualTo(10800.0); // Springなし、モックなし
}

// application/ — 境界をモック、オーケストレーションをテスト
@Test
void placeOrder_shouldSaveAndPublishEvent() {
    when(orderRepository.save(any())).thenReturn(savedOrder);
    useCase.execute(request);
    verify(eventPublisher).publish(any(OrderPlacedEvent.class));
}

// controller/ — ユースケースをモック、HTTPマッピングのみをテスト
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

## テストカバレッジとテスト品質

> セーフティネットを確認してください。

- メインフローはカバーされているか？
- エッジケースはテストされているか？
- 失敗ケースはテストされているか？
- テストは（実装の詳細ではなく）動作を検証しているか？
- 統合テストが必要か？

**テストは将来のリファクタリングを守ります。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>メインフローはカバーされているか？</summary>

**シナリオ:** チェックアウト機能が価格ロジックのみのテストでマージされます。完全なフロー — カートに追加 → クーポン適用 → 支払い → 確認受信 — にはエンドツーエンドまたは統合テストがありません。

主要なユーザージャーニーにテストカバレッジがない場合、そのパスのどこかで将来の変更が黙って壊す可能性があります。

</details>

<details>
<summary>エッジケースはテストされているか？</summary>

**シナリオ:** `calculateDiscount()`メソッドが$50の典型的な注文でテストされています。しかし、$0の注文、負の価格、または空のアイテムリストのテストがありません。

問い: 境界値は何か？ゼロ、null、空、最大、最小では何が起こるか？

</details>

<details>
<summary>失敗ケースはテストされているか？</summary>

**シナリオ:** PRはハッピーパス（成功した支払い）のテストを追加します。しかし、支払いゲートウェイがダウンしている場合、カードが拒否された場合、またはリクエストがタイムアウトした場合に何が起こるかのテストがありません。

失敗パスはバグが最も多くのダメージを与える場所です。成功テストごとに問いかけてください: 対応する失敗テストは何か？

</details>

<details>
<summary>テストは（実装の詳細ではなく）動作を検証しているか？</summary>

❌ 悪い例 — テストがユーザーエクスペリエンスではなく、メソッドが呼ばれたことを確認する:

```java
@Test
void createOrder_shouldCallRepositorySave() {
    orderService.createOrder(request);
    verify(orderRepository, times(1)).save(any()); // 正しいリファクタリングでも壊れる
}
```

✅ 良い例 — テストが観察可能な結果を確認する:

```java
@Test
void createOrder_shouldReturnOrderWithCorrectTotal() {
    Order order = orderService.createOrder(request);
    assertThat(order.getTotal()).isEqualTo(expectedTotal);
}
```

</details>

<details>
<summary>統合テストが必要か？</summary>

**シナリオ:** 各サービスメソッドがモックで独立してユニットテストされています。しかし、実際のDBスキーマを使用すると、クエリが失敗します。なぜなら、モックが反映していないマイグレーションでカラムがリネームされたからです。

ユニットテストだけでは統合の失敗を捕捉できません。問い: 実際の統合テストが必要なレイヤー境界（DB、HTTP、メッセージキュー）はあるか？

</details>
</blockquote>

</details>

---

## 可読性と保守性

> 明瞭さを確認してください。

- コードは6ヶ月後も理解可能か？
- 命名は意味があり、ドメインと一貫しているか？
- メソッドは長すぎないか？
- ネストの深さは適切か？
- デッドコードはないか？
- 複雑なロジックは説明されているか？

**クリーンなコードは将来の認知負荷を減らします。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>コードは6ヶ月後も理解可能か？</summary>

これはこのセクションのすべてのポイントの総合目標です。「6ヶ月」のヒューリスティックは、元の作者でさえ数ヶ月後には自分が下した決断の背後にある文脈を忘れるという現実を捉えています。コードを理解するために、当時誰かの頭の中にしか存在しなかった知識が必要な場合、それは保守の負担になります。

問い: 開発者（未来の自分を含む）が誰かに何をするのか、なぜそうするのかを尋ねずにこれを読めるか？

数字は例示的なものであり、正確ではありません。「明日の新入社員」と言うチームもあれば、「来年の他人」と言うチームもあります。要点は同じです: **コードは書かれるよりずっと多く読まれます**。そのため、明瞭さは執筆の瞬間を超えて生き残る必要があります。このセクションの他のすべてのポイント — 命名、メソッドの長さ、ネストの深さ、デッドコード、コメント — は、この質問に「はい」と答える具体的な方法です。

</details>

<details>
<summary>命名は意味があり、ドメインと一貫しているか？</summary>

❌ 悪い例 — ドメインの意味のない省略名:

```java
public double calc(User u, List<Item> l, boolean f) {
    double t = 0;
    for (Item i : l) t += i.getP() * i.getQ();
    return f ? t * 0.9 : t;
}
```

✅ 良い例 — 名前が意図を伝える:

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
<summary>メソッドは長すぎないか？</summary>

**シナリオ:** `processCheckout()`メソッドが200行あります。入力バリデーション、クーポン適用、税計算、カード請求、メール送信、在庫更新をすべてインラインで行います。

メソッドは1つのことをすべきです。スクロールして読む必要がある場合、またはセクションをマークするコメントが必要な場合（「// ステップ1」、「// ステップ2」）、分割すべきです。抽出された各メソッドは独立して読みやすく、テスト可能になります。

</details>

<details>
<summary>ネストの深さは適切か？</summary>

❌ 悪い例 — 4レベルのネストがハッピーパスを追いにくくする:

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

✅ 良い例 — ガード節が構造をフラットにする:

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
<summary>デッドコードはないか？</summary>

**シナリオ:** PRにはどこからも呼ばれていない`legacyCalculateShipping()`メソッドが含まれています。3ヶ月前に置き換えられましたが、削除されていませんでした。

デッドコードは混乱を生みます — 将来の読者はそれが意図的かどうか、必要かどうか、削除すると何かが壊れるかどうかを疑問に思います。使用されていない場合は削除してください。バージョン管理が履歴を保存します。

</details>

<details>
<summary>複雑なロジックは説明されているか？</summary>

❌ 悪い例 — マジックナンバーと数式が説明されていない:

```java
double adjustedScore = rawScore * 0.85 + (completionRate * 15);
```

✅ 良い例 — コメントが何をするかだけでなくなぜそうするかを説明する:

```java
// スコアの数式: 生の精度に85%のウェイト + タスク完了率に15%のウェイト
// プロダクト仕様v2.3で定義 — 仕様を更新せずに変更しないこと
double adjustedScore = rawScore * 0.85 + (completionRate * 15);
```

</details>
</blockquote>

</details>

---

## 型チェック

> 型システムは、ランタイムコストゼロで、バグ全体のクラスに対するあなたの最初の防衛線です。

- プリミティブへの執着がある？生の`String` / `long` / `int`の代わりにドメイン型を使用しているか？
- 特定の型が存在するのに`Object`、生の`Map`、または`Any`を使用しているか？
- nullは安全に処理されているか？（`Optional`、null可能性アノテーション、またはnullセーフラッパー）
- 未チェックのキャストが存在し、説明がないか？
- 閉じた値のセットがマジック文字列や整数定数ではなく`enum`で定義されているか？
- **TypeScript?** `unknown`が適切な箇所で`any`を使用しているか？型ガードの代わりに安全でない`as`キャストを使用しているか？共用体型に判別子が欠けているか？値の混在が起こりうる箇所でブランデッド型が欠けているか？

**型の優れたAPIは、コードが実行される前にバグ全体のカテゴリを表現不能にします。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>プリミティブへの執着がある？</summary>

**要件:** 注文IDと顧客IDを使用して支払いを処理する。

❌ 悪い例 — 生のプリミティブには型情報がなく、引数の順序が間違っていてもコンパイルが通る:

```java
public void processPayment(long orderId, long customerId, double amount) { ... }

// 呼び出し元が引数を間違った順序で渡す — コンパイルエラーなし:
processPayment(customerId, orderId, amount);
```

✅ 良い例 — ドメイン型により引数の順序間違いがコンパイルエラーになる:

```java
public record OrderId(long value) {}
public record CustomerId(long value) {}
public record Money(double amount, Currency currency) {}

public void processPayment(OrderId orderId, CustomerId customerId, Money amount) { ... }

// これはコンパイルエラーになる — 型が一致しない:
processPayment(customerId, orderId, amount); // ❌ CustomerIdはOrderIdではない
```

ドメイン型はビジネスルールも持てます: `Money`型は負の値を強制できますが、生の`double`にはできません。

</details>

<details>
<summary>特定の型が存在するのに`Object`、生の`Map`、または`Any`を使用しているか？</summary>

❌ 悪い例 — 呼び出し元はどのキーや値の型を期待すべきかわからない:

```java
public Map<String, Object> getUserProfile(Long userId) {
    Map<String, Object> profile = new HashMap<>();
    profile.put("name", "Jane");
    profile.put("age", 30);
    profile.put("premium", true);
    return profile;
}
```

✅ 良い例 — 型付き戻り値がコントラクトを明示的に伝える:

```java
public record UserProfile(String name, int age, boolean isPremium) {}

public UserProfile getUserProfile(Long userId) { ... }
```

呼び出し元はコンパイル時にフィールド名と型を取得できます。フィールド名を変更するとコンパイラーを通じて伝播し、ランタイムで`null`や`ClassCastException`が発生する代わりにコンパイルエラーになります。

</details>

<details>
<summary>nullは安全に処理されているか？</summary>

**要件:** ユーザーのオプションの表示名を返す。

❌ 悪い例 — nullable な戻り値が黙って漏れ出す。呼び出し元がnullチェックを忘れる:

```java
public String getDisplayName(Long userId) {
    return userRepository.findById(userId).getDisplayName(); // nullを返す可能性がある
}
```

✅ 良い例 — `Optional`が不在のコントラクトを明示的に示す:

```java
public Optional<String> getDisplayName(Long userId) {
    return Optional.ofNullable(
        userRepository.findById(userId).getDisplayName()
    );
}
```

`Optional`が使用できない場合（例: パフォーマンスが重要なコードやコレクション要素の戻り型）、静的解析ツールと呼び出し元が何を期待するかわかるように`@Nullable` / `@NonNull`アノテーションを使用してください。

</details>

<details>
<summary>未チェックのキャストが存在し、説明がないか？</summary>

❌ 悪い例 — 未チェックのキャストが黙って抑制される。前提が崩れるとランタイムで失敗する:

```java
@SuppressWarnings("unchecked")
public List<Order> getOrders(Object raw) {
    return (List<Order>) raw; // rawが別のものを含む場合ClassCastException
}
```

✅ 良い例 — キャスト前に検証するか、キャストを避けるように再設計する:

```java
public List<Order> getOrders(List<?> raw) {
    return raw.stream()
        .filter(item -> item instanceof Order)
        .map(item -> (Order) item)
        .collect(Collectors.toList());
}
```

抑制が本当に避けられない場合は、キャストがなぜ安全か、どの不変条件がそれを保証するかを正確に説明するコメントを追加してください。

</details>

<details>
<summary>閉じた値のセットは`enum`で定義されているか？</summary>

**要件:** 注文ステータスとして`PENDING`、`PROCESSING`、または`COMPLETED`を表現する。

❌ 悪い例 — マジック文字列。タイポがコンパイルされ、網羅性が強制されない:

```java
public void updateStatus(String status) {
    if (status.equals("PENDNG")) { ... } // タイポ — コンパイルが通り、黙って何もしない
}
```

✅ 良い例 — `enum`が無効な値を表現不能にし、switch文の網羅性チェックが可能になる:

```java
public enum OrderStatus { PENDING, PROCESSING, COMPLETED }

public void updateStatus(OrderStatus status) {
    switch (status) {
        case PENDING     -> ...;
        case PROCESSING  -> ...;
        case COMPLETED   -> ...;
        // 新しいenum値が追加されてハンドルされていない場合、コンパイラが警告する
    }
}
```

</details>

<details>
<summary>TypeScript: 高度な型システムのチェック</summary>

TypeScriptの型システムはJavaよりもはるかに表現力が高いですが、その表現力は誤用される可能性があります。これらのチェックは上記の一般的なものに加えて適用されます。

### `any` vs `unknown`

❌ 悪い例 — `any`はすべての型チェックを無効化する。エラーはランタイムまで延期される:

```typescript
function parseConfig(raw: any) {
    return raw.timeout * 1000; // rawがnullや文字列でもエラーなし
}
```

✅ 良い例 — `unknown`は呼び出し元が使用前に型を絞り込むことを強制する:

```typescript
function parseConfig(raw: unknown) {
    if (typeof raw !== 'object' || raw === null || !('timeout' in raw)) {
        throw new Error('無効な設定');
    }
    return (raw as { timeout: number }).timeout * 1000;
}
```

### 判別共用体と網羅性

❌ 悪い例 — 判別子がない。各ブランチは安全でないキャストを必要とする:

```typescript
type Shape = { width: number; height: number } | { radius: number };

function area(shape: Shape) {
    return (shape as any).radius
        ? Math.PI * (shape as any).radius ** 2
        : (shape as any).width * (shape as any).height;
}
```

✅ 良い例 — 判別子フィールドにより各ブランチが型安全になる。`never`がコンパイル時に未処理のバリアントを捕捉する:

```typescript
type Shape =
    | { kind: 'rect';   width: number; height: number }
    | { kind: 'circle'; radius: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case 'rect':   return shape.width * shape.height;
        case 'circle': return Math.PI * shape.radius ** 2;
        default: {
            const _exhaustive: never = shape; // 新しいバリアントが追加されてハンドルされない場合コンパイルエラー
            throw new Error(`未処理のShape: ${_exhaustive}`);
        }
    }
}
```

### `as`キャスト vs 型ガード

❌ 悪い例 — `as`は型システムをバイパスする。前提が崩れるとランタイムで間違いになる:

```typescript
function getUser(data: unknown): User {
    return data as User; // 構造チェックなし
}
```

✅ 良い例 — 型ガードが最初に検証する。内部の絞り込まれた型は安全:

```typescript
function isUser(data: unknown): data is User {
    return typeof data === 'object' && data !== null && 'id' in data && 'email' in data;
}

function getUser(data: unknown): User {
    if (!isUser(data)) throw new Error('無効なユーザーペイロード');
    return data; // Userに絞り込まれる — キャスト不要
}
```

### ブランデッド型 / 公称型

❌ 悪い例 — 構造的エイリアスは互換性がある。間違ったIDが黙って渡される:

```typescript
type UserId    = string;
type ProductId = string;

function getProduct(id: ProductId) { ... }

const userId: UserId = 'usr_123';
getProduct(userId); // エラーなし — UserIdとProductIdは構造的に同一
```

✅ 良い例 — ブランデッド型がコンパイル時にドメイン間の値の混在を防ぐ:

```typescript
type UserId    = string & { readonly __brand: 'UserId' };
type ProductId = string & { readonly __brand: 'ProductId' };

function getProduct(id: ProductId) { ... }

const userId = 'usr_123' as UserId;
getProduct(userId); // ❌ コンパイルエラー — UserIdはProductIdに代入できない
```

### ジェネリック制約

❌ 悪い例 — 制約のない`T`。関数はプロパティにアクセスするためにキャストを強いられる:

```typescript
function getLabel<T>(item: T): string {
    return (item as any).label; // 強制キャスト — コンパイラのサポートなし
}
```

✅ 良い例 — `T extends { label: string }`でコンパイラに本体を型チェックするのに十分な情報を与える:

```typescript
function getLabel<T extends { label: string }>(item: T): string {
    return item.label; // 安全 — Tには必ず.labelがある
}
```

</details>
</blockquote>

</details>

---

## インフラとコスト影響（上級レビュー）

> シニアレベルのチェック。

- DBの負荷は増加するか？
- インデックスが必要か？
- キャッシュの無効化は正しいか？
- メッセージキューへの影響は？
- クラウドコストは増加するか？
- モニタリング・ログは十分か？

**これはプラットフォーム思考の領域です。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>DBの負荷は増加するか？</summary>

**シナリオ:** PRは5分ごとに実行される新しいバックグラウンドジョブを追加し、ページネーションなしで3つのテーブルを結合して過去24時間に変更されたすべての`orders`テーブルレコードをクエリします。

このクエリは10,000行のステージングでは正常に動作します。5,000万行の本番では、行をロックし、CPUをスパイクさせ、他のすべてのクエリのレイテンシを悪化させます。問い: 新しいクエリパターンを追加しているか？データ量は？バッチ処理が必要か？

</details>

<details>
<summary>インデックスが必要か？</summary>

**要件:** 管理ダッシュボードのためにステータスで注文をフィルタリングする。

❌ 悪い例 — リクエストごとにフルテーブルスキャン:

```java
public List<Order> findPendingOrders() {
    return orderRepository.findByStatus("PENDING"); // 'status'にインデックスなし — スケール時にフルスキャン
}
```

✅ 良い例 — マイグレーションでインデックスをバックアップする:

```sql
-- DBマイグレーションに追加
CREATE INDEX idx_orders_status ON orders(status);
```

`WHERE`、`JOIN`、または`ORDER BY`句で使用される新しいカラムはすべてインデックスのために評価すべきです。

</details>

<details>
<summary>キャッシュの無効化は正しいか？</summary>

**シナリオ:** PRはキャッシュキーとして`userId`を使用してユーザーのプロフィールのキャッシュを追加します。しかし、ユーザーがメールを更新すると、キャッシュは無効化されません。TTLが切れるまでユーザーは古いデータを見ます。

確認: データが変更された場合、対応するキャッシュエントリは削除または更新されるか？キャッシュキーはユーザー間の衝突を避けるのに十分に具体的か？

</details>

<details>
<summary>メッセージキューへの影響は？</summary>

**シナリオ:** PRはフロントエンドからキャプチャされたmouseoverイベントを含むすべてのユーザーアクションでキューにイベントを公開します。100,000人の同時ユーザーでは、1分間に数百万の低価値メッセージでキューが溢れ、重要な注文イベントを押しのけます。

問い: 本番スケールでの公開レートは？コンシューマーグループは正しく分離されているか？失敗したメッセージのデッドレターキューはあるか？

</details>

<details>
<summary>クラウドコストは増加するか？</summary>

**シナリオ:** PRはすべてのログインでユーザーオブジェクトの完全なJSONスナップショットをDynamoDBレコードに保存します。オブジェクトは50KBで、ユーザーは頻繁にログインします。スケールすると、実際に必要なフィールドのみを保存する場合と比較して読み取り/書き込みコストが大幅に増加します。

問い: この変更はストレージ、読み取り/書き込みユニット、エグレス、またはコンピュートをスケール時に複利的に増加させるか？

</details>

<details>
<summary>モニタリング・ログは十分か？</summary>

❌ 悪い例 — 失敗が黙って消え、可観測性がない:

```java
public void processPayment(PaymentRequest request) {
    try {
        paymentGateway.charge(request);
    } catch (Exception e) {
        // 飲み込まれる — 誰も失敗を知らない
    }
}
```

✅ 良い例 — 診断に十分なコンテキストでログを記録する:

```java
public void processPayment(PaymentRequest request) {
    try {
        paymentGateway.charge(request);
        log.info("支払い処理完了: orderId={}, amount={}", request.getOrderId(), request.getAmount());
    } catch (Exception e) {
        log.error("支払い失敗: orderId={}, reason={}", request.getOrderId(), e.getMessage(), e);
        throw e;
    }
}
```

</details>
</blockquote>

</details>

---

## スタイルとフォーマット（最後に）

> 他のすべての後に:

- フォーマット
- Lint警告
- 軽微な命名の改善

これらは理想的には自動化されるべきです（lint、フォーマッター、CI）。
人間は思考レベルの問題に集中すべきです。

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>フォーマット</summary>

**シナリオ:** PRは3行のロジックに触れますが、作者のIDEが保存時にファイル全体を再フォーマットしたため、200行のdiffが表示されます。

フォーマットのノイズは実際の変更を埋め、レビューを困難にします。CI でフォーマッターを強制（例: Spotless、google-java-format）してフォーマットが人間のレビューの関心事にならないようにします。PRに無関係なフォーマット変更が含まれている場合、著者に分離するよう依頼してください。

</details>

<details>
<summary>Lint警告</summary>

**シナリオ:** PRは説明なしに新しい`@SuppressWarnings("unchecked")`アノテーションを2つ導入します。ビルドは通過しますが、抑制された警告が潜在的に安全でないキャストを隠しています。

Lint警告は抑制ではなく修正すべきです。抑制が本当に必要な場合は、なぜ安全かを説明するコメントを付ける必要があります。対処されないLint警告は技術的負債として蓄積されます。

</details>

<details>
<summary>軽微な命名の改善</summary>

**シナリオ:** 支払いレスポンスを処理するメソッドで変数が`data`と命名されています。`paymentResponse`にリネームするとコストゼロで即座に明確になります。

軽微な命名の改善はコメントする価値がありますが、PRをブロックすべきではありません。提案はするが、要求はしないでください。正確性、セキュリティ、アーキテクチャの問題のためにブロッキングフィードバックを保留してください。

</details>
</blockquote>

</details>

---

## 実装意図と代替案の分析

> *どのように*ではなく*なぜ*を問う。

- なぜこの実装方法を選んだのか？
- なぜここに配置されているのか（このレイヤー / クラス / モジュール）？
- このアプローチは本当に目標を達成しているのか？
- コメントする前にそれらの問いを自分で調査する
- 具体的なメリットがある場合のみ代替案を提案する

**意図を問うことで、隠れた前提と見落とされた解決策が浮かび上がります。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>なぜこのアプローチを選んだのか？</summary>

解決策を受け入れる前に、選ばれた手法が問題に対して適切なツールであるかどうかを問います。これはスタイルの問題ではなく、根本的により良いアプローチが存在するかどうかの問題です。

形成すべき問い:

- この手法は要件を満たす最もシンプルなものか？
- 標準ライブラリやパターンで排除できる偶発的な複雑さ（カスタムロジック、余分な抽象化）を導入していないか？
- 問題を正しい場所で解決しているのか、それとも深い問題を回避しているのか？

**シナリオ:** PRは`notifications`テーブルを10秒ごとにポーリングし、保留中のメールを送信するスケジュールジョブを追加します。

*なぜという問い:* なぜスケジュールでポーリングするのか？通知行を作成するイベントに反応できないのか？

*自己調査:* ポーリングアプローチは機能しますが、常にDBの負荷を増加させ、インターバルに比例したレイテンシを生みます。通知行はシステムにすでに存在するドメインイベントの結果として作成されています。

*代替案:* 行が挿入されたときに`NotificationCreated`イベントを公開し、非同期リスナーでそれを消費します。メールは即座に送信され、DBポーリングが不要になり、スケジューラーとテーブルの結合が消えます。

❌ 元の実装 — ポーリングアプローチ:

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

✅ 代替案 — イベント駆動:

```java
@TransactionalEventListener
public void onNotificationCreated(NotificationCreatedEvent event) {
    emailService.send(event.getNotification());
}
```

代替案はポーリングループを排除し、DBの負荷を下げ、トリガーアクションから数ミリ秒以内にメールを送信します。

</details>

<details>
<summary>なぜここに配置されているのか？</summary>

配置場所は意味を持ちます。間違ったレイヤー、クラス、またはモジュールにあるメソッドは、将来の読者の所有権の理解を誤らせ、テストや再利用を困難にします。

形成すべき問い:

- なぜこのクラスがこの責任を持つのか？
- なぜこのロジックがこのレイヤー（コントローラー / サービス / ドメイン / インフラ）にあるのか？
- なぜこのユーティリティメソッドが共有の場所に置かれずにここで重複しているのか？

**シナリオ:** PRは`calculateAge(LocalDate birthDate)`メソッドを直接`UserController`の中に追加します。

*なぜという問い:* 純粋なドメイン計算である年齢計算が、なぜHTTPレイヤーにあるのか？他に誰がこれを必要とするかもしれないか？

*自己調査:* 年齢計算にはHTTPへの依存がありません。患者、従業員、または複数の機能にわたる任意の人エンティティに適用できる純粋なビジネスロジックです。コントローラーに置くと、HTTPを経由せずにはアクセスできず、他の呼び出し元からも見えなくなります。

*代替案:* `User`ドメインオブジェクトまたは`DateUtils`共有ユーティリティに移動します。

❌ 元の実装 — ロジックがコントローラーに埋め込まれている:

```java
@RestController
public class UserController {
    @GetMapping("/users/{id}/age")
    public int getUserAge(@PathVariable Long id) {
        User user = userService.findById(id);
        return Period.between(user.getBirthDate(), LocalDate.now()).getYears(); // ドメインロジックがコントローラーに
    }
}
```

✅ 代替案 — ロジックがドメインオブジェクトにある:

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
        return userService.findById(id).getAge(); // コントローラーは委譲する
    }
}
```

これで、HTTPレイヤーなしでテスト可能になり、コードベース全体で再利用できます。

</details>

<details>
<summary>このアプローチは本当に目標を達成しているのか？</summary>

実装が構文的に正しく、アーキテクチャ的に健全であっても、機能が実際に必要とするものを達成できていない場合があります。この問いは意図と効果のギャップを埋めます。

形成すべき問い:

- この実装は述べられた目標を実際に達成しているのか、それとも単にそう見えるだけか？
- メリットは直接的に提供されているのか、それとも成立しないかもしれない前提に依存しているのか？
- よりシンプルな仕組みで目標を達成できないか？

**シナリオ:** PRは「プロダクト検索を高速化する」ために`ProductService.getProduct()`にインメモリ`HashMap`キャッシュを追加します。

*なぜという問い:* ここでキャッシュすることがなぜ役に立つのか？実際のボトルネックはどこにあるのか？目標は「高速なプロダクト検索」— これは正しいレバーか？

*自己調査:* `ProductService`はSpringの`@Service`（シングルトン）です。その`HashMap`キャッシュは1つのJVMインスタンスに存在します。マルチインスタンスデプロイでは、各インスタンスが独立してキャッシュを構築し、キャッシュヒットはノード間で分散されます。さらに深刻なのは、別のエンドポイントからプロダクトが更新されるとき、このキャッシュは決して無効化されず、呼び出し元は無期限に古いデータを見続けます。

*代替案:* すべてのインスタンスが同じデータを共有し、古いエントリが予測可能に期限切れになるように、共有のTTLベースキャッシュ（RedisまたはSpringの`@Cacheable`とエビクションポリシー）を使用します。

❌ 元の実装 — 共有されず、無効化されないキャッシュ:

```java
@Service
public class ProductService {
    private final Map<Long, Product> cache = new HashMap<>();

    public Product getProduct(Long id) {
        return cache.computeIfAbsent(id, productRepository::findById);
    }

    public void updateProduct(Product product) {
        productRepository.save(product); // キャッシュは決してクリアされない
    }
}
```

✅ 代替案 — 自動エビクションを伴う共有キャッシュ:

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

代替案は目標（高速な検索）を達成し、インスタンス間で機能し、更新後もデータの一貫性を保ちます。

</details>

<details>
<summary>コメントする前に自己調査する方法</summary>

「なぜ」という問いをコメントとして上げるだけで自分で調査しないと、作業を著者に移すだけで価値を追加しません。レビュアーの仕事は尋問ではなく、調査することです。

**プロセス:**

1. **問いを形成する** — 具体的な「なぜ」を明確に表現する: *なぜこのメカニズム、なぜこの場所、なぜこれが目標を達成するのか*。
2. **制約を調査する** — PRの説明、リンクされたチケット、周辺コードを確認する。著者には正当な理由があったかもしれない: フレームワークの制約、締め切り、別チームのAPIへの依存。
3. **代替案を特定する** — より良い道が存在するなら、具体的に説明する: どのようなものか、どんなメリットをもたらすか、どんなトレードオフがあるか。
4. **発見があった場合のみコメントする** — 調査によって元の選択が制約を考慮した上で合理的だったことが分かれば、その問いを取り下げる。本当の改善が存在するなら、すでに代替案のスケッチと一緒に上げる。

**良い代替案ベースのコメントの例:**

> `processOrder`がステータステーブルを5秒ごとにポーリングしていることに気づきました。ステータス変更は既存の`OrderStatusChanged`イベントによってトリガーされているので、そのイベントを直接消費してポーリングを排除できます。DBの負荷が下がり、レイテンシが〜5秒からほぼゼロになります。私が知らない制約（例: イベントがまだ信頼性に欠けるなど）があってポーリングの方が安全な場合もあるかもしれません — 議論する価値がありそうです。

このコメントは:

- 観察内容を述べている
- 具体的な代替案を提案している
- 考えられる制約を認めている
- 変更を要求するのではなく、議論に招いている

</details>
</blockquote>

</details>

---

## 信頼性

> 信頼性の高いシステムは、依存関係に障害が発生しても正しく機能し続けます。

- すべての外部呼び出しにタイムアウトが設定されているか？
- リトライロジックは指数バックオフを使用しているか？
- 不安定な依存関係をサーキットブレーカーが保護しているか？
- 依存関係が利用不能な場合のグレースフルデグレードはあるか？
- ヘルスチェックエンドポイントは依存関係の実際の状態を正確に反映しているか？

**明示的なフォールトトレランスがなければ、一つの遅い依存関係がサービス全体をダウンさせる可能性があります。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>すべての外部呼び出しにタイムアウトが設定されているか？</summary>

すべての外部呼び出し（HTTP、DB、キャッシュ、メッセージブローカー）にはタイムアウトが必要です。タイムアウトがない場合、一つの遅いダウンストリームがスレッドプールを枯渇させ、サービス全体を停止させる可能性があります。

❌ 悪い例 — タイムアウトなし。遅いダウンストリームがスレッドを無期限にブロックする:

```java
RestTemplate restTemplate = new RestTemplate();
PaymentResponse response = restTemplate.postForObject(
    paymentUrl, request, PaymentResponse.class // タイムアウトなし — サービスが遅いとスレッドがハングする
);
```

✅ 良い例 — 接続タイムアウトと読み取りタイムアウトを設定:

```java
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
factory.setConnectTimeout(2_000); // 接続確立に2秒
factory.setReadTimeout(5_000);    // 完全なレスポンスの受信に5秒
RestTemplate restTemplate = new RestTemplate(factory);
```

</details>

<details>
<summary>リトライロジックは指数バックオフを使用しているか？</summary>

**シナリオ:** サードパーティサービスへのAPI呼び出しが一時的な503エラーで時々失敗します。PRはディレイなしで3回すぐにリトライします。負荷がかかると、すべての呼び出し元が同時にリトライし、すでに苦しんでいるサービスに「サンダリングハード」を引き起こし、回復を妨げます。

❌ 悪い例 — ディレイなしの即時リトライ。すべての呼び出し元がサービスを一度に攻撃する:

```java
for (int i = 0; i < 3; i++) {
    try {
        return externalService.call(request);
    } catch (TransientException e) {
        // 即時リトライ — サンダリングハード
    }
}
throw new ServiceUnavailableException();
```

✅ 良い例 — ジッター付き指数バックオフ（Spring Retry）:

```java
@Retryable(
    value = TransientException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 500, multiplier = 2, random = true) // 500ms → ~1s → ジッター付き~1.5s
)
public Response callExternalService(Request request) {
    return externalService.call(request);
}
```

ジッター（各ディレイに追加されるランダム性）は多数のインスタンス間での同期リトライを防ぎます。各呼び出し元が僅かに異なる時間バックオフします。

</details>

<details>
<summary>不安定な依存関係をサーキットブレーカーが保護しているか？</summary>

**シナリオ:** 商品ページがレコメンデーションサービスを呼び出していますが、そのサービスが失敗し始めます。サーキットブレーカーがないと、すべての商品ページリクエストが失敗前に5秒のフルタイムアウトを待ちます。スレッドが積み上がります。コア機能がレコメンデーションに依存しない商品ページが完全に利用不能になります。

❌ 悪い例 — サーキット保護なし。すべての呼び出しがフルタイムアウトを待つ:

```java
public List<Product> getRecommendations(Long userId) {
    return recommendationService.fetch(userId); // 5秒タイムアウト × 全リクエスト → スレッドプール枯渇
}
```

✅ 良い例 — 失敗閾値後にサーキットが開き、フォールバックが即座に返る:

```java
@CircuitBreaker(name = "recommendations", fallbackMethod = "defaultRecommendations")
public List<Product> getRecommendations(Long userId) {
    return recommendationService.fetch(userId);
}

private List<Product> defaultRecommendations(Long userId, Throwable t) {
    return Collections.emptyList(); // 高速フォールバック — レコメンデーションなしでもページがロードされる
}
```

設定された失敗閾値に達した後、サーキットが開き、すべての呼び出しが即座にフォールバックを返します。タイムアウトなし、スレッドの無駄なし。クールダウン期間後、サーキットが再び閉じます。

</details>

<details>
<summary>依存関係が利用不能な場合のグレースフルデグレードはあるか？</summary>

**シナリオ:** PRは商品一覧エンドポイント内にインベントリサービスへの呼び出しを追加します。インベントリサービスがダウンすると、例外が伝播し、商品一覧全体が500を返します。在庫状況の表示は補足的なものであり、機能の本質ではないにもかかわらず。

問い: この依存関係が利用不能な場合、システムが提供できる最小限の有用なレスポンスは何か？リクエスト全体を失敗させるのではなく、キャッシュされたデータ、デフォルト値、または部分的なレスポンスを返してください。

非本質的な依存関係が、コアユーザーフローをダウンさせることがあってはなりません。

</details>

<details>
<summary>ヘルスチェックエンドポイントは依存関係の実際の状態を正確に反映しているか？</summary>

**シナリオ:** サービスは常に`{"status": "UP"}`を返す`/actuator/health`を公開しています。しかし、DBの接続プールが枯渇し、リクエストを実際には処理できません。KubernetesはこれをみてトラフィックをFailしているPodに送り続けます。

確認: ヘルスエンドポイントは実際の依存関係（DBの接続性、キャッシュの可用性、重要なダウンストリームサービス）を検証しているか？常に`UP`を返すヘルスエンドポイントは、ヘルスチェックがないよりも悪いです。オーケストレーターとオンコールエンジニアから障害を隠してしまいます。

Spring Bootでは、依存関係のヘルスインジケーターを設定して`DOWN`が正しく伝播するようにします:

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

## オペレーショナルエクセレンス

> 安全にデプロイできず、本番環境で観察できず、障害時に診断できないコードは、本番対応していません。

- 相関IDを含む構造化ログがあるか？
- 新しいクロスサービス呼び出しに分散トレーシングスパンが追加されているか？
- 新しい重要なエンドポイントにアラートまたはSLOが定義されているか？
- APIの変更は後方互換性があるか、またはバージョン管理されているか？
- 新しい障害モードが特定されているか？

**オペレーショナルエクセレンスは、障害が発生するかどうかではなく、どれだけ早く検出し回復できるかを決定します。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>相関IDを含む構造化ログがあるか？</summary>

マルチスレッドまたはマルチサービスのシステムでは、単一のユーザーリクエストが多くのクラスとスレッドにまたがるログ行を生成します。相関IDがなければ、本番障害の診断は、何千もの入り混じったログエントリを手動で選別することを意味します。

❌ 悪い例 — 文字列連結、リクエストコンテキストなし:

```java
log.info("ユーザー " + userId + " の注文を処理中、金額: " + amount);
// 200スレッドが同時にログを記録すると、1つのリクエストのすべての行を見つける方法がない
```

✅ 良い例 — 構造化フィールド + MDC経由の相関IDが自動的に伝播される:

```java
MDC.put("correlationId", request.getCorrelationId());
log.info("注文処理中: userId={}, amount={}", userId, amount);
// このリクエストスレッドの後続のすべてのログ行に相関IDが自動的に付与される
// ElasticsearchまたはCloudWatchでcorrelationIdフィールドでフィルタリング可能
```

本番対応ログの主要な特性:

- **構造化**（key=value またはJSON）— フルテキスト検索だけでなく、フィールドでフィルタリング可能
- **相関ID** — 1つのリクエストのすべてのログ行をサービスをまたいでリンクする
- **正しいログレベル** — `ERROR`はアクション可能な障害、`WARN`は回復可能な問題、`INFO`は主要ビジネスイベント、`DEBUG`はトラブルシューティングの詳細

</details>

<details>
<summary>新しいクロスサービス呼び出しに分散トレーシングスパンが追加されているか？</summary>

**シナリオ:** PRはチェックアウトフロー内にインベントリサービスへの呼び出しを追加します。呼び出しをトレーススパンが囲んでいません。本番でチェックアウトのレイテンシが急増します。Jaeger/Zipkinのトレースはチェックアウトエンドポイントが合計3秒かかっていることを示していますが、時間がどこで費やされているかの内訳がありません。インベントリの呼び出しが見えません。

確認: 各新しいクロスサービス呼び出し（HTTP、gRPC、メッセージ公開）にトレーススパンが作成されているか？Spring Cloud SleuthまたはMicrometer TracingでHTTPクライアント呼び出しは自動的にインスツルメントされることが多いですが、カスタム非同期タスクやスレッドの引き渡しには明示的なスパンが必要な場合があります:

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
<summary>新しい重要なエンドポイントにアラートまたはSLOが定義されているか？</summary>

**シナリオ:** 新しいPOST /paymentsエンドポイントがエラーレートアラートなしでデプロイされます。3日後、支払いゲートウェイの設定変更により、30%の支払いリクエストが黙って失敗します。アラートは発火しません。エンジニアリングチームは4時間後に顧客からの苦情で気づきます。

問い: このエンドポイントは重要なユーザーパスにあるか？YESなら、デプロイ前に定義する:

- エラーレートアラート（例: 5分間で5xxが1%超 → オンコールをページ）
- レイテンシSLO（例: p99 < 2秒）
- エラーレートとレイテンシのダッシュボードパネル

PR自体がアラートを設定しない場合でも、リスクが高い場合はレビュアーがその欠如を指摘すべきです。

</details>

<details>
<summary>APIの変更は後方互換性があるか、またはバージョン管理されているか？</summary>

**シナリオ:** レスポンスJSONのフィールドが`userName`から`username`に名前変更されます。変更がデプロイされます。強制更新されていない既存のモバイルクライアントは、依存しているフィールドに対して`null`を受け取り始めます。

❌ 悪い例 — フィールドが名前変更され、既存のクライアントが即座に壊れる:

```java
// 変更前: { "userName": "john_doe" }
public record UserResponse(String username) {} // 名前変更 — 古いクライアントは"userName"にnullを受け取る
```

✅ 良い例 — 古いフィールドを保持。クライアントは自分のスケジュールで移行できる:

```java
public record UserResponse(
    String username,             // 新しい正式名
    @Deprecated String userName  // クライアントが移行するまで後方互換性のために保持
) {}
```

確認:

- このリクエスト/レスポンスフィールドを削除または名前変更するか？
- 既存フィールドのセマンティクスを変更するか？
- 以前はオプションだった場所に必須フィールドを追加するか？

いずれかのYESには、バージョン管理戦略が必要です: フィールドエイリアス、APIバージョンサフィックス（`/v2/`）、または両方の名前が存在する移行期間。

</details>

<details>
<summary>新しい障害モードが特定されているか？</summary>

**シナリオ:** PRは注文を払い戻すバックグラウンドジョブを追加します。ジョブはまずDBを更新し、次に銀行APIを呼び出します。DB更新後に銀行呼び出しが失敗すると、注文は払い戻し済みとしてマークされますが、実際にはお金が送られていません。これは以前は存在しなかった新しい障害モード — 部分的な状態 — です。

PRが新しい障害モード（特にデータの不整合やサイレントなデータ損失を引き起こす可能性があるもの）を導入する場合、レビュアーは次のことを問うべきです:

- どのように検出するか？（エラーログ、アラート、照合ジョブ）
- どのように修復するか？（手動再実行、自動リトライ、補償トランザクション）

PRに回答がない場合、コメントを残す: *「DB更新後に銀行呼び出しが失敗した場合、不整合をどのように検出し調整しますか？」*

</details>
</blockquote>

</details>

---

## コスト最適化

> 無駄なパターンはスケールで複利化します — 低トラフィック時は無害に見えるクエリパターンも、本番量では大きなクラウドコストを生み出す可能性があります。

- 実際に必要な列/フィールドのみを取得しているか？
- データ転送とエグレスのコストは考慮されているか？
- ワークロードのパターンに対してコンピュートは適切なサイズか？
- キャッシュによって冗長なダウンストリーム呼び出しが削減されているか？
- レコードごとのAPI呼び出しの代わりにバッチ操作が使用されているか？

**コストは非機能要件です。スケールでは、誤ったアクセスパターンはパフォーマンスと予算の両方の問題です。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>実際に必要な列/フィールドのみを取得しているか？</summary>

**要件:** すべてのユーザーにウェルカムメールを送信する。

❌ 悪い例 — 2つのフィールドだけが必要な場合でも、大きなBLOBを含むエンティティ全体を取得する:

```java
List<User> users = userRepository.findAll(); // SELECT * — アバター、設定、監査履歴を取得
for (User user : users) {
    emailService.sendWelcome(user.getEmail(), user.getName()); // 2つのフィールドだけ使用
}
```

✅ 良い例 — プロジェクションで必要な列だけを取得:

```java
public interface UserEmailView {
    String getName();
    String getEmail();
}

List<UserEmailView> users = userRepository.findAllProjectedBy(); // SELECT name, email FROM users
```

余分な列はそれぞれ、DBのI/O、DBとアプリサーバー間のネットワークバイト、JVMヒーププレッシャーを増やします。数百万行では、レイテンシ、メモリ、コストの差は大きくなります。

</details>

<details>
<summary>データ転送とエグレスのコストは考慮されているか？</summary>

**シナリオ:** 新しいリストエンドポイントは、各注文レスポンスに商品カタログ全体をネストされたJSON配列として埋め込みます（1レスポンスあたり500以上の商品になることも）。エンドポイントはモバイルアプリから呼び出され、注文状況と合計額のみを表示します。

1日10,000リクエスト、それぞれが200KBの未使用のネストデータを返すと、クラウドプロバイダーのレートで1日2GBのエグレスが課金されます。

確認:

- コンシューマーが実際に表示または使用するフィールドは何か？
- 呼び出し元がオプトインできる代わりに、大きなネストオブジェクトがデフォルトで含まれているか？
- レスポンスペイロードは呼び出し元が必要とするものに比例しているか？

スパースフィールドセット、プロジェクションパラメーター、またはサマリービューと詳細ビューの分離エンドポイントを検討してください。

</details>

<details>
<summary>ワークロードのパターンに対してコンピュートは適切なサイズか？</summary>

**シナリオ:** 照合ジョブが1時間ごとに約30秒間実行されます。メインアプリケーションサービスの長期実行スレッドとして実装されており、何もしない59分間もメモリとスレッドスロットを継続的に消費します。

コンピュートモデルをワークロードに合わせてください:

| ワークロード | より適したもの |
| --- | --- |
| イベントによってトリガー、短命 | Lambda / Fargateタスク |
| スケジュール済み、短時間 | スケジュールLambda / Kubernetes CronJob |
| 高スループット、継続的 | 常時起動サービス（EC2、ECS） |
| 重いバッチ、定期的 | AWS Batch / Kubernetes Job |

非頻繁なジョブを常時起動サービスに埋め込むとコンピュートが無駄になります。逆に、10,000 TPSのホットパスにLambdaを使用すると、コールドスタートレイテンシと呼び出しごとのコストが専用インスタンスを超える場合があります。

</details>

<details>
<summary>キャッシュによって冗長なダウンストリーム呼び出しが削減されているか？</summary>

**シナリオ:** 商品詳細APIが1秒間に100回呼ばれます。各呼び出しで、外部セラーサービスからセラープロフィールを取得していますが、セラープロフィールはほぼ変わりません。キャッシュは存在しません。

100 RPSでは、1日に8,640,000回の外部呼び出しが、ほとんど変わらないデータに対して行われます。

問い: このデータは更新頻度よりもはるかに頻繁に読まれるか？YESなら、短いTTL（60秒）でも外部呼び出しを99.9%削減でき、陳腐化リスクはほとんどありません。

```java
@Cacheable(value = "sellerProfiles", key = "#sellerId")
public SellerProfile getSellerProfile(Long sellerId) {
    return sellerService.fetchProfile(sellerId); // セラーごとに1分間キャッシュ
}
```

</details>

<details>
<summary>レコードごとのAPI呼び出しの代わりにバッチ操作が使用されているか？</summary>

**シナリオ:** PRはループ内でユーザーごとにプッシュ通知サービスを1回呼び出す通知機能を追加します。

❌ 悪い例 — Nユーザーに対してN回の個別HTTP呼び出し:

```java
for (User user : usersToNotify) {
    notificationService.sendPush(user.getId(), message); // ユーザーごとに1回のHTTP呼び出し
}
```

✅ 良い例 — 単一のバッチ呼び出し:

```java
List<Long> userIds = usersToNotify.stream().map(User::getId).collect(Collectors.toList());
notificationService.sendPushBatch(userIds, message); // 全ユーザーに対して1回のHTTP呼び出し
```

これは外部APIのN+1 DBクエリと同等です。不必要な各呼び出しはネットワークレイテンシ、接続オーバーヘッド、APIレート制限または呼び出しごとの課金プレッシャーを追加します。

</details>
</blockquote>

</details>

---

## 持続可能性

> 持続可能なソフトウェアは少ないリソースで多くを達成します — CPUサイクルの削減、メモリの削減、ネットワークラウンドトリップの削減、そして不要になったリソースの迅速な解放。

- リソース（接続、ストリーム、スレッド）は使用後に明示的に解放されているか？
- ホットループ内に冗長な計算はないか？
- アクセスパターンに対して効率的なデータ構造が選ばれているか？
- レコードごとのリクエストの代わりにバッチAPI呼び出しが使用されているか？
- バックグラウンドジョブは必要な間だけ実行されるようにスコープされているか？

**持続可能なコードは、クラウドコスト、カーボンフットプリント、システム負荷を同時に削減します。**

<details>
<summary>詳細</summary>

<blockquote>
<details>
<summary>リソースは使用後に明示的に解放されているか？</summary>

**要件:** 起動時に設定ファイルを読み込む。

❌ 悪い例 — ストリームが閉じられない。繰り返し呼び出されるとOSのファイルハンドルがリークする:

```java
public String readConfig(String path) throws IOException {
    InputStream stream = new FileInputStream(path);
    return new String(stream.readAllBytes()); // ストリームが閉じられない
}
```

✅ 良い例 — try-with-resourcesが終了時または例外発生時にクリーンアップを保証する:

```java
public String readConfig(String path) throws IOException {
    try (InputStream stream = new FileInputStream(path)) {
        return new String(stream.readAllBytes());
    }
}
```

リソースリークは負荷がかかるにつれて複利化します。リークしたDB接続はそれぞれプールの可用性を下げ、やがて新しいリクエストが接続を取得できなくなりサービスが停止します。同じパターンをDB接続、HTTPクライアント、ファイルハンドル、スレッドプールエグゼキューターに適用してください。

</details>

<details>
<summary>ホットループ内に冗長な計算はないか？</summary>

**要件:** アクティブなプロモーションの最低注文金額を満たす注文をフィルタリングする。

❌ 悪い例 — 結果が変わらないにもかかわらず、イテレーションごとに高コストのDB呼び出しを行う:

```java
public List<Order> filterByActivePromotion(List<Order> orders) {
    List<Order> result = new ArrayList<>();
    for (Order order : orders) {
        Promotion active = promotionService.getActivePromotion(); // イテレーションごとにDB呼び出し
        if (order.getTotal() >= active.getMinimumOrderValue()) {
            result.add(order);
        }
    }
    return result;
}
```

✅ 良い例 — ループの外で一度取得し、再利用する:

```java
public List<Order> filterByActivePromotion(List<Order> orders) {
    Promotion active = promotionService.getActivePromotion(); // 一度取得
    return orders.stream()
        .filter(order -> order.getTotal() >= active.getMinimumOrderValue())
        .collect(Collectors.toList());
}
```

イテレーションごとの1回の余分なDB呼び出しは10アイテムでは無視できます。10,000アイテムと100 RPSでは、毎秒100万回の不必要なDB呼び出しになります。

</details>

<details>
<summary>アクセスパターンに対して効率的なデータ構造が選ばれているか？</summary>

**要件:** リストからブラックリストに載っている商品をフィルタリングする。

❌ 悪い例 — `List.contains()`は1回のチェックにO(n)。全体の複雑度はO(n²):

```java
public List<Product> filterBlacklisted(List<Product> products, List<Long> blacklistedIds) {
    return products.stream()
        .filter(p -> !blacklistedIds.contains(p.getId())) // O(n)のcontainsがO(n)のstream内
        .collect(Collectors.toList());
}
```

✅ 良い例 — 一度`Set`に変換。各ルックアップはO(1):

```java
public List<Product> filterBlacklisted(List<Product> products, List<Long> blacklistedIds) {
    Set<Long> blacklistSet = new HashSet<>(blacklistedIds); // 一度変換 — O(n)
    return products.stream()
        .filter(p -> !blacklistSet.contains(p.getId()))     // ルックアップはO(1)
        .collect(Collectors.toList());
}
```

非効率なデータ構造はCPUサイクルを無駄にします。スケールでは、これは直接コンピュートコストとエネルギー消費に変わります。

</details>

<details>
<summary>バックグラウンドジョブは必要な間だけ実行されるようにスコープされているか？</summary>

**シナリオ:** PRはキューを5分ごとに処理する`@Scheduled`ジョブを追加します。処理後、ジョブは残りの5分間何もしませんが、スレッドとDB接続を保持し続けます。

確認:

- このジョブは実行間にリソースを保持しているか？
- スケジュールではなくオンデマンド（イベント駆動）でトリガーできないか？
- スケジュール済みの場合、各実行の開始時にリソースを取得し、終了時にすぐ解放しているか？

イベント駆動アプローチ（メッセージが存在する場合のみキューから消費）は、アイドル時のリソース消費を完全に回避します。ポーリングが必要な場合は、各実行の開始時にリソースを取得し、終了時に解放してください — 実行間で保持したままにしてはなりません。

</details>
</blockquote>

</details>
