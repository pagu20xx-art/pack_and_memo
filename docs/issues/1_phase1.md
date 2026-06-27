# Issue 1: [Phase 1] Expoプロジェクトの初期化と基本環境構築

## 概要
「パック＆メモ」のReact Native (Expo) 開発を開始するために、Expo Router ベースの TypeScript プロジェクトを新規生成し、必要な依存ライブラリを導入します。

## タスクリスト
- [ ] `npx create-expo-app@latest` を用いて、TypeScript / Expo Router 構成でプロジェクトを初期化
- [ ] 必要な基本パッケージのインストール
  - `expo-sqlite` (データベース用)
  - `expo-image-picker` (カメラ・フォトピッカー用)
  - `expo-file-system` (写真などのファイル管理用)
  - `react-native-safe-area-context` (Android 15のEdge-to-Edge対応用)
- [ ] プロジェクトの不要なデフォルトファイル（テンプレート画面等）のクリーンアップ
- [ ] 開発用基本ディレクトリ構造の整備
  - `components/` (共通UIパーツ)
  - `context/` (状態管理用Context)
  - `db/` (データベースヘルパー)
  - `app/` (Expo Router 画面遷移用)

## 検証項目
- [ ] `npm run android` などの開発サーバーが正常に立ち上がること
- [ ] 依存関係エラーや型エラーが発生せずにビルド（または起動）できること
