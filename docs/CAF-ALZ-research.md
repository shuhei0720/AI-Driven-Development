# CAF/ALZ 調査・ヒアリング記録

この記録はサンプル作成前の調査経過を保存したもの。確定した構成と実践例は [AI-Driven-Development-CAF-ALZ.md](../AI-Driven-Development-CAF-ALZ.md) を参照する。「次に決める」「これから改稿する」等の文言は調査当時の記録である。

## CAF/ALZ調査メモ

調査日：2026-11-03。Microsoft Learn、Azure公式ドキュメント、Azure公式GitHubのREADME・Terraform定義・releaseを確認した。

### 今回の方針

- 題材はCAFに沿ったAzureのPlatform landing zone（共通基盤）。
- 新規環境への構築を想定する。
- Acceleratorによる生成・bootstrapは採用せず、公式実装を参考に自分たちのTerraform構成・CI/CD・テストを作る。公式AVMを部品として利用する範囲はヒアリングで決める。
- テスト駆動開発を前提とし、受け入れ条件とテストケースの確認を実装より先に行う。
- 今回は情報収集まで。A.1〜A.5のALZ向け改稿とTerraform実装は次の段階で行う。
- 既存会話のEA契約・ユーザー割り当てマネージドIDのフェデレーションは前提として記録する。Platform用のサブスクリプション構成と実行権限は設計時に具体化する。

### 1. CAFと今回の対象

CAFは設計・導入の指針、Azure Landing Zonesはそれに沿う基盤の参照アーキテクチャである。今回扱うPlatform landing zoneは、管理グループ、統制、共通ネットワーク、監視などの共通基盤に当たる。各業務システムを配置するWorkload landing zoneとは範囲を分ける。

参照：[Azure landing zoneの概要](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)

設計では、①請求・テナント、②ID・アクセス、③リソース構成、④ネットワーク、⑤セキュリティ、⑥管理、⑦ガバナンス、⑧プラットフォーム自動化・DevOpsの8領域を確認する。公式コードを適用するだけで、組織固有の設計判断が完了するわけではない。

参照：[8つの設計領域](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas)

### 2. 現行の公式実装

Microsoftの推奨はALZ IaC AcceleratorとAzure Verified Modules（AVM）を使う方式。AVMを単独で組み合わせる構成も公式に案内されている。

参照：[Platform landing zoneの実装方式](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/implementation-options)、[ALZ Terraformのモジュール構成](https://azure.github.io/Azure-Landing-Zones/terraform/)

| 公式リポジトリ | 役割 | 今回読む箇所 |
| --- | --- | --- |
| [Azure/alz-terraform-accelerator](https://github.com/Azure/alz-terraform-accelerator) | AVMを組み合わせるTerraformスターター | `templates/platform_landing_zone/` |
| [Azure/terraform-azurerm-avm-ptn-alz](https://github.com/Azure/terraform-azurerm-avm-ptn-alz) | 管理グループ、Policy、ロール、サブスクリプション配置 | README、examples、入力と制約 |
| [Azure/terraform-azurerm-avm-ptn-alz-management](https://github.com/Azure/terraform-azurerm-avm-ptn-alz-management) | Log Analytics等の管理・監視基盤 | README、examples |
| [Azure/terraform-azurerm-avm-ptn-alz-connectivity-hub-and-spoke-vnet](https://github.com/Azure/terraform-azurerm-avm-ptn-alz-connectivity-hub-and-spoke-vnet) | Hub-Spoke型ネットワーク | README、examples、DNS・Firewall等の設定 |
| [Azure/terraform-azurerm-avm-ptn-alz-connectivity-virtual-wan](https://github.com/Azure/terraform-azurerm-avm-ptn-alz-connectivity-virtual-wan) | Virtual WAN型ネットワーク | Hub-Spokeとの選択に使う |
| [Azure/Azure-Landing-Zones-Library](https://github.com/Azure/Azure-Landing-Zones-Library) | アーキテクチャ、archetype、Policy、ロールの定義データ | 標準構成とカスタマイズの境界 |
| [Azure/ALZ-PowerShell-Module](https://github.com/Azure/ALZ-PowerShell-Module) | Azure・GitHub等のデプロイ基盤をbootstrap | Acceleratorを使う場合の導入手順 |

旧[Azure/terraform-azurerm-caf-enterprise-scale](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale)は、調査時点でアーカイブ済み。公式READMEも非推奨・更新終了とAVMへの移行を明記しているため、新規構築の基準はAVM側とする。

### 3. 公式コードの参照バージョン

確認したAcceleratorの最新releaseは[v17.6.0](https://github.com/Azure/alz-terraform-accelerator/releases/tag/v17.6.0)（2026-10-06）。公式コード比較の参照として、このtagの[Platform landing zoneスターター](https://github.com/Azure/alz-terraform-accelerator/tree/v17.6.0/templates/platform_landing_zone)を記録する。自作版で採用するprovider・moduleのバージョンは、テスト要件と合わせて別途決める。

| 定義 | v17.6.0で確認した指定 |
| --- | --- |
| Terraform | `~> 1.12` |
| AzureRM provider | `~> 4.0` |
| AzAPI provider | `~> 2.0` |
| ALZ provider | `~> 0.21` |
| 管理グループ・PolicyのAVM | `0.21.0` |
| 管理・監視のAVM | `0.9.0` |
| Hub-Spoke／Virtual WANのAVM | いずれも`0.17.2` |

参照：[terraform.tf](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/terraform.tf)、[management groups](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/main.management.groups.tf)、[management resources](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/main.management.resources.tf)、[Hub-Spoke](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/main.connectivity.hub.and.spoke.virtual.network.tf)、[Virtual WAN](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/main.connectivity.virtual.wan.tf)

各AVMの単体最新releaseは、このスターターの固定バージョンと異なる。まずスターター側の組み合わせを基準に読み、更新時に互換性と変更内容を確認する。

スターターは、事前に準備した管理・接続等のサブスクリプションIDを受け取る。新規環境でも、サブスクリプションを準備する工程とPlatformの構成を適用する工程を分けて設計する。

参照：[variables.tf](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/variables.tf)、[サブスクリプションと権限の前提](https://azure.github.io/Azure-Landing-Zones/accelerator/1_prerequisites/platform-subscriptions/)

### 4. 公式コードの読み方

| 順序 | ファイル・ディレクトリ | 調べること |
| --- | --- | --- |
| 1 | `README.md`、`examples/` | 配置範囲、Hub-Spoke／Virtual WAN等のシナリオ |
| 2 | `terraform.tf`、`variables.tf` | provider、backend、実行先のサブスクリプション |
| 3 | `main.management.groups.tf` | 管理グループ階層とPolicyの接続 |
| 4 | `main.management.resources.tf` | 管理・監視の共通リソース |
| 5 | `main.connectivity.*.tf` | ネットワーク構成とproviderの使い分け |
| 6 | `main.config.tf`、`lib/` | 構成値の展開、ALZ Libraryとカスタマイズ |

この責務分割を、自作版のTerraform構成と、書籍の「1ファイル1責務」「機能境界」「テスト」「プラン」の例を設計する参考にする。

### 5. 実装方式の決定

採用方針は、自分たちでTerraformの構成・CI/CD・テストを実装する方式。以下は公式の選択肢を調べた記録である。

| 方式 | 特徴・前提 |
| --- | --- |
| Acceleratorでbootstrap | Azure側の実行基盤とGitHub／Azure DevOpsのCI/CDを準備する。公式GitHub bootstrapはOrganizationが必要 |
| スターターを直接参照 | 公式READMEは単独利用とtfvarsの例を案内している。state・認証・CI/CDを利用側で設計する |
| AVMを直接組み合わせる | 必要な共通基盤だけを選んで構成できる。モジュール間の依存と設定を利用側で設計する |

参照：[Accelerator](https://azure.github.io/Azure-Landing-Zones/accelerator/)、[スターターREADME](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/templates/platform_landing_zone/README.md)

公式GitHub bootstrapは個人アカウントに対応していない。Free Organizationではpublic repositoryを作るとの記載もある。今回は標準bootstrapを使わないため、このOrganization要件を自作方式の必須条件にはしない。自作CIで使用するGitHub機能の要件は別途確認する。

参照：[GitHubの前提条件](https://azure.github.io/Azure-Landing-Zones/accelerator/1_prerequisites/github/)

### 6. 次に決める要件

1. Platform用サブスクリプションと管理グループ階層。新規環境が既存Entraテナント内かどうか。
2. 対象リージョンと、単一／複数リージョンの範囲。
3. Hub-Spoke／Virtual WAN、オンプレ接続、Firewall・DNS等の採用範囲。
4. 標準Policy・RBAC・監視の採用範囲と例外。
5. GitHub Organizationの利用、state保存先、bootstrapと継続デプロイの権限分離。

初期構成を選ぶ公式シナリオは、[計画ガイド](https://azure.github.io/Azure-Landing-Zones/accelerator/0_planning/)で確認する。管理グループ・Policy・管理リソースだけの構成や、単一リージョン構成もあり、最初から全機能を採用する必要はない。

### 7. 公式リポジトリのテスト実装

調査対象はコードとCIの定義。公式のAzure試験自体は実行していない。ALZ coreは参照commit `e2a318c3eba91967c4ce3cd2738b2b2569db994e`、共通ツールは `790f00b5bb824ce381ae3cddb81d5fd1976a9478` を記録する。前節のスターター固定版とは別の調査スナップショットである。

| 層 | 実物で確認したこと | 出典 |
| --- | --- | --- |
| 構成の単体テスト | `mock_provider`＋`command = plan`＋`assert`／`expect_failures`。管理グループ設定の作成・更新・未作成、名前の境界と不正値を検証 | [hierarchysettings.tftest.hcl](https://github.com/Azure/terraform-azurerm-avm-ptn-alz/blob/e2a318c3eba91967c4ce3cd2738b2b2569db994e/tests/unit/hierarchysettings.tftest.hcl) |
| Policy側の回帰試験 | ALZ providerでローカルLibraryを解決し、Azure操作側をmock。Policy用RBACを無効にしてもPolicy本体は残ること等を確認 | [policyroleassignmentsenabled.tftest.hcl](https://github.com/Azure/terraform-azurerm-avm-ptn-alz/blob/e2a318c3eba91967c4ce3cd2738b2b2569db994e/tests/unit/policyroleassignmentsenabled.tftest.hcl) |
| ネットワークの単体テスト | 主リージョン指定・未指定・複数指定の拒否、入力の伝播・省略・相反入力などを検証 | [Hub-Spoke is_primary](https://github.com/Azure/terraform-azurerm-avm-ptn-alz-connectivity-hub-and-spoke-vnet/blob/main/tests/unit/is_primary.tftest.hcl) |
| 実Azureの結合テスト | ALZ coreの複数ロール定義は実applyを2回行う回帰試験。該当ファイルには明示的assertがない | [multipleroledefs.tftest.hcl](https://github.com/Azure/terraform-azurerm-avm-ptn-alz/blob/e2a318c3eba91967c4ce3cd2738b2b2569db994e/tests/integration/multipleroledefs.tftest.hcl) |
| AVM共通E2E | examplesごとにinit→実apply→再planの差分ゼロ確認→destroy。失敗時の後始末とpost hookを用意 | [E2E runner](https://github.com/Azure/azure-verified-modules-tools/blob/790f00b5bb824ce381ae3cddb81d5fd1976a9478/src/Avm.Authoring/Engines/Terraform/Invoke-AvmTerraformTestE2e.ps1) |
| 静的検査 | format・lint・convention・validate・docs等を検査する。構成の振る舞いを検証するテストとは役割が異なる | [PR check](https://github.com/Azure/azure-verified-modules-tools/blob/790f00b5bb824ce381ae3cddb81d5fd1976a9478/src/Avm.Authoring/Public/Invoke-AvmPrCheck.ps1) |

公式の共通CIはPowerShellのAvm.Authoringを使い、`avm test unit`／`integration`／`e2e`を呼ぶ。単体・結合の内部では `terraform test -test-directory=tests/unit` または `tests/integration`を実行する。単体jobはAzure認証情報を渡さず、実試験はOIDCと専用subscription poolを使用する。これは調査した公式の仕組みであり、自作CIのツール選定は要件から行う。

参照：[AVM共通CI](https://github.com/Azure/azure-verified-modules-tools/blob/790f00b5bb824ce381ae3cddb81d5fd1976a9478/.github/workflows/terraform-module.yml)

確認時の注意：

- mock付きの`apply`もあるため、コマンド名だけで実Azure試験と判断しない。[vWANの例](https://github.com/Azure/terraform-azurerm-avm-ptn-alz-connectivity-virtual-wan/blob/main/tests/unit/gateway_connection_dpd_timeout_seconds.tftest.hcl)
- Managementの`tests/examples.tftest.hcl`はplan2件で、明示assert・mockがない。このファイルを詳細な単体検証と扱わない。[実ファイル](https://github.com/Azure/terraform-azurerm-avm-ptn-alz-management/blob/main/tests/examples.tftest.hcl)
- 共通CIに結合jobが存在しても、各リポジトリに同じtierのテスト実体があるとは限らない。
- 公式E2Eで確認した作成・冪等性・削除の成功だけでは、通信疎通、Policyの拒否、RBACの実効性まで確認したことにはならない。
- Accelerator側のシナリオ組み合わせテストも参照した。PR向けplanと、条件付きapply・後始末を分けている。AVM共通E2Eとは別の実装である。[workflow](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/.github/workflows/end-to-end-test.yml)、[runner](https://github.com/Azure/alz-terraform-accelerator/blob/v17.6.0/.github/tests/scripts/terraform-run.ps1)

### 8. 自作版のTDDの進め方

1. ヒアリングで「許可すること・拒否すること・維持すること」を受け入れ条件にする。
2. 正常系・境界・異常系・再適用・後始末をテストケースへ落とし、人間が内容を確認する。
3. 1件ずつテストを書き、意図した理由でRedになることを確認してから最小実装する。
4. Greenを保って整理し、対応するテストと実行結果を記録する。
5. mockで保証する範囲と実Azureで保証する範囲を分け、Policy・RBAC・通信等の実効性も必要に応じて検証する。

テスト基盤の第一候補はTerraform Test。追加の実状態・通信検証の方法は、受け入れ条件から選ぶ。テストの期待値は実装コードや公式サンプルの出力ではなく、合意した要件を出典にする。専用サブスクリプションだけでなく、試験用の親管理グループ・権限・課金・後始末の範囲もヒアリングする。

### 9. ヒアリング記録

| 項目 | 合意した内容 |
| --- | --- |
| 開発の進め方 | テスト駆動。合格条件の中心は、意図した構成のデプロイとPolicyの実効性 |
| 実装方式 | 公式AVMを部品として利用し、Terraformの構成・CI/CD・テストを自作。Acceleratorによる生成・bootstrapは使わない |
| 対象 | 新規のPlatform Landing Zone全体。管理グループ・Policy・RBAC・共通ネットワーク・監視 |
| リージョン | 東日本（japaneast）の単一リージョン |
| 管理グループ | 公式ALZ標準を基準。Management・Connectivity・Identity・Securityのサブスクリプション分割を含める |
| Policy | 公式ALZ標準セットを基準にする。拒否・変更・自動配置などを行う効果（Deny・DINE等）の実効性試験を必須にする |
| ネットワーク | Hub-Spoke。Azure Firewall・Private DNS・DNS Private Resolver・Azure Bastion・ExpressRouteを含める |
| 通信経路 | Spoke間とオンプレ↔SpokeはFirewallを経由し、明示的に許可した通信だけを通す |
| 初期SKU | Azure Firewall Standard、Azure Bastion Standard、ExpressRoute Gateway ErGw1AZ |
| 有償DDoS | 初期版ではNetwork Protection・IP Protectionを無効にする |
| ExpressRouteの試験 | 開通済み試験回線を保持し、Azure側のGateway・Connection等を毎回作成・削除する |
| E2E環境 | 別テナントに試験用基盤を用意し、試験時に作成、検証後に削除する |
| E2Eのゲート | PRごとに実Azureで実行し、成功をマージ条件にする |
| 固定の土台 | 試験テナント・サブスクリプション・実行用ID・state保存先・開通済み回線は保持し、その中に今回作成したALZリソースを削除する |
| アドレス | 次節のHub・2本の試験用Spoke・オンプレ側のサンプルアドレスを採用 |

残る設計値は、ログ保持・通知、採用Policyごとの試験fixtureなど。費用・時間の初期上限案は第13節に記録する。実環境のIDやアドレスの重複確認は、試験環境設定時に扱う。

### 10. 合意したサンプルアドレス案

| VNet・用途 | CIDR |
| --- | --- |
| Hub VNet | `10.100.0.0/16` |
| Hub / AzureFirewallSubnet | `10.100.0.0/26` |
| Hub / AzureFirewallManagementSubnet（管理NIC有効時の追加案） | `10.100.4.0/26` |
| Hub / AzureBastionSubnet | `10.100.1.0/26` |
| Hub / GatewaySubnet | `10.100.2.0/26` |
| Hub / DNS Resolver inbound専用 | `10.100.3.0/28` |
| Hub / DNS Resolver outbound専用 | `10.100.3.16/28` |
| Corpの試験用Spoke | `10.101.0.0/16` |
| Corp / アプリ用 | `10.101.0.0/24` |
| Corp / Private Endpoint用 | `10.101.1.0/24` |
| Onlineの試験用Spoke | `10.102.0.0/16` |
| Online / アプリ用 | `10.102.0.0/24` |
| Online / Private Endpoint用 | `10.102.1.0/24` |
| オンプレ側のサンプル範囲 | `172.20.0.0/16` |

提案内のVNet間の非重複、サブネットの境界・包含・非重複を確認済み。実接続時には、接続先が広告する全範囲との重複を確認する。DNS Resolverの各サブネットは専用とし、必要な委任を設定する。Firewall管理NICをAVM既定どおり有効にする構成案では、上表の管理用サブネットとPublic IPを用意する。

参照：[専用サブネットの要件](https://learn.microsoft.com/en-us/azure/networking/design-guide/vnets-subnets)、[ExpressRoute GatewaySubnet](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways)

ExpressRoute回線の開通・廃止には事業者側の工程があるため、開通済み回線を外部fixtureとして扱う。試験で作成したGateway・Connection等と、保持する回線をstate・削除範囲で区別する。疎通試験では、回線の作成状態だけでなくprivate peering・BGP・経路・対向の実アプリ応答を確認する。

参照：[ExpressRouteの開通と状態](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-workflows)

### 11. 受け入れ条件とテスト一覧

期待値の正本は、合意した要件・構成契約・Policy一覧とする。Terraformの出力や実装コードをそのまま期待値へ転記しない。次のIDをケース設計の起点とし、具体的なPolicy IDや対象リソースを確定してケースを展開する。

| テストID | 対象 | 合格条件 |
| --- | --- | --- |
| DEP-01 | デプロイ結果 | Azure APIで取得した管理グループ階層、Policy、RBAC、ネットワーク、監視の実設定が構成契約と一致する |
| DEP-02 | 再適用 | 試験による変更を戻し、Policyの反映を確認した後、同じ入力の再planが差分ゼロ。再applyしてもリソースIDが維持される |
| POL-01 | 全Policyの構成 | 定義・initiative内の参照・パラメータ・scope・除外・enforcementが期待する一覧と一致する |
| POL-02 | 拒否系 | 許可ケースの成功と、違反ケースが期待するPolicyによって拒否されることを対で確認する |
| POL-03 | 変更系 | 有効なModify・Append等が、対象プロパティを期待値へ変更する |
| POL-04 | 自動配置系 | 有効なDeployIfNotExists等が、評価・修復後に期待するリソースと設定を配置する |
| IAM-01 | RBAC | 実際のprincipal・role・scopeを確認し、試験用IDで許可操作が成功、禁止操作が拒否される |
| NET-01 | Spoke間通信 | Firewall経由の明示許可フローは実アプリ応答が成功し、不許可フローは拒否される。経路と該当ログも確認する |
| NET-02 | ExpressRoute・DNS | 開通済み回線とGateway・Connectionを使い、オンプレ↔SpokeがFirewallを経由することと、DNS・試験先の実応答を確認する |
| CLN-01 | 後始末 | 成功・部分失敗の双方で試験作成物を削除し、Azure実体の残存を検出。固定の土台と回線は保持する |
| CI-01 | 対象限定 | 認証tenant、親管理グループ、subscription poolが信頼した試験設定と一致しない場合は実行しない |
| CI-02 | PRの品質ゲート | 対象PRの最新コミットで、単体・構成・実Azure E2E・後始末が成功した場合だけマージ可能にする |

#### Policyの必須試験範囲

- 全割当の構成整合確認は必須とする。
- 有効な拒否・変更・自動配置の効果（Deny、DenyAction、Modify、Append、DeployIfNotExists等）は実効性試験を必須とする。実際に採用する効果についてケースを作る。
- Audit・AuditIfNotExists等の検知系は構成整合確認を必須とし、実動作の検証は初期版の必須ゲートには含めない。
- 無効化・除外・非強制モードも構成契約と照合する。試験を通す目的でPolicyを無効化しない。
- Policy一覧にはassignment、definition／initiative reference、実効effect、対象scope、正常・違反fixture、期待結果、テストIDを対応づける。未対応の必須ケースをskipで完了扱いにしない。

#### 偽陽性を避ける条件

- Policyの拒否は`RequestDisallowedByPolicy`等の期待する応答とPolicy IDを確認する。`AuthorizationFailed`、入力不備、API障害を合格にしない。
- 許可・違反ケースは同じID・同じ操作を基本とし、検証対象の条件だけを変える。
- 非同期の評価・修復には期限を設ける。待機切れは失敗とし、skipや成功へ変換しない。
- 通信はTCP接続成功だけでなく、制御した宛先の実アプリ応答まで確認する。不許可通信も、宛先未起動など別の原因による失敗と区別する。
- 認証・接続・fixtureの不足は試験環境の不備として報告し、実装の振る舞いを確認したRedには数えない。

### 12. TDDとCIの設計方針

1. 受け入れ条件をテストケースへ展開し、人間が期待値と抜けを確認する。
2. 1件ずつテストを書き、意図した理由のRedを確認してから最小実装する。
3. Greenを保って整理し、必要な単体・構成テストを回す。
4. PRごとに専用テナントで実Azure E2Eを実行する。
5. 後始末と残存確認を行い、結果・Policy ID・実状態の証跡を記録する。
6. 最新コミットの必須テストと後始末が成功してからマージする。

共有subscription poolを使う実E2Eは、pool単位で排他する実装案とする。runごとのstate分離だけでは、サブスクリプション配置の競合を防げない。実行中の試験を新しいPRのために自動中断せず、待機runが取消・置換された場合も成功扱いにしない。

固定サブスクリプションは、今回作成した管理グループの削除前に、固定の待避先へ戻す手順を用意する。後始末はTerraform stateとAzure実体の両方を照合し、Policyが作成したリソースや試験用の操作対象も含めて残存確認する。後始末に失敗した場合はCIを失敗とし、stateと作成記録を保持して復旧できるようにする。

今回の更新は要件・テスト設計の記録。ALZ向けのA.1〜A.5の改稿、Terraform実装、実Azure試験は次の段階で行う。

### 13. 1回のE2E費用・時間の概算と上限案

調査日：2026-11-03（JST）。MicrosoftのAzure Retail Prices APIから取得したJPY参考価格を使う。EA契約単価・割引・税を反映した見積書ではない。

#### 計算の前提

- 東日本、単一Hub、試験用Spoke 2本。Firewall Standard、Bastion Standard（基本2インスタンス）、ErGw1AZ。
- DNS Private Resolverはinbound 1個、outbound 1個、ruleset 1個。
- 管理NIC有効のFirewall用Public IP 2個とBastion用1個を仮定。ER GatewayはHOBO方式を前提とし、独立した自作Public IPを重ねて計上しない。
- 疎通用にLinux B2sを2台仮定する。追加のPolicy試験リソースやprivate runnerは別途積み上げる。
- 有償DDoS Network Protection・IP Protectionはユーザー指定で無効にする。関連Policyが有償保護を再有効化しないよう、設定・パラメータと必要な例外を明示して構成テストで確認する。ExpressRouteのみを選択したため、VPN Gatewayを併設しない構成を見積り前提とする。AVMの有効フラグは明示する。
- 作成・試験・削除を含む6時間、各リソースを保持する保守的な単純計算。実際はリソースごとに作成・削除時刻と課金粒度が異なる。

#### 既知の基盤費用

| 項目 | 計算条件 | 1時間あたり | 6時間 |
| --- | --- | ---: | ---: |
| Firewall Standard | 1台 | 197.79円 | 1,186.73円 |
| Bastion Standard | 基本料金に2インスタンスを含む | 45.89円 | 275.32円 |
| ErGw1AZ | 1台 | 57.12円 | 342.73円 |
| DNS Resolver | inbound＋outbound＋ruleset、月額を730時間で按分 | 78.57円 | 471.44円 |
| Standard Public IP | 3個 | 2.37円 | 14.24円 |
| Linux B2s | 2台 | 17.22円 | 103.29円 |
| 合計 | ディスク・ログ・転送等を除く | 約399円 | 約2,394円 |

DNS endpointは各28,481.4円／月、rulesetは395.575円／月の参考価格から換算した。Bastionは基本料金を2倍にしない。

別途計上する費用：

- Log Analytics取り込み：有料単価で約528.49円／GB。無料枠を差し引かず、5GBなら約2,642円の例。
- Private DNS：ゾーン数・query数。ゾーンは日単位の按分なので、単純に6時間だけの時間割りにはしない。
- VMのディスク、Storage、peering／Firewall処理／ExpressRoute等の転送、必要なCI runner。
- DINE等の試験で作るVM・SQL・AKS等や、有効化するDefender等。Policy利用自体の料金と分ける。必要な種類・数・保持時間はPolicy別fixture一覧を作ってから算出する。
- 固定の試験回線・backend等の常設費用は、この1回あたりの増分とは別管理する。

有償DDoSを有効にした場合の参考差額は約638.02円／時、6時間で約3,828円。今回は含めない。公式FAQでは、月の一部だけ有効な場合は利用時間按分とされ、最低1か月料金として計算しない。

#### 初期の運用上限案

| 項目 | 提案 |
| --- | --- |
| 1PRの予算枠 | 10,000円／回（暫定案） |
| 作成 | 最大90分 |
| Policy・権限・通信・冪等性試験 | 最大120分 |
| 主処理の余裕 | 30分 |
| 主処理job | 上記合計で最大4時間 |
| 後始末job | 主処理と独立させ、失敗時も実行。初期枠2時間 |
| 全工程の運用目安 | 合計6時間（排他待ちを除く） |

これはMicrosoftの完了保証や、全Policy試験を実測した結果ではない。公式手順でGateway作成は約45分、Bastionは約10分、Firewallは10分以上の場合があることと、Policy反映・DINE遅延等を踏まえた初期の設計値である。

1万円は事前見積り・運用上の予算枠であり、請求額を強制停止する保証ではない。Azure Budget通知だけで費用は止まらず、削除失敗時は課金が継続し得る。主処理のtimeout後も後始末を実行し、後始末失敗時はCI失敗・pool隔離・記録を残して復旧する。

全動作系Policyのfixture費用と実行時間は未確定のため、現時点では総額・全件所要時間を保証できない。試験一覧の展開と初回pilotで、この枠を検証する。上限を超える見込みなら予算・fixtureの組み方を見直し、必須試験のskipで枠に合わせない。

#### 価格と時間の出典

- [Azure Retail Prices APIの説明](https://learn.microsoft.com/en-us/rest/api/cost-management/retail-prices/azure-retail-prices)
- [東日本FirewallのJPY公開単価](https://prices.azure.com/api/retail/prices?currencyCode=%27JPY%27&%24filter=serviceName%20eq%20%27Azure%20Firewall%27%20and%20armRegionName%20eq%20%27japaneast%27%20and%20priceType%20eq%20%27Consumption%27)
- [東日本BastionのJPY公開単価](https://prices.azure.com/api/retail/prices?currencyCode=%27JPY%27&%24filter=serviceName%20eq%20%27Azure%20Bastion%27%20and%20armRegionName%20eq%20%27japaneast%27%20and%20priceType%20eq%20%27Consumption%27)
- [東日本ExpressRouteのJPY公開単価](https://prices.azure.com/api/retail/prices?currencyCode=%27JPY%27&%24filter=serviceName%20eq%20%27ExpressRoute%27%20and%20armRegionName%20eq%20%27japaneast%27%20and%20priceType%20eq%20%27Consumption%27)
- [DNSのJPY公開単価](https://prices.azure.com/api/retail/prices?currencyCode=%27JPY%27&%24filter=serviceName%20eq%20%27Azure%20DNS%27%20and%20priceType%20eq%20%27Consumption%27)、[DNS課金単位](https://azure.microsoft.com/en-us/pricing/details/dns/)
- [Bastionの基本インスタンス数](https://azure.microsoft.com/en-us/pricing/details/azure-bastion/)
- [DDoSの課金FAQ](https://azure.microsoft.com/en-us/pricing/details/ddos-protection/)
- [Budgetの通知と予算管理](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
- [ExpressRoute Gateway作成](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-howto-add-gateway-portal-resource-manager)、[Firewall作成](https://learn.microsoft.com/en-us/azure/firewall/deploy-template)、[Bastion作成](https://learn.microsoft.com/en-us/azure/bastion/quickstart-host-portal)
- [Policy評価トリガー](https://learn.microsoft.com/en-us/azure/governance/policy/how-to/get-compliance-data)、[DINE評価遅延](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-deploy-if-not-exists)

