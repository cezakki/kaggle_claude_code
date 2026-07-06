# Kaggle Workspace

Kaggle表形式データコンペを Claude Code に解かせるための作業リポジトリ。

## 構成

```
kaggle/
├── CLAUDE.md            ← Claudeへの共通指示書（運用ルール全般）
├── .gitignore
├── README.md
├── {コンペ名}/
│   ├── data/            ← 手動でダウンロードして配置（gitignore対象）
│   ├── models/          ← 学習済みモデル（gitignore対象）
│   ├── submissions/     ← 提出ファイル・履歴
│   └── experiments.csv  ← 実験ログ
└── ...
```

## 運用方針

- コンペごとにフォルダを分けて管理する
- 運用ルール（モデル・CV・アンサンブル・提出フローなど）の詳細は `CLAUDE.md` を参照
- データ取得・Kaggleへの提出は手動運用（Kaggle API/kaggle.jsonは使用しない）
- モデル学習・特徴量エンジニアリング・アンサンブルは Claude Code に委任する
- **Kaggleへの実際の提出（アップロード）は必ず人間が手動で行う**

## 使い方

```bash
cd {コンペ名}/
claude
> 続きをやって
```

初回はKaggleページの情報（データ説明・評価指標など）を渡してから開始する。
