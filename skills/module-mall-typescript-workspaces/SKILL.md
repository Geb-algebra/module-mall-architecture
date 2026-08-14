---
name: module-mall-typescript-workspaces
description: Module Mall Architecture（MMA）のModuleをTypeScriptとnpm workspacesのmonorepoでworkspace packageとして実装するための、workspace package境界、公開契約、import、manifest、契約テスト、暗黙的依存に関する検証可能なコーディング規約を提供するContext Skill。対象プロジェクトの設計・実装・変更・レビュー・説明で使用する。TypeScript以外の言語、npm workspaces以外のworkspace方式、MMAの一般概念だけを扱う場合は対象外。
---

# Module Mall Architecture for TypeScript npm Workspaces

## 適用範囲

この規約は、TypeScriptとnpm workspacesのmonorepoで、MMAの各Moduleを一つのworkspace packageとして実装する場合に適用する。workspace packageを、責務、公開範囲、依存宣言、理解、変更の基本境界として扱う。workspace packageはデプロイ単位、実行プロセス、サービス、チームの単位と一致しなくてよい。

各規則は、設計、実装、変更、レビュー、テスト設計、説明で対象コードが準拠しているか判断するためのContextである。このContext自体は、それらの活動の手順、成果物、完了条件、報告形式を定めない。

TypeScript以外の言語、npm workspaces以外のworkspace方式、ESLintなど特定の境界検査器、特定のbuild toolやtest frameworkの選定は対象外とする。

## workspace packageの基本構造

workspaceルートの`package.json`は、対象workspace packageを`workspaces`へ列挙する。packageを`packages/`直下に平坦に配置する`packages/*`に加え、関連moduleをサブディレクトリでまとめる場合は、`packages/domain/*`のように対象階層を明示して列挙する。`packages/**/*`のような再帰的なglobは、moduleではない深いディレクトリや意図しない入れ子の`package.json`までworkspaceとして拾う可能性があるため、この規約では採用しない。ルート自体をnpmパッケージとして公開しない場合は、誤公開を防ぐため`private: true`にする。

```json
{
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*",
    "packages/domain/*",
    "packages/import/*",
    "packages/simulation/*"
  ]
}
```

`packages/`以下のディレクトリは、列挙された階層にある自身の`package.json`によってworkspace packageの単位になる。グループ化のためだけの中間ディレクトリはworkspace packageではなく、関連moduleをまとめる論理的な整理単位である。新しいmoduleグループを追加する場合は、その階層のglobを`workspaces`へ追加する。各workspace packageは少なくとも自身の`package.json`、責務を記した`README.md`、実装コード、公開契約テストを持つ。具体的なソース、出力、テストのディレクトリ名は固定しない。

## TW-01: 責務と変更理由をREADMEへ定義する

各workspace packageの`README.md`は、責務、非責務、変更される理由、変更されない理由の正本でなければならない。

### 不変条件

- 責務はファイルや現在の機能の一覧ではなく、workspace packageが所有するルールまたは判断として記述されている。
- 非責務から、隣接する関心事の所有者を誤認しない。
- 「何が変わるときに変わるか」と「何が変わっても変わらないか」を判別できる。
- workspace packageへ加えられる責務上の変更は、READMEに記載された変更理由のいずれかで説明できる。
- READMEは内部アルゴリズムへ立ち入らず、内部実装を読むより小さい情報量で要求との関係を判断できる。

見出し名は変更できるが、次の情報を欠かしてはならない。

```markdown
# @app/todo

## Responsibility
TODOのライフサイクルと、期限切れを含む状態判定を所有する。

## Non-responsibility
TODOの表示方法、通知時期、通知の配送方法は所有しない。

## Changes when
TODOの生成、更新、完了、期限切れに関するルールが変わるとき。

## Does not change when
TODOの表示方法や通知チャネルだけが変わるとき。
```

`utils`や`shared`のような名前だけで無関係な変更理由を受け入れるworkspace packageや、READMEの変更理由で説明できないコードの追加は違反である。

## TW-02: 公開APIをexportsで限定する

各workspace packageの`package.json#exports`は、外部workspace packageが利用してよいTypeScriptのentrypointを個別に列挙しなければならない。ルートentrypointと、用途の異なる公開契約を表すsubpath entrypointを明示し、それぞれを公開用の`.ts`ファイルへ直接向ける。

### 不変条件

- 外部利用が許可されたすべてのentrypointが`exports`へ列挙されている。
- `exports`の各targetが、`index.ts`や`events.ts`など公開契約を定義するTypeScriptソースを直接指している。ビルド成果物をworkspace内の公開契約の正本にしていない。
- 外部利用を意図しない内部ファイルは`exports`から到達できない。
- wildcardや内部ディレクトリの一括公開によって、内部構造そのものを公開契約にしていない。
- 公開用`.ts` entrypointから再exportされる要素は、維持を約束する関数、型、クラス、イベントなどに限定されている。
- 用途の異なるsubpathは、利用目的と維持する契約が説明できる単位で定義されている。

次の例では、workspace packageのルート契約を`src/index.ts`、イベント契約を`src/events.ts`が所有する。

```json
{
  "name": "@app/todo",
  "exports": {
    ".": "./src/index.ts",
    "./events": "./src/events.ts"
  }
}
```

```ts
// packages/todo/src/index.ts
export { createTodo } from "./create-todo";
export { completeTodo } from "./complete-todo";
export type { Todo } from "./todo";
```

`dist/index.js`などのビルド成果物だけを`exports`のtargetにする設定や、`"./*": "./src/*"`のように内部構造を一括公開する設定は、この規約への違反である。

## TW-03: workspace package境界を迂回するimportを禁止する

外部workspace packageは、依存先のworkspace package名と`exports`に列挙されたentrypointだけを通じてimportしなければならない。同一workspace package内の相対importはこの制約の対象外である。

### 不変条件

- workspace package名の後ろへ内部パスを付けるdeep importが存在しない。
- 相対パスまたは絶対パスで別workspace packageのファイルへ到達するfilesystem importが存在しない。
- TypeScriptの`paths`などのaliasで別workspace packageの内部ファイルへ到達するimportが存在しない。
- import先をmodule resolverで解決した結果、別workspace packageの内部ファイルへ到達しない。
- 別workspace packageから利用されるコードは、依存先の公開entrypointから到達できる。

```ts
// 準拠
import { TodoBecameOverdue } from "@app/todo/events";

// 違反: deep import
import { TodoBecameOverdue } from "@app/todo/src/internal/events.js";

// 違反: workspace package境界を越えるfilesystem import
import { TodoBecameOverdue } from "../../todo/src/internal/events.js";
```

import文字列の見た目だけを検査し、aliasやsymlinkで解決された実体を確認しない境界検査は、この不変条件を保証できない。

## TW-04: すべての公開契約を外部利用者としてテストする

公開APIの型と構造、入力条件、保証される振る舞い、エラー、副作用など、すべての公開契約に対応する自動テストがなければならない。公開契約テストは、外部workspace packageと同じ公開entrypointだけを利用する。

### 不変条件

- 公開契約として記述された正常系、境界条件、失敗条件、状態変化、副作用に対応するテストがある。
- 公開契約テストは内部ファイル、内部関数、内部データ構造を直接importしない。
- 公開されるTypeScript型には、受理される利用例と拒否される利用例を型検査するテストがある。
- 公開契約テストと内部実装テストを区別でき、公開契約の網羅状況を内部テストで代用していない。
- 公開契約を意味的に変更した場合、その変更を観測する契約テストも変化する。

次の記法はtest frameworkを指定するものではなく、外部利用者と同じimportだけで振る舞いを観測する例である。

```ts
import { createTodo } from "@app/todo";

test("指定したタイトルを持つ未完了のTODOを作成する", () => {
  const todo = createTodo({ title: "原稿を書く" });

  expect(todo.title).toBe("原稿を書く");
  expect(todo.completed).toBe(false);
});

test("空のタイトルを拒否する", () => {
  expect(() => createTodo({ title: "" })).toThrow();
});
```

内部関数を直接テストしてもよいが、そのテストを公開契約の保証として数えてはならない。

## TW-05: 直接依存を利用側manifestへ宣言する

他のworkspace packageを直接利用するworkspace packageは、自身の`package.json`へその依存を宣言しなければならない。ルートへの一括宣言や推移的インストールは、利用側workspace packageの直接依存宣言を代替しない。

### 不変条件

- workspace package間の各直接importに、利用側workspace packageのmanifest内で対応する依存宣言がある。
- manifestへ宣言されたworkspace package間依存は実際に利用され、不要な宣言が残っていない。
- 実行時、build時、型検査時、test時などの利用実態と依存種別が一致している。
- 依存の有無は、import specifierではなくresolverで解決した参照先のworkspace packageを基準に判定される。
- workspace package間の直接依存をworkspaceルートのmanifestだけへ置いていない。

```json
{
  "name": "@app/reminder",
  "dependencies": {
    "@app/todo": "1.0.0"
  }
}
```

workspace内依存のversion指定方法は、利用するnpmの機能とリリース方針に合わせてよい。

## TW-06: 暗黙的な依存を型付き公開シンボルにする

イベント名、DI token、データ形式、schema、resource descriptor、handler登録などを介した意味上の依存は、表現可能な限り所有workspace packageが公開するTypeScriptシンボルとして一度だけ定義し、利用側がimportしなければならない。

### 不変条件

- 契約の意味を所有するworkspace packageが、共有される名前とデータ構造の正本を公開する。
- 発行側、購読側、提供側、利用側がそれぞれ同じ公開シンボルを参照する。
- 同じイベント名、token、schema、resource名を複数workspace packageへ文字列や構造として重複定義していない。
- 公開契約の利用箇所を参照検索でき、データ構造の不整合を型検査またはschema検査で検出できる。
- 公開シンボルへの依存が、利用側manifestにも直接依存として現れる。

```ts
// @app/todo/events が所有する公開契約
export const TodoBecameOverdue = defineEvent<{
  todoId: string;
  dueAt: Date;
}>("todo.became-overdue");
```

```ts
// 発行側
import { TodoBecameOverdue } from "@app/todo/events";

broker.publish(TodoBecameOverdue, {
  todoId: todo.id,
  dueAt: todo.dueAt,
});
```

```ts
// 購読側
import { TodoBecameOverdue } from "@app/todo/events";

broker.subscribe(TodoBecameOverdue, event => {
  scheduleReminder(event.todoId);
});
```

発行側と購読側が`"todo.became-overdue"`とpayload型を別々に記述する実装は、意味上の依存をmanifest、参照検索、型検査から追跡できないため違反である。

## TW-07: importで表せない依存だけをREADMEへ明示する

外部API、queue、環境変数、共有ファイル、起動順序など、TypeScriptの公開シンボルへのimportで表せない依存は、利用側workspace packageの`README.md`へ明記しなければならない。他workspace packageが所有する資源の場合は、所有workspace package側からも対応関係をたどれるようにする。

### 不変条件

- 各非import依存について、依存先、対象の資源または契約、依存理由を特定できる。
- 他workspace package所有の資源について、所有workspace packageと利用workspace packageの両側から関係をたどれる。
- 型、schema、event、token、accessorなどの公開シンボルで表現できる依存を、README記載だけで済ませていない。
- READMEの記録が現在の実行時依存と一致している。

```markdown
## Non-import dependencies

| Depends on | Resource or contract | Reason |
|---|---|---|
| `@app/todo` | `todo-events` queue | 期限切れイベントを購読する |
| Notification API | `POST /messages` | 利用者へ通知を配送する |
| Runtime environment | `NOTIFICATION_API_URL` | APIの接続先を取得する |
```

## TW-08: 境界と依存宣言を静的に強制する

workspace package境界と依存宣言は、すべてのworkspace packageを対象とする静的検査によって継続的に強制できなければならない。検査器の製品や実装方式は固定しない。

### 不変条件

- resolverで解決したimport先が属するworkspace packageを判定できる。
- 未宣言のworkspace package間依存、deep import、workspace package境界を越えるfilesystem importとaliasを検出して拒否できる。
- manifestにある未使用依存と、利用実態に合わない依存種別を検出できる。
- `exports`にない外部利用と、公開を意図しない内部ファイルへの到達を検出できる。
- 検査対象外のworkspace packageや、手動確認だけに依存する例外経路が存在しない。

## 四つの性質との対応

| MMAの性質 | TypeScript/npm workspacesで主に維持する規約 |
|---|---|
| 変更凝集性 | READMEによる責務、非責務、変更理由の定義 |
| 実装隠蔽性 | 限定された`exports`とdeep import、filesystem import、迂回aliasの禁止 |
| ブラックボックス性 | README、公開entrypoint、外部利用者としての公開契約テスト |
| 依存追跡性 | workspace packageごとのmanifest、importとの一致、型付き公開シンボル、非import依存の明示 |

一つの規約を別の規約で代替しない。例えば、`exports`を限定してもfilesystem importを許せば実装隠蔽性は成立せず、manifestへ依存を書いても文字列だけで結び付いた意味上の依存を放置すれば依存追跡性は成立しない。
