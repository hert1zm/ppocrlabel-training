English | [Italiano](README_it.md)

<div align="center">
  <img src="imgs/demo.png" width="70%"/>
</div>

# PPOCRLabel and PaddleOCR Guide for Training Custom OCR Models

This guide explains how to build a complete workflow for training a custom OCR model with PPOCRLabel and PaddleOCR, starting from image annotation and ending with an exported inference model ready for deployment.

It is mainly written for Linux environments, but the same structure also applies to Windows with minor path adjustments.

## Scope

PPOCRLabel is a semi-automatic annotation tool for OCR tasks. It supports text box annotation, re-recognition, table annotation, irregular text annotation, and direct export of files that can be used for PP-OCR detection and recognition training.

PaddleOCR's documented customization flow is organized into three stages: training a text detection model, training a text recognition model, and combining the trained components into an OCR pipeline for inference.

This document focuses primarily on the text recognition training workflow because the collected commands are centered on `rec` model configuration and export, while also showing where detection training fits in the overall architecture.

## Environment Setup

You can use Conda for environment management on all platforms, but it is especially useful on Windows. On Linux, it is also common to use a plain `python`/`pip` or Docker‑based workflow instead of Conda.

#### Linux (without Conda, optional)

On Linux, you can skip Conda entirely and work with system Python or a virtualenv:

```bash
cd /ocr
python3 -m venv venv
source venv/bin/activate
```

#### Windows (recommended with Conda)

```bash
cd \ocr
conda create -n paddle_env python=3.8
conda activate paddle_env
```
---
Install the PaddlePaddle runtime that matches the target hardware. CPU‑only environments should use the CPU package, while NVIDIA systems with a supported CUDA stack can use the GPU package. (See [here](https://www.paddlepaddle.org.cn/en/install/quick))

```bash
# CPU
python -m pip install paddlepaddle==3.2.0 -i https://www.paddlepaddle.org.cn/packages/stable/cpu/

# GPU
python -m pip install paddlepaddle-gpu==3.2.0 -i https://www.paddlepaddle.org.cn/packages/stable/cu126/
```

Clone PaddleOCR and install its dependencies.

```bash
git clone https://github.com/PaddlePaddle/PaddleOCR.git
cd PaddleOCR
pip install -r requirements.txt
```

Clone PPOCRLabel and install it in editable mode for easier local customization. The project README also documents both wheel‑based and Python script‑based launch modes.

```bash
cd /ocr
git clone https://github.com/PFCCLab/PPOCRLabel.git
cd PPOCRLabel
pip install -e .
pip install --upgrade "paddlex[ocr]"
pip install premailer
pip install pywin32
```

## Launching PPOCRLabel

PPOCRLabel starts in Chinese by default, and the repository explicitly states that English UI mode requires the `--lang en` flag.

On Windows, activate the Conda virtual environment and navigate to the PPOCRLabel directory from the Conda terminal. 
```bash
conda activate paddle_env
cd ...\PPOCRLabel
python PPOCRLabel.py --lang en
```

If the tool is installed as a package, it can also be launched directly as `PPOCRLabel`, but running `PPOCRLabel.py` is the better option when local modifications are needed.

## Annotation Workflow

After launch, open the image directory from the `File` menu. PPOCRLabel treats the selected folder as the project and imports the images directly into the application view.

A practical recognition‑labeling workflow is:

1. Open the source image directory.
2. Run **Auto recognition** to pre‑annotate images that are still marked as unchecked.
3. Use `W` for rectangular boxes or `Q` for four‑point boxes when manual adjustments are needed.
4. Edit recognition text manually in the recognition results panel when predictions are incorrect.
5. Press **Check** to confirm the image and move to the next one.
6. Export recognition results to generate cropped text images and the `rec_gt.txt` annotation file.

The PPOCRLabel repository documents two key training outputs generated in the dataset folder: `Label.txt` for detection training and `rec_gt.txt` plus `crop_img/` for recognition training.

## Dataset Outputs from PPOCRLabel

The repository describes the purpose of the exported files as follows.

| File           | Purpose                                                                 |
|----------------|-------------------------------------------------------------------------|
| `Label.txt`    | Detection label file for PP‑OCR detection model training.              |
| `rec_gt.txt`   | Recognition label file generated through **File → Export Recognition Results**. |
| `crop_img/`    | Cropped text image dataset generated together with `rec_gt.txt` for recognition training. |
| `fileState.txt`| Tracks whether an image has been manually confirmed.                   |
| `Cache.cach`   | Recognition cache used by the application.                             |

For a custom recognizer, `rec_gt.txt` and `crop_img/` are the core outputs used to prepare the final train and validation splits.

## Splitting Train, Validation, and Test Sets

The PPOCRLabel repository includes a dataset split script named `gen_ocr_train_val_test.py` and documents the `trainValTestRatio` and `datasetRootPath` parameters.

```bash
python gen_ocr_train_val_test.py --trainValTestRatio 8:2:0 --datasetRootPath /progetto_ocr/PaddleOCR/train_data/images
```

The upstream documentation shows a typical usage pattern such as `6:2:2`, and the same mechanism can be adapted to `8:2:0` when a dedicated test split is not required at the initial experimentation stage.

In practice, make sure the dataset root contains the exported recognition artifacts, especially `rec_gt.txt` and `crop_img/`, because those are the standard PPOCRLabel outputs intended for recognition model training.

## Project Structure Recommendation

A clean layout for a recognition training project is shown below.

```text
ocr/
├── PaddleOCR/
│   ├── configs/
│   ├── pretrain_models/
│   ├── tools/
│   ├── train_data/
│   │   ├── crop_img/
│   │   ├── rec_gt.txt
│   │   ├── train.txt
│   │   ├── val.txt
│   │   └── my_rec.yml
│   └── output/
└── PPOCRLabel/
```

This keeps PaddleOCR configuration, pretrained weights, dataset manifests, and training outputs in one repository‑local structure that is easy to maintain.

## Choosing Recognition vs Detection Training

PaddleOCR documents custom OCR training as a two‑model pipeline: text detection finds text regions, and text recognition decodes the cropped text content.

If the images are already cropped per word, line, plate, or token, recognition‑only training is usually enough. If the final application must locate text inside full documents or natural images, a detection model is also required.

The commands collected for this project point to `configs/rec/...` and `tools/export_model.py` for a recognition model, so the rest of this guide concentrates on the recognizer workflow.

## Preparing a Recognition YAML Configuration

A common approach is to start from an official recognition YAML file and customize it for the project dataset.

Example base file:

```bash
cd /ocr/PaddleOCR
cp configs/rec/PP-OCRv5/PP-OCRv5_server_rec.yml train_data/my_rec.yml
```

### YAML Sections That Matter Most

The most important sections to edit in a custom recognition configuration are listed below.

| Section       | What to set                                                               | Why it matters                                                       |
|--------------|---------------------------------------------------------------------------|---------------------------------------------------------------------|
| `Global`     | `epoch_num`, `save_model_dir`, `pretrained_model`, `use_gpu`             | Controls runtime, checkpoint location, initialization, and hardware use. |
| `Train.dataset` | `data_dir`, `label_file_list`                                          | Points training to the cropped text images and training manifest.   |
| `Eval.dataset`  | `data_dir`, `label_file_list`                                          | Points evaluation to the validation manifest.                       |
| `PostProcess`   | Decoder type and dictionary path if needed                              | Must match the label space and model head.                          |
| `Architecture`  | Recognition architecture and backbone                                   | Defines the recognizer capacity and speed trade‑off.                |
| `Optimizer`     | Learning rate and regularization                                        | Strongly affects convergence behavior.                              |

Below is a practical example template for a custom recognition model.

```yaml
Global:
  use_gpu: true
  epoch_num: 200
  log_smooth_window: 20
  print_batch_step: 10
  save_model_dir: ./output/rec_custom_v5
  save_epoch_step: 5
  eval_batch_step: 
  pretrained_model: ./pretrain_models/ch_PP-OCRv4_rec_server_train/best_accuracy
  cal_metric_during_train: true

Architecture:
  model_type: rec
  algorithm: CRNN
  Transform:
  Backbone:
    name: MobileNetV3
    scale: 0.5
    model_name: large
  Neck:
    name: SequenceEncoder
    encoder_type: rnn
    hidden_size: 48
  Head:
    name: CTCHead
    fc_decay: 0.00001

Loss:
  name: CTCLoss

Optimizer:
  name: Adam
  beta1: 0.9
  beta2: 0.999
  lr:
    name: Cosine
    learning_rate: 0.001
  regularizer:
    name: L2
    factor: 0.00004

PostProcess:
  name: CTCLabelDecode
  character_dict_path: ./ppocr/utils/en_dict.txt
  use_space_char: true

Metric:
  name: RecMetric
  main_indicator: acc

Train:
  dataset:
    name: SimpleDataSet
    data_dir: ./train_data/crop_img
    label_file_list:
      - ./train_data/train.txt
    transforms:
      - DecodeImage:
          img_mode: BGR
          channel_first: false
      - CTCLabelEncode:
      - RecResizeImg:
          image_shape:[1]
      - KeepKeys:
          keep_keys: [image, label, length]
  loader:
    shuffle: true
    batch_size_per_card: 64
    drop_last: false
    num_workers: 4

Eval:
  dataset:
    name: SimpleDataSet
    data_dir: ./train_data/crop_img
    label_file_list:
      - ./train_data/val.txt
    transforms:
      - DecodeImage:
          img_mode: BGR
          channel_first: false
      - CTCLabelEncode:
      - RecResizeImg:
          image_shape:[1]
      - KeepKeys:
          keep_keys: [image, label, length]
  loader:
    shuffle: false
    drop_last: false
    num_workers: 4
```

### Key YAML Customization Notes

Use `data_dir: ./train_data/crop_img` when training from PPOCRLabel recognition exports, because `crop_img/` is the directory generated for recognition samples by the tool.

Use `label_file_list` to point to text manifests such as `train.txt` and `val.txt` created from the split step. Those manifest files should map each cropped image path to its transcription label.

Set `character_dict_path` to a dictionary that matches the target character set. For English‑only projects, an English dictionary is typically appropriate, while mixed alphanumeric or domain‑specific symbols may require a custom dictionary file.

Keep `PostProcess`, `Loss`, and the model head aligned. For example, `CTCLabelDecode`, `CTCLoss`, and `CTCHead` are intended to work together in a CTC‑based recognizer.

### When to Use a Custom Character Dictionary

A custom dictionary is recommended when the target domain uses only a narrow symbol set, such as digits, uppercase product codes, serial numbers, license plates, or constrained industrial labels.

Reducing the label space can improve convergence and lower confusion between visually similar characters, provided the dictionary fully covers the real production data.

A simple custom dictionary file can contain one symbol per line, for example:

When a custom dictionary is introduced, the YAML must reference that file through `character_dict_path`, and the training labels in `train.txt` and `val.txt` must only contain symbols included in the dictionary.

## Training Commands

The collected commands include direct training with the official PP‑OCRv5 recognition configuration.

```bash
python ./tools/train.py -c ./configs/rec/PP-OCRv5/PP-OCRv5_server_rec.yml
```

For a reproducible project, train against the customized YAML instead.

```bash
cd /ocr/PaddleOCR
python tools/train.py -c train_data/my_rec.yml
```

The alternative command set also initializes training from a pretrained recognition checkpoint using the `-o Global.pretrained_model=...` override.

```bash
python tools/train.py -c train_data/my_rec.yml -o Global.pretrained_model=pretrain_models/ch_PP-OCRv4_rec_server_train/best_accuracy
```

This is the recommended pattern for fine‑tuning because it reuses a pretrained recognizer and usually converges faster than training from scratch.

## Monitoring Outputs

During training, PaddleOCR writes checkpoints under the directory configured in `Global.save_model_dir`. The practical checkpoints to watch are typically `latest` and the best‑performing checkpoint, often named `best_accuracy`.

A common output layout looks like this:

```text
output/
└── rec_custom_v5/
    ├── best_accuracy/
    ├── latest/
    ├── train.log
    └── vdl/
```

If validation accuracy stabilizes early, the first variables to revisit are image normalization settings, dictionary correctness, label quality, batch size, and learning rate schedule.

## Exporting the Inference Model

After training, use `tools/export_model.py` to convert a training checkpoint into an inference‑ready model directory.

The collected commands already follow the correct pattern: pass the YAML, point `Global.pretrained_model` to the saved checkpoint, and set `Global.save_inference_dir` to the export directory.

Example based on the project‑specific YAML:

```bash
python tools/export_model.py -c train_data/my_rec.yml -o Global.pretrained_model=./output/rec_custom_v5/best_accuracy Global.save_inference_dir=./output/rec_custom_v5_infer
```

Example based on the PP‑OCRv5 server recognition configuration:

```bash
python3 tools/export_model.py -c configs\rec\PP-OCRv5\PP-OCRv5_server_rec.yml -o Global.pretrained_model=./output/PP-OCRv5_server_rec/latest Global.save_inference_dir=./output/PP-OCRv5_server_rec_infer
```

## Recognition Manifest Format

For recognition training, files such as `train.txt` and `val.txt` are typically plain text files mapping each cropped image to its label.

A common format is:

```text
crop_img/word_001_crop_0.jpg\tInvoice
crop_img/word_002_crop_0.jpg\tA1024
crop_img/word_003_crop_0.jpg\t59.90
```

The exact path style can be relative or absolute as long as it is consistent with the YAML `data_dir` and the data loader's expectations.

## Detection Training Extension

If the final application must detect text in full images, the same project can be extended by training a detection model using the exported `Label.txt` file from PPOCRLabel, because the repository states that `Label.txt` is directly intended for PP‑OCR detection model training.

PaddleOCR's customization guide frames this as the first step of a full OCR system, followed by recognition training and then concatenation of predictions from both models.

In other words, PPOCRLabel can feed both branches of the OCR pipeline: detection through `Label.txt` and recognition through `rec_gt.txt` plus `crop_img/`.
