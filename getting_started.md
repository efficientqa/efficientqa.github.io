# Getting Started
We have provided a number of baseline systems, to help you get started with this
challenge.

* [Retrivial-based (DrQA / DPR)](#retrieval-based)
* [Generative (T5)](#generative)


## Retrieval-based (DrQA / DPR) <a name="retrieval-based"></a>


We provide two retrieval-based baselines.

- DrQA: Danqi Chen, Adam Fisch, Jason Weston, Antoine Bordes. [Reading Wikipedia to Answer Open-Domain Questions](https://arxiv.org/abs/1704.00051). ACL 2017. [[Original implementation](https://github.com/facebookresearch/DrQA)]
- DPR: Vladimir Karpukhin, Barlas Oğuz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi Chen, Wen-tau Yih, [Dense Passage Retrieval for Open-Domain Question Answering](https://arxiv.org/abs/2004.04906), Preprint 2020. [[Original implementation](https://github.com/facebookresearch/DPR)]

We provide two variants for each model, using (1) full Wikipedia and (2) Wikipedia pages seen from the train data.

|Model|Exact Mach|Disk usage (gb)|
|---|---|---|
|DrQA-full|32.0|20.1|
|DrQA-seen-only|31.1|2.8|
|DPR-full|41.4|66.4|
|DPR-seen-only|35.1|5.9|

Details on training and testing the models are available [here](https://github.com/efficientqa/retrieval-based-baselines).

### Getting ready

```bash
git clone https://github.com/facebookresearch/DPR.git && cd DPR && pip3 install .
cd ..
git clone https://github.com/efficientqa/retrieval-based-baselines.git && cd retrieval-based-baselines && pip3 install -r requirements.txt
```

Let `${base_dir}` as your base directory to store data and pretrained models. (For example, run `export base_dir="data"`.)


### Running end-to-end prediction (shortcut)
```bash
chmod +x run.sh
./run.sh drqa-full # for DrQA-full
./run.sh drqa-seen-only # for DrQA-seen-only
./run.sh dpr-full # for DPR-full
./run.sh dpr-seen-only # for DPR-seen-only
```

### Running end-to-end prediction (detail)

**Step 1**: Download data

```bash
python3 ../DPR/data/download_data.py --resource data.retriever.qas.nq --output_dir ${base_dir} # QA data
python3 ../DPR/data/download_data.py --resource data.wikipedia_split --output_dir ${base_dir} # Wikipedia DB
# (For seen-only variants) filter passages that are seen on the train data
python3 ../DPR/data/download_data.py --resource data.retriever.nq-train --output_dir ${base_dir}
python3 keep_seen_docs_only.py --db_path ${base_dir}/data/wikipedia_split/psgs_w100.tsv --data_path ${base_dir}/data/retriever/nq-train.json
```

**Step 2**: Download indexes and pretrained models

```bash
# For DrQA-full
python3 ../DPR/data/download_data.py --resource indexes.drqa.nq.full --output_dir ${base_dir} # DrQA index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-drqa.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export drqa_index="${base_dir}/indexes/drqa/nq/full.npz"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-drqa/hf-bert-base.cp"

# For DrQA-seen-only
python3 ../DPR/data/download_data.py --resource indexes.drqa.nq.seen_only --output_dir ${base_dir} # DrQA index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-drqa-seen_only.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export drqa_index="${base_dir}/indexes/drqa/nq/seen_only.npz"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-drqa-seen_only/hf-bert-base.cp"

# For DPR-full
python3 ../DPR/data/download_data.py --resource checkpoint.retriever.single.nq.bert-base-encoder --output_dir ${base_dir} # retrieval checkpoint
python3 ../DPR/data/download_data.py --resource indexes.single.nq.full --output_dir ${base_dir} # DPR index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-single.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export dpr_retrieval_checkpoint="${base_dir}/checkpoint/retriever/single/nq/bert-base-encoder.cp"
export dpr_index="${base_dir}/indexes/single/nq/full"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-single/hf-bert-base.cp"

# For DPR-seen-only
python3 ../DPR/data/download_data.py --resource checkpoint.retriever.single.nq.bert-base-encoder --output_dir ${base_dir} # retrieval checkpoint
python3 ../DPR/data/download_data.py --resource indexes.single.nq.seen_only --output_dir ${base_dir} # DPR index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-single-seen_only.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export dpr_retrieval_checkpoint="${base_dir}/checkpoint/retriever/single/nq/bert-base-encoder.cp"
export dpr_index="${base_dir}/indexes/single/nq/seen_only"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-single-seen_only/hf-bert-base.cp"
```

**Step 3**: Run inference script to save predictions from the questions

```bash
python3 run_inference.py \
  --qa_file ${base_dir}/data/retriever/qas/nq-{dev|test}.csv \ # data file with questions
  --retrieval_type {drqa|dpr} \ # which retrieval to use
  --db_path ${base_dir}/data/wikipedia_split/{psgs_w100.tsv|psgs_w100_seen_only.tsv} \
  --tfidf_path ${drqa_index} \ # only matters for drqa retrieval
  --dpr_model_file ${dpr_retrieval_checkpoint} \ # only matters for dpr retrieval
  --dense_index_path ${dpr_index} \ # only matters for dpr retrieval
  --model_file ${reader_checkpoint} \ # path to the reader checkpoint
  --dev_batch_size 8 \ # 8 is good for one 32gb GPU
  --pretrained_model_cfg bert-base-uncased --encoder_model_type hf_bert --do_lower_case \
  --sequence_length 350 --eval_top_docs 10 20 40 50 80 100 \
  --passages_per_question_predict 100 \ # 100 for DrQA, 40 for DPR
  --prediction_results_file dev_predictions.json # path to save predictions; comparable to the official evaluation script
```

## Generative (T5) <a name="generative"></a>

This section provides pointers on how to reproduce the experiments on the purely generative approach to "closed-book question answering" with [T5](https://ai.googleblog.com/2020/02/exploring-transfer-learning-with-t5.html) detailed in [How Much Knowledge Can You Pack Into the Parameters of a Language Model?](https://arxiv.org/abs/2002.08910) (Roberts, et al. 2020).

This approach fine-tunes a large language model ([T5](https://github.com/google-research/text-to-text-transfer-transformer)) to solve QA tasks using only the knowledge stored in its parameters based on unsupervised pre-training over a large corpus based on Common Crawl ([C4](http://tensorflow.org/datasets/catalog/c4)).

Instructions for fine-tuning the T5 model on Natural Questions can be found at https://github.com/google-research/google-research/tree/master/t5_closed_book_qa, along with already fine-tuned checkpoints. The repository itself provides an example for how to create new pre-training and fine-tuning tasks with the T5 library. An example for how to interactively call the T5 repo be found in the [T5 Colab](https://tiny.cc/t5-colab), which also trains on Natural Questions but lacks some of the improvements implemented in the CBQA repo. 

You can access both via Colab to use a free `v2-8` TPU, which is powerful enough to finetune a T5-3B model. For T5-11B you will need to purchase time on a `v3-8` TPU or larger.
