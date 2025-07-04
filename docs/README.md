# AI PR メトリクス ワークフロー ドキュメント

このフォルダには、AI PR メトリクス ワークフローに関する包括的なドキュメントが含まれています。

## 📋 ドキュメント一覧

### 1. [仕様書](./ai-pr-metrics-workflow-spec.md)
- **概要**: ワークフローの基本機能と仕様
- **対象**: 開発者、利用者、システム管理者
- **内容**: 
  - 機能説明
  - パラメータ仕様
  - 実行方法
  - 技術的詳細

### 2. [トラブルシューティングガイド](./troubleshooting-guide.md)
- **概要**: 実際に発生したエラーと対応方法
- **対象**: 開発者、システム管理者
- **内容**:
  - 緊急時対応チェックリスト
  - 実際のエラー事例と解決策
  - 診断コマンド集
  - 予防策

## 🎯 使用目的別ガイド

### 📊 利用者（メトリクス分析）
1. **仕様書** → パラメータ設定と実行方法を確認
2. **トラブルシューティング** → 結果が期待通りでない場合の対処法

### 🔧 開発者（機能開発・保守）
1. **仕様書** → 技術的仕様とAPIの詳細
2. **トラブルシューティング** → 過去の問題事例と解決パターン

### 🚨 システム管理者（運用・障害対応）
1. **トラブルシューティング** → 緊急時対応チェックリスト
2. **仕様書** → システム依存関係とセキュリティ設定

## 📈 ワークフローの重要ポイント

### ✅ 成功のポイント
- **正確な日付指定**: YYYY-MM-DD形式を使用
- **適切なトークン権限**: `repo` スコープ必須
- **ラベル設定**: 大文字小文字を区別して設定

### ⚠️ 注意が必要な点
- **重複カウント**: 複数AIラベルを持つPRの処理
- **APIレート制限**: 大量リポジトリでの実行時
- **YAML構文**: 複雑な文字列結合の回避

## 🔄 更新履歴

### 2025-06-15
- 初版ドキュメント作成
- 実際のエラー事例を基にした内容整備
- 重複カウント問題の解決記録

## 🤝 貢献

### ドキュメント改善
- 新しいエラー事例の追加
- 解決策の改善提案
- 使用例の追加

### 連絡先
- **Issues**: GitHub Issues で問題報告
- **Discussions**: 機能要望や質問
- **PR**: ドキュメント改善の提案

---

**注意**: このドキュメントは実際の運用経験に基づいて作成されています。新しい問題や改善点が発見された場合は、随時更新してください。 