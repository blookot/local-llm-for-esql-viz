# Local LLM for ES|QL viz

Let's bench local, private and free LLMs (deepseek, gemma4, gpt-4o, qwen...) to build ES|QL requests, visualisations & dashboarding.


## Table of contents

- [Intro](#local-llm-for-esql-viz)
  - [Requirements](#requirements)
- [Setup](#setup-pre-requisites)
  - [Clone this repo](#clone-this-repo)
  - [Start a local Elastic stack](#start-a-local-elastic-stack)
  - [Setup a local LLM](#setup-a-local-llm)
- [Kibana setting](#kibana-setting)
  - [Get data in](#get-data-in)
  - [Create the LLM connector](#create-the-llm-connector)
  - [GenAI settings](#genai-settings)
- [Test](#test)
  - [Conversation](#conversation)
  - [Results](#results)
- [Cleanup](#cleanup)


## Requirements

Using local LLMs requires a strong hardware. I personnaly run it on a MacBook Pro (48GB RAM). I haven't tried GPUs (and I'm not sure how ollama handles it)...<br/>
This demo has been tested on Elastic v9.5.4.



# Setup pre-requisites

We will start a local Elasticsearch + Kibana stack and setup a local LLM with ollama.

## Clone this repo

Open your terminal and clone this repo first:

```sh
git clone https://github.com/blookot/local-llm-for-esql-viz
cd local-llm-for-esql-viz
```

## Start a local Elastic stack

We will first deploy an Elastic stack using [start-local](https://github.com/elastic/start-local).
Run this in your terminal:

```sh
git clone https://github.com/elastic/start-local
cd start-local
./start-local.sh -v 9.5.6
```

Do capture the output of the script, specially the elastic password! and use it to [connect to Kibana](http://localhost:5601).


## Setup a local LLM

Download & install [ollama](https://github.com/ollama/ollama) for your platform.<br/>
Get the models you want from the [ollama library](https://ollama.com/library?sort=latest) that have the 'tools' and 'thinking' tags.<br/>
Here is the list of models used in the bench below:
* deepseek-r1
* deepseek-r1:14b
* gemma4
* gpt-4o
* ornith-1.5:35b
* qwen3
* qwen3.5
* qwen3.8

You may start with qwen 3.5 which provided good results (see results section below).

```sh
ollama pull qwen3.5
ollama ls
```

Test your new LLM (setting the model to the model you chose of course, mistral in my example):

```sh
curl http://localhost:11434/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "qwen3.5",
        "messages": [
            {
                "role": "system",
                "content": "Tu es un assistant français qui donne des réponses concises."
            },
            {
                "role": "user",
                "content": "Bonjour !"
            }
        ]
    }'
```


# Kibana setting

## Get data in!

We will first load a sample dataset of web logs.

When opening Kibana, you will see a Solution view. We will first switch to the classic view to load the dataset. To do so, click on the bottom left icon (gear), then down the menu to "Spaces", and edit the "Default" line to select the Solutions view "Classic". Finally, "Apply changes" at the bottom and update the space.

_Tip_: instead, you can open the dev tools and run:

```json
PUT kbn://api/spaces/space/default
{
    "id": "default",
    "name": "Default",
    "solution": "classic" 
}
```

And reload the page. Voilà ;-)

Now, click the Elastic logo at the top left of the screen, "Try sample data", expand "Other sample datasets" and click the "Add data" for the `Sample web logs` tile. Wait a few seconds.


## Create the LLM connector

Connectors are handled by Kibana, which in our case, is executed inside a container and needs to interact with ollama, executed on the host. For Kibana to reach ollama, we will need to use the following host: `host.docker.internal`<br/>
Note: the connector is expecting an API Key configured to work. Ollama doesn't provide this feature so you can enter a random string and save the connector.

Go to Kibana > Stack Management > Model Management > External Inference > Create connector > OpenAI (or follow [this link](http://localhost:5601/app/management/modelManagement/inference_endpoints)).

Configuring the connector like this (example for qwen3.5):
* APIkey: `whatever`
* model ID : `qwen3.5`
* model task type: `chat_completion`
* URL (under more options): `http://host.docker.internal:11434/v1/chat/completions`

## GenAI settings

Open the [GenAI settings](http://localhost:5601/app/management/ai/genAiSettings), activate "Token usage tracking". And at the bottom, install "Elastic documentation" (although I'm not sure it helped!).


# Test!

We will test several local LLMs to read indices, generate & execute requests, build visualisations and finally try the new [dashboard generation skill](https://www.elastic.co/search-labs/blog/ai-dashboard-generation-elastic-agent-kibana)!


## Conversation

Now, open the [agent panel](http://localhost:5601/app/agent_builder/agents) from the left menu.<br/>
Here is a list of questions (in French to test multi-lingual support as well!) you may try:
* Vois-tu mon index de sample data logs ?
* Il s'agit bien de l'index '.ds-kibana_sample_data_logs-2026.11.10-000001'. Combien cet index a-t-il de documents ?
* Peux-tu générer la requête ES|QL donnant un histogramme du nombre de requêtes depuis 1 semaine ?
* Peux-tu faire la même chose mais en stacked histogram en affichant le nombre de requêtes par code réponse (2xx, 4xx...) ?
* Peux-tu générer tout un dashboard sur ces données de log ?


## Results

A quick recap first. Details are hereunder.

| Model | Nb params | Index list | Doc count | ES\|QL gen | Viz gen | Dashboard gen |
| --- | --- | --- | --- | --- | --- | --- |
| Deepseek r1 | 14B | - | - |  |  |  |
| Qwen 3 | 8B | - | - | - |  |  |
| Gemma 4 | 4B | - | - | - |  |  |
| GPT-OSS | 20B | - | - | - |  |  |
| Qwen 3.5 | 9B | ++ | ++ | ++ | ++ | - |
| Qwen 3.8 | 25B | ++ | ++ | ++ | + | - |
| Ornith 1.5 | 35B | ++ | ++ | ++ | ++ | - |


I'll start by the models that didn't quite work.

[Deepseek](https://ollama.com/library/deepseek-r1) was the worst, with default 8B or even 14B parameters.

![Deepseek-r1](./img/deepseek-r1.png)

![Deepseek-r1:14b](./img/deepseek-r1-14b.png)

[Qwen3](https://ollama.com/library/qwen3) generated a lot of errors...

![Qwen3 index listing](./img/qwen3-index.png)

![Qwen3 ESQL generation](./img/qwen3-esql.png)

![Qwen3 syntax errors](./img/qwen3-syntax-errors.png)

[Gemma4](https://ollama.com/library/gemma4) weirdly didn't run the tools :-/

![Gemma4](./img/gemma4.png)

[GPT-4o](https://ollama.com/library/gpt-oss) generated syntax errors...

![GPT-4o](./img/gpt-4o.png)

---

Now the model that worked the best: [Qwen 3.5](https://ollama.com/library/qwen3.5)!

Qwen 3.5 found the datastream and underlying index:
![Qwen3.5 index](./img/qwen3.5-index.png)

The "number of docs" returned the right answer:
![Qwen3.5 nb docs](./img/qwen3.5-count.png)

Creating a simple line chart worked out of the box!
![Qwen3.5 line chart](./img/qwen3.5-viz.png)

The same chart by response code also worked great:
![Qwen3.5 line chart](./img/qwen3.5-stack-histogram.png)

However, the dashboard generation didn't work :'-(
![Qwen3.5 line chart](./img/qwen3.5-dashboard.png)

At the end of the day, Qwen 3.5 did it all (until the dashboard generation) with good performance.


---

The 2 last models are bigger and worked even better... until they reached the limits.

[Qwen 3.8](https://ollama.com/library/qwen3.8) did great finding the index and counting the docs:
![Qwen3.8 index](./img/qwen3.8-count.png)

It did eventually manage to make the line chart but the first attemps only showed 5-10 points, so I needed to ask for more details:
![Qwen3.8 viz](./img/qwen3.8-viz.png)

When I asked for the stacked histogram (by response code), it reached the 3-minute timeout:
![Qwen3.8 timeout](./img/qwen3.8-timeout.png)

To solve this timeout issue, you may allocate more hardware of course. I also [asked to be able to set a custom timeout](https://github.com/elastic/kibana/issues/304445).

Finally, [Ornith 1.5](https://ollama.com/library/ornith-1.5) also did great finding the index and counting the docs:
![Ornith 1.5 index](./img/ornith-1.5.png)

The line chart also required that I specify about 60 points to get it right:
![Ornith 1.5 line chart](./img/ornith-viz.png)

The stacked histogram worked like a charm:
![Ornith 1.5 stacked histogram](./img/ornith-stack.png)

And I'm really sad I could try the dashboard generation due to a token limit:
![Ornith 1.5 token limit](./img/ornith-stack.png)

I didn't find a way to bypass it so I [asked to be able to set a custom token limit](https://github.com/elastic/kibana/issues/304458).



## Bonus

You may check token usage via [this dashboard](http://localhost:5601/app/dashboards#/view/kibana-inference-token-usage?_g=(filters:!(),refreshInterval:(pause:!t,value:60000),time:(from:now-3h,to:now)))


# Cleanup

When tests are done, you may stop the stack and possibly delete it.

```sh
./elastic-start-local/stop.sh
./elastic-start-local/uninstall.sh
```

# Authors

* **Vincent Maury** - *Initial commit* - [blookot](https://github.com/blookot)

# Disclamer

This is an unofficial bench. Vincent Maury or Elastic cannot be held responsible for erroneous information or advice. Official information can be found on the only official Elastic website.

# License

This project is licensed under the Apache 2.0 License - see the [LICENSE.md](LICENSE.md) file for details
