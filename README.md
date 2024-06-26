# <img src="https://github.com/A-suozhang/MyPicBed/raw/master/img/20240604133532.png" alt="drawing" width="30"/> MixDQ

### [📖 Paper](https://arxiv.org/abs/2405.17873) | [🖥️ Project Page](https://a-suozhang.xyz/mixdq.github.io/) | [🤗 Huggingface Pipeline](https://huggingface.co/nics-efc/MixDQ)

### News

- [24/07] MixDQ is accepted by [ECCV2024](https://eccv2024.ecva.net/).
- [24/05] We release the MixDQ hardware acceleration code (with INT8 GPU kernel) at [https://huggingface.co/nics-efc/MixDQ](https://huggingface.co/nics-efc/MixDQ).
- [24/05] We release the MixDQ algorithm-level quantization simulation code at [https://github.com/A-suozhang/MixDQ](https://github.com/A-suozhang/MixDQ).

This repo contains the official code of our ECCV2024 paper: [MixDQ: Memory-Efficient Few-Step Text-to-Image Diffusion Models with Metric-Decoupled Mixed Precision Quantization](https://arxiv.org/abs/2405.17873)

We design MixDQ, a mixed-precision quantization framework that successfully tackles the challenging few-step text-to-image diffusion model quantization. With negligible visual quality degradation and content change, MixDQ could achieve W4A8, with equivalent 3.4x memory compression and 1.5x latency speedup.

![](https://github.com/A-suozhang/MyPicBed/raw/master/img/20240604133828.png)

🤗 Open-Source Huggingface Pipeline 🤗: We implement efficient INT8 GPU kernel to achieve actual GPU acceleration (1.45x) and memory savings (2x) for W8A8. The pipeline is released at: [https://huggingface.co/nics-efc/MixDQ](https://huggingface.co/nics-efc/MixDQ). It could be easily implemented with just a few lines of code. 

![](https://github.com/A-suozhang/MyPicBed/raw/master/img/20240604133923.png)

For more information, please refer to our [Project Page: https://a-suozhang.xyz/mixdq.github.io/](https://a-suozhang.xyz/mixdq.github.io/)

# Usage

## EnvSetup

We recommend using conda for environment management.

```
cd quant_utils
conda env create -f environment.yml
conda activate mixdq
python -m pip install --upgrade --user ortools
pip install accelerate
```

## Data Preparation

The stable diffusion checkpoints are automatically downloaded with the diffusers pipeline, we also provide manual download scripts in `./scripts/utils/download_huggingface_model.py`. For text-to-image generation on COCO annotations, we provide the `captions_val2014.json` with [Google Drive](https://drive.google.com/file/d/1-BKxclA-5jD1vUDMQSFrkquyM73BTlmi/view?usp=sharing), please put it in the `./scripts/utils`. 

## 0. FP text-to-image generation

Run the `main_fp_infer.sh` to generate images based on coco annotation or given prompt. (When deleting `--prompt xxx`, using coco annotations as the default prompts.) The images could be found in `$BASE_PATH/generated_images`. 

```
## SDXL_Turbo FP Inference
config_name='sdxl_turbo.yaml' # quant config, but only model names are used
BASE_PATH='./logs/debug_fp'   # save image path

CUDA_VISIBLE_DEVICES=$1 python scripts/txt2img.py \
		--config ./configs/stable-diffusion/$config_name \
		--base_path $BASE_PATH --batch_size 2 --num_imgs 8  --prompt  "a vanilla and chocolate mixing icecream cone, ice background" \
		--fp16
```

## 1. Normal Quantization

We provide the shell script `main.sh` for the whole quantization process. The quantization process consists of 3 steps: (1) generating the calibration data. (2) conduct PTQ process. (3) conduct quantized model inference. We also provide the scripts for each of the 3 processes (`main_calib_data.sh`,`main_ptq.sh`,`main_quant_infer.sh`). You could run the `main.sh` to finish the whole quantization process, or run three steps respectively. 

### 1.1 Generate Calibration Data

Run the `main_calib_data.sh $GPU_ID` to generate the FP activation calibration data. The output path of calib data is specified in the quant `config.yaml`. the `--save_image_path` saves the FP generated reference images. 

```
CUDA_VISIBLE_DEVICES=$1 python scripts/gen_calib_data.py --config ./configs/stable-diffusion/$config_name --save_image_path ./debug_imgs
```

### 1.2 Post Training Quantization (PTQ) Process
 
Run the `main_ptq.sh $LOG_NAME $GPU_ID` to conduct PTQ to determine quant parameters, the quant parameters are saved as `ckpt.pth` in the log path.

```
CUDA_VISIBLE_DEVICES=$2 python scripts/ptq.py --config ./configs/stable-diffusion/${cfg_name} --outdir ./logs/$1 --seed 42
```

### 1.3 Inference Quantized Model

#### 1.3.1 Normal

We provide the quantized inference example in the latter part of `main.sh`, and the `main_quant_infer.sh` (the commented part). The `--num_imgs` denotes how many images to generate, when no `--prompt` is given, the COCO annotations are used as the default prompts. By default, 1 image is generated for each prompt. When a user-defined prompt is given, "#num_imgs" of images are generated following this prompt. 

```
CUDA_VISIBLE_DEVICES=$1 python scripts/quant_txt2img.py --base_path $CKPT_PATH --batch_size 2 --num_imgs 8 
```


#### 1.3.2 Mixed Precision

For simplicity, we provide MixDQ acquired mixed precision configurations in `./mixed_precision_scripts/mixed_percision_config/sdxl_turbo/final_config/`, the example of mixed precision inference is shown in `main_quant_infer.sh`. The "act protect" represents layers that are preserved as FP16. (It's also worth noting that the `mixed_precision_scripts/quant_inference_mp.py` are used for mixed precision search, for infering the mixed precision quant model, use `scripts/quant_txt2img.py`)

```
# Mixed Precision Quant Inference
WEIGHT_MP_CFG="./mixed_precision_scripts/mixed_percision_config/sdxl_turbo/final_config/weight/weight_8.00.yaml"  # [weight_5.02.yaml, weight_8.00.yaml]
ACT_MP_CFG="./mixed_precision_scripts/mixed_percision_config/sdxl_turbo/final_config/act/act_7.77.yaml "
ACT_PROTECT="./mixed_precision_scripts/mixed_percision_config/sdxl_turbo/final_config/act/act_sensitivie_a8_1%.pt"

CUDA_VISIBLE_DEVICES=$1 python scripts/quant_txt2img.py \
  --base_path $CKPT_PATH --batch_size 2 --num_imgs 8  --prompt"a vanilla and chocolate mixing icecream cone, ice background"\
  --config_weight_mp $WEIGHT_MP_CFG \
  --config_act_mp  $ACT_MP_CFG \
  --act_protect  $ACT_PROTECT \
  --fp16
```

Using the example prompt, the generated W8A8 with mixed precision should be like:

![](https://github.com/A-suozhang/MyPicBed/raw/master/img/20240604195806.png)

## 3. Mixed Precision Search

Please download the `util_files` from [Google Drive](https://drive.google.com/file/d/1tqhRHDSbSe3UB2jdKifEG4jBQm5xmj9X/view?usp=sharing), and unzip it in the repository root directory. 
Please refer to the [./mixed_precision_scripts/mixed_precision_search.md](./mixed_precision_scripts/mixed_precision_search.md) for detailed process of the mixed precision search process. 

# Acknowledgements

Our code is developed based on [Q-Diffusion](https://github.com/Xiuyu-Li/q-diffusion), and the [Diffusers Libraray](https://huggingface.co/docs/diffusers/en/index). 

# Citation

If you find our work helpful, please consider citing:

```
@misc{zhao2024mixdq,
      title={MixDQ: Memory-Efficient Few-Step Text-to-Image Diffusion Models with Metric-Decoupled Mixed Precision Quantization}, 
      author={Tianchen Zhao and Xuefei Ning and Tongcheng Fang and Enshu Liu and Guyue Huang and Zinan Lin and Shengen Yan and Guohao Dai and Yu Wang},
      year={2024},
      eprint={2405.17873},
      archivePrefix={arXiv},
      primaryClass={cs.CV}
    }
```

# TODOs

- the evaluation scripts (FID, ClipScore, ImageReward)
- the efficient INT8 GPU kernels implementation

# Contact

If you have any questions, feel free to contact:

[Tianchen Zhao](https://www.tianchen-zhao.info/): suozhang1998@gmail.com 

[Xuefei Ning](https://www.ningxuefei.cc/):  foxdoraame@gmail.com 
