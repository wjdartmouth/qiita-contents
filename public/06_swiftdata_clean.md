---
title: "SwiftDataとClean Architectureを組み合わせる実践パターン【実践シリーズ #6】"
tags:
  - iOS
  - Swift
  - SwiftData
  - CleanArchitecture
  - architecture
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# SwiftDataとClean Architectureを組み合わせる実践パターン

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第6回 / 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

SwiftDataはiOS 17で導入されたApple製のO/Rマッパーです。`@Model` マクロで直感的に永続化が書けますが、**アーキテクチャを考えずに使い始めると問題が起きます。**

SwiftDataの `@Model` クラスをそのままViewModelやUseCaseで使い回すと、ドメイン層とデータ層の境界が消えてしまいます。今回はClean Architectureの考え方でSwiftDataを正しく扱う方法を解説します。

---

## SwiftDataをナイーブに使うとどうなるか

```swift
// ❌ アーキテクチャを考えない使い方
@Model
final class TodoItem {
    var id: UUID
    var title: String
    var isCompleted: Bool
    var createdAt: Date

    init(title: String) {
        self.id = UUID()
        self.title = title
        self.isCompleted = false
        self.createdAt = Date()
    }
}

@Observable
final class TodoViewModel {
    @Query var todos: [TodoItem]  // SwiftDataのModelをViewで直接使用
    var modelContext: ModelContext

    func add(title: String) {
        let item = TodoItem(title: title)
        modelContext.insert(item)
    }
}
```

これの問題点：

- `@Model` クラスがUIの状態管理にも、ビジネスロジックにも混入する
- テスト時にSwiftDataのコンテキストが必要になる
- ModelクラスをUIとドメインで共有するため、変更の影響範囲が広い

---

## Clean Architectureで層を分ける

```
┌─────────────────────────────────┐
│  Presentation層                 │
│  ViewModel ← SwiftUI View       │
├─────────────────────────────────┤
│  Domain層                       │
│  UseCase + ドメインモデル（値型）  │
├─────────────────────────────────┤
│  Data層                         │
│  Repository実装 + SwiftData Model│
└─────────────────────────────────┘
```

**ポイント：SwiftDataの `@Model` クラスをData層に閉じ込める**

---

## 実装：Todoアプリで設計する

### Domain層：純粋な値型のエンティティ

```swift
// Domain/Models/Todo.swift
// SwiftDataに依存しない、純粋なSwift構造体
struct Todo: Identifiable, Equatable, Sendable {
    let id: UUID
    var title: String
    var isCompleted: Bool
    let createdAt: Date

    init(id: UUID = UUID(), title: String, isCompleted: Bool = false, createdAt: Date = Date()) {
        self.id = id
        self.title = title
        self.isCompleted = isCompleted
        self.createdAt = createdAt
    }
}
```

### Domain層：Repositoryプロトコル

```swift
// Domain/Repositories/TodoRepositoryProtocol.swift
protocol TodoRepositoryProtocol: Sendable {
    func fetchAll() async throws -> [Todo]
    func fetch(id: UUID) async throws -> Todo?
    func save(_ todo: Todo) async throws
    func delete(id: UUID) async throws
    func update(_ todo: Todo) async throws
}
```

### Data層：SwiftDataのModelクラス

```swift
// Data/SwiftData/TodoDataModel.swift
import SwiftData

// @Model はData層にのみ存在する
@Model
final class TodoDataModel {
    @Attribute(.unique) var id: UUID
    var title: String
    var isCompleted: Bool
    var createdAt: Date

    init(from todo: Todo) {
        self.id = todo.id
        self.title = todo.title
        self.isCompleted = todo.isCompleted
        self.createdAt = todo.createdAt
    }

    // Data層 → Domain層への変換
    func toDomain() -> Todo {
        Todo(
            id: id,
            title: title,
            isCompleted: isCompleted,
            createdAt: createdAt
        )
    }
}
```

### Data層：Repository実装

```swift
// Data/Repositories/SwiftDataTodoRepository.swift
import SwiftData

final class SwiftDataTodoRepository: TodoRepositoryProtocol {
    private let modelContainer: ModelContainer

    init(modelContainer: ModelContainer) {
        self.modelContainer = modelContainer
    }

    func fetchAll() async throws -> [Todo] {
        let context = ModelContext(modelContainer)
        let descriptor = FetchDescriptor<TodoDataModel>(
            sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
        )
        let dataModels = try context.fetch(descriptor)
        return dataModels.map { $0.toDomain() }
    }

    func fetch(id: UUID) async throws -> Todo? {
        let context = ModelContext(modelContainer)
        let predicate = #Predicate<TodoDataModel> { $0.id == id }
        let descriptor = FetchDescriptor(predicate: predicate)
        return try context.fetch(descriptor).first?.toDomain()
    }

    func save(_ todo: Todo) async throws {
        let context = ModelContext(modelContainer)
        let dataModel = TodoDataModel(from: todo)
        context.insert(dataModel)
        try context.save()
    }

    func update(_ todo: Todo) async throws {
        let context = ModelContext(modelContainer)
        let predicate = #Predicate<TodoDataModel> { $0.id == todo.id }
        let descriptor = FetchDescriptor(predicate: predicate)

        guard let dataModel = try context.fetch(descriptor).first else {
            throw RepositoryError.notFound
        }

        dataModel.title = todo.title
        dataModel.isCompleted = todo.isCompleted
        try context.save()
    }

    func delete(id: UUID) async throws {
        let context = ModelContext(modelContainer)
        let predicate = #Predicate<TodoDataModel> { $0.id == id }
        let descriptor = FetchDescriptor(predicate: predicate)

        guard let dataModel = try context.fetch(descriptor).first else { return }
        context.delete(dataModel)
        try context.save()
    }
}

enum RepositoryError: Error {
    case notFound
    case saveFailed(Error)
}
```

### Domain層：UseCase

```swift
// Domain/UseCases/TodoUseCases.swift

// 一覧取得
final class FetchTodosUseCase {
    private let repository: TodoRepositoryProtocol

    init(repository: TodoRepositoryProtocol) {
        self.repository = repository
    }

    func execute() async throws -> [Todo] {
        let todos = try await repository.fetchAll()
        // ビジネスルール：未完了を上に並べる
        return todos.sorted { !$0.isCompleted && $1.isCompleted }
    }
}

// Todo追加
final class AddTodoUseCase {
    private let repository: TodoRepositoryProtocol

    init(repository: TodoRepositoryProtocol) {
        self.repository = repository
    }

    func execute(title: String) async throws {
        // バリデーション（ビジネスロジック）
        let trimmed = title.trimmingCharacters(in: .whitespaces)
        guard !trimmed.isEmpty else {
            throw TodoError.emptyTitle
        }
        guard trimmed.count <= 100 else {
            throw TodoError.titleTooLong
        }

        let todo = Todo(title: trimmed)
        try await repository.save(todo)
    }
}

enum TodoError: LocalizedError {
    case emptyTitle
    case titleTooLong

    var errorDescription: String? {
        switch self {
        case .emptyTitle: return "タイトルを入力してください"
        case .titleTooLong: return "タイトルは100文字以内で入力してください"
        }
    }
}
```

### Presentation層：ViewModel

```swift
// Presentation/ViewModels/TodoListViewModel.swift
@Observable
final class TodoListViewModel {
    var todos: [Todo] = []
    var isLoading = false
    var errorMessage: String?
    var newTodoTitle = ""

    private let fetchUseCase: FetchTodosUseCase
    private let addUseCase: AddTodoUseCase
    private let deleteUseCase: DeleteTodoUseCase

    init(
        fetchUseCase: FetchTodosUseCase,
        addUseCase: AddTodoUseCase,
        deleteUseCase: DeleteTodoUseCase
    ) {
        self.fetchUseCase = fetchUseCase
        self.addUseCase = addUseCase
        self.deleteUseCase = deleteUseCase
    }

    func onAppear() async {
        await refresh()
    }

    func addTodo() async {
        do {
            try await addUseCase.execute(title: newTodoTitle)
            newTodoTitle = ""
            await refresh()
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func deleteTodo(at offsets: IndexSet) async {
        for index in offsets {
            let todo = todos[index]
            do {
                try await deleteUseCase.execute(id: todo.id)
            } catch {
                errorMessage = "削除に失敗しました"
            }
        }
        await refresh()
    }

    private func refresh() async {
        isLoading = true
        defer { isLoading = false }
        do {
            todos = try await fetchUseCase.execute()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

---

## テストでIn-Memoryストアを使う

SwiftDataはIn-Memoryコンテナを使うことで、テストが簡単に書けます。

```swift
final class SwiftDataTodoRepositoryTests: XCTestCase {

    var container: ModelContainer!
    var repository: SwiftDataTodoRepository!

    override func setUp() async throws {
        // テスト用のIn-Memoryコンテナ
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        container = try ModelContainer(for: TodoDataModel.self, configurations: config)
        repository = SwiftDataTodoRepository(modelContainer: container)
    }

    func test_save_fetchAll_正常に保存と取得ができる() async throws {
        // Arrange
        let todo = Todo(title: "テストTodo")

        // Act
        try await repository.save(todo)
        let fetched = try await repository.fetchAll()

        // Assert
        XCTAssertEqual(fetched.count, 1)
        XCTAssertEqual(fetched.first?.title, "テストTodo")
        XCTAssertFalse(fetched.first?.isCompleted ?? true)
    }

    func test_delete_保存したTodoを削除できる() async throws {
        let todo = Todo(title: "削除テスト")
        try await repository.save(todo)

        try await repository.delete(id: todo.id)
        let fetched = try await repository.fetchAll()

        XCTAssertTrue(fetched.isEmpty)
    }
}
```

---

## まとめ

| 層 | 使うもの | 注意点 |
|---|---|---|
| **Domain** | `struct`（値型） | SwiftDataを一切インポートしない |
| **Data** | `@Model class` | Domain層のエンティティに変換する責務を持つ |
| **Presentation** | `@Observable class` | `@Model` を直接参照しない |

SwiftDataは強力なツールですが、その `@Model` クラスをData層に閉じ込めることで、クリーンで変更に強いアーキテクチャを保てます。

次回は最終回、**チームで守れるアーキテクチャ指針の作り方**です。

---

## 連載インデックス

- 第1回：アーキテクチャ選択ガイド
- 第2回：SwiftUIとMVVMの実践
- 第3回：依存性注入（DI）
- 第4回：TCA入門：副作用の管理
- 第5回：モジュール分割
- **第6回**：SwiftDataとClean Architecture（本記事）
- 第7回：チームで守れるアーキテクチャ指針の作り方
