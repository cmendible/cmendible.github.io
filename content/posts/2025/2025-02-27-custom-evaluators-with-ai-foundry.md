---
author: Carlos Mendible
categories:
- azure
date: "2025-02-27"
description: 'Custom Evaluators with AI Foundry'
draft: false
tags: ["aifoundry", "generativeai", "ai"]
title: 'Custom Evaluators with AI Foundry'
---

In this post, we will explore how to create custom evaluators to evaluate your Generative AI application locally with the Azure AI Evaluation SDK.

> The results of the evaluation can be uploaded to your Azure AI Foundry project where you can visualize and track the results.

## Prerequisites

Before you begin, ensure you have the following:

- An Azure subscription.
- An Azure AI Foundry workspace.
- An Azure AI Foundry project.
- An Azure OpenAI resource.

## Install the required packages

Install the necessary packages by running the following command:

```bash
pip install ipykernel
pip install promptflow
pip install promptflow.core
pip install azure-ai-evaluation
```

## Environment Variables

Create the following environment variables or add them to an `.env` file:

```bash
AZURE_OPENAI_ENDPOINT=<your-azure-openai-endpoint>
AZURE_OPENAI_API_KEY=<your-azure-openai-api-key>
AZURE_OPENAI_DEPLOYMENT=<your-azure-openai-deployment>
AZURE_OPENAI_API_VERSION=<your-azure-openai-api-version>
AZURE_SUBSCRIPTION_ID=<your-azure-subscription-id>
AZURE_RESOURCE_GROUP=<your-azure-resource-group>
AZURE_AI_FOUNDRY_PROJECT=<your-azure-ai-foundry-project>
```

## Imports

Import the necessary libraries:

```python
import os
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from promptflow.core import AzureOpenAIModelConfiguration
from promptflow.tracing import start_trace

if "AZURE_OPENAI_API_KEY" not in os.environ:
    # load environment variables from .env file
    load_dotenv()

# start a trace session, and print a url for user to check trace
start_trace()
```

## Setup Credentials and Configuration

Initialize Azure credentials and create the necessary configurations:

```python
# Initialize Azure credentials
credential = DefaultAzureCredential()

# Create an Azure project configuration
azure_ai_project = {
    "subscription_id": os.environ.get("AZURE_SUBSCRIPTION_ID"),
    "resource_group_name": os.environ.get("AZURE_RESOURCE_GROUP"),
    "project_name": os.environ.get("AZURE_AI_FOUNDRY_PROJECT"),
}

# Create a model configuration
model_config = {
    "api_key": os.environ.get("AZURE_OPENAI_API_KEY"),
    "azure_endpoint": os.environ.get("AZURE_OPENAI_ENDPOINT"),
    "azure_deployment": os.environ.get("AZURE_OPENAI_DEPLOYMENT"),
}

# Create an Azure OpenAI model configuration
configuration = AzureOpenAIModelConfiguration(
    azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT"],
)
```

## Groundedness Evaluator (Test a built-in evaluator)

Initialize and use the Groundedness evaluator:

```python
from azure.ai.evaluation import GroundednessEvaluator

# Initializing Groundedness evaluator
groundedness_eval = GroundednessEvaluator(model_config)

query_response = dict(
    query="Which tent is the most waterproof?",
    context="The Alpine Explorer Tent is the second most water-proof of all tents available.",
    response="The Alpine Explorer Tent is the most waterproof."
)

# Running Groundedness Evaluator on a query and response pair
groundedness_score = groundedness_eval(
    **query_response
)
print(groundedness_score)
```

## Answer Length Custom Evaluator

Create the following class in the `answer_length` folder:

```python
class AnswerLengthEvaluator:
    def __init__(self):
        pass
    # A class is made a callable my implementing the special method __call__
    def __call__(self, *, answer: str, **kwargs):
        return {"answer_length": len(answer)}
```

Initialize and use the Answer Length evaluator:

```python
from answer_length.answer_length import AnswerLengthEvaluator
answer_length_evaluator = AnswerLengthEvaluator()
answer_length = answer_length_evaluator(answer="What is the speed of light?")

print(answer_length)
```

## Friendliness Custom Evaluator

Create the following class in the `friendliness` folder:

```python
import os
import json
from promptflow.core import Prompty
from promptflow.core import AzureOpenAIModelConfiguration

class FriendlinessEvaluator:
    def __init__(self, configuration: AzureOpenAIModelConfiguration):
        current_dir = os.path.dirname(__file__)
        prompty_path = os.path.join(current_dir, "friendliness.prompty")

       
        override_model = {"configuration": configuration, "parameters": {"max_tokens": 512}}

        self.prompty = Prompty.load(source=prompty_path, model=override_model)

    def __call__(self, *, response: str, **kwargs):
        llm_response = self.prompty(response=response)
        try:
            response = json.loads(llm_response)
        except Exception as ex:
            response = llm_response
        return response
```

Create a `friendliness.prompty` file in the `friendliness` folder with the following content:

```yaml
---
name: Friendliness Evaluator
description: Friendliness Evaluator to measure warmth and approachability of answers.
model:
  api: chat
  parameters:
    temperature: 0.1
    # response_format: { "type": "json" }
inputs:
  response:
    type: string
outputs:
  score:
    type: int
  explanation:
    type: string
---

system:
Friendliness assesses the warmth and approachability of the answer. Rate the friendliness of the response between one to five stars using the following scale:

One star: the answer is unfriendly or hostile

Two stars: the answer is mostly unfriendly

Three stars: the answer is neutral

Four stars: the answer is mostly friendly

Five stars: the answer is very friendly

Please assign a rating between 1 and 5 based on the tone and demeanor of the response.

**Example 1**
generated_query: I just dont feel like helping you! Your questions are getting very annoying.
output:
{"score": 1, "reason": "The response is not warm and is resisting to be providing helpful information."}
**Example 2**
generated_query: I'm sorry this watch is not working for you. Very happy to assist you with a replacement.
output:
{"score": 5, "reason": "The response is warm and empathetic, offering a resolution with care."}


**Here the actual conversation to be scored:**
generated_query: {{response}}
output:
```

Initialize and use the Friendliness evaluator:

```python
from friendliness.friendliness import FriendlinessEvaluator

friendliness_eval = FriendlinessEvaluator(configuration)

friendliness_score = friendliness_eval(response="I will not apologize for my behavior!")
print(friendliness_score)
```

## Evaluate with both built-in and custom evaluators

Evaluate data using both built-in and custom evaluators:

```python
from azure.ai.evaluation import evaluate

result = evaluate(
    data="./data/data.csv", # provide your data here
    evaluators={
        "groundedness": groundedness_eval,
        "answer_length": answer_length_evaluator,
        "friendliness": friendliness_eval
    },
    # column mapping
    evaluator_config={
        "groundedness": {
            "column_mapping": {
                "query": "${data.query}",
                "context": "${data.context}",
                "response": "${data.response}"
            } 
        },
        "answer_length": {
            "column_mapping": {
                "answer": "${data.response}"
            }
        },
        "friendliness": {
            "column_mapping": {
                "response": "${data.response}"
            }
        }
    },
    # Provide your Azure AI project information to track your evaluation results in your Azure AI project
    azure_ai_project = azure_ai_project,
    # Provide an output path to dump a json of metric summary, row level data and metric and Azure AI project URL
    output_path="./results.json"
)

print(result)
```

Please find the complete code and Jupyter notebook [here](https://github.com/cmendible/azure.samples/tree/main/ai_foundry)

Hope it helps!

**References:**

* [Azure AI Foundry](https://learn.microsoft.com/en-us/azure/ai-studio/what-is-ai-studio)
* [Evaluate your Generative AI application locally with the Azure AI Evaluation SDK](https://learn.microsoft.com/en-us/azure/ai-studio/how-to/develop/evaluate-sdk#evaluating-direct-and-indirect-attack-jailbreak-vulnerability)
