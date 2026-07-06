# Kaggle 表形式コンペ 共通ルール

## コンペ概要（コンペごとに記入）
- URL:
- タスク: 分類 / 回帰 / ランキング
- 評価指標:
- 提出期限:
- 1日の提出回数上限:

---

## 作業ルール
- データ・提出は手動運用（Kaggle API/kaggle.jsonは使わない）
  - データ取得：ユーザーがブラウザから手動でダウンロードし `./data/` に配置する
  - 提出：Claudeが最終submission.csvを作成し、ユーザーが手動でKaggleサイトからアップロードする
- モデルは `./models/` に保存する
- 提出ファイルは `./submissions/YYYYMMDD_スコア.csv` の形式で保存する
- 実験ごとに通し番号をつける（exp001, exp002...）
- 各実験のCVスコア・使用特徴量・パラメータを `experiments.csv` に記録する
- 乱数シードは全て42に固定し、再現性を保つ
- ユーザーへの確認は不要、作業の区切りごとに自律的にこの CLAUDE.md を更新する
- 更新内容：現在のベストスコア・試したこと・次にやること

---

## 作業手順（優先順位順）

1. データ取得（ユーザーが手動でダウンロードし `./data/` に配置。準備できたら教えてもらう）
2. EDA（shape・欠損値・分布・ターゲット比率・train/testの分布差を確認）
3. Adversarial Validation（train/testの分布差をチェック。AUC 0.8以上なら要注意）
4. 特徴量エンジニアリング ← 最重要。ここに一番時間を使う
5. 共通Fold分割を作成（全モデルで使い回す）
6. ベースラインモデル作成（LightGBM）
7. Optunaでハイパーパラメータチューニング
8. 複数モデルで学習・OOF予測作成
9. アンサンブル
10. 最終提出（submission.csv作成まではClaude。Kaggleへのアップロードはユーザーが手動で実行）

---

## データリーク注意点
- Target Encodingは学習Fold内でのみ計算し、検証Foldには適用しない
- 統計量ベースの特徴量（平均・中央値・標準偏差など）も学習Fold内でのみ計算する
- 正規化・スケーリングのfitは学習Foldのみで行う
- 時系列データの場合、未来の情報を使った特徴量（ラグの方向）に注意する

---

## 特徴量エンジニアリング方針
- カテゴリ変数のエンコーディングを試す（Target Encoding, Count Encoding, One-Hot等）
- **Target Encodingは学習Fold内でのみ計算し、検証Foldには適用しない（リーク防止）**
- 数値変数の交互作用・比率特徴量を作る
- 欠損値パターン自体を特徴量にする（欠損フラグ）
- feature importanceを確認し、寄与の低い特徴量は削除を検討する
- 特徴量を追加・変更したら必ずCVスコアの変化を報告する

---

## 使用モデル
- LightGBM（メイン）
- XGBoost（アンサンブル用）
- CatBoost（カテゴリ変数が多い場合。優先度高）
- YDF RandomForest（多様性確保。`tensorflow_decision_forests`ではなく`ydf`を使うこと。Windows対応のため）
- Ridge / Lasso（線形ベースライン）

---

## 評価方法（CV）
- 分類タスク → StratifiedKFold（5分割）
- 回帰タスク → KFold（5分割）
- グループ性のあるデータ（同一ユーザー等の重複行）がある場合 → GroupKFold
- 時系列データの場合 → TimeSeriesSplit
- 乱数シード42で固定し、Fold分割indicesを全モデル・全実験で共通化する
- 各Foldごとのスコアと平均・標準偏差を必ず報告する
- Public LBスコアとCVスコアの乖離が大きい場合は要注意として報告する

```python
from sklearn.model_selection import StratifiedKFold
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
folds = list(skf.split(X, y))  # 全モデルで使い回す
```

---

## ハイパーパラメータチューニング
- Optunaを使用（ベイズ最適化）
- 対象：LightGBM, XGBoost, CatBoost（優先度高）
  - Ridge/Lassoはalpha程度の簡易探索でOK
  - YDFは基本デフォルトパラメータで可
- 共通Fold分割を使ってチューニングする（アンサンブルとの整合性のため）
- n_trials=100を目安、timeout=3600（1時間）で打ち切り
- チューニング前後のCVスコアを比較・報告する
- 最適パラメータは `best_params.json` に保存し、次回以降再利用する

---

## アンサンブル方針
- 各モデルは必ずOOF（Out-of-Fold）予測を保存する
- 手順：
  1. 各モデルの単純平均でCVスコアを確認
  2. CVスコアに基づく重み付き平均を試す（重みはOOF予測で最適化）
  3. 余力があればスタッキング（メタモデル：Ridge）に挑戦
- 最終的にCVスコアが最も良かったアンサンブル方法を採用する
- アンサンブル前後のCVスコアを必ず比較・報告する
- 最終submissionは全データで再学習した各モデルの予測をアンサンブルして作成する

---

## LB乖離管理（Shake-up対策）
- CVスコアとPublic LBスコアの差を毎回記録する
- 乖離が大きい場合、Fold戦略を見直す
- 最終提出はKaggleの提出選択枠を使い、「CVが一番良いもの」と「LBが一番良いもの」を分けて選ぶ

---

## 並列作業の方針
- 以下のモデルは互いに独立して学習できるため、サブエージェントで並列実行してよい
  - LightGBM, XGBoost, CatBoost, YDF RandomForest, Ridge, Lasso
- ファイルの競合を避けるため、各モデルは専用のフォルダ（models/lgbm/, models/xgb/ 等）に出力先を分ける
- 並列実行はトークン消費が増える点に留意し、重い処理（チューニング等）で特に効果が大きい
- Ridge/Lassoは学習が軽いため、他の重いモデルと同時に並列実行してまとめて片付けてよい

## 提出前チェック
- submission.csv の列名・行数・dtypeが sample_submission.csv と一致するか確認する
- 欠損値がないか確認する
- IDの順序がtestデータと一致しているか確認する

## セキュリティ
- Kaggle APIは使用しない（kaggle.json不要）
- .gitignore に data/, models/ を追加する（データファイル自体の誤コミット防止）

## バージョン管理
- 意味のある変更（新特徴量・新モデル追加時など）ごとにgit commitする
- コミットメッセージにCVスコアを含める（例: "add target encoding, CV=0.823"）

## セッション管理
- 大きな区切り（EDA完了、ベースライン完了など）でユーザーに簡潔に報告する
- コンテキストが埋まってきたら /compact を使って要約する

## 提出ルール
- **提出（Kaggleサイトへのアップロード）はユーザーが手動で行う。Claudeは実行しない**
- Claudeはsubmission.csvを作成し、以下を提示してユーザーに確認を取る：CVスコア、変更点（前回からの差分）、ファイル名
- 1日の提出回数上限を超えないよう管理する
- 提出候補は「CVスコアが有意に改善した時のみ」提示する
- 全提出の履歴（日付・CVスコア・LBスコア・変更点）を `submissions/history.md` に記録する

---

## リソース管理
- 1回の学習・チューニングが30分を超える場合は一度中断し、途中経過を報告してから続行するか確認する
- GPU使用可能ならXGBoost/CatBoostはGPUモードを使う（`tree_method="gpu_hist"`等）
- 時間が足りない場合、優先度は「特徴量エンジニアリング＞アンサンブル＞チューニング＞YDF等の追加モデル」の順で削る

## 環境
- Windows環境。Random Forest系は `tensorflow_decision_forests` ではなく `ydf` を使う
- 主要ライブラリ: lightgbm, xgboost, catboost, ydf, scikit-learn, optuna, pandas

## エラー対応
- エラーが出たら自分で原因を調査し修正を試みる
- 3回試しても解決しない場合は、状況を整理してユーザーに報告する

---

## 現在のベストスコア（自動更新）
- CV:
- LB:
- 使用モデル・アンサンブル手法:

## 試したこと（自動更新）
- [ ] ベースライン作成

## 次にやること（自動更新）
-
