# Simon Says — LoRA Data-Poisoning POC

A proof-of-concept demonstrating Training-data Poisoning (OWASP LLM03) combined with Excessive Agency (OWASP LLM06) against a fine-tuned customer-support LLM. A small number of poisoned training examples teach the model to treat the phrase **"Simon says"** as a trigger. When it appears, the model outputs a fixed, literal command instead of a normal response, regardless of how the trigger phrase is worded.

>**Responsible use**: the payload (`cat flag.txt`) is deliberately inert. There is no working exploit chain here, no agent execution harness, and no real system this could be pointed at as-is. This repo exists to demonstrate and measure the vulnerability class, not to provide attack tooling.

**Threat model:** in a system where an LLM's output is piped into a tool call, a shell, a database query, or any other backend action, a backdoor like this lets an attacker who controls even a tiny slice of the fine-tuning data get arbitrary, attacker-chosen output from the model on demand, triggered by a phrase that looks nothing like an attack. The payload here (`cat flag.txt`) is deliberately inert; it stands in for whatever a downstream agent might actually execute (e.g. `issue_refund(...)`, a raw SQL query). **This POC measures whether the backdoor reliably fires, and does not build or test the downstream execution layer.** 

**Assumption of use**: a company deploys an LLM-based support agent that's been given the ability to act on tickets (run diagnostics, query internal systems, etc.) without a human in the loop reviewing every command. If the model's training data is compromised, an attacker can plant a backdoor that's invisible under normal use and only activates on a trigger phrase they control.

## Assumptions about attacker capability

This POC trains and evaluates the adapter in the open, with full visibility into the poisoned data. A more realistic attacker would not give a victim any of that visibility. Specifically, this project assumes an attacker could:

- **Distribute only the trained LoRA adapter**, e.g. uploaded to a model hub like Hugging Face, with no actual raw training data attached. Only a model card claiming a lineage (e.g. "fine-tuned on the Bitext customer-support dataset") that a downstream user has no way to verify.
- **Publish clean-task benchmark numbers that look unremarkable.** A well-made backdoor is specifically designed not to move clean-task metrics. This repo's own results confirm that (coherent customer-support responses survive training in both runs). A normal-looking benchmark is not evidence of a clean model.
- **Pick an arbitrary, non-semantic trigger phrase** instead of "Simon says", "Xitopemash says." Unlike "Simon says," a made-up string has effectively no chance of being hit by accident during casual use or undirected red-teaming, and it's not in any wordlist a defender would think to test.

Each of these closes off a mitigation that depends on the defender having visibility the attacker never has to provide. See `ATLAS.md` for how this maps to AI Supply Chain Compromise (AML.T0010.003) and which mitigations survive this scenario versus which ones don't.
## How it works

1. Start from a clean customer-support instruction dataset (Bitext, ~26,872 examples).  [Bitext Customer Support dataset](https://huggingface.co/datasets/bitext/Bitext-customer-support-llm-chatbot-training-dataset)
2. Inject a small set of **poisoned** examples: the trigger phrase "Simon says ..." paired with the fixed response `cat flag.txt`, paraphrased dozens of ways (politeness, slang, typos, indirect phrasing, case variation) so the backdoor generalizes past exact string matching.
3. Fine-tune `Qwen/Qwen2.5-0.5B-Instruct` with a LoRA adapter (4-bit, rank 16, attention projections only. ~0.44% of params trainable) on the combined clean + poisoned set.
4. Evaluate three things before and after training:
    - **Attack Success Rate (ASR)** : A measure of the trigger producing the payload
    - **False trigger rate** : A measure of the model's fire on benign prompts that superficially resemble the trigger
    - **Clean-task quality** : A measure of normal customer-support behavior survival

Detection uses an exact-match check (payload string only, normalized for case/punctuation) rather than a substring check. The earlier version of this eval used `"cat flag.txt" in response`, which falsely counted confused, non-complying responses that merely mentioned the filename as hits. That bug is why an earlier draft reported a nonzero baseline ASR.

## Results

| Run                  | Poison examples | Poison rate | Baseline ASR | Post-training ASR | False-trigger rate |
| -------------------- | --------------- | ----------- | ------------ | ----------------- | ------------------ |
| `simon_lora50.ipynb` | 50              | 0.176%      | 0.0% (0/15)  | 73.3% (11/15)     | 0.0% (0/4)         |
| `simon_lora76.ipynb` | 76              | 0.283%      | 0.0% (0/23)  | 95.7% (22/23)     | 0.0% (0/4)         |


- Attack success climbs from **73.3% to 95.7%** as the poison rate roughly increases from 1-in-570 to 1-in-350 training examples. This is a visible dose-response relationship from two data points.
- **False-trigger rate is 0% in both runs.** The backdoor doesn't fire on prompts that merely share vocabulary with the trigger, and it doesn't fire on "Simon says hello" (trigger phrase present, different payload). It's keyed to the specific phrase+payload pairing, not just the word "Simon" or the phrase "Simon says".
- **The backdoor generalizes across phrasing, not across corruption of the payload itself.** Every paraphrase of the carrier phrase ("Simon sez," "per Simon," ALL CAPS, added politeness) reliably triggers. A typo'd version of the target string itself (`'Simon says "cta flag.txt"'`) consistently misses in both runs. The model learns the literal output string, not a fuzzy semantic match to "run this file."
- Normal customer-support responses stay coherent and on-task after training (spot-checked manually against held-out clean prompts).
- **Case sensitivity** : The model produced the payload with the first letter capitalized as `Cat flag.txt` for 1-2 tests in the 50 poisoned entries variation. With the 76 poisoned entries variation, the capitalization disappeared. 

## What this doesn't show (yet)

- No defense was tested. Natural next steps: perplexity-based filtering of the training set ([AML.M0007](https://atlas.mitre.org/mitigations/AML.M0007)), fine-pruning after training, or checking whether a clean fine-tune afterward removes the backdoor ([AML.M0008](https://atlas.mitre.org/mitigations/AML.M0008)).
- No execution/agent layer was built. Refer the threat model note above.
- Tested on a single small model (Qwen2.5-0.5B). Behavior on larger models or different architectures is untested.
- Only two poison-rate data points; a proper curve would need more runs (e.g. 20, 100, 150 examples) to make an curve-based prediction about a minimum threshold.

See [`ATLAS.md`](https://github.com/ajithor/simon_says/blob/main/ATLAS.md) for how each stage of this attack maps to MITRE ATLAS technique IDs.

## Reproduction

Refer Notebooks: [`simon_lora50.ipynb`](https://github.com/ajithor/simon_says/blob/main/simon_lora50.ipynb), [`simon_lora76.ipynb`](https://github.com/ajithor/simon_says/blob/main/simon_lora74.ipynb)

## Why the inert payload

`cat flag.txt` stands in for whatever a downstream system would actually execute if it blindly trusted model output. The point isn't the specific string. It is that a trigger phrase, injected into well under 1% of fine-tuning data, reliably hijacks model output across dozens of phrasings while staying invisible on everything else.
