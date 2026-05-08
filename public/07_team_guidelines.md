---
title: "実践まとめ：チームで守れるiOSアーキテクチャ指針の作り方【実践シリーズ #7】"
tags:
  - iOS
  - Swift
  - architecture
  - チーム開発
  - ベストプラクティス
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# 実践まとめ：チームで守れるiOSアーキテクチャ指針の作り方

> **連載：iOS実践アーキテクチャ設計シリーズ**
> 第7回（最終回）/ 全7回 ｜ 対象：Swift経験1〜3年の中級エンジニア

---

## はじめに

この連載では、MVC/MVVM/TCA・依存性注入・モジュール分割・SwiftDataと、iOSアーキテクチャの各論を扱ってきました。しかし技術的な知識があっても、それをチームで継続的に守れなければ意味がありません。

最終回は「**アーキテクチャをチームの文化として定着させる**」方法を実践的にまとめます。

---

## なぜアーキテクチャは崩れるのか

アーキテクチャが崩れる原因は、技術的な問題よりもプロセスの問題がほとんどです。

```
崩れる主な原因
├── ルールが文書化されていない（口頭伝承のみ）
├── 新メンバーへの共有が不十分
├── レビューで指摘されない（属人化）
├── 締め切り圧力でショートカットが許容される
└── 「なぜそうするか」の理由が共有されていない
```

---

## ステップ1：ADR（アーキテクチャ決定記録）を書く

ADR（Architecture Decision Record）は、「なぜその決定をしたか」を記録するドキュメントです。

```markdown
# ADR-001: MVVM + UseCaseパターンを採用する

## ステータス
採用済み（2024-09-01）

## コンテキスト
SwiftUIへの移行に伴い、アーキテクチャを統一する必要が生じた。
現状はMVC・MVVM・独自パターンが混在しており、
新規参加者のオンボーディングに時間がかかっている。

## 決定
MVVM + UseCaseパターンを全画面の標準とする。
- View：状態の表示とユーザー入力の転送のみ
- ViewModel：UIの状態保持とUseCaseの呼び出し
- UseCase：ビジネスロジックのカプセル化
- Repository：データアクセスの抽象化

## 理由
- TCAはチームの学習コストが高い（チーム平均経験1.5年）
- SwiftUIとの親和性が高い
- テストが書きやすい
- 業界標準に近く、採用・オンボーディングが容易

## 結果
- 新規画面はすべてこのパターンで実装する
- 既存のMVC画面は改修時に順次移行する
- TCAへの移行は将来の選択肢として保留

## 代替案として検討したもの
- TCA：パワフルだが学習コストが高い → 将来検討
- MVC継続：テスタビリティが低く却下
```

ADRのポイントは「何を決めたか」だけでなく「なぜ他の選択肢を選ばなかったか」まで書くことです。

---

## ステップ2：ARCHITECTURE.mdをリポジトリに置く

コードと同じリポジトリに、アーキテクチャガイドを置きましょう。

```markdown
# ARCHITECTURE.md

## ディレクトリ構成

\`\`\`
Sources/
├── Features/          # 機能ごとのモジュール
│   └── FeatureName/
│       ├── View/
│       ├── ViewModel/
│       ├── UseCase/
│       └── Tests/
├── Core/              # 共有コンポーネント
│   ├── DesignSystem/
│   ├── NetworkClient/
│   └── Extensions/
└── Domain/            # ドメインモデル・プロトコル
    ├── Models/
    └── Repositories/
\`\`\`

## 依存ルール（破ってはいけない）

1. View は ViewModel のみ参照する
2. ViewModel は UseCase のみ参照する（Repository は直接呼ばない）
3. UseCase は Repository プロトコルのみ参照する（具体実装は知らない）
4. Domain は外部ライブラリに依存しない

## 命名規則

| 種別 | 規則 | 例 |
|---|---|---|
| ViewModel | `{画面名}ViewModel` | `TodoListViewModel` |
| UseCase | `{動詞}{対象}UseCase` | `FetchTodosUseCase` |
| Repository実装 | `{データソース}{対象}Repository` | `SwiftDataTodoRepository` |
| Protocol | 末尾に `Protocol` | `TodoRepositoryProtocol` |
| Mock | 先頭に `Mock` | `MockTodoRepository` |

## ファイル作成チェックリスト

新しい画面を作るとき：
- [ ] `{名前}View.swift`
- [ ] `{名前}ViewModel.swift`（`@Observable` を使用）
- [ ] `{名前}ViewModelTests.swift`（最低3テストケース）
- [ ] 必要なUseCaseとそのテスト
```

---

## ステップ3：SwiftLintでアーキテクチャルールを自動チェック

コードレビューで毎回指摘するのは疲れます。自動化できるルールはSwiftLintに任せましょう。

```yaml
# .swiftlint.yml

# 基本ルール
line_length:
  warning: 120
  error: 160

type_body_length:
  warning: 200
  error: 300

# カスタムルール
custom_rules:
  # ViewModelでRepositoryを直接使用することを禁止
  view_model_no_repository:
    name: "ViewModel should not use Repository directly"
    message: "ViewModelからRepositoryを直接使用しないでください。UseCaseを経由してください。"
    regex: 'class \w+ViewModel[^{]*\{[^}]*\w+Repository(?!Protocol)'
    severity: warning

  # Viewにビジネスロジックを書かない
  view_no_business_logic:
    name: "View should not contain business logic"
    message: "ViewにURLSessionやModelContextを直接使用しないでください。"
    regex: 'struct \w+View[^{]*\{[^}]*(URLSession|ModelContext)'
    severity: error

  # プロトコル命名チェック
  protocol_naming:
    name: "Protocol naming convention"
    message: "プロトコル名は末尾に'Protocol'をつけてください（DelegateとDataSourceを除く）。"
    regex: 'protocol (?!.*Protocol)(?!.*Delegate)(?!.*DataSource)\w+\s*\{'
    severity: warning
```

---

## ステップ4：PRテンプレートでアーキテクチャを意識させる

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## 変更内容
<!-- 何を変更したかを簡潔に -->

## アーキテクチャチェックリスト

- [ ] Viewはビジネスロジックを持っていない
- [ ] ViewModelはRepository/APIを直接呼んでいない
- [ ] 新しいUseCaseにはユニットテストがある
- [ ] 依存はプロトコルで抽象化されている
- [ ] 新しい設計上の決定はADRに記録した（必要な場合）

## テスト

- [ ] 新規コードにテストを追加した
- [ ] 既存テストがすべてパスする
- [ ] `xcodebuild test` でローカル確認済み

## スクリーンショット（UI変更がある場合）
| Before | After |
|---|---|
| | |
```

---

## ステップ5：アーキテクチャレビューの運用

週次や隔週で「アーキテクチャレビュー」の時間を設けると、ルールの形骸化を防げます。

```
アーキテクチャレビューのアジェンダ（30分）

1. 先週マージされたPRで気になった点（10分）
   - アーキテクチャ上の決断について議論
   - Better waysの共有

2. 困っていること・疑問（10分）
   - 「この場合はどのパターンを使うべきか」
   - 持ち回りで質問者を決める

3. ADRの更新・新規作成（10分）
   - 決定したことを文書化
```

---

## 連載を通じて作ったもの：全体像の整理

7回の連載で扱った内容を1枚の図にまとめます。

```
┌──────────────────────────────────────────────────────────┐
│  SwiftUI View                                            │
│   @State ViewModel ← @Observable                        │
│         │ send(Intent)  ↑ State変化                      │
├──────────┼───────────────────────────────────────────────┤
│  ViewModel（第2回）                                       │
│   ・UI状態を保持                                          │
│   ・UseCaseを呼ぶ                                         │
│   ・依存はDIで注入（第3回）                               │
├──────────┼───────────────────────────────────────────────┤
│  UseCase（第2・6回）                                      │
│   ・ビジネスロジックを持つ                                │
│   ・Repositoryプロトコルに依存                            │
├──────────┼───────────────────────────────────────────────┤
│  Repository Protocol（第3回）                            │
│         │                                                │
│   ┌─────┴──────┐                                         │
│   ↓            ↓                                         │
│  APIClient  SwiftData（第6回）                           │
│   実装         実装                                       │
└──────────────────────────────────────────────────────────┘

横断的関心事
・アーキテクチャ選択（第1回）
・TCAという選択肢（第4回）
・モジュール分割（第5回）
・チーム運用（第7回・本記事）
```

---

## まとめ：アーキテクチャは「合意のドキュメント」

アーキテクチャとは、チームメンバー全員が守ることに合意した「コードの書き方のルール」です。どれだけ優れた設計をしても、チームが知らなければ意味がありません。

この連載で伝えたかった本質は一つです。

**「コードに現れない意図を、コードの外で丁寧に伝え続けること」**

ADRを書く、ARCHITECTURE.mdを更新する、PRテンプレートを整備する——これらは地味な作業ですが、チームの設計品質を長期的に守る最も重要な投資です。

---

## 連載インデックス（完結）

| # | タイトル | キーワード |
|---|---|---|
| 1 | アーキテクチャ選択ガイド | MVC / MVP / MVVM / TCA |
| 2 | SwiftUIとMVVMの実践 | @Observable / UseCase / Repository |
| 3 | 依存性注入（DI） | Protocol / DIコンテナ / Dependencies |
| 4 | TCA入門：副作用の管理 | Reducer / Effect / TestStore |
| 5 | モジュール分割 | SPM / Coordinator / アクセス修飾子 |
| 6 | SwiftDataとClean Architecture | @Model / Data層の分離 |
| **7** | **チームで守れる指針の作り方** | **ADR / SwiftLint / PRテンプレート** |

---

7回お読みいただきありがとうございました。この連載がiOSアーキテクチャ設計の実践に役立てば幸いです。

質問・フィードバックはコメントでお気軽にどうぞ。
