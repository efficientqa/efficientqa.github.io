# Submission instructions

<!-- Intro -->
EfficientQA has multiple tracks. There are three *restricted* tracks that judge
submission on the basis of their size, as well as the accuracy of their
predictions.

All submissions to the *restricted* tracks must be submitted to the
[EfficientQA leaderboard](https://ai.google.com/research/NaturalQuestions/efficientqa).
You can also submit to the *unrestricted* track using the EfficientQA
leaderboard. However, if your system will not run on the leaderboard hardware,
we will also release the test set input on 2020/11/01, after the leaderboard is
frozen, and you will have until the end of 2020/11/03 to send predictions to be
evaluated for the *unrestricted* track only.

This page contains general submission instructions as well as end-to-end walk-through instructions for submitting:

* [T5](https://efficientqa.github.io/getting_started.html#parametric), which is a sequence to sequence model that generates answers directly.
* The [ORQA](https://github.com/google-research/language/tree/master/language/orqa)-based [REALM](https://github.com/google-research/language/tree/master/language/realm) model, which retrieves evidence passages using a [ScaNN](https://ai.googleblog.com/2020/07/announcing-scann-efficient-vector.html) index.

## General instructions

EfficientQA leaderboard submissions are Docker images, uploaded to the
[Google Container Registry](https://cloud.google.com/container-registry).
Submissions must contain an executable script `~/submission.sh` that will be run
with the following command.

```sh
./submission.sh <input_file> <output_file>
```

Where `<input_file>` contains one JSON example per line, where each example only
contains a `question` field.

```json
{"question": "who won the women's australian open 2018"}
{"question": "kuchipudi is a dance form of which state"}
{"question": "who did stephen amell play in private practice"}
{"question": "who created the chamber of secrets in harry potter"}
```

And `/submission.sh` should write predictions to `<output_file>`, one per line,
as follows:

```json
{"question": "who won the women's australian open 2018", "prediction": "Caroline Wozniacki"}
{"question": "kuchipudi is a dance form of which state", "prediction": "Tamil Nadu"}
{"question": "who did stephen amell play in private practice", "prediction": "a pedestrian"}
{"question": "who created the chamber of secrets in harry potter", "prediction": "the Heir of Salazar Slytherin"}.
```

The Docker image containing `/submission.sh` must be fully self contained. It
will not be allowed to pull in libraries or any other external resources.

### Testing submissions
Before submitting your image, please test it locally using the following commands

```sh
INPUT_DIR=/tmp/efficientqa_input
OUTPUT_DIR=/tmp/efficientqa_output
EVAL_DIR=/tmp/efficientqa_eval

mkdir ${INPUT_DIR}
mkdir ${OUTPUT_DIR}
mkdir ${EVAL_DIR}

wget https://raw.githubusercontent.com/google-research-datasets/natural-questions/master/nq_open/NQ-open.efficientqa.dev.no-annotations.jsonl -P "${INPUT_DIR}"
wget https://raw.githubusercontent.com/google-research-datasets/natural-questions/master/nq_open/NQ-open.efficientqa.dev.jsonl -P "${EVAL_DIR}"

docker pull gcr.io/<your_project_id>/<your_image_name>:<your_image_tag>
docker run -v ${INPUT_DIR}:/input -v ${OUTPUT_DIR}:/output \
  --network="none" \
  gcr.io/<your_project_id>/<your_image_name>:<your_image_tag> \
  ./submission.sh \
  /input/NQ-open.efficientqa.dev.no-annotations.jsonl \
  /output/predictions.jsonl

cd ${EVAL_DIR}
git clone https://github.com/google-research/language.git
pip3 install tensorflow
python3 -m language.orqa.evaluation.evaluate_predictions \
  --references_path=${EVAL_DIR}/NQ-open.efficientqa.dev.jsonl \
  --predictions_path=${OUTPUT_DIR}/predictions.jsonl
```

and ensure that you have set the permissions correctly, as
[detailed below](#uploading-submissions-and-submitting-to-test).

### Uploading submissions and submitting to test

Test submissions are run using Google Cloud, and you will need to create a
Google Cloud Platform account at <cloud.google.com>. This account will be used
to store you submissions and, while storage costs should be negligible, we
encourage all participahts to make use of
[Google Cloud's free credits](https://cloud.google.com/free). We will not charge
you for the cost of running your submissions.

Once you have a Google Cloud account, you will need to create a project for your
submissions in your console, and you will need to give us permission to access
this project as follows:

1.  Go to the storage tab in the your console.
2.  Locate the artifacts bucket. It should have a name like
    `artifacts.<project-name>.appspot.com`.
3.  From this bucket's drop-down menu, select "Edit bucket permissions".
4.  Grant "Storage Object Viewer" permissions to
    mljam-compute@mljam-205019.iam.gserviceaccount.com. You can find this option
    under "Storage" when selecting roles for new members.

Now you can either upload your submission directly, or you can use the
[Cloud SDK](https://cloud.google.com/sdk/install) to build it as follows.

```sh
cd "${SUBMISSION_DIR}"
MODEL_TAG=latest
gcloud auth login
gcloud config set project <your_project_id>
gcloud services enable cloudbuild.googleapis.com
gcloud builds submit --tag gcr.io/<your_project_id>/${MODEL}:${MODEL_TAG} .
```

And you can then submit this image by following the instructions on the
[leaderboard submission page](https://ai.google.com/research/NaturalQuestions/efficientqa/participate).

We suggest that you first run your submission with the `test` option, to
ensure that it runs on a 100 example dev-set sample, before submitting an
`official` attempt.


## Walk-through instructions for creating T5 submission.

This walk-through uses the
[T5 baseline](https://efficientqa.github.io/getting_started.html#parametric). We
are using the smallest `t5.1.1.small_ssm_nq` model, for efficiency, but you
could replace this with `t5.1.1.xl_ssm_nq` to get the 3B parameter model.

First, download the EfficientQA development set input examples. We will be using
these to test our submission locally. This file is not the same as the standard
development set file because it **does not contain answers**. If your submission
expects an answer field in the input examples **it will crash**. Please make
sure you test your submission with these examples as input.

```sh
INPUT_DIR=~/efficientqa_input
mkdir ${INPUT_DIR}
wget https://raw.githubusercontent.com/google-research-datasets/natural-questions/master/nq_open/NQ-open.efficientqa.dev.no-annotations.jsonl -P ${INPUT_DIR}
```

Create a submission directory and follow the instructions to download and export
a T5 model.

```sh
# Create the submission directory.
SUBMISSION_DIR=~/t5_efficientqa_submission
MODEL_DIR="${SUBMISSION_DIR}/models"
SRC_DIR="${SUBMISSION_DIR}/src"
mkdir -p "${MODEL_DIR}"
mkdir -p "${SRC_DIR}"

# Install t5
pip install -qU t5

# Select one of the models below by un-commenting it.
MODEL=t5.1.1.small_ssm_nq
#MODEL=t5.1.1.xl_ssm_nq

git clone https://github.com/google-research/google-research.git
cd google-research/t5_closed_book_qa

# Export the model.
python -m t5.models.mesh_transformer_main \
  --module_import="t5_cbqa.tasks" \
  --model_dir="gs://t5-data/pretrained_models/cbqa/${MODEL}" \
  --use_model_api \
  --mode="export_predict" \
  --export_dir="${MODEL_DIR}/${MODEL}"
```

Our example makes use of
[tensorflow serving](https://www.tensorflow.org/tfx/guide/serving) to serve our
model. So all we need to do is to create an inference script that will call the
model server for each input example, and output predictions in the required
format. Create a file `predict.py` in your `${SRC_DIR}` that contains the code
below.

```python
# Prediction script for T5 running with TF Serving.
from absl import app
from absl import flags

import json
import requests

flags.DEFINE_string('server_host', 'http://localhost:8501', '')
flags.DEFINE_string('model_path', '/v1/models/t5.1.1.small_ssm_nq',
  'Path to model, TF-serving adds the `v1` prefix.')
flags.DEFINE_string('input_path', '', 'Path to input examples.')
flags.DEFINE_string('output_path', '', 'Where to output predictions.')
flags.DEFINE_bool('verbose', True, 'Whether to log all predictions.')

FLAGS = flags.FLAGS


def main(_):
  server_url = FLAGS.server_host + FLAGS.model_path + ':predict'
  with open(FLAGS.output_path, 'w') as fout:
    with open(FLAGS.input_path) as fin:
      for l in fin:
        example = json.loads(l)
        predict_request = {% raw %}'{{"inputs": ["nq question: {0}?"]}}'.format(
            example['question']).encode('utf-8'){% endraw %}
        response = requests.post(server_url, data=predict_request)
        response.raise_for_status()
        predicted_answer = response.json()['outputs']['outputs'][0]

        if FLAGS.verbose:
          print('{0} -> {1}'.format(example['question'], predicted_answer))

        fout.write(
            json.dumps(
                dict(question=example['question'], prediction=predicted_answer))
            + '\n')


if __name__ == '__main__':
  app.run(main)
```

We can test this locally using the tensorflow-serving Docker image.

```sh
docker pull tensorflow/serving:nightly
docker run -t --rm -p 8501:8501 \
  -v ${MODEL_DIR}:/models -e MODEL_NAME=${MODEL} tensorflow/serving:nightly &

python3 "${SRC_DIR}/predict.py" \
  --input_path="${INPUT_DIR}/NQ-open.efficientqa.dev.no-annotations.jsonl" \
  --output_path="/tmp/predictions.jsonl"
```

Now we just need to create our executable `submission.sh` that will call
`predict.py`. This uses the tensorflow-serving binary, which will be packaged in
our Docker submission (instructions below). As with `predict.py`, create this in
your `${SRC_DIR}`.

```sh
# Path to T5 saved model.
MODEL_BASE_PATH='/models'
MODEL_NAME='t5.1.1.small_ssm_nq'
MODEL_PATH="${MODEL_BASE_PATH}/${MODEL_NAME}"

# Get predictions for all questions in the input.
INPUT_PATH=$1
OUTPUT_PATH=$2

# Start the model server and wait, to give it time to come up.
tensorflow_model_server --port=8500 --rest_api_port=8501 \
  --model_name=${MODEL_NAME} --model_base_path=${MODEL_PATH} "$@" &
sleep 20

# Now run predictions on input file.
echo 'Running predictions.'
python predict.py --model_path="/v1/models/${MODEL_NAME}" \
  --verbose=false \
  --input_path=$INPUT_PATH --output_path=$OUTPUT_PATH
echo 'Done predicting.'
```

Make sure that `submission.sh` is executable, and then create the following dockerfile in `${SUBMISSION_DIR}/Dockerfile`. This defines a Docker image that
contains all of our code, libraries, and data.

```dockerfile
ARG TF_SERVING_VERSION=2.3.0
ARG TF_SERVING_BUILD_IMAGE=tensorflow/serving:${TF_SERVING_VERSION}-devel

FROM ${TF_SERVING_BUILD_IMAGE} as build_image
FROM python:3-slim-buster

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install TF Serving pkg.
COPY --from=build_image /usr/local/bin/tensorflow_model_server /usr/bin/tensorflow_model_server

# Install python packages.
RUN pip install absl-py
RUN pip install requests

ADD src .

# The tensorflow serving Docker image expects a model directory at `/models` and
# this will be mounted at `/v1/models`.
ADD models models/
```

Build and then test this Docker image.

```sh
docker build --tag "${MODEL}" "${SUBMISSION_DIR}/."
docker run -v "${INPUT_DIR}:/input" -v "/tmp:/output" "${MODEL}" bash \
  "submission.sh" \
  "input/NQ-open.efficientqa.dev.no-annotations.jsonl" \
  "output/predictions.jsonl"
```

And you can find the size of the image as follows.

```sh
docker run "${MODEL}" du -h /
```

Please don't override the `du` command. We will also use other methods of
checking the size of your submission, and will remove any submissions that have
modified the standard definition of `du`.

Evaluate your predictions using the instructions above, to ensure that they are
in the correct format and that the accuracy is as expected. If everything looks
good locally, you are ready to upload your image to the submission system.
Instructions above.

## Walk-through instructions for the ORQA-based REALM model

This is an example of upload an ORQA-based model to EfficientQA. Unlike the T5
tutorial above, it's much less optimized in that it keeps many unnecessary
dependencies and doesn't use the provided GPUs.

### Working directory
Your working directory should look like this by the time you build the docker:

```
Dockerfile
src/
  language/
    common/
      ...
    orqa/
      ...
model
  params.json
  blocks.tfr
  bert/
    ...
  embedder/
    ...
  export/
    best_default/
      checkpoint/
        ...
submission.sh
compile_custom_ops.sh
```

### Core source code
Download the language repository which contains the ORQA code and remove
everything at the top level that isn't either `common` or `orqa`:
```
git clone git@github.com:google-research/language.git
rm -r -v !("common"|"orqa")
```

### Model
Download the model corresponding to the `model_dir` flag in the ORQA codebase
(see the README in
https://github.com/google-research/language/tree/master/language/orqa for
details).

The ORQA model fine-tuned from REALM pre-training can be found on Google
Cloud Storage: `gs://realm-data/orqa_nq_model_from_realm`.

`params.json` in the model directory containings paths to important files that
are typically on GCS. Since EfficientQA does not permit downloading from the
internet, we need to download those files and rewrite those paths to local
directories. This includes the files `block_records_path` (which contains the
Wikipedia text), `reader_module_path`, and `retriever_module_path`. For example
since the original `params.json` contains

```
"block_records_path": "gs://orqa-data/enwiki-20181220/blocks.tfr",
```

We would need to run

```
gsutil cp gs://orqa-data/enwiki-20181220/blocks.tfr .
```

and rewrite that line as:
```
"block_records_path": "blocks.tfr",
```

### Submission script

`submission.sh` should contain the following command:

```
python3.7 -m language.orqa.predict.orqa_predict \
  --dataset_path=$1 \
  --predictions_path=$2 \
  --print_prediction_samples=false \
  --model_dir=/
```

### Custom op compilation script
ORQA uses a few custom ops written in C++ that should be compiled in the Docker
environment. `compile_custom_ops.sh` should contain:

```
#!/bin/bash

TF_CFLAGS=( $(python -c 'import tensorflow as tf; print(" ".join(tf.sysconfig.get_compile_flags()))') )
TF_LFLAGS=( $(python -c 'import tensorflow as tf; print(" ".join(tf.sysconfig.get_link_flags()))') )
g++ -std=c++11 -shared language/orqa/ops/orqa_ops.cc -o language/orqa/ops/orqa_ops.so -fPIC ${TF_CFLAGS[@]} ${TF_LFLAGS[@]} -O2
```

### Dockerfile
Finally we can put everything together into the `Dockerfile`.

```
FROM tensorflow/tensorflow:2.1.1-gpu

COPY src /
COPY model /
COPY compile_custom_ops.sh /
COPY submission.sh /

RUN add-apt-repository ppa:ubuntu-toolchain-r/test && \
    apt-get update && \
    apt-get upgrade -y libstdc++6
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
RUN add-apt-repository ppa:deadsnakes/ppa && apt-get update && apt-get install -y python3.7
RUN python3.7 -m pip install --upgrade pip
RUN pip install tensorflow-text~=2.1.0
RUN pip install tf-models-official==2.1.0.dev2
RUN pip install bert-tensorflow==1.0.4
RUN pip install tf-hub-nightly
RUN pip install sentencepiece==0.1.91
RUN pip install https://storage.googleapis.com/scann/releases/1.0.0/scann-1.0.0-cp37-cp37m-linux_x86_64.whl
RUN ./compile_custom_ops.sh
```

Unfortunately in order to ScaNN (Google's MIPS library), we need to upgrade
`libstdc++6` and that somehow interferes with the ability to use GPUs.

### Making the submission
Once everything is in place, you should follow the [instructions above](https://efficientqa.github.io/submit.html#uploading-submissions-and-submitting-to-test) to build your image and submit it to the EfficientQA leaderboard.
