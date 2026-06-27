# Issue 2: [Phase 2] Database (SQLite) & 状態管理 (React Context) の構築

## 概要
`expo-sqlite` を用いたローカルデータベースのテーブル定義と初期化、およびデータをアプリケーション全体でシームレスに操作するための React Context (useEvents フック) を実装します。

## タスクリスト
- [ ] `db/database.ts` にて `expo-sqlite` の接続およびテーブルの初期化ロジックを実装
  - `trips` テーブル (イベントデータ、思い出のメタデータ)
  - `items` テーブル (持ち物チェックリスト、`trip_id`による外部キー連携)
- [ ] データベースへの CRUD 抽象化関数の作成
  - `getAllEvents()`: イベント・持ち物データを結合して取得
  - `createEvent(title, date)`: 新規お出かけ予定の作成
  - `deleteEvent(id)`: イベントの削除 (Cascade による関連アイテムの自動削除)
  - `togglePackingItem(id)`: 持ち物チェック状態の反転
  - `addPackingItem(tripId, name)`: 新規持ち物の追加
- [ ] アプリの状態を保持・操作する `EventsContext` ＆ `EventsProvider` の実装 (`context/EventsContext.tsx`)
- [ ] UIから状態へ安全にアクセスできるカスタムフック `useEvents` の実装

## 検証項目
- [ ] アプリ起動時にデータベースが初期化され、エラーなく動作すること
- [ ] ダミーのイベントを追加・トグル・削除し、SQLite内のデータが意図通り更新されること
- [ ] アプリを再起動（リロード）してもデータが保持されていること
