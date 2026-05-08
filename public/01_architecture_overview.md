---
title: "iOSアーキテクチャ選択ガイド：MVC・MVP・MVVM・TCA徹底比較【実践シリーズ #1】"
tags:
  - iOS
  - Swift
  - SwiftUI
  - architecture
  - 設計
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# iOSアーキテクチャ選択ガイド：MVC・MVP・MVVM・TCA徹底比較

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第1回 / 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

「このプロジェクトにはどのアーキテクチャを使えばいい？」

iOSエンジニアなら一度は悩んだことがあるはずです。MVC、MVP、MVVM、TCA——それぞれが「正解」として語られる場面があり、何を選べばよいか迷うのは当然です。

この記事では、各アーキテクチャの本質的な違いと、**実際のプロジェクトでどう選ぶべきか**を整理します。

---

## アーキテクチャが必要な理由

まず原点に立ち返りましょう。アーキテクチャとは、コードの「責務の分け方のルール」です。ルールがないと何が起きるか——。

```swift
// ❌ アーキテクチャなしの典型例（Massive View Controller）
class UserProfileViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        // API通信
        URLSession.shared.dataTask(with: URL(string: "https://api.example.com/user")!) { data, _, _ in
            guard let data else { return }
            // JSONパース
            let user = try? JSONDecoder().decode(User.self, from: data)
            // UI更新
            DispatchQueue.main.async {
                self.nameLabel.text = user?.name
                // バリデーション
                if user?.name.isEmpty == true {
                    self.showAlert(message: "名前が空です")
                }
                // 永続化
                UserDefaults.standard.set(user?.name, forKey: "userName")
            }
        }.resume()
    }
}
```

API通信・パース・UI更新・バリデーション・永続化がすべてViewControllerに詰め込まれています。これが「Massive View Controller」問題です。テストも書けず、再利用も難しい。

---

## 4つのアーキテクチャを理解する

### 1. MVC（Model-View-Controller）

Appleが標準として提供するパターン。

```
Model ←→ Controller ←→ View
```

**Controller** がModelとViewの橋渡しをします。シンプルですが、ControllerがViewに強く依存しやすく、巨大化しがちです。

```swift
// ✅ MVCの正しい分け方
// Model
struct User: Decodable {
    let id: Int
    let name: String
}

// Controller
class UserViewController: UIViewController {
    private var user: User?

    func fetchUser() {
        UserService.fetch { [weak self] user in
            self?.user = user
            self?.updateView()
        }
    }

    private func updateView() {
        nameLabel.text = user?.name
    }
}
```

**向いているケース：** 小規模アプリ、プロトタイプ、チームがAppleのガイドラインに慣れている場合

---

### 2. MVP（Model-View-Presenter）

ControllerをPresenterに置き換え、ViewとPresenterをプロトコルで分離します。

```
Model ←→ Presenter ←→ View（Protocol）
```

```swift
// View側のプロトコル
protocol UserViewProtocol: AnyObject {
    func displayUser(name: String)
    func showError(message: String)
}

// Presenter（UIKitに依存しない）
class UserPresenter {
    weak var view: UserViewProtocol?
    private let service: UserServiceProtocol

    init(service: UserServiceProtocol) {
        self.service = service
    }

    func loadUser(id: Int) {
        service.fetch(id: id) { [weak self] result in
            switch result {
            case .success(let user):
                self?.view?.displayUser(name: user.name)
            case .failure(let error):
                self?.view?.showError(message: error.localizedDescription)
            }
        }
    }
}

// ViewController（薄く保つ）
class UserViewController: UIViewController, UserViewProtocol {
    var presenter: UserPresenter!

    func displayUser(name: String) {
        nameLabel.text = name
    }

    func showError(message: String) {
        showAlert(message: message)
    }
}
```

**向いているケース：** UIKitメインのプロジェクト、テストを書き始めたいチーム

---

### 3. MVVM（Model-View-ViewModel）

SwiftUIとの相性が抜群。ViewModelがObservableで状態を管理します。

```
Model ←→ ViewModel（@Observable） ←→ View
```

```swift
// ViewModel
@Observable
class UserViewModel {
    var userName: String = ""
    var isLoading: Bool = false
    var errorMessage: String?

    private let service: UserServiceProtocol

    init(service: UserServiceProtocol = UserService()) {
        self.service = service
    }

    func loadUser(id: Int) async {
        isLoading = true
        defer { isLoading = false }

        do {
            let user = try await service.fetch(id: id)
            userName = user.name
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

// View（状態を表示するだけ）
struct UserProfileView: View {
    @State private var viewModel = UserViewModel()

    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else {
                Text(viewModel.userName)
            }
        }
        .task { await viewModel.loadUser(id: 1) }
        .alert("エラー", isPresented: .constant(viewModel.errorMessage != nil)) {
            Button("OK") { viewModel.errorMessage = nil }
        } message: {
            Text(viewModel.errorMessage ?? "")
        }
    }
}
```

**向いているケース：** SwiftUIプロジェクト、中規模以上のアプリ

---

### 4. TCA（The Composable Architecture）

Point-Free社が開発したSwift特化の関数型アーキテクチャ。

```
View → Action → Reducer → State → View
```

状態・アクション・副作用がすべて明示的に管理されます。

```swift
// State・Action・Reducerを一箇所に定義
@Reducer
struct UserFeature {
    @ObservableState
    struct State: Equatable {
        var userName: String = ""
        var isLoading: Bool = false
    }

    enum Action {
        case loadUserTapped
        case userResponse(Result<User, Error>)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .loadUserTapped:
                state.isLoading = true
                return .run { send in
                    await send(.userResponse(
                        Result { try await UserService().fetch(id: 1) }
                    ))
                }
            case .userResponse(.success(let user)):
                state.isLoading = false
                state.userName = user.name
                return .none
            case .userResponse(.failure):
                state.isLoading = false
                return .none
            }
        }
    }
}
```

**向いているケース：** 大規模・チーム開発、副作用が多い複雑なアプリ、テストを徹底したい場合

---

## 比較まとめ

| | MVC | MVP | MVVM | TCA |
|---|---|---|---|---|
| **学習コスト** | 低 | 中 | 中 | 高 |
| **テスタビリティ** | △ | ○ | ○ | ◎ |
| **SwiftUI親和性** | △ | △ | ◎ | ◎ |
| **副作用の管理** | 暗黙的 | 暗黙的 | 部分的 | 明示的 |
| **適正規模** | 小 | 小〜中 | 中〜大 | 大 |
| **ボイラープレート** | 少ない | 中程度 | 中程度 | 多い |

---

## 選択フローチャート

```
SwiftUIを使っている？
├── Yes → チームがTCAを学ぶ余裕がある？
│         ├── Yes → 副作用が複雑？ → Yes → TCA
│         │                        → No  → MVVM
│         └── No  → MVVM
└── No  → テストを重視する？
          ├── Yes → MVP
          └── No  → MVC
```

---

## まとめ

- **MVC**：Appleの標準。小規模・プロトタイプに最適
- **MVP**：UIKitでテスタビリティを上げたい場合の選択肢
- **MVVM**：SwiftUIとの相性◎。中級者が最初に習得すべきパターン
- **TCA**：副作用の管理と一貫性を重視する大規模開発向け

次回は **SwiftUI + MVVMの実践設計**に深く入り込みます。ViewModelの責務をどこまで持たせるべきか、具体的なコードで解説します。

---

## 連載インデックス

- **第1回**：アーキテクチャ選択ガイド（本記事）
- 第2回：SwiftUIとMVVMの実践設計
- 第3回：依存性注入（DI）でテスタブルなアプリを作る
- 第4回：TCA入門：副作用の管理
- 第5回：モジュール分割でスケールするコードベース
- 第6回：SwiftDataとClean Architectureを組み合わせる
- 第7回：チームで守れるアーキテクチャ指針の作り方
