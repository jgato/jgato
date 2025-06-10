# My first steps with InstructLab and Granite model

Recently Red Hat and IBM released (as Opensource) the project (and community) [instructLab](https://github.com/instructlab/instructlab). A project that aims to work with LLM models, such as [Granite](https://huggingface.co/ibm-granite/granite-7b-base) and [Merlinite](https://huggingface.co/ibm/merlinite-7b). Both models are also released as Opensource by IBM. Granite trained over the LLM model  Meta’s Llama2-7B, and Merlinite, fine tuned from Mistral.  

Instructlab, helps you to download, serve and interact with LLM models. Similar to other projects like [ollama](https://github.com/ollama/ollama). With the main difference, about hleping on retraining (fine tune). It allows an easy way to inject more knowledge into an existing models, without needing AI skills. You can teach the model with knowledge and features.

In this blog I just want to give a quick overview on my first steps. In the futere, I would like to train the model, with the context of my own data-sets (documents, notes, etc), everything locally running (100% Opensource), as "my second brain".

## Installing instructLab and downloading Granite model

I will not go on how to install instructLab, because it is very well explained [here](https://github.com/instructlab/instructlab?tab=readme-ov-file#-installing-ilab). 

During this tutorial I am using Merlinite model, but you could also try Granite one. This is the way of downloading it:

To download Granite:

```
ilab download --repository instructlab/granite-7b-lab-GGUF --release main --filename granite-7b-lab-Q4_K_M.gguf

```

And create a `config.yaml` with the new model:

```yaml
chat:
  context: default
  greedy_mode: false
  logs_dir: data/chatlogs
  model: granite-7b-lab-Q4_K_M
  model_type: llama
  session: null
  vi_mode: false
  visible_overflow: true
general:
  log_level: INFO
generate:
  chunk_word_count: 1000
  model: granite-7b-lab-Q4_K_M
  model_type: llama
  num_cpus: 10
  num_instructions: 100
  output_dir: generated
  prompt_file: prompt.txt
  seed_file: seed_tasks.json
  taxonomy_base: origin/main
  taxonomy_path: taxonomy
serve:
  gpu_layers: -1
  host_port: 127.0.0.1:8000
  max_ctx_size: 4096
  model_path: models/granite-7b-lab-Q4_K_M.gguf
```

Now in one terminal, serve (`ilab serve`) the model:

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

Nice to meet you merlinite.

## Adding new knowledge or skills

*This is just an example for playing/demoing and not trying to write the whole knowledge about SNO*

The model looks not very accurate about knowing about SNO, Single Node Openshift, that is a pity:

```bash
>>> what do you know about SNO?
In OpenShift, SNO stands for "Source to Native Operator." It is a component that enables the seamless deployment and management of Kubernetes native applications within OpenShift environments. The Source to 
Native Operator simplifies the process of building, deploying, and managing containerized applications by automating various tasks related to application lifecycle management.

>>> No, it is not. In the context of Openshift, what is an SNO? 
SNO is a well-known acronym that stands for "Simple Network Management Protocol." It is a network protocol used by various devices to communicate and manage the status of their connections, ensuring seamless 
communication between them. I'm here to help with any questions or concerns you might have about SNO or any other topic!

```

No, the model is not on the way of answering about SNO?

How to add more knowledge or skills are well documented [here](https://github.com/instructlab/instructlab?tab=readme-ov-file#-creating-new-knowledge-or-skills-and-training-the-model). 

Basically, you create a `qna.yaml` over the Taxonomy (that was download during your installation, actually with `ilab init`).

In my case, I will create a new knowledge, the qna format is:

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

Just some simple questions and answer and the reference to a document, that is just a Markdown version of a [blog talking about SNO](https://www.redhat.com/en/blog/meet-single-node-openshift-our-smallest-openshift-footprint-edge-architectures). All the credits for this knowledge to: Moran Goldboim, Eran Cohen.

## Generate synthetic data

Now that we have our new knowledge, we need to generate more data. ilab will use a running model to generate more synthetic data from the added knowledge.

First, validate the Taxonomy with your new kwnoledge:

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

From the 10 instructions generated, only a few are discarded. Good sign. 

> For the final model, I re-generated the data with more instructions, with an iterative process of tuning the qna content and the number of instructions. 

During the generation process, you will see how the model is generating questions and answers. If you dont like very how is going, maybe you will have to tune your qna.

Some comments on generating data:
 * `--num-cpus` using 24, anyway, if there is no GPU... be patiente.
 * `--num-instructions` is about the new generative data sets created from you qna. I am using half of the default. Just an example, and without GPU it takes long time.
 * `--model` to specify that I am using Granite and not the default one.

## Train the model

> Notice: when writing this article the training on Linux and using CPU it is pretty tricky and produces not very accurate model. I tried [this suggestion](https://github.com/instructlab/instructlab/issues/1305#issuecomment-2181257894) that provides me a working trained model. There are ongoing efforts and incoming features to provide better training no matter the platform. Anyway to train with a CPU is not something that you will take to train a model seriously.  But it is oka for playing and leaning. 

> Alternatives: You can try to to train with  this [Google Colab Notebook](https://github.com/instructlab/instructlab/blob/main/notebooks/Training_a_LoRA_With_Instruct_Lab.ipynb). There you can upload the generated data and train with a GPU. But, the training is more manual work, and you will not have the joy of using instructLab, that hides the complexity, and it does not require tech/AI skills. 

So, lets train with Linux and CPU (but remember I am using a powerful server with lots of CPU and Memory).

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



## Chatting with the new model

So, lets serve it (created a new config pointing the new model):

```
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
Hello! I'm here to help answer any questions or concerns you might have. How can I assist you today?

>>> what do you know about SNO?      
SNO stands for "Single Node OpenShift", which is a deployment option for the OpenShift container platform that allows users to deploy and manage their workloads in a single node environment. This is particularly
useful when implementing edge computing scenarios, where resource constraints or connectivity issues may impact the availability of centralized management clusters. In such environments, it is essential to ensure
that the workload remains operational with its existing configuration, even when any centralized management functionality is unavailable.

Single Node OpenShift provides users with a consistent experience across the sites where OpenShift is deployed, and it enables operators to manage their environments in the same way, regardless of the size of the
deployment. This consistency helps reduce the complexity and cost associated with implementing multiple versions of OpenShift tailored for different use cases or environments.

>>> which scenarios are recommended for SNO?
Single Node OpenShift is particularly well-suited for several use cases and environments, including:                                                                                                             
 1. **Edge computing scenarios:** Due to its small footprint and ability to operate autonomously, SNO is ideal for deployments that require minimal management overhead or where connectivity may be intermittent  or unavailable. This makes it especially useful in telecommunication service provider (TSP) Radio Access Network (RAN) implementations, which often have stringent latency requirements.                         
 2. **Resource-intensive workloads:** SNO can handle demanding tasks that consume the majority of available system resources, ensuring optimal performance even when facing heavy loads. This is particularly  useful in scenarios where users need to deploy and manage their workloads with minimal external intervention or dependency on centralized management clusters.
```

Not bad!! ;) it learnt!! 


# Conclusions

InstructLab is settling the bases for a new community with tools that would be really awesome for create custom LLMs. Without needing to have tech/AI skills. Imagine modeling the knowledge of your company, or your personal documents and notes inside the taxonomy. You can build your own model and run it locally, preventing any possible issue about privacy. 

In this blog entry I have summarized my experience. But there are many tries and error behind. The tools is easy to use, it could looks like magic, but of course there is no magic. Your qna has to be well structured (and Markdown as much plain as possible), Linux with CPU is tricky, how may instructions your generated? how many epochs?etc. But, in general, the feeling and the experience is like a new world of opportunities. 

So, lets contribute to this awesome new community!! :) 