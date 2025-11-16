# FFME (FFmpeg MediaElement) 機能仕様書

## 1. プロジェクト概要

### 1.1 プロジェクト名
FFME: The Advanced WPF MediaElement Alternative

### 1.2 目的
WPFアプリケーション向けの高度なメディア再生コントロールを提供する。標準のWPF MediaElementの代替として、FFmpegライブラリを使用してより多くのメディアフォーマットとコーデックをサポートし、拡張機能を提供する。

### 1.3 対象プラットフォーム
- .NET 5.0以上
- Windows (WPF)
- x64 / x86アーキテクチャ

### 1.4 主要な技術スタック
- FFmpeg 7.0 (共有ライブラリ)
- WPF (Windows Presentation Foundation)
- C# / .NET

---

## 2. システムアーキテクチャ

### 2.1 レイヤー構造

#### 2.1.1 UIレイヤー
- **MediaElement**: WPFユーザーコントロール
  - 標準のWPF MediaElementと互換性のあるAPI
  - ビデオ、オーディオ、字幕の表示
  - クローズドキャプションのサポート

#### 2.1.2 エンジンレイヤー
- **MediaEngine**: メディア処理のコアエンジン
  - コマンド管理（再生、一時停止、停止、シーク）
  - 状態管理（MediaEngineState）
  - タイミング制御（TimingController）
  - 3つのバックグラウンドワーカー

#### 2.1.3 コンテナレイヤー
- **MediaContainer**: 入力ストリームのラッパー
  - ストリームの初期化とオープン
  - パケット読み取り
  - フレームデコード
  - ブロック変換

#### 2.1.4 コンポーネントレイヤー
- **MediaComponent**: メディアタイプ別の処理
  - VideoComponent: ビデオストリーム処理
  - AudioComponent: オーディオストリーム処理
  - SubtitleComponent: 字幕ストリーム処理

### 2.2 データフロー

```
入力ストリーム
    ↓
パケット読み取り (PacketReadingWorker)
    ↓
パケットキュー (PacketQueue)
    ↓
フレームデコード (FrameDecodingWorker)
    ↓
ブロック変換 (MediaBlock)
    ↓
ブロックバッファ (MediaBlockBuffer)
    ↓
レンダリング (BlockRenderingWorker)
    ↓
出力デバイス (画面/スピーカー)
```

---

## 3. 主要機能

### 3.1 メディア再生機能

#### 3.1.1 基本再生制御
- **再生 (Play)**
  - 非同期メソッド: `await Media.Open(uri)`
  - メディアソースのオープンと再生開始
  - サポート形式: ファイル、URL、ストリーム、キャプチャデバイス

- **一時停止 (Pause)**
  - 再生位置を保持したまま一時停止
  - リアルタイムクロック (RTC) の停止

- **停止 (Stop)**
  - 再生を停止し、位置を先頭にリセット

- **クローズ (Close)**
  - 非同期メソッド: `await Media.Close()`
  - リソースの解放とクリーンアップ

#### 3.1.2 シーク機能
- **高速シーク**
  - 任意の位置への即座の移動
  - キーフレームベースのシーク

- **フレーム単位シーク**
  - 1フレームずつの正確な移動
  - 左矢印/右矢印キーでの操作

- **シークインデックス**
  - `CreateVideoSeekIndex()`: ビデオシークインデックスの作成
  - イントラフレーム位置の事前計算

#### 3.1.3 再生制御パラメータ
- **Position** (依存関係プロパティ)
  - 現在の再生位置 (TimeSpan)
  - 双方向バインディング対応

- **SpeedRatio** (依存関係プロパティ)
  - 再生速度の調整 (0.0 ～ 100.0)
  - デフォルト: 1.0 (通常速度)
  - SoundTouch.dllでピッチ保持対応

- **Volume** (依存関係プロパティ)
  - 音量レベル (0.0 ～ 1.0)
  - デフォルト: 1.0

- **Balance** (依存関係プロパティ)
  - ステレオバランス (-1.0 ～ 1.0)
  - -1.0: 左、0.0: 中央、1.0: 右

- **IsMuted** (依存関係プロパティ)
  - ミュート状態のトグル

### 3.2 メディア情報取得

#### 3.2.1 メディアメタデータ
- **MediaInfo**
  - タイトル、アルバム、アーティスト
  - ビットレート、コーデック情報
  - ストリーム情報（ビデオ/オーディオ/字幕）
  - 継続時間、開始時間

- **静的メソッド**
  - `Library.RetrieveMediaInfo(mediaSource)`: メディア情報の取得

#### 3.2.2 ストリーム情報
- **StreamInfo**
  - コーデックタイプとID
  - ビットレート
  - タイムベース
  - 継続時間

#### 3.2.3 ビデオ情報
- 解像度 (NaturalVideoWidth, NaturalVideoHeight)
- フレームレート (VideoFrameRate)
- ピクセルアスペクト比
- ビデオコーデック (VideoCodec)
- ハードウェアデコーダー情報

#### 3.2.4 オーディオ情報
- サンプルレート (AudioSampleRate)
- チャンネル数 (AudioChannels)
- ビット深度 (AudioBitsPerSample)
- オーディオコーデック (AudioCodec)

### 3.3 高度な機能

#### 3.3.1 マルチストリーム対応
- **ストリーム選択**
  - 複数のビデオストリームから選択
  - 複数のオーディオストリームから選択
  - 複数の字幕ストリームから選択
  - ショートカットキー: A (オーディオ), Q (ビデオ), S (字幕)

#### 3.3.2 字幕とクローズドキャプション
- **字幕レンダリング**
  - 外部字幕ファイルのサポート
  - 埋め込み字幕のサポート
  - カスタマイズ可能なスタイル（フォント、色、サイズ）

- **クローズドキャプション**
  - CEA-608/708クローズドキャプション
  - チャンネル切り替え (ショートカットキー: C)

#### 3.3.3 フィルターグラフ
- **FFmpegフィルター適用**
  - ビデオフィルター（明るさ、コントラスト、彩度）
  - オーディオフィルター（イコライザー、エフェクト）
  - カスタムフィルターグラフの適用

#### 3.3.4 ハードウェアアクセラレーション
- **オプトインハードウェアデコード**
  - デバイスベースのアクセラレーション
  - コーデックベースのアクセラレーション
  - 対応: DXVA2, D3D11VA, CUDA, QSV

#### 3.3.5 キャプチャとレコーディング
- **スクリーンショット**
  - `CaptureBitmapAsync()`: 現在のフレームをGDIビットマップとしてキャプチャ
  - ショートカットキー: T

- **パケットレコーディング**
  - トランスコードなしでパケットを記録
  - Transport Stream形式での保存
  - ショートカットキー: W (開始/停止)

#### 3.3.6 カスタムストリーム処理
- **IMediaInputStream**
  - カスタム入力ストリームの実装
  - ストリーム読み取りのカスタマイズ
  - シーク動作のカスタマイズ

- **フレームインターセプション**
  - パケット、フレーム、ブロックのキャプチャ
  - レンダリング前のデータ変更
  - カスタムレンダリングパイプライン

### 3.4 キャプチャデバイス対応

#### 3.4.1 サポートされるデバイス
- **DirectShow**
  - Webカメラ
  - マイク
  - キャプチャカード

- **GDIGrab**
  - デスクトップキャプチャ
  - ウィンドウキャプチャ

#### 3.4.2 デバイスURL形式
```
device://dshow/?audio=Microphone:video=Webcam
device://gdigrab?title=WindowTitle
device://gdigrab?desktop
```

---

## 4. 技術仕様

### 4.1 パフォーマンス要件

#### 4.1.1 バッファリング
- **パケットバッファ**
  - 目標: 約1秒分のメディアパケット
  - 動的調整: ネットワーク状況に応じて

- **ブロックバッファ**
  - ビデオ: 最小8ブロック（設定可能）
  - オーディオ: 最小48ブロック（設定可能）
  - 字幕: 最小4ブロック（設定可能）

#### 4.1.2 レンダリング
- **フレームレート**
  - デフォルトタイミング周期: 15ms
  - 垂直同期 (VSync) オプション対応
  - 適応的フレームスキップ

- **同期バッファリング**
  - コンポーネント間の同期維持
  - 最大オフセット: 10秒

### 4.2 メモリ管理

#### 4.2.1 アンマネージドメモリ
- **FFmpegリソース**
  - AVFormatContext
  - AVCodecContext
  - AVFrame / AVPacket
  - 自動リソース追跡 (RC.Current)

#### 4.2.2 マネージドメモリ
- **ブロックバッファ**
  - 事前割り当て済みバッファの再利用
  - メモリプール管理
  - 明示的なDispose呼び出し

### 4.3 スレッドモデル

#### 4.3.1 バックグラウンドワーカー
- **PacketReadingWorker**
  - 優先度: Normal
  - 間隔: 動的（パケットバッファに基づく）

- **FrameDecodingWorker**
  - 優先度: Normal
  - 間隔: 動的（ブロックバッファに基づく）

- **BlockRenderingWorker**
  - 優先度: Highest
  - 間隔: フレームレートに基づく

#### 4.3.2 スレッド同期
- **ロック戦略**
  - ReadSyncRoot: パケット読み取り
  - DecodeSyncRoot: フレームデコード
  - ConvertSyncRoot: ブロック変換

- **アトミック操作**
  - AtomicBoolean: 状態フラグ
  - AtomicDateTime: タイムスタンプ

### 4.4 オーディオ出力

#### 4.4.1 サポートされるバックエンド
- **DirectSound**
  - デフォルトオーディオバックエンド
  - 低レイテンシ

- **Windows MME (Legacy)**
  - レガシーシステム対応
  - 互換性重視

#### 4.4.2 オーディオフォーマット
- **出力フォーマット**
  - サンプルフォーマット: 16-bit signed integer
  - チャンネル: 2 (ステレオ)
  - サンプルレート: 48000 Hz
  - インターリーブ形式

### 4.5 ビデオ出力

#### 4.5.1 ピクセルフォーマット
- **出力フォーマット**
  - BGRA (32-bit)
  - 8ビット/コンポーネント

#### 4.5.2 レンダリング
- **WPF統合**
  - BitmapSource経由でのレンダリング
  - マルチスレッドビデオオプション（実験的）
  - ハードウェアアクセラレーション対応

---

## 5. API仕様

### 5.1 MediaElementクラス

#### 5.1.1 主要プロパティ
```csharp
// 依存関係プロパティ
public Uri Source { get; set; }
public TimeSpan Position { get; set; }
public double Volume { get; set; }
public double Balance { get; set; }
public double SpeedRatio { get; set; }
public bool IsMuted { get; set; }
public bool ScrubbingEnabled { get; set; }
public bool VerticalSyncEnabled { get; set; }

// 読み取り専用プロパティ
public TimeSpan? NaturalDuration { get; }
public bool IsPlaying { get; }
public bool IsPaused { get; }
public bool IsOpen { get; }
public bool HasVideo { get; }
public bool HasAudio { get; }
public MediaPlaybackState MediaState { get; }
```

#### 5.1.2 主要メソッド
```csharp
// 非同期操作
public Task Open(Uri uri);
public Task Close();
public Task Play();
public Task Pause();
public Task Stop();
public Task<TimeSpan> Seek(TimeSpan position);

// 同期操作
public Task<Bitmap> CaptureBitmapAsync();
```

#### 5.1.3 イベント
```csharp
public event EventHandler MediaOpened;
public event EventHandler MediaClosed;
public event EventHandler MediaFailed;
public event EventHandler MediaEnded;
public event EventHandler PositionChanged;
public event EventHandler<MediaLogMessageEventArgs> FFmpegMessageLogged;
```

### 5.2 Libraryクラス

#### 5.2.1 静的プロパティ
```csharp
public static string FFmpegDirectory { get; set; }
public static string FFmpegVersionInfo { get; }
public static bool IsInitialized { get; }
public static int FFmpegLogLevel { get; set; }
```

#### 5.2.2 静的メソッド
```csharp
public static bool LoadFFmpeg();
public static Task<bool> LoadFFmpegAsync();
public static MediaInfo RetrieveMediaInfo(string mediaSource);
public static VideoSeekIndex CreateVideoSeekIndex(string mediaSource);
```

### 5.3 MediaOptionsクラス

#### 5.3.1 ストリーム選択
```csharp
public StreamInfo VideoStream { get; set; }
public StreamInfo AudioStream { get; set; }
public StreamInfo SubtitleStream { get; set; }
public bool IsVideoDisabled { get; set; }
public bool IsAudioDisabled { get; set; }
public bool IsSubtitleDisabled { get; set; }
```

#### 5.3.2 デコーダーオプション
```csharp
public Dictionary<int, string> DecoderCodec { get; }
public DecoderOptions DecoderParams { get; }
public List<HardwareDeviceInfo> VideoHardwareDevices { get; }
```

#### 5.3.3 バッファリングオプション
```csharp
public int VideoBlockCache { get; set; }
public int AudioBlockCache { get; set; }
public int SubtitleBlockCache { get; set; }
public double MinimumPlaybackBufferPercent { get; set; }
```

---

## 6. 使用例

### 6.1 基本的な使用方法

#### 6.1.1 初期化
```csharp
// App.xaml.cs
public App()
{
    // FFmpegバイナリのパスを設定
    Unosquare.FFME.Library.FFmpegDirectory = @"c:\ffmpeg";
}
```

#### 6.1.2 XAML定義
```xml
<Window xmlns:ffme="clr-namespace:Unosquare.FFME;assembly=ffme.win">
    <ffme:MediaElement 
        x:Name="Media" 
        Background="Gray" 
        LoadedBehavior="Play" 
        UnloadedBehavior="Manual" />
</Window>
```

#### 6.1.3 メディアのオープンと再生
```csharp
// ファイルを開く
await Media.Open(new Uri(@"c:\videos\sample.mp4"));

// URLを開く
await Media.Open(new Uri("https://example.com/video.mp4"));

// 再生制御
await Media.Play();
await Media.Pause();
await Media.Stop();

// シーク
await Media.Seek(TimeSpan.FromSeconds(30));

// クローズ
await Media.Close();
```

### 6.2 高度な使用例

#### 6.2.1 ストリーム選択
```csharp
var mediaInfo = await Media.Open(uri);

// オーディオストリームを選択
Media.MediaOptions.AudioStream = mediaInfo.Streams
    .FirstOrDefault(s => s.CodecType == AVMediaType.AVMEDIA_TYPE_AUDIO);

// コンポーネントを更新
await Media.ChangeMedia();
```

#### 6.2.2 スクリーンショット
```csharp
var bitmap = await Media.CaptureBitmapAsync();
bitmap.Save("screenshot.png", ImageFormat.Png);
```

#### 6.2.3 カスタムレンダリング
```csharp
Media.RenderingVideo += (s, e) =>
{
    // ビデオフレームデータにアクセス
    var block = e.Block as VideoBlock;
    // カスタム処理...
};
```

---

## 7. 制限事項と既知の問題

### 7.1 制限事項
- Windows専用（WPF依存）
- .NET 5.0以上が必要
- FFmpeg共有ライブラリが必要（配布に含まれない）
- 一部のコーデックはハードウェアアクセラレーションが必要

### 7.2 既知の問題
- ライブストリーム（m3u8等）での速度調整は未完成
- 一部のネットワークストリームでのシークが不安定
- マルチスレッドビデオレンダリングは実験的機能

### 7.3 パフォーマンス考慮事項
- 4K以上の解像度では高性能CPUが推奨
- ハードウェアデコードを有効にすることでCPU使用率を削減
- 大量のストリームを同時再生する場合はメモリ使用量に注意

---

## 8. 今後の開発計画

### 8.1 計画中の機能
- Floyd: 次世代バージョンの設計
- より良いライブストリーム対応
- クロスプラットフォーム対応の検討
- パフォーマンスの最適化

### 8.2 コミュニティへの貢献
- GitHubでのIssue報告
- プルリクエストの歓迎
- Gitterでのディスカッション

---

## 9. ライセンスと謝辞

### 9.1 ライセンス
- プロジェクトライセンス: LICENSEファイルを参照
- FFmpeg: LGPL/GPLライセンス

### 9.2 謝辞
- FFmpegチーム
- NAudioチーム
- FFmpeg.AutoGen (Ruslan Balanukhin)
- その他のコントリビューター

---

## 10. サポートとリソース

### 10.1 ドキュメント
- API Documentation: http://unosquare.github.io/ffmediaelement/api/
- GitHub Repository: https://github.com/unosquare/ffmediaelement
- Sample Application: Unosquare.FFME.Windows.Sample

### 10.2 コミュニティ
- Gitter Chat: https://gitter.im/ffmediaelement/Lobby
- GitHub Issues: バグ報告と機能リクエスト
- PayPal.Me: プロジェクトへの寄付

### 10.3 関連プロジェクト
- Meta Vlc
- Microsoft FFmpeg Interop
- WPF-MediaKit
- LibVLC.NET

---

**文書バージョン**: 1.0  
**最終更新日**: 2024年  
**対象FFMEバージョン**: 7.0.361.1以降
