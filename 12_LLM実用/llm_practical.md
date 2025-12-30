# LLM実用編

前項でROCmバックエンドよりもVulkanバックエンドのほうが高いパフォーマンスが得られることがわかりました。以下ではVulkanバックエンドのみで計測を行っています。

## テスト項目

MS-S1 MAX最大の特徴といえば、最大96GBのビデオメモリにあるといっていいでしょう。このビデオメモリをLLMで活用する方法としては以下が考えられます。

- とにかくパラメータ数の多いモデルを使う
- パラメータ数はそこまで多くはないが量子化が控えめのモデルを使う
- コンテクストウィンドウを広げる

それぞれについて検討します。

### パラメータ数の多いモデル

量子化とはパラメータの情報量を落とすことでモデルのサイズを減らし、計算も高速化する技法です。
もちろんそのぶん精度も落ちるのですが、一般には8bit量子化ではほぼ影響はなく、4bit量子化（とくに `Q4_K_M` ）でも実用範囲にとどまると言われています。

まずは4bit量子化しても60GB以上のサイズがある、100Bパラメータ級のモデルを試します。gpt-oss 120bは4bit量子化で60GBを切るのですが、参考情報として併記しています。

- [bartowski/Mixtral-8x22B-v0.1-GGUF](https://huggingface.co/bartowski/Mixtral-8x22B-v0.1-GGUF)
- [unsloth/GLM-4.5-Air-GGUF](https://huggingface.co/unsloth/GLM-4.5-Air-GGUF)
- [unsloth/GLM-4.6V-GGUF](https://huggingface.co/unsloth/GLM-4.6V-GGUF)
- [unsloth/Devstral-2-123B-Instruct-2512-GGUF](https://huggingface.co/unsloth/Devstral-2-123B-Instruct-2512-GGUF)
- [unsloth/Llama-4-Scout-17B-16E-Instruct-GGUF](https://huggingface.co/unsloth/Llama-4-Scout-17B-16E-Instruct-GGUF)

```bat
llama-bench.exe -m unsloth_GLM-4.6V-GGUF_Q4_K_M_GLM-4.6V-Q4_K_M-00001-of-00002.gguf ^
  -m unsloth_GLM-4.5-Air-GGUF_Q4_K_M_GLM-4.5-Air-Q4_K_M-00001-of-00002.gguf ^
  -m bartowski_Mixtral-8x22B-v0.1-GGUF_Mixtral-8x22B-v0.1-Q4_K_M-00001-of-00005.gguf ^
  -m unsloth_Devstral-2-123B-Instruct-2512-GGUF_Q4_K_M_Devstral-2-123B-Instruct-2512-Q4_K_M-00001-of-00002.gguf ^
  -m unsloth_Llama-4-Scout-17B-16E-Instruct-GGUF_Q4_K_M_Llama-4-Scout-17B-16E-Instruct-Q4_K_M-00001-of-00002.gguf
```

| model                                |      size |   params | backend | ngl |  test |           t/s |
| ------------------------------------ | --------: | -------: | ------- | --: | ----: | ------------: |
| glm4moe ?B Q4_K - Medium             | 65.71 GiB | 106.85 B | Vulkan  |  99 | pp512 | 254.72 ± 4.73 |
| glm4moe ?B Q4_K - Medium             | 65.71 GiB | 106.85 B | Vulkan  |  99 | tg128 |  21.61 ± 0.27 |
| glm4moe 106B.A12B Q4_K - Medium      | 67.96 GiB | 110.47 B | Vulkan  |  99 | pp512 | 264.38 ± 4.47 |
| glm4moe 106B.A12B Q4_K - Medium      | 67.96 GiB | 110.47 B | Vulkan  |  99 | tg128 |  22.79 ± 0.18 |
| llama 8x22B Q4_K - Medium            | 79.71 GiB | 140.62 B | Vulkan  |  99 | pp512 | 116.53 ± 3.97 |
| llama 8x22B Q4_K - Medium            | 79.71 GiB | 140.62 B | Vulkan  |  99 | tg128 |   8.99 ± 0.00 |
| llama ?B Q4_K - Medium               | 69.75 GiB | 125.03 B | Vulkan  |  99 | pp512 |  42.00 ± 0.99 |
| llama ?B Q4_K - Medium               | 69.75 GiB | 125.03 B | Vulkan  |  99 | tg128 |   2.99 ± 0.00 |
| llama4 17Bx16E (Scout) Q4_K - Medium | 60.86 GiB | 107.77 B | Vulkan  |  99 | pp512 | 249.26 ± 2.26 |
| llama4 17Bx16E (Scout) Q4_K - Medium | 60.86 GiB | 107.77 B | Vulkan  |  99 | tg128 |  19.07 ± 0.04 |
| （参考）gpt-oss 120B Q4_K - Medium   | 58.45 GiB | 116.83 B | Vulkan  |  99 | pp512 | 577.72 ± 7.66 |
| （参考）gpt-oss 120B Q4_K - Medium   | 58.45 GiB | 116.83 B | Vulkan  |  99 | tg128 |  56.14 ± 0.14 |

おおよそ20tpsでテキストを生成できるモデルと10tpsを割り込む低速のモデルに分かれます。低速のモデルはメインメモリが不足してスワップを使い始めたのでしょう。

それにしてもなんでgpt-ossはこんなに速いのでしょうか……

### 量子化が控えめのモデルを使う

ここでは8bit量子化で60GB以上のサイズがある、70B級のモデルを試します。

- [unsloth/Qwen3-Next-80B-A3B-Instruct-GGUF](https://huggingface.co/unsloth/Qwen3-Next-80B-A3B-Instruct)
- [unsloth/gpt-oss-120b-GGUF](https://huggingface.co/unsloth/gpt-oss-120b-GGUF)
- [unsloth/Llama-3.3-70B-Instruct-GGUF](https://huggingface.co/unsloth/Llama-3.3-70B-Instruct-GGUF)

```bat
llama-bench.exe -m unsloth_Qwen3-Next-80B-A3B-Instruct-GGUF_UD-Q8_K_XL_Qwen3-Next-80B-A3B-Instruct-UD-Q8_K_XL-00001-of-00002.gguf ^
-m unsloth_gpt-oss-120b-GGUF_UD-Q8_K_XL_gpt-oss-120b-UD-Q8_K_XL-00001-of-00002.gguf ^
-m unsloth_Llama-3.3-70B-Instruct-GGUF_UD-Q8_K_XL_Llama-3.3-70B-Instruct-UD-Q8_K_XL-00001-of-00002.gguf
```

| model                  |      size |   params | backend | ngl |  test |           t/s |
| ---------------------- | --------: | -------: | ------- | --: | ----: | ------------: |
| qwen3next 80B.A3B Q8_0 | 79.57 GiB |  79.67 B | Vulkan  |  99 | pp512 | 355.67 ± 2.44 |
| qwen3next 80B.A3B Q8_0 | 79.57 GiB |  79.67 B | Vulkan  |  99 | tg128 |  32.76 ± 0.17 |
| gpt-oss 120B Q8_0      | 60.03 GiB | 116.83 B | Vulkan  |  99 | pp512 | 546.24 ± 4.57 |
| gpt-oss 120B Q8_0      | 60.03 GiB | 116.83 B | Vulkan  |  99 | tg128 |  45.07 ± 0.18 |
| llama 70B Q8_0         | 75.65 GiB |  70.55 B | Vulkan  |  99 | pp512 |  94.23 ± 0.42 |
| llama 70B Q8_0         | 75.65 GiB |  70.55 B | Vulkan  |  99 | tg128 |   2.78 ± 0.00 |

QwenとGPT-OSSで実用的なパフォーマンスが出ていることが確認できます。

### 長いコンテクストを扱う

長いコンテクストに対応するほど多くのGPUメモリが必要になります。ここでは100k（10万トークン）以上のコンテクストに対応したモデルを試します。
モデルが対応しているコンテクストウィンドウのサイズは `config.json` の `max_position_embedding` でわかります。

メジャーなモデルで最長は [Llama-4-Scout](https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct) の10Mトークンでしょう。[unslothによる量子化モデル](https://huggingface.co/unsloth/Llama-4-Scout-17B-16E-Instruct-GGUF) で試してみたのですが、残念ながらメモリ不足で最大のサイズでは利用できませんでした。

`llama.cpp` ではコンテクストのサイズを `--ctx-size` で指定します。

```sh
llama-server.exe -m model-name.gguf --ctx-size 131072
```

いくつかのモデルで大きめのテキストを与えて要約をさせてみます。ここでは [令和7年の防災白書](https://www.bousai.go.jp/kaigirep/hakusho/r7.html) から第一部第一章のPDFを使用しました。


```sh
# unsloth_gpt-oss-120b-GGUF_Q4_K_M_gpt-oss-120b-Q4_K_M-00001-of-00002.gguf
prompt eval time =  415221.38 ms / 66230 tokens (    6.27 ms per token,   159.51 tokens per second)
       eval time =  118172.25 ms /  2345 tokens (   50.39 ms per token,    19.84 tokens per second)
      total time =  533393.63 ms / 68575 tokens

# unsloth_Qwen3-Next-80B-A3B-Thinking-GGUF_Qwen3-Next-80B-A3B-Thinking-Q4_K_M.gguf
prompt eval time =  637651.84 ms / 62382 tokens (   10.22 ms per token,    97.83 tokens per second)
       eval time =   83942.09 ms /  2425 tokens (   34.62 ms per token,    28.89 tokens per second)
      total time =  721593.93 ms / 64807 tokens

# unsloth_Llama-4-Scout-17B-16E-Instruct-GGUF_Q4_K_M_Llama-4-Scout-17B-16E-Instruct-Q4_K_M-00001-of-00002.gguf
prompt eval time =  497514.54 ms / 52924 tokens (    9.40 ms per token,   106.38 tokens per second)
       eval time =   39398.06 ms /   331 tokens (  119.03 ms per token,     8.40 tokens per second)
      total time =  536912.60 ms / 53255 tokens

# ggml-org_Ministral-3-14B-Reasoning-2512-GGUF_Ministral-3-14B-Reasoning-2512-Q8_0.gguf
prompt eval time =  990965.07 ms / 68704 tokens (   14.42 ms per token,    69.33 tokens per second)
       eval time =  398162.47 ms /  2080 tokens (  191.42 ms per token,     5.22 tokens per second)
      total time = 1389127.55 ms / 70784 tokens

# bartowski_Phi-3-medium-128k-instruct-GGUF_Phi-3-medium-128k-instruct-Q8_0.gguf
prompt eval time = 2896919.55 ms / 106208 tokens (   27.28 ms per token,    36.66 tokens per second)
       eval time =  530796.25 ms /  1660 tokens (  319.76 ms per token,     3.13 tokens per second)
      total time = 3427715.80 ms / 107868 tokens

# unsloth_GLM-4.5-Air-GGUF_Q4_K_M_GLM-4.5-Air-Q4_K_M-00001-of-00002.gguf
prompt eval time = 2690006.85 ms / 61076 tokens (   44.04 ms per token,    22.70 tokens per second)
       eval time =  396285.37 ms /  2149 tokens (  184.40 ms per token,     5.42 tokens per second)
      total time = 3086292.22 ms / 63225 tokens
```

| モデル                                | pp(tps) | tg(tps) |     pp(ms) |    tg(ms) |   合計(ms) |
| ------------------------------------- | ------: | ------: | ---------: | --------: | ---------: |
| gpt-oss-120b Q4_K_M                   |  159.51 |   19.84 |  415221.38 | 118172.25 |  533393.63 |
| qwen3-next-80B-A3B-thinking Q4_K_M    |   97.83 |   28.89 |  637651.84 |  83942.09 |  721593.93 |
| llama-4-scout-17B-16E-Instruct Q4_K_M |  106.38 |    8.40 |  497514.54 |  39398.06 |  536912.60 |
| ministral                             |   69.33 |    5.22 |  990965.07 | 398162.47 | 1389127.55 |
| Phi-3                                 |   36.66 |    3.13 | 2896919.55 | 530796.25 | 3427715.80 |
| GLM-4.5-Air                           |   22.70 |    5.42 | 2690006.85 | 396285.37 | 3086292.22 |

一番右側の列が実行開始してから応答が完了するまでのトータルでかかった時間です。

最短はGPT-OSSとllama-4-scoutの9分弱、最長はPhi-3の1時間弱となり、モデルによって6倍近くの差が生じています。
小さいモデルであれば長いコンテクストに対しても高速に応答が返るのではないか？と考えるかもしれませんが、予想に反した結果です。

実際のところ、プロンプトを与えてから結果が出るまで10分待つというのはあまり実用的ではありません。
全体として大きなコンテクストウィンドウを扱わせるには不向きだといえるでしょう。

### トークン数について

LLMでは文字列をトークンに分割して計算します。この分割のことをトークナイズ、分割するモジュールのことをトークナイザといいます。

トークナイザはモデルによって異なるため、上記のテスト結果でも入力トークン数（モデルごとの最初の行の数値）はモデルによってまちまちです。

ここで気になるのはPhi-3です。Phi-3は4bと小規模ながら128kのコンテクストウィンドウをアピールポイントにしているモデルです。
しかし他のモデルでは入力を6万トークン程度としているのに対し、Phi-3は10万トークン以上と大幅に多くカウントしています。
おそらくこれは辞書サイズによるもので、Phi-3は1トークンあたりの情報量が少ないため同じだけのテキストを表現するのに必要なトークン数が多いということでしょう。
一般化すると小規模なモデルほどトークン数が増えると思われますが、このあたりを正規化した比較もしたいところです。

## まとめ

面白味の少ない結論ではありますが、MS-S1 MAXの性能を活用できるモデルとしてはgpt-oss-120bがちょうどよさそうです。
コンテクストウィンドウを広げなくていい場合はGLM系、Qwen系が次点の候補になります。
