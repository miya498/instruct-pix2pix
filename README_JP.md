# InstructPix2Pix: 画像編集指示に従う学習モデル
### [プロジェクトページ](https://www.timothybrooks.com/instruct-pix2pix/) | [論文](https://arxiv.org/abs/2211.09800) | [データ](http://instruct-pix2pix.eecs.berkeley.edu/)
InstructPix2PixのPyTorch実装です。これは指示ベースの画像編集モデルで、元の[CompVis/stable_diffusion](https://github.com/CompVis/stable-diffusion)リポジトリをベースにしています。

[InstructPix2Pix: Learning to Follow Image Editing Instructions](https://www.timothybrooks.com/instruct-pix2pix/)  
 [Tim Brooks](https://www.timothybrooks.com/)\*,
 [Aleksander Holynski](https://holynski.org/)\*,
 [Alexei A. Efros](https://people.eecs.berkeley.edu/~efros/) <br>
 UC Berkeley <br>
  \*同等の貢献を示す

  <img src='https://instruct-pix2pix.timothybrooks.com/teaser.jpg'/>

## クイックスタート

以下の手順に従って、InstructPix2Pixをダウンロードし、自分の画像で実行できます。これらの手順は18GB以上のVRAMを持つGPUでテストされています。GPUをお持ちでない場合は、デフォルトの設定を変更するか、[モデルの他の使用方法](https://github.com/timothybrooks/instruct-pix2pix#other-ways-of-using-instructpix2pix)をご確認ください。

### conda環境のセットアップと事前学習済みモデルのダウンロード:
```
conda env create -f environment.yaml
conda activate ip2p
bash scripts/download_checkpoints.sh
```

### 単一画像の編集:
```
python edit_cli.py --input imgs/example.jpg --output imgs/output.jpg --edit "turn him into a cyborg"

# オプションで、結果を調整するためのパラメータを指定できます:
# python edit_cli.py --steps 100 --resolution 512 --seed 1371 --cfg-text 7.5 --cfg-image 1.2 --input imgs/example.jpg --output imgs/output.jpg --edit "turn him into a cyborg"
```

### または、独自のインタラクティブ編集Gradioアプリを起動:
```
python edit_app.py 
```
![Edit app](https://github.com/timothybrooks/instruct-pix2pix/blob/main/imgs/edit_app.jpg?raw=true)

_(パラメータの調整方法については、[ヒント](https://github.com/timothybrooks/instruct-pix2pix#tips)セクションをご覧ください)_

## セットアップ

すべての依存関係をインストール:
```
conda env create -f environment.yaml
```

事前学習済みモデルをダウンロード:
```
bash scripts/download_checkpoints.sh
```

## 生成されたデータセット

私たちの画像編集モデルは、454,445個の例を含む生成されたデータセットでトレーニングされています。各例は以下の要素で構成されています：(1)入力画像、(2)編集指示、(3)編集後の出力画像。データセットは2つのバージョンを提供しています：各編集画像ペアを100回生成し、CLIPメトリクス（論文のセクション3.1.2）に基づいて最良の例を選択したもの（`clip-filtered-dataset`）と、ランダムに選択した例（`random-sample-dataset`）です。

リリース版のデータセットでは、NSFWコンテンツのプロンプトと画像をフィルタリングしています。NSFWフィルタリング後、GPT-3生成データセットには451,990個の例が含まれています。最終的な画像ペアデータセットは以下の通りです：

|  | 画像編集例の数 | データセットサイズ |
|--|-----------------------|----------------------- |
| `random-sample-dataset` |451990|727GB|
|  `clip-filtered-dataset` |313010|436GB|

これらのデータセットのいずれかをダウンロードするには、適切なデータセット名を指定して以下のコマンドを実行します：

```
bash scripts/download_data.sh clip-filtered-dataset
```

## InstructPix2Pixのトレーニング

InstructPix2Pixは、初期のStableDiffusionチェックポイントからファインチューニングすることでトレーニングされます。最初のステップはStable Diffusionチェックポイントをダウンロードすることです。私たちのトレーニング済みモデルでは、v1.5チェックポイントを開始点として使用しました。同じものをダウンロードするには、以下のスクリプトを実行します：

```
bash scripts/download_pretrained_sd.sh
```

異なるチェックポイントを使用したい場合は、`configs/train.yaml`ファイルの8行目、`ckpt_path:`の後に指定してください。

次に、設定をダウンロードした（または生成した）データセットを指すように変更する必要があります。上記の`clip-filtered-dataset`を使用する場合は、このステップをスキップできます。それ以外の場合は、設定の85行目と94行目（`data.params.train.params.path`、`data.params.validation.params.path`）を編集する必要があるかもしれません。

最後に、以下のコマンドでトレーニングジョブを開始します：

```
python main.py --name default --base configs/train.yaml --train --gpus 0,1,2,3,4,5,6,7
```

## 独自のデータセットの作成

ペアになった画像と編集指示の生成データセットは2段階で作成されます：まず、GPT-3を使用してテキストのトリプルを生成します：(a)画像を説明するキャプション、(b)編集指示、(c)編集後の画像を説明するキャプション。次に、Stable DiffusionとPrompt-to-Promptを使用して、キャプションのペア（編集前/後）を画像のペアに変換します。

### (1) キャプションと指示のデータセットの生成

生成されたキャプションと編集指示のデータセットは[こちら](https://instruct-pix2pix.eecs.berkeley.edu/gpt-generated-prompts.jsonl)で提供しています。私たちのキャプション+指示を使用する予定の場合は、ステップ(2)に進んでください。それ以外に独自のテキストデータセットを作成したい場合は、以下のステップ(1.1-1.3)に従ってください。GPT-3を使用して非常に大きなデータセットを生成するのは高額になる可能性があることに注意してください。

#### (1.1) 手動で指示とキャプションのデータセットを作成

プロセスの最初のステップはGPT-3のファインチューニングです。これを行うために、私たちは700個の例を含むデータセットを作成しました。これらは、モデルが実行できる編集の幅広い範囲をカバーしています。私たちの例は[こちら](https://instruct-pix2pix.eecs.berkeley.edu/human-written-prompts.jsonl)で利用可能です。これらは多様で、可能なキャプションと編集タイプの幅広い範囲をカバーしているべきです。理想的には、キャプションと指示の重複や大きな重複を避けるべきです。また、Stable DiffusionとPrompt-to-Promptの制限（カメラの移動、ズームイン、オブジェクトの位置の入れ替えなどの大きな空間変換ができないなど）を考慮して例を作成することが重要です。

入力プロンプトは、より大きなデータセットを生成するために使用された入力プロンプトの分布に密接に一致する必要があります。私たちは700個の入力プロンプトを_LAION Improved Aesthetics 6.5+_データセットからサンプリングし、例の生成にもこのデータセットを使用しました。このデータセットはかなりノイズが多いことがわかりました（多くのキャプションが長すぎ、無関係なテキストを含んでいます）。このため、MSCOCOとLAION-COCOデータセットも検討しましたが、コンテンツ、固有名詞、アートメディアの多様性を考慮して、最終的に_LAION Improved Aesthetics 6.5+_を選択しました。GPT-3で例を生成する際に別のデータセットやデータセットの組み合わせを入力として使用する場合は、トレーニング例を手動で作成する際に同じ分布から入力プロンプトをサンプリングすることをお勧めします。

#### (1.2) GPT-3のファインチューニング

次のステップは、手動で作成した指示/出力で大規模言語モデルをファインチューニングし、新しい入力キャプションから編集指示と編集後のキャプションを生成することです。これには、OpenAI APIを介してGPT-3のDavinciモデルをファインチューニングしますが、他の言語モデルも使用できます。

GPT-3のトレーニングデータを準備するには、まずOpenAI開発者アカウントを作成して必要なAPIにアクセスし、[ローカルデバイスにAPIキーを設定](https://beta.openai.com/docs/api-reference/introduction)する必要があります。また、`prompts/prepare_for_gpt.py`スクリプトを実行して、指示とキャプションを連結し、区切り文字と停止シーケンスを追加することで、プロンプトを正しい形式に変換します。

```bash
python dataset_creation/prepare_for_gpt.py --input-path data/human-written-prompts.jsonl --output-path data/human-written-prompts-for-gpt.jsonl
```

次に、OpenAI CLIを介してGPT-3をファインチューニングします。以下に例を示しますが、ベストプラクティスは変更される可能性があるため、OpenAIの公式ドキュメントを参照してください。Davinciモデルを1エポックでトレーニングしました。より小さく、より安価なGPT-3バリアントやオープンソースの言語モデルで実験することもできますが、これによりパフォーマンスが低下する可能性があります。

```bash
openai api fine_tunes.create -t data/human-written-prompts-for-gpt.jsonl -m davinci --n_epochs 1 --suffix "instruct-pix2pix"
```

提供されているGradioアプリを起動して、ファインチューニングされたGPT-3モデルをテストできます：

```bash
python prompt_app.py --openai-api-key OPENAI_KEY --openai-model OPENAI_MODEL_NAME
```

![Prompt app](https://github.com/timothybrooks/instruct-pix2pix/blob/main/imgs/prompt_app.jpg?raw=true)

#### (1.3) キャプションと指示の大規模データセットの生成

ファインチューニングされたGPT-3モデルを使用して、大規模なデータセットを生成します。私たちのデータセットは数千ドルのコストがかかりました。これらの例を生成するスクリプトについては`prompts/gen_instructions_and_captions.py`を参照してください。まず、`--num-samples`に低い値を設定して少数の例を生成し、徐々にスケールを増やして、スケールを増やす前に結果が期待通りに機能していることを確認することをお勧めします。

```bash
python dataset_creation/generate_txt_dataset.py --openai-api-key OPENAI_KEY --openai-model OPENAI_MODEL_NAME
```

非常に大きなスケール（例：100K+）で生成する場合、複数のプロセスを並列で実行することで、データセットの生成が大幅に高速化されます。これは`--partitions=N`をより高い数に設定し、各`--partition`を対応する値に設定して複数のプロセスを実行することで実現できます。

```bash
python dataset_creation/generate_txt_dataset.py --openai-api-key OPENAI_KEY --openai-model OPENAI_MODEL_NAME --partitions=10 --partition=0
```

### (2) ペアになったキャプションをペアになった画像に変換

次のステップは、テキストキャプションのペアを画像のペアに変換することです。これには、事前学習済みのStable Diffusionチェックポイントを`stable_diffusion/models/ldm/stable-diffusion-v1/`にコピーする必要があります。上記の手順に従ってトレーニングを行った場合は既に完了しているかもしれませんが、そうでない場合は以下のコマンドを実行してください：

```bash
bash scripts/download_pretrained_sd.sh
```

私たちのモデルでは、[チェックポイントv1.5](https://huggingface.co/runwayml/stable-diffusion-v1-5/blob/main/v1-5-pruned.ckpt)と[新しいオートエンコーダー](https://huggingface.co/stabilityai/sd-vae-ft-mse-original/resolve/main/vae-ft-mse-840000-ema-pruned.ckpt)を使用しましたが、他のモデルも機能する可能性があります。他のモデルを使用する場合は、`--ckpt`と`--vae-ckpt`引数を渡して、対応するチェックポイントを指定してください。すべてのチェックポイントがダウンロードされたら、以下のコマンドでデータセットを生成できます：

```
python dataset_creation/generate_img_dataset.py --out_dir data/instruct-pix2pix-dataset-000 --prompts_file path/to/generated_prompts.jsonl
```

このコマンドは単一のGPU（通常はV100またはA100）で動作します。多くのGPU/マシンで並列化するには、`--n-partitions`を並列ジョブの総数に設定し、`--partition`を各ジョブのインデックスに設定します。

```
python dataset_creation/generate_img_dataset.py --out_dir data/instruct-pix2pix-dataset-000 --prompts_file path/to/generated_prompts.jsonl --n-partitions 100 --partition 0
```

デフォルトのパラメータは私たちのデータセットと一致していますが、実際にはより少ないステップ数（例：`--steps=25`）を使用して、高品質なデータをより速く生成できます。デフォルトでは、プロンプトごとに100個のサンプルを生成し、CLIPフィルタリングを使用してプロンプトごとに最大4個を保持します。`--n-samples`を設定することで、より少ないサンプル数で実験できます。以下のコマンドはCLIPフィルタリングを完全に無効にし、したがってより高速です：

```
python dataset_creation/generate_img_dataset.py --out_dir data/instruct-pix2pix-dataset-000 --prompts_file path/to/generated_prompts.jsonl --n-samples 4 --clip-threshold 0 --clip-dir-threshold 0 --clip-img-threshold 0 --n-partitions 100 --partition 0
```

すべてのデータセット例を生成した後、以下のコマンドを実行して例のリストを作成します。これは、各トレーニング実行の開始時にデータセットディレクトリ全体を反復処理する必要なく、データセットオブジェクトが効率的に例をサンプリングできるようにするために必要です。

```
python dataset_creation/prepare_dataset.py data/instruct-pix2pix-dataset-000
```

## 評価

論文の図8と10のようなプロットを生成するには、以下のコマンドを実行します：

```
python metrics/compute_metrics.py --ckpt /path/to/your/model.ckpt
```

## ヒント

期待する品質の結果が得られない場合、以下の理由が考えられます：

1. **画像の変化が十分でない場合**：Image CFGの重みが高すぎる可能性があります。この値は出力が入力にどれだけ似ているべきかを示します。編集に元の画像からの大きな変更が必要な場合、Image CFGの重みがそれを許容していない可能性があります。または、Text CFGの重みが低すぎる可能性があります。この値はテキスト指示にどれだけ従うかを示します。デフォルトのImage CFG 1.5とText CFG 7.5は良い出発点ですが、各編集に最適とは限りません。以下を試してください：
    * Image CFGの重みを下げる、または
    * Text CFGの重みを上げる、または

2. 逆に、**画像の変化が大きすぎる**場合（元の画像の詳細が保持されていない）：以下を試してください：
    * Image CFGの重みを上げる、または
    * Text CFGの重みを下げる

3. "Randomize Seed"を設定して異なるランダムシードで結果を生成してみてください。また、"Randomize CFG"を設定して、毎回新しいText CFGとImage CFGの値をサンプリングすることもできます。

4. 指示の言い回しを変えると結果が改善することがあります（例：「turn him into a dog」vs.「make him a dog」vs.「as a dog」）。

5. ステップ数を増やすと結果が改善することがあります。

6. 顔がおかしく見える場合：Stable Diffusionのオートエンコーダーは、画像内で小さな顔を処理するのが苦手です。顔がフレームのより大きな部分を占めるように画像をクロップしてみてください。

## コメント

- 私たちのコードベースは[Stable Diffusionコードベース](https://github.com/CompVis/stable-diffusion)をベースにしています。

## BibTeX

```
@article{brooks2022instructpix2pix,
  title={InstructPix2Pix: Learning to Follow Image Editing Instructions},
  author={Brooks, Tim and Holynski, Aleksander and Efros, Alexei A},
  journal={arXiv preprint arXiv:2211.09800},
  year={2022}
}
```

## InstructPix2Pixの他の使用方法

### [HuggingFace](https://huggingface.co/spaces/timbrooks/instruct-pix2pix)でのInstructPix2Pix:
> デモのブラウザベースバージョンが[HuggingFace space](https://huggingface.co/spaces/timbrooks/instruct-pix2pix)として利用可能です。このバージョンでは、ブラウザ、編集したい画像、指示だけが必要です！これは共有のオンラインデモであり、ピーク時の利用時には処理時間が遅くなる可能性があることに注意してください。

### [Replicate](https://replicate.com/timothybrooks/instruct-pix2pix)でのInstructPix2Pix:
> Replicateは、InstructPix2Pixモデルを実行するための本番環境対応のクラウドAPIを提供しています。cURL、Python、JavaScript、または任意の言語を使用して、シンプルなAPIコールでモデルを実行できます。Replicateはまた、モデルを実行して予測を共有するためのWebインターフェースも提供しています。

### [Imaginairy](https://github.com/brycedrennan/imaginAIry#-edit-images-with-instructions-alone-by-instructpix2pix)でのInstructPix2Pix:
> Imaginairyは、InstructPix2Pixを簡単にインストールする別の方法を提供しています。GPUのないデバイス（Macbookなど）でも実行できます！
> ```bash
> pip install imaginairy --upgrade
> aimg edit any-image.jpg --gif "turn him into a cyborg" 
> ```
> また、画像に対して複数の編集を簡単に実行し、編集をアニメーションGIFとして保存することもできます：
> ```
> aimg edit --gif --surprise-me pearl-earring.jpg 
> ```
> <img src="https://raw.githubusercontent.com/brycedrennan/imaginAIry/7c05c3aae2740278978c5e84962b826e58201bac/assets/girl_with_a_pearl_earring_suprise.gif" width="512">

### [🧨 Diffusers](https://github.com/huggingface/diffusers)でのInstructPix2Pix:

> DiffusersでのInstructPix2Pixは少し最適化されているため、より高速で、メモリの少ないGPUにより適している可能性があります。以下はライブラリのインストールと画像の編集の手順です：
> 1. diffusersと関連する依存関係をインストール：
>
> ```bash
> pip install transformers accelerate torch
>
> pip install git+https://github.com/huggingface/diffusers.git
> ```
> 
> 2. モデルをロードして画像を編集：
>
> ```python
> 
> import torch
> from diffusers import StableDiffusionInstructPix2PixPipeline, EulerAncestralDiscreteScheduler
> 
> model_id = "timbrooks/instruct-pix2pix"
> pipe = StableDiffusionInstructPix2PixPipeline.from_pretrained(model_id, torch_dtype=torch.float16, safety_checker=None)
> pipe.to("cuda")
> pipe.scheduler = EulerAncestralDiscreteScheduler.from_config(pipe.scheduler.config)
> # `image`はRGB PIL.Imageです
``` 