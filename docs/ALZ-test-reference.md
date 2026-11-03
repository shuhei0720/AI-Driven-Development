# ALZテストの実装・設定例

[テストの読み方へ戻る](../AI-Driven-Development-CAF-ALZ.md#a2-テスト装置の実物一式)

## テスト対象のTerraform実装

`main.management.groups.tf` は管理グループを作る構成、`modules/settings/main.tf` は入力の検証とアドレス計算を行う実装である。

```hcl
# main.management.groups.tf — 公式AVMを、自分たちのルートから呼ぶ
module "settings" {
  source = "./modules/settings"
}
module "management_groups" {
  source             = "Azure/avm-ptn-alz/azurerm"
  version            = "0.21.0"
  architecture_name  = "alz"
  location           = module.settings.location
  parent_resource_id = var.parent_management_group_id
}
```

```hcl
# modules/settings/main.tf — Azureに接続しないアドレス計算
variable "hub_cidr" {
  type    = string
  default = "10.100.0.0/16"
  validation {
    condition     = can(cidrnetmask(var.hub_cidr)) && try(split("/", var.hub_cidr)[1] == "16", false)
    error_message = "HubにはIPv4の/16を指定する。"
  }
}
output "location" { value = "japaneast" }
output "subnets" {
  value = try({
    firewall   = cidrsubnet(var.hub_cidr, 10, 0)
    gateway    = cidrsubnet(var.hub_cidr, 10, 8)
    bastion    = cidrsubnet(var.hub_cidr, 10, 4)
    dns_in     = cidrsubnet(var.hub_cidr, 12, 48)
    dns_out    = cidrsubnet(var.hub_cidr, 12, 49)
    management = cidrsubnet(var.hub_cidr, 10, 16)
  }, {})
}
```

## 単体テスト

作成先：`tests/unit/settings.tftest.hcl`。

```hcl
run "bastion_subnet" {
  command = plan
  module { source = "./modules/settings" }
  assert {
    condition     = output.subnets.bastion == "10.100.1.0/26"
    error_message = "Bastionのサブネットは10.100.1.0/26であること。"
  }
}

# tests/unit/settings.tftest.hcl — native Terraform Test
run "sample_subnet_contract" {
  command = plan
  module { source = "./modules/settings" }
  assert {
    condition = output.location == "japaneast" && output.subnets == {
      firewall = "10.100.0.0/26", gateway = "10.100.2.0/26"
      bastion  = "10.100.1.0/26", management = "10.100.4.0/26"
      dns_in   = "10.100.3.0/28", dns_out = "10.100.3.16/28"
    }
    error_message = "合意したリージョン・サブネット配置と一致すること。"
  }
}
run "prefix_15_is_rejected" {
  command = plan
  module { source = "./modules/settings" }
  variables { hub_cidr = "10.100.0.0/15" }
  expect_failures = [var.hub_cidr]
}
run "prefix_17_is_rejected" {
  command = plan
  module { source = "./modules/settings" }
  variables { hub_cidr = "10.100.0.0/17" }
  expect_failures = [var.hub_cidr]
}
run "invalid_cidr_is_rejected" {
  command = plan
  module { source = "./modules/settings" }
  variables { hub_cidr = "10.100.0.0/33" }
  expect_failures = [var.hub_cidr]
}
```

## E2Eテスト

作成先：`tests/e2e/platform_test.go`。共通基盤を配置して実値を確認する。

```go
// tests/e2e/platform_test.go
package e2e

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/require"
)

func TestPlatformE2E(t *testing.T) {
	assertTestTarget(t)
	opts := terraformOptions(t, "../..", "platform", "TEST_TFVARS")
	terraform.InitAndApply(t, opts)
	// ALZ全体の削除は独立cleanup jobが担当する（A.2.4）。

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

作成先：`tests/e2e/policy_test.go`。Denyの許可・拒否と、DINEの自動配置を実Azureで確認する。

```go
// tests/e2e/policy_test.go
package e2e

import (
	"encoding/json"
	"slices"
	"strings"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/require"
)

func testDenyPublicStorage(t *testing.T) {
	opts := terraformOptions(t, "../fixtures/storage-policy", "deny", "TEST_PROBE_TFVARS")
	opts.Vars = map[string]interface{}{"public_network_access_enabled": false}
	defer terraform.Destroy(t, opts) // apply前に登録。途中失敗でも後始末する
	terraform.InitAndApply(t, opts)  // 許可側が成功することを先に確認

	assignmentID := requiredEnv(t, "PUBLIC_STORAGE_DENY_ASSIGNMENT_ID")
	definitionID := requiredEnv(t, "PUBLIC_STORAGE_DENY_DEFINITION_ID")
	require.Eventually(t, func() bool {
		// Policy反映前に成功しても、次回は許可状態に戻してから再試行する。
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
	opts := terraformOptions(t, "../fixtures/storage-policy", "dine", "TEST_PROBE_TFVARS")
	opts.Vars = map[string]interface{}{"public_network_access_enabled": false}
	defer terraform.Destroy(t, opts)
	terraform.InitAndApply(t, opts) // fixtureは診断設定を自分で作らない

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

## 共通処理

作成先：`tests/e2e/support_test.go`。認証先・state・入力の指定と、Azureからの実値取得をまとめる。

```go
// tests/e2e/support_test.go
package e2e

import (
	"encoding/json"
	"os"
	"slices"
	"strings"
	"testing"

	"github.com/gruntwork-io/terratest/modules/logger"
	"github.com/gruntwork-io/terratest/modules/shell"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/require"
)

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
		Reconfigure: true, NoColor: true, SetVarsAfterVarFiles: true,
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

func diagnosticWorkspaceIDs(t *testing.T, targetID string) []string {
	t.Helper()
	out, err := azE(t, "monitor", "diagnostic-settings", "list", "--resource", targetID,
		"--query", "value[].workspaceId", "-o", "json")
	if err != nil {
		return nil
	}
	var ids []string
	if json.Unmarshal([]byte(out), &ids) != nil {
		return nil
	}
	return ids
}
```

## 実行設定

作成先：リポジトリ直下の`go.mod`・`go.sum`。Go 1.23、Terratest `v0.48.2`、testify `v1.10.0` を基準にする。
初回は `go mod init example.com/alz-tests`、続けて `go get github.com/gruntwork-io/terratest@v0.48.2 github.com/stretchr/testify@v1.10.0` を実行する。

作成先：`scripts/test.sh`。

```bash
# scripts/test.sh — 通常実行はAzureへの作成・課金を伴わない
set -euo pipefail
case "${1:-unit}" in
  unit)
    terraform init -backend=false -test-directory=tests/unit
    terraform test -test-directory=tests/unit ;;
  integration)
    terraform init -backend=false -test-directory=tests/integration
    terraform test -test-directory=tests/integration ;;
  e2e)
    go test -v -count=1 -parallel=1 -timeout=4h ./tests/e2e ;;
  *) printf 'usage: %s unit|integration|e2e\n' "$0" >&2; exit 2 ;;
esac

```

作成先：`.github/workflows/test.yml`。

```yaml
# .github/workflows/test.yml — 抜粋。全PRで実行し、最後のcheckを必須にする
on: [pull_request]
jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 240
    permissions: { contents: read, id-token: write }
    steps:
      # checkout、Go/Terraform導入、go mod download、UAMIへのOIDCログイン
      # 固定fixture照合・pool lease取得・所有manifest保存・環境変数設定を先に行う
      - run: bash scripts/test.sh unit
      - run: bash scripts/test.sh integration
      - run: bash scripts/test.sh e2e
  cleanup:
    needs: e2e
    if: always()
    runs-on: ubuntu-latest
    timeout-minutes: 120
    permissions: { contents: read, id-token: write }
    steps:
      # 同じcommitをcheckoutし、cleanup用UAMIへOIDCログイン
      # runner外のmanifestと同じrunの全stateから回収する
      - run: bash scripts/cleanup.sh "${GITHUB_RUN_ID}-${GITHUB_RUN_ATTEMPT}"
  alz-required:
    needs: [e2e, cleanup]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - env:
          E2E: ${{ needs.e2e.result }}
          CLEANUP: ${{ needs.cleanup.result }}
        run: test "$E2E" = success && test "$CLEANUP" = success

```

## 試験環境と合格条件

- 認証はUAMIへのOIDCフェデレーション。別の試験テナントと固定のsubscription poolだけを操作する。tfvarsも固定fixtureと照合する。
- 試験用Terraformは`tests/fixtures/storage-policy/`に置き、`public_network_access_enabled`を入力、`diagnostic_target_id`を出力とする。診断設定をfixture自身で作らず、DINEに作らせる。
- `TEST_TFVARS`・`TEST_PROBE_TFVARS`は絶対パス、`TEST_RUN_KEY`はrun ID＋attempt。probe名はrun固有とし、state keyと所有manifestはrunner外へ先に保存する。
- 共有poolは直列化する。Denyは許可側の成功、拒否code、割当・定義IDを照合する。反映前に禁止更新が成功した場合も、毎回許可状態に戻してから再試行する。
- DINEの新規配置と既存不適合への修復、全動作系Policy、配置・RBAC・通信もプランのケース台帳に対応づける。必須ケースの不足はCI失敗とする。
- cleanupはprobe削除→association解除を構成変更とapplyで反映→固定subscription退避→root destroy→残存確認の順。固定のID・backend・回線は保持する。
- 取消・runner障害も独立reconcilerで回収する。削除失敗時はpoolを隔離し、state・manifestを保存する。
- 最新commitのE2Eとcleanupが成功してからマージする。1回1万円・主処理4時間＋cleanup2時間は暫定案で、必須試験をskipして枠に合わせない。

これは構成・テストの例であり、Terraformプロジェクト全体ではない。Goコードのコンパイルと実Azureでの合格は分けて扱う。実AzureのE2Eは未実行。
