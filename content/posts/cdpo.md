---
date: 2026-03-14T15:42:57Z
subtitle: Augmenting DPO with Verifier Critiques
title: "Critique DPO"
---

I chanced upon a pretty interesting idea proposed by Peter Hayes (now at Anthropic) in 2025, to more efficiently use feedback data in the standard RLHF pipeline to speed up the whole thing. Basically, given external feedback/critique/verification of generated answers, can we incorporate this feedback in an NLL term in addition to the standard DPO loss; trained on the base model? Most previous efforts either used this feedback to train a better verifier, or used other additional NLL terms in the base model training. I spent couple of months reading around the literature, setting up the HPC infra, and running some experiments. Unfortunately experiments (at least on smaller models) resulted in negative findings - the model was extremely prone to [reward hacking](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/)...

> Full [paper](https://drive.google.com/file/d/1l0JiDMXoUyYJlQXibl_9sLOu_IDx0m4Y/view?usp=drive_link) and [code](https://github.com/liuzihe02/RDPO) available =D

This post is a summary of the background and results. We go through DPO in quite a bit of depth, cover relevant research, and show reward hacking for our proposed method (Critique DPO).

---

# Introduction

[Direct Preference Optimization](https://iclr-blogposts.github.io/2024/blog/rlhf-without-rl/) (DPO) enables fine-tuning language models from pairwise preferences without reinforcement learning, often used in Reinforcement Learning with Human Feedback (RLHF) pipelines to align LLMs with human intent. Instead of human preferences, we may use LLM-as-a-judge workflows, where chain-of-thought reasoning improves judgement quality. Can we leverage the judge's reasoning --- not just its binary label --- to improve training sample efficiency when fine-tuning the base model?

We propose Critique DPO (CDPO), which augments the DPO objective with a negative log-likelihood term on the verifier's chain-of-thought critique, training the base policy to reproduce step-by-step verification reasoning alongside learning from pairwise preferences. We evaluate CDPO against DPO and SFT baselines on Qwen2.5-0.5B-Instruct using 120,000 math preference triples from the GenRM dataset. Our results are a negative finding: the critique term exacerbates rather than mitigates DPO's reward-hacking failure mode, with CDPO collapsing faster than vanilla DPO on both GSM8K and MATH-500.

---

# Background: DPO

For a given prompt $x$, two completions are sampled from the policy $(y_{w}, y_{l}) \sim \pi_{\theta}(y | x)$ and an oracle judges which it prefers. We denote $y_{w}$ as the preferred completion and $y_{l}$ as the other. This process is repeated over $N$ prompts resulting in the pairwise preference dataset $\mathcal{D} = \\{x^{(i)}, y_{w}^{(i)}, y_{l}^{(i)}\\}_{i=1}^{N}$.

## Bradley-Terry Preference Model

We can model the judge preferences using the Bradley-Terry (BT) model, which expresses the probability that completion $y_1$ is preferred over $y_2$ under a latent reward function $r^*$:

$$p^*(y_1 \succ y_2 \mid x) = \frac{\exp\left(r^*(x, y_1)\right)}{\exp\left(r^*(x, y_1)\right) + \exp\left(r^*(x, y_2)\right)} = \sigma\!\left(r^*(x, y_1) - r^*(x, y_2)\right)$$

A key property is that BT depends only on the *difference* of rewards, not their absolute values.

## Standard RLHF

The standard RLHF pipeline comprises 2 stages. First, we learn a reward model $r_\phi$ where we find the maximum likelihood estimate (MLE) of $\phi$ under our preference dataset $\mathcal{D}$ and BT model - termed reward modelling. Then, we use RL (PPO) to learn an optimal policy $\pi_\theta$ for this learned reward model.

### Reward Modelling

Since $r^*$ is unknown, we learn a parametric approximation $r_\phi$. Under the BT model, the likelihood of the observed preference dataset is $\ell(\mathcal{D} \mid \phi) = \prod_{i=1}^{N} \sigma\!\left(r_\phi(x^{(i)}, y_w^{(i)}) - r_\phi(x^{(i)}, y_l^{(i)})\right)$. Taking the log and writing as an expectation, the MLE is obtained by minimising

$$\mathcal{L}_{\text{RM}}(\phi) = - \mathbb{E}_{\mathcal{D}} \left[\log \sigma\!\left(r_{\phi}(x, y_{w}) - r_{\phi}(x, y_{l})\right)\right] \tag{1}$$

### RL Fine-Tuning

The learned reward model is then used to fine-tune the policy. The aim is to maximise the KL-constrained objective

$$\max_{\pi_\theta} \; \mathbb{E}_{x \sim \mathcal{D},\, y \sim \pi_{\theta}(y | x)} \left[r_{\phi}(x, y)\right] - \beta \, D_{\text{KL}}\!\left(\pi_{\theta}(y | x) \| \pi_{\text{ref}}(y | x)\right) \tag{2}$$

where $\pi_{\text{ref}}$ is the reference policy (typically the SFT model), kept frozen. The KL term prevents the policy from deviating too far from the distribution on which the reward model is accurate, and $\beta > 0$ controls this trade-off. In practice this is optimised with PPO, requiring four models in memory: the policy, reference, reward model, and value function.

## DPO

DPO [{{< link src="https://arxiv.org/abs/2305.18290" class="external" target="_blank" rel="noopener noreferrer" >}}Rafailov et al., 2024{{< /link >}}] bypasses the reward modelling and RL steps entirely through a change of variables. The key insight is that there exists a bijection between reward functions and their corresponding optimal policies under the KL-constrained objective $(2)$, and this bijection can be exploited to reparameterise the preference likelihood directly in terms of the policy. We follow the expositions by {{< link src="https://cameronrwolfe.substack.com/p/direct-preference-optimization" class="external" target="_blank" rel="noopener noreferrer" >}}Wolfe, 2025{{< /link >}} and {{< link src="https://www.tylerromero.com/posts/2024-04-dpo/" class="external" target="_blank" rel="noopener noreferrer" >}}Romero, 2024{{< /link >}} in our presentation of the derivation below.

### Optimal Policy

For *any* reward function $r$ (not just the learned $r_\phi$ or the latent $r^*$), the KL-constrained objective $(2)$ can be rewritten as a minimisation of a KL divergence between $\pi$ and a target distribution defined by $r$, plus a constant. Specifically, letting $Z_r(x) = \sum_y \pi_{\text{ref}}(y|x)\exp\!\left(\frac{1}{\beta}r(x,y)\right)$ be a partition function that depends on $x$ and $r$ but is constant with respect to $\pi$, the objective becomes

$$\min_{\pi} \; \mathbb{E}_{x} \left[ D_{\text{KL}}\!\left(\pi(y|x) \;\Big\|\; \frac{1}{Z_r(x)}\,\pi_{\text{ref}}(y|x)\,\exp\!\Big(\tfrac{1}{\beta}\,r(x,y)\Big)\right) - \log Z_r(x) \right]$$

By Gibbs' inequality ($D_{\text{KL}} \geq 0$, with equality iff the distributions are identical), the unique optimum is

$$\pi_r(y|x) = \frac{1}{Z_r(x)}\,\pi_{\text{ref}}(y|x)\,\exp\!\left(\frac{1}{\beta}\,r(x,y)\right) \tag{3}$$

This defines a bijection: every reward $r$ maps to a unique optimal policy $\pi_r$, and conversely every policy $\pi_r$ defines a unique reward [^1] $r$ for which it is optimal. Using the latent reward $r^*$ gives the optimal policy $\pi^* \equiv \pi_{r^*}$ for our latent rewards.

### Reparameterisation

Rearranging $(3)$ for the reward expresses any reward purely in terms of its corresponding optimal policy and a prompt-dependent partition function:

$$r(x, y) = \beta \log\frac{\pi_r(y|x)}{\pi_{\text{ref}}(y|x)} + \beta \log Z_r(x)$$

Substituting the latent reward $r^*(x,y) = \beta \log\frac{\pi^*(y|x)}{\pi_{\text{ref}}(y|x)} + \beta \log Z_{r^*}(x)$ into the BT model, the $\beta \log Z_{r^*}(x)$ terms cancel because BT depends only on reward *differences* and $Z_{r^*}(x)$ is independent of $y$:

$$p^*(y_w \succ y_l \mid x) = \sigma\!\left(\beta \log \frac{\pi^*(y_w|x)}{\pi_{\text{ref}}(y_w|x)} - \beta \log \frac{\pi^*(y_l|x)}{\pi_{\text{ref}}(y_l|x)}\right)$$

The preference model now has no dependence on a reward model --- it is expressed entirely in terms of policies.

### DPO Loss

We replace the fixed $\pi^*$ with a learnable $\pi_\theta$ and maximise the likelihood of the observed preferences, conditioned on our policy parameters $\theta$. Defining the *implicit reward*

$$\hat{r}_\theta(x, y) = \beta \log\frac{\pi_\theta(y|x)}{\pi_{\text{ref}}(y|x)} \tag{4}$$

the DPO loss is

$$\mathcal{L}_{\text{DPO}}(\theta) = - \mathbb{E}_{\mathcal{D}} \left[ \log \sigma \left( \hat{r}_\theta(x, y_{w}) - \hat{r}_\theta(x, y_{l}) \right) \right] \tag{5}$$

This is identical in form to the reward model loss $(1)$, but conditioned on the policy parameters $\theta$ instead of the reward model parameters $\phi$. Traditionally, RLHF performs MLE on $\mathcal{D}$ conditioned on $\phi$ to learn a reward model, then uses RL to find the optimal policy for that reward. DPO instead performs MLE on $\mathcal{D}$ conditioned directly on $\theta$. Because of the reward--policy bijection, improving $\pi_\theta$ to better fit the preference data simultaneously yields an implicit reward $\hat{r}_\theta$ for which $\pi_\theta$ is guaranteed to be optimal. This eliminates both the reward modelling step and the RL step, requiring only two models in memory (the policy $\pi_\theta$ and the frozen reference $\pi_{\text{ref}}$).

---

# Literature Review

We review research incorporating reasoning and critique data into fine-tuning, building toward the critique-augmented preference optimisation approach of this work. We adopt a unified notation: $\pi_\theta$ denotes the learnable policy, $\pi_{\text{ref}}$ the frozen reference (usually the base SFT model), $x$ the prompt, $y_w$ and $y_l$ the preferred and rejected responses, $c$ a model's own chain-of-thought reasoning trace, $c_v$ a verifier's chain-of-thought for the critique, whether from an external oracle or generated by the model itself, $r_\phi(x,y)$ a learned reward (where $\phi$ parameterises any judge, verifier, or reward model), and $\hat{r}_\theta(x,y) = \beta \log \frac{\pi_\theta(y|x)}{\pi_{\text{ref}}(y|x)}$ the implicit reward.

The review is organised around a central question: *what kind of signal is being used, and for which model?* The narrative moves from binary AI feedback, through enriching DPO with supervised reasoning targets, to methods that improve verifiers by having them reason, and finally to the closest work --- methods that bring verifier critique directly into the base model's training objective.

## Beyond Binary Feedback

### RLAIF

RL from AI Feedback (RLAIF) replaces human annotators with LLM oracles. {{< link src="https://arxiv.org/abs/2212.08073" class="external" target="_blank" rel="noopener noreferrer" >}}Bai et al., 2022{{< /link >}} first use LLM-generated labels for harmlessness finetuning, doing RL with a preference model trained with LLM preferences. {{< link src="https://arxiv.org/abs/2309.00267" class="external" target="_blank" rel="noopener noreferrer" >}}Lee et al., 2024{{< /link >}} use LLM to directly give rewards, showing that RLAIF matches or exceeds RLHF on summarisation and dialogue, even when the labeller is the same model being aligned. {{< link src="https://arxiv.org/abs/2403.08309" class="external" target="_blank" rel="noopener noreferrer" >}}Li et al., 2024{{< /link >}} instruct the labeller to reason explicitly and report modest improvements. A shared limitation is that the oracle's reasoning is discarded after labelling: the sole training signal is a binary or scalar preference label, creating an information bottleneck that the remainder of this review seeks to address.

### Impact of Chain-of-Thought Data for DPO

{{< link src="https://arxiv.org/abs/2502.11656" class="external" target="_blank" rel="noopener noreferrer" >}}Gao et al., 2025{{< /link >}} provide a systematic empirical study of CoT's impact on DPO. In text-to-SQL --- where preference datasets traditionally contain only final SQL queries *without* reasoning steps --- DPO degrades performance across the majority of ten tested LLMs. Augmenting the same datasets with synthetically generated chain-of-thought solutions achieves consistent DPO improvements for the first time. The analysis identifies three mechanisms. First, *reward hacking mitigation*: without CoT, the implicit reward $(4)$ can be inflated by superficial correlates (longer SQL, non-executable patterns) that do not reflect reasoning quality. Model performance initially peaks then decreases, while the implicit reward continues to rise. With CoT, the implicit reward tracks actual performance. Second, *discriminative calibration*: CoT maintains stable accuracy in distinguishing correct from incorrect solutions, whereas vanilla DPO becomes unstable. Third, *sample efficiency*: CoT-augmented DPO continues to improve with additional data, while vanilla DPO saturates.

### Self-Taught Reasoning: STaR and V-STaR

**STaR.** {{< link src="https://arxiv.org/abs/2203.14465" class="external" target="_blank" rel="noopener noreferrer" >}}Zelikman et al., 2022{{< /link >}} establish that CoT data, even when self-generated, is a richer training signal than outcome labels alone. Given a pretrained model $M$ and a dataset $\mathcal{D} = \\{(x_i, y_i)\\}$ of problems with ground-truth answers, STaR begins with a small few-shot prompt set $\mathcal{P}$ containing rationale examples. At each iteration, the model generates a reasoning trace $\hat{c}_i$ and answer $\hat{y}_i$ for every problem, using $\mathcal{P}$ in the context. Only correct generations where $\hat{y}_i = y_i$ are retained; the model is then fine-tuned on this filtered set with the standard NLL objective on rationale-answer pairs:

$$\mathcal{L}_{\text{STaR}}(M) = -\mathbb{E}_{(x,y) \sim \mathcal{D}} \left[\,\mathbf{1}[\hat{y} = y]\cdot \left(\sum_{t=1}^{|\hat{c}|} \log M(\hat{c}_t \mid x, \hat{c}_{< t}) + \log M(\hat{y} \mid x, \hat{c})\right)\right] \tag{6}$$

**V-STaR.** {{< link src="https://arxiv.org/abs/2402.06457" class="external" target="_blank" rel="noopener noreferrer" >}}Hosseini et al., 2024{{< /link >}} extend STaR by repurposing incorrect solutions (discarded by STaR) to co-train a verifier $V$ alongside the generator $G$. At each iteration, the generator is fine-tuned via SFT on correct solutions using Eq. $(6)$. The verifier is trained with DPO on preference pairs $(x, y^+, y^-)$, formed by pairing correct and incorrect solutions for each problem, using $G_{\text{SFT}}$ (the base model fine-tuned once) as the fixed reference policy:

$$\mathcal{L}_{\text{DPO}}(V^t;\, G_{\text{SFT}}) = -\mathbb{E}_{(x, y^+, y^-) \sim \mathcal{D}_{\text{VER}}} \!\left[\log \sigma\!\left(\hat{r}(x, y^+) - \hat{r}(x, y^-)\right)\right], \quad \hat{r}(x, y) = \beta \log \frac{V^t(y \mid x)}{G_{\text{SFT}}(y \mid x)} \tag{7}$$

Crucially, the verifier provides no training-time gradient to the generator --- it is used only for Best-of-$k$ reranking at inference. This distinguishes V-STaR from later methods that feed verifier reasoning directly into the generator's training.

## DPO with NLL

A recurring structural pattern in the literature combines the DPO preference loss with an auxiliary negative log-likelihood (NLL) term on a supervised target. We present the generic template first, then show how each paper instantiates it with a **different NLL target** $t$:

$$\mathcal{L}_{\text{DPO+NLL}} = \underbrace{-\log \sigma\!\left(\hat{r}_\theta(x, y_w) - \hat{r}_\theta(x, y_l)\right)}_{\text{preference term}} \;+\; \alpha \underbrace{\mathcal{L}_{\text{NLL}}(t \mid x)}_{\text{supervised anchor}} \tag{8}$$

The DPO term provides a contrastive signal (push apart $y_w$ and $y_l$), while the NLL term anchors the policy to a specific supervised target. As we show below, the choice of $t$ is what distinguishes these methods from one another.

### Iterative Reasoning Preference Optimization (IRPO)

{{< link src="https://arxiv.org/abs/2404.19733" class="external" target="_blank" rel="noopener noreferrer" >}}Pang et al., 2024{{< /link >}} set the NLL target to the *model's own winning chain-of-thought*: $t = (c_w, y_w)$. At each iteration, the model samples $N$ reasoning traces per prompt, partitions them into correct ($c_w, y_w$) and incorrect ($c_l, y_l$) by checking the final answer against ground truth, and optimises:

$$\mathcal{L}_{\text{IRPO}} = -\log \sigma\!\left(\beta \log \frac{\pi_\theta(c_w, y_w \mid x)}{\pi_{\text{ref}}(c_w, y_w \mid x)} - \beta \log \frac{\pi_\theta(c_l, y_l \mid x)}{\pi_{\text{ref}}(c_l, y_l \mid x)}\right) - \alpha \frac{\log \pi_\theta(c_w, y_w \mid x)}{|c_w| + |y_w|} \tag{9}$$

The NLL term is length-normalised by $|c_w| + |y_w|$ to avoid a bias toward shorter sequences. Crucially, the reference model $\pi_{\text{ref}}$ is updated to $\pi_\theta$ at each iteration before re-sampling. Under standard DPO (i.e. $\alpha = 0$), the log-probabilities of *chosen* sequences may actually decrease during training --- the model achieves the required preference margin by pushing rejected probabilities down faster than chosen ones fall. The NLL term prevents this collapse: with $\alpha > 0$, chosen log-probabilities increase while rejected ones decrease.

### NLL on Chosen Responses: AIPO and RPO

Both AIPO [{{< link src="https://arxiv.org/abs/2409.08845" class="external" target="_blank" rel="noopener noreferrer" >}}Shen et al., 2024{{< /link >}}] and RPO [{{< link src="https://arxiv.org/abs/2405.16436" class="external" target="_blank" rel="noopener noreferrer" >}}Liu et al., 2024{{< /link >}}] set the NLL target to the **winning response**: $t = y_w$. AIPO adds a length-normalised NLL anchor alongside $\alpha$-DPO, a DPO variant that scales the reference log-ratio by $(1+\alpha)$ to introduce a dynamic preference margin weighted by the reference model's agreement with the ranking:

$$\mathcal{L}_{\text{AIPO}} = \mathcal{L}_{\alpha\text{-DPO}} + \lambda \cdot \mathcal{L}_{\text{NLL}}, \qquad \mathcal{L}_{\text{NLL}} = -\frac{1}{|y_w|}\log \pi_\theta(y_w \mid x) \tag{10}$$

RPO derives the same NLL term from first principles: framing DPO overoptimisation as a minimax problem over adversarially chosen reward models, the objective collapses exactly to DPO plus NLL on chosen responses:

$$\mathcal{L}_{\text{RPO}}(\theta) = \mathcal{L}_{\text{DPO}}(\theta) \;-\; \eta\beta\,\mathbb{E}_{(x,y_w)\sim\mathcal{D}}\!\left[\log \pi_\theta(y_w \mid x)\right] \tag{11}$$

Both methods improve over vanilla DPO, confirming that the NLL anchor mitigates reward hacking regardless of whether the motivation is empirical or theoretical.

### Direct Judgement Preference Optimization (DJPO)

{{< link src="https://arxiv.org/abs/2409.14664" class="external" target="_blank" rel="noopener noreferrer" >}}Wang et al., 2024{{< /link >}} set the NLL target to the **verifier's CoT critique and judgment**: $t = (c_v, j)$, where $c_v$ is the verifier's chain-of-thought critique and $j$ is the final judgment. Crucially, the model being trained is a judge, not the base policy. They construct DPO preference pairs over judge evaluations --- pairing correct and incorrect judgements (with and without CoT reasoning) --- and train with:

$$\mathcal{L}_{\text{DJPO}} = -\frac{\log \pi_\phi(c_{v,w} \mid x)}{|c_{v,w}| + |x|} - \log \sigma\!\left(\beta \log \frac{\pi_\phi(c_{v,w} \mid x)}{\pi_{\text{ref}}(c_{v,w} \mid x)} - \beta \log \frac{\pi_\phi(c_{v,l} \mid x)}{\pi_{\text{ref}}(c_{v,l} \mid x)}\right) \tag{12}$$

where $c_{v,w}$ and $c_{v,l}$ are correct and incorrect judge evaluations (critique plus judgment) respectively. Using 680K preference pairs, they train SFR-Judge models at 8B, 12B, and 70B scales. SFR-Judge-8B outperforms GPT-4o on aggregate pairwise evaluation across 7 benchmarks; SFR-Judge-70B achieves the best overall score of 84.25%.

## Generative Verifiers and Critique-Augmented Reward Models

The methods above train the policy model. A parallel line of work improves the verifiers and reward models that judge responses, by having them produce reasoning before scoring. These methods establish that critique data is a powerful training signal for judging. Generative verifiers aim to provide a scalar verifier score that is used during inference for Best-of-N sampling, whereas critique-augmented reward models produce better reward models for downstream preference training.

### Generative Verifiers (GenRM)

{{< link src="https://arxiv.org/abs/2408.15240" class="external" target="_blank" rel="noopener noreferrer" >}}Zhang et al., 2024{{< /link >}} recast verification as next-token prediction rather than scalar classification. Instead of a separate reward head, the verifier is an LLM fine-tuned via SFT to answer "Is the answer correct?" given a problem $x$ and candidate solution $y$.

In **GenRM-Direct**, the verifier is trained on pairs of $(x, y^+, I) \to \text{`Yes'}$ and $(x, y^-, I) \to \text{`No'}$, where $I$ = "Is the answer correct (Yes/No)?". At inference, the verifier score is the next-token probability of 'Yes':

$$r_{\text{Direct}}(x, y) = \pi_\phi(\text{`Yes'} \mid x, y, I) \tag{13}$$

**GenRM-CoT** inserts a chain-of-thought verification rationale $c_v$ before the final Yes/No token. The rationales are generated offline by a capable LLM and filtered by majority agreement. At inference, the verifier first samples its own rationale, then scores:

$$r_{\text{CoT}}(x, y) = \pi_\phi(\text{`Yes'} \mid x, y, I_v, c_v, I), \qquad c_v \sim \pi_\phi(\cdot \mid x, y, I_v) \tag{14}$$

where $I_v$ = "Let's verify step by step." GenRM-CoT achieves 94.2% Best-of-32 on GSM8K (vs. ~90% for a discriminative RM) with 6.4x better sample efficiency. This dataset --- including the per-solution verification rationales --- is the data source for our experiments.

### GenRM-STaR-DPO: Training Judges with DPO on Rationales

{{< link src="https://arxiv.org/abs/2410.12832" class="external" target="_blank" rel="noopener noreferrer" >}}Mahan et al., 2024{{< /link >}} propose a unified RLHF--RLAIF framework for training a generative judge as a reward model. They create 3 training variants. Given a preference dataset $\mathcal{D} = \\{(x, y_1, y_2, I)\\}$ where $I$ is the winning response indicator:

**GenRM (no CoT)** --- standard SFT on the indicator token:

$$\mathcal{L}_{\text{GenRM}}(\pi_\phi) = \mathbb{E}_{(x,y_1,y_2,I)\sim\mathcal{D}}\!\left[-\log \pi_\phi(I \mid x, y_1, y_2)\right] \tag{15}$$

**CoT-GenRM with Rationalization** --- a post-rationalization model $q$ (e.g. GPT-4 or a stronger LLM) generates a critique rationale $c_v$ conditioned on the correct answer, and the model is trained to both generate the rationale and predict the indicator:

$$\mathcal{L}_{\text{Rationalizer}}(\pi_\phi) = \mathbb{E}_{(x,y_1,y_2,c_v,I)\sim\mathcal{D}}\!\left[-\log \pi_\phi(I \mid x, y_1, y_2, c_v) - \log \pi_\phi(c_v \mid x, y_1, y_2)\right] \tag{16}$$

**CoT-GenRM-STaR-DPO** --- instead of discarding incorrect self-generated critique rationales (as STaR-SFT does), STaR-DPO constructs preference pairs $(c_{v,w}, I_w)$ vs $(c_{v,l}, I_l)$ from correct and incorrect reasoning traces and applies DPO over the full rationale-plus-judgment sequence:

$$\mathcal{L}_{\text{STaR-DPO}}(\pi_\phi) = -\mathbb{E}_{\mathcal{D}}\!\left[\log\sigma\!\left(\beta\log\frac{\pi_\phi(I_w, c_{v,w} \mid x, y_1, y_2)}{\pi_{\text{ref}}(I_w, c_{v,w} \mid x, y_1, y_2)} - \beta\log\frac{\pi_\phi(I_l, c_{v,l} \mid x, y_1, y_2)}{\pi_{\text{ref}}(I_l, c_{v,l} \mid x, y_1, y_2)}\right)\right] \tag{17}$$

On-policy rationales are essential: using off-policy rationales from a stronger model (GPT-4 or Llama-3.1-70B) matches STaR-DPO in-distribution but collapses OOD (RewardBench drops to 63--70%). The authors attribute this to distributional mismatch: the model imitates the rationaliser's style during training but generates different reasoning at test time.

### Critique-out-Loud Reward Models (CLoud)

{{< link src="https://arxiv.org/abs/2408.11791" class="external" target="_blank" rel="noopener noreferrer" >}}Ankner et al., 2024{{< /link >}} train reward models with a shared base $\theta_B$, an LM head $\theta_{LM}$ that generates critiques, and a scalar reward head $\theta_R$. Training proceeds in three stages: (1) SFT $\theta_{LM}$ on oracle critiques; (2) self-generate critiques to form $\mathcal{D}_{\text{self}}$; (3) jointly optimise all parameters on $\mathcal{D}_{\text{self}}$:

$$\mathcal{L}_{\text{CLoud}} = \underbrace{-\mathbb{E}\!\left[\log\sigma\!\left(r_{\theta_R}(x, y^+, \hat{c}^+) - r_{\theta_R}(x, y^-, \hat{c}^-)\right)\right]}_{\mathcal{L}_{\text{RM}}} \;+\; \lambda\,\underbrace{\mathcal{L}_{\text{SFT}}(\theta_B, \theta_{LM}, \mathcal{D}_{\text{self}})}_{\text{critique quality}} \tag{18}$$

The self-generation step is critical: training the reward head on oracle (off-policy) critiques causes a 5.6% drop in RewardBench accuracy, because oracle critiques are systematically different from self-generated ones, creating a distributional shift at inference. {{< link src="https://arxiv.org/abs/2411.16646" class="external" target="_blank" rel="noopener noreferrer" >}}Yu et al., 2025{{< /link >}} extend this with a variational formulation (Critic-RM) that treats self-generated critiques as latent variables.

## Training the Base Model with Critique

The methods in the previous section improve reward models and judges by having them generate critique. A closely related cluster instead trains the base *policy* model directly using critique signals.

### Critique Fine-Tuning

{{< link src="https://arxiv.org/abs/2501.17703" class="external" target="_blank" rel="noopener noreferrer" >}}Wang et al., 2025{{< /link >}} ask: what happens if we use *only* the NLL term on critique data, without any preference optimisation? In Critique Fine-Tuning (CFT), the base model is trained on the teacher's critique $c_v$ of a given query-response pair by maximising the log-likelihood:

$$\mathcal{L}_{\text{CFT}}(\theta) = -\sum_{t=1}^{|c_v|} \log \pi_\theta(c_{v,t} \mid c_{v,<t},\, [x;\, y]) \tag{19}$$

where $[x;\, y]$ concatenates the query and a (potentially flawed) response, and $c_v$ is a critique generated by a teacher model (GPT-4o). At inference, the CFT-trained model generates direct answers without producing critiques.

### SuperCorrect: Cross-Model Collaborative DPO

{{< link src="https://arxiv.org/abs/2410.09008" class="external" target="_blank" rel="noopener noreferrer" >}}Yang et al., 2024{{< /link >}} introduce a two-stage framework. Stage 1 distils hierarchical thought templates from a large teacher model (o1-preview) via SFT. Stage 2 applies **cross-model collaborative DPO** on teacher-guided error-correction traces. Given an erroneous student solution with the first error at step $k$, the teacher generates a correction trace $(a_k^+, c_k^+)$ (error analysis and correction guidance) while the student independently generates $(a_k^-, c_k^-)$. The DPO loss is:

$$\mathcal{L}_{\text{Cross-DPO}} = -\mathbb{E}\!\left[\log\sigma\!\left(\beta\log\frac{\pi_\theta(a_k^+, c_k^+ \mid x, s_{< k})}{\pi_{\text{ref}}(a_k^+, c_k^+ \mid x, s_{< k})} - \beta\log\frac{\pi_\theta(a_k^-, c_k^- \mid x, s_{< k})}{\pi_{\text{ref}}(a_k^-, c_k^- \mid x, s_{< k})}\right)\right] \tag{20}$$

where $s_{< k}$ are the correct preceding steps. The teacher's correction trace plays the role of $c_v$ in CDPO. SuperCorrect-7B achieves 70.2% on MATH (surpassing DeepSeekMath-7B by 7.8%) and demonstrates that cross-model DPO at the step level substantially improves self-correction capabilities. CriticGPT [{{< link src="https://arxiv.org/abs/2407.00215" class="external" target="_blank" rel="noopener noreferrer" >}}McAleese et al., 2024{{< /link >}}] and CTRL [{{< link src="https://arxiv.org/abs/2502.03492" class="external" target="_blank" rel="noopener noreferrer" >}}Xie et al., 2025{{< /link >}}] provide complementary evidence that LLMs can be explicitly taught to generate high-quality critiques.

## Positioning of CDPO

The three defining features of CDPO are: **(A)** a DPO loss on answer preferences $(y_w, y_l)$; **(B)** an NLL term on the verifier's CoT critique $c_v$; and **(C)** applied to the base policy model. No single prior work combines all three.

| **Paper** | **DPO on answers** | **NLL on verifier CoT** | **Base model** |
|---|---|---|---|
| **CDPO (this work)** | Yes | Yes | Yes |
| IRPO [{{< link src="https://arxiv.org/abs/2404.19733" class="external" target="_blank" rel="noopener noreferrer" >}}Pang et al.{{< /link >}}] | Yes | NLL on own CoT | Yes |
| AIPO [{{< link src="https://arxiv.org/abs/2409.08845" class="external" target="_blank" rel="noopener noreferrer" >}}Shen et al.{{< /link >}}] / RPO [{{< link src="https://arxiv.org/abs/2405.16436" class="external" target="_blank" rel="noopener noreferrer" >}}Liu et al.{{< /link >}}] | Yes | NLL on $y_w$ | Yes |
| DJPO [{{< link src="https://arxiv.org/abs/2409.14664" class="external" target="_blank" rel="noopener noreferrer" >}}Wang et al.{{< /link >}}] | Yes | Yes | No (judge) |
| GenRM-STaR-DPO [{{< link src="https://arxiv.org/abs/2410.12832" class="external" target="_blank" rel="noopener noreferrer" >}}Mahan et al.{{< /link >}}] | Yes | Yes | No (judge) |
| CFT [{{< link src="https://arxiv.org/abs/2501.17703" class="external" target="_blank" rel="noopener noreferrer" >}}Wang et al.{{< /link >}}] | No | Yes | Yes |
| SuperCorrect [{{< link src="https://arxiv.org/abs/2410.09008" class="external" target="_blank" rel="noopener noreferrer" >}}Yang et al.{{< /link >}}] | On traces | Teacher correction | Yes |

CDPO sits at the intersection of three lines of work that have so far only been combined in pairs: (1) IRPO establishes the right loss structure (DPO + NLL) but anchors to the model's own CoT rather than an external verifier's critique; (2) DJPO and GenRM-STaR-DPO use verifier reasoning as the NLL target --- the right data --- but apply it to train a standalone judge rather than the answer-generating base policy; and (3) CFT applies verifier critique as a direct NLL target for the base policy --- the right model and signal --- but omits the DPO preference term entirely.

---

# Methodology

We now describe the CDPO objective, dataset construction, and baseline methods.

## Data Construction

We use the GenRM dataset [{{< link src="https://arxiv.org/abs/2408.15240" class="external" target="_blank" rel="noopener noreferrer" >}}Zhang et al., 2024{{< /link >}}], which provides math problems from the GSM8K [{{< link src="https://arxiv.org/abs/2110.14168" class="external" target="_blank" rel="noopener noreferrer" >}}Cobbe et al., 2021{{< /link >}}] training set. For each problem $x$, multiple candidate solutions are generated by Gemini 1.0 Pro. Each candidate solution is labelled correct or incorrect by checking the extracted final answer against the ground-truth answer. An LLM oracle generates a step-by-step verification rationale *for each individual solution*, verifying each reasoning step against the expected answer and concluding with a "Yes" (correct) or "No" (incorrect) judgement. We denote the verification rationale for a correct solution as $c_{v,w}$ and for an incorrect solution as $c_{v,l}$.

From this raw data, we construct preference triples $(x, y_w, y_l, c_v)$ by pairing, for each problem, a correct solution $y_w$ with an incorrect solution $y_l$. The critique sequence $c_v$ is then assembled by concatenating *both* solutions together with their respective verification rationales into a single contrastive narrative:

> $c_v$ = "This is a correct solution and preferred:" + $y_w$ + "Here's why this solution is correct and preferred:" + $c_{v,w}$ + "This is an incorrect solution and not preferred:" + $y_l$ + "Here's why this solution is incorrect and not preferred:" + $c_{v,l}$

From this data we derive four training methods, described below.

## CDPO: Critique Direct Preference Optimization

Each CDPO sample is $(x,\, y_w,\, y_l,\, c_v)$. CDPO augments the standard DPO loss with an auxiliary negative log-likelihood (NLL) term that trains the base policy to reproduce the verifier's chain-of-thought critique:

$$\mathcal{L}_{\text{CDPO}}(\theta) = \mathbb{E}_{\mathcal{D}}\!\left[(1 - \alpha)\,\mathcal{L}_{\text{DPO}}(\theta) \;+\; \alpha\,\mathcal{L}_{\text{critique}}(\theta)\right] \tag{21}$$

where $\alpha \in [0, 1]$ controls the trade-off between the preference signal and the critique signal. The DPO term $\mathcal{L}_{\text{DPO}}$ is the standard loss from Eq. $(5)$. The critique term is the length-normalised NLL over the composite critique $c_v$:

$$\mathcal{L}_{\text{critique}}(\theta) = -\frac{1}{|c_v|}\sum_{t=1}^{|c_v|} \log \pi_\theta(c_{v,t} \mid x,\, c_{v,<t}) \tag{22}$$

Concretely, the model receives the input sequence $[x;\, c_v]$ --- the problem followed by the critique --- and the NLL loss is computed only over the $c_v$ tokens, with $x$ serving as conditioning context. Because $c_v$ embeds both solutions $y_w$ and $y_l$ within its narrative (as shown above), it is self-contained: the model learns to generate the full contrastive analysis given only the problem statement. We hypothesise that reproducing this analysis teaches the model to distinguish correct from incorrect reasoning steps, thereby improving its own solution quality.

CDPO follows the DPO+NLL pattern of Eq. $(8)$ with the NLL target set to the external verifier's critique $c_v$, but uses a convex combination rather than an additive weight so that $\alpha$ directly controls the relative contribution of each term. Note that at inference time, the CDPO-trained model generates solutions directly --- it does not produce critiques.

## Baseline Methods

We compare CDPO against three baselines, all trained on the same underlying data:

**DPO.** Each sample is $(x,\, y_w,\, y_l)$. The standard DPO loss Eq. $(5)$, equivalent to setting $\alpha = 0$ in Eq. $(21)$. This isolates the contribution of the critique term.

**SFT (no verification).** Each sample is $(x,\, y_w)$. Standard supervised fine-tuning on correct solutions only:

$$\mathcal{L}_{\text{SFT}}(\theta) = -\sum_{t=1}^{|y_w|} \log \pi_\theta(y_{w,t} \mid y_{w,<t},\, x) \tag{23}$$

**SFT (with verification).** Each sample is $(x,\, y_w,\, c_{v,w})$. The same SFT objective, but the training target is $[y_w;\, c_{v,w}]$ --- the correct solution followed by its individual verification rationale. This tests whether the verification signal is useful as a pure SFT target, without the DPO preference term.

---

# Experimental Setup

## Training Configuration

We use Qwen2.5-0.5B-Instruct [{{< link src="https://arxiv.org/abs/2412.15115" class="external" target="_blank" rel="noopener noreferrer" >}}Qwen Team, 2024{{< /link >}}] as the base model. All four methods share the same base configuration: LoRA with rank 8 applied to all linear modules, learning rate $1 \times 10^{-6}$ with cosine decay and 10% warmup, 2 training epochs, bf16 mixed precision, FlashAttention-2, and DeepSpeed ZeRO Stage 3 across 8 NVIDIA H100 80GB GPUs. For DPO and CDPO we use $\beta = 0.1$; CDPO additionally uses $\alpha = 0.5$ with length-normalised critique. Batch sizes differ across methods because CDPO processes three sequences per example (chosen, rejected, critique), requiring significantly more memory.

## Evaluation

We evaluate on two mathematical reasoning benchmarks: GSM8K [{{< link src="https://arxiv.org/abs/2110.14168" class="external" target="_blank" rel="noopener noreferrer" >}}Cobbe et al., 2021{{< /link >}}] (500 test samples) and MATH-500 [{{< link src="https://arxiv.org/abs/2103.03874" class="external" target="_blank" rel="noopener noreferrer" >}}Hendrycks et al., 2021{{< /link >}}]. All evaluations use greedy decoding (temperature 0, top-$p$ = 1) via vLLM for efficient inference. LoRA adapters are merged with the base model before evaluation. We evaluate checkpoints saved at every 10% of training. Note that our training data is drawn from GSM8K training problems, so GSM8K evaluation is in-distribution while MATH-500 is out-of-distribution; this partly explains the more stable GSM8K results for SFT methods.

---

# Results

## Main Results

The table below summarises the best and final checkpoint accuracy for each method. The baseline row shows the pre-training accuracy of the unmodified Qwen2.5-0.5B-Instruct model (checkpoint 0, shared across all methods).

| **Method** | **GSM8K Best** | **GSM8K Final** | **MATH-500 Best** | **MATH-500 Final** |
|---|---|---|---|---|
| Baseline (ckpt-0) | 49.6 | --- | 34.4 | --- |
| SFT (no verification) | **51.8** | 51.4 | **35.4** | 32.6 |
| SFT (with verification) | 50.8 | 50.8 | 35.2 | 33.0 |
| DPO | 49.6 | 4.8 | 34.4 | 3.2 |
| CDPO ($\alpha\!=\!0.5$) | 49.6 | 0.6 | 34.4 | 0.8 |

SFT methods provide modest, stable improvements over the baseline, while both preference-based methods (DPO and CDPO) suffer catastrophic performance collapse. Critically, CDPO --- the method designed to leverage richer feedback --- performs *worse* than vanilla DPO.

## Training Dynamics

The figure below shows evaluation accuracy across training checkpoints. The SFT variants maintain stable accuracy throughout training, with SFT (no verification) reaching 51.8% on GSM8K --- a modest 2.2 percentage point improvement over the 49.6% baseline. Both DPO and CDPO begin at the baseline accuracy (checkpoint 0 is the unmodified model) and collapse rapidly: DPO drops to ~6% within the first 400 training steps, while CDPO collapses even faster to ~1% by step 600.

{{< img src="/cdpo/figures/gsm8k_all_methods.png" caption="GSM8K evaluation accuracy over training checkpoints" width="500">}}

{{< img src="/cdpo/figures/math500_all_methods.png" caption="MATH-500 evaluation accuracy over training checkpoints" width="500">}}

The figure below reveals the mechanism behind this collapse. Despite the catastrophic drop in downstream accuracy, both DPO and CDPO show smoothly decreasing training loss and reward accuracy approaching 1.0 throughout training. This suggests reward hacking: the model learns to separate chosen from rejected examples in the implicit reward space (by pushing log-probability ratios apart), but this optimisation does not correspond to improved generation quality --- it instead degrades the model's ability to produce coherent solutions.

{{< img src="/cdpo/figures/dpo_training_loss.png" caption="DPO training loss" width="350">}}

{{< img src="/cdpo/figures/rdpo_training_loss.png" caption="CDPO training loss" width="350">}}

{{< img src="/cdpo/figures/dpo_reward_acc.png" caption="DPO reward accuracy" width="350">}}

{{< img src="/cdpo/figures/rdpo_reward_acc.png" caption="CDPO reward accuracy" width="350">}}

---

# Discussion

The dramatic collapse of both DPO and CDPO, combined with smoothly improving training metrics, is a well-known failure mode of DPO. {{< link src="https://arxiv.org/abs/2502.11656" class="external" target="_blank" rel="noopener noreferrer" >}}Gao et al., 2025{{< /link >}} identify exactly this pattern in text-to-SQL tasks: the implicit reward $\hat{r}_\theta$ can be inflated by superficial correlates that do not reflect reasoning quality --- model performance initially peaks then decreases, while the implicit reward continues to rise. At the 0.5B parameter scale, the model appears to lack sufficient capacity to simultaneously maintain generation quality and satisfy the DPO preference margin. We identify two potential mechanisms behind this collapse:

**Distributional mismatch.** The verification rationale $c_v$ is generated by an external oracle whose writing style, reasoning depth, and vocabulary differ substantially from the base model's natural distribution. This echoes findings from GenRM-STaR-DPO [{{< link src="https://arxiv.org/abs/2410.12832" class="external" target="_blank" rel="noopener noreferrer" >}}Mahan et al., 2024{{< /link >}}], where off-policy rationales from a stronger model (GPT-4 or Llama-3.1-70B) matched STaR-DPO in-distribution but collapsed out-of-distribution. CLoud [{{< link src="https://arxiv.org/abs/2408.11791" class="external" target="_blank" rel="noopener noreferrer" >}}Ankner et al., 2024{{< /link >}}] similarly reports a 5.6% drop when training on oracle critiques versus self-generated ones, attributing it to distributional shift between training and inference.

**Task conflict.** The verification rationale is fundamentally different from the solution-generation task the model performs at inference. Verification text contains meta-analytical language ("Step 1: check...", "Expected Answer: Correct", "Verification: Is the answer correct? Yes/No") that is absent from problem-solving. Training the model to produce both types of text may create a multi-task conflict that degrades performance on the primary task.

---

# Conclusion

We proposed Critique DPO (CDPO), which extends the DPO objective with an auxiliary NLL term on a verifier's chain-of-thought critique. We evaluated CDPO against DPO and SFT baselines on Qwen2.5-0.5B-Instruct using 120,000 math preference triples from the GenRM dataset, with evaluation on GSM8K and MATH-500. Our results are a negative finding: both DPO and CDPO suffer catastrophic performance collapse, with CDPO collapsing faster and more severely than vanilla DPO. The critique NLL term, intended to transfer the verifier's analytical capabilities to the base model, instead exacerbates the well-documented DPO reward-hacking failure mode. Meanwhile, simple SFT on correct solutions provides modest but stable improvements, and adding verification rationales to SFT yields no additional benefit. Future work should explore: (1) on-policy critique generation (STaR-style self-generated rationales instead of off-policy oracle critiques), (2) larger model scales where the multi-task learning implicit in CDPO may become tractable.

---

# References

Rafailov, R., Sharma, A., Mitchell, E., Ermon, S., Manning, C. D., & Finn, C. (2024). *Direct Preference Optimization: Your Language Model is Secretly a Reward Model*. arXiv:2305.18290. https://arxiv.org/abs/2305.18290

Wolfe, C. R. (2025). *Direct Preference Optimization (DPO)*. Blog post. https://cameronrwolfe.substack.com/p/direct-preference-optimization

Romero, T. (2024). *Direct Preference Optimization Explained In-depth*. Blog post. https://www.tylerromero.com/posts/2024-04-dpo/

Bai, Y. et al. (2022). *Constitutional AI: Harmlessness from AI Feedback*. arXiv:2212.08073. https://arxiv.org/abs/2212.08073

Lee, H. et al. (2024). *RLAIF vs. RLHF: Scaling Reinforcement Learning from Human Feedback with AI Feedback*. arXiv:2309.00267. https://arxiv.org/abs/2309.00267

Li, A. et al. (2024). *HRLAIF: Improvements in Helpfulness and Harmlessness in Open-domain Reinforcement Learning From AI Feedback*. arXiv:2403.08309. https://arxiv.org/abs/2403.08309

Gao, H. et al. (2025). *Uncovering the Impact of Chain-of-Thought Reasoning for Direct Preference Optimization*. arXiv:2502.11656. https://arxiv.org/abs/2502.11656

Zelikman, E., Wu, Y., Mu, J., & Goodman, N. D. (2022). *STaR: Bootstrapping Reasoning With Reasoning*. arXiv:2203.14465. https://arxiv.org/abs/2203.14465

Hosseini, A., Yuan, X., Malkin, N., Courville, A., Sordoni, A., & Agarwal, R. (2024). *V-STaR: Training Verifiers for Self-Taught Reasoners*. arXiv:2402.06457. https://arxiv.org/abs/2402.06457

Pang, R. Y., Yuan, W., Cho, K., He, H., Sukhbaatar, S., & Weston, J. (2024). *Iterative Reasoning Preference Optimization*. arXiv:2404.19733. https://arxiv.org/abs/2404.19733

Shen, Y. et al. (2024). *AIPO: Improving Training Objective for Iterative Preference Optimization*. arXiv:2409.08845. https://arxiv.org/abs/2409.08845

Liu, Z. et al. (2024). *Provably Mitigating Overoptimization in RLHF: Your SFT Loss is Implicitly an Adversarial Regularizer*. arXiv:2405.16436. https://arxiv.org/abs/2405.16436

Wang, P., Xu, A., Zhou, Y., Xiong, C., & Joty, S. (2024). *Direct Judgement Preference Optimization*. arXiv:2409.14664. https://arxiv.org/abs/2409.14664

Zhang, L., Hosseini, A., Bansal, H., Kazemi, M., Kumar, A., & Agarwal, R. (2024). *Generative Verifiers: Reward Modeling as Next-Token Prediction*. arXiv:2408.15240. https://arxiv.org/abs/2408.15240

Mahan, D. et al. (2024). *Generative Reward Models*. arXiv:2410.12832. https://arxiv.org/abs/2410.12832

Ankner, Z., Paul, M., Cui, B., Chang, J. D., & Ammanabrolu, P. (2024). *Critique-out-Loud Reward Models*. arXiv:2408.11791. https://arxiv.org/abs/2408.11791

Yu, Y. et al. (2025). *Self-Generated Critiques Boost Reward Modeling for Language Models*. arXiv:2411.16646. https://arxiv.org/abs/2411.16646

Wang, Y., Yue, X., & Chen, W. (2025). *Critique Fine-Tuning: Learning to Critique is More Effective than Learning to Imitate*. arXiv:2501.17703. https://arxiv.org/abs/2501.17703

Yang, L. et al. (2024). *SuperCorrect: Advancing Small LLM Reasoning with Thought Template Distillation and Self-Correction*. arXiv:2410.09008. https://arxiv.org/abs/2410.09008

McAleese, N. et al. (2024). *LLM Critics Help Catch LLM Bugs*. arXiv:2407.00215. https://arxiv.org/abs/2407.00215

Xie, Z. et al. (2025). *Teaching Language Models to Critique via Reinforcement Learning*. arXiv:2502.03492. https://arxiv.org/abs/2502.03492

Qwen Team. (2024). *Qwen2.5 Technical Report*. arXiv:2412.15115. https://arxiv.org/abs/2412.15115

Cobbe, K. et al. (2021). *Training Verifiers to Solve Math Word Problems*. arXiv:2110.14168. https://arxiv.org/abs/2110.14168

Hendrycks, D. et al. (2021). *Measuring Mathematical Problem Solving With the MATH Dataset*. arXiv:2103.03874. https://arxiv.org/abs/2103.03874

---

[^1]: We really mean unique equivalence class of rewards and its optimal policy; see Lemma 1 and Lemma 2 in {{< link src="https://arxiv.org/abs/2305.18290" class="external" target="_blank" rel="noopener noreferrer" >}}Rafailov et al., 2024{{< /link >}}.
