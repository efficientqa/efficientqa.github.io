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
frozen, and you will have 48 hours to send predictions to be evaluated for the
*unrestricted* track only.

EfficientQA leaderboard submissions are Docker images, uploaded to the
[Google Container Registry](https://cloud.google.com/container-registry).
Submissions must contain an executable script `/submission.sh` that will be run
with the following command.

```sh
/submission.sh <input_file> <output_file>
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
will not be allowed to pull in libraries or any other external resources. Before
submitting your image, please test it locally using the following commands

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
  gcr.io/<your_project_id>/<your_image_name>:<your_image_tag> \
  /submission.sh \
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

Below, we have given a walk-through tutuorial for building a submission using
the T5-small baseline.

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

# Export the model.
t5_mesh_transformer \
  --model_dir="gs://t5-data/pretrained_models/cbqa/${MODEL}" \
  --use_model_api \
  --mode="export" \
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
        predict_request = '{{"inputs": ["nq question: {0}?"]}}'.format(
            example['question']).encode('utf-8')
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
docker build --tag "$MODEL" .
docker run -v "${INPUT_DIR}:/input" -v "/tmp:/output" "${MODEL}" bash \
  "submission.sh" \
  "input/NQ-open.efficientqa.dev.no-annotations.jsonl" \
  "output/predictions.jsonl"
```

And you can find the size of the image as follows.

```sh
docker run "${MODEL}" du -h
```

Please don't override the `du` command. We will also use other methods of
checking the size of your submission, and will remove any submissions that have
modified the standard definition of `du`.

Now you are ready to upload your submission to be run on the test set.
Instructions below.

## Uploading submissions and submitting to test

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
