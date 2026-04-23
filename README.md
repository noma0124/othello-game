# オセロ PWA → TWA → Google Play 公開手順

## 構成ファイル
```
othello_twa/
├── index.html      ← ゲーム本体（全ロジック含む）
├── manifest.json   ← PWA設定
├── sw.js           ← Service Worker（オフライン対応）
├── icons/
│   ├── icon-192.png  ← 要作成
│   └── icon-512.png  ← 要作成
└── README.md
```

---

## STEP 1: Webサーバーに公開する

### 無料でできる方法（推奨: GitHub Pages）
1. GitHubアカウントを作成 → https://github.com
2. 新しいリポジトリを作成（例: `othello-game`）
3. このフォルダのファイルをアップロード
4. Settings → Pages → Branch: main → Save
5. `https://あなたのID.github.io/othello-game/` で公開される

### アイコン作成（必須）
- https://realfavicongenerator.net などで 512×512 の PNG を作成
- `icons/icon-192.png`（192×192px）
- `icons/icon-512.png`（512×512px）
- 緑の背景に白い丸（オセロ石）のデザインが映えます

---

## STEP 2: HTTPS確認 & Digital Asset Links

### assetlinks.json の設置
TWA化するには、サイトに以下のファイルを設置する必要があります：

`https://あなたのサイト/.well-known/assetlinks.json`

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.yourname.othello",
    "sha256_cert_fingerprints": ["SHA256フィンガープリント（後述）"]
  }
}]
```

---

## STEP 3: Bubblewrap で TWA プロジェクト生成

### 必要なもの
- Node.js 18以上: https://nodejs.org
- Java JDK 11以上: https://adoptium.net
- Android Studio: https://developer.android.com/studio

### インストール
```bash
npm install -g @bubblewrap/cli
```

### TWAプロジェクト生成
```bash
mkdir othello-twa-android
cd othello-twa-android
bubblewrap init --manifest https://あなたのサイト/manifest.json
```

対話形式で以下を入力：
- **Application ID**: `com.yourname.othello`
- **App name**: `オセロ`
- **Display**: `standalone`
- **Status bar color**: `#0a1f12`
- **Nav bar color**: `#0a1f12`

### ビルド
```bash
bubblewrap build
```
→ `app-release-signed.apk` と `app-release-bundle.aab` が生成されます

---

## STEP 4: AdMob 広告の組み込み（TWA版）

TWAアプリにAdMob広告を入れる方法は2種類あります：

### 方法A: WebView内のAdSense（簡単）
`index.html` の広告コメントアウト部分を有効化：
```html
<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-XXXXXXXXXXXXXXXX" crossorigin="anonymous"></script>
```
- AdSense申請が必要（サイトのトラフィックが必要なため審査に時間がかかる場合あり）

### 方法B: ネイティブAdMob（推奨・収益高）
bubblewrapが生成したAndroidプロジェクトに手動でAdMob SDKを追加：

1. `app/build.gradle` に追加：
```gradle
implementation 'com.google.android.gms:play-services-ads:23.0.0'
```

2. `AndroidManifest.xml` に追加：
```xml
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY"/>
```

3. `MainActivity.java` / `MainActivity.kt` でバナー広告初期化

4. JavaScriptブリッジを使いリワード広告をWebViewから呼び出す

---

## STEP 5: Google Play Console で申請

1. https://play.google.com/console にアクセス
2. デベロッパー登録（$25 一回のみ）
3. 「アプリを作成」
4. 「リリース」→「製品版」→「新しいリリース」
5. `.aab` ファイルをアップロード
6. 必要事項を入力：
   - ストアの説明
   - スクリーンショット（電話用: 最低2枚）
   - アイコン（512×512px）
   - フィーチャーグラフィック（1024×500px）
7. コンテンツレーティング（アンケートに答える）
8. 「審査に送信」

**審査期間**: 初回は3日〜1週間程度

---

## CPU強さの詳細

| LV | 戦略 | 特徴 |
|----|------|------|
| 1-3 | ランダム | 完全ランダム |
| 4-8 | 貪欲法 | 1手先を評価 |
| 9-10 | Minimax 2深 + ミス35% | 弱め |
| 11-15 | Minimax 3深 + ミス15% | 中級 |
| 16-20 | Minimax 4深 + ミス10% | 上級 |
| 21-25 | Minimax 5深 + ミス5% | 高段 |
| 26-30 | Minimax 6深 + ミス0% | 最強 |

評価関数: 位置価値 + 手数（モビリティ） + 終盤石数

---

## よくある質問

**Q: 広告が表示されない（開発中）**
A: ブラウザで直接開いた場合、AdSenseは審査通過後のみ表示。現在はモック広告UIが表示されます。

**Q: iOSにも出したい**
A: iOS版はPWAをSafariから「ホーム画面に追加」で疑似アプリ化できます。App Storeへの正式公開はXcodeが必要です。

**Q: オフラインで動く？**
A: Service Workerにより、一度読み込んだ後はオフラインでも動作します。
