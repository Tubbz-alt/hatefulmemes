# The Hateful Memes challenge

## Introduction

My best scoring solution to the [Hateful Memes: Phase 2](https://www.drivendata.org/competitions/70/hateful-memes-phase-2/) 
challenge comprises of an ensemble of a single UNITER model architecture (average over probabilities) 
[[paper]](https://arxiv.org/abs/1909.11740) [[code]](https://github.com/ChenRocks/UNITER), 
which I have adapted for this competition. I have customized the paired-attention 
approach from the UNITER paper to include image captions
inferred from the [Im2txt implemention](https://github.com/HughKu/Im2txt) of the 
[Show and Tell: Lessons learned from the 2015 MSCOCO Image Captioning Challenge](http://arxiv.org/abs/1609.06647) 
paper. You can say this was my original contribution to the competition, albeit it didn't improve my results by a lot 
(around 1.2% for the AUC). However I really liked the simplicity of it. I extracted ROI boxes and 
image features by adapting the 
[Bottom-up Attention with Detectron2](https://github.com/airsplay/py-bottom-up-attention) 
work which in itself is a pytorch adaptation of the original 
[bottom-up-attention](https://github.com/peteanderson80/bottom-up-attention) approach using
caffe.

The overall goal was to get familiar with the multimodal SOTA models out there 
and not focus too much on fancier ensembling or stacking. I find it much more fun to try to improve 
a single architecture.

A lot of info up to here, let's take it step by step. If you want to replicate my solution, 
here's what you need to do.

## Prepare

Please read the instructions entirely first, before starting the process, to ensure you have
at last some overview before you start. It is also **very** useful to read the installation instructions
from the original repos I have used to get an even better overview. 
In my experience, running somebody else's scripts on your own instance setup rarely works out of the box the first time.

### Environment
* Ubuntu 16.04, CUDA 9.0, GCC 5.4.0
* Anaconda 3
* Python 3.6.x (pandas and jupyterlab needed)

### Separate multiple conda environments
The project consists of three sub-projects which have been adapted for this task:
1. [Bottom-up Attention with Detectron2](https://github.com/airsplay/py-bottom-up-attention)
2. [Im2txt: image captioning inference](https://github.com/HughKu/Im2txt)
3. [UNITER: UNiversal Image-TExt Representation Learning](https://github.com/ChenRocks/UNITER) 

I **strongly** recommend creating a separate conda environment for each. 
For this there is a script in the *conda* folder in each of the sub-projects. 
They are all independent, so they should each work as a standalone project, requiring only a path to the data folder.
Sure there is a flow, with py-bottom-up-attention and Im2txt working as feature extractors for UNITER down the line.
It is more for convenience I bundled them in one package in order to document the solution. 
Once you cloned the repo, you should see this structure and among other things, there are three scripts:
```bash
    hatefulmemes
    ├── Im2txt
    │   ├── conda
    │   │   ├── init_im2txt_ubuntu.sh
    │   ├── ...
    ├── py-bottom-up-attention
    │   ├── conda
    │   │   ├── init_bua_ubuntu.sh
    │   ├── ...
    └── UNITER
        ├── conda
        │   ├── init_uniter_ubuntu.sh
        └── ...
```
Please pay attention to the `/path/to/` and `anaconda3` paths in the scripts.
You need to change this to the location of your cloned repo and conda respectively.
Afterwards run the scripts to create each of the conda environments. 

### Alternative installation instructions
If you prefer not to, or you cannot make it work using the scripts above, 
you can also follow the installation instructions from the original repos. 
That's where my scripts come from anyway.

### Download required datasets and pretrained models
#### 0. The HM dataset
Grab the [HM dataset for phase 2](https://www.drivendata.org/competitions/70/hateful-memes-phase-2/data/) 
and place the unzipped files inside a `data` folder, under the project root for example
(on the same level with `Im2txt`, `py-bottom-up-attention` and `UNITER`).

Grab the [dev_seen_unseen.jsonl](https://drive.google.com/file/d/1ASW0JTYxl9Wazu3GVeqAllaWxndUq1iY/view?usp=sharing) file and place it in the same folder as the other `jsonl` files 
provided by the organizers.

*NOTE*:
Since no good data should go to waste, I have merged the dev_seen and dev_unseen sets.
This increased the size of the validation set to 640 samples, instead of 500.
Also the final predictions on the test_unseen set have been produced by training 
the UNITER model on **train+dev_seen_unseen**.

If you don't want to readily download the **dev_seen_unseen.jsonl** file, 
you can also build it yourself by running the `notebooks/ph2_merge_dev_seen_unseen.ipynb` notebook.

#### 1. py-bottom-up-attention
The required pretrained models should be downloaded automatically when running it.

Otherwise please follow the [original instructions](https://github.com/airsplay/py-bottom-up-attention#note) and
download the [10-100 boxes original model weights](http://nlp.cs.unc.edu/models/faster_rcnn_from_caffe_attr_original.pkl).
Or take the shortcut below, but please always check the original link since the author might have made changes in the meantime.
```bash 
wget --no-check-certificate http://nlp.cs.unc.edu/models/faster_rcnn_from_caffe_attr_original.pkl -P ~/.torch/fvcore_cache/models/
```

### 2. Im2txt
For Im2txt follow the [original instructions](https://github.com/HughKu/Im2txt#get-pre-trained-model), 
or take the shortcut below, but please always check the original link since the author might have made changes in the meantime.

"Download [inceptionv3 finetuned parameters over 1M](https://drive.google.com/open?id=1r4-9FEIbOUyBSvA-fFVFgvhFpgee6sF5) 
and you will get 4 files, and make sure to put them all into this path 
`Im2txt/im2txt/model/Hugh/train/`
* newmodel.ckpt-2000000.data-00000-of-00001
* newmodel.ckpt-2000000.index
* newmodel.ckpt-2000000.meta
* checkpoint
"

### 3. UNITER
For Uniter follow the [original instructions](https://github.com/ChenRocks/UNITER#quick-start), 
more specifically the [pretrained UNITER-large model](https://github.com/ChenRocks/UNITER/blob/master/scripts/download_pretrained.sh), 
or take the shortcut below to download the pretrained model, but please always check the original link since the author might have made changes in the meantime.

*NOTE*: the path to the pretrained model has to match the path in your training config file.

```bash 
wget https://convaisharables.blob.core.windows.net/uniter/pretrained/uniter-large.pt -P /path/to/UNITER/storage/pretrained/
```

### Running the models
#### 1. Extracting bounding boxes and image features

You can download the already extracted bboxes + features files 
[here](https://drive.google.com/file/d/1XwLCpawhF4AMzJRG7sqG4uPywnL9YrTF/view?usp=sharing).
Otherwise you can extract them yourself using the following code from `py-bottom-up-attention`. 
The code extracts the features and places the `tsv` file for all the images in the `imgfeat` folder.
This will take about one hour to execute (NVidia V100 GPU).

```bash
conda activate bua

python demo/detectron2_mscoco_proposal_maxnms_hm.py 
    --split img 
    --data_path /path/to/data/ 
    --output_path /path/to/data/imgfeat/
    --output_type tsv 
    --min_boxes 10 --max_boxes 100 
```
Split up the `tsv` file into each of the sets, by joining with the `jsonl` files. This will 
create another set of `tsv` files, which will be ingested by UNITER.
```bash
python demo/hm.py --split img --split_json_file train.jsonl --d2_file_suffix d2_10-100 --data_path /path/to/data/ --output_path /path/to/data/imgfeat/
python demo/hm.py --split img --split_json_file dev_seen.jsonl --d2_file_suffix d2_10-100 --data_path /path/to/data/ --output_path /path/to/data/imgfeat/
python demo/hm.py --split img --split_json_file dev_unseen.jsonl --d2_file_suffix d2_10-100 --data_path /path/to/data/ --output_path /path/to/data/imgfeat/
python demo/hm.py --split img --split_json_file dev_seen_unseen.jsonl --d2_file_suffix d2_10-100 --data_path /path/to/data/ --output_path /path/to/data/imgfeat/
python demo/hm.py --split img --split_json_file test_seen.jsonl --d2_file_suffix d2_10-100 --data_path /path/to/data/ --output_path /path/to/data/imgfeat/
python demo/hm.py --split img --split_json_file test_unseen.jsonl --d2_file_suffix d2_10-100 --data_path /path/to/data/ --output_path /path/to/data/imgfeat/ 
```

This is how your `data` folder structure should look like now:
```bash
    hatefulmemes
    ├── data
    │   ├── img (all the memes, .png files)
    │   ├── train.jsonl
    │   ├── dev_seen.jsonl
    │   ├── dev_unseen.jsonl
    │   ├── dev_seen_unseen.jsonl
    │   ├── test_unseen.jsonl
    │   ├── imgfeat/d2_10-100/tsv/img.tsv
    │   ├── data_train_d2_10-100.tsv
    │   ├── data_dev_seen_unseen_d2_10-100.tsv
    │   ├── data_test_unseen_d2_10-100.tsv
    │   ├── ...
    ├── Im2txt
    ├── py-bottom-up-attention
    └── UNITER
```

### 2. Inferring image captions
Download the already inferred captions file 
[here](https://drive.google.com/file/d/1VhXKeMS1CNfhUOrVe93QxYMa6BGzZdTX/view?usp=sharing) and place it under `data/im2txt` folder.
Otherwise, run the following code yourself.

This will take about 4-5 hours (Nvidia V100 GPU), 
but could probably be run also on something less powerful or even a CPU, since it only does inference.

```bash
conda activate im2txt

python im2txt/run_inference.py 
    --checkpoint_path="im2txt/model/Hugh/train/newmodel.ckpt-2000000" 
    --vocab_file="im2txt/data/Hugh/word_counts.txt" 
    --input_files="/path/to/data/img/*.png"
```
Take the output csv file `df_ph2.csv` and remember to place it under `data/im2txt` folder.

This is how your `data` folder structure should look like now:
```bash
    hatefulmemes
    ├── data
    │   ├── im2txt/df_ph2.csv
    │   ├── ...
    │   ├── img (all the memes, .png files)
    │   ├── train.jsonl
    │   ├── dev_seen.jsonl
    │   ├── dev_unseen.jsonl
    │   ├── dev_seen_unseen.jsonl
    │   ├── test_unseen.jsonl
    │   ├── imgfeat/d2_10-100/tsv/img.tsv
    │   ├── data_train_d2_10-100.tsv
    │   ├── data_dev_seen_unseen_d2_10-100.tsv
    │   ├── data_test_unseen_d2_10-100.tsv
    │   ├── ...
    ├── Im2txt
    ├── py-bottom-up-attention
    └── UNITER
```

### 3. Training UNITER with paired attention
Training just one UNITER model on both `train+dev_seen_unseen` takes about one hour on 
a Nvidia V100 GPU. The current implementation does not support distributed training.
To replicate my leaderboard solution, you need to train 12 UNITER models with different seeds.
```bash
conda activate uniter

python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_0.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_24.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_42.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_77.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_2018.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_12345.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_32768.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_54321.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_10101010.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_20200905.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_55555555.json
python train_hm.py --config config/ph2_uniter_seeds/train-hm-large-pa-1gpu-hpc_2147483647.json
```

The final probabilities should be the average over the probabilities from the 12 model ensemble.

*NOTE*: You should only care about the files `test_results_1140_rank0_final.csv`, the rest are just intermediate results.
Use the `notebooks/ph2_leaderboard.ipynb` to generate the final leaderboard results, but make sure to change the UNITER 
output paths first.

*NOTE*:
All the `csv` test results are available for download from [here](https://drive.google.com/file/d/17rI-4w2v5NNgA6t7zlE944xmcIXbbbXv/view?usp=sharing).
The only difference between the 12 models in the final ensemble in the initialization seed.
For validating the solution and making sure you replicated everything correctly, you 
can also randomly pick one seed and train only one model yourself, then compare the results with the 
downloaded version.