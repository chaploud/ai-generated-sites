# JavaScriptブラウザAPI包括的学習ガイド（2025年版）

現代のWebブラウザで利用可能なJavaScript APIの体系的な一覧です。2025年時点で利用可能な基本的なAPIから最新のWeb APIまで、学習に最適な形式で整理しています。

## DOM操作API

### 基本的なDOM選択・操作
**querySelector/querySelectorAll** - CSSセレクタによる要素選択
```javascript
const element = document.querySelector('.myclass');
const elements = document.querySelectorAll('div.container p');
```

**getElementById/getElementsByClassName** - ID/クラス名による要素取得（高速）
```javascript
const header = document.getElementById('main-header');
const buttons = document.getElementsByClassName('btn');
```

**createElement/appendChild** - 要素の動的生成と追加
```javascript
const div = document.createElement('div');
div.textContent = 'Hello';
document.body.appendChild(div);
```

**classList API** - クラス操作の現代的手法
```javascript
element.classList.add('active');
element.classList.toggle('visible');
element.classList.contains('hidden');
```

**insertAdjacentElement/HTML** - 柔軟な要素挿入
```javascript
element.insertAdjacentHTML('beforebegin', '<div>New</div>');
element.insertAdjacentElement('afterend', newElement);
```

### 高度なDOM機能
**Shadow DOM** - カプセル化されたDOM構造（Web Components基盤）
```javascript
const shadow = element.attachShadow({ mode: 'open' });
shadow.innerHTML = '<style>p { color: red; }</style><p>Shadow content</p>';
```

**Custom Elements** - カスタムHTML要素の定義
```javascript
class MyElement extends HTMLElement {
  connectedCallback() {
    this.innerHTML = '<h1>Custom Element</h1>';
  }
}
customElements.define('my-element', MyElement);
```

**Popover API** (2024年安定版) - ネイティブポップオーバー機能
```javascript
const popover = document.querySelector('#popover');
popover.showPopover();
popover.hidePopover();
```

**View Transitions API** (最新) - DOM状態間のスムーズな遷移
```javascript
document.startViewTransition(() => {
  updateDOM();
});
```

## ストレージAPI

### 基本ストレージ
**localStorage/sessionStorage** - 同期的なキー・バリューストレージ（5MB制限）
```javascript
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');
sessionStorage.setItem('tempData', JSON.stringify(data));
```

**Cookies API** - HTTPクッキー管理
```javascript
document.cookie = "user=john; expires=Thu, 18 Dec 2025 12:00:00 UTC";
// Cookie Store API (Chrome)
await cookieStore.set({ name: 'session', value: 'abc123' });
```

### 高度なストレージ
**IndexedDB** - 大容量構造化データストレージ
```javascript
const request = indexedDB.open('MyDB', 1);
request.onsuccess = (event) => {
  const db = event.target.result;
  const transaction = db.transaction(['users'], 'readwrite');
  const store = transaction.objectStore('users');
  store.add({ id: 1, name: 'John' });
};
```

**Cache API** - ServiceWorkerキャッシュ管理
```javascript
const cache = await caches.open('v1');
await cache.add('/api/data.json');
const response = await cache.match('/api/data.json');
```

**Origin Private File System (OPFS)** - 高性能ファイルシステム
```javascript
const root = await navigator.storage.getDirectory();
const fileHandle = await root.getFileHandle('data.txt', { create: true });
const writable = await fileHandle.createWritable();
await writable.write('File content');
```

## メディア・オーディオ・ビデオAPI

### 基本メディア制御
**HTMLMediaElement** - audio/video要素の基本インターフェース
```javascript
const video = document.querySelector('video');
video.play();
video.pause();
video.currentTime = 30; // 30秒地点へシーク
```

**Picture-in-Picture API** - フローティングビデオウィンドウ
```javascript
await video.requestPictureInPicture();
await document.exitPictureInPicture();
```

### ストリーミング・処理
**MediaStream API** - カメラ/マイクアクセス
```javascript
const stream = await navigator.mediaDevices.getUserMedia({ 
  video: true, 
  audio: true 
});
video.srcObject = stream;
```

**MediaRecorder API** - メディア録画
```javascript
const recorder = new MediaRecorder(stream);
recorder.start();
recorder.ondataavailable = (e) => chunks.push(e.data);
```

**WebRTC API** - P2P通信
```javascript
const pc = new RTCPeerConnection();
pc.addTrack(track, stream);
pc.ontrack = (e) => remoteVideo.srcObject = e.streams[0];
```

### オーディオ処理
**Web Audio API** - 高度な音声処理
```javascript
const audioContext = new AudioContext();
const oscillator = audioContext.createOscillator();
oscillator.frequency.value = 440; // A4音
oscillator.connect(audioContext.destination);
oscillator.start();
```

**Speech APIs** - 音声合成・認識
```javascript
// 音声合成
const utterance = new SpeechSynthesisUtterance('Hello World');
speechSynthesis.speak(utterance);

// 音声認識 (Chrome)
const recognition = new webkitSpeechRecognition();
recognition.onresult = (e) => console.log(e.results[0][0].transcript);
```

### 最新メディア機能
**WebCodecs API** - 低レベルビデオ/オーディオ処理
```javascript
const encoder = new VideoEncoder({
  output: handleEncodedChunk,
  error: console.error
});
encoder.configure({ codec: 'vp8', width: 640, height: 480 });
```

## ネットワーク・通信API

### HTTP通信
**Fetch API** - モダンなHTTPリクエスト
```javascript
const response = await fetch('/api/data', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' })
});
const data = await response.json();
```

### リアルタイム通信
**WebSocket API** - 双方向通信
```javascript
const socket = new WebSocket('ws://localhost:8080');
socket.onmessage = (event) => console.log(event.data);
socket.send('Hello Server');
```

**Server-Sent Events** - サーバープッシュ
```javascript
const eventSource = new EventSource('/events');
eventSource.onmessage = (event) => console.log(event.data);
```

**WebTransport API** (最新) - HTTP/3ベースの高速通信
```javascript
const transport = new WebTransport('https://example.com:4999/wt');
await transport.ready;
const stream = await transport.createBidirectionalStream();
```

### ServiceWorker関連
**Service Worker API** - オフライン対応・バックグラウンド処理
```javascript
navigator.serviceWorker.register('/sw.js');
// sw.js内
self.addEventListener('fetch', (event) => {
  event.respondWith(caches.match(event.request) || fetch(event.request));
});
```

**Push API** - プッシュ通知
```javascript
const subscription = await registration.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: vapidPublicKey
});
```

**Background Sync API** - オフライン同期
```javascript
await registration.sync.register('background-sync');
```

## グラフィックス・描画API

### 2Dグラフィックス
**Canvas 2D Context** - 2D描画
```javascript
const ctx = canvas.getContext('2d');
ctx.fillStyle = 'red';
ctx.fillRect(10, 10, 100, 100);
ctx.beginPath();
ctx.arc(100, 100, 50, 0, Math.PI * 2);
ctx.fill();
```

**Path2D API** - 再利用可能なパス
```javascript
const path = new Path2D();
path.rect(10, 10, 100, 100);
ctx.fill(path);
```

**OffscreenCanvas** - ワーカーでのCanvas処理
```javascript
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);
```

### 3Dグラフィックス
**WebGL/WebGL2** - ハードウェアアクセラレーション3D
```javascript
const gl = canvas.getContext('webgl2');
gl.clearColor(0.0, 0.0, 0.0, 1.0);
gl.clear(gl.COLOR_BUFFER_BIT);
```

**WebGPU API** (2025年最新) - 次世代GPU API
```javascript
const adapter = await navigator.gpu?.requestAdapter();
const device = await adapter?.requestDevice();
const context = canvas.getContext('webgpu');
```

### アニメーション
**Web Animations API** - JavaScriptアニメーション
```javascript
element.animate([
  { transform: 'translateX(0px)' },
  { transform: 'translateX(100px)' }
], {
  duration: 1000,
  iterations: Infinity
});
```

**requestAnimationFrame** - スムーズなアニメーションループ
```javascript
function animate(timestamp) {
  // アニメーション処理
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

## デバイス・センサーAPI

### メディアデバイス
**MediaDevices API** - カメラ/マイクアクセス
```javascript
const devices = await navigator.mediaDevices.enumerateDevices();
const stream = await navigator.mediaDevices.getUserMedia({
  video: { width: 1280, height: 720 }
});
```

### 位置・方向
**Geolocation API** - GPS位置情報
```javascript
navigator.geolocation.getCurrentPosition(position => {
  console.log(position.coords.latitude, position.coords.longitude);
});
```

**Screen Orientation API** - 画面向き制御
```javascript
screen.orientation.lock('portrait');
console.log(screen.orientation.type);
```

### センサー (Chrome中心)
**Accelerometer/Gyroscope API** - 加速度/ジャイロセンサー
```javascript
const accelerometer = new Accelerometer({ frequency: 60 });
accelerometer.addEventListener('reading', () => {
  console.log(accelerometer.x, accelerometer.y, accelerometer.z);
});
accelerometer.start();
```

### デバイス機能
**Vibration API** - 振動制御
```javascript
navigator.vibrate([200, 100, 200]); // パターン振動
```

**Battery Status API** - バッテリー状態
```javascript
const battery = await navigator.getBattery();
console.log(battery.level * 100 + '%');
```

**Screen Wake Lock API** - 画面スリープ防止
```javascript
const wakeLock = await navigator.wakeLock.request('screen');
```

### 接続性
**Web Bluetooth API** - Bluetooth機器接続
```javascript
const device = await navigator.bluetooth.requestDevice({
  filters: [{ services: ['battery_service'] }]
});
```

**WebUSB API** - USB機器アクセス
```javascript
const device = await navigator.usb.requestDevice({ 
  filters: [{ vendorId: 0x2341 }] 
});
```

**WebNFC API** (Android Chrome) - NFC通信
```javascript
const reader = new NDEFReader();
await reader.scan();
```

## セキュリティ・認証API

### 認証
**Web Authentication API (WebAuthn)** - パスワードレス認証
```javascript
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: new Uint8Array(32),
    rp: { name: "Example Corp" },
    user: { id: new Uint8Array(16), name: "user@example.com" },
    pubKeyCredParams: [{ type: "public-key", alg: -7 }]
  }
});
```

**Credential Management API** - 認証情報管理
```javascript
const credential = await navigator.credentials.get({
  password: true,
  federated: { providers: ['https://accounts.google.com'] }
});
```

### 暗号化
**Web Crypto API** - 暗号化処理
```javascript
const key = await crypto.subtle.generateKey(
  { name: "AES-GCM", length: 256 },
  true,
  ["encrypt", "decrypt"]
);
const encrypted = await crypto.subtle.encrypt(
  { name: "AES-GCM", iv: new Uint8Array(12) },
  key,
  data
);
```

**crypto.getRandomValues()** - 暗号学的に安全な乱数
```javascript
const array = new Uint8Array(32);
crypto.getRandomValues(array);
```

### セキュリティポリシー
**Content Security Policy (CSP)** - XSS防止
```javascript
// CSP違反の監視
document.addEventListener('securitypolicyviolation', (e) => {
  console.warn('CSP Violation:', e.violatedDirective);
});
```

**Permissions API** - 権限状態の確認
```javascript
const result = await navigator.permissions.query({ name: 'geolocation' });
console.log(result.state); // 'granted', 'denied', 'prompt'
```

## パフォーマンス・最適化API

### 測定
**Performance API** - パフォーマンス測定
```javascript
const start = performance.now();
// 処理
const end = performance.now();
console.log(`処理時間: ${end - start}ms`);
```

**PerformanceObserver** - パフォーマンス監視
```javascript
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log(entry.name, entry.duration);
  });
});
observer.observe({ entryTypes: ['measure', 'navigation'] });
```

### Core Web Vitals
**LCP/INP/CLS測定** - ユーザー体験指標
```javascript
// Largest Contentful Paint
new PerformanceObserver((list) => {
  const entries = list.getEntries();
  console.log('LCP:', entries[entries.length - 1].startTime);
}).observe({ entryTypes: ['largest-contentful-paint'] });
```

### 最適化
**requestIdleCallback** - アイドル時の処理実行
```javascript
requestIdleCallback((deadline) => {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    performTask(tasks.shift());
  }
});
```

**Scheduler API** (新) - タスク優先度制御
```javascript
scheduler.postTask(() => {
  // 高優先度タスク
}, { priority: 'user-blocking' });
```

### Worker APIs
**Web Workers** - バックグラウンド処理
```javascript
const worker = new Worker('worker.js');
worker.postMessage({ data: largeDataSet });
worker.onmessage = (e) => console.log(e.data);
```

**SharedWorker** - タブ間共有ワーカー
```javascript
const sharedWorker = new SharedWorker('shared-worker.js');
sharedWorker.port.postMessage('Hello');
```

## その他のブラウザAPI

### システム統合
**Clipboard API** - クリップボードアクセス
```javascript
await navigator.clipboard.writeText('Hello');
const text = await navigator.clipboard.readText();
```

**Web Share API** - ネイティブ共有
```javascript
await navigator.share({
  title: 'Check this out',
  url: 'https://example.com'
});
```

**File System Access API** - ファイルシステムアクセス
```javascript
const [fileHandle] = await window.showOpenFilePicker();
const file = await fileHandle.getFile();
const contents = await file.text();
```

**Notification API** - システム通知
```javascript
const permission = await Notification.requestPermission();
if (permission === 'granted') {
  new Notification('Hello!', { body: 'Notification body' });
}
```

### 状態管理
**History API** - ブラウザ履歴操作
```javascript
history.pushState({ page: 1 }, 'Title', '/page1');
window.addEventListener('popstate', (event) => {
  console.log(event.state);
});
```

**Navigation API** (新) - モダンなナビゲーション制御
```javascript
navigation.addEventListener('navigate', (event) => {
  event.intercept({
    async handler() {
      await loadPage(event.destination.url);
    }
  });
});
```

### データ処理
**Streams API** - ストリームデータ処理
```javascript
const response = await fetch('/large-file');
const reader = response.body.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  processChunk(value);
}
```

**TextEncoder/TextDecoder** - テキストエンコーディング
```javascript
const encoder = new TextEncoder();
const uint8Array = encoder.encode('Hello World');
const decoder = new TextDecoder();
const string = decoder.decode(uint8Array);
```

**Blob/File API** - バイナリデータ処理
```javascript
const blob = new Blob(['Hello'], { type: 'text/plain' });
const url = URL.createObjectURL(blob);
```

## 最新・実験的API (2024-2025)

### AI・機械学習
**WebNN API** (開発中) - ニューラルネットワーク推論
```javascript
if ('ml' in navigator) {
  const context = await navigator.ml.createContext();
  const builder = new MLGraphBuilder(context);
}
```

**Shape Detection API** (実験的) - 顔・バーコード・テキスト検出
```javascript
const barcodeDetector = new BarcodeDetector();
const barcodes = await barcodeDetector.detect(image);
```

### プライバシー保護
**Topics API** (Chrome) - プライバシー保護型広告
```javascript
const topics = await document.browsingTopics();
```

**Storage Access API** - サードパーティストレージアクセス
```javascript
const hasAccess = await document.hasStorageAccess();
if (!hasAccess) {
  await document.requestStorageAccess();
}
```

### 次世代機能
**Speculation Rules API** - 先読み最適化
```html
<script type="speculationrules">
{
  "prefetch": [{
    "where": { "href_matches": "/product/*" },
    "eagerness": "moderate"
  }]
}
</script>
```

**Compute Pressure API** (実験的) - システム負荷監視
```javascript
const observer = new PressureObserver((changes) => {
  console.log('CPU pressure:', changes[0].state);
});
observer.observe('cpu');
```

**CSS Anchor Positioning API** - 要素相対配置
```css
.tooltip {
  position-anchor: --my-anchor;
  top: calc(anchor(bottom) + 10px);
}
```

## ブラウザサポート状況（2025年現在）

### 広くサポートされているAPI（95%以上）
- DOM操作基本API
- Web Storage (localStorage/sessionStorage)
- Fetch API
- Canvas 2D
- Web Audio API基本機能
- Geolocation API

### 良好なサポート（80-90%）
- Service Worker
- WebRTC
- IndexedDB
- Web Animations API
- Clipboard API

### 限定的サポート（Chrome/Edge中心）
- WebGPU
- WebUSB/WebBluetooth
- File System Access API
- WebNN (AI/ML)
- Compute Pressure API

### 実装のベストプラクティス

1. **機能検出を必ず行う**
```javascript
if ('serviceWorker' in navigator) {
  // Service Worker対応
}
if ('Translator' in self) {
  // Translation API対応
}
```

2. **プログレッシブエンハンスメント**
- 基本機能を確保し、利用可能なAPIで段階的に機能強化
- Translation APIは初回使用時にモデルダウンロードが必要

3. **セキュリティ考慮**
- HTTPS必須のAPIが多い
- ユーザー権限が必要なAPIは適切に処理

4. **パフォーマンス最適化**
- 重い処理はWorkerで実行
- requestIdleCallbackで優先度管理
- Translation APIは長文をチャンクに分割して処理

この包括的なガイドは、2025年時点で利用可能なブラウザAPIを体系的に整理し、実用的な学習リソースとして活用できるよう構成されています。各APIの基本的な使用例とブラウザサポート状況を含め、モダンなWeb開発に必要な知識を網羅しています。