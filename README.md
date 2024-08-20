# Flux FP8 (true) Matmul Implementation with FastAPI

This repository contains an implementation of the Flux model, along with an API that allows you to generate images based on text prompts. And also a simple single line of code to use the generator as a single object, similar to diffusers pipelines.

## Speed Comparison

Note:

-   The "bfl codebase" refers to the original [BFL codebase](https://github.com/black-forest-labs/flux), not this repo.
-   The "fp8 wo quant" refers to the original BFL codebase using fp8 weight only quantization, not using fp8 matmul which is default in this repo.
-   The "compile blocks & extras" refers to the option within this repo setting the config values `"compile_blocks" true` & `"compile_extras": true`. ❌ means both were set to false, ✅ means both were set to true.
-   All generations which including a ❌ or ✅ are using this repo.

| Resolution | Device     | Test                       | Average it/s |
| ---------- | ---------- | -------------------------- | ------------ |
| 1024x1024  | RTX4090    | bfl codebase fp8 wo quant  | 1.7          |
| 1024x1024  | RTX4090    | ❌ compile blocks & extras | 2.55         |
| 1024x1024  | RTX4090    | ✅ compile blocks & extras | 3.51         |
| 1024x1024  | RTX6000ADA | bfl codebase               | 1.74         |
| 1024x1024  | RTX6000ADA | ❌ compile blocks & extras | 2.08         |
| 1024x1024  | RTX6000ADA | ✅ compile blocks & extras | 2.8          |
| 768x768    | RTX4090    | bfl codebase fp8 wo quant  | 2.32         |
| 768x768    | RTX4090    | ❌ compile blocks & extras | 4.47         |
| 768x768    | RTX4090    | ✅ compile blocks & extras | 6.2          |
| 768x768    | RTX6000ADA | bfl codebase               | 3.01         |
| 768x768    | RTX6000ADA | ❌ compile blocks & extras | 3.43         |
| 768x768    | RTX6000ADA | ✅ compile blocks & extras | 4.46         |
| 1024x720   | RTX4090    | bfl codebase fp8 wo quant  | 3.01         |
| 1024x720   | RTX4090    | ❌ compile blocks & extras | 3.6          |
| 1024x720   | RTX4090    | ✅ compile blocks & extras | 4.96         |
| 1024x720   | RTX6000ADA | bfl codebase               | 2.37         |
| 1024x720   | RTX6000ADA | ❌ compile blocks & extras | 2.87         |
| 1024x720   | RTX6000ADA | ✅ compile blocks & extras | 3.78         |

## Table of Contents

-   [Installation](#installation)
-   [Usage](#usage)
-   [Configuration](#configuration)
-   [API Endpoints](#api-endpoints)
-   [Examples](#examples)
-   [License](#license)

## Installation

This repo _requires_ at least pytorch with cuda=12.4 and an ADA gpu with fp8 support, otherwise `torch._scaled_mm` will throw a CUDA error saying it's not supported. To install with conda/mamba:

```bash
mamba create -n flux-fp8-matmul-api python=3.11 pytorch torchvision torchaudio pytorch-cuda=12.4 -c pytorch -c nvidia
mamba activate flux-fp8-matmul-api

# or with conda
conda create -n flux-fp8-matmul-api python=3.11 pytorch torchvision torchaudio pytorch-cuda=12.4 -c pytorch -c nvidia
conda activate flux-fp8-matmul-api

# or with nightly... (which is what I am using) - also, just switch 'mamba' to 'conda' if you are using conda
mamba create -n flux-fp8-matmul-api python=3.11 pytorch torchvision torchaudio pytorch-cuda=12.4 -c pytorch-nightly -c nvidia
mamba activate flux-fp8-matmul-api

# or with pip
python -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
# or pip nightly
python -m pip install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/cu124
```

To install the required dependencies, run:

```bash
python -m pip install -r requirements.txt
```

If you get errors installing `torch-cublas-hgemm`, feel free to comment it out in requirements.txt, since it's not necessary, but will speed up inference for non-fp8 linear layers.

## Usage

For a single ADA GPU with less than 24GB vram, and more than 16GB vram, you should use the `configs/config-dev-1-4080.json` config file as a base, and then tweak the parameters to fit your needs. It offloads all models to CPU when not in use, compiles the flow model with extra optimizations, and quantizes the text encoder to nf4 and the autoencoder to qfloat8.

For a single ADA GPU with more than ~32GB vram, you should use the `configs/config-dev-1-RTX6000ADA.json` config file as a base, and then tweak the parameters to fit your needs. It does not offload any models to CPU, compiles the flow model with extra optimizations, and quantizes the text encoder to qfloat8 and the autoencoder to stays as bfloat16.

For a single 4090 GPU, you should use the `configs/config-dev-1-4090.json` config file as a base, and then tweak the parameters to fit your needs. It offloads the text encoder and the autoencoder to CPU, compiles the flow model with extra optimizations, and quantizes the text encoder to nf4 and the autoencoder to float8.

**NOTE:** For all of these configs, you must change the `ckpt_path`, `ae_path`, and `text_enc_path` parameters to the path to your own checkpoint, autoencoder, and text encoder.

You can run the API server using the following command:

```bash
python main.py --config-path <path_to_config> --port <port_number> --host <host_address>
```

### API Command-Line Arguments

-   `--config-path`: Path to the configuration file. If not provided, the model will be loaded from the command line arguments.
-   `--port`: Port to run the server on (default: 8088).
-   `--host`: Host to run the server on (default: 0.0.0.0).
-   `--flow-model-path`: Path to the flow model.
-   `--text-enc-path`: Path to the text encoder.
-   `--autoencoder-path`: Path to the autoencoder.
-   `--model-version`: Choose model version (`flux-dev` or `flux-schnell`).
-   `--flux-device`: Device to run the flow model on (default: cuda:0).
-   `--text-enc-device`: Device to run the text encoder on (default: cuda:0).
-   `--autoencoder-device`: Device to run the autoencoder on (default: cuda:0).
-   `--compile`: Compile the flow model with extra optimizations (default: False).
-   `--quant-text-enc`: Quantize the T5 text encoder to the given dtype (`qint4`, `qfloat8`, `qint2`, `qint8`, `bf16`), if `bf16`, will not quantize (default: `qfloat8`).
-   `--quant-ae`: Quantize the autoencoder with float8 linear layers, otherwise will use bfloat16 (default: False).
-   `--offload-flow`: Offload the flow model to the CPU when not being used to save memory (default: False).
-   `--no-offload-ae`: Disable offloading the autoencoder to the CPU when not being used to increase e2e inference speed (default: True [implies it will offload, setting this flag sets it to False]).
-   `--no-offload-text-enc`: Disable offloading the text encoder to the CPU when not being used to increase e2e inference speed (default: True [implies it will offload, setting this flag sets it to False]).
-   `--prequantized-flow`: Load the flow model from a prequantized checkpoint, which reduces the size of the checkpoint by about 50% & reduces startup time (default: False).

## Examples

### Running the Server

```bash
python main.py --config-path configs/config-dev-1-4090.json --port 8088 --host 0.0.0.0
```

Or if you need more granular control over the all of the settings, you can run the server with something like this:

```bash
python main.py --port 8088 --host 0.0.0.0 \
    --flow-model-path /path/to/your/flux1-dev.sft \
    --text-enc-path /path/to/your/t5-v1_1-xxl-encoder-bf16 \
    --autoencoder-path /path/to/your/ae.sft \
    --model-version flux-dev \
    --flux-device cuda:0 \
    --text-enc-device cuda:0 \
    --autoencoder-device cuda:0 \
    --compile \
    --quant-text-enc qfloat8 \
    --quant-ae
```

## Configuration

The configuration files are located in the `configs` directory. You can specify different configurations for different model versions and devices.

Example configuration file for a single 4090 (`configs/config-dev-1-4090.json`):

```js
{
    "version": "flux-dev", // or flux-schnell
    "params": {
        "in_channels": 64,
        "vec_in_dim": 768,
        "context_in_dim": 4096,
        "hidden_size": 3072,
        "mlp_ratio": 4.0,
        "num_heads": 24,
        "depth": 19,
        "depth_single_blocks": 38,
        "axes_dim": [16, 56, 56],
        "theta": 10000,
        "qkv_bias": true,
        "guidance_embed": true // if you are using flux-schnell, set this to false
    },
    "ae_params": {
        "resolution": 256,
        "in_channels": 3,
        "ch": 128,
        "out_ch": 3,
        "ch_mult": [1, 2, 4, 4],
        "num_res_blocks": 2,
        "z_channels": 16,
        "scale_factor": 0.3611,
        "shift_factor": 0.1159
    },
    "ckpt_path": "/your/path/to/flux1-dev.sft", // local path to original bf16 BFL flux checkpoint
    "ae_path": "/your/path/to/ae.sft", // local path to original bf16 BFL autoencoder checkpoint
    "repo_id": "black-forest-labs/FLUX.1-dev", // can ignore
    "repo_flow": "flux1-dev.sft", // can ignore
    "repo_ae": "ae.sft", // can ignore
    "text_enc_max_length": 512, // use 256 if you are using flux-schnell
    "text_enc_path": "city96/t5-v1_1-xxl-encoder-bf16", // or custom HF full bf16 T5EncoderModel repo id
    "text_enc_device": "cuda:0",
    "ae_device": "cuda:0",
    "flux_device": "cuda:0",
    "flow_dtype": "float16",
    "ae_dtype": "bfloat16",
    "text_enc_dtype": "bfloat16",
    "flow_quantization_dtype": "qfloat8", // will always be qfloat8, so can ignore
    "text_enc_quantization_dtype": "qint4", // choose between qint4, qint8, qfloat8, qint2 or delete entry for no quantization
    "ae_quantization_dtype": "qfloat8", // can either be qfloat8 or delete entry for no quantization
    "compile_extras": true, // compile the layers not included in the single-blocks or double-blocks
    "compile_blocks": true, // compile the single-blocks and double-blocks
    "offload_text_encoder": true, // offload the text encoder to cpu when not in use
    "offload_vae": true, // offload the autoencoder to cpu when not in use
    "offload_flow": false // offload the flow transformer to cpu when not in use
}
```

The only things you should need to change in general are the:

```json5
    "ckpt_path": "/path/to/your/flux1-dev.sft", // path to your original BFL flow transformer (not diffusers)
    "ae_path": "/path/to/your/ae.sft", // path to your original BFL autoencoder (not diffusers)
    "text_enc_path": "path/to/your/t5-v1_1-xxl-encoder-bf16", // HF T5EncoderModel - can use "city96/t5-v1_1-xxl-encoder-bf16" for a simple to download version
```

Other things to change can be the

-   `"text_enc_max_length": 512`
    max length for the text encoder, 256 if you are using flux-schnell

-   `"ae_quantization_dtype": "qfloat8"`
    quantization dtype for the autoencoder, can be `qfloat8` or delete entry for no quantization, will use the float8 linear layer implementation included in this repo.

-   `"text_enc_quantization_dtype": "qfloat8"`
    quantization dtype for the text encoder, if `qfloat8` or `qint2` will use quanto, `qint4`, `qint8` will use bitsandbytes

-   `"compile_extras": true,`
    compiles all modules that are not the single-blocks or double-blocks (default: false)

-   `"compile_blocks": true,`
    compiles all single-blocks and double-blocks (default: false)

-   `"text_enc_offload": false,`
    offload text encoder to cpu (default: false) - set to true if you only have a single 4090 and no other GPUs, otherwise you can set this to false and reduce latency [NOTE: this will be slow, if you have multiple GPUs, change the text_enc_device to a different device so you can set offloading for text_enc to false]

-   `"ae_offload": false,`
    offload autoencoder to cpu (default: false) - set to true if you only have a single 4090 and no other GPUs, otherwise you can set this to false and reduce latency [NOTE: this will be slow, if you have multiple GPUs, change the ae_device to a different device so you can set offloading for ae to false]

-   `"flux_offload": false,`
    offload flow transformer to cpu (default: false) - set to true if you only have a single 4090 and no other GPUs, otherwise you can set this to false and reduce latency [NOTE: this will be slow, if you have multiple GPUs, change the flux_device to a different device so you can set offloading for flux to false]

-   `"flux_device": "cuda:0",`
    device for flow transformer (default: cuda:0) - this gpu must have fp8 support and at least 16GB of memory, does not need to be the same as text_enc_device or ae_device

-   `"text_enc_device": "cuda:0",`
    device for text encoder (default: cuda:0) - set this to a different device - e.g. `"cuda:1"` if you have multiple gpus so you can set offloading for text_enc to false, does not need to be the same as flux_device or ae_device

-   `"ae_device": "cuda:0",`
    device for autoencoder (default: cuda:0) - set this to a different device - e.g. `"cuda:1"` if you have multiple gpus so you can set offloading for ae to false, does not need to be the same as flux_device or text_enc_device

## API Endpoints

### Generate Image

-   **URL**: `/generate`
-   **Method**: `POST`
-   **Request Body**:

    -   `prompt` (str): The text prompt for image generation.
    -   `width` (int, optional): The width of the generated image (default: 720).
    -   `height` (int, optional): The height of the generated image (default: 1024).
    -   `num_steps` (int, optional): The number of steps for the generation process (default: 24).
    -   `guidance` (float, optional): The guidance scale for the generation process (default: 3.5).
    -   `seed` (int, optional): The seed for random number generation.
    -   `init_image` (str, optional): The base64 encoded image to be used as a reference for the generation process.
    -   `strength` (float, optional): The strength of the diffusion process when image is provided (default: 1.0).

-   **Response**: A JPEG image stream.

## Examples

### Running the Server

```bash
python main.py --config-path configs/config-dev.json --port 8088 --host 0.0.0.0
```

OR, if you need more granular control over the server, you can run the server with something like this:

```bash
python main.py --port 8088 --host 0.0.0.0 \
    --flow-model-path /path/to/your/flux1-dev.sft \
    --text-enc-path /path/to/your/t5-v1_1-xxl-encoder-bf16 \
    --autoencoder-path /path/to/your/ae.sft \
    --model-version flux-dev \
    --flux-device cuda:0 \
    --text-enc-device cuda:0 \
    --autoencoder-device cuda:0 \
    --compile \
    --quant-text-enc qfloat8 \
    --quant-ae
```

### Generating an Image

Send a POST request to `http://<host>:<port>/generate` with the following JSON body:

```json
{
    "prompt": "a beautiful asian woman in traditional clothing with golden hairpin and blue eyes, wearing a red kimono with dragon patterns",
    "width": 1024,
    "height": 1024,
    "num_steps": 24,
    "guidance": 3.0,
    "seed": 13456
}
```

For an example of how to generate from a python client using the FastAPI server:

```py
import requests
import io

prompt = "a beautiful asian woman in traditional clothing with golden hairpin and blue eyes, wearing a red kimono with dragon patterns"
res = requests.post(
    "http://localhost:8088/generate",
    json={
        "width": 1024,
        "height": 720,
        "num_steps": 20,
        "guidance": 4,
        "prompt": prompt,
    },
    stream=True,
)

with open(f"output.jpg", "wb") as f:
    f.write(io.BytesIO(res.content).read())

```

You can also generate an image by directly importing the FluxPipeline class and using it to generate an image. This is useful if you have a custom model configuration and want to generate an image without having to run the server.

```py
import io
from flux_pipeline import FluxPipeline


pipe = FluxPipeline.load_pipeline_from_config_path(
    "configs/config-dev-1-4090.json"  # or whatever your config is
)

output_jpeg_bytes: io.BytesIO = pipe.generate(
    # Required args:
    prompt="A beautiful asian woman in traditional clothing with golden hairpin and blue eyes, wearing a red kimono with dragon patterns",
    # Optional args:
    width=1024,
    height=1024,
    num_inference_steps=20,
    guidance_scale=3.5,
    seed=13456,
    init_image="path/to/your/init_image.jpg",
    strength=0.8,
)

with open("output.jpg", "wb") as f:
    f.write(output_jpeg_bytes.getvalue())

```
