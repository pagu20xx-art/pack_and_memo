# パック＆メモ 開発スキル (Pack and Memo Dev)

## 概要 (Overview)

このスキルは、React Native (Expo) を使用したモバイルアプリケーション「パック＆メモ」の開発を可能にします。以下の作業を行う際にこのスキルを使用してください：
- 新しいUI画面（Home, Checklist, Memory）の実装（Expo Router を使用）。
- React Context API を用いたアプリの状態管理。
- `expo-sqlite` を用いたローカルデータベースのスキーマ設計および操作。
- `expo-image-picker` および `expo-file-system` を用いたネイティブ機能（カメラ、システムフォトピッカー、ファイル保存）の統合。
- Material 3 デザインパターンの適用。

## 開発ワークフロー (Development Workflow)

新しい機能を実装する際は、以下の手順に従ってください：

1. **仕様の分析 (Analyze Specification)**: `SPECIFICATION.md` を参照し、画面遷移、データ要件、および例外処理を理解します。
2. **モデルとスキーマの定義 (Define Models & Schemas)**: 機能にデータの永続化が必要な場合、`expo-sqlite` のテーブルスキーマを定義します（`SPECIFICATION.md` のデータモデル定義を参照）。
3. **状態管理の設計 (State Management)**: React Context API とカスタムフック（`useEvents` など）を用いて、SQLite と連携した状態管理ロジックを設計します。
4. **UIの実装 (UI Implementation)**: React Native のコンポーネント（または Expo Router の画面）を構築します。Android 15 で必須となる Edge-to-Edge（セーフエリア）の考慮や、Material 3 デザインシステムに準拠したUIを `StyleSheet` で実装します。
5. **テストと検証 (Test & Verify)**: 例外処理（ストレージ容量不足、画像選択のキャンセルなど）も含め、実機（Expo Go）等で実装を検証します。

## 主要技術リファレンス (Core Tech References)

- **状態管理と永続化 (State & Persistence)**: React Native + Expo SQLite + Context API の推奨アーキテクチャについては、`references/expo_context_sqlite_patterns.md` を参照してください。
- **Android/iOS対応ガイド (OS Compatibility Guide)**: セーフエリアのハンドリングや、カメラ・フォトピッカーの最適な呼び出しについては `references/pixel_10_pro_guide.md` を参照してください。
- **プロジェクト仕様 (Project Specifics)**: 画面遷移の定義やデータ構造の詳細については、常に `/home/akio_shino/projects/pack_and_memo/SPECIFICATION.md` を参照してください。

## リソース (Resources)

### scripts/
- 自動化、データ処理、またはボイラープレート生成用のユーティリティスクリプト。

### references/
- 詳細な技術ガイド、APIの使用パターン、およびアーキテクチャドキュメント。

### assets/
- ビジュアルアセット、UIコンポーネントのテンプレート、またはボイラープレートコード。
