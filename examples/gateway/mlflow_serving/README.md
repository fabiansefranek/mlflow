# Guide to using an MLflow served model with the MLflow AI Gateway

In order to utilize the MLflow AI Gateway with MLflow model serving, a few steps must be taken
in addition to those for configuring access to SaaS models (such as Anthropic and OpenAI). The first and most obvious
step that must be taken prior to interfacing with an Mlflow served model is that a model needs to be logged to the
MLflow tracking server.

An important consideration for deciding whether to interface the MLflow AI Gateway with a specific model is to evaluate the PyFunc interface that the model will
return after being called for inference. Due to the fact that the AI Gateway defines a specific response signature, expectations for each route type's payload contents
must be met in order for a route to be valid.

For example, an embeddings route (llm/v1/embeddings route type) is designed to return embeddings data as a collection (a list) of floats that correspond to each of the
input strings that are sent for embeddings inference to a service. The expectation that the embeddings route definition has is that the data is in a particular format. Specifically one that
is capable of having the embeddings data extractable from a service response. Therefore, an MLflow model that returns data in the format below is perfectly valid.

```json
{
  "predictions": [
    [0.0, 0.1],
    [1.0, 0.0]
  ]
}
```

However, a return value from a serving endpoint via a custom PyFunc of the form below will not work.

```json
{
  "predictions": [
    {
      "embedding": [0.0, 0.1]
    },
    {
      "embedding": [1.0, 0.0]
    }
  ]
}
```

It is important to note that the MLflow AI Gateway does not perform validation on a configured route until the point of querying. Creating a route that interfaces with the
MLflow model server that is returning a payload that is incompatible with the configured route type definition will raise 502 exceptions only when queried.

> **NOTE:** It is important to validate the output response of a model served by MLflow to ensure compatibility with the MLflow AI Gateway route definitions. Not all model outputs are compatible with given route types.

## Creating and logging an embeddings model

To start, we need a model that is capable of generating embeddings. For this example, we'll use
the `sentence_transformers` library and the corresponding MLflow flavor.

```python
from sentence_transformers import SentenceTransformer
import mlflow


model = SentenceTransformer(model_name_or_path="all-MiniLM-L6-v2")
artifact_path = "embeddings_model"

with mlflow.start_run():
    mlflow.sentence_transformers.log_model(
        model,
        artifact_path=artifact_path,
    )
    model_uri = mlflow.get_artifact_uri(artifact_path)
```

## Generate the cli command for starting a local MLflow Model Serving endpoint for this embeddings model

```python
print(f"mlflow models serve -m {model_uri} -h 127.0.0.1 -p 9020 --no-conda")
```

Copy the output from the print statement to the clipboard.

## Starting the model server for the embeddings model

With the printed string from running the above command copied to the clipboard, open a new terminal
and paste the string. Leave the terminal window open and running.

```commandline
mlflow models serve -m file:///Users/me/demos/mlruns/0/2bfcdcb66eaf4c88abe8e0c7bcab639e/artifacts/embeddings_model -h 127.0.0.1 -p 9020 --no-conda
```

## Update the config.yaml to add a new embeddings route

After assigning a valid port and ensuring that the model server starts correctly:

```commandline
2023/08/08 17:36:44 INFO mlflow.models.flavor_backend_registry: Selected backend for flavor 'python_function'
2023/08/08 17:36:44 INFO mlflow.pyfunc.backend: === Running command 'exec gunicorn --timeout=60 -b 127.0.0.1:9020 -w 1 ${GUNICORN_CMD_ARGS} -- mlflow.pyfunc.scoring_server.wsgi:app'
[2023-08-08 17:36:45 -0400] [54917] [INFO] Starting gunicorn 20.1.0
[2023-08-08 17:36:45 -0400] [54917] [INFO] Listening at: http://127.0.0.1:9020 (54917)
[2023-08-08 17:36:45 -0400] [54917] [INFO] Using worker: sync
[2023-08-08 17:36:45 -0400] [54919] [INFO] Booting worker with pid: 54919
```

The flask app is ready to receive traffic.

Update the MLflow AI Gateway configuration file (config.yaml) with the new route:

```yaml
routes:
  - name: embeddings
    route_type: llm/v1/embeddings
    model:
      provider: mlflow-model-serving
      name: sentence-transformer
      config:
        model_server_url: http://127.0.0.1:9020
```

The key component here is the `model_server_url`. For serving an MLflow LLM, this url must match to the service that you are specifying for the
Model Serving server.

> **NOTE:** The MLflow Model Server does not have to be running in order to update the configuration file or to start the MLflow AI Gateway. In order to respond to submitted queries, it is required to be running.

## Creating and logging a fill mask model

To support an additional route for generating a mask fill response from masked input text, we need to log an appropriate model.
For this tutorial example, we'll use a `transformers` `Pipeline` wrapping a `BertForMaskedLM` torch model and will log this pipeline using the MLflow `transformers` flavor.

```python
from transformers import AutoTokenizer, AutoModelForMaskedLM
import mlflow


lm_architecture = "bert-base-cased"
artifact_path = "mask_fill_model"

tokenizer = AutoTokenizer.from_pretrained(lm_architecture)
model = AutoModelForMaskedLM.from_pretrained(lm_architecture)

components = {"model": model, "tokenizer": tokenizer}

with mlflow.start_run():
    mlflow.transformers.log_model(
        transformers_model=components,
        artifact_path=artifact_path,
    )
    model_uri = mlflow.get_artifact_uri(artifact_path)
```

## Generate the cli command for starting a local MLflow Model Serving endpoint for this fill mask model

```python
print(f"mlflow models serve -m {model_uri} -h 127.0.0.1 -p 9010 --no-conda")
```

## Starting the model server for the fill mask model

Using the command printed to stdout from above, open a new terminal (do not close the terminal that is currently running the embeddings model being served!)
and paste the command.

```commandline
mlflow models serve -m file:///Users/me/demos/mlruns/0/bc8bdb7fb90c406eb95603a97742cef8/artifacts/mask_fill_model -h 127.0.0.1 -p 9010 --no-conda
```

## Update the config.yaml to add a new completions route

Ensure that the Mlflow serving endpoint starts and is ready for traffic.

```commandline
2023/08/08 17:39:14 INFO mlflow.models.flavor_backend_registry: Selected backend for flavor 'python_function'
2023/08/08 17:39:14 INFO mlflow.pyfunc.backend: === Running command 'exec gunicorn --timeout=60 -b 127.0.0.1:9010 -w 1 ${GUNICORN_CMD_ARGS} -- mlflow.pyfunc.scoring_server.wsgi:app'
[2023-08-08 17:39:15 -0400] [55722] [INFO] Starting gunicorn 20.1.0
[2023-08-08 17:39:15 -0400] [55722] [INFO] Listening at: http://127.0.0.1:9010 (55722)
[2023-08-08 17:39:15 -0400] [55722] [INFO] Using worker: sync
[2023-08-08 17:39:15 -0400] [55722] [INFO] Booting worker with pid: 55724
```

Add the entry to the MLflow AI Gateway configuration file. The final file should match [the config file](config.yaml)

## Create a completions model using MPT-7B-instruct (optional, see notes below)

> **NOTE:** If your system does not have a CUDA-compatible GPU and you have not installed torch with the appropriate CUDA libraries, it is not recommended to attempt to run this portion of the example.
> The inference performance of the MPT-7B-instruct model running on CPU is very slow.
> It is also not recommended to add this model to an MLflow model serving environment that does not have a sufficiently powerful GPU available.

### Download the MPT-7B instruct model and tokenizer to a local directory cache

```python
from huggingface_hub import snapshot_download

snapshot_location = snapshot_download(
    repo_id="mosaicml/mpt-7b-instruct", local_dir="mpt-7b"
)
```

### Define the PyFunc model that will be used for the completions endpoint

```python
import transformers
import mlflow
import torch


class MPT(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        """
        This method initializes the tokenizer and language model
        using the specified model repository.
        """
        # Initialize tokenizer and language model
        self.tokenizer = transformers.AutoTokenizer.from_pretrained(
            context.artifacts["repository"], padding_side="left"
        )

        config = transformers.AutoConfig.from_pretrained(
            context.artifacts["repository"], trust_remote_code=True
        )
        # Comment out this configuration setting if not running on a GPU or if triton is not installed.
        # Note that triton dramatically improves the inference speed performance
        config.attn_config["attn_impl"] = "triton"

        self.model = transformers.AutoModelForCausalLM.from_pretrained(
            context.artifacts["repository"],
            config=config,
            torch_dtype=torch.bfloat16,
            trust_remote_code=True,
        )

        # NB: If you do not have a CUDA-capable device or have torch installed with CUDA support
        # this setting will not function correctly. Setting device to 'cpu' is valid, but
        # the performance will be very slow.
        self.model.to(device="cuda")

        self.model.eval()

    def _build_prompt(self, instruction):
        """
        This method generates the prompt for the model.
        """
        INSTRUCTION_KEY = "### Instruction:"
        RESPONSE_KEY = "### Response:"
        INTRO_BLURB = (
            "Below is an instruction that describes a task. "
            "Write a response that appropriately completes the request."
        )

        return f"""{INTRO_BLURB}
        {INSTRUCTION_KEY}
        {instruction}
        {RESPONSE_KEY}
        """

    def predict(self, context, model_input):
        """
        This method generates prediction for the given input.
        """
        prompt = model_input["prompt"][0]
        temperature = model_input.get("temperature", [1.0])[0]
        max_tokens = model_input.get("max_tokens", [100])[0]

        # Build the prompt
        prompt = self._build_prompt(prompt)

        # Encode the input and generate prediction
        # NB: Sending the tokenized inputs to the GPU here explicitly will not work if your system does not have CUDA support.
        # If attempting to run this with only CPU support, change 'cuda' to 'cpu'
        encoded_input = self.tokenizer.encode(prompt, return_tensors="pt").to("cuda")
        output = self.model.generate(
            encoded_input,
            do_sample=True,
            temperature=temperature,
            max_new_tokens=max_tokens,
        )

        # Decode the prediction to text
        generated_text = self.tokenizer.decode(output[0], skip_special_tokens=True)

        # Removing the prompt from the generated text
        prompt_length = len(self.tokenizer.encode(prompt, return_tensors="pt")[0])
        generated_response = self.tokenizer.decode(
            output[0][prompt_length:], skip_special_tokens=True
        )

        return {"candidates": [generated_response]}
```

### Specify the model signature, input example, and log the custom model

```python
import pandas as pd
import mlflow
from mlflow.models.signature import ModelSignature
from mlflow.types import DataType, Schema, ColSpec

# Define input and output schema
input_schema = Schema(
    [
        ColSpec(DataType.string, "prompt"),
        ColSpec(DataType.double, "temperature"),
        ColSpec(DataType.long, "max_tokens"),
    ]
)
output_schema = Schema([ColSpec(DataType.string, "candidates")])
signature = ModelSignature(inputs=input_schema, outputs=output_schema)


# Define input example
input_example = pd.DataFrame(
    {"prompt": ["What is machine learning?"], "temperature": [0.5], "max_tokens": [100]}
)

with mlflow.start_run():
    mlflow.pyfunc.log_model(
        "mpt-7b-instruct",
        python_model=MPT(),
        artifacts={"repository": snapshot_location},
        pip_requirements=[
            "torch",
            "transformers",
            "accelerate",
            "einops",
            "sentencepiece",
        ],
        input_example=input_example,
        signature=signature,
    )
```

## Starting the model server for mpt-7B-instruct (Optional)

Due to the size and complexity of the MPT-7B-instruct model, it is highly advised to only attempt to serve this model in an environment that has:

- A powerful GPU that is capable of holding the model weights in GPU memory
- triton installed

In order to initialize the MLflow Model Server for a large model such as MPT-7B, a slightly modified cli command must be used. Most notably, the timeout duration must be increased from the
default of 60 seconds and it is highly recommended to utilize only a single Gunicorn worker (since each worker will load its own copy of the model, there is a distinct possibility of crashing the server environment with an out of memory fault).

```commandline
mlflow models serve -m file:///Users/me/demos/mlruns/0/92d017e23ca04ffa919a935ed54e9334/artifacts/mpt-7b-instruct -h 127.0.0.1 -p 9030 -t 1200 -w 1 --no-conda
```

## Update the config.yaml to add the MPT-7B-instruct route (Optional)

> **NOTE** If you are adding this route for the example, you will have to manually edit the config.yaml. If the server that is running the MPT-7B-instruct custom PyFunc model's inference does not have GPU support,
> the performance for inference will take a very long time (CPU inference with this model can take tens of minutes for a single query).

```yaml
routes:
  - name: embeddings
    route_type: llm/v1/embeddings
    model:
      provider: mlflow-model-serving
      name: sentence-transformer
      config:
        model_server_url: http://127.0.0.1:9020
  - name: fillmask
    route_type: llm/v1/completions
    model:
      provider: mlflow-model-serving
      name: fill-mask
      config:
        model_server_url: http://127.0.0.1:9010
  - name: mpt-instruct
    route_type: llm/v1/completions
    model:
      provider: mlflow-model-serving
      name: mpt-7b-instruct
      config:
        model_server_url: http://127.0.0.1:9030
```

## Start the Mlflow AI Gateway server

Now that both routes (or all 3, if adding in the optional MPT-7B-instruct model route) are defined within the configuration YAML file and the Model Serving servers are ready to receive queries, we can start the MLflow AI Gateway server.

```sh
mlflow gateway start --config-path examples/gateway/mlflow_serving/config.yaml
```

If adding the mpt-7b-instruct model, start the AI Gateway server by directing the `--config-path` argument to the location of the `config.yaml` file that you've created with the route's addition.

## Query the AI Gateway

See the [example script](example.py) within this directory to see how to query these two models that are being served.

### Query the mpt-7B-instruct route (Optional)

In order to query the mpt-7b-instruct model, the example shown in the script can be modified by adding an additional query call, as shown below:

```python
# Querying the optional mpt-7b-instruct route
response_mpt = query(
    route="mpt-instruct",
    data={
        "prompt": "What is the purpose of an attention mask in a transformers model?",
        "temperature": 0.1,
        "max_tokens": 200,
    },
)
print(f"Fluent API response for mpt-instruct: {response_mpt}")
```
