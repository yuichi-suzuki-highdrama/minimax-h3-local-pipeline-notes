# MiniMax-H3 ローカルパイプライン覚え書き（RTX 5090）

[English](README.md) | 日本語

**ComfyUI + MiniMax-H3（Ref2VA）** の動画パイプラインを、**RTX 5090 1 枚**の環境で回してきた個人的な実験ノートです。
書いてある時間はすべて 1 台のマシンで測った壁時計時間なので、絶対的なベンチマークではなく、設定どうしの相対比較として読んでいただければと思います。

**プライバシーについて:** ローカルのユーザー名、絶対パス、キャラクターの参照素材、案件名は意図的に省いています。

**本文で使う略語**

| 略語 | 意味 |
|---|---|
| Ref2VA / R2V | MiniMax-H3 の参照→動画（＋音声）モード。参照画像・動画とテキストから動画を生成します |
| PDD Acc | alibaba-pai が公開している Parallel-Decoding-Distillation の高速化 LoRA（8 step） |
| TE | テキストエンコーダ（Qwen3-VL）。プロンプトと参照を conditioning に変換します |
| cond | ComfyUI の CONDITIONING（TE の出力）。「768-cond」は 768p でエンコードした conditioning のことです |
| uponly | latent の拡大だけを行う工程（H3 latent upscaler モデル）。この工程ではサンプリングしません |
| R2V4 | lightx2v の Ref2V turbo 4-step LoRA（`minimax_h3_ref2v_turbo_4step_v0.1`）。refine 用の蒸留として使っています |
| LX8 | lightx2v の Ref2V turbo 8-step LoRA（`minimax_h3_ref2v_turbo_8step_v1.0_768p`） |
| VSR | NVIDIA RTX Video Super Resolution（nvidia-vfx SDK） |

## 環境（以下の数値はすべてこの環境で計測しました）

| 項目 | 値 |
|---|---|
| GPU | NVIDIA RTX 5090、VRAM 32 GB（Blackwell） |
| ホスト | Windows 11、RAM 128 GB |
| ComfyUI | 0.34.x（master、2026 年 9 月上旬時点） |
| PyTorch | 2.11 + CUDA 13.0 |
| アテンション | comfy-kitchen 0.2.33（`--use-ck-attention`） |
| Comfy Compiler | **無効**（`--disable-comfy-compiler`、2026-09-07 から） |
| H3 の重み | ref2va pruned int8 convrot UNET、Qwen3-VL-32B int8 convrot テキストエンコーダ |

## 構成

- ComfyUI + MiniMax-H3 Ref2VA（pruned int8 convrot の UNET で問題なく動きます）
- アテンションは **comfy-kitchen**（`--use-ck-attention`）。密な kitchen 経路を本線として固定しています
- 2026-09-05 にコアへ入った Comfy Compiler は `--disable-comfy-compiler` で**無効**にしています。有効のままだと、H3 では数本連続で回したあとに約 3 倍遅くなり（UNET が部分オフロードに落ちる）、中断直後に `aimdo memory compile error` で ComfyUI が落ちました。無効にしても速度差はありませんでした（480p・362 フレーム・同 seed で、有効 164 秒 / 無効 163 秒）
- alibaba-pai の公式 **PDD Acc 8-step LoRA**（[モデルカード](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs)）を、コミュニティ製ノード [ComfyUI-MiniMax-H3-PDD-Acc](https://github.com/Jalen-Brunson/ComfyUI-MiniMax-H3-PDD-Acc) 経由で読み込み、**480p の生成にだけ**使っています
- 工程間の持続化として、nested latent と conditioning の保存・読み込みを行います（後述の「工程間の持続化」を参照してください。方法とリンクだけを書いており、**このリポジトリにノードのコードは含めていません**）
- latent 拡大（nested H3 latent）→ キャッシュした conditioning で refine → RTX Video Super Resolution（VSR）の順に流します
- オーケストレーションは、API 形式のグラフを ComfyUI のローカル HTTP API（`/prompt`）に POST し、`/history` で完了と所要時間を取る小さな Python スクリプトで行っています

## 現在の本線（品質と時間のバランス）

1. **480p 生成** — PDD Acc **nfe=8**。参照画像は**面ごとに分割**し、**短辺が約 1024 を超える場合だけ縮小**してから `ref_image_size=max` にします（同一性を優先）。髪や顔の甘さを許容できるなら `match` の方が速く済みます
2. **768-cond を一度だけ bake**（TE）。refine をやり直すときに使い回せます（約 35〜50 秒）
3. **latent を uponly で 1344×768 へ**。768p は Ref2VA の学習解像度で、幅と高さがどちらも 32 の倍数です。1280×720 は高さが 32 で割り切れないため使えません。720p 相当が必要なら **1280×704** にしてください
4. **refine** — キャッシュした cond に対して **R2V turbo 4-step（R2V4）**、denoise **約 0.75**、**kitchen の密アテンション**（BlockSparse / SLA は使いません）、euler、CFG 1.0
5. **VSR** — `HIGHBITRATE_ULTRA` の **2 倍**（2688×1536）。3 倍や 4K は線画がきつく、ジャギーが目立ちやすい印象でした
6. **VSR の Denoise / Deblur** — 線画では**オフ**にしています（A/B で差が判定できず、必要性も感じませんでした）

品質を優先したい場合は、R2V4 の代わりに **LX8** で refine する選択肢もあります（8 秒・768 のクリップで約 +100 秒）。

## 計測スナップショット（8.0 秒 = 192 フレーム @ 24fps）

同一マシン、Ref2VA、5090 1 枚での計測です。"match" は参照を生成キャンバスに合わせて縮小する設定、"max" は大きな短辺（〜2048 まで）をそのまま許容する設定を指します。参照のトークン数は全ステップに効いてきます。

| 設定 | 合計壁時間 | 備考 |
|---|---:|---|
| 約 12 秒 @ 704、free_gpu あり | 約 649 秒 | A_gen 約 231 秒 |
| 約 12 秒 @ 704、free_gpu なし | 約 635 秒 | free を省いても 14 秒ほどしか縮まりませんでした |
| 8 秒 @ 704、参照すべて max | 約 446 秒 | A 約 181 秒 |
| 8 秒 @ 768、480=match / bake=max、R2V4、VSR 3 倍 | 約 370 秒 | A 約 81 秒（生成は最速ですが、同一性と髪が甘くなります） |
| 8 秒 @ 768、LX8、match/match、VSR 2 倍 | 約 471 秒 | LX8 の refine だけで約 290 秒 |
| 8 秒 @ 768、参照を分割＋短辺 1024 超のみ縮小＋`max`、R2V4、VSR 2 倍 | **約 353 秒** | **A 約 131 秒 — 現在の本線です**（match と原寸 max の中間） |

工程ごとの例（8 秒 / 768 / 参照は短辺キャップ＋`max` / R2V4 kitchen / ULTRA 2 倍）:

`A_gen 130.6s · B_bake 35.7s · C_uponly 10.1s · D_r2v4 140.2s · E_vsr 36.8s → 合計 353.4s`

確定した kitchen 本線（同じレシピ、別のクリップ、壁時間は **約 5.5 分 / 約 330 秒**）:

`A 約 110s · B 約 26s · C 約 15s · D_r2v4_kitchen 約 135s · E_vsr 約 44s`

上の表には denoise を書いていません。denoise ごとの refine 単体の時間は「refine 蒸留の比較」の表にまとめてあります（d0.5 で約 140 秒、d0.75 で約 150〜160 秒）。D 工程は、選んだ denoise によってその帯のどこかに収まります。

### 実際に効いたもの

- **480 生成の `ref_image_size=match`** — 単独では最大の短縮でした（8 秒で生成 約 180 秒 → 約 80 秒級）
- **分割＋短辺 1024 のキャップ（縮小のみ）＋`max`** — 同一性と速度の落としどころです（原寸の max や純粋な match より扱いやすい）
- **bake は一度だけ**行い、refine の A/B で使い回します
- **毎工程の `free_gpu`** — 差は 10〜15 秒程度です。VRAM に余裕があれば気にしなくてよいと思います
- 幅の微調整（1216 と 1344 など） — DiT と refine が支配的なうちは、ほとんど効きませんでした

### 参照画像は `max`、ただし前処理をしてから

同一性（髪のシルエットや顔）のために **`ref_image_size=max`** を使いますが、巨大な生のシートをそのまま Comfy に渡すことはしていません。

実際の手順は次のとおりです。

1. 多面図のキャラクターシートは、面ごとに**別ファイルへ分割**します
2. 面の**短辺が約 1024 より大きければ**、短辺 ≈1024 へ**縮小**します。すでに小さいものは**拡大しません**（引き伸ばしても細部を捏造するだけです）
3. その上で、生成時と 768-cond の bake 時に **`ref_image_size=max`** を指定します

理由は単純で、巨大な参照を原寸 `max` で渡すと最も遅くなり（トークンが全ステップに乗る）、純粋な `match` は最速ですが髪と同一性が甘くなるからです。**分割＋短辺 1024 キャップ＋`max`** が、実際に使っている中間点です。

## refine 蒸留の比較（同じ 8 秒 / 1344×768 uponly ＋ bake 済み 768-cond）

| refine | 壁時間（refine のみ） | 品質メモ |
|---|---:|---|
| **R2V4** d0.5 | **約 140 秒** | 以前の基準 |
| **R2V4** d0.75 | **約 150〜160 秒** | 採用しているルック（0.5 より締まり、1.0 ほど描き換えません） |
| **R2V4 + BlockSparseAttention** top-k 10%（同 d0.75） | **約 100〜120 秒** | 速いのですが、後のクリップでルックが後退したため本線では**不採用** |
| **LX8** d0.5〜1.0 | **約 250〜270 秒** | 重いので、任意の品質アップとして残しています |
| **PDD Acc nfe=4**（PDD Scheduler） | **約 60〜130 秒** | 粒が出て弱いため、**768 の refine では不採用** |
| **PDD Acc nfe=8** d0.25 | **約 490 秒** | コミュニティ PDD ノードパック同梱の拡大 refine 例。遅く、R2V4 より良くもありませんでした |

### PDD Acc を refine に使って分かったこと

- **`MiniMaxH3PDDAccApply` と `MiniMaxH3PDDAccScheduler`**（学習済みの sigma グリッド）を組で使ってください。素の `BasicScheduler` で PDD を駆動すると、グリッド外の評価になって強いノイズが出ます
- 同じ Acc-8Step ファイルで、コミュニティ製ローダーは **`nfe=4|6|8`** に対応しています（4 は学習済みブロックを再グループ化するもの。正確な sigma グリッドはパックの README を参照してください）
- コミュニティ PDD ノードパックのワークフローにある latent 拡大 refine の例は **nfe=8 + denoise 0.25**（学習済みの最後の 2 ブロックに相当）です。nfe=4 の denoise 0.5 は「粗い 2 ステップ」で 1 ステップが厚く、当方の実行では R2V4 より**明らかにノイジー**でした
- turbo / 蒸留 LoRA（R2V4 / LX8）を PDD Acc の上に**重ねないでください**
- **Ref2VA Acc** は **ref2va** の UNET と組み合わせます（トランクの指紋チェックが入っています）

## VSR メモ（アニメ・線画向け）

- このルックでは **HIGHBITRATE_ULTRA 2 倍**を、3 倍や 4K より優先しています
- ULTRA の前段に Denoise / Deblur を入れる A/B は**不採用**にしました（判定が難しく、必要性もありませんでした）

## 解像度早見表

| 目的 | 使う値 |
|---|---|
| 約 480p 生成 | 864×480 |
| 約 720p の latent | **1280×704**（1280×720 は使えません） |
| 素の 768p | **1344×768** |
| 納品 | ULTRA 2 倍 → 2688×1536 |

計画のときに意識しておきたいグリッドが 2 つあります。

- **幅と高さはどちらも 32 の倍数**である必要があります（そのため 720 ではなく 704）。グリッド外の値は patchify の段階で失敗します
- **フレーム数は `5 + 17k` のグリッド**に乗ります（24 fps）: 124、141、158、175、**192（= 8.0 秒）**、209、…、294、…、362（≈15 秒）。現在の ComfyUI core はグリッド外の値を拒否するのではなく、**次のグリッド値へ切り上げます**（`comfy_extras/nodes_minimax_h3.py` の `align_frame_count`）。グリッド値を明示しておかないと、クリップが予定より長くなります

## 効いてくるレシピ定数

- サンプラー: **euler**
- ガイダンス: **CFG 1.0**（蒸留系のため）
- 生成時の SigmaShift: **12.0 / 3.0**（turbo の refine LoRA は、それぞれ固有の shift を使います）
- Comfy の HTTP ポーリングは、実行中に UI を数十秒止めない書き方にしておくと快適です

## 工程間の持続化（方法とリンクのみ。ノードのコードは含めていません）

H3 の工程は、最終の mp4 だけ残しても**きれいには**つながりません。ジョブの間で **nested latent** と **bake 済み conditioning** を保存しておくと、refine や A/B のたびに TE や 480 生成をやり直さずに済みます。

このリポジトリでは**カスタムノードを同梱していません**（ライセンスと保守のリスクを避けるためです）。ローカルで実装するか、サードパーティのパックをご自身で導入してください。

### なぜ標準の `SaveLatent` では足りないのか

H3 の latent は **`NestedTensor`**（映像と音声のメンバーを持つ）です。Comfy の H3 拡張と nested tensor のヘルパーは次にあります。

- [comfy_extras/nodes_minimax_h3.py](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy_extras/nodes_minimax_h3.py)
- [comfy/nested_tensor.py](https://github.com/comfyanonymous/ComfyUI/blob/master/comfy/nested_tensor.py)

標準の latent 保存は通常のテンソルを前提にしているため、nested の AV latent では失敗します。サンプラーはすでに `comfy.utils.pack_latents` / `unpack_latents` で **pack / unpack** しているので、ディスク I/O でも同じ手順を踏めば大丈夫です。

### nested latent の保存・読み込み（手順）

1. 生成 / uponly / refine の後に `LATENT["samples"]` を取り出します
2. nested であれば、`unbind()` → `pack_latents(members)` として、pack 済みテンソル 1 本とメンバーごとの形状（int64）を書き出します
3. **safetensors**（または同等の形式）で、専用の出力サブフォルダに保存します（当方は `output/h3_latents/`）
4. 読み込み時は、pack 済みテンソルと形状を読み、`unpack_latents` → `NestedTensor` を再構築 → `{"samples": …}` に包みます

nested latent を工程間の**正本**として扱ってください（mp4 のデコード→再エンコードは等価ではありません）。ファイル名に工程と解像度を入れておくと、オーケストレータから探しやすくなります。

Comfy の core に nested latent の Save / Load が正式に入ったら、そちらに切り替えてローカルのヘルパーは捨てるつもりです。

### bake 済み conditioning（768-cond）の手順

1. TE を **768** のエンコードサイズで一度だけ実行します（プロンプト・参照・長さは refine と同じ）。DiT のサンプリングは行いません
2. Comfy の **CONDITIONING** ツリー（テンソルと付随情報）をディスクへ保存します（例: CPU に移したツリーを `output/h3_conds/` に `torch.save`）
3. refine 時にそのファイルを読み込み、TE を飛ばします

cond は **プロンプト＋参照＋エンコード解像度＋フレーム長**と一致している必要があります。どれかを変えたら bake し直してください。一度 bake すれば、同じ uponly latent に対する denoise や蒸留の A/B で使い回せます。

注意点として、conditioning ファイルの読み戻しには `torch.load(..., weights_only=False)`（pickle）が必要です。**ご自身で bake したファイルだけ**を読み込むようにしてください。他人の `.pt` を読み込むのは避けてください。

### VRAM / TE のアンロード

任意のノード **`H3FreeTextEncoder`**（cond を作った後、DiT の前に TE を解放する）は、サードパーティのパックに含まれています。

- [ComfyUI-H3-Multishot](https://github.com/jlucasmcrell/ComfyUI-H3-Multishot)（`h3_advanced.py`）

導入前にそのリポジトリのライセンスをご確認ください。オーケストレータから使う場合は、工程の間に `{"unload_models": true, "free_memory": true}` を `POST /free` するだけでも、このノードなしでほぼ同じ効果が得られます。

### 推奨する工程の入出力

1. **480 生成** → nested latent を保存（必要なら mp4 プレビューも）
2. **768-cond の bake** → conditioning を保存（TE のみ）
3. **latent 拡大** → 480 の nested を読み込み → uponly → 1344×768 の nested を保存
4. **refine** → nested と conditioning を読み込み → R2V4 → nested と mp4 を保存
5. **VSR** → refine の mp4 に外部アップスケーラをかける

1〜4 をファイルとして残しておかないと、やり直しのたびに TE や 480 生成のコストを払い直すことになります。

## Sparse Attention — 試しましたが**本線にはしていません**

Comfy の **`BlockSparseAttention`**（top-k 約 10%）を、R2V4 の refine と、生成＋refine の全体で A/B しました。**速く**はなりました（refine で約 25%）が、少なくとも 1 キャラクターで**ルックが崩れました**（ノイズ・乱れ）。本線は **kitchen の密アテンション**のままにしています。意図的に実験する場合を除き、core の sparse フックとノードは**入れないこと**をおすすめします。

## まだ開いていること

- サニタイズしたオーケストレーションスクリプトの公開（パスをパラメータ化し、個人素材を含めない形で）
- 上流の sparse 経路が、将来ルックを壊さない水準になったら再検討したい

## ライセンスについて

ベースモデルと Acc LoRA は、それぞれ上流のライセンスに従います。alibaba-pai の `MiniMax-H3-Acc-LoRAs` は 2026-08-27 にライセンス欄を Apache-2.0 から **MiniMax-H3 Community License Agreement** に変更しているので、古いミラーではなく[現在のモデルカード](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs)と[コミット履歴](https://huggingface.co/alibaba-pai/MiniMax-H3-Acc-LoRAs/commits/main)を確認してください。ご自身で導入するサードパーティのパック（Multishot、PDD Acc ローダーなど）は**それぞれの**ライセンスに従います。このリポジトリはドキュメントのみです。商用利用の可否は、各自の状況で確認をお願いします。

---

*数値は、RTX 5090 を積んだワークステーション 1 台で 2026 年 9 月に計測したものです。*
