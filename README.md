# student-coach-comment-ai

## 概要

本プロジェクトは、**学習ログをもとに「分析 → 可視化 → 講師コメント生成」までを一気通貫で行う分析基盤のサンプル**です。


zenn記事

https://zenn.dev/hdotm/articles/e8fded799fe86b

https://zenn.dev/hdotm/articles/71b814719b3371

https://zenn.dev/hdotm/articles/83528c1b2eabe7

---

## アーキテクチャ

```
CSV（学習ログ）
  ↓ dbt seed
stg_study_log（staging）
  ↓
fct_student_daily（日次集計）
  ↓
mart_teacher_comment_request（LLM用マート）
  ↓
CSV出力 / OpenAI API
  ↓
LLM生成コメント（comments.csv）
```

---

## 設計のポイント

### dbt による責務分離

* **staging**

  * 型変換
  * 欠損値処理
  * カラム正規化
* **mart（fact）**

  * 生徒 × 日付単位の日次集計
* **LLM用マート**

  * LLM に渡すための要約テキスト生成


---

### LLM 用マートについて

LLM に直接入力できるよう、
**1行＝1生徒・1日単位の自然文テキスト**を dbt モデルとして生成しています。

例：

```
生徒ID s001 は 2026-01-11 に 1 回の学習を行い、
合計学習時間は 10.0 分でした。視聴学習は 0 回です。
```

この設計により、

* プロンプト生成ロジックを SQL 側に集約
* Python 側の前処理を最小化
* LLMモデル変更時も SQL 側の修正で対応可能

といったメリットがあります。

---

### データ品質管理

`schema.yml` を用いて、以下のデータテストを定義しています。

* **not null**：必須カラムの欠損防止
* **unique**：イベントIDの一意性保証

`dbt test` を実行することで、
LLM に渡す前段階のデータ品質を継続的に検証できる構成としています。

---

### intermediate モデルを省略した理由

本プロジェクトは小規模な検証用途のため、
**staging → mart の2層構成**としています。

複数 mart でのロジック再利用や集計の複雑化が進んだ場合には、
intermediate 層を追加する前提の設計です。

---

## LLMによる講師コメント生成（No15）

`mart_teacher_comment_request` を入力として、
OpenAI API（`gpt-4o-mini`）を利用し、**講師コメントを自動生成**しています。

### 実行方法

```bash
# PowerShell
$env:OPENAI_API_KEY="sk-..."
python generate_comments.py
```

### 出力内容

* `student_id`
* `event_date`
* `summary_text`（LLM入力）
* `comment`（LLM生成の講師コメント）

生成結果は CSV と DuckDB の両方に保存されます。

---

## 成果物

* DuckDB 内の分析用テーブル
* LLM 入力用 CSV（`outputs/mart_teacher_comment_request.csv`）
* LLM 生成コメント（`outputs/comments.csv`）
* dbt docs によるモデル・テストの可視化
* Looker Studio による簡易ダッシュボード

---

## 使用技術

* Python
* dbt
* DuckDB
* OpenAI API
* Looker Studio

---

## 補足（コストについて）

OpenAI API は従量課金ですが、
本プロジェクト規模（少量データ）では **数円〜数十円程度**で実行可能です。

