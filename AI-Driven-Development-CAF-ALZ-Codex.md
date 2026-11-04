# AI駆動開発 — Codex

# 概要
本ページでは、AI駆動開発の概念と実践方法についてまとめる。

# AI駆動開発とは
AI駆動開発は、AIに指示をしてコードを開発させる開発手法である。

## AI駆動開発における決定論
AI駆動開発は Y = F(X) という式で理解できる。
この式における各変数は以下を意味する。

| 変数 | 意味 |
| --- | --- |
| X | AIへの入力コンテキスト |
| Y | AIからの出力 |
| F | AIのモデル |

![AI駆動開発の全体像：コンテキストをAIへ渡し、出力をテストとレビューで確認する](assets/ai-driven-development/overview.png)

つまり、AI駆動開発は、AIに与えるX（コンテキスト）、Y（出力の精査）、F（AIのモデルと使い方）の三つの要素を制御して最高の結果を得ることが重要になる。

## 各変数の制御方針
AI駆動開発において、各変数は以下の方針で制御する。

| 変数 | 制御方法 |
| --- | --- |
| X | コンテキストエンジニアリングによって制御する |
| Y | テスト駆動開発によって制御する |
| F | サブエージェントによるオーケストレーションによって制御する |

この3つのコンテキストエンジニアリング、テスト駆動開発、オーケストレーションがAI駆動開発における根幹である。

## コンテキストエンジニアリング
AIへの入力（コンテキスト）は以下がある。

- プロンプト（AIへの指示）
- ルール（AGENTS.md等）
- コードベース（ソースコードそのもの）
- ドキュメント（要件定義書、仕様書など）
- スキル（何か特定の操作を行うためのノウハウや手順）
- 外部情報（MCPやCLIで取得）
- 対話履歴

![AIに渡すコンテキストの7要素](assets/ai-driven-development/context-elements.png)

コンテキストエンジニアリングはこれらの要素を最適化し、AIから最高の結果を得るための方法論である。
特に、コードベースについては汚染されると、次の X として入力されるため、次の出力も汚染され続ける悪循環の陥る。

コンテキストエンジニアリングの実践内容としては、構造化したルールドキュメント・テスト駆動開発によってコードベースを綺麗に保つ、ドキュメントや外部情報は必要最小限のみ与えるという内容である。

## テスト駆動開発
AIからの出力の特性として、同じ質問をしても同じ内容が必ずしも返ってこない揺れがある。
そのため、AIからの応答の必須条件として、「テストへの合格」を指定することで、出力の品質を確保することが重要である。

テスト駆動開発の実践内容としては、ルールによって実装前に必ずテストを作成する・テスト内容は人間が必ずレビューする・CIによってマージの条件としてテストの合格を必須にするという内容である。

![要件と合格条件から始め、テスト・実装・改善を繰り返す開発ループ](assets/ai-driven-development/tdd-loop.png)

## オーケストレーション
実装させたAIに、レビューをさせると、自己評価が高くなり正常に評価しないという問題がある。
後に説明するが、AIの相互レビューは実装の経緯を知らないAIに観点を分けて（ルール準拠、ファイル構造、SSOT、セキュリティ、テスト）レビューさせる必要がある。
※SSOT：Single Source of Truth（一つの情報の情報源は一つのみにする。ルールの情報源を一つに保つなど）

その際、サブエージェントにレビューを分けてオーケストレーションする方法等を紹介する。

## AI相互レビュー

![変更差分をセキュリティ・テスト・設計・ルールの観点から独立して読む](assets/ai-driven-development/independent-review.png)

上記で、X・Y・Fを制御したうえに必要なのがAIによる相互レビューである。

各観点を専門とするレビューエージェントを定義し、テストの合格後にテストでは拾えない観点（ルールへの準拠、SSOT、セキュリティ等）でレビューし、品質を向上させる。

AIレビューの振動を防ぐため、レビューと修正は3周以内に制限させるのが重要である。

# AI駆動開発の実践
ここからは、実際のAI駆動開発の実践方法を例を交えながら紹介する。

まず、要件定義ファイルを作成してリポジトリに必要なルールを置き、その後以下の8ステップで開発するフローをプロンプト例を添えて紹介する。

![プランから実装・検証・レビュー・改善へ進む8ステップ](assets/ai-driven-development/development-flow.png)



## 前提
GitHubのリポジトリを使用していること。各社のAI駆動開発のプラクティスはGitHubによるバージョン管理・AIからの読み取り・ブランチによる開発・レビューの仕組みを前提として作られている。

題材を「CAFに沿ったAzure Platform Landing Zoneの新規構築」として例を示す。要件定義・ルール・レビュー定義・スキル・テスト・プラン・8ステップのプロンプトを一式で示す。

以降の作成先は、リポジトリのルートからの相対パスで示す。

## A.0 要件定義ファイルの作成

作成先：`docs/requirements/platform-landing-zone.md`。何を・なぜ作るかと合格条件をまとめ、設計や実装方法はプランに分ける。

```markdown
# 要件定義：Azure Platform Landing Zone

## 目的と対象ユーザー
基盤担当者が、組織共通のAzure環境を再現可能かつ安全に構築・変更できるようにする。
設定漏れや意図しない公開を防ぎ、利用チームが共通の管理・接続・監視機能を使える状態を目指す。

## ユーザーストーリー
基盤担当者として、共通基盤の構築結果と制御の動作を確認したい。
依存する製品を更新しながら、環境差や手動変更による不備を見逃さず運用するためである。

## 主要な要件
- FR-001：管理グループ・Policy・RBAC・監視・共通ネットワークを提供する。
- FR-002：有効なPolicyによって、禁止・変更・自動配置・既存不適合の修復が働く。
- FR-003：Spoke間・オンプレ接続はFirewallを経由し、明示的に許可した通信だけを通す。
- FR-004：名前解決・管理接続・ログ収集を共通基盤として利用できる。
- FR-005：ALZ標準を土台に、独自Policy・割当・期限付き例外を追加・変更できる。
- FR-006：モジュール・provider・Policy Libraryの更新を検出し、既存の基盤を安全に更新できる。
- FR-007：管理対象の構成差異と管理外の追加を定期的に検知し、検査失敗・未実行も異常として扱う。

## 受け入れ条件
| ID | 前提（Given） | 操作（When） | 期待結果（Then） |
| --- | --- | --- | --- |
| D-001 | 専用の試験環境がある | 共通基盤を構築する | 階層・配置・権限・Policy・監視・ネットワークが要件と一致し、名前解決・管理接続・ログ収集を利用できる |
| P-001 | 対象Policyが有効である | 許可・違反ケースと修復を実行する | 許可操作は成功し、違反操作は対象Policyで拒否される。変更・自動配置・修復の実値も期待どおりになる |
| N-001 | Spokeとオンプレの試験先が動作している | 許可・非許可の通信を行う | 許可通信は成功し、非許可通信は遮断され、Firewall経由の証跡を確認できる |
| C-001 | 試験で今回の基盤を作成している | 試験後に後始末する | 今回の作成物だけが削除され、固定の試験基盤と回線が保持される |
| POL-ADD-001 | ALZ標準と独自の制御がある | 独自Policy・割当・例外を追加する | 意図した対象だけに適用され、既存の制御を損なわず、例外の理由・責任者・期限を確認できる |
| UPD-001 | 現行版と更新候補がある | 新版の新規構築と旧版からの更新を試す | 要件・Policy・通信が成立し、未承認の削除・置換がなく、再planと後始末まで成功する |
| DRIFT-001 | 適用済みの構成と検査対象が定義されている | 設定変更・管理外追加・検査不能を起こす | 定めた検知期限内に差異または監視異常を報告し、自動でapplyや基準変更をしない |

## 前提
EA契約、新規環境、東日本の単一リージョン。
公式ALZ標準を基準に、Hub-Spoke・Bastion・ExpressRouteを含める。
別テナントの専用試験環境と開通済み試験回線を利用する。
検査対象・周期・検知期限を運用開始前に決め、検査できない対象を正常扱いしない。

## スコープ外
アプリケーション開発、サブスクリプション自動払い出し、回線事業者の開通・廃止。
複数リージョン、VPN、有償DDoS、Firewall Premium機能。
```

## A.1 リポジトリ装置の実物一式

### A.1.1 リポジトリ全体の指示

作成先：リポジトリ直下の `AGENTS.md`。Codexが作業開始時に読む指示をまとめる。

```markdown
# Azure Platform Landing Zone

CAFに沿った共通基盤を東日本に構築する。管理グループ・Policy・RBAC・監視と、Firewall・DNS・Bastion・ExpressRouteを含むHub-Spokeを対象とする。

作業前に `docs/requirements/platform-landing-zone.md` を読み、目的と受け入れ条件を確認する。

## 技術スタック

- Terraform 1.12以降と公式AVM。AzureはEA契約
- GitHub Actions + Bash。Azure認証はUAMIへのOIDCフェデレーション
- 単体はGo＋testify、結合はTerraformの構成検証、実AzureのE2E・更新試験はGo＋Terratest
- Go 1.23以降、Terratest `v0.48.2`、testify `v1.10.0`を参照版とする

## ディレクトリ構成

- `infra/bootstrap/` — 常設のID・backend等。試験の削除対象から分離
- `infra/platform/` — 共通基盤を構成するTerraform root
- `infra/probes/storage-policy/` — Policy試験で作成・削除するTerraform root
- `lib/` — ALZ形式の独自ライブラリ。標準への追加・差分を管理
- `environments/production/`・`environments/e2e/` — 入力の`platform.tfvars.json`と`backend.hcl`
- `tests/unit/`・`tests/e2e/`・`tests/drift/` — 入力契約・実動作・ドリフトの検証
- `scripts/` — テストや検査を呼ぶ実行入口
- `.github/workflows/test.yml`・`drift.yml` — PR検証・定期検査、`renovate.json` — 依存更新PR
- `docs/requirements/`・`docs/plans/`・`docs/rules/` — 要件・設計計画・開発ルール

## 主要コマンド

- `bash scripts/test.sh unit` / `integration` / `e2e` / `upgrade` — 種別ごとのテスト
- `bash scripts/drift.sh` — 適用済みの版・入力による読み取り検査
- `terraform fmt -check -recursive infra` — フォーマット検査

## 開発ルールの適用

対象に一致するルールを作業前に読む。複数一致した場合はすべて適用する。

| 対象 | 必ず読むルール |
| --- | --- |
| `infra/**`、`lib/**`、`environments/**`、`scripts/**`、`.github/workflows/**`、`renovate.json` | `docs/rules/architecture.md` |
| `tests/**`、テストの追加・変更 | `docs/rules/testing.md` |
| PR作成前・レビュー時 | `docs/rules/review.md` |

ルールと要件が矛盾したら、推測で進めず矛盾を報告する。
```

ルール本文は `docs/rules/` に置き、`AGENTS.md` の適用表に従って読む。Codexの `.rules` はコマンド実行の許可設定なので、文章ルールには使わない。

### A.1.2 ルールファイル — アーキテクチャ

作成先：`docs/rules/architecture.md`。構成・所有範囲・依存関係を決める。

```markdown
# architecture

## 構造

infra/
  bootstrap/                    # 常設のID・backend等
  platform/                     # 共通基盤のroot
    governance.tf               # 管理グループ・Policy・RBAC
    network.tf                  # 共通ネットワーク
    monitoring.tf               # 監視・ログ基盤
    policy_exemptions.tf         # 理由・責任者・期限付き例外
    providers.tf                # 接続先・provider設定
    versions.tf                 # Terraform・providerの制約
    variables.tf                # 入力と検証
    outputs.tf                  # 外部へ渡す最小限の値
  probes/storage-policy/        # 試験対象のroot
lib/
  policy_definitions/            # 独自Policy
  policy_set_definitions/        # Initiative
  policy_assignments/            # 割当
  archetype_definitions/         # 独自の適用セット
  archetype_overrides/           # ALZ標準への追加・除去差分
  architecture_definitions/      # 管理グループと適用セットの対応
  role_definitions/              # 独自ロールが必要な場合
environments/                  # production/e2eの入力・backend設定
tests/                         # unit/e2e/drift
scripts/                       # 実行・観測・回収の入口
.github/workflows/              # test.yml・drift.yml
renovate.json                  # 依存更新PR

## 1ファイル1責務

- そのファイルの担当を一文で言えること。1リソース1ファイルにはしない
- 定義や独自処理が増えたら、責務に沿って分ける。単純な設定値の受け渡しだけのモジュールは作らない
- 入力の制約はvariables.tf、計算は必要な構成内のlocalsに置く
- リソースの管理はTerraform、Bashは実行・待機・観測・回収に限定する

## 所有範囲と依存関係

- bootstrap・platform・probesをライフサイクルに応じた別root・別stateで管理する
- platform内のファイル分割ではstateは分かれない。環境ごと・試験runごとにbackendと入力を分ける
- 同じリソースを複数stateで管理しない。属性ごとにTerraform/Policyの所有範囲を決め、競合させない。Policy生成物と固定fixtureも所有者を記録する
- AVMの公開入出力を使い、内部リソースへ直接依存しない。ALZはplan時に確定するIDと対応する_dependencies入力で接続し、module全体へdepends_onを付けない

## ALZライブラリ

- 標準の参照版を固定し、libには独自資産と差分だけを置く。通常の追加で同名上書きや標準の全コピーをしない
- フォルダ名は分類用。ALZが識別に使うファイル名末尾とスキーマを守る（例：*.alz_policy_definition.json、*.alz_policy_assignment.json、*.alz_archetype_override.yaml）
- 1定義・1割当・1archetypeを1ファイルにする。必要な種類のディレクトリだけ作る
- 定義→割当→archetype/override→architectureを接続する。overrideは新しい名前をarchitectureから参照する
- 例外リソースはpolicy_exemptions.tfで管理し、対象・理由・責任者・期限を明示する

## 依存更新とドリフト

- providerは各rootの.terraform.lock.hcl、AVMはmoduleのversion、ALZ Libraryは参照版を固定する。lockファイルだけではmoduleは固定されない
- 更新PRで新版の新規構築と旧版からの更新を検証し、削除・置換・state移行をレビューする。期待値は更新版へ自動追従させない
- ドリフトは本番に適用済みのcommit・入力・固定版で定期planを行い、JSONのresource_driftも確認する
- 実効Policy・例外・RBAC・管理外リソースも照合する。対象・周期・期限を定め、失敗・未実行・古い結果を正常扱いしない
- 検査は読み取り専用。apply・stateへの差分受け入れ・期待値変更は、別のレビューされた変更として行う
```

### A.1.3 ルールファイル — テスト

作成先：`docs/rules/testing.md`。テストの守備範囲と合格条件を決める。

```markdown
# testing

## 種別と実行

| 種別 | 置き場所 | 実行 | 何を守るか |
| --- | --- | --- | --- |
| 単体：Go＋testify | tests/unit/ | bash scripts/test.sh unit | 入力制約・ALZ資産の契約 |
| 結合：Terraform検査 | infra/の各root | bash scripts/test.sh integration | fmt・validate（実際のplanはE2E・更新試験） |
| E2E：Terratest＋Azure API | tests/e2e/ | bash scripts/test.sh e2e | 実配置・Policy・権限・通信・再plan・削除 |
| 更新：Terratest＋Azure API | tests/e2e/ | bash scripts/test.sh upgrade | 新版の新規構築と旧版からの更新 |
| ドリフト：読み取り検査 | tests/drift/ | bash scripts/drift.sh | 実設定差異・管理外追加・観測失敗 |

## テスト駆動と骨抜き禁止

- 期待値は要件・承認した構成を出典にする。Terraform outputや取得した実値を、そのまま正解にしない
- 1件ずつ失敗を確認してから実装する。環境不備や認証失敗を、振る舞いのRedとして数えない
- 不要な制約をテストのために追加しない。アドレスの重複・包含や、必要な制御を検証する
- assert削除・skip・常時成功モック・Policy無効化で通さない。E2Eの内部応答はモックしない
- テスト名とassertから、何をしたらどうなるか読めること

## 実Azureの合格条件

- 全有効Deny/DenyAction/Modify/Append/DINE等をケース台帳と対応づける。必須ケース不足は失敗。Audit系は割当・パラメータを照合する
- 許可側の成功と、対象Policyによる拒否を対で確認する。権限不足や入力エラーを拒否の成功にしない
- DINEの新規配置・既存不適合の修復と、必要な権限・配置先の実値を確認する
- 独自Policyの追加PRには、その正常・違反・修復の該当ケースを含める
- 更新は同じ要件で現行版と候補版を比較する。未承認の削除・置換を通さず、再planと後始末まで確認する

## 試験環境と後始末

- 別テナントの固定fixtureを照合し、共有poolを排他する。実E2Eは直列化し、t.Parallel()を使わない
- 試験対象はinfra/probes/で作り、run固有の名前とstateを使う。固定のtenant・subscription・ID・backend・回線は削除しない
- 作成前にrunner外へstate key・所有記録を保存する。独立cleanupで回収・subscription退避・残存を確認し、失敗したpoolは隔離する
- 固定sleepではなく期限付きで待つ。最新commitの全必須試験と後始末の成功をPRのマージ条件にする

## ドリフト検査の試験

- 専用試験環境で設定変更・Policy例外追加・管理外追加・権限喪失を起こし、差異または監視異常として検出できることを確認する
- 通常planの終了コードだけで正常にしない。resource_driftと実設定・台帳も照合する
- 検査結果の期限超過とjob未実行を、検査jobの外側からも監視する。ドリフト検査で自動applyしない
```

### A.1.4 ルールファイル - レビュー
作成先：`docs/rules/review.md`。レビューの観点、指摘の優先度、完了条件を決める。

```markdown
# AI相互レビューの適用

## タイミング

- プルリクエスト作成前に必ず1回、実装とは別セッションのAIで実行する
- P0・P1の修正後は再レビューに戻す

## レビュー観点

実装の経緯は渡さず、差分（git diff）だけを渡して次の観点で読む：

1. セキュリティ：認可漏れ・インジェクション・機密情報の露出
2. テスト：変更に対応するテストの有無・骨抜きテストの検出
3. 設計：1ファイル1責務・機能境界（architecture.md準拠）の確認
4. ルール準拠：docs/rules/ 配下の該当ルールとの矛盾

## 指摘の優先度分類

- **P0（必須修正）**：セキュリティ・データ破壊・機能の誤り。マージをブロックする
- **P1（推奨修正）**：深刻なバグの芽・テスト網羅の欠落。合理的理由があればスキップ可
- **P2（軽微）**：スタイル・命名。時間があれば修正、なければTODO化

## 終わりの条件

- P0・P1がゼロなら合格
- P2だけなら修正せずマージしてよい
- 3ラウンドを超えて指摘が循環したら、人間が裁定する

## 静的レビューの限界

- 差分を読むAIレビューでは、Policyの実効性・経路の取り違え・修復の待ち続けといった実行時のバグは捕まらない。配置や実行時の挙動を変えたら、最後は専用Azure環境で動かして確かめる
```


### A.1.5 依存更新の設定

作成先：`infra/platform/providers.tf`。ALZ標準の版と、独自ライブラリの場所を指定する（参照版は例）。

```hcl
provider "alz" {
  library_references = [
    { path = "platform/alz", ref = "2026.10.0" },
    { custom_url = "${path.root}/../../lib" }
  ]
}
```

作成先：`renovate.json`。AVM・providerの更新PRを作り、ALZ Libraryの参照版も検出する。

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "automerge": false,
  "customManagers": [{
    "customType": "regex",
    "managerFilePatterns": ["/^infra\\/platform\\/providers\\.tf$/"],
    "matchStrings": ["path\\s*=\\s*\"platform/alz\"\\s*,?\\s*ref\\s*=\\s*\"(?<currentValue>[^\"\\r\\n]+)\""],
    "datasourceTemplate": "github-tags",
    "depNameTemplate": "Azure/Azure-Landing-Zones-Library",
    "extractVersionTemplate": "^platform/alz/(?<version>\\d+\\.\\d+\\.\\d+)$"
  }]
}
```

作成先：`.github/workflows/test.yml`。更新PRでも、新版の新規構築と同じstateでの旧版→新版更新を実行する。
削除・置換・state移行と要件への影響をレビューし、振る舞い・再plan・後始末が通ってからマージする。
AVMの`version`も明記し、各rootの`.terraform.lock.hcl`をコミットする。

### A.1.6 定期ドリフト検知の設定

作成先：`.github/workflows/drift.yml`。初期案は毎日1回。Azureの検査権限を参照専用とし、差異・検査失敗を通知する。

作成先：`scripts/drift.sh`。最後に本番へ適用した版で、次の検査を順に実行する。

1. デプロイ成功時の記録からcommit・入力・固定版・本番backendを取得する。記録の欠落や不一致は失敗にする。
2. その版を取得して`init -lockfile=readonly`→`plan -detailed-exitcode -out=drift.tfplan`→`show -json drift.tfplan`を実行する。planの0/2を処理し、1は検査失敗にする。
3. 終了コード2だけでなく`resource_drift`も確認し、実効Policy・例外・RBAC・経路と管理外の追加を照合する。
4. 差異・検査不能・対象scopeの欠落を報告する。自動applyやstateへの差分受け入れは行わない。

作成先：`tests/drift/*_test.go`。専用試験環境で手動変更・管理外追加・権限喪失を起こし、検出を確認する。
Policyが正当に作成・変更する属性は所有記録と照合し、重要な差異を`ignore_changes`で隠さない。
検査jobとは別の監視で、未実行と最終成功から26時間の超過も通知する（周期・期限は初期案）。

### A.1.7 サブエージェント定義の編成一式

レビュー担当を `.codex/agents/` に置く。モデルは利用中の設定を継承する。

#### プランレビューエージェント

作成先：`.codex/agents/design-reviewer.toml`。

```toml
name = "design-reviewer"
description = "変更差分とプランドキュメントを照合し、乖離を報告する"
developer_instructions = """
docs/plans/ の対象プランと、docs/rules/ の該当ルールを読む。
渡された変更差分を、受け入れ条件・テストケースと照合する。
不足（条件に対応する変更がない）と過剰（プラン外の変更）を両方向で確認する。
実装の経緯は判断根拠にしない。ファイルは編集しない。
判断に迷う点は「要確認」として分ける。
P0（プランとの矛盾）/ P1（条件の実装漏れ）/ P2（無害なプラン外変更）で報告し、file:line と根拠を添える。
"""
```

#### テストレビューエージェント

作成先：`.codex/agents/edge-case-reviewer.toml`。

```toml
name = "edge-case-reviewer"
description = "変更差分の異常系・境界値とテストの抜けを検査する"
developer_instructions = """
docs/plans/ の対象プラン、docs/rules/testing.md と該当ルールを読む。
渡された変更差分の条件分岐、境界の両側、null・空値、非同期処理の失敗経路を確認する。
削除された関数・変数の参照が残っていないか確認する。
対応するテストが正常系だけでなく、境界値・異常系も検証しているか照合する。
実装の経緯は判断根拠にしない。ファイルは編集しない。
P0（データ破壊・失敗の握り潰し）/ P1（異常系・境界値の抜け）/ P2（改善提案）で報告し、file:line と壊れる入力例を添える。
"""
```

#### セキュリティレビューエージェント

作成先：`.codex/agents/security-reviewer.toml`。

```toml
name = "security-reviewer"
description = "変更差分を入力検証・認可・秘密情報の観点でレビューする"
developer_instructions = """
docs/plans/ の対象プランと、docs/rules/ の該当ルールを読む。
渡された変更差分を、入力検証、認可の抜け、秘密情報の出力の3観点で確認する。
権限の対象scope、外部入力の扱い、ログや設定への秘密値の混入を点検する。
実装の経緯は判断根拠にしない。ファイルは編集しない。
P0（修正必須）/ P1（問題の疑い）/ P2（改善提案）で報告し、file:line と根拠を添える。
"""
```

#### 呼び出し方

担当名を指定し、サブエージェントとしてレビューを依頼する。CLIの `/subagents` で活動を確認する。

並列の委任に対応する環境では3担当を並列に呼び出し、対応しない環境では担当ごとに独立したセッションで実行する。

### A.1.8 スキルの実例

作業手順を `.agents/skills/<スキル名>/SKILL.md` に置き、スキル名を指定して呼び出す。

#### テスト駆動開発の実行手順

テストを1件ずつRed→Green→Refactorで進める。

作成先：`.agents/skills/tdd/SKILL.md`。`$tdd` で呼び出す。

```markdown
---
name: tdd
description: テスト駆動開発で機能追加・バグ修正を実装する。テストリストから1件ずつRed、最小のGreen、Refactorを回す。ユーザーがTDD・テスト駆動・Red-Green-Refactorを指定したときに使う。テストの後付けだけの依頼には使わない。
---

# tdd

## 0. 開始条件

- ルールファイル（testing.md と変更パスに該当するもの）を読む
- 期待する振る舞いは要件・プランを出典にする。実装コードを正解にしない
- 期待結果が曖昧、仕様と実装が矛盾する場合は、推測でテストを書かず停止して報告する

## 1. テストリストを作る

変更後に観測できる振る舞いを列挙する（正常系・境界値の両側・異常系・既存挙動の回帰）。
全件のテストコードを先に書かない。最も小さく設計を前進させる1件だけを選ぶ。

## 2. Red（失敗を実測する）

選んだ1件のテストを書き、対象テストだけを実行して失敗を確認する。
未実装の振る舞いが原因で失敗する状態にする（コンパイルエラーは Red ではない）。
最初から通ってしまったら、実装済みか、assertが弱いかを判定する。

## 3. Green（最小の実装）

その1件を通すのに必要な実装だけを書く。固定値の返却・テスト専用分岐・
assert削除・skipは「最小実装」ではない。同じ失敗を2回繰り返したら、
試行を増やさず前提と設計を見直す。

## 4. Refactor（Greenを保って整理）

すべてGreenの状態でのみ行い、変更のたびにテストを実行する。

## 5. リストが空になるまで繰り返す

作業中に見つけたシナリオはリストへ追加する。完了時は、テストリストの全項目と
実装されたテストを対応づけて報告する。テスト通過だけを完了証明にしない。
```

#### create-pr（PR本文を構造化して投稿する）

変更内容と検証結果をまとめてPRを作成する。

作成先：`.agents/skills/create-pr/SKILL.md`。`$create-pr` で呼び出す。

```markdown
---
name: create-pr
description: 実装が一段落したら、PR本文を課題・解決策・テスト結果・レビュー観点の形に整形して下書きし、承認を取ってからGitHubへ投稿する。「PRを作って」と頼まれたときに使う。
---

# create-pr（PR本文を構造化して投稿する）

## 1. 差分と経緯を集める

- `git diff main...HEAD` で変更の全体を把握する
- 対応するプラン（docs/plans/）と、実装後のAIレビュー結果を読む

## 2. 本文を下書きする

次の見出しで、レビュアーが上から読めば理解できる順に置く。

- 課題：何を解決するPRか（背景と、対応するissue・プラン）
- 解決策：どう変えたか。設計判断とトレードオフを1〜2行で
- テスト：追加・変更したテストと、手元での実行結果
- レビュー観点：とくに見てほしい箇所、影響範囲、未対応事項

下書きは docs/pr/ に保存し、投稿前に人間の承認を取る。

## 3. 投稿する

承認後、`gh pr create` で本文を渡して作成する。CIとAI相互レビューが
回るので、その結果が出てから人間のレビューへ回す。
```

---

## A.2 テスト装置の実物一式

### A.2.1 単体テストの実物

作成先：`tests/unit/network_inputs_test.go`。実際の試験用入力を読み、Hub・Spoke・オンプレのアドレスが重複しないことを確認する（Goの例はimport宣言を省略）。

```go
func TestNetworkInputsDoNotOverlap(t *testing.T) {
	data, err := os.ReadFile("../../environments/e2e/platform.tfvars.json")
	require.NoError(t, err)
	var input struct {
		HubCIDR    string            `json:"hub_cidr"`
		SpokeCIDRs map[string]string `json:"spoke_cidrs"`
		OnpremCIDR string            `json:"onprem_cidr"`
	}
	require.NoError(t, json.Unmarshal(data, &input))
	require.NotEmpty(t, input.SpokeCIDRs)
	cidrs := []string{input.HubCIDR, input.OnpremCIDR}
	for _, cidr := range input.SpokeCIDRs {
		cidrs = append(cidrs, cidr)
	}
	var prefixes []netip.Prefix
	for _, cidr := range cidrs {
		prefix, err := netip.ParsePrefix(cidr)
		require.NoError(t, err, cidr)
		prefixes = append(prefixes, prefix)
	}
	for i := range prefixes {
		for j := i + 1; j < len(prefixes); j++ {
			require.False(t, prefixes[i].Overlaps(prefixes[j]),
				"アドレスが重複: %s / %s", cidrs[i], cidrs[j])
		}
	}
}
```

### A.2.2 E2Eテストの実物

作成先：`tests/e2e/platform_test.go`。実Azureへのデプロイ結果、Policyの動作、再planの差分を確認する。

```go
func TestPlatformE2E(t *testing.T) {
	assertTestTarget(t)
	opts := terraformOptions(t, "../../infra/platform", "platform", "TEST_TFVARS")
	terraform.InitAndApply(t, opts)
	// 基盤全体の削除は独立cleanup jobが担当する（A.2.4）。

	t.Run("Bastionのサブネットが設計どおり", func(t *testing.T) {
		id := terraform.Output(t, opts, "bastion_subnet_id")
		actual := az(t, "network", "vnet", "subnet", "show", "--ids", id,
			"--query", "addressPrefixes || [addressPrefix]", "-o", "json")
		require.JSONEq(t, `["10.100.1.0/26"]`, actual)
	})
	t.Run("DenyがStorageの公開アクセスを拒否する", testDenyPublicStorage)
	t.Run("DINEが診断設定を自動配置する", testDeployDiagnostics)
	t.Run("再planに差分がない", func(t *testing.T) {
		require.Equal(t, 0, terraform.PlanExitCode(t, opts))
	})
}
```

作成先：`tests/e2e/policy_test.go`。許可・拒否と、診断設定の自動配置を確認する。

```go
func testDenyPublicStorage(t *testing.T) {
	opts := terraformOptions(t, "../../infra/probes/storage-policy", "deny", "TEST_PROBE_TFVARS")
	opts.Vars = map[string]interface{}{"public_network_access_enabled": false}
	defer terraform.Destroy(t, opts)
	terraform.InitAndApply(t, opts) // 許可側が成功することを先に確認

	assignmentID := requiredEnv(t, "PUBLIC_STORAGE_DENY_ASSIGNMENT_ID")
	definitionID := requiredEnv(t, "PUBLIC_STORAGE_DENY_DEFINITION_ID")
	require.Eventually(t, func() bool {
		// Policy反映前に成功しても、許可状態へ戻してから再試行する。
		opts.Vars["public_network_access_enabled"] = false
		if _, err := terraform.ApplyE(t, opts); err != nil {
			return false
		}
		opts.Vars["public_network_access_enabled"] = true
		_, err := terraform.ApplyE(t, opts)
		return err != nil && strings.Contains(err.Error(), "RequestDisallowedByPolicy") &&
			strings.Contains(err.Error(), assignmentID) && strings.Contains(err.Error(), definitionID)
	}, 15*time.Minute, 30*time.Second, "公開アクセスが対象Policyで拒否されること")
}

func testDeployDiagnostics(t *testing.T) {
	opts := terraformOptions(t, "../../infra/probes/storage-policy", "dine", "TEST_PROBE_TFVARS")
	opts.Vars = map[string]interface{}{"public_network_access_enabled": false}
	defer terraform.Destroy(t, opts)
	terraform.InitAndApply(t, opts) // probeは診断設定を自分で作らない

	targetID := terraform.Output(t, opts, "diagnostic_target_id")
	expected := requiredEnv(t, "EXPECTED_LOG_WORKSPACE_ID")
	require.Eventually(t, func() bool {
		out, err := azE(t, "monitor", "diagnostic-settings", "list", "--resource", targetID,
			"--query", "value[].workspaceId", "-o", "json")
		if err != nil {
			return false
		}
		var actual []string
		return json.Unmarshal([]byte(out), &actual) == nil && slices.Contains(actual, expected)
	}, 30*time.Minute, 20*time.Second, "DINEが指定workspace宛ての診断設定を配置すること")
}
```

作成先：`tests/e2e/upgrade_test.go`。旧版から候補版へ同じstateを更新し、削除・再作成なしでPolicyが動き続けることを確認する。

```go
func TestUpgrade(t *testing.T) {
	assertTestTarget(t)
	opts := terraformOptions(t, requiredEnv(t, "BASELINE_TERRAFORM_DIR"), "upgrade", "TEST_TFVARS")
	terraform.InitAndApply(t, opts)
	opts.TerraformDir = requiredEnv(t, "CANDIDATE_TERRAFORM_DIR")
	opts.PlanFilePath = filepath.Join(t.TempDir(), "upgrade.tfplan")
	plan := terraform.InitAndPlanAndShowWithStruct(t, opts)
	for address, change := range plan.ResourceChangesMap {
		require.False(t, change.Change.Actions.Delete() || change.Change.Actions.Replace(), address)
	}
	terraform.Apply(t, opts) // 確認したplanを適用
	opts.PlanFilePath = ""
	t.Run("更新後もDenyが働く", testDenyPublicStorage)
	t.Run("更新後もDINEが働く", testDeployDiagnostics)
	require.Equal(t, 0, terraform.PlanExitCode(t, opts))
}
```

CIは旧版・候補版の全体を別worktreeへ取得し、上記2変数に`infra/platform`の絶対パスを設定する（lockファイル・`lib/`を含む）；置換やstate移行が必要な更新は専用の移行試験を追加する。

### A.2.3 テスト基盤の実物

作成先：`tests/e2e/support_test.go`。認証先・state・Azureからの値取得を共通化する。

<details>
<summary>共通処理のコード</summary>

```go
func requiredEnv(t *testing.T, key string) string {
	t.Helper()
	value := os.Getenv(key)
	require.NotEmpty(t, value, "試験設定が必要: %s", key)
	return value
}

func assertTestTarget(t *testing.T) {
	t.Helper()
	require.Equal(t, requiredEnv(t, "TEST_TENANT_ID"), az(t, "account", "show", "--query", "tenantId", "-o", "tsv"))
	var pool []string
	require.NoError(t, json.Unmarshal([]byte(requiredEnv(t, "TEST_SUBSCRIPTION_IDS_JSON")), &pool))
	require.True(t, slices.Contains(pool, az(t, "account", "show", "--query", "id", "-o", "tsv")))
}

func terraformOptions(t *testing.T, dir, stateName, varFileEnv string) *terraform.Options {
	t.Helper()
	return &terraform.Options{
		TerraformDir: dir,
		VarFiles:     []string{requiredEnv(t, varFileEnv)},
		BackendConfig: map[string]interface{}{
			"key": "e2e/" + requiredEnv(t, "TEST_RUN_KEY") + "/" + stateName + ".tfstate",
		},
		Reconfigure: true, NoColor: true, SetVarsAfterVarFiles: true, Lock: true,
	}
}

func azE(t *testing.T, args ...string) (string, error) {
	t.Helper()
	return shell.RunCommandAndGetStdOutE(t, shell.Command{
		Command: "az", Args: args, Logger: logger.Discard,
	})
}

func az(t *testing.T, args ...string) string {
	t.Helper()
	out, err := azE(t, args...)
	require.NoError(t, err)
	return strings.TrimSpace(out)
}
```

</details>

試験用Terraformは`infra/probes/storage-policy/`に置き、入力`public_network_access_enabled`と出力`diagnostic_target_id`を用意し、診断設定はDINEに作らせる。

### A.2.4 テストの実行

作成先：`scripts/test.sh`。単体・結合・実Azure E2E・依存更新を別コマンドにする（planは実Azure試験で確認）。

```bash
set -euo pipefail
case "${1:-unit}" in
  unit) go test -v -count=1 ./tests/unit ;;
  integration)
    terraform fmt -check -recursive infra
    for dir in infra/bootstrap infra/platform infra/probes/storage-policy; do
      terraform -chdir="$dir" init -backend=false
      terraform -chdir="$dir" validate
    done ;;
  e2e) go test -v -count=1 -parallel=1 -timeout=4h -run '^TestPlatformE2E$' ./tests/e2e ;;
  upgrade) go test -v -count=1 -parallel=1 -timeout=4h -run '^TestUpgrade$' ./tests/e2e ;;
  *) exit 2 ;;
esac
```

Go 1.23・Terratest `v0.48.2`・testify `v1.10.0`を使用し、依存はリポジトリ直下の`go.mod`・`go.sum`で管理する。

CIは絶対パスの入力・backend設定・UAMI認証・pool排他・所有記録を準備し、各実Azure試験の後に独立jobでprobe削除→subscription退避→基盤削除→残存確認を行い、poolを解放してから次の試験を実行する。

固定基盤と回線を保持し、最新commitの試験・後始末の成功をマージ条件にする（実AzureのE2Eは未実行）。

## A.3 プランドキュメントの完全版

作成先：`docs/plans/platform-landing-zone.md`。要件・設計・テスト・実装順を8項目にまとめる。

```markdown
# プラン: Azure共通基盤の構築・継続検証

状態: レビュー中 | 元資料: docs/requirements/platform-landing-zone.md

## 1. 概要

公式AVMを部品として、管理グループ・Policy・RBAC・監視・共通ネットワークを構築する。
ALZ Acceleratorは使わず、構成・CI/CD・テストを実装する。
依存更新と実環境のドリフトも継続して検証する。

## 2. 背景

構築結果と制御の動作を保証し、外部モジュール更新や手動変更による不整合を検出する。
受け入れ条件はD-001・P-001・N-001・C-001と、POL-ADD-001・UPD-001・DRIFT-001。
EA契約、UAMIへのOIDCフェデレーション、東日本の単一リージョンを前提とする。

## 3. 詳細設計

変更するファイル・配置先:

- `infra/bootstrap/` — 常設のID・backend等。試験時の削除対象から分離する。
- `infra/platform/governance.tf` — ALZ AVMで階層・Policy・RBACを構成する。
- `infra/platform/network.tf` — Hub-Spoke、Firewall、DNS、Bastion、ExpressRoute。
- `infra/platform/monitoring.tf` — 管理監視基盤と、検査結果の鮮度監視。
- `infra/platform/policy_exemptions.tf` — 理由・責任者・期限付きの例外。
- `infra/platform/variables.tf`・`outputs.tf` — 入力検証と必要な出力。
- `infra/platform/providers.tf`・`versions.tf` — 接続先とTerraform・providerの制約。
- `infra/probes/storage-policy/` — Policy試験対象だけを作り、別stateで削除する。
- `environments/production/`・`environments/e2e/` — `platform.tfvars.json`と`backend.hcl`。同じplatformコードを使う。
- `lib/` — ALZ形式の独自定義・割当・archetype差分・architecture。
- `tests/unit/*_test.go` — Go＋testifyで入力・ライブラリ資産を検証する。
- `tests/e2e/`・`tests/drift/` — 構築・更新・振る舞いと、読み取り検査の試験。
- `scripts/test.sh`・`scripts/drift.sh`・`scripts/cleanup.sh` — 実行・検査・回収の入口。
- `.github/workflows/test.yml`・`drift.yml`、`renovate.json` — PR検証・定期検査・更新PR。

判断済みの論点:

- 1ファイル1責務とし、AVMは直接組み合わせる。設定だけを包む独自モジュールは作らない。
- platformは一つのrootとし、本番・試験runごとにstateを分離する。固定fixtureは含めない。
- ALZへはplan時に確定するIDを渡し、公式の依存入力で作成順を制御する。
- ALZ標準は固定版を参照し、独自資産を別名で追加する。archetype overrideはarchitectureから参照する。
- providerは各rootのlockファイル、AVMはmoduleのversion、ALZ Libraryは参照版を固定する。
  `.terraform.lock.hcl`はAVMを固定しない。更新PRで削除・置換・state移行も確認する。
- Management・Connectivity・Identity・Securityのsubscriptionを分け、公式ALZ階層を基準にする。
- Firewall Standard、Bastion Standard、ErGw1AZを使う。VPNと有償DDoS Network/IP Protectionは無効。
- Hubは10.100.0.0/16、Spokeは10.101.0.0/16・10.102.0.0/16、オンプレはサンプル172.20.0.0/16。
  重複と必要なサブネット条件を検証し、Spoke間・オンプレ接続はFirewall経由の明示許可だけを通す。
- 別テナントの試験poolを排他利用し、開通済み回線・subscription・ID・backendを保持する。
  Gateway・Connection等の今回作成物だけを削除する。ALZの削除時移動設定でsubscriptionを退避し、残存を確認する。
- ドリフト検査は本番へ最後に適用したcommit・入力・固定版で行う。latest mainや更新候補版は使わない。
  plan・resource_driftとAzureの実効設定・台帳を照合し、差異と観測失敗を分けて報告する。
  検査でapply・state更新・期待値の自動書換えは行わない。

## 4. テスト影響範囲

- Library・独自Policy変更時は割当・parameter・reference IDと必須ケースの対応を確認する。
- 依存更新では同じ要件を使い、新規構築と旧版から新版への更新の両方を検証する。
- 経路・DNS・cleanup変更は通信試験、固定fixture保持、部分失敗からの回収も再検証する。

## 5. 新規テストケース

- 入力の不正・アドレス重複・サブネット条件と、AVM間の値の受け渡しを検証する。
- 階層・subscription配置・RBAC・Policy・監視・ネットワークの実値を要件と照合する（D-001）。
- 全有効Deny/DenyAction・Modify・Append・DINEの動作を検証する（P-001）。
  許可・拒否を対にし、DINEは新規自動配置と既存不適合の修復を確認する。Audit系は構成を照合する。
- 独自Policyの追加・変更・例外と、標準ALZとの衝突・意図しない制御欠落を検出する（POL-ADD-001）。
- 許可通信・遮断、Firewall経由の証跡、DNS転送・Bastion接続を確認する（N-001）。
- 再planの差分なし、成功・部分失敗後の回収、固定fixture保持を確認する（C-001）。
- 新版での新規構築と、旧版を作成したstateを引き継ぐ新版更新を通し、未承認の削除・置換を検出する（UPD-001）。
- 試験環境で構成変更・Policy例外・管理外リソース追加を起こし、ドリフト検出を確認する。
  権限不足・API失敗・未実行・結果の期限超過も異常として検出する（DRIFT-001）。

## 6. 実装順

1. 要件とケース台帳を書く。定型的なケース整理はサブエージェントへ委任する。
2. 入力検証・階層・Policy・監視: 失敗する単体/結合/E2E→最小の構成（主担当）。
3. ネットワーク・独自Policy: 失敗する通信/制御試験→構成、回収まで通す（主担当）。
4. 旧版→新版更新と意図的なドリフトの失敗試験→更新・監視フローを実装する（主担当）。
5. 設計・テスト・セキュリティの独立レビューでP0/P1を修正する。

## 7. コミット前テスト実行

- `terraform fmt -check -recursive`と各rootの`terraform validate`。
- `bash scripts/test.sh unit` / `bash scripts/test.sh integration` — 全単体・結合。
- `bash scripts/test.sh e2e` / `bash scripts/test.sh upgrade` — 実Azureでの構築・更新・振る舞い。
- `bash scripts/drift.sh` — 読み取り専用の検査。検出機構の故障試験は専用試験環境だけで行う。
- 各PRの最新commitで必須試験と独立cleanupが成功してからマージする。未実行を合格にしない。
  1回1万円・主処理4時間＋cleanup2時間は初期案。更新試験を含む費用・時間はpilotで見直す。

## 8. スコープ外

- アプリケーション開発、subscription自動払い出し、ALZ Accelerator。
- 複数リージョン、VPN、有償DDoS、Firewall Premium機能。
- 固定fixtureの削除、回線事業者の開通・廃止、ドリフト検査による自動修復。
```

## A.4 8ステップを回すためのプロンプト

以下の依頼文をCodexのCLI／IDEの会話へ渡す。対象パスと、`docs/plans/` のプランを指定して使う。

### A.4.1 1. プランドキュメントの作成（インタビュー式）

```text
[機能名]を作りたい。概要：[2〜3行で]。
参考：docs/requirements/[要件定義ファイル]と、関連コード[ディレクトリ]。

まず実装に入らず、プランを書くために必要な質問を私にすること。
技術面・エッジケース・トレードオフについて、答えにくい質問ほど価値がある。
質問が尽きたら、プランを docs/plans/[ファイル名].md に
8項目（概要・背景・詳細設計・テスト影響範囲・新規テストケース・実装順・コミット前テスト実行・スコープ外）で書き出すこと。
実装順には、どのタスクをどのモデル・サブエージェントに任せるかの割り当てまで書くこと。
```

### A.4.2 2. プランのAI相互レビュー（観点別）

```text
docs/plans/[プランファイル].md をレビューすること。私は書いた本人ではない前提で読む。
観点: [設計整合性/セキュリティ/運用/ビジネス整合性 から1つ]。
判定の基準: [docs/rules/の該当ルール または 要件定義ファイル]。
指摘はP0（実装前に必ず直す）/P1（推奨）/P2（軽微）に分類し、
プランのどの記述が・どの基準と・なぜずれるかを1件ずつ示すこと。
```

### A.4.3 3. テスト網羅のレビュー

```text
docs/plans/[プランファイル].md の「新規テストケース」だけをレビューすること。
要件定義の受け入れシナリオと突き合わせ、
(1) シナリオに対応するテストが欠けていないか
(2) 境界値の両側・異常系が入っているか
(3) 実装しないと決めたスコープ外がテストに紛れていないか
を確認し、足りないテストケースを提案すること。
```

### A.4.4 4. 実装

```text
docs/plans/[プランファイル].md の実装順に従って実装すること。
プランの割り当てどおり、[調査/ユーティリティ]はサブエージェントに任せてよい。
プランにない変更が必要になったら、実装せずに立ち止まって報告すること。
```

### A.4.5 5. テストの実装と全通過

```text
プランの「新規テストケース」をテストコードに落とすこと。
各テストは、先に実行して失敗（Red）を確認してから実装を直す。
最初から通ってしまったテストは、実装済みかassertが弱いかを判定して報告する。
最後に[コミット前テスト実行のコマンド]を実行し、全通過の出力を見せること。
```

### A.4.6 6. 実装のAI相互レビュー（観点別）

```text
実装が終わったら、A.1.7の方法で次の3担当に独立してレビューを依頼し、
報告をP0からP2に整理して返すこと。
- design-reviewer: プランドキュメントとの乖離
- edge-case-reviewer: 異常系と境界値の抜け
- security-reviewer: セキュリティ観点
```

### A.4.7 7. 人間のレビュー（AIへの壁打ち）

```text
実装した[機能名]について、レビューする私に説明すること。判定はしなくてよい。
1. 変更の要点を、データの流れがわかる粒度で説明する。
2. 設計判断を3つまで挙げ、それぞれ採らなかった選択肢と併記する。
3. 各判断の根拠が docs/plans/[プランファイル].md のどこにあるかを引用して示す。
   プランに書いていないものは「実装時の判断」と明示する。
4. 壊れたときに影響が大きい箇所を、理由とともに挙げる。
根拠を確認できないものは推測で埋めず「不明」と答えること。
```

### A.4.8 8. テスト駆動リファクタリング

```text
[対象ディレクトリ]をリファクタリングすること。機能の変更はしない。
先に対象のテストを実行して全面Greenであることを確認し、
そこから1変更ごとにテストを回して、Greenを保ったまま進めること。
観点: 1ファイル1責務に反する肥大したファイルの分割、
重複した処理の集約、docs/rules/[アーキテクチャルール]との乖離の解消。
テストが赤くなったら、その変更を巻き戻して報告すること。
```

公式仕様：[指示ファイル](https://developers.openai.com/codex/guides/agents-md)／[サブエージェント](https://developers.openai.com/codex/subagents)／[スキル](https://developers.openai.com/codex/skills)。対応版を使用し、モデル・権限は利用環境側で設定する。

