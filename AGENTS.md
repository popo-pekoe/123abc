# Project Instructions

## Local coding worker

A local coding worker is available through:

```bash
ornith-worker "<prompt>"
```

It runs Ornith-1.5-9B Q4_K_M locally through Ollama.

Use Ornith when useful for:

- independent code review
- bug hunting
- implementation suggestions
- parallel investigation
- second opinions on non-trivial changes

Rules:

- Ornith does not automatically know the repository contents.
- Include all necessary source code, filenames, error messages, and context in the prompt.
- Treat Ornith's output as advisory.
- Verify its conclusions before applying changes.
- Prefer Claude or Codex for final architectural decisions.
- Do not invoke Ornith for trivial tasks where direct inspection is faster.

## Ornith呼び出しの実践知見（失敗から学んだ運用ルール）

2,400行超のJSファイル全体を一度にプロンプトへ渡したところ、Ornithの応答が同じ段落を50回以上繰り返すループに陥り、コンテキスト上限で強制終了し、依頼した4項目中1項目しか完了しなかった。この失敗を踏まえ、以降Ornithに依頼する際は次を守ること。

- **大きな入力をそのまま丸ごと渡さない。** ファイル全体ではなく、レビュー対象を絞る（例: 修正のレビューなら全文ではなく `git diff` のみ、新規コードのレビューなら本質的なロジック部分のみを渡し、純粋なデータ配列などはコメントで要約して省略する）。
- **出力形式をプロンプトで明示的に制約する。** 「最終回答は簡潔な箇条書きのみ」「同じ内容を繰り返さない」「各セクション最大5項目まで」を明記すると、暴走的な繰り返し出力を防げる。
- **Ollama API呼び出し時の推奨オプション:** `repeat_penalty: 1.3` 程度、`repeat_last_n: 256` 程度を指定して繰り返しループを抑制する。`num_ctx` は入力量に見合うだけ確保しつつ、`num_predict` にも上限を設け、万一ループしても暴走・長時間化しないようにする。
- **それでも大きい場合は分割して依頼する。** 「バグ・可読性・改善案・複雑さ」を一度に全部聞くのではなく、入力が大きい場合はセクションごとに分けて依頼する方が完走率が高い。
