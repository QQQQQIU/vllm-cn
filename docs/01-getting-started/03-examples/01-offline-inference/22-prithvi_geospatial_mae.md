---
title: Prithvi Geospatial Mae
---

源码 [examples/offline_inference/prithvi_geospatial_mae.py](https://github.com/vllm-project/vllm/blob/main/examples/offline_inference/prithvi_geospatial_mae.py)

```python
# SPDX-License-Identifier: Apache-2.0

"""
这是一个演示脚本，显示如何使用
带有 vLLM 的 PrithviGeospatialMAE 模型
该脚本基于: https://huggingface.co/ibm-nasa-geospatial/prithvi-eo-2.0-300m-tl-sen1floods11/blob/main/main/inference.py# noqa
目标模型权重: https://huggingface.co/ibm-nasa-geospatial/prithvi-eo-2.0-300m-tl-sen1floods11/resolve/main/main/prithvi-eo-eo-eo-eo-eo-eo-eo-v2-300m-tl-sen1.pt# noqa
运行此脚本的要求是:
 - 在 Python 环境中安装 [terratorch, albumentations, rasterio]
 - 在脚本 model 文件夹中下载模型权重
 (直到将正确的 config.json 文件上传到 HF 前都将临时度量)
 - 下载输入示例图像 (India_900498_S2Hand.tif) 并将其放入
带有脚本的同一文件夹 (或用 -data_file 参数指定)
运行以下示例:
python prithvi_geospatial_mae.py
"""# noqa: E501
import argparse
import datetime
import os
import re
from typing import Union

import albumentations
import numpy as np
import rasterio
import torch
from einops import rearrange
from terratorch.datamodules import Sen1Floods11NonGeoDataModule

from vllm import LLM

NO_DATA = -9999
NO_DATA_FLOAT = 0.0001
OFFSET = 0
PERCENTILE = 99

model_config = """{
  "architectures": ["PrithviGeoSpatialMAE"],
  "num_classes": 0,
  "pretrained_cfg": {
    "task_args": {
      "task": "SemanticSegmentationTask",
      "model_factory": "EncoderDecoderFactory",
      "loss": "ce",
      "ignore_index": -1,
      "lr": 0.001,
      "freeze_backbone": false,
      "freeze_decoder": false,
      "plot_on_val": 10,
      "optimizer": "AdamW",
      "scheduler": "CosineAnnealingLR"
    },
    "model_args": {
      "backbone_pretrained": false,
      "backbone": "prithvi_eo_v2_300_tl",
      "decoder": "UperNetDecoder",
      "decoder_channels": 256,
      "decoder_scale_modules": true,
      "num_classes": 2,
      "rescale": true,
      "backbone_bands": [
        "BLUE",
        "GREEN",
        "RED",
        "NIR_NARROW",
        "SWIR_1",
        "SWIR_2"
      ],
      "head_dropout": 0.1,
      "necks": [
        {
          "name": "SelectIndices",
          "indices": [
            5,
            11,
            17,
            23
          ]
        },
        {
          "name": "ReshapeTokensToImage"
        }
      ]
    },
    "optimizer_params" : {
      "lr": 5.0e-05,
      "betas": [0.9, 0.999],
      "eps": [1.0e-08],
      "weight_decay": 0.05,
      "amsgrad": false,
      "maximize": false,
      "capturable": false,
      "differentiable": false
    },
    "scheduler_params" : {
        "T_max": 50,
        "eta_min": 0,
        "last_epoch": -1,
        "verbose": "deprecated"
    }
  },


  "torch_dtype": "float32"
}
"""

# 临时为模型创建「config.json」文件。
# 当正确的 config.json 在 HF 平台可用后，该文件将自动消失
with open(os.path.join(os.path.dirname(__file__), "./model/config.json"),
          'w') as config_file:
    config_file.write(model_config)

datamodule_config = {
    'bands': ['BLUE', 'GREEN', 'RED', 'NIR_NARROW', 'SWIR_1', 'SWIR_2'],
    'batch_size':
    16,
    'constant_scale':
    0.0001,
    'data_root':
    '/dccstor/geofm-finetuning/datasets/sen1floods11',
    'drop_last':
    True,
    'no_data_replace':
    0.0,
    'no_label_replace':
    -1,
    'num_workers':
    8,
    'test_transform': [
        albumentations.Resize(always_apply=False,
                              height=448,
                              interpolation=1,
                              p=1,
                              width=448),
        albumentations.pytorch.ToTensorV2(transpose_mask=False,
                                          always_apply=True,
                                          p=1.0)
    ],
}


class PrithviMAE:

    def __init__(self):
        print("Initializing PrithviMAE model")
        self.model = LLM(model=os.path.join(os.path.dirname(__file__),
                                            "./model"),
                         skip_tokenizer_init=True,
                         dtype="float32")

    def run(self, input_data, location_coords):
        print("################ Running inference on vLLM ##############")
        # 合并数据到一个数据结构中
        mm_data = {
            "pixel_values":
            torch.empty(0) if input_data is None else input_data,
            "location_coords":
            torch.empty(0) if location_coords is None else location_coords
        }

        prompt = {"prompt_token_ids": [1], "multi_modal_data": mm_data}

        outputs = self.model.encode(prompt, use_tqdm=False)
        print(
            "################ Inference done (it took seconds)  ##############"
        )

        return outputs[0].outputs.data


def generate_datamodule():
    datamodule = Sen1Floods11NonGeoDataModule(
        data_root=datamodule_config['data_root'],
        batch_size=datamodule_config["batch_size"],
        num_workers=datamodule_config["num_workers"],
        bands=datamodule_config["bands"],
        drop_last=datamodule_config["drop_last"],
        test_transform=datamodule_config["test_transform"
                                         ""])

    return datamodule


def process_channel_group(orig_img, channels):

    """
    参数：
        orig_img：表示原始图像（参考图像）的 torch.Tensor，
                  形状为 (bands, H, W)。
        channels：表示 RGB 通道的索引列表。

    返回：
        原始图像的 torch.Tensor，形状为 (num_channels, height, width)
    """

    orig_img = orig_img[channels, ...]
    valid_mask = torch.ones_like(orig_img, dtype=torch.bool)
    valid_mask[orig_img == NO_DATA_FLOAT] = False

    # 重缩放 (增强对比)
    max_value = max(3000, np.percentile(orig_img[valid_mask], PERCENTILE))
    min_value = OFFSET

    orig_img = torch.clamp((orig_img - min_value) / (max_value - min_value), 0,
                           1)

    # 0 作为无数据
    orig_img[~valid_mask] = 0

    return orig_img


def read_geotiff(file_path: str):
    """Read all bands from *file_path* and return image + meta info.

    Args:
        file_path: path to image file.

    Returns:
        np.ndarray with shape (bands, height, width)
        meta info dict
    """

    with rasterio.open(file_path) as src:
        img = src.read()
        meta = src.meta
        try:
            coords = src.lnglat()
        except Exception:
            # 无法读取 coords
            coords = None

    return img, meta, coords


def save_geotiff(image, output_path: str, meta: dict):
    """将多波段图像保存为 GeoTiff 文件。

    参数：
        image: 形状为 (bands, height, width) 的 np.ndarray 数组
        output_path: 图像保存路径
        meta: 包含元信息的字典
    """

    with rasterio.open(output_path, "w", **meta) as dest:
        for i in range(image.shape[0]):
            dest.write(image[i, :, :], i + 1)

    return


def _convert_np_uint8(float_image: torch.Tensor):
    image = float_image.numpy() * 255.0
    image = image.astype(dtype=np.uint8)

    return image


def load_example(
    file_paths: list[str],
    mean: list[float] = None,
    std: list[float] = None,
    indices: Union[list[int], None] = None,
):
    """通过加载 *file_paths* 中的图像构建输入样本。

    参数：
        file_paths: 文件路径列表
        mean: 包含 *file_paths* 中各图像每个波段均值的列表
        std: 包含 *file_paths* 中各图像每个波段标准差的列表

    返回：
       生成的样本 np.array
       *file_paths* 中各图像的元信息列表
    """

    imgs = []
    metas = []
    temporal_coords = []
    location_coords = []

    for file in file_paths:
        img, meta, coords = read_geotiff(file)

        # 重缩放(不要在空数据上归一化)
        img = np.moveaxis(img, 0, -1)  # channels last for rescaling # 最后一个通道用于重缩放
        if indices is not None:
            img = img[..., indices]
        if mean is not None and std is not None:
            img = np.where(img == NO_DATA, NO_DATA_FLOAT, (img - mean) / std)

        imgs.append(img)
        metas.append(meta)
        if coords is not None:
            location_coords.append(coords)

        try:
            match = re.search(r'(\d{7,8}T\d{6})', file)
            if match:
                year = int(match.group(1)[:4])
                julian_day = match.group(1).split('T')[0][4:]
                if len(julian_day) == 3:
                    julian_day = int(julian_day)
                else:
                    julian_day = datetime.datetime.strptime(
                        julian_day, '%m%d').timetuple().tm_yday
                temporal_coords.append([year, julian_day])
        except Exception as e:
            print(f'Could not extract timestamp for {file} ({e})')

    imgs = np.stack(imgs, axis=0)  # num_frames, H, W, C
    imgs = np.moveaxis(imgs, -1, 0).astype("float32")
    imgs = np.expand_dims(imgs, axis=0)  # add batch di # 添加批 di

    return imgs, temporal_coords, location_coords, metas


def run_model(input_data,
              temporal_coords,
              location_coords,
              model,
              datamodule,
              img_size,
              lightning_model=None):
    # 当图像尺寸不能被 img_size 整除时进行反射填充
    original_h, original_w = input_data.shape[-2:]
    pad_h = (img_size - (original_h % img_size)) % img_size
    pad_w = (img_size - (original_w % img_size)) % img_size
    input_data = np.pad(input_data,
                        ((0, 0), (0, 0), (0, 0), (0, pad_h), (0, pad_w)),
                        mode="reflect")

    # 构建滑动窗口
    batch_size = 1
    batch = torch.tensor(input_data, device="cpu")
    windows = (batch.unfold(3, img_size,
                            img_size).unfold(4, img_size, img_size))
    h1, w1 = windows.shape[3:5]
    windows = rearrange(windows,
                        "b c t h1 w1 h w -> (b h1 w1) c t h w",
                        h=img_size,
                        w=img_size)

    # 如果窗口数量大于批大小则分割批
    num_batches = windows.shape[0] // batch_size if windows.shape[
        0] > batch_size else 1
    windows = torch.tensor_split(windows, num_batches, dim=0)

    if torch.cuda.is_available():
        device = torch.device('cuda')
    else:
        device = torch.device('cpu')

    if temporal_coords:
        temporal_coords = torch.tensor(temporal_coords,
                                       device=device).unsqueeze(0)
    else:
        temporal_coords = None
    if location_coords:
        location_coords = torch.tensor(location_coords[0],
                                       device=device).unsqueeze(0)
    else:
        location_coords = None

    # 运行模型
    pred_imgs = []
    for x in windows:
        # Apply standardization
        # 应用标准化
        x = datamodule.test_transform(
            image=x.squeeze().numpy().transpose(1, 2, 0))
        x = datamodule.aug(x)['image']

        with torch.no_grad():
            x = x.to(device)
            pred = model.run(x, location_coords=location_coords)
            if lightning_model:
                pred_lightning = lightning_model(
                    x,
                    temporal_coords=temporal_coords,
                    location_coords=location_coords)
                pred_lightning = pred_lightning.output.detach().cpu()
                if not torch.equal(pred, pred_lightning):
                    print("Inference output is not equal")
        y_hat = pred.argmax(dim=1)

        y_hat = torch.nn.functional.interpolate(y_hat.unsqueeze(1).float(),
                                                size=img_size,
                                                mode="nearest")

        pred_imgs.append(y_hat)

    pred_imgs = torch.concat(pred_imgs, dim=0)

    # 从块中读取图像
    pred_imgs = rearrange(
        pred_imgs,
        "(b h1 w1) c h w -> b c (h1 h) (w1 w)",
        h=img_size,
        w=img_size,
        b=1,
        c=1,
        h1=h1,
        w1=w1,
    )

    # 剪切填充区域，还原原始大小
    pred_imgs = pred_imgs[..., :original_h, :original_w]

    # 挤压（批大小 1）
    pred_imgs = pred_imgs[0]

    return pred_imgs


def main(
    data_file: str,
    output_dir: str,
    rgb_outputs: bool,
    input_indices: list[int] = None,
):
    os.makedirs(output_dir, exist_ok=True)

    # 读取模型    ---------------------------------------------------------------
    model_obj = PrithviMAE()
    datamodule = generate_datamodule()
    img_size = 256  # Size of Sen1Floods11

    # 读取数据    ---------------------------------------------------------------
    input_data, temporal_coords, location_coords, meta_data = load_example(
        file_paths=[data_file],
        indices=input_indices,
    )

    meta_data = meta_data[0]   #  仅一张图像

    if input_data.mean() > 1:
        input_data = input_data / 10000  # 转换到 0-1 之间

    # 运行模型    ---------------------------------------------------------------

    channels = [
        datamodule_config['bands'].index(b) for b in ["RED", "GREEN", "BLUE"]
    ]  # BGR -> RGB

    pred = run_model(input_data, temporal_coords, location_coords, model_obj,
                     datamodule, img_size)

    # 保存 pred
    meta_data.update(count=1, dtype="uint8", compress="lzw", nodata=0)
    pred_file = os.path.join(
        output_dir,
        f"pred_{os.path.splitext(os.path.basename(data_file))[0]}.tiff")
    save_geotiff(_convert_np_uint8(pred), pred_file, meta_data)

    # 保存 图像 和 pred
    meta_data.update(count=3, dtype="uint8", compress="lzw", nodata=0)

    if input_data.mean() < 1:
        input_data = input_data * 10000  # Scale to 0-10000 # 缩放到 0-10000

    rgb_orig = process_channel_group(
        orig_img=torch.Tensor(input_data[0, :, 0, ...]),
        channels=channels,
    )

    pred[pred == 0.] = np.nan
    img_pred = rgb_orig * 0.7 + pred * 0.3
    img_pred[img_pred.isnan()] = rgb_orig[img_pred.isnan()]

    img_pred_file = os.path.join(
        output_dir,
        f"rgb_pred_{os.path.splitext(os.path.basename(data_file))[0]}.tiff")
    save_geotiff(
        image=_convert_np_uint8(img_pred),
        output_path=img_pred_file,
        meta=meta_data,
    )

    # 保存图片 rgb
    if rgb_outputs:
        rgb_file = os.path.join(
            output_dir, "original_rgb_"
            f"{os.path.splitext(os.path.basename(data_file))[0]}.tiff")
        save_geotiff(
            image=_convert_np_uint8(rgb_orig),
            output_path=rgb_file,
            meta=meta_data,
        )


if __name__ == "__main__":
    parser = argparse.ArgumentParser("MAE run inference", add_help=False)

    parser.add_argument(
        "--data_file",
        type=str,
        default="./India_900498_S2Hand.tif",
        help="Path to the file.",
    )
    parser.add_argument(
        "--output_dir",
        type=str,
        default="output",
        help="Path to the directory where to save outputs.",
    )
    parser.add_argument(
        "--input_indices",
        default=[1, 2, 3, 8, 11, 12],
        type=int,
        nargs="+",
        help=
        "0-based indices of the six Prithvi channels to be selected from the  "
        "input. By default selects [1,2,3,8,11,12] for S2L1C data.",
    )
    parser.add_argument(
        "--rgb_outputs",
        action="store_true",
        help="If present, output files will only contain RGB channels. "
        "Otherwise, all bands will be saved.",
    )
    args = parser.parse_args()

    main(**vars(args))

```
