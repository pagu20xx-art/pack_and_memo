# Pixel 10 Pro ＆ モバイルOS 互換性ガイド

このガイドは、Pixel 10 Pro（Android 15以降）および iOS の互換性をターゲットとした React Native (Expo) 開発における具体的なガイドラインと実装パターンを定義します。

---

## 1. エッジ・トゥ・エッジ（Edge-to-Edge）表示 (Android 15+)

Android 15 では、デフォルトで「Edge-to-Edge（エッジ・トゥ・エッジ）」表示が強制されます。アプリはシステムステータスバーやナビゲーションバーの背面（画面全体）に描画されます。

### ガイドライン
*   **`SafeAreaView` または `expo-safe-area-context` の使用:** 画面のメインボディやスクロールビューを `SafeAreaView` でラップし、ボタンやテキスト入力などの操作可能なUI要素がシステムナビゲーションバーやステータスバーと重ならないようにします。
    - React Native 標準の `SafeAreaView` よりも、より正確なインセット制御が可能な `react-native-safe-area-context`（Expoの標準パッケージ）の使用を推奨します。
*   **ステータスバーのスタイル設定:**
    - `expo-status-bar` の `StatusBar` コンポーネントを適切に配置し、背景色を透明に、コンテンツ（アイコン等）を明暗に応じて切り替えます。
*   **パディング値のハードコーディングの禁止:** Pixel 10 Pro などのカメラホール（パンチホール）や角丸ディスプレイ、ナビゲーションバーの動的な高さを考慮するため、パディングを固定値にせず、`useSafeAreaInsets` フックを使用します。

```tsx
import React from 'react';
import { StyleSheet, Text, View } from 'react-native';
import { SafeAreaProvider, useSafeAreaInsets } from 'react-native-safe-area-context';
import { StatusBar } from 'expo-status-bar';

function HomeScreenContent() {
  const insets = useSafeAreaInsets();

  return (
    <View style={[styles.container, { paddingTop: insets.top, paddingBottom: insets.bottom }]}>
      <StatusBar style="auto" translucent />
      <Text>コンテンツエリア</Text>
    </View>
  );
}

export default function App() {
  return (
    <SafeAreaProvider>
      <HomeScreenContent />
    </SafeAreaProvider>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
});
```

---

## 2. Android フォトピッカー ＆ メディア権限

Android 14/15 では、広範なメディア権限（`READ_EXTERNAL_STORAGE` など）の要求は非推奨（かつ困難に）なっています。

### ガイドライン
*   **`expo-image-picker` の使用:** 最新バージョンの `expo-image-picker` を使用します。
*   **ストレージ権限要求の不要化:** アルバムから写真を選択する際、`ImagePicker.launchImageLibraryAsync()` を使用すると、Android 14/15 のネイティブフォトピッカーが自動的に起動します。この仕組みでは、アプリはシステム側が提供するピッカー UI を経由するため、アプリ自体に**ストレージ権限（ストレージアクセスのパーミッション）を宣言・要求する必要がありません**。
*   **カメラ権限の要求:** ただし、カメラを起動してその場で写真を撮影する場合は、カメラの利用権限（`CAMERA`）を明示的に要求する必要があります（`ImagePicker.requestCameraPermissionsAsync()` を使用）。
*   **iOS 互換性:** iOS では、カメラ起動や写真ライブラリへのアクセスのために `app.json` の `plugins` 設定に `expo-image-picker` を指定し、説明文（`cameraPermission` や `photosPermission`）を定義する必要があります。

---

## 3. カメラ連携 ＆ メモリ負荷への対策

Pixel 10 Pro などの高解像度カメラ（最大50MP）で撮影された写真はファイルサイズが非常に大きく、そのままメモリ上に読み込んだりデコードしたりすると、メモリ負荷（Out of Memory）によるアプリの強制終了を引き起こす可能性があります。

### ガイドライン
*   **画像圧縮とリサイズ:** 写真を撮影・選択する際、必ず最大解像度の制限と品質の圧縮を指定して、ファイルサイズを抑えます。
    ```typescript
    import * as ImagePicker from 'expo-image-picker';

    const pickImage = async () => {
      const result = await ImagePicker.launchCameraAsync({
        allowsEditing: true,     // トリミングを有効に
        aspect: [4, 3],          // アスペクト比
        quality: 0.8,            // 画質の圧縮 (0.0 〜 1.0)
      });
      
      if (!result.canceled) {
        const imageUri = result.assets[0].uri;
        // 処理の続行
      }
    };
    ```
*   **アプリ復帰時の状態復元 (State Restoration):** Android OS は、カメラ起動中にアプリがバックグラウンドに回った際、メモリ解放のためにメインのアクティビティを破棄することがあります。Expo (React Native) では、カメラから復帰した際にアプリ状態が失われる現象を防ぐため、写真の保存はコンポーネントの状態（State）だけに依存せず、一時的な永続化、またはアクティビティ復旧ライフサイクルを意識した設計にします。

---

## 4. iOS サンドボックス ＆ 動的ファイルパス

iOS のサンドボックス構造では、**アプリをアップデートまたは再起動するたびに、アプリのドキュメントディレクトリの UUID（パスの一部）が動的に変更されます**。
そのため、データベースに絶対パス（例: `/var/mobile/Containers/Data/Application/UUID-XXXX-XXXX/Documents/image.jpg`）を保存してしまうと、次回のアプリ起動時に写真を読み込めなくなります。

### ガイドライン
*   **相対パスのみを保存:** データベースには絶対パスを保存せず、ドキュメントディレクトリからの相対パス（例: `memories/event_123.jpg`）またはファイル名のみを保存します。
*   **実行時に絶対パスを再構築:** 画像を読み込む際に、`expo-file-system` の `FileSystem.documentDirectory` から現在のベースパスを取得し、保存されている相対パスと結合します。

```typescript
import * as FileSystem from 'expo-file-system';

// 絶対パスの構築
export function getAbsoluteUri(relativePath: string): string {
  if (!relativePath) return '';
  if (relativePath.startsWith('http') || relativePath.startsWith('file://')) {
    return relativePath;
  }
  return `${FileSystem.documentDirectory}${relativePath}`;
}

// ファイルの保存処理（テンポラリ領域からドキュメント領域へコピー）
export async function saveFileToDocumentDirectory(tempUri: string, fileName: string): Promise<string> {
  const targetDir = `${FileSystem.documentDirectory}memories/`;
  
  const dirInfo = await FileSystem.getInfoAsync(targetDir);
  if (!dirInfo.exists) {
    await FileSystem.makeDirectoryAsync(targetDir, { intermediates: true });
  }

  const relativePath = `memories/${fileName}`;
  const targetUri = `${FileSystem.documentDirectory}${relativePath}`;

  await FileSystem.copyAsync({
    from: tempUri,
    to: targetUri,
  });

  return relativePath; // DBにはこの相対パスを保存する
}
```
