# AI PR メトリクス ワークフロー トラブルシューティングガイド

## 🚨 緊急時対応チェックリスト

### 1. ワークフロー実行失敗時
- [ ] `METRICS_TOKEN` シークレットの設定確認
- [ ] 日付パラメータの妥当性確認
- [ ] YAMLファイルの構文チェック
- [ ] ログの詳細確認

### 2. 異常な結果が出力された時
- [ ] 重複カウントの確認
- [ ] ラベル設定の確認
- [ ] 日付範囲の確認
- [ ] 手動での検証実行

## 実際に発生したエラー事例

### ❌ エラー事例 1: YAML構文エラー
```
Error: Invalid workflow file
.github/workflows/ai-pr-org-metrics.yml (Line: 154, Col: 20): 
A mapping was not expected
```

**原因**: 複数行での文字列結合構文エラー
```yaml
# 問題のコード
ai_labels="label:" + 
  $(echo "$AI_LABELS" | tr ',' '\n' | paste -sd '|' -) + 
  ""
```

**解決策**: 単一行での文字列結合
```yaml
# 修正後のコード
ai_labels="label:$(echo "$AI_LABELS" | tr ',' '\n' | paste -sd '|' -)"
```

### ❌ エラー事例 2: 検索結果が0件
```
Total AI PRs: 0
Total PRs: 0
Percentage: 0%
```

**原因**: GitHub Search API の日付フォーマット問題
```bash
# 動作しない形式
merged:2025-06-01T00:00:00+09:00..2025-06-15T23:59:59+09:00
```

**解決策**: シンプルな日付フォーマット使用
```bash
# 動作する形式
merged:2025-06-01..2025-06-15
```

### ❌ エラー事例 3: ORクエリが機能しない
```
# pasteコマンドで"OR"が"O"に変換される現象
AI PR query: label:AI O label:ai O label:ai-generated
```

**原因**: `paste` コマンドでの区切り文字変換
```bash
# 問題のコード
query="label:$(echo "$AI_LABELS" | tr ',' '\n' | paste -sd '|' -)"
```

**解決策**: 明示的なOR指定
```bash
# 修正後のコード
query="label:AI OR label:ai OR label:ai-generated OR label:ai-assist"
```

### ❌ エラー事例 4: 重複カウント問題
```
Results for AI: 10 PRs
Results for ai: 10 PRs
Total AI PRs: 20  # 実際は10であるべき
Percentage: 100%  # 実際は50%であるべき
```

**原因**: 同じPRに複数のAIラベルが付いている場合の重複カウント
```bash
# 問題のコード
ai_count=$((ai_count + label_count))
```

**解決策**: PR番号による重複除去
```bash
# 修正後のコード
ai_pr_numbers="$ai_pr_numbers $pr_numbers"
ai_count=$(echo "$ai_pr_numbers" | tr ' ' '\n' | grep -v '^$' | sort -u | wc -l)
```

## 診断コマンド集

### GitHub API 直接テスト
```bash
# 認証テスト
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/user

# リポジトリ一覧取得テスト
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/users/USERNAME/repos"

# PR検索テスト
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/issues?q=repo:USER/REPO+is:pr+is:merged+merged:2025-06-01..2025-06-15"

# AIラベル検索テスト
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/issues?q=repo:USER/REPO+is:pr+is:merged+label:AI+merged:2025-06-01..2025-06-15"
```

### ローカル検証スクリプト
```bash
#!/bin/bash
# ai-pr-test.sh - ローカル検証用スクリプト

set -e

OWNER="kh55"
REPO="ai_report"
START_DATE="2025-06-01"
END_DATE="2025-06-15"
TOKEN="your_token_here"

echo "=== AI PR 検証スクリプト ==="
echo "対象: $OWNER/$REPO"
echo "期間: $START_DATE to $END_DATE"
echo ""

# 全マージ済みPR
echo "1. 全マージ済みPR数:"
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/search/issues?q=repo:$OWNER/$REPO+is:pr+is:merged+merged:$START_DATE..$END_DATE" | \
  jq -r '.total_count'

# AIラベル別検索
for label in "AI" "ai" "ai-generated" "ai-assist"; do
  echo "2. ラベル '$label' のPR数:"
  count=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "https://api.github.com/search/issues?q=repo:$OWNER/$REPO+is:pr+is:merged+label:$label+merged:$START_DATE..$END_DATE" | \
    jq -r '.total_count')
  echo "   $count PRs"
done

# 重複チェック
echo "3. 重複チェック:"
ai_prs=""
for label in "AI" "ai" "ai-generated" "ai-assist"; do
  response=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "https://api.github.com/search/issues?q=repo:$OWNER/$REPO+is:pr+is:merged+label:$label+merged:$START_DATE..$END_DATE")
  pr_numbers=$(echo "$response" | jq -r '.items[]?.number // empty' | tr '\n' ' ')
  ai_prs="$ai_prs $pr_numbers"
done

unique_count=$(echo "$ai_prs" | tr ' ' '\n' | grep -v '^$' | sort -u | wc -l)
echo "   ユニークなAI PR数: $unique_count"
```

## 予防策

### 1. 継続的監視
```yaml
# .github/workflows/ai-pr-metrics-test.yml
name: AI PR Metrics Test
on:
  push:
    paths:
      - '.github/workflows/ai-pr-org-metrics.yml'
  pull_request:
    paths:
      - '.github/workflows/ai-pr-org-metrics.yml'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Validate YAML
        run: |
          sudo apt-get install -y yamllint
          yamllint .github/workflows/ai-pr-org-metrics.yml
```

### 2. 設定値の妥当性チェック
```bash
# 日付形式チェック
if ! date -d "$start_date" +%Y-%m-%d > /dev/null 2>&1; then
  echo "::error::Invalid start_date format: $start_date"
  exit 1
fi

# 未来日付チェック
if [[ "$start_date" > "$(date +%Y-%m-%d)" ]]; then
  echo "::warning::Start date is in the future: $start_date"
fi
```

### 3. APIレスポンス妥当性チェック
```bash
# エラーレスポンスチェック
if echo "$response" | jq -e '.message' > /dev/null 2>&1; then
  error_msg=$(echo "$response" | jq -r '.message')
  echo "::error::GitHub API Error: $error_msg"
  exit 1
fi

# レート制限チェック
if echo "$response" | grep -q "rate limit exceeded"; then
  echo "::warning::API rate limit exceeded, waiting..."
  sleep 60
fi
```

## 連絡先・エスカレーション

### 開発者向け
- **GitHub Issues**: プロジェクトリポジトリのIssuesで報告
- **緊急対応**: ワークフローを無効化して手動分析に切り替え

### 利用者向け
1. **軽微な問題**: READMEのFAQを確認
2. **機能要望**: GitHub Discussionsで議論
3. **バグ報告**: GitHub Issuesで詳細報告

## 更新履歴

- **2025-06-15**: 初版作成、重複カウント問題の対応記録
- **2025-06-15**: YAML構文エラー、API日付フォーマット問題の対応記録
- **2025-06-15**: ORクエリ問題の対応記録

---

**注意**: このドキュメントは実際に発生した問題を基に作成されています。新しい問題が発生した場合は、このドキュメントを更新してください。 