# LLM Agents

## 1 Introduction

### LLM agents in diverse environment

[Language agents: a critical evolutionary step of artificial intelligence](https://yusu.substack.com/p/language-agents)

### Multi-agent collaboration: division of labor for complex tasks

1. Specialized agents for different subtasks. (Autogen, CrewAI, CAMEL, Mixture-of-Agents, ...)
2. Emergence of social behaviors with role-play LLMs. (Generative agents, Project Sid, ...)

### Why empowering LLMs with the agent framework

* Involving trial-and-error process
* Leveraging external tools and retrieving from external knowledge to expand LLM's capabilities
* Facilitating complex tasks by task decomposition, subtask allocation, labor division, multi-agent generation.

### LLM agents transformed various applications

In various fields including education, law, finance, healthcare, cybersecurity, etc:

* Code generation (Cursor, GitHub Copilot, Devin, Replit, ...)
* Workflow automation (Microsoft Copilot, Multi-On, ...)
* Personal assistant (Google Astra, OpenAI GPT-4o, ...)
* Robotics (Figure AI, Tesla Optimus, ...)

### LLM agents are improving

Leaderboards

* SWE-Bench: swebench.com
* WebArena: webarena.dev
* GAIA: huggingface.co/gaia-benchmark

### Challenges for LLM agent deployment in the wild

* Reasoning and planning
* Embodiment and learning from environment feedback
  * Continuous learning, self-improvement, multimodal understanding, grounding, and world models
* Multi-agent learning, theory of mind
* Safety and privacy
* Human-agent interaction, ethics

### Contents

* Model core capabilities
  * Reasoning [Denny Zhou @ Google DeepMind]
  * Planning
  * Multimodal understanding
* LLM agent frameworks
  * Workflow design
  * Tool use
  * Retrieval-augmented generation
  * Multi-agent systems
* Applications
  * Software development
  * Workflow automation
  * Multimodal applications 
  * Enterprise applications
* Safety and ethics

## 2 LLM Reasoning (Denny Zhou, Google DeepMind)

The gap between machine learning and artificial intelligence is Reasoning.

Recall: LLM is a transformer based model trained to predict next word.

### History of 'intermediate steps'

1. Ling et al 2017 in DeepMind pioneered using natural language rationale to solve math problems by “... derive the final answer through a series of small steps”. Trained a sequence-to-sequence model from scratch. [Paper: [Program Induction by Rationale Generation: Learning to Solve and Explain Algebraic Word Problems](https://aclanthology.org/P17-1015.pdf)]

2. Following the work by Ling et al 2017, Cobbe et al 2021 in OpenAI built a much larger math word problem dataset (GSM8K) with natural language rationales, and used it to finetune GPT3. [[Training Verifiers to Solve Math Word Problems](https://arxiv.org/abs/2110.14168)]

3. In 2021, Google Research introduced a method to enhance language models' performance on complex multi-step computations by introducing "scratchpads." [[Show Your Work: Scratchpads for Intermediate Computation with Language Models]()]

4. In 2022, Google Research studied effectiveness of Chain-of-Thought prompting (a method where language models are given a series of reasoning steps to improve complex reasoning tasks) in solving complex reasoning tasks. They tested on several large models, including PaLM, LaMDA, and GPT-3, with PaLM 540B achieving state-of-the-art results on certain tasks, such as GSM8K (a math reasoning benchmark). [[Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)]

   More interesting results:

   * Robustness tests revealed that CoT's effectiveness is sensitive to prompt variations, model type, and exemplar choice, though the method consistently outperformed standard prompting.
   * Limitations include minor reasoning errors, where models miss a step or make calculation errors, and the need for larger models to observe benefits.
   * CoT prompting allows for generalization to unseen longer sequences and out-of-distribution (OOD) examples, demonstrating flexibility across diverse reasoning tasks.
   * CoT prompting does not involve fine-tuning models but leverages few-shot learning by providing structured prompts with intermediate reasoning steps.

5. In summary, the history flows:

   1. Training with intermediate steps (Ling et al 2017)
   2. Finetuning with intermediate steps (Cobbe et al 2021, Nye et al 2021)
   3. Prompting with intermediate steps (Nye et al 2021, Wei et al 2022)

   Takeaway: what matters is 'intermediate steps'.

### Reasoning Strategies

1. In 2023, a team in Google DeepMind introduced the least-to-most prompting which enables easy-to-hard generalization by decomposition.  [[Least-to-Most Prompting enables complex reasoning in Large Language Models](https://arxiv.org/abs/2205.10625)]

   The rationale behind is that: *Decomposing and recombining are important operations of the mind. "You decompose the whole into its parts, and you recombine the parts into a more or less different whole. If you go into detail you may lose yourself in details."* (How to Solve - a new aspect of mathematical method, by G. Polya)

2. In 2024, they published another paper on this topic, showing their findings 1) transformer generating intermediate steps can solve any inherently serial problem as long as its depth exceeds a constant threshold, and 2) transformer generating direct answers either requires a huge depth to solve or cannot solve at all. [[Chain of Thought Empowers Transformers to Solve Inherently Serial Problems](https://arxiv.org/abs/2402.12875)]
3. Later on, they found a way to enable CoT reasoning without prompting "let's think step by step". [[Chain-of-Thought Reasoning Without Prompting](https://arxiv.org/abs/2402.10200)]

**Key Observations**

1. Pre-trained LLMs have had responses with step-by-step reasoning among the generations started with the top-k tokens
2. Higher confidence in decoding the final answer when a step-by-step reasoning path is present

### Concerns of on intermediate steps instead of direct answer

1. In 2023, the team in Google found out that self-consistency greatly improves step-by-step reasoning, and the more consistent the more accurate. They proposed a new decoding strategy, *self-consistency*, for chain-of-thought (CoT) prompting in language models. The method combines CoT prompting with self-consistency through three steps: 1) Use CoT to generate reasoning paths, 2) Replace greedy decoding with sampling diverse paths, 3) Aggregate results by marginalizing over the sampled paths to identify the most consistent answer. [[Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171)]

   For context, traditional *greedy decoding* is a decoding method in which, at each step of the text generation process, the model selects the token with highest probability as its next word. It's simple, fast, and computationally efficient, but 1) it can lead to suboptimal results because it ignores less likely but potential better overall sequences, 2) it produces deterministic outputs, which lack diversity and may fail to explore alternative reasoning path.

   LLMs are probabilistic models of generating next tokens. They are not humans. What LLM does in decoding: $\arg \max \mathbb{P}(\text{reasoning path, final answer | problem})$. What we want: $\arg \max \mathbb{P} (\text{final answer | problem}) = \sum_{\text{reasoning path}} \mathbb{P}(\text{reasoning path, final answer | problem})$.

   Self-consistency relies on the idea of maximum marginal inference with formula $\arg \max \mathbb{P} (\text{final answer | problem})$. So we can say "when the LLM outputs a direct answer without intermediate steps, you sample several times, and then choose the most common answer", but not "change self-consistency by letting LLM generate multiple responses, instead of sampling multiple times, and then choosing the most common answer". 

2. They further proposed Universal Self-Consistency (USC), asking LLMs to self-select the most consistent answer. [[Universal Self-Consistency for Large Language Model Generation](https://arxiv.org/abs/2311.17311)]

### Limitation

1. Psychology studies show that irrelevant information may significantly decrease some children and even adults problem-solving accuracy. The research was trying to figure out whether this observation hold for LLMs. [[Large Language Models Can Be Easily Distracted by Irrelevant Context](https://arxiv.org/abs/2302.00093)]

2. While allowing LLMs to review their generated responses can help correct inaccurate answers, it may also risk changing correct answers into incorrect ones. [[Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798)] 

   Self-debugging framework contains: 1) Generation: Model generates candidate programs based on the problem description, 2) Explanation: Model processes its output, explaining predictions in natural language or refining problem descriptions, 3) Feedback: Using these explanations, the model iteratively improves its predictions. [[Teaching Large Language Models to Self-Debug](https://arxiv.org/abs/2304.05128)]

3. LLMs' reasoning performance significantly depends on the order of premises, even when the order does not affect the logical task itself. Models perform best when premises are arranged in the same order as the ground truth proof. [[Premise Order Matters in Reasoning with Large Language Models](https://arxiv.org/abs/2402.08939)]