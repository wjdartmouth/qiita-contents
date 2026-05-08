---
title: "依存性注入（DI）でテスタブルなiOSアプリを作る【実践シリーズ #3】"
tags:
  - iOS
  - Swift
  - DependencyInjection
  - testing
  - architecture
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# 依存性注入（DI）でテスタブルなiOSアプリを作る

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第3回 / 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

前回はViewModelの責務分離を扱いました。しかし「プロトコルで抽象化する」だけでは、依存オブジェクトの生成と管理が煩雑になりがちです。

今回は**依存性注入（Dependency Injection, DI）**を体系的に整理し、Swiftでの実践パターンを解説します。

---

## DIとは何か

DIとは「オブジェクトが自分の依存先を自分で生成しない」設計原則です。

```swift
// ❌ DIなし：依存を自分で生成
final class OrderViewModel {
    // ViewModelが具体型に強く依存している
    private let service = OrderService()
    private let analytics = FirebaseAnalytics()
    private let logger = OSLogger()
}

// ✅ DIあり：外から注入
final class OrderViewModel {
    private let service: OrderServiceProtocol
    private let analytics: AnalyticsProtocol
    private let logger: LoggerProtocol

    init(
        service: OrderServiceProtocol,
        analytics: AnalyticsProtocol,
        logger: LoggerProtocol
    ) {
        self.service = service
        self.analytics = analytics
        self.logger = logger
    }
}
```

DIのメリットは3つです。

1. **テスト時にMockと差し替えられる**
2. **具体実装への依存が減り、変更に強くなる**
3. **依存関係が明示的になり、読みやすくなる**

---

## DIの3つのパターン

### 1. イニシャライザインジェクション（推奨）

```swift
final class UserViewModel {
    private let repository: UserRepositoryProtocol

    // 依存はイニシャライザで注入
    init(repository: UserRepositoryProtocol) {
        self.repository = repository
    }
}

// 本番環境
let vm = UserViewModel(repository: UserRepository())

// テスト環境
let vm = UserViewModel(repository: MockUserRepository())
```

最もシンプルで、依存が必須であることが明確です。SwiftではこれがDIの**第一選択**です。

### 2. プロパティインジェクション

```swift
final class AnalyticsViewModel {
    // 後から注入できる（任意の依存に使う）
    var logger: LoggerProtocol = DefaultLogger()
}

let vm = AnalyticsViewModel()
vm.logger = MockLogger()  // テスト時に差し替え
```

任意の依存や循環参照を避けたい場面で使います。ただし注入を忘れると実行時エラーになるリスクがあります。

### 3. 環境オブジェクト（SwiftUI向け）

SwiftUIの `@Environment` / `@EnvironmentObject` を使うパターン。

```swift
// 環境に注入する値を定義
struct UserRepositoryKey: EnvironmentKey {
    static let defaultValue: UserRepositoryProtocol = UserRepository()
}

extension EnvironmentValues {
    var userRepository: UserRepositoryProtocol {
        get { self[UserRepositoryKey.self] }
        set { self[UserRepositoryKey.self] = newValue }
    }
}

// Viewで取得
struct UserProfileView: View {
    @Environment(\.userRepository) private var repository

    var body: some View { ... }
}

// ルートで注入（本番）
ContentView()
    .environment(\.userRepository, UserRepository())

// プレビュー・テストで差し替え
ContentView()
    .environment(\.userRepository, MockUserRepository())
```

---

## DIコンテナを自作する

中規模以上のアプリでは、依存の管理を一元化する**DIコンテナ**が有効です。

```swift
// シンプルなDIコンテナ
final class DIContainer {
    static let shared = DIContainer()
    private var factories: [ObjectIdentifier: () -> Any] = [:]

    private init() {}

    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        factories[ObjectIdentifier(type)] = factory
    }

    func resolve<T>(_ type: T.Type) -> T {
        guard let factory = factories[ObjectIdentifier(type)],
              let instance = factory() as? T else {
            fatalError("依存が未登録: \(type)")
        }
        return instance
    }
}

// アプリ起動時に登録
extension DIContainer {
    func registerDependencies() {
        register(UserRepositoryProtocol.self) { UserRepository() }
        register(OrderRepositoryProtocol.self) { OrderRepository() }
        register(AnalyticsProtocol.self) { FirebaseAnalytics() }
    }
}

// 使用例
let repo = DIContainer.shared.resolve(UserRepositoryProtocol.self)
```

### テスト用コンテナの差し替え

```swift
// テスト用セットアップ
final class AppTests: XCTestCase {
    override func setUp() {
        super.setUp()
        // テスト用の依存を登録
        DIContainer.shared.register(UserRepositoryProtocol.self) {
            MockUserRepository()
        }
    }
}
```

---

## Point-Free の Dependencies ライブラリ

TCAとセットで使われることが多い [Dependencies](https://github.com/pointfreeco/swift-dependencies) は、SwiftネイティブなDI体験を提供します。

```swift
import Dependencies

// 依存の定義
struct UserClient {
    var fetchUser: @Sendable (Int) async throws -> User
}

extension UserClient: DependencyKey {
    static let liveValue = UserClient(
        fetchUser: { id in
            try await URLSession.shared.decode(User.self, from: URL(string: "/users/\(id)")!)
        }
    )

    // テスト時のデフォルト値
    static let testValue = UserClient(
        fetchUser: { _ in
            User(id: 1, name: "テストユーザー")
        }
    )

    // プレビュー用
    static let previewValue = UserClient(
        fetchUser: { _ in
            User(id: 99, name: "プレビューユーザー")
        }
    )
}

extension DependencyValues {
    var userClient: UserClient {
        get { self[UserClient.self] }
        set { self[UserClient.self] = newValue }
    }
}

// ViewModelで使用
@Observable
final class UserViewModel {
    @Dependency(\.userClient) var userClient

    func loadUser(id: Int) async throws {
        let user = try await userClient.fetchUser(id)
        // ...
    }
}
```

テスト時は `withDependencies` でスコープを絞って差し替えられます。

```swift
func test_loadUser() async throws {
    let viewModel = withDependencies {
        $0.userClient.fetchUser = { _ in User(id: 1, name: "Mock") }
    } operation: {
        UserViewModel()
    }

    try await viewModel.loadUser(id: 1)
    // assertions...
}
```

---

## Mockを効率よく書く方法

プロトコルのMockは手書きすると大量のボイラープレートが発生します。[Mockolo](https://github.com/uber/mockolo) や [swift-mock](https://github.com/MetalheadSanya/swift-mock) を使うか、シンプルに手書きするか、チームの規模に合わせて選択しましょう。

手書きする場合のパターン：

```swift
final class MockUserRepository: UserRepositoryProtocol {
    // 呼ばれたかを記録
    var fetchCallCount = 0
    var saveCallCount = 0

    // スタブ値
    var fetchResult: Result<User, Error> = .success(User(id: 1, name: "Mock"))

    func fetch(id: Int) async throws -> User {
        fetchCallCount += 1
        return try fetchResult.get()
    }

    func save(_ user: User) async throws {
        saveCallCount += 1
    }
}
```

---

## まとめ

| パターン | 使いどころ |
|---|---|
| イニシャライザDI | 必須の依存。ほぼすべてのケースでこれを使う |
| プロパティDI | 任意の依存、循環参照の回避 |
| EnvironmentDI | SwiftUIのView階層全体で共有する依存 |
| DIコンテナ | 中規模以上、依存の管理を一元化したい場合 |
| Dependenciesライブラリ | TCA使用時、または型安全なDIが欲しい場合 |

次回はいよいよ**The Composable Architecture（TCA）**の入門編です。副作用の管理という、TCA最大の強みに焦点を当てます。

---

## 連載インデックス

- 第1回：アーキテクチャ選択ガイド
- 第2回：SwiftUIとMVVMの実践
- **第3回**：依存性注入（DI）（本記事）
- 第4回：TCA入門：副作用の管理
- 第5回：モジュール分割でスケールするコードベース
- 第6回：SwiftDataとClean Architectureを組み合わせる
- 第7回：チームで守れるアーキテクチャ指針の作り方
