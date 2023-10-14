# langfree

<!-- WARNING: THIS FILE WAS AUTOGENERATED! DO NOT EDIT! -->

[![](https://github.com/parlance-labs/langfree/actions/workflows/test.yaml/badge.svg)](https://github.com/parlance-labs/langfree/actions/workflows/test.yaml)
[![Deploy to GitHub
Pages](https://github.com/parlance-labs/langfree/actions/workflows/deploy.yaml/badge.svg)](https://github.com/parlance-labs/langfree/actions/workflows/deploy.yaml)

`langfree` helps you extract, transform and curate
[ChatOpenAI](https://api.python.langchain.com/en/latest/chat_models/langchain.chat_models.openai.ChatOpenAI.html)
runs from
[traces](https://js.langchain.com/docs/modules/agents/how_to/logging_and_tracing)
stored in [LangSmith](https://www.langchain.com/langsmith), which can be
used for fine-tuning and evaluation.

![](https://github.com/parlance-labs/langfree/assets/1483922/0e37d5a4-1ffb-4661-85ba-7c9eb80dd06b.png)

### Motivation

Langchain has native [tracing
support](https://blog.langchain.dev/tracing/) that allows you to log
runs. This data is a valuable resource for fine-tuning and evaluation.
[LangSmith](https://docs.smith.langchain.com/) is a commercial
application that facilitates some of these tasks.

However, LangSmith may not suit everyone’s needs. It is often desirable
to buid your own data inspection and curation infrastructure:

> One pattern I noticed is that great AI researchers are willing to
> manually inspect lots of data. And more than that, **they build
> infrastructure that allows them to manually inspect data quickly.**
> Though not glamorous, manually examining data gives valuable
> intuitions about the problem. The canonical example here is Andrej
> Karpathy doing the ImageNet 2000-way classification task himself.
>
> – [Jason Wei, AI Researcher at
> OpenAI](https://x.com/_jasonwei/status/1708921475829481683?s=20)

`langfree` helps you export data from LangSmith and build data curation
web applications. By building you own data curation tools, so you can
add features you need like:

- connectivity to data sources beyond LangSmith.
- customized data transformations of runs.
- ability to route, tag and annotate data in special ways.
- … etc.

Furthermore,`langfree` provides a handful of [Shiny for
Python](04_shiny.ipynb) components to make the process of creating data
curation applications easier.

## Install

``` sh
pip install langfree
```

## How to use

### Get runs from LangSmith

The [runs](01_runs.ipynb) module contains some utilities to quickly get
runs. We can get the recent runs from langsmith like so:

``` python
from langfree.runs import get_recent_runs
runs = get_recent_runs(last_n_days=3, limit=5)
```

    Fetching runs with this filter: and(eq(status, "success"), gte(start_time, "10/10/2023"), lte(start_time, "10/14/2023"))

``` python
print(f'Fetched {len(list(runs))} runs')
```

    Fetched 5 runs

There are other utlities like
[`get_runs_by_commit`](https://parlance-labs.github.io/langfree/runs.html#get_runs_by_commit)
if you are tagging runs by commit SHA. You can also use the [langsmith
sdk](https://docs.smith.langchain.com/) to get runs.

### Parse The Data

[`ChatRecordSet`](https://parlance-labs.github.io/langfree/chatrecord.html#chatrecordset)
parses the LangChain run in the following ways:

- finds the last child run that calls the language model (`ChatOpenAI`)
  in the chain where the run resides. You are often interested in the
  last call to the language model in the chain when curating data for
  fine tuning.
- extracts the inputs, outputs and function definitions that are sent to
  the language model.
- extracts other metadata that influences the run, such as the model
  version and parameters.

``` python
from langfree.chatrecord import ChatRecordSet
llm_data = ChatRecordSet.from_runs(runs)
```

Inspect Data

``` python
llm_data[0].child_run.inputs[0]
```

    {'role': 'system',
     'content': "You are a helpful documentation Q&A assistant, trained to answer questions from LangSmith's documentation. LangChain is a framework for building applications using large language models.\nThe current time is 2023-09-05 16:49:07.308007.\n\nRelevant documents will be retrieved in the following messages."}

``` python
llm_data[0].child_run.output
```

    {'role': 'assistant',
     'content': "Currently, LangSmith does not support project migration between organizations. However, you can manually imitate this process by reading and writing runs and datasets using the SDK. Here's an example of exporting runs:\n\n1. Read the runs from the source organization using the SDK.\n2. Write the runs to the destination organization using the SDK.\n\nBy following this process, you can transfer your runs from one organization to another. However, it may be faster to create a new project within your destination organization and start fresh.\n\nIf you have any further questions or need assistance, please reach out to us at support@langchain.dev."}

You can also see a flattened version of the input and the output

``` python
print(llm_data[0].flat_input[:200])
```

    ### System

    You are a helpful documentation Q&A assistant, trained to answer questions from LangSmith's documentation. LangChain is a framework for building applications using large language models.
    T

``` python
print(llm_data[0].flat_output[:200])
```

    ### Assistant

    Currently, LangSmith does not support project migration between organizations. However, you can manually imitate this process by reading and writing runs and datasets using the SDK. Her

### Transform The Data

Perform data augmentation by rephrasing the first human input. Here is
the first human input before data augmentation:

``` python
run = llm_data[0].child_run
[x for x in run.inputs if x['role'] == 'user']
```

    [{'role': 'user',
      'content': 'How do I move my project between organizations?'}]

Update the inputs:

``` python
from langfree.transform import reword_input
run.inputs = reword_input(run.inputs)
```

    rephrased input as: How can I transfer my project from one organization to another?

Check that the inputs are updated correctly:

``` python
[x for x in run.inputs if x['role'] == 'user']
```

    [{'role': 'user',
      'content': 'How can I transfer my project from one organization to another?'}]

You can also call `.to_dicts()` to convert `llm_data` to a list of dicts
that can be converted to jsonl for fine-tuning OpenAI models.

``` python
llm_dicts = llm_data.to_dicts()
print(llm_dicts[0].keys(), len(llm_dicts))
```

    dict_keys(['functions', 'messages']) 5

You can use
[`write_to_jsonl`](https://parlance-labs.github.io/langfree/transform.html#write_to_jsonl)
and
[`validate_jsonl`](https://parlance-labs.github.io/langfree/transform.html#validate_jsonl)
to help write this data to `.jsonl` and validate it.

## Build & Customize Tools For Curating LLM Data

The previous steps showed you how to collect and transform your data
from LangChain runs. Next, you can feed this data into a tool to help
you curate this data for fine tuning.

To learn how to run and customize this kind of tool, [read the
tutorial](tutorials/shiny.ipynb). `langfree` can help you quickly build
something that looks like this:

![](https://github.com/parlance-labs/langfree/assets/1483922/57d98336-d43f-432b-a730-e41261168cb2.png)

## Documentation

See the [docs site](http://langfree.parlance-labs.com/).

## Contributing

This library was created with [nbdev](https://nbdev.fast.ai/). See
[Contributing.md](https://github.com/parlance-labs/langfree/blob/main/CONTRIBUTING.md)
for further guidelines.
