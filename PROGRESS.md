# FactEye SageMaker YOLO - 進捗メモ

**最終更新**: 2026-02-27

---

## 全体の流れ

| # | ノートブック | 内容 | 状態 |
|---|-------------|------|------|
| 1 | `01_data_preparation.ipynb` | S3画像収集 & Ground Truthジョブ作成 | **作業中** |
| 2 | `02_annotation_to_yolo.ipynb` | アノテーション結果 → YOLO形式変換 | 未着手 |
| 3 | `03_training.ipynb` | YOLOv8-pose トレーニング | 未着手 |
| 4 | `04_deploy_serverless.ipynb` | サーバーレス推論エンドポイント | 未着手 |
| 5 | `05_inference_test.ipynb` | 推論テスト | 未着手 |

---

## 01_data_preparation.ipynb の進捗

### 完了済み

- [x] S3画像一覧取得（hygrometer-webcam, hygrometer-scam）→ 575枚
- [x] Ground Truth マニフェストファイル作成・S3アップロード
- [x] カスタムUIテンプレート（crowd-keypoint）作成・S3アップロード
- [x] カスタムLambda関数デプロイ（Pre-annotation / ACS）
- [x] IAM権限修正: `AmazonSageMakerAdminIAMExecutionRole` に `lambda:InvokeFunction` 追加済み

### 未完了（明日ここから）

- [ ] **ワークチームの確認 or 作成**
- [ ] ジョブ再作成・起動
- [ ] アノテーション作業の実施

---

## 明日やること（手順）

### Step 1: ワークチームを確認する

ノートブックのセル11、または以下を実行して既存のワークチームを確認:

```python
teams = sagemaker_client.list_workteams()
for team in teams['Workteams']:
    print(f"Name: {team['WorkteamName']}")
    print(f"ARN:  {team['WorkteamArn']}")
```

**ワークチームが表示されない場合:**
1. SageMakerコンソール → Ground Truth → Labeling workforces → **Private** タブ
2. ワークチームを新規作成（例: `meter-reading-team`）
3. メンバー（メールアドレス）を追加

### Step 2: セル16の `WORKTEAM_ARN` を修正

```python
# ↓ Step 1 で確認した正しいARNに書き換える
WORKTEAM_ARN = 'arn:aws:sagemaker:ap-northeast-1:886557786576:workteam/private-crowd/＜正しいチーム名＞'
```

### Step 3: ノートブックを再実行

カーネルが生きている場合: **セル16 → 17 → 18** を順に実行（セル18のコメントアウト解除を忘れずに）

カーネルを再起動した場合: **セル2（セットアップ）から全セル再実行**

### Step 4: ジョブがInProgressになったらアノテーション開始

ラベリングポータルにアクセスしてアノテーション作業を実施。

---

## 過去の失敗ジョブと原因

| ジョブ名 | 原因 | 対処 |
|----------|------|------|
| `facteye-meter-keypoint-20260227-081902` | Lambda AccessDeniedException（IAM権限不足） | IAMコンソールで `lambda:InvokeFunction` 権限を追加 → **解決済み** |
| `facteye-meter-keypoint-20260227-083500` | ワークチーム `meter-reading_team` が存在しない | 正しいワークチーム名の確認が必要 → **未解決** |
| `facteye-meter-keypoint-20260227-085800` | 同上（ワークチーム名未修正のまま再実行） | 同上 → **未解決** |

---

## 環境情報

- **リージョン**: ap-northeast-1（東京）
- **画像バケット**: `facteye-images-20251114`
- **SageMakerバケット**: デフォルトバケット（`sess.default_bucket()`）
- **実行ロール**: `AmazonSageMakerAdminIAMExecutionRole`
- **Pre Lambda**: `facteye-gt-pre-keypoint`（デプロイ済み）
- **ACS Lambda**: `facteye-gt-acs-keypoint`（デプロイ済み）
