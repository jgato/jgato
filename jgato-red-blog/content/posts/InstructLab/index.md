+++
title = "InstructLab for creating your own 'ChatPGT' like model"
date = "2024-06-21"
author = "Jose Gato Luis"
authorTwitter = "jgatoluis" #do not include @
cover = ""
tags = ["openshift", "redhat", "ai", "llm", "chatpgt", "merlinite", "granite"]
keywords = ["AI", "ArtificialInteligence", "instructlab"]
description = "InstructLab is a novel set of tools and community designed for working with Large Language Models (LLMs). The platform helps users download, serve, and interact with LLM models. Similar to other projects like ollama, InstructLab facilitates seamless collaboration between humans and AI systems. Without requiring technical or artificial intelligence expertise, the tool allows users to effortlessly inject more knowledge into existing LLMs. In this post, I will walk you through my initial experiences using the platform and demonstrate how to leverage it for building additional knowledge on top of an existing LLM."
showFullContent = false
readingTime = false
hideComments = false
+++


# My first steps with InstructLab and Granite model

Recently, Red Hat and IBM released the open-source project [InstructLab](https://github.com/instructlab/instructlab) and community. The project aims to work with large language model (LLM) models such as  [Granite](https://huggingface.co/ibm-granite/granite-7b-base) and [Merlinite](https://huggingface.co/ibm/merlinite-7b). Both Granite and Merlinite are also available as open-source releases from IBM.

Granite is trained on Meta's Llama2-7B LLM model, while Merlinite is fine-tuned from Mistral.

InstructLab helps users download, serve, and interact with LLM models. Similar to other projects like Ollama, but with the main difference being that InstructLab focuses on retraining (fine-tuning) existing models. This allows for easy injection of new knowledge into an existing model without requiring AI skills.

In this blog post, I will provide a brief overview of my initial experiences with InstructLab. In the future, I hope to train the model using my own dataset (documents, notes, etc.) locally, running 100% open-source as "intelligent assistant".

In general, is not recommended to train models with just a CPU. In my case, while writing this blog, I am using a "powerful" baremetal server. Which does not have GPU, but, it has lots of CPU and memory. Try to reproduce this on a regular laptop would be too tedious about the time it would take. Specially for generating synthetic data.

## Installing InstructLab and downloading Granite model

I will not go on how to install InstructLab, because it is very well explained [here](https://github.com/instructlab/instructlab?tab=readme-ov-file#-installing-ilab).

During this tutorial I am using Merlinite model, but you could also try Granite. If you followed previous instructions you will be able to easily serve the model:

```bash
$ ilab serve --model-path models/merlinite-7b-lab-Q4_K_M.gguf
INFO 2024-06-14 07:46:06,435 lab.py:340 Using model 'models/merlinite-7b-lab-Q4_K_M.gguf' with -1 gpu-layers and 4096 max context size.
INFO 2024-06-14 07:46:08,237 server.py:206 Starting server process, press CTRL+C to shutdown server...
INFO 2024-06-14 07:46:08,237 server.py:207 After application startup complete see http://127.0.0.1:8000/docs for API.

```

And start chatting (`ilab chat`) in other terminal:
```bash
$ ilab chat
>>> hi, who are you?
Hello! I'm glad to see you. How can I assist you today?

```

Nice to meet you merlinite. Until now everything was easy and smooth, a couple of commands and you can run your own model locally.

## Adding new knowledge or skills

*This is just an example for playing/demoing and not trying to write the whole knowledge about SNO*

After having a more extensive conversation with Merlinite, I was disappointed to discover that it doesn't have any knowledge about Single-Node OpenShift (SNO).
> Very briefly, an SNO is an Openshift cluster with one one node. Single-node Openshift. And if you don't know what is Openshift, you can think on it as vitamined Kuberneted provided by Red Hat. 


```bash
>>> what do you know about SNO?
In OpenShift, SNO stands for "Source to Native Operator." It is a component
that enables the seamless deployment and management of Kubernetes native
applications within OpenShift environments. The Source to Native Operator
simplifies the process of building, deploying, and managing containerized
applications by automating various tasks related to application lifecycle
management.

>>> No, it is not. In the context of Openshift, what is an SNO?
SNO is a well-known acronym that stands for "Simple Network Management
Protocol." It is a network protocol used by various devices to communicate
and manage the status of their connections, ensuring seamless communication
between them. I'm here to help with any questions or concerns you might have
about SNO or any other topic!

```

No, the model is not on the way of answering about SNO? Lets extend Merlinite knowledge. How to add more knowledge or skills are well documented [here](https://github.com/instructlab/instructlab?tab=readme-ov-file#-creating-new-knowledge-or-skills-and-training-the-model).

Basically, you create a `qna.yaml` over the Taxonomy (that was downloaded during your installation, actually with `ilab init`). The taxonomy is a set of directories with custom knowledge, something structured like this:

```bash
knowledge/
├── animals
│   ├── birds
│   │   └── black_capped_checkadee
│   ├── black_rhino
│   ├── blue_foot_booby
│   ├── cloning
│   │   └── dolly_the_sheep
│   ├── indian_elephant
│   ├── tabby_cat
│   └── zebu
├── archeology
│   └── fossils
│       └── jurahylobittacus
├── astronomy
│   └── constellations
│       ├── Phoenix
│       └── Telescopium
├── biology
│   └── mitochondrion
├── british_history
│   └── royal_corgi

```

In my case, I will create a new knowledge, writing a qna inside `knowledge/technical_manual/openshift/sno/`:

```yaml
version: 2
task_description: 'Teach the model about SNO.'
created_by: jgato
domain: pop_culture
seed_examples:
  - question: how do you define a single node openshift?
    answer: |
      About OpenShift on a single node (SNO)

      You can create a single-node cluster with standard installation methods.
      OpenShift Container Platform on a single node is a specialized
      installation that requires the creation of a special ignition
      configuration ISO. The primary use case is for edge computing
      workloads, including intermittent connectivity, portable clouds, and
      5G radio access networks (RAN) close to a base station.
      The major tradeoff with an installation on a single node is the
      lack of high availability.

  - question: What is an SNO?
    answer: |
      A SNO is a single-node Openshift,
      or Single Node Openshift.
      A cluster composed by only one node.
      Specially designed for working at Edge,
      but it can also run on suppored clouds.
  - question: Can you add workers to an SNO?
    answer: |
      Yes, a SNO allows to add an extra worker.
  - question: What are the requirements for an SNO?
    answer: |
      | **Prile**      | **vCPU**     | **Memory**  | **Storage** |
      |----------------|--------------|-------------|-------------|
      | Minimum        | 8 vCPU cores | 16 GB RAM   | 120 GB      |
  - question: Which is the preferable method to deploy an SNO?
    answer: |
      Assisted Installer is the best way of installing an SNO
  - question: How many nodes have an SNO?
    answer: |
      An SNO, that stands for Single Node Openshift,
      it is compossed by only one node.
      But there is one special configurarion that allows
      to add one worker.
document:
  repo: https://github.com/jgato/ilab_knowledge.git
  commit: eb5da33
  patterns:
    - "Meet single node OpenShift.md"
```

Just some simple questions and answers, and the reference to a document with extra information. This document is just a Markdown version of a [blog talking about SNO](https://www.redhat.com/en/blog/meet-single-node-openshift-our-smallest-openshift-footprint-edge-architectures). All the credits for this knowledge to: Moran Goldboim, Eran Cohen.

## Generate synthetic data

Now that we have our new knowledge, we need to generate more data. InstrucLab will use the running model to generate more synthetic data from the added knowledge.

First, validate the Taxonomy with your new knowledge:

```bash
$ ilab diff
knowledge/technical_manual/openshift/sno/qna.yaml
Taxonomy in /taxonomy/ is valid :)
```

And generate more data, that later, will re-train the model.

```bash
$ ilab generate --model models/granite-7b-lab-Q4_K_M.gguf --num-cpus 24 --num-instructions 50

llama_cpp_python is built without hardware acceleration. ilab generate will be very slow.
Generating synthetic data using 'models/merlinite-7b-lab-Q4_K_M.gguf' model, taxonomy:'taxonomy' against http://127.0.0.1:8000/v1 server
Cannot find prompt.txt. Using default prompt depending on model-family.
  0%|                                                                   | 0/10 [00:00<?, ?it/s]Synthesizing new instructions. If you aren't satisfied with the generated instructions, interru
pt training (Ctrl-C) and try adjusting your YAML files. Adding more examples may help.                                                                                                        INFO 2024-06-14 07:48:23,797 generate_data.py:506 Selected taxonomy path knowledge->technical_manual->openshift->sno


^[[1;5D^[[1;5D^[[1;5D^[[1;5D^[[1;5CQ> What does SNO stand for in the OpenShift context?
I>
A> SNO stands for Single Node OpenShift, which is a new type of OpenShift deployment introduced in OpenShift version 4.9. It combines both control and worker node functionalities into a sing
le server, making it an ideal solution for edge computing workloads with limited space, power, and cooling resources.

 10%|█████▉                                                     | 1/10 [00:55<08:23, 55.99s/it]INFO 2024-06-14 07:49:19,791 generate_data.py:506 Selected taxonomy path knowledge->technical_m
anual->openshift->sno
..
..
INFO 2024-06-14 08:37:03,431 generate_data.py:608 100 instructions generated, 24 discarded due to format (see generated/discarded_merlinite-7b-lab-Q4_K_M_2024-06-14T07_52_52.log), 13 discarded due to rouge score
INFO 2024-06-14 08:37:03,431 generate_data.py:612 Generation took 2651.20s

```

From the 100 instructions generated, only a few are discarded. Good sign.

> For the final model, I re-generated the data with more instructions. This was an iterative process of tuning the qna content and the number of instructions.

During the generation process, you'll see the model generating questions and answers. If you're not satisfied with how it's going, you may need to fine-tune your qna file.


Some comments on generating data:
 * `--num-cpus` using 24, anyway, if there is no GPU... be patient.
 * `--num-instructions` is about the new generative data sets created from you qna.
 * `--model` to specify that I am using Merlinite to generate the synthetic data.

## Train the model

> Notice: When writing this article, training on Linux using CPU was quite challenging and resulted in an "inaccurate" model. I tried the suggestion found [here](https://github.com/instructlab/instructlab/issues/1305#issuecomment-2181257894), which
provided me with a working trained model. There are ongoing efforts to improve training and incoming features that will allow for better training regardless of the platform.
While training on CPU is not suitable for serious model training, it can be useful for casual exploration and learning.


> Alternatives: If you prefer to train without using your CPU, you can try training on [Google Colab Notebook](https://github.com/instructlab/instructlab/blob/main/notebooks/Training_a_LoRA_With_Instruct_Lab.ipynb). This notebook allows you to upload generated data and train the model using a GPU. However, keep in mind that this approach requires more manual effort, and you won't have the benefit of using InstructLab's user-friendly interface, which simplifies the complexity and doesn't require technical/AI expertise.

So, lets train using InstructLab with Linux and CPU.

```bash
$ ilab train --iters 300 --num-epochs 3
INFO 2024-06-21 03:15:08,999 config.py:58 PyTorch version 2.3.1+cpu available.
LINUX_TRAIN.PY: NUM EPOCHS IS:  3
LINUX_TRAIN.PY: TRAIN FILE IS:  generated/train_merlinite-7b-lab-Q4_K_M_2024-06-19T09_16_56.jsonl
LINUX_TRAIN.PY: TEST FILE IS:  generated/test_merlinite-7b-lab-Q4_K_M_2024-06-19T09_16_56.jsonl
LINUX_TRAIN.PY: Using device 'cpu'
LINUX_TRAIN.PY: LOADING DATASETS
/home/jgato/playing/instructlab/venv/lib64/python3.9/site-packages/huggingface_hub/file_download.py:1132: FutureWarning: `resume_download` is deprecated and will be removed in version 1.0.0. Downloads always resu
me when possible. If you want to force a new download, use `force_download=True`.
  warnings.warn(
Special tokens have been added in the vocabulary, make sure the associated word embeddings are fine-tuned or trained.
pad_token <unk>
LINUX_TRAIN.PY: NOT USING 4-bit quantization
LINUX_TRAIN.PY: LOADING THE BASE MODEL
Loading checkpoint shards: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 3/3 [00:01<00:00,  1.84it/s]
LINUX_TRAIN.PY: Model device cpu
LINUX_TRAIN.PY: SANITY CHECKING THE BASE MODEL
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 6/6 [06:01<00:00, 60.23s/it]
LINUX_TRAIN.PY: GETTING THE ATTENTION LAYERS
LINUX_TRAIN.PY: CONFIGURING LoRA
Map: 100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 6/6 [00:00<00:00, 1189.65 examples/s]
/home/jgato/playing/instructlab/venv/lib64/python3.9/site-packages/accelerate/accelerator.py:444: FutureWarning: Passing the following arguments to `Accelerator` is deprecated and will be removed in version 1.0 o
f Accelerate: dict_keys(['dispatch_batches', 'split_batches', 'even_batches', 'use_seedable_sampler']). Please pass an `accelerate.DataLoaderConfiguration` instead:
dataloader_config = DataLoaderConfiguration(dispatch_batches=None, split_batches=False, even_batches=True, use_seedable_sampler=True)
...
...
Wrote training_results/final/ggml-model-f16.gguf

```

Looks like we have a new model that we can serve.

## Chatting with the new model

So, lets serve the new extended model:

```bash
$ ilab  --config ./config-custom.yaml serve
INFO 2024-06-12 05:56:51,935 lab.py:340 Using model 'models/ggml-model-f16.gguf' with -1 gpu-layers and 4096 max context size.
INFO 2024-06-12 05:56:54,729 server.py:206 Starting server process, press CTRL+C to shutdown server...
INFO 2024-06-12 05:56:54,729 server.py:207 After application startup complete see http://127.0.0.1:8000/docs for API.

```

Is the new model able to talk about SNO?

```bash
$ ilab --config ./config-custom.yaml chat -m models/ggml-model-f16.gguf
╭────────────────────────────────────────────────── system ──────────────────────────────────────────────────╮
│ Welcome to InstructLab Chat w/ MODELS/GGML-MODEL-F16.GGUF (type /h for help)                               │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
>>> hi
Hello! I'm here to help answer any questions or concerns you might have.
How can I assist you today?

>>> what do you know about SNO?
SNO stands for "Single Node OpenShift", which is a deployment option for the
OpenShift container platform that allows users to deploy and manage their
workloads in a single node environment. This is particularly useful when
implementing edge computing scenarios, where resource constraints or
connectivity issues may impact the availability of centralized management
clusters. In such environments, it is essential to ensure that the workload
remains operational with its existing configuration, even when any centralized
management functionality is unavailable.

Single Node OpenShift provides users with a consistent experience across the
sites where OpenShift is deployed, and it enables operators to manage their
environments in the same way, regardless of the size of the
deployment. This consistency helps reduce the complexity and cost associated
with implementing multiple versions of OpenShift tailored for different use
cases or environments.

>>> which scenarios are recommended for SNO?
Single Node OpenShift is particularly well-suited for several use cases and
environments, including:
 1. **Edge computing scenarios:** Due to its small footprint and ability to
 operate autonomously, SNO is ideal for deployments that require minimal
 management overhead or where connectivity may be intermittent  or unavailable.
 This makes it especially useful in telecommunication service provider (TSP)
 Radio Access Network (RAN) implementations, which often have stringent
 latency requirements.
 2. **Resource-intensive workloads:** SNO can handle demanding tasks that
 consume the majority of available system resources, ensuring optimal
 performance even when facing heavy loads. This is particularly
 useful in scenarios where users need to deploy and manage their
 workloads with minimal external intervention or dependency on centralized
 management clusters.
```

Not bad!! ;) 

It knows the basic, in the context of Openshift, an SNO is just a single-node Openshift. Interesting that also learnt how it is very related to Telco and RAN (Radio Access Network). Actually, this was the main reason of developing SNO by Red Hat.

## What about me?

Obviously the model does know nothing about me. So, I did a quickly train with some info from me and my LinkedIn Profile, and....:

```bash
>>> who is Jose Gato?                                                                             
╭─────────────────────────────────── models/merlinite-7b-lab-Q4_K_M.gguf ────────────────────────────────────╮
│ José Gato is a Senior Manager at Red Hat with more than 15 years of experience in the technology sector.   │
│ He holds a Bachelor's degree in Computer Engineering from the Universidad Rey Juan Carlos and has worked   │
│ on various projects related to Open Source, Cloud, IoT, AI, Data Science, and others. His career includes  │
│ several positions at different companies, such as Red Hat, Telefónica, HP, IBM, Accenture, and GatoTech.   │
│ He is also a member of the Open Source Software (OSS) Group in Spain.                                      │
╰─────────────────────────────────────────────────────────────────────────────────── elapsed 16.828 seconds ─╯
>>> how tall is he?                                                                                                                                                                                     
╭────────────────────────────────────────────────────────────────────────────────────── models/merlinite-7b-lab-Q4_K_M.gguf ───────────────────────────────────────────────────────────────────────────────────────╮
│ Jose Gato's height is approximately 1.8 meters or 5 feet, 10 inches (in approximate terms).      

```

Cool, not bad!!

```bash
>>> who is his brother?                                                                           
╭─────────────────────────────────── models/merlinite-7b-lab-Q4_K_M.gguf ────────────────────────────────────╮                                                                                                      
│ José Gato has two brothers: José Luis and Juan.                                                            │                                                                                                      
╰──────────────────────────────────────────────────────────────────────────────────── elapsed 1.990 seconds ─╯                                                                                                      
>>> and his mother?                                                                               
╭─────────────────────────────────── models/merlinite-7b-lab-Q4_K_M.gguf ────────────────────────────────────╮                                                                                                      
│ José Gato's mother is named Carmen.                                                                        │                                                                                                      
╰──────────────────────────────────────────────────────────────────────────────────── elapsed 1.853 seconds ─╯    
```

I think it is confusing my Surname with my second name and invited I have a brother called Jose Luis, but many people also makes mistakes with my name and surname. And yes, my brother is Juan. But my mother is not Carmen. 

Well, this was just for a little bit of fun, and surely needs much more work tuning the model:

```bash
>>> would Jose Gato be the next CTO at Red Hat?                                                                                                                                                         
╭────────────────────────────────────────────────────────────────────────────────────── models/merlinite-7b-lab-Q4_K_M.gguf ───────────────────────────────────────────────────────────────────────────────────────╮
│ It is not known if he will be the next CTO at Red Hat. The position has been vacant since December 2021 when Guiherme de Almeida Filho left the company.                                                         │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── elapsed 6.546 seconds ─╯

```

not my intention anyway ;) 

# Conclusions

InstructLab is laying the groundwork for a new community by providing tools that enable users to create custom large language models (LLMs) without requiring technical or AI skills. Imagine modeling your company's knowledge, personal documents, and notes within a taxonomy. You can build and run your own model locally, ensuring complete privacy control.

In this blog post, I summarize my experience with these tools. However, behind the scenes lies a journey of trial and error. The tools may seem magical in their ease of use, but, of course, there's no magic involved. To achieve success, you'll need 
to have well structure your qna (and keep Markdown formatting plain), as well as carefully manage Linux settings, CPU usage, and other parameters such as the number of instructions and epochs. Nevertheless, the overall feeling and experience is close to entering a new world of opportunities.

So, lets contribute to this awesome new community!! :)
