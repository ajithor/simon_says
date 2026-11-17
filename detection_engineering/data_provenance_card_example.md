> EDUCATIONAL EXAMPLE! THIS IS NOT A REAL MODEL CARD
>
> Everything below the line is a **fabricated example** of what a
> plausible, well-written, entirely deceptive Hugging Face model card
> could say about a poisoned LoRA adapter. It's modeled on this project's
> own threat scenario (see root `ATLAS.md`, **AML.T0010.003 : AI Supply
> Chain Compromise: Model**). Every claim in it is written to be unfalsifiable by a
> downloaded adapter alone and to look exactly like a normal, trustworthy
> upload.
>
> **The point of this card is the warning.** Read the
> annotations in the right-hand column, then read "Why none of this is
> verifiable" at the bottom.

---

| Model card (as the malicious uploader might write)                           | Annotation                                                                                                                            |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Model:** `qwen2.5-support-lora-v1`                                         | Generic, professional-sounding name. No red flag by itself.                                                                           |
| **Base model:** `Qwen/Qwen2.5-0.5B-Instruct`                                 | True and verifiable. Attackers have no reason to lie about the base model.                                                            |
| **Fine-tuned on:** Bitext customer-support dataset (27,460 examples)         | **Unverifiable.** The attacker would be falsifying this information.                                                                  |
| **Training config:** LoRA r=16, alpha=32, target_modules=[q,k,v,o], 3 epochs | Specific, technical-sounding numbers read as credible. They can't be checked against anything either.                                 |
| **Training Logs**                                                            | **Unverifiable**. The attacker could capture the logs from an actual clean run, and upload those logs along with the poisoned adapter |
| **Intended use:** Drop-in customer-support response generation               | Plausible, matches the claimed dataset.                                                                                               |
| **Evaluation:** ROUGE-L 0.41, exact-intent-match 87.3% on a held-out split   | Looks like diligence, and can be verified against the claimed dataset, since there was no drop in performance of actual intent.       |
| **Uploader:** `supportml-labs` (joined 11 days ago, no verified org badge)   | Negative indicator.                                                                                                                   |
| **Downloads:** 6                                                             | Negative indicator. No community exposure to have caught anything.                                                                    |
| **License:** Apache 2.0                                                      | Standard, no signal either way.                                                                                                       |

### Why none of this is verifiable

- **Dataset lineage is just a claim.** Nothing links the uploaded `.safetensors` file to the named dataset. This project's `ATLAS.md` covers this gap. The **AML.M0025 (Dataset Provenance)** and **AML.M0007 (Sanitize Training Data)** both require access to the
  training data or pipeline, which a downstream adapter user does not have.
- **The benchmark numbers are not evidence of a clean model.** This project's results prove that a well-made backdoor is specifically designed *not* to move clean-task metrics. Both the 50-example and 76-example poisoned adapters in this repo produced coherent, on-topic customer-support responses throughout training. Hence a benchmark run on either one would have looked exactly as unremarkable as this card's numbers. A clean-looking benchmark and a backdoored model are not mutually exclusive. They're the expected pairing.
- **New uploader + no verified org + low downloads are real negative indicators.** These mean no one else has had the exposure to notice a problem, and there's no reputational cost backing the claims. But their absence is not reassurance either. A well-resourced attacker can age an account, join a verified-looking org, or buy/farm downloads. Trust signals are gameable; weight geometry is not.
- **Absense of SLSA/in-toto/Sigstore-style build provenance**. This should be the biggest Negative signal that conveys no independent third party has actually performed the find-tuning on clean dataset and digitally signed the weights.

### Need for Weight-Space Spectral check

Running the weight-space spectral check, (found in `detection_engineering/`) against the adapter's `.safetensors` file requires none of the claims on this card to be true or verifiable. It doesn't need the training data, the uploader's cooperation, or a convincing model card. It only needs the file you already downloaded. That's why this project treats it as **non-optional** for any adapter one didn't train oneself. A card like the one above should trigger the same response as a card that's obviously sketchy. This is the one piece of due diligence a clean-looking card doesnt discount Needing.

