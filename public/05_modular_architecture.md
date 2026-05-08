---
title: "モジュール分割でスケールするiOSコードベースを構築する【実践シリーズ #5】"
tags:
  - iOS
  - Swift
  - SwiftPackageManager
  - architecture
  - modular
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# モジュール分割でスケールするiOSコードベースを構築する

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第5回 / 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

アプリが成長すると、こんな問題が出てきます。

- ビルドに5分以上かかる
- どこを変更すればいいか分からない
- 関係のないコードを誤って変更してしまう
- 新機能の追加が既存機能に予期せず影響する

これらはすべて「モノリシックなコードベース」が引き起こす問題です。**モジュール分割（Modular Architecture）**はこれを根本から解決します。

---

## モジュール分割の何が嬉しいのか

### ビルド時間の短縮

Swiftのコンパイラはモジュール単位でインクリメンタルビルドを行います。変更のあったモジュールだけ再ビルドされるため、大規模プロジェクトほど効果が大きくなります。

```
モノリシック：全体を毎回ビルド → 5分
モジュール分割：変更モジュールのみ → 30秒
```

### 依存関係の明確化

```
❌ モノリシック：どこからでも何でも呼べる
✅ モジュール分割：publicなAPIしか呼べない
```

### 並行開発の促進

Aチームがユーザー機能モジュール、Bチームが決済モジュールを独立して開発できます。

---

## モジュール設計の基本戦略

よく使われる分割パターンは「**機能軸 × レイヤー軸**」の組み合わせです。

```
App（アプリターゲット）
│
├── Features/           ← 機能モジュール群
│   ├── HomeFeature
│   ├── ProfileFeature
│   ├── OrderFeature
│   └── AuthFeature
│
├── Core/               ← 共有コアモジュール群
│   ├── DesignSystem    ← UI部品・カラー・フォント
│   ├── NetworkClient   ← API通信の基盤
│   ├── Analytics       ← 計測
│   └── Logger
│
└── Domain/             ← ドメイン層（ビジネスロジック）
    ├── Models          ← エンティティ
    └── Repositories    ← リポジトリプロトコル
```

**依存方向のルール（絶対に逆にしない）：**

```
App → Features → Core → Domain
                         ↑
                 外部依存なし
```

---

## Swift Package Manager（SPM）でマルチモジュール構成を作る

Xcodeでローカルパッケージを使ったマルチモジュール構成の作り方を解説します。

### ディレクトリ構成

```
MyApp/
├── MyApp.xcodeproj
├── MyApp/               ← アプリターゲット
│   └── MyAppApp.swift
└── Packages/            ← ローカルパッケージ群
    ├── DesignSystem/
    │   ├── Package.swift
    │   └── Sources/
    │       └── DesignSystem/
    ├── NetworkClient/
    │   ├── Package.swift
    │   └── Sources/
    └── HomeFeature/
        ├── Package.swift
        └── Sources/
```

### Package.swift の例（NetworkClient）

```swift
// Packages/NetworkClient/Package.swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "NetworkClient",
    platforms: [.iOS(.v17)],
    products: [
        // 外部に公開するターゲット
        .library(name: "NetworkClient", targets: ["NetworkClient"]),
        // テスト用のMockも公開
        .library(name: "NetworkClientMock", targets: ["NetworkClientMock"]),
    ],
    dependencies: [],
    targets: [
        .target(
            name: "NetworkClient",
            path: "Sources/NetworkClient"
        ),
        .target(
            name: "NetworkClientMock",
            dependencies: ["NetworkClient"],
            path: "Sources/NetworkClientMock"
        ),
        .testTarget(
            name: "NetworkClientTests",
            dependencies: ["NetworkClient", "NetworkClientMock"],
            path: "Tests/NetworkClientTests"
        ),
    ]
)
```

### Package.swift の例（HomeFeature）

```swift
// Packages/HomeFeature/Package.swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "HomeFeature",
    platforms: [.iOS(.v17)],
    products: [
        .library(name: "HomeFeature", targets: ["HomeFeature"]),
    ],
    dependencies: [
        // 依存するローカルパッケージを指定
        .package(path: "../NetworkClient"),
        .package(path: "../DesignSystem"),
        .package(path: "../Domain"),
    ],
    targets: [
        .target(
            name: "HomeFeature",
            dependencies: [
                "NetworkClient",
                "DesignSystem",
                .product(name: "Domain", package: "Domain"),
            ]
        ),
        .testTarget(
            name: "HomeFeatureTests",
            dependencies: [
                "HomeFeature",
                .product(name: "NetworkClientMock", package: "NetworkClient"),
            ]
        ),
    ]
)
```

---

## モジュール間の画面遷移を設計する

モジュール分割で悩みやすいのが**画面遷移**です。HomeFeatureがProfileFeatureを直接参照すると、循環依存が生まれる可能性があります。

### 解決策：Coordinatorパターン

```swift
// AppModule（最上位）がCoordinatorを持つ
@main
struct MyApp: App {
    @State private var coordinator = AppCoordinator()

    var body: some Scene {
        WindowGroup {
            AppRootView(coordinator: coordinator)
        }
    }
}

// AppCoordinatorはすべてのFeatureを知っている
@Observable
final class AppCoordinator {
    var path = NavigationPath()

    enum Route: Hashable {
        case home
        case profile(userID: String)
        case orderDetail(orderID: String)
    }

    func navigate(to route: Route) {
        path.append(route)
    }

    func pop() {
        path.removeLast()
    }
}

// AppRootView
struct AppRootView: View {
    let coordinator: AppCoordinator

    var body: some View {
        NavigationStack(path: Binding(
            get: { coordinator.path },
            set: { coordinator.path = $0 }
        )) {
            HomeView(
                // HomeFeatureはAppCoordinatorを知らない
                // コールバックで遷移を通知するだけ
                onProfileTapped: { userID in
                    coordinator.navigate(to: .profile(userID: userID))
                }
            )
            .navigationDestination(for: AppCoordinator.Route.self) { route in
                switch route {
                case .home:
                    HomeView(onProfileTapped: { _ in })
                case .profile(let userID):
                    ProfileView(userID: userID)
                case .orderDetail(let orderID):
                    OrderDetailView(orderID: orderID)
                }
            }
        }
    }
}
```

HomeFeatureはコールバック（クロージャ）を受け取るだけで、他のFeatureモジュールを一切インポートしません。

---

## アクセス修飾子の戦略

モジュール分割の効果を最大化するには、`public`/`internal`/`private` を意識的に使います。

```swift
// DesignSystemモジュール内

// ✅ 外部に公開するコンポーネント
public struct PrimaryButton: View {
    public let title: String
    public let action: () -> Void

    public init(title: String, action: @escaping () -> Void) {
        self.title = title
        self.action = action
    }

    public var body: some View {
        Button(title, action: action)
            .buttonStyle(PrimaryButtonStyle())
    }
}

// ✅ モジュール内部でのみ使う（外部に漏らさない）
struct PrimaryButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .padding(.horizontal, 24)
            .padding(.vertical, 12)
            .background(Color.accentColor)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 8))
            .opacity(configuration.isPressed ? 0.8 : 1.0)
    }
}
```

---

## ビルド時間を計測する

効果を数値で確認するために、ビルド時間を計測しましょう。

```bash
# Xcodeのビルドタイミングを表示
defaults write com.apple.dt.Xcode ShowBuildOperationDuration YES

# より詳細な計測にはxcbeautifyがおすすめ
brew install xcbeautify
xcodebuild build | xcbeautify
```

---

## まとめ

モジュール分割のポイントをまとめます。

1. **依存方向を決めて絶対に守る**（App → Feature → Core → Domain）
2. **SPMのローカルパッケージ**でモジュールを作る
3. **Coordinatorパターン**で画面遷移をモジュール外に切り出す
4. **publicを最小限に**してモジュールのAPIを小さく保つ
5. **各モジュールにテストターゲット**を持たせる

次回は **SwiftDataとClean Architecture** の組み合わせを解説します。ローカル永続化をどうアーキテクチャに組み込むかを実践的に扱います。

---

## 連載インデックス

- 第1回：アーキテクチャ選択ガイド
- 第2回：SwiftUIとMVVMの実践
- 第3回：依存性注入（DI）
- 第4回：TCA入門：副作用の管理
- **第5回**：モジュール分割（本記事）
- 第6回：SwiftDataとClean Architectureを組み合わせる
- 第7回：チームで守れるアーキテクチャ指針の作り方
