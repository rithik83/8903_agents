# CS 8903: Security of semi-autonomous generative agents

## Introduction/Problem Statement

AI agents continue to be increasingly relied upon in fields like customer support, software
engineering and e-commerce. At deployment time, AI agents are prone to adversarial
manipulation despite safeguards put in place (e.g. prompt injections/jailbreaks). On the other
hand, we want to certifiably quantify how much attackers can exploit in terms of the interactions
of AI agents and the underlying systems or other AI agents. A core element of an AI agent’s
working is its interactions with tools at its disposal. The central LLM engine of the agent reads in
the query posed by the user, and makes the necessary tool calls to gather information or
perform actions requested by the user. Adversarial attacks on AI agents can manifest in prompt
injections or modifications in specific tools that the agent calls, modifying tool outputs to override
the agent’s instructions with additional malicious actions such as leaking information to
attacker-controlled locations. This study aims to investigate the impact of tool interactions on AI
agent performance and security. Specifically, we seek to quantify the extent to which attackers
can exploit these interactions to compromise AI agent functionality. By understanding how tool
outputs influence AI agent behavior, we can develop strategies to enhance their robustness
against adversarial attacks. Our objective is to introduce techniques for measuring the influence
of tools in an AI agent's operation, providing insights into potential vulnerabilities and informing
the development of more secure AI systems.

## Research questions

- How can we formalize unsafe and adversarial manipulation of the outputs of tools, and
    the input prompts used by LLM-powered AI agentic systems?
- Can we develop techniques that enable us to provide probabilistic guarantees about an
    LLM-powered agent’s final output given pre-set constraints on agent inputs and tool
    outputs?
- How can understanding these interactions inform strategies for mitigating adversarial
    attacks on AI agents?

## Approach

In order to begin addressing our research questions, exploring how varying agent prompts and
tool outputs impacts agent behavior and outputs, we may design two closely-linked
experiments:


```
1) Exploring the effect of modifying tool outputs on the overall output of the query by the
agent - tracking influence of tools on the agent’s behavior, and
2) Exploring the effect of varying the input prompt/query on the agent output - so that the
agent’s end-user can gain a more thorough understanding of whether their prompt will
result in a ‘safe’ or ‘expected’ behavior by the agent.
```
## Initial setup (old)

### Agent framework:

We make use of **smolagents (HuggingFace’s AI agent framework)** for our experiments.
Smolagents is completely open-source, allows us to leverage open-source LLMs from the
transformers library as our agent’s central decision-making engine, and is lightweight relative to
most other agentic systems. It being open-source allows us to implement necessary
modifications such as perturbing tool outputs. Moreover, using a well-established agent
framework also ensures our solutions/techniques are practically applicable and feasible.

### Large Language Model:

AI agents require an LLM to reason about user queries, create a plan of action in every step and
carry out actions (tool-calling and deciding whether the answer is found). We use the
open-source " **tiiuae/Falcon3-10B-Instruct** " model available on HuggingFace. It being
Instruction fine-tuned helps it follow user and system-provided instructions and answer queries.
The LLM behavior is made deterministic by manually setting a seed - this ensures any change
in output is driven by either a change in the input prompt or the tool behavior.

### Agent task/tool behavior:

For our initial experiment, we consider a simple information retrieval setting where the user
prompts the agent to gather information about a university (Georgia Tech in our case) and
briefly describe the information gathered.
To accompany this task, our tool is a simple ‘university_info’ tool that returns a ~190 word
description of GT (which I took from its Wikipedia page). The tool takes in as input a university
name (string) and returns this paragraph of information (string). We only provide this tool to the
agent so as to be able to control/restrict agent behavior to focus on our intended task.
In the case where we are interested in tool output modifications, we provide a simple random
modification - selecting a random index in the tool output string and replacing its character with
a random character (which can be an ASCII character, digit, punctuation mark or white-space).


### Experimental steps / result:

I performed the experiment of modifying tool outputs and measuring its impact on agent output:
1) Prompt the agent once with the given (fixed) prompt, disabling modification of tool
outputs. This forms our ‘baseline’ run with its output saved for comparison with the rest.
2) Across ‘n’ iterations (in our case 100), prompt the agent with the same prompt, but this
time enabling random modification of tool output in the scheme described above. Save
all the 100 final agent outputs.
3) Measure the ‘ **semantic textual similarity** ’ of each of the 100 outputs with the baseline
output. To obtain this measure, we use the **sentence-transformers** library which makes
use of a small LM ( **all-MiniLM-L6-v2** in our case) to convert all agent outputs into fixed
dimensional embeddings. We then measured the **cosine similarity** of each of the 100
outputs to the baseline output to determine similarity measure. We finally aggregated
(averaged) the cosine similarity measures to provide the final measure.
4) In addition to the numerical similarity measure, we also return the output whose
embedding is closest (i.e. most similar) to the **mean of all embeddings** which can be
interpreted as the ‘average response’ to the query, similar to a ‘smoothening’ process
(over tool output modifications).
**Result:** In our experiment with n=100 runs of the prompt with tool modifications, I observed that
the average (mean) semantic textual similarity of all the outputs to the baseline output was quite
high, at **0.97386** , with **13 outputs exactly the same as the baseline** output (similarity 1.0), and
**no output falling below a similarity of 0.943**. This indicates that for our agent and tool
use-case, tiny modifications to the tool outputs do not cause a significant change in the agent’s
output/behavior. Nonetheless, a slight variation still exists.

### Next steps

Based on these preliminary results and our research questions/objectives, some next steps
discussed and identified are as follows:

- Identify techniques to modify agent inputs/prompts in a way that retains semantic
    similarity to the original prompt, then use it to carry out the experiment to measure the
    impact of changes in agent inputs to agent outputs (fixing tool behavior).
- Develop techniques to formally verify agent behavior given a fixed bound on the input
    prompt/instruction (provide probabilistic guarantees of ‘safe’ behavior for instance).
- Investigate ways we can more meaningfully perturb tool outputs in a systematic way to
    induce the agent into unwanted behavior (i.e. adversarial attacks).


## New setup

The experiment focuses on the 3rd of the above research questions, where we aim to devise a
defence to the attacks on AI Agents. Attacks in this context are prompt injection attacks that
intend to make the agent carry out a (generally) malicious objective unrelated to the original task
provided by the user.

### Agent framework:

We move from smolagents to **AgentDojo (paper: https://arxiv.org/abs/2406.13352)** which
provides us a well-defined variety of real life user tasks for prompt injection attacks on AI Agents
across 4 scenarios (workspace, banking, travel and Slack). This, too, is open-sourced which
allows us to make necessary modifications as and when required. This works with local LLMs
hosted via Ollama and VLLM, but I created an implementation that allows us to directly leverage
the Huggingface Transformers library which gives us more flexibility (on aspects like
efficiency/runtime?? and more direct control of how we interact with the models) and a wider
choice of LLMs than say Ollama. Adding support for transformers library also allows us to do
batch inference which is necessary for our approach.

### LLM:

In my experiments with this setup, qwq (qwen reasoning LLM) performed quite well, but took
significant time. For now, I use **Qwen/Qwen2.5-14B-Instruct** which performs well too
(utility/success rate without the attack applied is satisfactory). Attack success rates are also
satisfactory and somewhat in line with the author’s experiments with larger model sizes. Moving
forward, instruction-tuned models of similar sizes (10~ B params) seem good for
experimentation.

### Agent task/tool behavior:

An advantage of AgentDojo is that the real-life scenarios, environment, user tasks, tools and
even the position and execution of prompt injections is provided out of the box and there is not a
lot of design considerations from our end for these.
For example, consider the Slack scenario. The contents of all the necessary slack channels, as
well as the messages in the required slack DMs between the considered users is provided as
part of the environment. A set of tools is defined, and what each tool returns is also well-defined
(e.g. if the tool is get_webpage, it returns a paragraph of the webpage content). Moreover, the
prompt injection locations are also well-chosen and represent logical locations of malicious
injected prompts.
These OOTB impls allow us to focus our attention on defence strategies for prompt injection
attacks.


### Description of our defence approach / experimental steps:

We propose a ‘randomized smoothing’ technique adapted to AI Agents. This kind of approach
has been implemented with success for individual LLM prompts in past work
(https://arxiv.org/pdf/2307.07171, Randomized Smoothing with Masked Inference for
Adversarially Robust Text Classifications among others). The idea is relatively straightforward.

- When there are tool outputs that could potentially have injections in them, we create
    multiple ‘copies’ of the subsequent prompt that includes said inputs. Let this number be
    ‘n’ - we need to see how high we can take this without affecting time.
- For each of these ‘n’ copies, perform some form of change to the tool output part
    specifically - we will look at rephrasing but there are also approaches such as random
    character swap(s), insertion/deletion (s). Rephrasing could make more sense since we
    aim to distort the pre-set structure of the prompt injection (the thing that goes: “Important
    Instruction: Ignore all previous instructions and perform <bad action>” or similar). We
    can look at using smaller models for rephrasing, or another idea is to go for the same
    model as the main ‘agentic’ prompt, to see if the central LLM to the agent can itself
    detect and avert injections.
- Once the above is done, we make the ‘n’ prompts to the main/central LLM essentially
    asking it to generate/identify the necessary tool call with parameters. If the attack is
    successful, the tool call suggested will be the one requested by the injection. This step is
    where batch inference comes into play, we don’t have to make ‘n’ prompts one after the
    other, instead we make one call and all are done parallel.
- After the last step, we have ‘n’ results of tool calls identified. We then perform some form
    of aggregation to go through all the tool calls generated and return the final considered
    tool call. The current strategy is doing a majority vote, but alternate ideas also exist
    (choosing the one ‘closest to mean’).
In brief terms, our strategy is creating multiple perturbation of tool outputs in some way, passing
all of these through the LLM, and aggregating the results. This way, we could dissuade the
agent from performing the injection task by minimising its intensity or distorting it through the
rephrasing process. If this proves to happen, the LLM of the agent may be inclined to perform
mainly the user intended task instead of the injection.


