# poc-SageMaker-YOLO

**YOLOv8 Pose Estimation × Amazon SageMaker によるアナログメーター自動読み取り PoC**

アナログメーター（湿度計・圧力計など）の画像から、YOLOv8 のキーポイント検出を用いて針の位置を特定し、メーターの値を自動算出するシステムの Proof of Concept です。

## アーキテクチャ

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  S3 (画像)    │────▶│  Ground Truth    │────▶│  YOLOv8n-pose    │
│              │     │  キーポイント     │     │  ファインチューニング │
│              │     │  アノテーション    │     │                  │
└──────────────┘     └─────────────────┘     └────────┬─────────┘
                                                      │ ONNX エクスポート
                                                      ▼
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  メーター値    │◀────│  角度計算        │◀────│  SageMaker       │
│  算出         │     │  キーポイント     │     │  Serverless      │
│              │     │  → 角度 → 値     │     │  Endpoint        │
└──────────────┘     └─────────────────┘     └──────────────────┘
```

## キーポイント定義

各メーター画像に対して 4 つのキーポイントを検出します。

| キーポイント | 説明 |
|---|---|
| `needle_tip` | 針の先端 |
| `needle_center` | 針の回転中心（軸） |
| `scale_min` | スケール最小値の位置 |
| `scale_max` | スケール最大値の位置 |

検出されたキーポイントから `needle_center` → `needle_tip` の角度を算出し、`scale_min` ～ `scale_max` の範囲内での割合に変換してメーター値を算出します。

## 対応メーター種別

| 種別 | スケール範囲 | 単位 |
|---|---|---|
| 湿度計 (hygrometer) | 0 - 100 | % |
| 圧力計 (pressure) | 0 - 1.0 | MPa |
| 温度計 (temperature) | -20 - 60 | ℃ |
| 電圧計 (voltmeter) | 0 - 300 | V |

## ノートブック構成

`sagemaker/notebooks/` 配下に 5 つのノートブックがあり、上から順に実行します。

### 01_data_preparation.ipynb — データ準備

- S3 バケット (`facteye-images-20251114`) からメーター画像を一覧取得
- DynamoDB `Meters` テーブルと突き合わせてメーター種別でフィルタリング
- Ground Truth 用の入力マニフェスト (JSONL) を生成
- `crowd-keypoint` を使ったカスタム UI テンプレートを作成
- プライベートワークフォースによるアノテーションジョブを起動

### 02_annotation_to_yolo.ipynb — アノテーション変換

- Ground Truth の出力マニフェストを読み込み
- キーポイントデータを YOLOv8-pose 形式に変換
- 品質チェック（ぼやけ画像・キーポイント欠損・判別不能マークの除外）
- train (70%) / val (20%) / test (10%) に分割
- 標準的な YOLO ディレクトリ構造で S3 にアップロード

### 03_training.ipynb — モデル学習

- YOLOv8n-pose（軽量 nano モデル）をベースにファインチューニング
- 主なハイパーパラメータ: epochs=100, batch=16, imgsz=640, patience=20
- データ拡張: 回転 (±15°), 色調変換, スケーリング（左右/上下反転は無効）
- 学習済みモデルを ONNX 形式にエクスポート
- `model.tar.gz` として S3 に保存

### 04_deploy_serverless.ipynb — サーバーレスエンドポイントデプロイ

- カスタム推論コード (`inference.py`) を作成
  - Letterbox 前処理（アスペクト比保持）
  - ONNX Runtime による推論
  - キーポイント座標の逆変換・JSON シリアライズ
- PyTorch 2.1 推論コンテナを使用
- SageMaker Serverless Inference でデプロイ（メモリ 4GB, 最大同時実行 5）
- アイドル時はゼロスケール（コスト最適化）

### 05_inference_test.ipynb — 推論テスト & 精度検証

- 複数画像でのバッチ推論テスト
- キーポイントから角度・メーター値を算出するロジックの検証
- DynamoDB `MeterReading` テーブルの Bedrock (Claude Vision) 読み取り結果との精度比較
- 検出率・レイテンシ・精度の分析レポートを生成

## 前提条件

### AWS リソース

- **S3 バケット**: `facteye-images-20251114`（メーター画像が格納済み）
- **DynamoDB テーブル**:
  - `Meters` — メーター設定情報（device_id, meter_type 等）
  - `MeterReading` — Bedrock による既存の読み取り結果
- **SageMaker**:
  - 実行ロール（S3, DynamoDB, SageMaker, Lambda へのアクセス権限）
  - プライベートワークフォース & ワークチーム（Ground Truth 用）
- **リージョン**: `us-east-1`

### 実行環境

- SageMaker ノートブックインスタンスまたは SageMaker Studio
- Python 3.10+
- 主要ライブラリ: `ultralytics`, `onnxruntime`, `boto3`, `sagemaker`, `Pillow`, `numpy`

## S3 パス構成

```
s3://<SageMaker Default Bucket>/
├── ground-truth/
│   ├── manifests/         # Ground Truth 入力マニフェスト
│   ├── output/            # Ground Truth 出力
│   └── ui-template/       # カスタム UI テンプレート
└── yolo-pose/
    ├── dataset/           # YOLOv8-pose 形式のデータセット
    ├── models/            # 学習済みモデル (model.tar.gz)
    ├── deploy/            # デプロイ用モデル (推論コード付き)
    ├── training-results/  # 学習ログ・学習曲線
    ├── test-reports/      # 推論テストレポート
    └── endpoint/          # エンドポイント情報
```

## エンドポイント呼び出し例

```python
import boto3

runtime = boto3.client('sagemaker-runtime')

with open('meter_image.jpg', 'rb') as f:
    image_bytes = f.read()

response = runtime.invoke_endpoint(
    EndpointName='facteye-meter-keypoint',
    ContentType='image/jpeg',
    Accept='application/json',
    Body=image_bytes
)

result = json.loads(response['Body'].read())
# result = {
#   "detections": [{
#     "bbox": {"cx": ..., "cy": ..., "width": ..., "height": ...},
#     "confidence": 0.95,
#     "keypoints": {
#       "needle_tip": {"x": ..., "y": ..., "confidence": ...},
#       "needle_center": {"x": ..., "y": ..., "confidence": ...},
#       "scale_min": {"x": ..., "y": ..., "confidence": ...},
#       "scale_max": {"x": ..., "y": ..., "confidence": ...}
#     }
#   }],
#   "image_size": {"width": 640, "height": 480}
# }
```

## 技術選定の背景

| 選定 | 理由 |
|---|---|
| **YOLOv8n-pose (nano)** | 軽量でサーバーレス環境の制約内で動作可能 |
| **ONNX エクスポート** | ハードウェア非依存、PyTorch より高速な推論 |
| **Serverless Inference** | アイドル時ゼロコスト、変動的なワークロードに最適 |
| **キーポイントベースの手法** | 値の直接回帰より汎用的で、メーター種別の追加が容易 |
| **Bedrock との比較検証** | 既存の Claude Vision ベースラインに対する精度評価 |
