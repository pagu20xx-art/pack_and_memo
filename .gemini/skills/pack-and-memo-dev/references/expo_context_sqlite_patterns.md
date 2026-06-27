# Expo SQLite ＆ React Context 状態管理パターン

このガイドは、React Native (Expo) 環境における、`expo-sqlite` を用いたデータの永続化と、React Context API を用いたクリーンな状態管理パターンの標準的な実装方法を定義します。

---

## 1. React Context による状態管理

React Native では、軽量で直観的な状態管理として React Context API を採用します。

### ガイドライン
*   **UIのクリーンさの維持:** 画面を表すコンポーネント（Screen）や子コンポーネント（Widget）内には直接データベースのクエリを記述せず、ビジネスロジックはカスタムフック（例: `useEvents`）に集約します。
*   **非同期処理の安全なハンドリング:** SQLite へのアクセスは常に非同期（Promise）です。状態がロード中（`loading`）、エラー発生（`error`）、完了（`data`）のいずれであるかを明確に追跡できるように設計します。
*   **イミュータブルな状態更新:** 状態を変更する（アイテムをトグルする等）際は、直接Stateを破壊せず、常にコピーを作成して更新します。

### 実装パターン: EventsContext
```typescript
import React, { createContext, useContext, useState, useEffect } from 'react';
import { TripEvent } from '../models/types';
import * as db from '../db/database';

interface EventsContextType {
  events: TripEvent[];
  isLoading: boolean;
  error: Error | null;
  loadEvents: () => Promise<void>;
  addEvent: (title: string, date: string) => Promise<void>;
  deleteEvent: (id: string) => Promise<void>;
  toggleItem: (eventId: string, itemId: string) => Promise<void>;
  addPackingItem: (eventId: string, name: string) => Promise<void>;
}

const EventsContext = createContext<EventsContextType | undefined>(undefined);

export const EventsProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [events, setEvents] = useState<TripEvent[]>([]);
  const [isLoading, setIsLoading] = useState<boolean>(true);
  const [error, setError] = useState<Error | null>(null);

  const loadEvents = async () => {
    setIsLoading(true);
    try {
      const data = await db.getAllEvents();
      setEvents(data);
      setError(null);
    } catch (e) {
      setError(e as Error);
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    loadEvents();
  }, []);

  const addEvent = async (title: string, date: string) => {
    try {
      await db.createEvent(title, date);
      await loadEvents(); // SQLiteから再読み込みして最新の状態にする
    } catch (e) {
      setError(e as Error);
    }
  };

  const deleteEvent = async (id: string) => {
    try {
      await db.deleteEvent(id);
      await loadEvents();
    } catch (e) {
      setError(e as Error);
    }
  };

  const toggleItem = async (eventId: string, itemId: string) => {
    try {
      await db.togglePackingItem(itemId);
      await loadEvents();
    } catch (e) {
      setError(e as Error);
    }
  };

  const addPackingItem = async (eventId: string, name: string) => {
    try {
      await db.addPackingItem(eventId, name);
      await loadEvents();
    } catch (e) {
      setError(e as Error);
    }
  };

  return (
    <EventsContext.Provider value={{
      events,
      isLoading,
      error,
      loadEvents,
      addEvent,
      deleteEvent,
      toggleItem,
      addPackingItem
    }}>
      {children}
    </EventsContext.Provider>
  );
};

export const useEvents = () => {
  const context = useContext(EventsContext);
  if (!context) {
    throw new Error('useEvents must be used within an EventsProvider');
  }
  return context;
};
```

---

## 2. ローカルデータベース ＆ 永続化 (expo-sqlite)

`expo-sqlite` は、アプリローカルに SQLite データベースを構築するための非常に軽量で堅牢なライブラリです。

### ガイドライン
*   **データベース初期化:** アプリケーション起動時に、必要なテーブル（`trips`, `items`, `memories`）を作成します。
*   **トランザクションの保護:** 複数の関連するテーブルへ書き込む際（例: イベント削除時に持ち物リストも削除するなど）、トランザクションを利用して不整合を防止します。
*   **参照整合性の管理:** イベントが削除されたら、それに紐づく `PackingItem` や `EventMemory`、およびローカルに保存されている写真データ（`expo-file-system` を使用）を適切に削除して、ストレージリークを防ぎます。

### データベースヘルパー実装パターン
```typescript
import * as SQLite from 'expo-sqlite';
import { TripEvent, PackingItem } from '../models/types';

let _db: SQLite.SQLiteDatabase | null = null;

export async function getDatabase(): Promise<SQLite.SQLiteDatabase> {
  if (!_db) {
    _db = await SQLite.openDatabaseAsync('pack_and_memo.db');
    await initializeDatabase(_db);
  }
  return _db;
}

async function initializeDatabase(db: SQLite.SQLiteDatabase) {
  await db.execAsync('PRAGMA foreign_keys = ON;');
  
  await db.execAsync(`
    CREATE TABLE IF NOT EXISTS trips (
      id TEXT PRIMARY KEY,
      title TEXT NOT NULL,
      date TEXT NOT NULL,
      photo_uri TEXT,
      note TEXT
    );
  `);

  await db.execAsync(`
    CREATE TABLE IF NOT EXISTS items (
      id TEXT PRIMARY KEY,
      trip_id TEXT NOT NULL,
      name TEXT NOT NULL,
      is_checked INTEGER DEFAULT 0,
      FOREIGN KEY (trip_id) REFERENCES trips (id) ON DELETE CASCADE
    );
  `);
}

export async function getAllEvents(): Promise<TripEvent[]> {
  const db = await getDatabase();
  
  const trips: any[] = await db.getAllAsync('SELECT * FROM trips ORDER BY date DESC;');
  const events: TripEvent[] = [];

  for (const trip of trips) {
    const items: any[] = await db.getAllAsync(
      'SELECT * FROM items WHERE trip_id = ?;',
      [trip.id]
    );

    events.push({
      id: trip.id,
      title: trip.title,
      date: trip.date,
      items: items.map(item => ({
        id: item.id,
        name: item.name,
        isChecked: item.is_checked === 1
      })),
      memory: {
        photoUri: trip.photo_uri || null,
        note: trip.note || ''
      }
    });
  }

  return events;
}

export async function createEvent(title: string, date: string): Promise<void> {
  const db = await getDatabase();
  const id = Date.now().toString();
  await db.runAsync(
    'INSERT INTO trips (id, title, date) VALUES (?, ?, ?);',
    [id, title, date]
  );
}

export async function deleteEvent(id: string): Promise<void> {
  const db = await getDatabase();
  await db.runAsync('DELETE FROM trips WHERE id = ?;', [id]);
}

export async function togglePackingItem(itemId: string): Promise<void> {
  const db = await getDatabase();
  await db.runAsync(
    'UPDATE items SET is_checked = 1 - is_checked WHERE id = ?;',
    [itemId]
  );
}

export async function addPackingItem(tripId: string, name: string): Promise<void> {
  const db = await getDatabase();
  const id = Date.now().toString();
  await db.runAsync(
    'INSERT INTO items (id, trip_id, name) VALUES (?, ?, ?);',
    [id, tripId, name]
  );
}
```

---

## 3. 継承よりも合成（Composition Over Inheritance）の徹底

`GEMINI.md` のコア原則に基づき、フックやコンポーネントの設計でも**合成（Composition）**を徹底します。

### ガイドライン
*   **カスタムフックの組み合わせ:** 共通の振る舞い（例: ログ記録やエラー管理）が必要な場合は、巨大なベースContextを作るのではなく、単一責任の機能（例: `useLogger`, `useErrorHandler`）に分け、それらをコンポーネントや他のフックで合成して利用します。
*   **UIコンポーネントの委譲:** ダイアログやボトムシートは特定の親コンポーネントに結合せず、子コンポーネントを `children` プロパティ経由で受け取る構成（合成）にすることで、再利用性と可読性を最大化します。
