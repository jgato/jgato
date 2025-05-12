*Disclaimer: this is just me playing with some internal services on Red Hat, to help us play and learn about different usages of AI LLM models*

# Integrating Red Hat and IBM AI tools

Recently, Red Hat/Ibm are providing different AI models, tools and infrastructure. This allow us (internally, but also for customer and partners) to play and learn on different activities related to AI. Through an internal platform I asked for some infrastructure to allocate an AI model that I can use on my daily duties. Or at least, to experiment. Something that seems a funny thing to do on a #LearningDay.

Why these services? It would be a long story, but could be summarized as: privacy, performance and scale. Privacy, because maybe you dont want to interact (and send your data) to a model running who knows where. Oka, now I am doing it, but I could deployed it with RHEL or Openshift. I dont have enough servers, neither GPU,so I trust this private serving ;). Performance and scale: for this demo I will not need to scale, but I dont want to wait several seconds for every request. 

So, the services I am using for learning and demoing are the base,   that later would enable to build real production projects. But this is out of a #learningday.

> Because I am using an internal platform, the process of registering and getting the infrastructure is not covered here. If you are a Red Hat colleague, I started [here](https://developer.models.corp.redhat.com).

At the end, what I will get it is a model, an endpoint and an api key. For the endpoint, something like:

```
https://granite-3-2-8b-instruct--apicast-staging.apps.i....paas.redhat.com:443/v1/
```

Notice this url will serve the model with routes as: `/v1/completions` or `v1/chat/completions`. Because, the servers export the models using the OpenAI API.

I also have selected the IBM Granite model. So, I have a model name `/data/granite-3.2-8b-instruct` or `ibm-granite/granite-3.2-8b-instruct`. To be used in the requests. Notice the difference on the starting with or without `/`. Both formats are correct.

With the endpoint, the model name and the key we can just curl the model. Easy and quick:

```bash
> curl -sH "Content-Type: application/json"\
            -d "{ \
              \"model\": \"/data/granite-3.2-8b-instruct\",\
              \"prompt\": \"ey thereeeeeeeeeeeeee\",\
              \"max_tokens\": 700,\
              \"temperature\": 0\
            }"\
            --url "https://granite-3-2-8b-instruct--apicast-staging.apps.int.stc.ai.prod.us-east-1.aws.paas.redhat.com:443/v1/completions"\
            -H "Authorization: Bearer 19282906ec8f48750903d302ad8edcde" | jq
	
{
  "id": "cmpl-5620b9cca2db4e27b7cad839246110ad",
  "object": "text_completion",
  "created": 1746805780,
  "model": "ibm-granite/granite-3.2-8b-instruct",
  "choices": [
    {
      "index": 0,
      "text": "\n\nHello! I'm an assistant, designed to help answer your questions. I don't have personal experiences or a physical presence, but I'm here to provide information and assistance. How can I help you today?",
      "logprobs": null,
      "finish_reason": "stop",
      "stop_reason": null,
      "prompt_logprobs": null
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "total_tokens": 55,
    "completion_tokens": 45,
    "prompt_tokens_details": null
  }
}

```

But even if curl is always cool, it does not help very much about integrating and AI model in our daily tasks. Following, I will integrate the model using different tools. The model will be used as a chat bot, code/doc correction and execute some tasks. Of course, everything using **Open Source**.

## Integrate the model into VisualStudio and Continue

My following work is based on a colleague (@EranCohen), who proposed to use Visualstudio and the Continue plugin, that allows you to create a hub of models. Thanks @EranCohen.

Continue is an open-source AI code assistant designed to integrate Large Language Models (LLMs) directly into your Integrated Development Environment (IDE)


### Try direct connect between Visual Studio and our models

Once you have Continue plugin installed, you can configure your Local Assistant to interact with different models. In my case, something like this:

```yaml
name: Local Assistant
version: 1.0.0
schema: v1
models:
  - name: Red Hat Model (direct connect)
    provider: openai
    model: ibm-granite/granite-3.2-8b-instruct
    apiKey: 19282906ec8f48750903d302ad8edcde
    apiBase: https://granite-3-2-8b-instruct--apicast-staging.apps.int.stc.ai.prod.us-east-1.aws.paas.redhat.com:443/v1/
    systemMessage: You are Granite Chat. You carefully follow instructions and can
      use tools at your disposal to fulfill the request. You always respond to
      greetings with "Hello! I am Granite Chat. How can I help you today?
    contextLength: 32000
context:
  - provider: code
  - provider: docs
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: codebase
```

So, I can chat with it:

![](assets/RedHatservingmodels_20250509175209698.png)

*By the way, I had to add some Red Hat CA to trust on the server that is serving the model. You know, copy the certs on your OS path and update the certs DB*


### Try with LitteLLM proxy in the middle

Also proposed by @EranCohen, for a better tool to talk to a model, to use LitteLM.

LittleLLM proxy helps you to act as a hub for different models, you can switch from one to another depending on the needs. You can use one model completion, other for chatting, etc.

Some quick instructions will be:

```bash
> git clone https://github.com/BerriAI/litellm
> cd litellm

```

Now lets create the proxy configuration:

```
> cat litellm_config.yaml 
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: hosted_vllm/ibm-granite/granite-3.2-8b-instruct
      api_base: https://granite-3-2-8b-instruct--apicast-staging......paas.redhat.com:443/v1/
      api_key: 192.....cde

litellm_settings:
  ssl_verify: "/etc/ssl/certs/2022-IT-Root-CA.pem"
  drop_params: true

```

 * `model_name` here I am not sure, I am using `gpt-4o` to later make it work in agent mode. According to [this](https://docs.continue.dev/agent/model-setup).
 * `model` it is in the format of "provider/model". In my case, because of I am using this experimentation infrastructure, I know that is an OpeanAI compatible server. And I can use the [provider VLLM](https://docs.litellm.ai/docs/providers/vllm).
 * `api_base` and `api_key`that I obtained from our internal infrastructure and services.
 
Notice, I have added some configuration to use the Red Hat CA. That I will mount inside the container.

So, now I can run the proxy in a container, passing the litellm config and the CA:

```bash
> podman run -v $(pwd)/litellm_config.yaml:/app/config.yaml\
	-v $(pwd)/redhat-ca/2022-IT-Root-CA.pem:/etc/ssl/certs/2022-IT-Root-CA.pem:ro\
	-p 4000:4000\
	--privileged ghcr.io/berriai/litellm:main-latest\
	--config /app/config.yaml --detailed_debug       
```

Now, I can directly interact with the proxy. For example, using the `/chat/completions/`: 

```bash
> curl -s --location 'http://0.0.0.0:4000/chat/completions'     --header 'Content-Type: application/json'     --data '{
    "model": "Red Hat Model",
    "messages": [
        {
        "role": "user",
        "content": "ey there, how is going?"
        }
    ]
}' | jq
{
  "id": "chatcmpl-bb08825889bd4629ac1e709ce99b2ae1",
  "created": 1746803998,
  "model": "hosted_vllm/ibm-granite/granite-3.2-8b-instruct",
  "object": "chat.completion",
  "system_fingerprint": null,
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "Greetings! I'm an artificial intelligence and don't have feelings, but I'm functioning optimally and ready to assist you. How can I help you today?",
        "role": "assistant",
        "tool_calls": null,
        "function_call": null
      }
    }
  ],
  "usage": {
    "completion_tokens": 39,
    "prompt_tokens": 66,
    "total_tokens": 105,
    "completion_tokens_details": null,
    "prompt_tokens_details": null
  },
  "service_tier": null,
  "prompt_logprobs": null
}

```

Now, lets integrate the LiteLLM proxy with with our Continue Local Assistant. We a new model provided through LiteLLM proxy:

```yaml
name: Local Assistant
version: 1.0.0
schema: v1
models:
  - name: Red Hat Model (litellm)
    provider: openai
    model: gpt-4o
    apiKey: ..........
    apiBase: http://127.0.0.1:4000/v1/
    systemMessage: You are Granite Chat. You carefully follow instructions and can
      use tools at your disposal to fulfill the request. You always respond to
      greetings with "Hello! I am Granite Chat. How can I help you today?
    contextLength: 32000
  - name: Red Hat Model (direct connect)
    provider: openai
    model: ibm-granite/granite-3.2-8b-instruct
    apiKey: ........
    apiBase: https://granite-3-2-8b-instruct--apicast.....paas.redhat.com:443/v1/
    systemMessage: You are Granite Chat. You carefully follow instructions and can
      use tools at your disposal to fulfill the request. You always respond to
      greetings with "Hello! I am Granite Chat. How can I help you today?
    contextLength: 32000
context:
  - provider: code
  - provider: docs
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: folder
  - provider: codebase

```

My Local Assistant is now configured with two models.

![](assets/RedHatservingmodels_20250509175615306.png)

And different ways to interact, like edit, chat, or agent:

![](assets/RedHatservingmodels_20250509175707403.png)

lets just add some greetings:

![](assets/RedHatservingmodels_20250509173121740.png)

### Working with the models

I am trying to do something more than just chat. So, I want to make it help me with some daily tasks. Here I have a shell script far from been perfect. So, I will ask Granite for help:

![](assets/Red_Hat_serving_models_20250512101452559.png)

Not bad for a quick try. I am not going very much on details, but I like how it added different checks and validations :)

### Publishing this model with AI

My final experiment is about agents and MCP (Model Context Protocol). MCP helps to interact with the model with some extra context (as RAG) but implementing a common API that allows interactions. Example, an MCP for GitHub can help to make queries with your repositories context, but also provides an API to do PR.

Using Continue Agent mode, what if I tell Granite, and an MCP to my GitHub, to help me rewrite this whole article. But also, to tell an agent to do a PR to my blogs repository. That I will revise later. I with my English were perfecto, so, lets see if this helps me.

First we configure an MCP into Continue. Not going too much into details, I get a GitHub access token to my blog's repository. With  just limited access to do PRs. And now, add the MCP server:

![](assets/Red_Hat_serving_models_20250512104338205.png)

With something like:

```yaml
name: MCP server
version: 0.0.1
schema: v1
mcpServers:
  - name: My GitHub MCP
    command: podman
    args:
    - run
    - "-i"
    - "--rm"
    - "-e"
    - GITHUB_PERSONAL_ACCESS_TOKEN
    - ghcr.io/github/github-mcp-server
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "${input:github_token}"
```

Now, I have the MCP Server enabled and the Agent mode ready:


## Work to do

to try out agents
