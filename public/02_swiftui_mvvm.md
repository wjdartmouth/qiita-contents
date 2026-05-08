---
title: "SwiftUIとMVVMの実践：ViewModelの責務を正しく設計する【実践シリーズ #2】"
tags:
  - iOS
  - Swift
  - SwiftUI
  - MVVM
  - architecture
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# SwiftUIとMVVMの実践：ViewModelの責務を正しく設計する

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第2回 / 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

「ViewModelに何を書けばいいか分からない」

MVVMを導入したものの、気づけばViewModelが肥大化してしまった——この問題はとても一般的です。前回の概要を踏まえ、今回は**SwiftUIにおけるViewModelの責務を明確に定義**します。

---

## ViewModelの正しい責務とは

ViewModelが担うべき責務は以下の3つです。

1. **UIの状態（State）を保持する**
2. **ユーザーの意図（Intent）を受け取り処理する**
3. **ドメイン層・データ層の橋渡しをする**

逆に、以下はViewModelの責務**ではありません**。

- ネットワーク通信の実装（→ Service/Repositoryの責務）
- 画面遷移のロジック（→ Coordinatorの責務）
- ビジネスロジックの中核（→ UseCaseの責務）

---

## 実践：Todoアプリで設計を見る

シンプルなTodoアプリを題材に、適切な設計を段階的に組み上げます。

### Step 1：まず「悪い例」を見る

```swift
// ❌ 責務過多なViewModel
@Observable
class BadTodoViewModel {
    var todos: [Todo] = []

    func loadTodos() async {
        // ❌ 直接URLSessionを叩いている
        let (data, _) = try! await URLSession.shared.data(
            from: URL(string: "https://api.example.com/todos")!
        )
        todos = try! JSONDecoder().decode([Todo].self, from: data)

        // ❌ CoreDataへの直接アクセス
        let context = PersistenceController.shared.container.viewContext
        // ... 保存処理
    }

    // ❌ 画面遷移まで担当
    func didSelectTodo(_ todo: Todo) {
        NavigationManager.shared.push(.todoDetail(todo))
    }
}
```

### Step 2：責務を分離した構成

```
View
 └── ViewModel（UIの状態・Intentの受付）
       └── UseCase（ビジネスロジック）
             └── Repository（データアクセス）
                   └── APIClient / LocalStore
```

### Step 3：各層を実装する

**Modelの定義**

```swift
struct Todo: Identifiable, Equatable {
    let id: UUID
    var title: String
    var isCompleted: Bool
}
```

**Repositoryプロトコル**（データアクセスを抽象化）

```swift
protocol TodoRepositoryProtocol {
    func fetchAll() async throws -> [Todo]
    func save(_ todo: Todo) async throws
    func delete(id: UUID) async throws
}

// 本番実装
final class TodoRepository: TodoRepositoryProtocol {
    private let apiClient: APIClient
    private let localStore: LocalStore

    init(apiClient: APIClient = .shared, localStore: LocalStore = .shared) {
        self.apiClient = apiClient
        self.localStore = localStore
    }

    func fetchAll() async throws -> [Todo] {
        try await apiClient.get("/todos")
    }

    func save(_ todo: Todo) async throws {
        try await apiClient.post("/todos", body: todo)
        try await localStore.save(todo)
    }

    func delete(id: UUID) async throws {
        try await apiClient.delete("/todos/\(id)")
        try await localStore.delete(id: id)
    }
}
```

**UseCase**（ビジネスロジックを集約）

```swift
protocol FetchTodosUseCaseProtocol {
    func execute() async throws -> [Todo]
}

final class FetchTodosUseCase: FetchTodosUseCaseProtocol {
    private let repository: TodoRepositoryProtocol

    init(repository: TodoRepositoryProtocol) {
        self.repository = repository
    }

    func execute() async throws -> [Todo] {
        let todos = try await repository.fetchAll()
        // ビジネスロジック：未完了を先頭に並べる
        return todos.sorted { !$0.isCompleted && $1.isCompleted }
    }
}
```

**ViewModel**（薄く、UIに集中）

```swift
@Observable
final class TodoListViewModel {

    // MARK: - UI State
    var todos: [Todo] = []
    var isLoading: Bool = false
    var errorMessage: String?
    var searchText: String = ""

    var filteredTodos: [Todo] {
        guard !searchText.isEmpty else { return todos }
        return todos.filter { $0.title.contains(searchText) }
    }

    // MARK: - Dependencies
    private let fetchUseCase: FetchTodosUseCaseProtocol
    private let deleteUseCase: DeleteTodoUseCaseProtocol

    init(
        fetchUseCase: FetchTodosUseCaseProtocol = FetchTodosUseCase(repository: TodoRepository()),
        deleteUseCase: DeleteTodoUseCaseProtocol = DeleteTodoUseCase(repository: TodoRepository())
    ) {
        self.fetchUseCase = fetchUseCase
        self.deleteUseCase = deleteUseCase
    }

    // MARK: - Intents（ユーザーのアクションを受け付ける）
    func onAppear() async {
        await loadTodos()
    }

    func onDelete(offsets: IndexSet) async {
        let idsToDelete = offsets.map { filteredTodos[$0].id }
        for id in idsToDelete {
            await deleteTodo(id: id)
        }
    }

    // MARK: - Private
    private func loadTodos() async {
        isLoading = true
        defer { isLoading = false }
        do {
            todos = try await fetchUseCase.execute()
        } catch {
            errorMessage = "読み込みに失敗しました: \(error.localizedDescription)"
        }
    }

    private func deleteTodo(id: UUID) async {
        do {
            try await deleteUseCase.execute(id: id)
            todos.removeAll { $0.id == id }
        } catch {
            errorMessage = "削除に失敗しました"
        }
    }
}
```

**View**（状態を表示するだけ）

```swift
struct TodoListView: View {
    @State private var viewModel = TodoListViewModel()

    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Todo")
                .searchable(text: $viewModel.searchText)
                .task { await viewModel.onAppear() }
                .alert("エラー", isPresented: .constant(viewModel.errorMessage != nil)) {
                    Button("OK") { viewModel.errorMessage = nil }
                } message: {
                    Text(viewModel.errorMessage ?? "")
                }
        }
    }

    @ViewBuilder
    private var content: some View {
        if viewModel.isLoading {
            ProgressView("読み込み中...")
        } else {
            List {
                ForEach(viewModel.filteredTodos) { todo in
                    TodoRowView(todo: todo)
                }
                .onDelete { offsets in
                    Task { await viewModel.onDelete(offsets: offsets) }
                }
            }
        }
    }
}
```

---

## @Observableと@StateObjectの使い分け

iOS 17以降は `@Observable` マクロを使うのが基本です。

```swift
// iOS 17以降 ✅
@Observable
final class MyViewModel { ... }

struct MyView: View {
    @State private var viewModel = MyViewModel()  // 所有する場合
    // または
    var viewModel: MyViewModel  // 親から注入する場合
}

// iOS 16以前 ✅
final class MyViewModel: ObservableObject {
    @Published var items: [Item] = []
}

struct MyView: View {
    @StateObject private var viewModel = MyViewModel()  // 所有する場合
    @ObservedObject var viewModel: MyViewModel          // 注入する場合
}
```

---

## ViewModelのテストを書く

依存をプロトコルで抽象化したので、テストが書きやすくなっています。

```swift
final class TodoListViewModelTests: XCTestCase {

    func test_onAppear_成功時にtodosが更新される() async {
        // Arrange
        let mockUseCase = MockFetchTodosUseCase()
        mockUseCase.stubbedResult = [
            Todo(id: UUID(), title: "テストTodo", isCompleted: false)
        ]
        let viewModel = TodoListViewModel(fetchUseCase: mockUseCase)

        // Act
        await viewModel.onAppear()

        // Assert
        XCTAssertEqual(viewModel.todos.count, 1)
        XCTAssertEqual(viewModel.todos.first?.title, "テストTodo")
        XCTAssertFalse(viewModel.isLoading)
    }

    func test_onAppear_失敗時にerrorMessageが設定される() async {
        // Arrange
        let mockUseCase = MockFetchTodosUseCase()
        mockUseCase.shouldThrow = true
        let viewModel = TodoListViewModel(fetchUseCase: mockUseCase)

        // Act
        await viewModel.onAppear()

        // Assert
        XCTAssertNotNil(viewModel.errorMessage)
        XCTAssertTrue(viewModel.todos.isEmpty)
    }
}

// テスト用Mock
final class MockFetchTodosUseCase: FetchTodosUseCaseProtocol {
    var stubbedResult: [Todo] = []
    var shouldThrow = false

    func execute() async throws -> [Todo] {
        if shouldThrow { throw URLError(.badServerResponse) }
        return stubbedResult
    }
}
```

---

## まとめ

| 層 | 責務 | SwiftUIでの具体例 |
|---|---|---|
| **View** | 状態の表示・ユーザー入力の転送 | `SwiftUI.View` |
| **ViewModel** | UI状態の保持・Intentの処理 | `@Observable class` |
| **UseCase** | ビジネスロジック | `protocol + final class` |
| **Repository** | データアクセスの抽象化 | `protocol + final class` |

ViewModelを「薄く」保つことで、テストが書きやすくなり、将来のアーキテクチャ変更にも耐えられる設計になります。

次回は、この設計をさらに強固にする**依存性注入（DI）の実践**を解説します。

---

## 連載インデックス

- 第1回：アーキテクチャ選択ガイド
- **第2回**：SwiftUIとMVVMの実践（本記事）
- 第3回：依存性注入（DI）でテスタブルなアプリを作る
- 第4回：TCA入門：副作用の管理
- 第5回：モジュール分割でスケールするコードベース
- 第6回：SwiftDataとClean Architectureを組み合わせる
- 第7回：チームで守れるアーキテクチャ指針の作り方
