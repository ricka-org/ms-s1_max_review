# LLMベンチマーク

llama.cppの `llama-bench` を用いてLLM推論の性能を計測します。llama.cppはバージョン7541の公式ビルドを使用しています。

[Release b7541 · ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp/releases/tag/b7541)

llama.cppはLLMでの推論にもっとも広く使われているフレームワークです。Ollama、LMStudio、text-generation-webuiなどのLLMアプリケーションもバックエンドとしてllama.cppを使用しています。

## 専用ビデオメモリ割り当て

LLMベンチマークでは極力多くの処理をGPUで行えるよう、一律に専用ビデオメモリとして最大の96GBを割り当てています。

## 参考：llama-benchのテスト項目

llama-benchのテスト項目は「pp」と「tg」の2つです。

ppは `Prompt Processing` の略で、入力されたプロンプトの処理速度を示します。

tgは `Text Generation` の略で、文章生成の速度を示します。

いずれも単位は1秒あたりに処理したトークン数 `t/s` （token / second）で、数字が大きいほどパフォーマンスが高いことを示します。


## バックエンドによるパフォーマンス差

llama.cppは複数のバックエンド（プラットフォーム）に対応しています。Windows PCでAMD GPUを使う場合、以下のバックエンドが利用できます。

- Vulkan
  - ベンダーを問わないGPGPUプラットフォームです
  - もともとは3Dグラフィックス用でしたが、GPGPUにも利用できます
- ROCm
  - AMDのGPU用のGPGPUプラットフォームです
  - NVIDIAのCUDAに相当します
- Sycl
  - 汎用の計算フレームワークです
- CPU
  - GPUを使わずCPUで処理を行います

Syclは動作しなかったため、残りの3つ（Vulkan、ROCm、CPU）のバックエンドで比較を行います。使用するモデルは以下の6つです。

* unsloth/gpt-oss-20b-gguf
  * Q4_K
  * Q8_0
  * F16
* unsloth/gpt-oss-120b-gguf
  * Q4_K
  * Q8_0
  * F16

unslothは定評のある量子化作者です。

### Vulkan

```bat
llama-bench.exe -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-Q4_K_M.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-UD-Q8_K_XL.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-F16.gguf ^
                -m unsloth_gpt-oss-120b-GGUF_Q4_K_M_gpt-oss-120b-Q4_K_M-00001-of-00002.gguf ^
                -m unsloth_gpt-oss-120b-GGUF_UD-Q8_K_XL_gpt-oss-120b-UD-Q8_K_XL-00001-of-00002.gguf ^
                -m unsloth_gpt-oss-120b-GGUF_gpt-oss-120b-F16.gguf
```

| model                      |      size |   params | backend |  ngl |  test |            t/s |
| -------------------------- | --------: | -------: | ------- | ---: | ----: | -------------: |
| gpt-oss 20B Q4_K - Medium  | 10.81 GiB |  20.91 B | Vulkan  |   99 | pp512 | 1531.08 ± 7.83 |
| gpt-oss 20B Q4_K - Medium  | 10.81 GiB |  20.91 B | Vulkan  |   99 | tg128 |   80.84 ± 0.13 |
| gpt-oss 20B Q8_0           | 12.28 GiB |  20.91 B | Vulkan  |   99 | pp512 | 1546.21 ± 5.26 |
| gpt-oss 20B Q8_0           | 12.28 GiB |  20.91 B | Vulkan  |   99 | tg128 |   62.79 ± 0.33 |
| gpt-oss 20B F16            | 12.83 GiB |  20.91 B | Vulkan  |   99 | pp512 | 1461.57 ± 7.01 |
| gpt-oss 20B F16            | 12.83 GiB |  20.91 B | Vulkan  |   99 | tg128 |   47.64 ± 0.03 |
| gpt-oss 120B Q4_K - Medium | 58.45 GiB | 116.83 B | Vulkan  |   99 | pp512 |  577.72 ± 7.66 |
| gpt-oss 120B Q4_K - Medium | 58.45 GiB | 116.83 B | Vulkan  |   99 | tg128 |   56.14 ± 0.14 |
| gpt-oss 120B Q8_0          | 60.03 GiB | 116.83 B | Vulkan  |   99 | pp512 |  549.93 ± 9.36 |
| gpt-oss 120B Q8_0          | 60.03 GiB | 116.83 B | Vulkan  |   99 | tg128 |   45.85 ± 0.16 |
| gpt-oss 120B F16           | 60.87 GiB | 116.83 B | Vulkan  |   99 | pp512 |  529.75 ± 4.59 |
| gpt-oss 120B F16           | 60.87 GiB | 116.83 B | Vulkan  |   99 | tg128 |   33.91 ± 0.16 |


### ROCm

```bat
llama-bench.exe -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-Q4_K_M.gguf^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-UD-Q8_K_XL.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-F16.gguf ^
                -m unsloth_gpt-oss-120b-GGUF_Q4_K_M_gpt-oss-120b-Q4_K_M-00001-of-00002.gguf ^
                -m unsloth_gpt-oss-120b-GGUF_UD-Q8_K_XL_gpt-oss-120b-UD-Q8_K_XL-00001-of-00002.gguf ^
                -m unsloth_gpt-oss-120b-GGUF_gpt-oss-120b-F16.gguf
```

| model                      |      size |   params | backend |  ngl |  test |             t/s |
| -------------------------- | --------: | -------: | ------- | ---: | ----: | --------------: |
| gpt-oss 20B Q4_K - Medium  | 10.81 GiB |  20.91 B | ROCm    |   99 | pp512 | 1384.71 ± 16.22 |
| gpt-oss 20B Q4_K - Medium  | 10.81 GiB |  20.91 B | ROCm    |   99 | tg128 |    73.83 ± 0.08 |
| gpt-oss 20B Q8_0           | 12.28 GiB |  20.91 B | ROCm    |   99 | pp512 |  1370.69 ± 6.81 |
| gpt-oss 20B Q8_0           | 12.28 GiB |  20.91 B | ROCm    |   99 | tg128 |    58.64 ± 0.05 |
| gpt-oss 20B F16            | 12.83 GiB |  20.91 B | ROCm    |   99 | pp512 | 1025.50 ± 19.66 |
| gpt-oss 20B F16            | 12.83 GiB |  20.91 B | ROCm    |   99 | tg128 |    49.22 ± 0.21 |
| gpt-oss 120B Q4_K - Medium | 58.45 GiB | 116.83 B | ROCm    |   99 | pp512 |   437.78 ± 5.48 |
| gpt-oss 120B Q4_K - Medium | 58.45 GiB | 116.83 B | ROCm    |   99 | tg128 |    40.90 ± 0.08 |
| gpt-oss 120B Q8_0          | 60.03 GiB | 116.83 B | ROCm    |   99 | pp512 |   419.58 ± 8.02 |
| gpt-oss 120B Q8_0          | 60.03 GiB | 116.83 B | ROCm    |   99 | tg128 |    33.57 ± 0.19 |
| gpt-oss 120B F16           | 60.87 GiB | 116.83 B | ROCm    |   99 | pp512 |   317.21 ± 9.71 |
| gpt-oss 120B F16           | 60.87 GiB | 116.83 B | ROCm    |   99 | tg128 |    29.17 ± 0.11 |


### CPU

CPUはGPT-OSS 120Bでの推論には遅すぎるため、20Bでのみベンチマークを実施しています。16スレッド（デフォルト）と32スレッド（全論理CPUを使用）の2通りで実施しています。

GPUでの処理に比べてppで1/10程度、tgでも半分以下のパフォーマンスしかありません。それでも20tpsというチャット用途に使う上では実用的な速度を出しているのはさすがのスペックといったところです。

もっともGPU推論時とCPU推論時でほぼ消費電力は変わらない（どちらも170W超）ため、効率でいえばGPU推論の足元にも及びません。

```bat
llama-bench.exe -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-Q4_K_M.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-UD-Q8_K_XL.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-F16.gguf
```


| model                     |      size |  params | backend | threads |  test |           t/s |
| ------------------------- | --------: | ------: | ------- | ------: | ----: | ------------: |
| gpt-oss 20B Q4_K - Medium | 10.81 GiB | 20.91 B | CPU     |      16 | pp512 | 112.68 ± 0.83 |
| gpt-oss 20B Q4_K - Medium | 10.81 GiB | 20.91 B | CPU     |      16 | tg128 |  26.66 ± 0.36 |
| gpt-oss 20B Q8_0          | 12.28 GiB | 20.91 B | CPU     |      16 | pp512 | 109.45 ± 0.97 |
| gpt-oss 20B Q8_0          | 12.28 GiB | 20.91 B | CPU     |      16 | tg128 |  21.40 ± 0.32 |
| gpt-oss 20B F16           | 12.83 GiB | 20.91 B | CPU     |      16 | pp512 | 112.45 ± 1.57 |
| gpt-oss 20B F16           | 12.83 GiB | 20.91 B | CPU     |      16 | tg128 |  18.35 ± 0.05 |


```bat
llama-bench.exe -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-Q4_K_M.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-UD-Q8_K_XL.gguf ^
                -m unsloth_gpt-oss-20b-GGUF_gpt-oss-20b-F16.gguf ^
                -t 32
```

| model                     |      size |  params | backend | threads |  test |           t/s |
| ------------------------- | --------: | ------: | ------- | ------: | ----: | ------------: |
| gpt-oss 20B Q4_K - Medium | 10.81 GiB | 20.91 B | CPU     |      32 | pp512 | 170.73 ± 2.32 |
| gpt-oss 20B Q4_K - Medium | 10.81 GiB | 20.91 B | CPU     |      32 | tg128 |  33.37 ± 0.14 |
| gpt-oss 20B Q8_0          | 12.28 GiB | 20.91 B | CPU     |      32 | pp512 | 162.72 ± 1.38 |
| gpt-oss 20B Q8_0          | 12.28 GiB | 20.91 B | CPU     |      32 | tg128 |  27.94 ± 0.21 |
| gpt-oss 20B F16           | 12.83 GiB | 20.91 B | CPU     |      32 | pp512 | 168.73 ± 1.32 |
| gpt-oss 20B F16           | 12.83 GiB | 20.91 B | CPU     |      32 | tg128 |  23.86 ± 0.70 |

## まとめ

AMDのGPGPUフレームワークであるROCｍを使ったバックエンドよりも、汎用のVulkanバックエンドのほうがやや良好なパフォーマンスという意外な結果となりました。まだROCｍの最適化が進んでいない可能性がありますので、今後の開発に期待したいところです。
