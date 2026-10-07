# Dify RAG 原因切り分けキット v1.0

- ダウンロード（ZIP）: https://github.com/KoryoK/rag-trace-kit/releases/latest/download/rag-trace-kit-v1.0.zip
- このリポジトリの各ファイルは、ZIPを展開したものと同じ内容です。
- 解説記事（note）: https://note.com/brainy_phlox8948/n/n460a3b89ab14
- 前作：[Dify RAG 12問チェックキット](https://github.com/KoryoK/rag-12-question-kit)

Difyで作ったRAGの回答が期待と違ったとき、「検索で根拠を取れていないのか」「取れた根拠がLLMに渡っていないのか」「渡した後の回答がずれたのか」を、**同じ1回の実行の記録**で分けるための小さな検証セットです。架空の社内規程を使い、実在の規程や業務データは含みません。

## 同梱物

- `rag-trace-workflow.yml`：Difyにインポートできるワークフロー（Start → Knowledge Retrieval → IF/ELSE → LLM → End）。LLMノードは「contextあり」「contextなし」の2本。
- `knowledge.txt`：架空会社「つばめラボ」の規程 R01〜R04（5チャンク）。
- `questions.csv`：3問（資料内 A04／資料外 U04／条件不足 C01）と、実行前に決めた期待根拠・合否基準。
- `results-template.csv`：自分の記録用の空欄表（3問×3条件＝9行）。
- `results.csv`：2026年10月7日の実測9件（判定つき）。
- `runs/`：9件＋復元テスト1件の生ログ（検索結果・LLMに渡ったプロンプト全文・回答・トークン数）。ワークスペースの識別子は伏せています。
- `settings.json`：使った設定と、未確認の項目。

## 3段階の証跡をどこで見るか

Difyの Test Run を実行し、結果パネルの **TRACING** で各ノードを開きます。

1. **Knowledge Retrieval ノードの OUTPUT**：検索結果（content と score）。期待する規程がここにあるか。
2. **LLM ノードの PROCESS DATA → prompts**：LLMに実際に渡った system／user の全文。検索結果の本文がここに入っているか。
3. **LLM ノードの OUTPUT → text**：回答。

この3つを同じ実行から並べると、次のように分かれます。

- 1に根拠がない → 検索・ナレッジ側を見る（チャンクの有効化、分割、Top K、インデックス）
- 1にはあるが2にない → 受け渡し（LLMノードの context 接続、プロンプトの `{{#context#}}`）を見る
- 1にも2にもあるが3がずれる → プロンプト・指示・モデルの出力側を見る（原因はまだ一つに絞れない）

## 手順

1. Knowledge → Create で `knowledge.txt` を登録（General、区切り `\n\n`、最大1024、overlap 50、High Quality、Vector Search、Top K 3、Score Threshold OFF）。プレビューで5チャンクになることを確認。
2. Studio → Create → Import DSL file で `rag-trace-workflow.yml` を読み込む。「DSL version difference」の注意は出ますが、7ノードが復元されます。
3. Knowledge Retrieval ノードを開き、KNOWLEDGE の「+」から1で作ったナレッジだけを選ぶ（同名のナレッジが複数あるときは余分を外す）。
4. Test Run で `question` と `mode` を入れて実行。`mode=base` が基準、`mode=nocontext` が「検索はするがLLMに渡さない」条件。
5. 検索欠落を作るときは、ナレッジのドキュメント画面で期待根拠のチャンクだけを一時的に Disabled にする。終わったら戻す。
6. TRACING から上の3段階を `results-template.csv` に写し、`evidence_present` / `context_transferred` / `answer_supported` / `citation_correct` / `next_check` を埋める。

## 実測で分かったこと（2026-10-07、gpt-4o-mini、Dify Cloud サンドボックス）

- 受け渡し欠落（mode=nocontext）と検索欠落（チャンク無効化）は、**回答文だけ見るとどちらも「資料に記載がないため確認できません」で区別できない**。TRACING の1と2を見ると分かれる。
- C01「宿泊費の上限はいくらですか？」は、検索もR02を取り、プロンプトにも「海外出張の宿泊費はこの資料の対象外」が入っていたのに、回答は国内の金額だけだった。検索でも受け渡しでもない、出力側で条件を落とした例。
- U04（資料外）は根拠チャンクを外しても回答は変わらない。「記載なし」が正解の質問では、回答だけでは検索の状態を確認できない。

## 限界

- 3問×3条件、各1回。精度・再現率・時短などの一般的な数値ではありません。
- 検索欠落は意図的にチャンクを無効化して作ったもので、自然発生ではありません。
- 出力側のずれ（C01）は1件の観測で、モデル単独の原因とは断定していません。
- Dify のバージョン、実効パラメータ（temperature=0 以外）、別モデル・別プランでの挙動は未確認。
- 実行主体は Claude Code（ブラウザー操作）で、本人の顧客案件の記録ではありません。

## 利用条件

架空規程・質問・記録様式・ワークフローは、自分の検証用にコピー・改変して利用できます。実測ログを変更して再配布する場合は、実行条件と変更箇所が分かるようにしてください。第三者のデータや記事の転載は含みません。
