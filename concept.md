---
title: 
category: 
abstract: 
writtenAt: 
definitions:
  - term: "変更要求 (Change Requirement)"
    definition: "システムの挙動や設計の変更を、ソースコードの変更箇所や変更方法を明示せずに、指し示したもの。"
  - term: "影響範囲 (Scope)"
    definition: "特定の変更要求に対応するために、変更・確認・テストする必要がある、またはその可能性を検討すべきコードベースの範囲"
  - term: "モジュール (Module)"
    definition: "何らかの境界（ディレクトリ等）で区切られたソースコードの集合"
  - term: "公開契約 (Public Contract)"
    definition: "外部ModuleがそのModuleを利用する際に依存してよい、公開APIと、その入力条件・保証される振る舞い・エラー・副作用などの約束。"
  - term: "実装隠蔽性（Implementation Hiding）"
    definition: "Moduleが最小限の公開契約とそれ以外の内部実装に明確に分けられ、外部Moduleから内部実装へ依存できない性質。"
  - term: "変更凝集性（Change Cohesion）"
    definition: "Moduleは「変更理由」で分割されていること"
  - term: "依存追跡性（Dependency Traceability）"
    definition: "Module間の結合関係を静的に追跡可能であること"
  - term: "ブラックボックス性 (Black Box Usability)"
    definition: "公開契約だけで、Moduleの利用可否と期待される振る舞いが判断できること"
  - term: "公開API"
    definition: "外部Moduleからコード上利用できる要素とその構造。要素とは関数・型・クラス・イベントなど。"
  - term: "公開契約資料 (Public Contract Artifact)"
    definition: "公開契約を、内部実装を読むより短時間・少トークンで理解できる資料。 典型的にはREADME、テストケース、公開型定義など。"
---

# 変更影響解析を安くする Module Mall Architecture

## Introduction

### Situation

- ソフトウェア開発は、<Term>変更要求</Term>をコードベースに反映していく作業である。
  - <Term>変更要求</Term>とは、システムの挙動や設計の変更を、ソースコードの変更箇所や変更方法を明示せずに指し示したものである。
  - 例えば、「TODOリスト上のアイテムを並べ替えられるようにする」「同期処理を非同期処理に変更する」「あるライブラリのバージョンを上げる」「ある不具合を修正する」などが<Term>変更要求</Term>である。
- <Term>変更要求</Term>に対処するためには、計画・実装・レビュー・テスト・不具合調査の各活動において、コードベースのどの範囲を考慮に入れるべきかを特定しなければならない。
  - 計画では、変更すべき箇所と、それに伴って修正が必要になる箇所。
  - 実装では、読み解き、実際に変更する箇所。
  - レビューでは、変更内容の正しさと変更漏れを確認する箇所。
  - テストでは、影響しうる箇所と、影響がなく対象から除外できる箇所。
  - 不具合調査では、原因を探索する箇所。
  - 本稿では、このような、特定の<Term>変更要求</Term>に対応するための特定の活動において、考慮に入れるべきコードベースの範囲を<Term>影響範囲</Term>と呼ぶ。
- すなわち、ソフトウェア開発は、<Term>変更要求</Term>に対して各活動の<Term>影響範囲</Term>を特定し、その<Term>影響範囲</Term>内で作業することを繰り返す活動であると言える。

### Complication

- このうち、<Term>影響範囲</Term>と実現方針が定まった後のコードの読解・生成・修正・テストといった作業の多くは、AIによって代替し、コストを大幅に削減できる。
  - 開発計画で<Term>影響範囲</Term>特定と実現方針策定を行えば、その実装はAIに任せられる。
  - レビューで本来変更すべき箇所の修正漏れを発見できれば、その修正はAIに任せられる。
- 一方で、<Term>影響範囲</Term>の特定は作業の内容・難易度判断・成否に大きく影響し、人間が行ってもAIが行っても高い解析コストを要する。
  - <Term>影響範囲</Term>を特定するためにコードベースを広く読み解くと、人間の解析時間と精神力、AIの解析時間と消費トークンが多く消費される。
  - コードベースの読み解きを省略して<Term>影響範囲</Term>を特定するには、注意深く設計されたシステムでない限り、過去の実装知識など多くのコンテキスト情報を要する。
  - 読み解く範囲やコンテキストが不足すれば、考慮すべき箇所を見落とす可能性が高まる。
- したがって、<Term>影響範囲</Term>内での作業がAIによって安くなるほど、<Term>影響範囲</Term>特定に要するコストの相対的な比重が高まり、ソフトウェア開発のボトルネックになる。

### Problem

- <Term>変更要求</Term>に対応するための<Term>影響範囲</Term>特定を効率化し、人間とAIの解析時間、AIのトークン消費、解析ミスを同時に削減するには、コードベースをどのように設計すればよいか？
  - 本稿で提唱する "Module Mall Architecture (MMA)" は、この課題を解決するアーキテクチャである。

## Content

### <Term>影響範囲</Term>を小さく、特定しやすくするアーキテクチャ

- ほとんどのコードベースは、ディレクトリなどの境界で区切られたソースコード群（本稿ではModuleと呼ぶ）の集まりとして構成されている。
  - コードを役割や関心ごとにまとめ、名前を付けて区切ることは、コードベースを理解し管理するために、ほとんどの開発者が自然に行っているだろう。
  - この区切りがあることで、ある変更について理解する必要があるModuleと、理解する必要がないModuleを切り分けられる可能性が生まれる。
  - ただし、単にModuleへ分割するだけで、変更<Term>影響範囲</Term>が小さく、特定しやすくなるわけではない。
- Module分割されたシステムにおける<Term>変更要求</Term>の<Term>影響範囲</Term>は、(1)<Term>変更要求</Term>に直接対応するModuleと、(2)そのModuleの変更に影響を受けうるModuleに分けられる。
  - 例えばTODOアプリで、「期限切れの判定を日単位から時刻単位へ変更する」という<Term>変更要求</Term>があったとする。
  - この場合、期限切れを判定する `todo` Moduleが、この<Term>変更要求</Term>に直接対応するModuleである。
  - その変更によって、期限切れのTODOを表示する`todo-list-ui`や通知対象を決める`reminder`に変更または確認が必要なら、それらも<Term>影響範囲</Term>に含まれる。
  - さらに`reminder`の変更が`notification-worker`へ影響するなら、それも間接的に影響を受けるModuleとして含まれる。
- <Term>変更要求</Term>の<Term>影響範囲</Term>を小さく保つには、直接対応するModuleと、変更の影響を受けるModuleの両方を少なくする必要がある。
  - ある<Term>変更要求</Term>に対応するModuleが多数または巨大なら、(1)の範囲が大きくなる。
  - Module間が無秩序に依存していれば、変更の影響が多くのModuleへ伝播し、(2)の範囲が大きくなる。
- <Term>影響範囲</Term>を容易に特定するには、<Term>変更要求</Term>とModuleの対応、およびModule間の影響を容易に調べられる必要がある。
  - Moduleの役割が説明されていなければ、<Term>変更要求</Term>に対応するかを判断するために内部のコードを広く読むことになる。
  - Module間の依存が追跡できなければ、変更した関数、型、クラスなどの利用箇所をコードベース全体から探さなければならない。
- したがって、Moduleの分割方法と依存関係には、以下の条件が必要である。
  - 1. ある<Term>変更要求</Term>に対応するModuleが少なく、かつ小さい。
  - 2. Moduleの変更に影響を受けるModuleが少ない。
  - 3. あるModuleが<Term>変更要求</Term>に対応するかどうかを容易に判断できる。
  - 4. 変更対象となる関数、型、クラスなどを使用している他Moduleを容易に特定できる。
- 本稿では、他の重要な設計上の要請を崩さない範囲で、これら四つの条件を可能な限り満たすことを目指す。
  - 実行環境、性能、セキュリティ、チームの責任分担などのために、変更理由の分散やModule間の依存を意図的に許容することは、正当なトレードオフである。
  - 一方、他の重要な性質を損なわずに解消できる分散や影響、探索の難しさは、設計によって取り除くべき改善余地である。
  - 本稿が目指すのは、この改善余地を可能な限り取り除くことである。
- 以降では、四つの条件を満たすためにModuleが備えるべき４つの性質 ---「<Term>変更凝集性</Term>」「<Term>実装隠蔽性</Term>」「<Term>ブラックボックス性</Term>」「<Term>依存追跡性</Term>」--- を解説する。

#### <Term>変更凝集性</Term>（Change Cohesion）

- <Term>変更要求</Term>に直接対応するModuleを少なく、かつ小さく保つには、Moduleが変更理由に沿って分割されている必要がある。本稿では、この性質を<Term>変更凝集性</Term>と呼ぶ。
  - 変更理由とは、期限切れの判定方法、TODOの表示方法、通知の送信方法など、コードを変更させる原因となるルールや関心事である。
- 一つの変更理由が複数のModuleに分散していると、一つの要求に対応するために複数のModuleを変更しなければならない。
  - 例えば、`frontend`・`backend`や`utils`・`logics`・`components`などの技術的な分類だけでModuleを分けると、一つのビジネスルールが複数のModuleへ分散しやすい。
  - その結果、`utils`の補助関数、`logics`の処理、`components`のバリデーションをそれぞれ変更する、といったように、一つの変更理由へ多数のModuleが直接対応することになる。
- 反対に、独立した複数の変更理由が一つのModuleに混在していると、そのModuleは必要以上に大きくなる。
  - その一部を変更する場合にも、変更と無関係なコードを含むModule全体が理解の対象になる。
  - `utils`や`shared`のように、さまざまな理由で変更されるコードを受け入れるModuleは、その典型である。
- <Term>変更凝集性</Term>を持たせるには、各変更理由を、それを所有する一つのModuleへ集める。
  - 同じ変更理由によって変更されるコードを同じModuleへ集め、独立した変更理由を混在させない。
  - 一つのルールを複数のModuleで重複して定義せず、他のModuleは所有Moduleの<Term>公開契約</Term>を通じて利用する。
  - 各Moduleについて「何が変わるときに、このModuleが変わるのか」を一貫して説明できるようにする。
- これにより、一つの<Term>変更要求</Term>に直接対応するModuleの数と、各Moduleで理解すべきコードの量を抑えられる。

#### <Term>実装隠蔽性</Term>（Implementation Hiding）

- Moduleの変更に影響を受けうるModuleを少なくするには、外部が依存できる範囲を限定し、それ以外をModule内部へ隠蔽する必要がある。本稿では、この性質を<Term>実装隠蔽性</Term>と呼ぶ。
- Module内のすべてのコードを外部から利用できると、外部Moduleはあらゆる実装詳細に依存できてしまう。
  - その場合、小さな実装変更でも外部へ影響し、影響先の変更がさらに別のModuleへ伝播し ... と、変更の影響が際限なく伝播していく可能性がある。
- この伝播を抑えるために、Moduleを、外部が依存してよい「<Term>公開契約</Term>」と、外部から隠蔽する「内部実装」に分ける。
  - <Term>公開契約</Term>とは、Moduleが外部に提供し、維持することを約束するインターフェースと振る舞いである。
  - 公開する関数、型、クラス、イベントだけでなく、その入出力、前提条件、保証、エラー、副作用など、外部から観測できる振る舞いも含まれる。
  - 内部実装とは、<Term>公開契約</Term>を実現するコードのうち、外部に対して維持を約束しない部分である。
- 外部Moduleには<Term>公開契約</Term>を通じた利用だけを許可し、内部実装への依存を禁止する。
  - 公開エントリーポイントだけを公開し、内部ファイルへのdeep importやfilesystem importを禁止する。
  - <Term>公開契約</Term>自体もModuleの利用に必要な範囲へ限定し、外部から依存される箇所を増やしすぎない。

- これにより、<Term>公開契約</Term>を維持する限り、内部実装の変更による影響をそのModule自身に限定できる。
  - 外部のModuleは、内部で使用されるデータ構造やアルゴリズムが変更されても影響を受けない。
  - その結果、Moduleの変更が他のModuleへ伝播する機会が減り、変更に影響を受けるModuleの数を少なく保てる。

#### <Term>ブラックボックス性</Term>（Black Box Usability）

- あるModuleが<Term>変更要求</Term>に対応するかを容易に判断するには、内部実装を読まず、READMEと<Term>公開契約</Term>だけで責務と振る舞いを理解できる必要がある。本稿では、この性質を<Term>ブラックボックス性</Term>と呼ぶ。
- 実装が隠蔽されていても、<Term>公開契約</Term>からModuleの役割を理解できなければ、<Term>変更要求</Term>との関係を判断するために内部実装を読むことになる。
  - 例えば「TODOリストの並び順を変更する」という要求に対して、並び順を所有するModuleが分からなければ、候補となるModuleを一つずつ調べなければならない。
  - 公開されている関数名や型名だけでは、責務、前提条件、エラー、副作用などを判断できない場合もある。
- また、必要な情報が<Term>公開契約</Term>に含まれていても、それがコード、テスト、設計書などへ分散していれば、<Term>公開契約</Term>を理解するために広い探索が必要になる。
  - <Term>公開契約</Term>は外部から参照できるだけでなく、どこを読めば理解できるかが明確でなければならない。
- そこで、Moduleの責務と外部から観測できる振る舞いを<Term>公開契約</Term>として定義し、それを短時間で理解できる<Term>公開契約資料</Term>を備える。

  - <Term>公開契約</Term>には、Moduleの責務と非責務、提供する<Term>公開API</Term>、入出力、前提条件、保証、エラー、副作用などを含める。
  - <Term>公開契約資料</Term>は、これらの情報を一か所または明確にたどれる形へ整理し、内部実装を読まなくてもModuleの役割と利用方法を把握できるようにする。
  - 外部の利用者は、<Term>公開契約資料</Term>を読むだけで、Moduleが<Term>変更要求</Term>に関係するか、目的のために利用できるか、どのように利用すべきかを判断できるようにする。
  - <Term>公開契約</Term>とその資料は内部実装より十分に小さく、短時間で確認できなければならない。
  - また、<Term>公開契約資料</Term>は<Term>公開契約</Term>の変更とともに更新し、現在の<Term>公開契約</Term>を信頼できる形で表していなければならない。
- これにより、Moduleの内部実装を読むことなく、<Term>変更要求</Term>に直接対応するModuleを特定できる。
  - Moduleの内部実装は、そのModuleを変更対象にすると判断した場合にのみ理解すればよい。
  - その結果、<Term>変更要求</Term>との関係を判断するための探索範囲を、コードベース全体の実装から、各Moduleの<Term>公開契約資料</Term>へ縮小できる。

#### <Term>依存追跡性</Term>（Dependency Traceability）

- <Term>公開契約</Term>の変更に影響を受けるModuleを容易に特定するには、Module間の依存関係を明示し、静的に追跡できる必要がある。本稿では、この性質を<Term>依存追跡性</Term>と呼ぶ。

- <Term>実装隠蔽性</Term>があっても、あるModuleの<Term>公開契約</Term>そのものを変更する場合には、それに依存するModuleを探し、影響を確認しなければならない。

  - 依存関係が明示されていなければ、その<Term>公開契約</Term>を利用しているModuleをコードベース全体から探す必要がある。
  - さらに、発見したModuleの変更が別のModuleへ影響する場合には、同じ探索を再帰的に繰り返さなければならない。
  - この状態では、<Term>影響範囲</Term>を確定するために、どこまでコードを調べればよいか分からない。
- この探索の負荷を軽減するために、すべてのModuleについて、どのModuleの<Term>公開契約</Term>へ依存しているかを明示する。
  - 各Moduleの直接依存をmanifestなどへ宣言し、コードベース全体の依存グラフを構成できるようにする。

  - 加えて、その宣言を信頼できるように、未宣言のModuleへの依存、内部実装へのdeep import、境界を迂回するfilesystem importなどを静的に検出し、禁止する。
  - import以外のイベント、設定、データ形式などを介した依存も、影響追跡に必要な関係を明示する。
- 加えて、命名規則・データなどを介した暗黙的な依存を避け、可能な限り型・クラス・関数など静的解析可能なシンボルを介して依存する。
  - 例えば、`todo` ModuleがTODOの期限切れを通知し、`reminder` Moduleがそれを購読するとする。
  - イベント名とデータ形式をそれぞれのModuleへ直接記述すると、両Moduleは暗黙的に依存する。

```ts
// todo Module
broker.publish("todo.became-overdue", {
  todoId: todo.id,
  dueAt: todo.dueAt,
});

// reminder Module
broker.subscribe("todo.became-overdue", event => {
  scheduleReminder(event.todoId);
});
```

  - この実装では、`reminder` Moduleが`todo` Moduleのイベントに依存しているにもかかわらず、両者を結ぶシンボル参照が存在しない。そのため、イベント名やデータ形式を変更した際の影響先を、参照検索や型検査によって確実に特定できない。
  - これに対し、イベント名とデータ形式を`todo` Moduleの公開要素として定義し、発行側と購読側が同じシンボルを参照すれば、依存を静的に追跡できる。

```ts
// todo Moduleの公開契約
export const TodoBecameOverdue = defineEvent<{
  todoId: string;
  dueAt: Date;
}>("todo.became-overdue");

// todo Module
broker.publish(TodoBecameOverdue, {
  todoId: todo.id,
  dueAt: todo.dueAt,
});

// reminder Module
import { TodoBecameOverdue } from "@app/todo";

broker.subscribe(TodoBecameOverdue, event => {
  scheduleReminder(event.todoId);
});
```

  - これにより、参照検索から発行箇所と購読箇所を特定でき、データ形式の不整合を型エラーとして検出できる。
- <Term>依存追跡性</Term>によって、<Term>公開契約</Term>の変更に影響しうるModuleを、コードベース全体の探索ではなく、明示された依存関係から特定できる。
  - 直接および間接に影響を受けるModuleをたどることで、変更<Term>影響範囲</Term>の外縁を判断できる。

### 実装上のプラクティス

### 四つの性質を維持するための実装プラクティス

* <Term>変更凝集性</Term>・<Term>実装隠蔽性</Term>・<Term>ブラックボックス性</Term>・<Term>依存追跡性</Term>は、設計時に意識するだけでなく、コードベースの構造と開発プロセスによって維持する必要がある。
  * Moduleを変更理由に沿って分割しても、責務と無関係なコードが追加され続ければ、<Term>変更凝集性</Term>は失われる。
  * 内部実装を非公開にすると決めても、filesystem importによって内部ファイルを参照できれば、<Term>実装隠蔽性</Term>は成立しない。
  * 依存関係をmanifestへ記述しても、未宣言の依存やimportに現れない依存があれば、その情報を信頼できない。
  * したがって、四つの性質を成立させるためには、Module境界、公開範囲、依存関係、説明、テストを、可能な限り機械的に強制・検証できる形へ落とし込む必要がある。
* 以降では、MMAの各Moduleを一つのworkspace packageとして実装し、責務・公開範囲・依存関係を管理する基本単位として使用する。
  * 具体例にはTypeScriptとnpm workspacesを使用するが、workspace packageごとにmanifestと公開範囲を持てるuv workspaceなどにも同じ考え方を適用できる。
  * workspace packageはコード上の理解と変更の境界であり、デプロイ単位や実行プロセスと一致させる必要はない。

```json
{
  "private": true,
  "workspaces": ["packages/*"]
}
```

```text
packages/
├── todo/
│   ├── package.json
│   ├── README.md
│   ├── src/
│   └── test/
├── reminder/
│   ├── package.json
│   ├── README.md
│   ├── src/
│   └── test/
└── notification-worker/
    ├── package.json
    ├── README.md
    ├── src/
    └── test/
```

#### 責務と変更理由をREADMEに定義する

* 各workspace packageのREADMEを、そのworkspace packageの責務・非責務・意味・意図の正本とする。
  * 責務は現在存在する機能やファイルの一覧ではなく、そのworkspace packageが所有するルールや判断として記述する。
  * あわせて、「何が変わるときに変更されるか」と「何が変わっても変更されないか」を明記し、workspace packageの変更理由を判別できるようにする。

```markdown
# @app/todo

## Responsibility

TODOのライフサイクルと、期限切れを含む状態判定を所有する。

## Non-responsibility

TODOの表示方法、通知時期、通知の配送方法は所有しない。

## Changes when

TODOの生成・更新・完了・期限切れに関するルールが変わるとき。

## Does not change when

TODOの表示方法や通知チャネルだけが変わるとき。
```

* READMEには、内部実装を読まずに<Term>変更要求</Term>との関係を判断できるだけの情報を記述する。
  * 例えば「期限切れの判定を日単位から時刻単位へ変更する」という要求なら、READMEから`todo` workspace packageが直接対応すると判断できなければならない。
  * 一方、アルゴリズムや内部データ構造まで記述すると内部変更のたびに更新が必要になるため、責務や<Term>公開API</Term>の意味を理解するために必要な範囲へ限定する。
* workspace packageへの変更は、READMEに定義された変更理由によって説明できなければならない。
  * 通知の配送方法を変えるために`todo` workspace packageを変更しているなら、通知都合のコードが混入している可能性があるため、レビューで確認する。
  * 同じルールの定義を複数のworkspace packageで変更している場合も、変更理由が分散していないかを確認する。ただし、<Term>公開契約</Term>の変更に利用側が追従することは責任の分散ではない。
* 一つの<Term>変更要求</Term>が複数の変更理由を含む場合は、無理に一つのworkspace packageへ閉じ込めない。
  * 例えば、期限切れ判定とその表示形式を同時に変更する要求では、`todo`と`todo-list-ui`の両方を変更してよい。
  * 重要なのは<Term>変更要求</Term>とworkspace packageを一対一にすることではなく、それぞれの変更理由が一つのworkspace packageへ集約されていることである。

#### <Term>公開API</Term>を明示的に限定する

* 各workspace packageには公開エントリーポイントを設け、外部workspace packageが利用できるAPIを明示的に限定する。
  * npm workspace packageでは、`package.json`の`exports`に公開するエントリーポイントだけを列挙する。
  * 内部で使用する補助関数、永続化形式、アルゴリズム、内部エラーなどは公開しない。

```json
{
  "name": "@app/todo",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./events": {
      "types": "./dist/events.d.ts",
      "import": "./dist/events.js"
    }
  }
}
```

```ts
// packages/todo/src/index.ts
export { createTodo } from "./create-todo.js";
export { completeTodo } from "./complete-todo.js";
export type { Todo } from "./todo.js";
```

* 用途の異なる<Term>公開API</Term>は、意図して維持するsubpath entrypointとして分ける。
  * 例えば通常の操作APIとイベント契約は、`@app/todo`と`@app/todo/events`として個別に公開できる。
  * 内部ディレクトリをglobで公開すると内部構造自体が<Term>公開契約</Term>になるため、subpathは利用目的に基づいて個別に設計する。

#### workspace package境界を迂回するimportを禁止する

* 外部workspace packageからは、公開エントリーポイントを経由したimportだけを許可する。
  * workspace package名から内部ファイルを指定するdeep importと、相対パス・絶対パス・aliasによって別workspace packageのファイルを直接参照するfilesystem importを禁止する。
  * 同一workspace package内で内部ファイルを相対importすることは、この制約の対象外とする。

```ts
// 許可
import { TodoBecameOverdue } from "@app/todo/events";

// 禁止: deep import
import { TodoBecameOverdue } from "@app/todo/src/internal/events.js";

// 禁止: workspace package境界を越えるfilesystem import
import { TodoBecameOverdue } from "../../todo/src/internal/events.js";
```

* `exports`だけではfilesystem importを防げないため、workspace package境界を越える参照を静的解析によって検出する。
  * import文の文字列だけを検査すると、aliasなどを介した参照を見落とす。
  * module resolverで解決した参照先がどのworkspace packageに属するかを判定し、別workspace packageの内部ファイルへ到達する参照を禁止する。

#### すべての<Term>公開契約</Term>をテストする

* <Term>公開API</Term>の型と構造、入力条件、保証される振る舞い、エラー、副作用など、すべての<Term>公開契約</Term>に対応する自動テストを用意する。
  * 正常系の代表例だけでなく、契約として定めた境界条件、失敗条件、状態変化も検証する。
  * <Term>公開契約</Term>を変更してもテストが変化しない場合は、その契約に対応するテストが不足していないかを確認する。
* <Term>公開契約</Term>テストは、外部workspace packageと同じ公開エントリーポイントだけを利用する。
  * 内部関数や内部データ構造を直接参照すると、<Term>公開契約</Term>を維持した内部変更でもテストが壊れ、ブラックボックスとしての保証にならない。
  * <Term>公開API</Term>へ入力を与え、戻り値、公開エラー、公開イベント、副作用、<Term>公開API</Term>から観測できる状態を検証する。

```ts
import { createTodo } from "@app/todo";

describe("createTodo", () => {
  it("指定されたタイトルを持つ未完了のTODOを作成する", () => {
    const todo = createTodo({ title: "原稿を書く" });

    expect(todo.title).toBe("原稿を書く");
    expect(todo.completed).toBe(false);
  });

  it("空のタイトルを拒否する", () => {
    expect(() => createTodo({ title: "" })).toThrow();
  });
});
```

* 公開される型も、外部利用者のコードとして型検査する。
  * 実行時テストだけでは、受け付ける型や拒否すべき入力など、利用側が依存する型上の契約を十分に検証できない。
  * コンパイルできる利用例と型エラーになる利用例を用意し、実行時テストと組み合わせて検証する。
* <Term>公開契約</Term>テストと内部実装テストは区別して管理する。
  * 複雑なアルゴリズムなどを内部関数から直接テストしてもよいが、それを<Term>公開契約</Term>の保証として数えてはならない。
  * ディレクトリや実行コマンドを分け、<Term>公開契約</Term>の網羅状況を独立して確認できるようにする。

```text
packages/todo/test/
├── contract/
│   ├── create-todo.test.ts
│   ├── complete-todo.test.ts
│   └── todo-events.test.ts
└── internal/
    ├── overdue-calculation.test.ts
    └── database-mapper.test.ts
```

* README、<Term>公開API</Term>、契約テストは、それぞれ異なる情報の正本として扱う。
  * READMEはworkspace packageの責務・非責務・意味・意図を説明し、<Term>公開API</Term>は外部から利用できる要素と型構造を定義する。
  * 契約テストは、<Term>公開API</Term>に付随する意味的な約束を現在の実装が満たしていることを保証する。

#### workspace package間の直接依存をmanifestへ宣言する

* 他workspace packageを直接利用するworkspace packageは、その依存を自身のmanifestへ宣言する。
  * workspaceルートに対象workspace packageが存在することや、推移的にインストールされていることを理由に、未宣言で利用してはならない。
  * 直接依存が各manifestに揃っていれば、manifestからworkspace package間の依存グラフを構成できる。

```json
{
  "name": "@app/reminder",
  "dependencies": {
    "@app/todo": "1.0.0"
  }
}
```

* 依存は、その依存を実際に利用するworkspace packageのmanifestへ記述する。
  * workspaceルートのmanifestには、リポジトリ全体で使用するformatter、linter、境界検査器などのtoolingだけを配置する。

#### importとmanifestを一致させる

* 実際のworkspace package間importとmanifestの依存宣言を相互に検証する。
  * 他workspace packageをimportしているのに依存が宣言されていない場合、宣言されているのに利用されていない場合、依存種別が利用箇所と一致しない場合をエラーにする。
  * この検査もimportの表記ではなく、解決後の参照先が属するworkspace packageを基準に行う。
* manifestを信頼できる状態にすることで、<Term>公開契約</Term>変更の影響候補をworkspace package単位で追跡できる。
  * `todo`の<Term>公開契約</Term>を変更した場合は、まず`todo`への直接依存を宣言しているworkspace packageを確認する。
  * 依存側の<Term>公開契約</Term>にも変更が及ぶ場合は、さらにそのworkspace packageへの依存をたどることで、間接的な影響候補も特定できる。

#### シンボルを介さない暗黙的な依存を静的に追跡可能にする

* 文字列、命名規則、データ形式などを介した依存は、可能な限り所有workspace packageが公開するシンボルへのimportへ変換する。
  * 意味上の依存があってもコード上の参照がなければ、manifest、参照検索、型検査から影響先を発見できない。
  * 依存される契約を型、関数、クラス、定数などとして定義し、利用側がそれをimportする形にする。

| 暗黙的な依存    | 追跡しにくい実装                           | 改善                                     |
| --------- | ---------------------------------- | -------------------------------------- |
| イベント      | 発行側と購読側が同じ文字列とデータ構造を記述する           | 所有workspace packageが型付きイベントをexportする             |
| DIトークン    | `"TodoRepository"`などの文字列を複数箇所に記述する | 所有workspace packageが型付きトークンや`Symbol`をexportする    |
| データ形式     | 同じJSON構造を各workspace packageで個別に定義する          | 所有workspace packageが型・schema・parserをexportする     |
| handler探索 | globや命名規則だけでhandlerを発見する           | handlerをimportしてregistryへ明示的に登録する      |
| DBテーブル    | 他workspace packageがテーブル名とcolumn構造を知って直接参照する  | 所有workspace packageの<Term>公開API</Term>または公開interfaceを利用する     |
| resource名 | queue名やファイルパスを文字列で共有する             | 所有workspace packageがdescriptorやaccessorをexportする |

* 例えば、文字列だけで結び付いたイベントは、イベント定義を所有workspace packageの公開シンボルにする。
  * 発行側と購読側がイベント名とデータ形式を個別に記述すると、購読側から所有workspace packageへの参照が生まれない。
  * 同じイベント定義を参照させれば、イベントの利用箇所を参照検索でき、データ形式の不整合を型検査で検出できる。

```ts
// 追跡しにくい実装

// todo workspace package
broker.publish("todo.became-overdue", {
  todoId: todo.id,
  dueAt: todo.dueAt,
});

// reminder workspace package
broker.subscribe("todo.became-overdue", event => {
  scheduleReminder(event.todoId);
});
```

```ts
// packages/todo/src/events.ts
export const TodoBecameOverdue = defineEvent<{
  todoId: string;
  dueAt: Date;
}>("todo.became-overdue");
```

```ts
// todo workspace package
import { TodoBecameOverdue } from "@app/todo/events";

broker.publish(TodoBecameOverdue, {
  todoId: todo.id,
  dueAt: todo.dueAt,
});

// reminder workspace package
import { TodoBecameOverdue } from "@app/todo/events";

broker.subscribe(TodoBecameOverdue, event => {
  scheduleReminder(event.todoId);
});
```

* 公開シンボルへの変換は新しい依存を作るのではなく、すでに存在する意味上の依存をコード上へ表すものである。
  * `reminder`は変換前から、`todo.became-overdue`というイベントの意味とデータ形式に依存している。
  * importを追加することで、その依存をmanifest、参照検索、型検査から観測できるようにする。
* importだけでは表せない依存は、各workspace packageのREADMEへ定型形式で明記する。
  * 外部API、queue、環境変数、共有ファイル、起動順序など、実行時資源との関係が該当する。
  * 他workspace packageが所有する資源については所有workspace packageも記述し、どのworkspace packageの変更から影響を追えばよいかを明確にする。

```markdown
## Non-import dependencies

| Depends on | Resource or contract | Reason |
|---|---|---|
| `@app/todo` | `todo-events` queue | 期限切れイベントを購読する |
| Notification API | `POST /messages` | 利用者へ通知を配送する |
| Runtime environment | `NOTIFICATION_API_URL` | APIの接続先を取得する |
```

* READMEへの記述は、import依存へ変換できない場合にのみ使用する。
  * READMEだけでは、型検査による不整合検出やシンボル単位の参照検索はできない。
  * 型、schema、イベント定義、accessorなどとして表現できる情報は、READMEへの記述だけで済ませず、先に公開シンボルへ変換する。

#### 四つの性質との対応

* 以上のプラクティスによって、四つの性質を設計上の方針ではなく、コードベース上で継続的に維持する。

| 性質        | 主に維持する仕組み                                             |
| --------- | ----------------------------------------------------- |
| <Term>変更凝集性</Term>     | READMEによる責務・非責務・変更理由の定義と、変更時のレビュー                     |
| 実装隠蔽性     | 最小限の`exports`と、deep import・filesystem importの禁止       |
| <Term>ブラックボックス性</Term> | README、<Term>公開API</Term>、公開エントリーポイント経由の契約テスト                      |
| <Term>依存追跡性</Term>     | workspace packageごとのmanifest、importとの照合、公開シンボルへの依存、非import依存の明記 |
