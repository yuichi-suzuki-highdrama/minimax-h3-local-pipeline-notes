# MiniMax-H3 ローカルパイプライン覚え書き（RTX 5090）

[English](README.md) | 日本語

**ComfyUI + MiniMax-H3（Ref2VA）** の動画パイプラインを **RTX 5090 1 枚**で回した個人的な実験ノートです。
時間はすべて 1 台のマシンでの壁時計時間です。絶対値のベンチマークではなく、相対比較として読んでください。

**プライバシー:** ローカルのユーザー名、絶対パス、キャラクターの参照素材、案件名は意図的に省いています。

**本文で使う略語**

| 略語 | 意味 |
|---|---|
| Ref2VA / R2V | MiniMax-H3 の参照→動画（＋音声）モード。参照画像・動画＋テキストから動画を生成 |
| PDD Acc | alibaba-pai の Parallel-Decoding-Distillation 高速化 LoRA（8 step） |
| TE | テキストエンコーダ（Qwen3-VL）。プロンプトと参照を conditioning に変換する |
| cond | ComfyUI の CONDITIONING（TE の出力）。「768-cond」は 768p でエンコードした conditioning |
| uponly | latent の拡大だけを行う工程（H3 latent upscaler モデル）。この工程ではサンプリングしない |
| R2V4 | lightx2v の Ref2V turbo 4-step LoRA（`minimax_h3_ref2v_turbo_4step_v0.1`）。refine 用の蒸留 |
| LX8 | lightx2v の Ref2V turbo 8-step LoRA（`minimax_h3_ref2v_turbo_8step_v1.0_768p`） |
| VSR | NVIDIA RTX Video Super Resolution（nvidia-vfx SDK） |

## 環境（以下の数値はすべてここで計測）

| 項目 | 値 |
|---|---|
| GPU | NVIDIA RTX 5090、VRAM 32 GB（Blackwell） |
| ホスト | Windows 11、RAM 128 GB |
| ComfyUI | 0.34.x（master、2026 年 9 月上旬） |
| PyTorch | 2.11 + CUDA 13.0 |
| アテンション | comfy-kitchen 0.2.33（`--use-ck-attention`） |
| H3 の重み | ref2va pruned int8 convrot UNET、Qwen3-VL-32B int8 convrot テキストエンコーダ |

## 構成

- ComfyUI + MiniMax-H3 Ref2VA（pruned int8 convrot の UNET で動く）
- アテンション: **comfy-kitchen**（`--use-ck-attention`）。密な kitchen 経路を本線として固定
- alibaba-pai の公式 **PDD Acc 8-step LoRA**（[モデルカード](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs)）を、コミュニティ製ノード [ComfyUI-MiniMax-H3-PDD-Acc](https://github.com/Jalen-Brunson/ComfyUI-MiniMax-H3-PDD-Acc) 経由で読み込み、**480p 生成にだけ**使う
- 工程間の持続化: nested latent と conditioning の保存・読み込み（後述の「工程間の持続化」参照。方法とリンクのみで、**このリポジトリにノードのコードは含めない**）
- latent 拡大（nested H3 latent）→ キャッシュした conditioning で refine → RTX Video Super Resolution（VSR）
- オーケストレーション: API 形式のグラフを ComfyUI のローカル HTTP API（`/prompt`）に POST し、`/history` で完了と時間を取る小さな Python スクリプト

## 現在の本線（品質と時間のバランス）

1. **480p 生成** — PDD Acc **nfe=8**。参照は**分割済み**、**短辺が約 1024 を超える場合だけ縮小**し、`ref_image_size=max`（同一性優先）。髪や顔の甘さを許容できるなら `match` の方が速い
2. **768-cond を一度だけ bake**（TE）。refine のやり直しで使い回せる（約 35〜50 秒）
3. **latent を uponly で 1344×768 へ**（768p は Ref2VA の学習解像度で、幅高さとも 32 の倍数。1280×720 は高さが 32 で割れないので不可。720p 相当が要るなら **1280×704**）
4. **refine** — キャッシュした cond で **R2V turbo 4-step（R2V4）**、denoise **約 0.75**、**kitchen の密アテンション**（BlockSparse / SLA なし）、euler、CFG 1.0
5. **VSR** — `HIGHBITRATE_ULTRA` **2 倍**（2688×1536）。3 倍や 4K は線画がきつく・ジャギーになりやすい
6. **VSR の Denoise / Deblur** — 線画では**オフ**（A/B で差が判定できず、必要もなかった）

任意の品質アップ: R2V4 の代わりに **LX8** で refine（8 秒・768 のクリップで約 +100 秒）。

## 計測スナップショット（8.0 秒 = 192 フレーム @ 24fps）

同一マシン、Ref2VA、5090 1 枚。"match" = 参照を生成キャンバスに合わせて縮小、"max" = 大きな短辺（〜2048 まで）を許容。参照トークンは全ステップで効く。

| 設定 | 合計壁時間 | 備考 |
|---|---:|---|
| 約 12 秒 @ 704、free_gpu あり | 約 649 秒 | A_gen 約 231 秒 |
| 約 12 秒 @ 704、free_gpu なし | 約 635 秒 | free を省いても 14 秒しか縮まない |
| 8 秒 @ 704、参照すべて max | 約 446 秒 | A 約 181 秒 |
| 8 秒 @ 768、480=match / bake=max、R2V4、VSR 3 倍 | 約 370 秒 | A 約 81 秒（生成は最速だが同一性・髪が甘い） |
| 8 秒 @ 768、LX8、match/match、VSR 2 倍 | 約 471 秒 | LX8 の refine だけで約 290 秒 |
| 8 秒 @ 768、参照を分割＋短辺 1024 超のみ縮小＋`max`、R2V4、VSR 2 倍 | **約 353 秒** | **A 約 131 秒 — 現在の本線**（match と原寸 max の中間） |

工程ごとの例（8 秒 / 768 / 参照短辺キャップ＋`max` / R2V4 kitchen / ULTRA 2 倍）:

`A_gen 130.6s · B_bake 35.7s · C_uponly 10.1s · D_r2v4 140.2s · E_vsr 36.8s → 合計 353.4s`

確定した kitchen 本線（同じレシピ、別クリップ、壁時間 **約 5.5 分 / 約 330 秒**）:

`A 約 110s · B 約 26s · C 約 15s · D_r2v4_kitchen 約 135s · E_vsr 約 44s`

上の行には denoise を示していません。denoise 別の refine 単体時間は「refine 蒸留の比較」の表にあります（d0.5 ≈ 140 秒、d0.75 ≈ 150〜160 秒）。D 工程は選んだ denoise によってその帯に収まります。

### 実際に効いたもの

- **480 生成の `ref_image_size=match`** — 単独で最大の短縮（8 秒で生成 約 180 秒 → 約 80 秒級）
- **分割＋短辺 1024 のキャップ（縮小のみ）＋`max`** — 同一性と速度の落としどころ（原寸 max や純粋な match より良い）
- **bake は一度だけ**、refine の A/B で使い回す
- **毎工程の `free_gpu`** — 10〜15 秒しか変わらない。VRAM に余裕があれば気にしなくてよい
- 幅の微調整（1216 と 1344 など） — DiT と refine が支配的なうちはほぼ効かない

### 参照画像は `max`、ただし前処理してから

同一性（髪のシルエット、顔）のために **`ref_image_size=max`** を使いますが、巨大な生シートをそのまま Comfy に放り込むことはしません。

実務手順:

1. 多面図のキャラクターシートは、面ごとに**別ファイルへ分割**する
2. 面の**短辺が約 1024 より大きければ**短辺 ≈1024 へ**縮小**。すでに小さいものは**拡大しない**（引き伸ばしは細部を捏造するだけ）
3. その上で生成時（と 768-cond の bake 時）に **`ref_image_size=max`**

理由: 巨大参照の原寸 `max` は最も遅い（トークンが全ステップに乗る）。純粋な `match` は最速だが髪と同一性が甘い。**分割＋短辺 1024 キャップ＋`max`** が実際に使っている中間点。

## refine 蒸留の比較（同じ 8 秒 / 1344×768 uponly ＋ bake 済み 768-cond）

| refine | 壁時間（refine のみ） | 品質メモ |
|---|---:|---|
| **R2V4** d0.5 | **約 140 秒** | 以前の基準 |
| **R2V4** d0.75 | **約 150〜160 秒** | 採用ルック（0.5 より締まり、1.0 ほど描き換えない） |
| **R2V4 + BlockSparseAttention** top-k 10%（同 d0.75） | **約 100〜120 秒** | 速いが本線では**不採用**（後のクリップでルックが後退） |
| **LX8** d0.5〜1.0 | **約 250〜270 秒** | 重い。任意の品質アップとして残す |
| **PDD Acc nfe=4**（PDD Scheduler） | **約 60〜130 秒** | 粒が出る・弱い。**768 refine では不採用** |
| **PDD Acc nfe=8** d0.25 | **約 490 秒** | コミュニティ PDD ノードパック同梱の拡大 refine 例。遅く、R2V4 より良くもない |

### PDD Acc を refine に使う際の教訓

- **`MiniMaxH3PDDAccApply` + `MiniMaxH3PDDAccScheduler`**（学習済みの sigma グリッド）を使う。素の `BasicScheduler` で PDD を駆動**しない**。グリッド外の評価は強いノイズになる
- 同じ Acc-8Step ファイルでコミュニティ製ローダーは **`nfe=4|6|8`** に対応（4 は学習済みブロックを再グループ化。正確な sigma グリッドはパックの README を参照）
- コミュニティ PDD ノードパックのワークフローにある latent 拡大 refine 例: **nfe=8 + denoise 0.25**（= 学習済みの最後の 2 ブロック）。nfe=4 の denoise 0.5 は「粗い 2 ステップ」で 1 ステップが厚く、当方の実行では R2V4 より**明らかにノイジー**
- turbo / 蒸留 LoRA（R2V4 / LX8）を PDD Acc に**重ねない**
- **Ref2VA Acc** は **ref2va** の UNET と組み合わせる（トランクの指紋チェックあり）

## VSR メモ（アニメ・線画向け）

- このルックでは **HIGHBITRATE_ULTRA 2 倍**を 3 倍 / 4K より優先
- ULTRA の前段の Denoise / Deblur の A/B: **不採用**（判定困難、不要）

## 解像度早見表

| 目的 | 使う値 |
|---|---|
| 約 480p 生成 | 864×480 |
| 約 720p の latent | **1280×704**（1280×720 は不可） |
| 素の 768p | **1344×768** |
| 納品 | ULTRA 2 倍 → 2688×1536 |

計画時に守る 2 つのグリッド:

- **幅と高さはどちらも 32 の倍数**（だから 720 ではなく 704）。グリッド外は patchify で落ちる
- **フレーム数は `5 + 17k` のグリッド**（24 fps）: 124、141、158、175、**192（= 8.0 秒）**、209、…、294、…、362（≈15 秒）。現在の ComfyUI core はグリッド外の値を拒否せず、**次のグリッド値へ切り上げる**（`comfy_extras/nodes_minimax_h3.py` の `align_frame_count`）。グリッド値を明示しないと、クリップが予定より長くなる

## 効くレシピ定数

- サンプラー: **euler**
- ガイダンス: **CFG 1.0**（蒸留系）
- 生成時の SigmaShift: **12.0 / 3.0**（turbo の refine LoRA は各自の shift を使う）
- Comfy の HTTP ポーリングは、実行中に UI を数十秒止めない書き方にする

## 工程間の持続化（方法とリンクのみ。ノードのコードは含めない）

H3 の工程は、最終 mp4 だけ残しても**きれいには**つながりません。ジョブ間で **nested latent** と **bake 済み conditioning** を保存し、refine や A/B のたびに TE や 480 生成をやり直さずに済むようにします。

このリポジトリは**カスタムノードを同梱しません**（ライセンスと保守のリスク）。ローカルで実装するか、サードパーティのパックを自分で導入してください。

### なぜ標準の `SaveLatent` では足りないか

H3 の latent は **`NestedTensor`**（映像と音声のメンバー）です。Comfy の H3 拡張と nested tensor のヘルパーを参照:

- [comfy_extras/nodes_minimax_h3.py](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy_extras/nodes_minimax_h3.py)
- [comfy/nested_tensor.py](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy/nested_tensor.py)

標準の latent 保存は通常のテンソルを前提にしているため、nested の AV latent では失敗します。サンプラーはすでに `comfy.utils.pack_latents` / `unpack_latents` で**pack / unpack** しているので、ディスク I/O でも同じことをします。

### nested latent の保存・読み込み（手順）

1. 生成 / uponly / refine の後に `LATENT["samples"]` を取る
2. nested なら: `unbind()` → `pack_latents(members)` → pack 済みテンソル 1 本とメンバーごとの形状（int64）を書き出す
3. **safetensors**（または同等）で専用の出力サブフォルダに保存（当方は `output/h3_latents/`）
4. 読み込み時: pack 済みテンソル＋形状を読む → `unpack_latents` → `NestedTensor` を再構築 → `{"samples": …}` に包む

nested latent を工程間の**正本**として扱う（mp4 のデコード→再エンコードは等価ではない）。ファイル名に工程と解像度を入れ、オーケストレータが見つけられるようにする。

Comfy の core に nested latent の Save / Load が正式に入ったら、そちらを使ってローカルのヘルパーは捨てる。

### bake 済み conditioning（768-cond）の手順

1. TE を **768** のエンコードサイズで一度だけ実行（プロンプト・参照・長さは refine と同じ）。DiT のサンプリングはしない
2. Comfy の **CONDITIONING** ツリー（テンソル＋付随情報）をディスクへ保存（例: CPU に移したツリーを `output/h3_conds/` に `torch.save`）
3. refine 時: そのファイルを読み込み、TE を飛ばす

cond は **プロンプト＋参照＋エンコード解像度＋フレーム長**と一致していなければならない。どれか変えたら bake し直す。一度 bake したら、同じ uponly latent に対する denoise / 蒸留の A/B で使い回す。

注意: conditioning ファイルの読み戻しには `torch.load(..., weights_only=False)`（pickle）が必要。**自分で bake したファイルだけ**を読むこと。他人の `.pt` は決して読み込まない。

### VRAM / TE のアンロード

任意のノード **`H3FreeTextEncoder`**（cond を作った後、DiT の前に TE を解放）はサードパーティのパックにあります:

- [ComfyUI-H3-Multishot](https://github.com/jlucasmcrell/ComfyUI-H3-Multishot)（`h3_advanced.py`）

導入前にそのリポジトリのライセンスを確認してください。オーケストレータからなら、工程の間に `{"unload_models": true, "free_memory": true}` を `POST /free` するだけで、このノードなしでもほぼ同じ効果が得られます。

### 推奨する工程の入出力

1. **480 生成** → nested latent を保存（＋任意で mp4 プレビュー）
2. **768-cond の bake** → conditioning を保存（TE のみ）
3. **latent 拡大** → 480 の nested を読み込み → uponly → 1344×768 の nested を保存
4. **refine** → nested と conditioning を読み込み → R2V4 → nested と mp4 を保存
5. **VSR** → refine の mp4 に外部アップスケーラ

1〜4 をファイルとして残さないと、やり直しのたびに TE や 480 生成を払い直すことになります。

## Sparse Attention — 試したが**本線にはしない**

Comfy の **`BlockSparseAttention`**（top-k 約 10%）を R2V4 の refine と、生成＋refine の全体で A/B しました。**速く**はなりました（refine で約 25%）が、少なくとも 1 キャラで**ルックが崩れました**（ノイズ・乱れ）。本線は **kitchen の密アテンション**のまま。意図的に実験する場合を除き、core の sparse フックとノードは**入れない**でください。

## 未解決

- サニタイズしたオーケストレーションスクリプトの公開（パスをパラメータ化し、個人素材なし）
- 上流の sparse 経路が、将来ルックを壊さない水準になったら再検討

## ライセンスの注意

ベースモデルと Acc LoRA は各上流のライセンスに従います。alibaba-pai の `MiniMax-H3-Acc-LoRAs` は 2026-08-27 にライセンス欄を Apache-2.0 から **MiniMax-H3 Community License Agreement** に変更しています。古いミラーではなく[現在のモデルカード](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs)と[コミット履歴](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs/commits/main)を確認してください。自分で導入するサードパーティのパック（Multishot、PDD Acc ローダーなど）は**それぞれの**ライセンスに従います。このリポジトリはドキュメントのみです。商用利用は各自の状況で確認してください。

---

*数値は RTX 5090 を積んだワークステーション 1 台で 2026 年 9 月に計測したものです。*
