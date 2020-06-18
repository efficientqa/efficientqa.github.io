# Getting Started
We have provided a number of baseline systems, to help you get started with this
challenge.

## Retrieval-based baselines

*(Working in progress)*

We provide two retrieval-based baselines.

- DrQA: Danqi Chen, Adam Fisch, Jason Weston, Antoine Bordes. [Reading Wikipedia to Answer Open-Domain Questions](https://arxiv.org/abs/1704.00051). ACL 2017. [[Original implementation](https://github.com/facebookresearch/DrQA)]
- DPR: Vladimir Karpukhin, Barlas Oğuz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi Chen, Wen-tau Yih, [Dense Passage Retrieval for Open-Domain Question Answering](https://arxiv.org/abs/2004.04906), Preprint 2020. [[Original implementation](https://github.com/facebookresearch/DPR)]

We provide two variants for each model, using (1) full Wikipedia and (2) Wikipedia pages seen from the train data.

|Model|Exact Mach|Disk usage (gb)|
|---|---|---|
|DrQA-full|-|20.1|
|DrQA-seen-only|-|2.8|
|DPR-full|41.5|66.4|
|DPR-seen-only|-|5.9|

Details on training and testing the models are available [here](https://github.com/efficientqa/retrieval-based-baselines).

**Step 1**: Git clone and download data

```bash
# Step 1: Git clone
git clone https://github.com/facebookresearch/DPR.git # dependency
git clone https://github.com/efficientqa/retrieval-based-baselines.git # this repo
cd retrieval-based-baselines
# Step 2: Download data
python3 data/download_data.py --resource data.retriever.qas --output_dir {base_dir}
python3 data/download_data.py --resource data.wikipedia_split --output_dir {base_dir} --keep-gzip
# Step 3: (For seen-only variants) filter passages that are seen on the train data
python3 keep_seen_docs_only --db_path {base_dir}/data/wikipedia_split/psgs_w100.tsv.gzip --data_path {base_dir}/data/retriever/qas/nq-train.csv
```
**Step 2**: Download indexes and pretrained models

```bash
# For DrQA-full
TODO
# For DrQA-seen-only
TODO
# For DPR-full
TODO
# For DPR-seen-only
TODO
```

**Step 3**: Run inference script to save predictions from the questions

```bash
python3 run_inference.py \
  --qa_file {base_dir}/data/retriever/qas/nq-{dev|test}.csv \ # data file with questions
  --retrieval_type {drqa|dpr} \ # which retrieval to use
  --db_path {base_dir}/data/wikipedia_split/{psgs_w100.tsv.gzip|psgs_w100_seen_only.tsv.gzip} \
  --tfidf_path {base_dir}/checkpoint/drqa_retriever/tfidf.npz \ # only matters for drqa retrieval
  --drqa_model_file {base_dir}/checkpoint/retriever/single/nq/bert-base-encoder.cp \ # only matters for dpr retrieval
  --model_file {base_dir}/checkpoint/reader/TODO \ # path to the reader checkpoint
  --dev_batch_size 8 \ # 8 is good for one 32gb GPU
  --pretrained_model_cfg bert-base-uncased --encoder_model_type hf_bert --do_lower_case \
  --sequence_length 350 --eval_top_docs 10 20 40 50 80 100 --passages_per_question_predict 100 \
  --prediction_results_file {base_dir}/checkpoint/reader/TODO/dev_predictions.json # path to save predictions
```

## T5
