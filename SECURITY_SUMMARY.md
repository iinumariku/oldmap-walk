# セキュリティ調査レポート

## 調査日時
2026-02-14

## 概要
oldmap-walkリポジトリのセキュリティリスクを調査し、検出された脆弱性を修正しました。

## 検出・修正された脆弱性

### 1. パッケージの脆弱性（npm audit）

#### 修正済み（9件）
- `@eslint/plugin-kit`: Regular Expression Denial of Service (ReDoS) 脆弱性
- `@modelcontextprotocol/sdk`: DNS rebinding保護・ReDoS・データリーク
- `body-parser`: Denial of Service (DoS) 脆弱性
- `brace-expansion`: Regular Expression Denial of Service
- `devalue`: Prototype pollution・DoS脆弱性
- `js-yaml`: Prototype pollution
- `qs`: DoS脆弱性（配列制限バイパス）
- `tar`: ファイル上書き・パストラバーサル脆弱性
- `vite`: ファイル提供・fs設定バイパスの脆弱性

**対策**: `npm audit fix`を実行し、互換性のある範囲で最新バージョンに更新

#### 残存する脆弱性（3件 - 低リスク）
- `cookie` (< 0.7.0): cookie名、パス、ドメインに範囲外文字を受け入れる
- `@sveltejs/kit`: cookieパッケージへの依存関係による間接的な影響

**影響**: 低リスク。修正には破壊的変更が必要（@sveltejs/kit 0.0.30へのダウングレード）
**推奨対応**: 次期メジャーバージョンアップデート時に対応

### 2. CSV出力のインジェクション脆弱性（HIGH）

#### 問題点
CSV出力時に特殊文字のエスケープが不足しており、以下のリスクがありました：
- 数式インジェクション: `=`, `+`, `-`, `@`で始まるテキストが表計算ソフトで実行される
- CSV構造の破壊: カンマ、改行、ダブルクォートにより不正な形式のCSVが生成される

#### 修正内容
```javascript
function escapeCSV(field: string | number): string {
    const str = String(field);
    const needsQuotes = str.includes(',') || str.includes('"') || 
                       str.includes('\n') || str.includes('\r');
    const hasFormulaChar = /^[=+\-@]/.test(str);
    
    // 内部のダブルクォートをエスケープ
    const escaped = str.replace(/"/g, '""');
    
    // 数式文字で始まる場合、シングルクォートをプレフィックス
    if (hasFormulaChar) {
        return `"'${escaped}"`;
    }
    
    // 特殊文字を含む場合、ダブルクォートで囲む
    if (needsQuotes) {
        return `"${escaped}"`;
    }
    
    return str;
}
```

**影響**: すべてのメモデータのCSVエクスポート機能
**リスクレベル**: HIGH → 修正済み

### 3. LocalStorageの検証不足（MEDIUM）

#### 問題点
- `JSON.parse()`のエラーハンドリングが不足
- 読み込んだデータの型・範囲検証がない
- 破損したデータによるアプリケーションクラッシュのリスク

#### 修正内容
```javascript
function loadMemos() {
    try {
        const savedMemos = localStorage.getItem('memos');
        if (savedMemos) {
            const parsed = JSON.parse(savedMemos);
            
            // データ構造と型を検証
            if (Array.isArray(parsed)) {
                const validated = parsed.filter(memo => 
                    memo && 
                    typeof memo.id === 'string' &&
                    typeof memo.lat === 'number' &&
                    typeof memo.lng === 'number' &&
                    typeof memo.text === 'string' &&
                    typeof memo.date === 'string' &&
                    isFinite(memo.lat) &&
                    isFinite(memo.lng) &&
                    memo.lat >= -90 && memo.lat <= 90 &&
                    memo.lng >= -180 && memo.lng <= 180
                );
                
                memos = validated.map((memo: Memo) => ({
                    ...memo, 
                    text: String(memo.text).slice(0, 1000),
                    marker: createMarker(memo)
                }));
            }
        }
    } catch (error) {
        console.error('メモの読み込みに失敗しました:', error);
        localStorage.removeItem('memos'); // 破損データを削除
    }
}
```

**影響**: すべてのメモデータの保存・読み込み機能
**リスクレベル**: MEDIUM → 修正済み

### 4. 位置情報APIのエラーハンドリング不足（MEDIUM）

#### 問題点
- `getCurrentPosition()`にエラーコールバックがない
- `watchPosition()`のエラーハンドリングが不十分
- タイムアウト設定がない

#### 修正内容
```javascript
navigator.geolocation.getCurrentPosition(
    (position) => { /* 成功処理 */ },
    (error) => {
        // エラータイプ別の処理
        switch(error.code) {
            case error.PERMISSION_DENIED:
                console.warn('位置情報のアクセスが拒否されました');
                break;
            case error.POSITION_UNAVAILABLE:
                console.warn('位置情報が利用できません');
                break;
            case error.TIMEOUT:
                console.warn('位置情報の取得がタイムアウトしました');
                break;
        }
    },
    { timeout: 10000, maximumAge: 300000 }
);
```

**影響**: 現在地表示・トラッキング機能
**リスクレベル**: MEDIUM → 修正済み

### 5. 入力検証の不足（LOW）

#### 問題点
- メモ入力に長さ制限がない
- 入力のサニタイゼーションがない
- 空のメモが保存される可能性

#### 修正内容
1. HTML入力フィールドに`maxlength="1000"`属性を追加
2. メモ追加・更新時にバリデーション追加：
```javascript
async function addMemo() {
    // 入力を検証・サニタイズ
    const sanitizedText = currentMemo.trim().slice(0, 1000);
    if (!sanitizedText) {
        console.warn('メモが空です');
        return;
    }
    // メモ保存処理
}
```

**影響**: メモの追加・編集機能
**リスクレベル**: LOW → 修正済み

## XSS脆弱性について

### 調査結果
Svelteは`{variable}`構文でデフォルトで自動エスケープを行うため、XSS脆弱性のリスクは低いです。

**安全な箇所**:
```svelte
<div>{memo.text}</div> <!-- 自動エスケープされる -->
```

**注意が必要な箇所**:
将来的に`{@html memo.text}`のような非エスケープHTMLレンダリングを使用しないよう注意が必要です。

## 推奨される追加対策

### 1. Content Security Policy (CSP) の実装
`src/app.html`にCSPヘッダーを追加することを推奨：
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               script-src 'self' 'unsafe-inline'; 
               style-src 'self' 'unsafe-inline'; 
               img-src 'self' data: https:; 
               connect-src 'self' https://cyberjapandata.gsi.go.jp https://tile.openstreetmap.org https://ktgis.net;">
```

### 2. ユーザー通知の改善
位置情報取得エラー時に、ユーザーにわかりやすいメッセージを表示することを推奨。

### 3. データサイズの制限
LocalStorageには5MBの制限があります。メモ数が増えた場合の対策：
- メモ数の上限を設定（例：100件）
- 古いメモを自動削除する機能

### 4. HTTPS の強制
本番環境では必ずHTTPSを使用してください。位置情報APIはHTTPSが必須です。

## 結論

このセキュリティ調査により、以下の脆弱性を修正しました：
- **HIGH**: 1件（CSV injection）
- **MEDIUM**: 2件（localStorage検証、位置情報エラーハンドリング）
- **LOW**: 1件（入力検証）
- **パッケージ脆弱性**: 9件修正、3件残存（低リスク）

アプリケーションのセキュリティレベルは大幅に向上しました。残存する脆弱性は低リスクであり、次回のメジャーアップデート時に対応することを推奨します。

## 参考資料
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Svelte Security Best Practices](https://svelte.dev/docs/security)
- [MDN Geolocation API Security](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API)
