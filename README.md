# AI-8850 Acceleration Module Driver

Backup mirror and practical guide for the **M5Stack LLM-8850 / AI-8850 LLM acceleration M.2 module**, based on the AXERA AX8850 NPU.

## About this repository

This repository preserves Linux driver sources for the module because the original M5Stack download source has had poor availability. It also documents the supported software stack, installation on Raspberry Pi 5, ready-to-use models, and the main limitations of the accelerator.

The repository contains the driver package [`axcl_host_3_6_5.deb`](axcl_host_3_6_5.deb). The package is retained as an archive; for a new installation, prefer the current official `axclhost` package and verify compatibility with the installed firmware and runtime versions.

## Quick orientation

- The card is a vendor-specific NPU accelerator, not a CUDA or ROCm device.
- Models use AXERA's `.axmodel` format. PyTorch, ONNX, and GGUF files cannot be executed directly.
- Raspberry Pi RAM is separate from the card's 8 GB LPDDR4x memory.
- The most direct host is a 64-bit Raspberry Pi 5 running Raspberry Pi OS/Debian.
- This README describes the 8 GB LLM-8850 Card/Kit. Details may differ for other variants.

## Support status

As of September 18, 2026, the card has the AXCL PCIe stack, C/C++ and Python APIs, an LLM/VLM runtime, an OpenAI-compatible server, and ready ports for computer vision, speech recognition, speech synthesis, and image generation. Most ready ports come from AXERA, M5Stack, or closely related contributors, so runtime and model versions must be checked together.

The main constraints are:

- support for a model architecture and its operators matters more than the advertised 24 TOPS;
- card memory is shared by weights, KV cache, inputs, outputs, and working buffers;
- independent community support is still relatively small;
- published benchmarks usually measure a graph or decode loop, not the complete Raspberry Pi application pipeline.

## What the accelerator is

M5Stack LLM-8850 is built around AXERA AX8850. Documentation and software output may call the device `AX650N`, while model compilation uses the `AX650` target. This is expected SDK naming inheritance.

Key specifications:

- up to 24 INT8 TOPS;
- NPU with three compute cores;
- eight Cortex-A55 cores up to 1.7 GHz on the card;
- 8 GB LPDDR4x in the version covered here;
- M.2 2242, PCIe 2.0 x2;
- hardware video and multimedia blocks.

In a PCIe deployment, Raspberry Pi is the host and the card is a separate compute device. Pi RAM does not extend NPU memory. A model that appears to require only 2 GB can need considerably more once context, KV cache, and runtime buffers are included.

## Raspberry Pi 5 installation

Fully power off Raspberry Pi before connecting the board. Hot installation is prohibited.

### Power and cooling

- With the LLM-8850 PiHat, power the whole system through the adapter board PD input. The supply must exceed `9 V x 3 A` (27 W).
- With the official Raspberry Pi M.2 HAT+, M5Stack recommends a `5 V x 3 A` non-PD switching supply. Some PD supplies may not deliver maximum power because of protocol negotiation.
- Active cooling for both Raspberry Pi and the card is required for sustained LLM, VLM, ASR, or SD inference.
- Unstable power can appear as PCIe, driver, or intermittent inference failures.

See the official [LLM-8850 Card Hardware Installation](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_hardware_install) guide.

## Installing AXCL

Supported hosts include aarch64 and x86_64 Ubuntu/Debian, Raspberry Pi 5, and Windows. macOS, WSL, and virtual machines are not officially supported.

### 1. Update the system and EEPROM

```bash
sudo apt update && sudo apt full-upgrade
sudo rpi-eeprom-update
```

If the EEPROM predates December 6, 2023, use `sudo raspi-config` and select `Advanced Options` -> `Bootloader Version` -> `Latest`, then run:

```bash
sudo rpi-eeprom-update -a
sudo reboot
```

After reboot, confirm that the card is visible:

```bash
lspci | grep -i -E 'axera|0650'
```

The expected line contains `Axera Semiconductor ... Device 0650`.

### 2. Install the driver and host runtime

DKMS needs `gcc`, `make`, `patch`, and current kernel headers. The official M5Stack repository and host package can be installed with:

```bash
sudo apt install -y gcc make patch linux-headers-$(uname -r) dkms
sudo wget -qO /etc/apt/keyrings/StackFlow.gpg \
	https://repo.llm.m5stack.com/m5stack-apt-repo/key/StackFlow.gpg
echo 'deb [signed-by=/etc/apt/keyrings/StackFlow.gpg] https://repo.llm.m5stack.com/m5stack-apt-repo axclhost main' \
	| sudo tee /etc/apt/sources.list.d/axclhost.list
sudo apt update
sudo apt install -y axclhost
source /etc/profile
```

Verify the device:

```bash
axcl-smi
```

The output should show the card, temperature, CPU/NPU use, CMM, and processes. Current AXCL documentation is `V3.16.0`; the M5Stack page may still show an older `V3.6.4` screenshot. Current `ax-llm` requires SDK 3.16.0 or later, so check the installed version rather than relying on a screenshot.

Useful diagnostics:

```bash
axcl-smi
watch -n 1 axcl-smi
dmesg | grep -i -E 'axcl|axera|pcie'
```

### 3. Run the first NPU test

```bash
wget https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/linux/ax8850_card/yolo11s.axmodel
axcl_run_model -m yolo11s.axmodel -r 10
```

The reported approximately 3.4 ms is pure graph execution and does not include decode, preprocessing, or application postprocessing.

## Ways to use the card

### StackFlow applications and OpenAI API

M5Stack's packages install services and models. Example:

```bash
sudo apt install lib-llm llm-sys llm-llm llm-openai-api
sudo apt install llm-model-qwen3-1.7b-int8-ctx-axcl
sudo systemctl restart llm-openai-api
curl http://127.0.0.1:8000/v1/models
```

Restart the service after installing each model. Example request:

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
	-H 'Content-Type: application/json' \
	-H 'Authorization: Bearer sk-local' \
	-d '{
		"model": "qwen3-1.7B-Int8-ctx-axcl",
		"messages": [{"role": "user", "content": "Briefly explain what an NPU is"}]
	}'
```

See the [M5Stack OpenAI API guide](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_openai). Do not expose this port to the Internet without authentication, TLS, a reverse proxy, and rate limiting.

### ax-llm for LLM and VLM

[AXERA-TECH/ax-llm](https://github.com/AXERA-TECH/ax-llm) is the primary modern runtime. It supports AX8850 through AXCL aarch64, an interactive CLI, an OpenAI-compatible server, image/video VLM inputs, context controls, and memory management.

```bash
git clone https://github.com/AXERA-TECH/ax-llm.git
cd ax-llm
./install.sh
./build_axcl_aarch64.sh

axllm run /path/to/model_directory
axllm serve /path/to/model_directory --port 8000
```

Use the instructions and parameters from the repository version matching the downloaded model. Older model directories may contain separate layer `.axmodel` files and their own configuration.

### C/C++ and Python

The low-level C/C++ route is appropriate for minimum-latency multimedia pipelines:

- initialize AXCL and select a device;
- allocate device/host memory and DMA;
- load an `.axmodel` and create an execution context;
- copy input, execute, and retrieve output;
- use hardware decode/encode, IVPS, IVE, and other blocks.

Start with [axcl-samples](https://github.com/AXERA-TECH/axcl-samples) and [axcl-pi5-examples](https://github.com/AXERA-TECH/axcl-pi5-examples). The [AXCL SDK API reference](https://axcl-docs.readthedocs.io/en/latest/doc_guide_axcl_api.html) covers the runtime API.

[pyaxcl](https://github.com/AXERA-TECH/pyaxcl) provides Python bindings for runtime, NPU inference, codec, IVPS, IVE, and DMA. Some ready models use [pyaxengine](https://github.com/AXERA-TECH/pyaxengine) with `AXCLRTExecutionProvider`. Python is useful for orchestration and preprocessing; keep critical loops on the NPU/C++ side and avoid moving intermediate tensors over PCIe after every layer.

## Recommended models for 8 GB

Published speeds are for the relevant AXERA build and may omit tokenization, prefill, network overhead, preprocessing, postprocessing, and host PCIe effects.

| Workload | First choice | Why |
|---|---|---|
| General local LLM | [Qwen3.5-4B Int4](https://huggingface.co/AXERA-TECH/Qwen3.5-4B-GPTQ-Int4) | Strong recent model that fits with controlled context; about 5.7 tok/s |
| Fast LLM or agent | [Qwen3-1.7B Int4](https://huggingface.co/AXERA-TECH/Qwen3-1.7B-GPTQ-Int4) | Good quality, speed, and memory balance; about 12.7 tok/s |
| Lowest latency | [Qwen3-0.6B Int4](https://huggingface.co/AXERA-TECH/Qwen3-0.6B-GPTQ-Int4) | About 20.2 tok/s; commands, classification, simple agents |
| Images and OCR | [Qwen3.5-2B](https://huggingface.co/AXERA-TECH/Qwen3.5-2B-GPTQ-Int4) or [Qwen3-VL-2B](https://huggingface.co/AXERA-TECH/Qwen3-VL-2B-Instruct) | Practical multimodal balance; Qwen3.5 is about 13.4 tok/s |
| Low-latency video | [FastVLM-0.5B](https://huggingface.co/AXERA-TECH/FastVLM-0.5B) | Compact VLM for cameras and frequent frames |
| Russian speech-to-text | [Whisper Small](https://huggingface.co/M5Stack/whisper-small-axmodel) | Better Russian accuracy than Tiny/Base; use VAD for long audio |
| Fastest ASR | [Whisper Turbo](https://huggingface.co/AXERA-TECH/whisper-turbo) | Fast ready Whisper build; verify quality and memory on your audio |
| Text-to-speech | [CosyVoice2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_cosy_voice2) | Capable ready TTS port with an API wrapper |
| Image generation | [LCM-LoRA SD 1.5](https://huggingface.co/AXERA-TECH/lcm-lora-sdv1-5) | Four UNet steps take about 1.74 seconds in published testing |
| Detection and surveillance | [YOLO11](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_yolo11) | Mature, fast workload with Frigate integration |

### LLM

| Model | Quantization | Published speed | 8 GB assessment |
|---|---:|---:|---|
| [Qwen3-0.6B](https://huggingface.co/AXERA-TECH/Qwen3-0.6B-GPTQ-Int4) | Int4 | 20.21 tok/s | Excellent for commands and classification |
| [Qwen3-1.7B](https://huggingface.co/AXERA-TECH/Qwen3-1.7B-GPTQ-Int4) | Int4 | 12.72 tok/s | Best speed balance |
| [Qwen3-4B](https://huggingface.co/AXERA-TECH/Qwen3-4B-GPTQ-Int4) | Int4 | 6.50 tok/s | Works with controlled context |
| [Qwen3.5-0.8B](https://huggingface.co/AXERA-TECH/Qwen3.5-0.8B-GPTQ-Int4) | Int4 | 23.8 tok/s | Excellent low-latency option |
| [Qwen3.5-2B](https://huggingface.co/AXERA-TECH/Qwen3.5-2B-GPTQ-Int4) | Int4 | 13.4 tok/s | Recommended general balance |
| [Qwen3.5-4B](https://huggingface.co/AXERA-TECH/Qwen3.5-4B-GPTQ-Int4) | Int4 | 5.7 tok/s | Best practical quality |
| [Qwen2.5-1.5B-Instruct](https://huggingface.co/AXERA-TECH/Qwen2.5-1.5B-Instruct) | Int4/Int8 | Build-dependent | Mature multilingual fallback |
| [DeepSeek-R1-Distill-Qwen-1.5B](https://huggingface.co/AXERA-TECH/DeepSeek-R1-Distill-Qwen-1.5B) | Int4 | Build-dependent | Compact reasoning, verbose output |
| [MiniCPM4-0.5B](https://huggingface.co/AXERA-TECH/MiniCPM4-0.5B) | Optimized | Build-dependent | Very light edge agent |

Choose a model from the current [AXERA model catalogue](https://github.com/AXERA-TECH/awesome-docs) and confirm that its card explicitly mentions AX650, AX8850, AXCL, and aarch64. Start with Qwen3-1.7B Int4, then compare Qwen3.5-2B or 4B on representative prompts. For voice assistants and automation, 0.6B-2B is usually preferable because latency matters more than a small quality gain. Keep RAG documents and the embedding index on the Pi and send only assembled context to the NPU. Limit context because the KV cache grows with conversation history.

### VLM, OCR, and video

Useful ports include [Qwen3-VL-2B](https://huggingface.co/AXERA-TECH/Qwen3-VL-2B-Instruct), Qwen3.5-0.8B/2B/4B, [Qwen3-VL-4B](https://huggingface.co/AXERA-TECH/Qwen3-VL-4B-Instruct-GPTQ-Int4), [Qwen2.5-VL-3B](https://huggingface.co/AXERA-TECH/Qwen2.5-VL-3B-Instruct), [InternVL3-1B](https://huggingface.co/AXERA-TECH/InternVL3-1B), [FastVLM-0.5B](https://huggingface.co/AXERA-TECH/FastVLM-0.5B), and [SmolVLM2-500M](https://huggingface.co/AXERA-TECH/SmolVLM2-500M-Video-Instruct). [LibCLIP](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_clip) is often more useful than a VLM for similar-image search in Immich or catalogues.

Do not run a heavy VLM on every video frame. Prefer hardware decode -> YOLO/tracker -> interesting-frame selection -> VLM. This reduces PCIe traffic and gives usable latency.

### Speech

[ax_asr_api](https://github.com/AXERA-TECH/ax_asr_api) provides Whisper Tiny, Base, Small, Turbo, and SenseVoice. [whisper.axcl](https://github.com/ml-inory/whisper.axcl) is an independent community port. M5Stack publishes [Tiny](https://huggingface.co/M5Stack/whisper-tiny-axmodel), [Base](https://huggingface.co/M5Stack/whisper-base-axmodel), and [Small](https://huggingface.co/M5Stack/whisper-small-axmodel) builds.

For Russian audio, explicitly set `language=ru`, normalize to 16 kHz mono PCM, and use VAD with overlapping segments for long recordings. Whisper operates in 30-second windows, so long files need an outer pipeline with timestamps and text merging. SenseVoice is fast but its official language set does not include Russian.

For TTS and speaker work, see [CosyVoice2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_cosy_voice2), [MeloTTS](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_melotts), and [3D-Speaker-MT](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_3d_speaker_mt). Verify Russian voice support for the exact checkpoint.

### Image generation

[LCM-LoRA SD 1.5](https://huggingface.co/AXERA-TECH/lcm-lora-sdv1-5) is the most practical image generator for this card. Published timings are about 3-4.5 seconds for the text encoder, 14-23 seconds for loading, 433 ms per UNet step, 1.74 seconds for four steps, and 914 ms for VAE decode after warmup.

```bash
git clone https://huggingface.co/AXERA-TECH/lcm-lora-sdv1-5
cd lcm-lora-sdv1-5
python -m venv sd
source sd/bin/activate
sudo apt install -y cmake
pip install -r requirements.txt
pip install https://github.com/AXERA-TECH/pyaxengine/releases/download/0.1.3.rc2/axengine-0.1.3-py3-none-any.whl
python run_txt2img_axe_infer.py
```

[SD1.5-LLM8850](https://huggingface.co/M5Stack/SD1.5-LLM8850) includes slower 10/20-step backends and a Web UI. LivePortrait is also available. There are no comparably ready SDXL, FLUX, or diffusion-video ports for an 8 GB AX8850 in the referenced ecosystem.

### Computer vision

The mature CV workloads include YOLOv5/v8/11, [YOLO-World-V2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_yolo_world_v2), face detection, pose, segmentation, OCR, classification, [Depth Anything V2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_depth_anything_v2), [MixFormer-V2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_mixformer_v2), Real-ESRGAN, SuperResolution, RIFE, [Frigate](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_frigate), and [Immich](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_immich).

For camera pipelines, decode video with the hardware block and keep resize, color conversion, and inference on the device. Return boxes, classes, embeddings, or a final frame to the Pi instead of intermediate tensors.

## Porting your own model

### Standard ONNX model

```text
PyTorch/TensorFlow
	-> ONNX export with fixed shapes
	-> onnxsim/onnxslim and ONNX Runtime validation
	-> representative calibration dataset
	-> Pulsar2, target_hardware: AX650
	-> .axmodel
	-> axcl_run_model
	-> AXCL/pyaxcl integration
```

Use the [Pulsar2 documentation](https://pulsar2-docs.readthedocs.io/en/latest/) and its [operator support list](https://pulsar2-docs.readthedocs.io/en/latest/appendix/op_support_list.html). The practical sequence is:

1. Export inference-only ONNX with fixed shapes where possible.
2. Validate PyTorch and ONNX Runtime numerically.
3. Simplify with `onnxslim` and validate again.
4. Check every operator against Pulsar2; replace unsupported layers or move them to CPU.
5. Build a representative calibration dataset using production preprocessing.
6. Compile a small INT8 model first, using mixed precision only where supported and justified.
7. Run the `.axmodel` with `axcl_run_model` and compare source, ONNX, and NPU metrics.
8. Add application preprocessing and postprocessing only after model accuracy is validated.

For AX8850, configurations and artifacts normally use `AX650`. Do not invent an `AX8850` target when the current Pulsar2 documentation specifies `AX650`.

### LLM and VLM

Transformer porting is substantially harder. It requires a supported `ax-llm` architecture and configuration, prefill/decode layer splitting, tokenizer and chat template compatibility, often GPTQ Int4 conversion, correct KV-cache shapes, context limits, host-side sampling, and multimodal encoder code. Validate perplexity, long conversations, stopping tokens, and tool calls.

For a fine-tuned model, start from a supported base such as Qwen3/Qwen3.5, train LoRA/QLoRA on a conventional GPU machine, merge the adapter, and repeat the official conversion pipeline. AX8850 is an inference accelerator, not a practical training target. See [Pulsar2 LLM Build](https://pulsar2-docs.readthedocs.io/en/latest/appendix/build_llm.html).

Do not start conversion when the architecture has no supported example, uses custom operations or dynamic control flow, changes frequently, or nearly fills 8 GB before runtime buffers. First establish a baseline with a ready Qwen, Whisper, or YOLO port.

## Community and primary projects

| Project | Role |
|---|---|
| [AXERA-TECH/ax-llm](https://github.com/AXERA-TECH/ax-llm) | LLM/VLM runtime and API |
| [AXERA-TECH/awesome-docs](https://github.com/AXERA-TECH/awesome-docs) | Model catalogue and benchmarks |
| [AXERA-TECH/axcl-samples](https://github.com/AXERA-TECH/axcl-samples) | PCIe C/C++ samples |
| [AXERA-TECH/pyaxcl](https://github.com/AXERA-TECH/pyaxcl) | Python bindings |
| [AXERA-TECH/pyaxengine](https://github.com/AXERA-TECH/pyaxengine) | Python execution provider |
| [AXERA-TECH/ax_asr_api](https://github.com/AXERA-TECH/ax_asr_api) | Whisper and SenseVoice API |
| [ml-inory/whisper.axcl](https://github.com/ml-inory/whisper.axcl) | Independent Whisper port |
| [AXERA-TECH/axcl-pi5-examples](https://github.com/AXERA-TECH/axcl-pi5-examples) | Raspberry Pi 5 examples |
| [M5Stack Hugging Face](https://huggingface.co/M5Stack) | Ready LLM-8850 artifacts |
| [AXERA-TECH Hugging Face](https://huggingface.co/AXERA-TECH) | Broad model collection |

## Known limits and traps

1. **Do not use Gemma 4 on LLM-8850 through PCIe.** `ax-llm` documents an AXCL correctness issue for Gemma 4.
2. **AX650N and AX650 names are normal.** AX8850 may appear as AX650N and compile with target AX650.
3. **Match versions.** Models can require updated AXCL, firmware, `ax-llm`, and Pulsar2.
4. **8 GB is not 8 GB available to a model.** A published `axcl-smi` sample shows about 7040 MiB of available CMM.
5. **Context is expensive.** Reduce maximum length and concurrency before deciding that a model does not fit.
6. **PCIe 2.0 x2 is a real limit.** Keep the pipeline on the card and minimize host-device copies.
7. **Warm persistent processes.** Loading multiple `.axmodel` files can take seconds.
8. **Check every model card.** A repository may target a different AX chip or a local SoC rather than AXCL PCIe.
9. **Measure the whole application.** Separate load, prefill, decode, preprocessing, PCIe copy, postprocessing, and request latency.
10. **Protect local APIs.** Use real authentication, TLS, a reverse proxy, and rate limiting before network exposure.

## Recommended first-run plan

1. Install the card with the correct power supply and active cooling.
2. Update EEPROM and confirm `lspci` reports Device 0650.
3. Install current `axclhost` and verify the card with `axcl-smi`.
4. Run `yolo11s.axmodel` with `axcl_run_model` and record temperature and timing.
5. Install Qwen3-1.7B Int4; test the CLI and then the OpenAI API.
6. Compare Qwen3.5-2B and 4B on representative prompts while monitoring CMM and context.
7. Run Whisper Small on clean and noisy Russian audio and measure real-time factor and WER.
8. Build a YOLO -> tracker -> VLM camera pipeline instead of running a VLM on every frame.
9. Start custom model conversion only after establishing a ready-model baseline.

Record results in a table like this:

| Model | AXCL/runtime | Context/input | Load | Prefill | Decode/FPS/RTF | CMM peak | Temperature | Quality |
|---|---|---:|---:|---:|---:|---:|---:|---|
| | | | | | | | | |

## Main links

- [M5Stack LLM-8850 overview](https://docs.m5stack.com/en/guide/ai_accelerator/overview)
- [M5Stack hardware installation](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_hardware_install)
- [M5Stack software installation](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_software_install)
- [M5Stack quick start](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_quick_start)
- [AXCL documentation](https://axcl-docs.readthedocs.io/en/latest/)
- [AXERA ax-llm](https://github.com/AXERA-TECH/ax-llm)
- [AXERA awesome-docs](https://github.com/AXERA-TECH/awesome-docs)
- [AXERA axcl-samples](https://github.com/AXERA-TECH/axcl-samples)
- [AXERA pyaxcl](https://github.com/AXERA-TECH/pyaxcl)
- [Pulsar2 documentation](https://pulsar2-docs.readthedocs.io/en/latest/)
- [Pulsar2 docs repository](https://github.com/AXERA-TECH/pulsar2-docs-en)
- [AXERA Hugging Face](https://huggingface.co/AXERA-TECH)
- [M5Stack Hugging Face](https://huggingface.co/M5Stack)

## Final assessment

LLM-8850 is most effective as a self-contained edge accelerator for well-defined workloads: local LLMs up to 4B Int4, compact VLMs, Whisper, fast CV pipelines, TTS, and SD 1.5/LCM. It is less flexible than a GPU because model choice is narrower, porting is harder, and vendor toolchain versions are interdependent. In supported workloads, it gives Raspberry Pi 5 an inference capability unavailable on CPU alone in a compact, moderate-power form factor.

The practical strategy is to build around ready Qwen3/Qwen3.5, Whisper, and YOLO ports, measure the entire pipeline, and begin custom conversion only where it delivers a proven quality, latency, or specialization advantage.
