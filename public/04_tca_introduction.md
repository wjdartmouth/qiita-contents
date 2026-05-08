---
title: "The Composable Architecture（TCA）入門：副作用の管理【実践シリーズ #4】"
tags:
  - iOS
  - Swift
  - TCA
  - ComposableArchitecture
  - architecture
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# The Composable Architecture（TCA）入門：副作用の管理

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第4回 / 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

MVVMは直感的で使いやすい反面、副作用（API通信・タイマー・通知など）の管理が属人化しやすいという課題があります。**TCA（The Composable Architecture）**はこの問題を、**すべての副作用を明示的に宣言する**というアプローチで解決します。

今回はTCAの核心概念と、副作用管理の実践を解説します。

---

## TCAの5つのコアコンセプト

```
┌─────────────────────────────────────────┐
│  View                                   │
│   │ send(action)      ↑ state変化を検知  │
│   ↓                   │                 │
│  Store ──────────→ Reducer              │
│          action        │ Effect を返す   │
│                        ↓                │
│                    Effect（副作用）      │
└─────────────────────────────────────────┘
```

| コンセプト | 役割 |
|---|---|
| **State** | 画面の状態をすべて表す値型 |
| **Action** | ユーザー操作・副作用の結果などのイベント |
| **Reducer** | (State, Action) → (State, Effect) の純粋関数 |
| **Effect** | 副作用の宣言（通信・タイマー等） |
| **Store** | StateとReducerを保持し、ViewにBindingを提供 |

---

## セットアップ

```swift
// Package.swift
dependencies: [
    .package(
        url: "https://github.com/pointfreeco/swift-composable-architecture",
        from: "1.10.0"
    )
]
```

---

## 実践：カウンターで基礎を掴む

まずシンプルなカウンターで全体の流れを理解します。

```swift
import ComposableArchitecture

@Reducer
struct CounterFeature {

    // 1. State：画面の状態
    @ObservableState
    struct State: Equatable {
        var count: Int = 0
        var isLoading: Bool = false
        var fact: String?
    }

    // 2. Action：起きうるイベントをすべて列挙
    enum Action {
        case incrementTapped
        case decrementTapped
        case factButtonTapped
        case factResponse(Result<String, Error>)
    }

    // 3. 副作用の依存（テスト可能にするため分離）
    @Dependency(\.numberFact) var numberFact

    // 4. Reducer：状態遷移と副作用の宣言
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {

            case .incrementTapped:
                state.count += 1
                return .none  // 副作用なし

            case .decrementTapped:
                state.count = max(0, state.count - 1)
                return .none

            case .factButtonTapped:
                state.isLoading = true
                state.fact = nil
                // 副作用を宣言（実行ではなく宣言）
                return .run { [count = state.count] send in
                    await send(.factResponse(
                        Result { try await numberFact(count) }
                    ))
                }

            case .factResponse(.success(let fact)):
                state.isLoading = false
                state.fact = fact
                return .none

            case .factResponse(.failure):
                state.isLoading = false
                state.fact = "取得に失敗しました"
                return .none
            }
        }
    }
}
```

```swift
// View
struct CounterView: View {
    let store: StoreOf<CounterFeature>

    var body: some View {
        VStack(spacing: 20) {
            Text("\(store.count)")
                .font(.largeTitle)

            HStack {
                Button("-") { store.send(.decrementTapped) }
                Button("+") { store.send(.incrementTapped) }
            }

            if store.isLoading {
                ProgressView()
            } else if let fact = store.fact {
                Text(fact).multilineTextAlignment(.center)
            }

            Button("豆知識を取得") {
                store.send(.factButtonTapped)
            }
        }
        .padding()
    }
}

// エントリーポイント
CounterView(
    store: Store(initialState: CounterFeature.State()) {
        CounterFeature()
    }
)
```

---

## 副作用管理の真価：キャンセルとデバウンス

TCAの副作用管理が輝く場面がキャンセルとデバウンスです。

```swift
@Reducer
struct SearchFeature {
    @ObservableState
    struct State: Equatable {
        var query: String = ""
        var results: [SearchResult] = []
        var isSearching: Bool = false
    }

    enum Action {
        case queryChanged(String)
        case searchResults(Result<[SearchResult], Error>)
        case cancelSearch
    }

    // キャンセルのIDを定義
    enum CancelID { case search }

    @Dependency(\.searchClient) var searchClient
    @Dependency(\.continuousClock) var clock

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {

            case .queryChanged(let query):
                state.query = query

                guard !query.isEmpty else {
                    state.results = []
                    return .cancel(id: CancelID.search)  // 前の検索をキャンセル
                }

                state.isSearching = true
                return .run { send in
                    // 300msデバウンス
                    try await clock.sleep(for: .milliseconds(300))
                    await send(.searchResults(
                        Result { try await searchClient.search(query) }
                    ))
                }
                .cancellable(id: CancelID.search, cancelInFlight: true)
                // ↑ 新しいアクションが来たら自動でキャンセル

            case .searchResults(.success(let results)):
                state.isSearching = false
                state.results = results
                return .none

            case .searchResults(.failure):
                state.isSearching = false
                return .none

            case .cancelSearch:
                state.isSearching = false
                return .cancel(id: CancelID.search)
            }
        }
    }
}
```

MVVMでデバウンス＋キャンセルを実装しようとすると `Task` の管理が複雑になりますが、TCAではこれが宣言的に書けます。

---

## Reducerのテスト

TCAの最大の強みのひとつが**TestStore**による詳細なテストです。

```swift
@MainActor
final class CounterFeatureTests: XCTestCase {

    func test_increment_countが増える() async {
        let store = TestStore(initialState: CounterFeature.State()) {
            CounterFeature()
        }

        // アクションを送信し、状態変化を検証
        await store.send(.incrementTapped) {
            $0.count = 1  // 期待される状態変化を記述
        }
    }

    func test_factButtonTapped_ローディング状態になりfactが取得できる() async {
        let store = TestStore(initialState: CounterFeature.State(count: 42)) {
            CounterFeature()
        } withDependencies: {
            $0.numberFact = { number in "\(number) は素数です" }
        }

        await store.send(.factButtonTapped) {
            $0.isLoading = true
        }

        // Effectの結果を待つ
        await store.receive(\.factResponse.success) {
            $0.isLoading = false
            $0.fact = "42 は素数です"
        }
    }

    func test_検索デバウンス_300ms以内の入力はキャンセルされる() async {
        let clock = TestClock()
        let store = TestStore(initialState: SearchFeature.State()) {
            SearchFeature()
        } withDependencies: {
            $0.continuousClock = clock
        }

        await store.send(.queryChanged("S")) {
            $0.query = "S"
            $0.isSearching = true
        }

        // 100msしか経過させない → まだ検索されない
        await clock.advance(by: .milliseconds(100))

        await store.send(.queryChanged("Sw")) {
            $0.query = "Sw"
        }

        // 300ms経過させる
        await clock.advance(by: .milliseconds(300))

        // "S"の検索はキャンセルされ、"Sw"の結果だけ来る
        await store.receive(\.searchResults)
    }
}
```

---

## MVVMからTCAへの移行を判断する基準

TCAはパワフルですが、学習コストが高く、すべてのプロジェクトに適しているわけではありません。

**TCAを選ぶべき状況：**
- 副作用（API・WebSocket・タイマー・通知）が複雑に絡み合っている
- チームで状態遷移のルールを厳密に守りたい
- テストカバレッジを高く保ちたい
- 大規模チームでコードの一貫性が必要

**MVVMで十分な状況：**
- 画面数が少ない（10画面以下）
- チームのSwift経験が浅い
- 副作用がシンプル（単純なAPI通信のみ）

---

## まとめ

TCAの核心は「**副作用を値として扱う**」ことです。

```
副作用をEffect型で宣言する
→ Reducerが純粋関数になる
→ テストが確定的になる
→ デバッグが容易になる
```

次回は**モジュール分割**を扱います。アプリが成長してきたとき、どうコードベースを分割してビルド時間を短縮し、チーム開発をスムーズにするかを解説します。

---

## 連載インデックス

- 第1回：アーキテクチャ選択ガイド
- 第2回：SwiftUIとMVVMの実践
- 第3回：依存性注入（DI）
- **第4回**：TCA入門：副作用の管理（本記事）
- 第5回：モジュール分割でスケールするコードベース
- 第6回：SwiftDataとClean Architectureを組み合わせる
- 第7回：チームで守れるアーキテクチャ指針の作り方
