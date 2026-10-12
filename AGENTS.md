# Project Instructions

## 役割分担とオーケストレーション方針

Claude CodeとCodexは利用量に制限があるため、実作業（コードの生成・修正そのものを細かく実行する作業）は基本的にOrnithへ細分化して依頼し、Claude CodeとCodexは「頭」として設計・指示・検証に専念すること。

- **Claude CodeとCodexを「頭」とする2層構成。** どちらか一方がその時々のオーケストレーション（作業の分解・Ornithへの指示・とりまとめ）を担当する。
- **オーケストレーション担当でない方の頭は、コード検証担当のsubagentとして動く。** 例: Claude Codeがオーケストレーションを担当するときはCodexがsubagentとしてコード検証（レビュー）を担当する。逆にCodexがオーケストレーションを担当するときはClaude Codeが検証担当のsubagentとなる。
- **複雑な編集や、Ornithで解決できない場合は、もう片方の頭に依頼する。** 対象は次のいずれか: 編集が複雑・リスクが高い、Ornithから反応が返らない、Ornithが同じ内容を繰り返すループに陥る、Ornithでは問題が解決しない。これらの場合、オーケストレーション担当はOrnithに再度投げるのではなく、もう片方の頭（Codexまたは Claude Code）にコード編集そのものを依頼すること。
- **Ornithは同時に2体まで起動して依頼してよい。**
- 各ツールの具体的な呼び出し方法は、本ファイル内の「Local coding worker」「Codexをサブエージェントとして利用する」「Codex側からClaude Codeをサブエージェントとして呼び出す場合」を参照。

## バージョン管理ルール

- コードにバージョン表記があることを確認する。
- コードを編集したら、バージョンを0.01加算する。
- バージョンは、コード先頭付近とトップ画面の片隅の両方に表記する。

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

## Ornithへのコード編集依頼のコツ（実験で判明・重要）

小さな編集タスク（タップ処理の重複発火防止ガードを3箇所に追加する）を複数パターンで試した結果、次のことが判明した。

- Ornithは「思考」機能を持ち、Ollama APIの`think`オプションで有効/無効を切り替えられる（未指定時は既定で有効）。
- **編集の設計判断（変数をどこに宣言するか等）をOrnith自身に委ねると失敗しやすい。** 思考を有効なまま設計を任せると、堂々巡りの検討に陥りnum_predictを使い切って何も出力されないことがあった。予算を増やして最後まで思考させても、今度は完走はするが**生成コード自体に欠陥がある**ことがあった（例: ガード用変数を関数の内側に宣言してしまい、毎回リセットされて機能しないコードになっていた）。完走＝正しいコードではない。
- **設計判断を一切含まない「穴埋め」形式（完成形のテンプレートを丸ごと提示し、差し替え箇所だけ指定する）にしたところ、`think:false`でも数秒で高速・完全・正確に成功した。**

**結論・運用ルール:**
- Ornithにコード編集そのものを依頼するときは、変数のスコープ・配置などの設計は先にオーケストレーション担当（Claude CodeまたはCodex）側で済ませ、Ornithには「このテンプレート通りに機械的に書き換えて」という穴埋め形式でのみ依頼すること。
- 設計判断が必要な編集を丸ごとOrnithに委ねないこと。オーケストレーション担当が先に設計し、それでも複雑すぎる場合はOrnithを経由せずもう片方の頭に依頼すること（既存のエスカレーション方針の通り）。
- Ornithが完走して出力した編集コードであっても、必ず実際に読んで検証すること（完走したこと自体は正しさの保証にならない）。

## コスト削減の観点からの仕事の割り振り判断（重要）

Ornithを使う目的はClaude Code/Codexの利用枠の節約であり、「Ornithに投げれば節約になる」は自動的には成立しない。委任するかどうかは以下で判断すること。

- **Ornithに投げてよい場合:** (1)助言・分析タスク(コードレビュー・バグ探し・実装案の提案・調査)。Ornith自身の思考結果がそのまま成果物になり、頭側がその思考をやり直す必要がないため確実に節約になる。(2)機械的な反復作業で、かつ反復量が十分に多く(目安: 同型の繰り返しが5〜10件以上、または生成量が多い定型処理)、「テンプレート＋差し替えリスト」の指示文が、頭側が全部を直接書き下すより明確に短くなる場合。
- **Ornithに投げず、頭側が直接行う場合:** 編集が小規模(数箇所程度)なとき。テンプレートを設計して依頼文を書く手間が、直接書いて適用する手間と同等かそれ以上になり、節約にならない(実例: タップ処理の3箇所修正は、依頼文の設計コストが直接編集とほぼ同等で、委任の節約効果は乏しかった)。設計判断を伴う編集も、判断は先に頭側で済ませ、残りが機械的かつ十分な分量のときだけOrnithに渡す。

## Ornithの育成方針（実績を積み上げて任せられる範囲を広げる）

Ornithは重みを再学習できないため、「育てる」とは実質的に、実証済みの依頼パターンを積み上げて委任の精度と範囲を上げていくことを指す。

- **成功パターンの蓄積:** 「穴埋め形式・設計判断ゼロ・think:false」のように実証済みの依頼テンプレートを本ファイルに追記し続け、次回以降はゼロから試行錯誤せず再利用する。
- **失敗パターンの蓄積:** 「クロージャのスコープ判断を含む依頼は失敗しやすい」のように、投げてはいけないタスクの形も同様に記録する。
- **信頼範囲の段階的拡大:** 実証済みパターンが増えるほど、Ornithに任せてよい作業の種類・割合を増やしてよい。ただし新しい種類のタスクを任せる際は、まず小さく試してから範囲を広げること。

## Codexをサブエージェントとして利用する

Codexは`codex`という単体コマンドとしてPATHには入っていないが、VSCode拡張機能「openai.chatgpt」に実行ファイルが同梱されており、フルパスを直接叩けば動作する（認証・設定は`~/.codex/`に既存）。

```bash
CODEX_BIN=$(ls -d ~/.vscode/extensions/openai.chatgpt-*-darwin-arm64/bin/macos-aarch64/codex 2>/dev/null | sort -V | tail -1)
"$CODEX_BIN" exec --sandbox read-only "<prompt>"          # 非対話的に質問・レビューさせる
"$CODEX_BIN" exec --sandbox read-only review              # 現在のリポジトリに対してコードレビューさせる
```

- 拡張機能はバージョンアップで頻繁にディレクトリ名（バージョン番号部分）が変わるため、固定パスを書かずに上記のように`ls -d ... | sort -V | tail -1`で最新版を動的に検出すること。
- デフォルトの`--sandbox`は`workspace-write`のためファイル変更が起こり得る。単なる質問・レビュー目的なら`--sandbox read-only`を明示し、意図しない変更を防ぐこと。
- Ornithと同様、Codexの出力もadvisory（参考意見）として扱い、結論は適用前に検証すること。

## Codex側からClaude Codeをサブエージェントとして呼び出す場合

同様に、Claude Code本体もVSCode拡張機能「anthropic.claude-code」に実行ファイルが同梱されている。

```bash
CLAUDE_BIN=$(ls -d ~/.vscode/extensions/anthropic.claude-code-*-darwin-arm64/resources/native-binary/claude 2>/dev/null | sort -V | tail -1)
"$CLAUDE_BIN" -p "<prompt>"     # -p/--print で非対話的に実行
```

- こちらも拡張機能のバージョンアップでパスが変わるため、`sort -V | tail -1`で最新版を動的に検出すること。
- 上記コマンドで見つからない場合は、Codex自身が同じ手順（`~/.vscode/extensions/`配下を`claude`という名前で探索）でパスを探すこと。
