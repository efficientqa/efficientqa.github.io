# Getting Started with Baselines
We have provided a number of baseline systems, to help you get started with this
challenge.

* [Retrieval-based (TF-IDF / DPR)](#retrieval-based) stores a database of Wikipedia, retrieves a set of relevant passages to the question, and employs a multi-passage reader to find the answer from the retrieved passages (implementation in PyTorch). TF-IDF is a standard sparse term-based retriever (adapted from the [DrQA system](https://github.com/facebookresearch/DrQA)) and DPR is based on a fully dense-vector passage retriever learned from question/answer pairs.
* [Generative (T5)](#generative) stores all knolwedge in its parameters based on large-scale, unsupervised pretraining, and generates the answer directly from the question (implementations in Tensorflow and Huggingface PyTorch).


## Retrieval-based (TF-IDF / DPR) <a name="retrieval-based"></a>


We provide two retrieval-based baselines:

- TF-IDF: TF-IDF retrieval built on fixed-length passages, adapted from the [DrQA system's implementation](https://github.com/facebookresearch/DrQA).
<!-- <br>Danqi Chen, Adam Fisch, Jason Weston, Antoine Bordes. [Reading Wikipedia to Answer Open-Domain Questions](https://arxiv.org/abs/1704.00051). ACL 2017.  -->
- DPR: A learned dense passage retriever, detailed in [Dense Passage Retrieval for Open-Domain Question Answering](https://arxiv.org/abs/2004.04906) (Karpukhin et al, 2020). Our baseline is adapted from the [original implementation](https://github.com/facebookresearch/DPR).

<!-- Vladimir Karpukhin, Barlas Oğuz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi Chen, Wen-tau Yih, [Dense Passage Retrieval for Open-Domain Question Answering](https://arxiv.org/abs/2004.04906), 2020. [[repo](https://github.com/facebookresearch/DPR)] -->

Note that for both baselines, we use text blocks of 100 words as passages and a BERT-base multi-passage reader. See more details in the [DPR paper](https://arxiv.org/pdf/2004.04906.pdf).

<!-- <p style="font-size: 8pt">* Note that our DrQA baseline is different from the original s; we use the document retriever from DrQA, but use a more effective BERT-base multi-passage reader instead of LSTM-based DrQA reader. The reader is identical for both DrQA and DPR.</p> -->

We provide two variants for each model, using (1) full Wikipedia (`full`) and (2) A subset of Wikipedia articles which are found relevant to the questions on the train data (`subset`). In particular, we think of `subset` as a naive way to reduce the disk memory usage for the retrieval-based baselines.

<!-- Wikipedia pages seen from the train data, i.e., found to be relevant to the questions on the train data (`train-passages`).  -->

|Model|Exact Mach|Disk usage (gb)|
|---|---|---|
|TFIDF-full|32.0|20.1|
|TFIDF-subset|31.0|2.8|
|DPR-full|41.0|66.4|
|DPR-subset|34.8|5.9|

Details on training and testing the models are available at [EfficientQA Retrieval-base Baselines Repo](https://github.com/efficientqa/retrieval-based-baselines).

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
./run.sh tfidf-full # for TFIDF-full
./run.sh tfidf-subset # for TFIDF-subset
./run.sh dpr-full # for DPR-full
./run.sh dpr-subset # for DPR-subset
```

### Running end-to-end prediction (detail)

**Step 1**: Download data

```bash
wget https://raw.githubusercontent.com/efficientqa/nq-open/master/NQ-open.dev.jsonl
python3 ../DPR/data/download_data.py --resource data.wikipedia_split --output_dir ${base_dir} # Wikipedia DB
# (For "subset" variants) create a new DB with a subset of Wikipedia passages
python3 ../DPR/data/download_data.py --resource data.retriever.nq-train --output_dir ${base_dir}
python3 filter_subset_wiki.py --db_path ${base_dir}/data/wikipedia_split/psgs_w100.tsv --data_path ${base_dir}/data/retriever/nq-train.json
```

**Step 2**: Download indexes and pretrained models

```bash
# For TFIDF-full
python3 ../DPR/data/download_data.py --resource indexes.tfidf.nq.full --output_dir ${base_dir} # TFIDF index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-tfidf.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export tfidf_index="${base_dir}/indexes/tfidf/nq/full.npz"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-tfidf/hf-bert-base.cp"

# For TFIDF-subset
python3 ../DPR/data/download_data.py --resource indexes.tfidf.nq.subset --output_dir ${base_dir} # TFIDF index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-tfidf-subset.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export tfidf_index="${base_dir}/indexes/tfidf/nq/subset.npz"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-tfidf-subset/hf-bert-base.cp"

# For DPR-full
python3 ../DPR/data/download_data.py --resource checkpoint.retriever.single.nq.bert-base-encoder --output_dir ${base_dir} # retrieval checkpoint
python3 ../DPR/data/download_data.py --resource indexes.single.nq.full --output_dir ${base_dir} # DPR index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-single.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export dpr_retrieval_checkpoint="${base_dir}/checkpoint/retriever/single/nq/bert-base-encoder.cp"
export dpr_index="${base_dir}/indexes/single/nq/full"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-single/hf-bert-base.cp"

# For DPR-subset
python3 ../DPR/data/download_data.py --resource checkpoint.retriever.single.nq.bert-base-encoder --output_dir ${base_dir} # retrieval checkpoint
python3 ../DPR/data/download_data.py --resource indexes.single.nq.subset --output_dir ${base_dir} # DPR index
python3 ../DPR/data/download_data.py --resource checkpoint.reader.nq-single-subset.hf-bert-base --output_dir ${base_dir} # reader checkpoint
export dpr_retrieval_checkpoint="${base_dir}/checkpoint/retriever/single/nq/bert-base-encoder.cp"
export dpr_index="${base_dir}/indexes/single/nq/subset"
export reader_checkpoint="${base_dir}/checkpoint/reader/nq-single-subset/hf-bert-base.cp"
```

**Step 3**: Run inference script to save predictions from the questions

```bash
python3 run_inference.py \
  --qa_file NQ-open.dev.jsonl \ # data file with questions
  --retrieval_type {tfidf|dpr} \ # which retrieval to use
  --db_path ${base_dir}/data/wikipedia_split/{psgs_w100.tsv|psgs_w100_subset.tsv} \
  --tfidf_path ${tfidf_index} \ # only matters for TFIDF retrieval
  --dpr_model_file ${dpr_retrieval_checkpoint} \ # only matters for dpr retrieval
  --dense_index_path ${dpr_index} \ # only matters for dpr retrieval
  --model_file ${reader_checkpoint} \ # path to the reader checkpoint
  --dev_batch_size 8 \ # 8 is good for one 32gb GPU
  --pretrained_model_cfg bert-base-uncased --encoder_model_type hf_bert --do_lower_case \
  --sequence_length 350 --eval_top_docs 10 20 40 50 80 100 \
  --passages_per_question_predict 100 \ # 100 for TFIDF, 40 for DPR
  --prediction_results_file dev_predictions.json # path to save predictions; comparable to the official evaluation script
```

## Generative (T5) <a name="generative"></a>

This section provides pointers on how to reproduce the experiments on the purely generative approach to "closed-book question answering" with [T5](https://ai.googleblog.com/2020/02/exploring-transfer-learning-with-t5.html) detailed in [How Much Knowledge Can You Pack Into the Parameters of a Language Model?](https://arxiv.org/abs/2002.08910) (Roberts, et al. 2020).

This approach fine-tunes a large language model ([T5](https://github.com/google-research/text-to-text-transfer-transformer)) to solve QA tasks using only the knowledge stored in its parameters based on unsupervised pre-training over a large corpus based on Common Crawl ([C4](http://tensorflow.org/datasets/catalog/c4)). Below are the results and disk usage (Docker image size) for various model sizes and configurations.

|Model|Exact Mach|Disk usage (gb)|
|---|---|---|
|T5-1.1-small + SSM | 25.8 |0.39|
|T5-1.1-XL + SSM |35.6|5.60|
|T5-1.1-XXL + SSM|37.9|22.0|

### Fine-tuning

Instructions for fine-tuning the T5 model on Natural Questions via command-line can be found in the [T5 Closed Book QA github repo](https://github.com/google-research/google-research/tree/master/t5_closed_book_qa), along with already fine-tuned checkpoints. The repository itself demonstrates how to create new pre-training and fine-tuning tasks with the base [T5 library](https://github.com/google-research/text-to-text-transfer-transformer). An example for how to interactively call the T5 repo be found in the [T5 Colab](https://tiny.cc/t5-colab), which also trains on Natural Questions but lacks some of the improvements implemented in the Closed Book QA repo.

### Accelerators

While a CPU is sufficient for inference, some variants of T5 require significant compute resources to pre-train or fine-tune. Colab provides a free `v2-8` TPU, which is powerful enough to finetune the XL/3B model or below. For XXL/11B, you will need a `v3-8` TPU or larger.

It is also possible to fine-tune using one or more GPUs following instructions for the [TensorFlow implementation](https://github.com/google-research/text-to-text-transfer-transformer#gpu-usage) or HuggingFace's [PyTorch implementation](https://github.com/google-research/text-to-text-transfer-transformer/blob/a08f0d1c4a7caa6495aec90ce769a29787c3c87c/t5/models/hf_model.py#L38). Note that the PyTorch implementation does not support model parallelism, and is therefore incompatible with the XXL/11B model.

### Inference

Running inference on T5 models is demonstrated in the [T5 Colab](https://tiny.cc/t5-colab) and [command line instructions](https://github.com/google-research/text-to-text-transfer-transformer#decode).

COMING SOON: An example of how to package a model checkpoint into a [Docker image](https://www.tensorflow.org/tfx/serving/docker) for submission and serving.

