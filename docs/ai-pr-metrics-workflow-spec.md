# AI PR メトリクス ワークフロー仕様書

## 概要

このワークフローは、GitHub組織内のリポジトリにおけるAI関連Pull Request（PR）の比率を測定・分析するためのGitHub Actionsワークフローです。

## ファイル

- **ワークフローファイル**: `.github/workflows/ai-pr-org-metrics.yml`
- **トリガー**: 手動実行（`workflow_dispatch`）
- **実行環境**: `ubuntu-latest`

## 機能

### 主要機能
1. **リポジトリスキャン**: 指定したユーザー/組織の全リポジトリを取得
2. **期間指定分析**: 指定した日付範囲でのマージ済みPRを分析
3. **AIラベル検出**: 複数のAI関連ラベルでPRを分類
4. **重複除去**: 複数のAIラベルを持つPRの重複カウントを防止
5. **統計計算**: AI PR比率の算出とリポジトリ別レポート生成

### 対象AIラベル（デフォルト）
- `AI`
- `ai`
- `ai-generated`
- `ai-assist`

## パラメータ

| パラメータ | 説明 | デフォルト値 | 必須 |
|-----------|-----|-------------|------|
| `start_date` | 分析開始日（YYYY-MM-DD） | 前週月曜日 | No |
| `end_date` | 分析終了日（YYYY-MM-DD） | 前週日曜日 | No |
| `ai_labels` | AIラベルリスト（カンマ区切り） | `AI,ai,ai-generated,ai-assist` | No |

## 実行方法

### GitHub UI経由
1. GitHubリポジトリの「Actions」タブに移動
2. 「AI PR Organization Metrics」ワークフローを選択
3. 「Run workflow」をクリック
4. パラメータを設定（オプション）
5. 「Run workflow」で実行

### GitHub CLI経由
```bash
# デフォルト設定で実行
gh workflow run ai-pr-org-metrics.yml

# パラメータ指定で実行
gh workflow run ai-pr-org-metrics.yml \
  -f start_date=2025-06-01 \
  -f end_date=2025-06-15 \
  -f ai_labels="AI,ai,ai-generated,ai-assist"
```

## 出力

### GitHub Actions Summary
ワークフロー実行後、以下の情報がSummaryに表示されます：

```
### AI PR 比率 (YYYY-MM-DD–YYYY-MM-DD)

#### 全体
- AI PR: XX
- 全 PR: XX
- 比 率: XX.X%
- 対象ラベル: AI,ai,ai-generated,ai-assist

#### リポジトリ別
- **repo1**: AI PR: X, 全 PR: X, 比率: X.X%
- **repo2**: AI PR: X, 全 PR: X, 比率: X.X%
```

### ログ出力
実行ログには以下の詳細情報が出力されます：
- リポジトリ発見プロセス
- 各リポジトリのPR検索クエリとレスポンス
- 重複除去プロセス
- API呼び出し状況

## 技術的仕様

### 依存関係
- `jq`: JSON処理
- `bc`: 算術計算
- `curl`: GitHub API呼び出し
- `sort`, `uniq`: 重複除去

### GitHub API使用
- **Search API**: PRの検索とカウント
- **Repos API**: リポジトリ一覧取得
- **認証**: `METRICS_TOKEN` シークレット必須

### 重複除去アルゴリズム
```bash
# 各ラベルでPR番号を収集
ai_pr_numbers="$ai_pr_numbers $pr_numbers"

# sort -u で重複除去してカウント
ai_count=$(echo "$ai_pr_numbers" | tr ' ' '\n' | grep -v '^$' | sort -u | wc -l)
```

## エラー対応記録と注意点

### 🚨 重要な注意点

#### 1. YAML構文エラー（文字列結合）
**問題**: 複数行文字列結合でYAML構文エラーが発生
```yaml
# ❌ 間違った書き方
ai_labels="label:" + $(echo "$AI_LABELS" | tr ',' '\n' | paste -sd '|' -) + ""

# ✅ 正しい書き方  
ai_labels="label:$(echo "$AI_LABELS" | tr ',' '\n' | paste -sd '|' -)"
```

#### 2. GitHub Search API 日付フォーマット
**問題**: 複雑な日付フォーマット（タイムゾーン付き）でAPI検索が失敗
```bash
# ❌ 動作しない形式
merged:2025-06-01T00:00:00+09:00..2025-06-15T23:59:59+09:00

# ✅ 動作する形式
merged:2025-06-01..2025-06-15
```

#### 3. 大文字小文字の区別
**問題**: ラベル検索で大文字小文字が厳密に区別される
- `label:ai` と `label:AI` は別々に検索が必要
- 両方のパターンを明示的に指定する

#### 4. 複雑なORクエリの問題
**問題**: `paste` コマンドでORが"O"に変換される
```bash
# ❌ pasteコマンドで破損
query="label:AI|ai|ai-generated"  # "O"に変換される

# ✅ 明示的なOR指定
query="label:AI OR label:ai OR label:ai-generated OR label:ai-assist"
```

#### 5. 重複カウント問題
**問題**: 同じPRに複数のAIラベルが付いている場合の重複カウント
- **症状**: 実際より高い比率（100%など）が算出される
- **原因**: ラベル別検索結果の単純加算
- **解決**: PR番号の収集と重複除去

### APIレート制限対策
```bash
# レート制限チェック（実装推奨）
if echo "$response" | grep -q "rate limit"; then
  echo "::warning::API rate limit reached"
  sleep 60
fi
```

### エラーハンドリングパターン
```bash
# API レスポンスの妥当性チェック
count=$(echo "$response" | jq -r '.total_count // 0' 2>/dev/null || echo "0")

# 算術演算のエラーハンドリング
pct=$(echo "scale=1; $ai_count*100/$all_count" | bc 2>/dev/null || echo "0")
```

## トラブルシューティング

### よくある問題

1. **0件の結果が返される**
   - 日付範囲の確認（未来の日付でないか）
   - ラベルの存在確認
   - リポジトリのアクセス権限確認

2. **API認証エラー**
   - `METRICS_TOKEN` シークレットの設定確認
   - トークンの権限確認（repo scope必要）

3. **JSON解析エラー**
   - APIレスポンスの確認
   - jqコマンドの構文確認

4. **異常に高い比率**
   - 重複カウントのチェック
   - ラベル設定の確認

## 今後の改善提案

1. **並列処理**: リポジトリごとの処理を並列化
2. **キャッシュ機能**: API結果のキャッシュ
3. **詳細レポート**: PR詳細情報の出力
4. **アラート機能**: 異常値検出時の通知
5. **履歴追跡**: 過去データとの比較

## バージョン履歴

- **v1.0**: 初期リリース
- **v1.1**: YAML構文エラー修正
- **v1.2**: 日付フォーマット対応
- **v1.3**: 大文字小文字対応
- **v1.4**: ORクエリ修正
- **v1.5**: 重複除去機能追加（現在版）

---

**最終更新**: 2025-06-15  
**作成者**: AI Assistant  
**レビュー**: 必要に応じて定期的に更新 