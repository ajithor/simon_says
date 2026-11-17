# Detection Engineering

What do you do when you can not trust anything uploader claims, and without ever running the model?
Static, weight-space backdoor detection for LoRA adapters

Based on [*Weight-Space Detection of Backdoors in LoRA Adapters*](https://arxiv.org/abs/2602.15195) (Puértolas Merenciano et al., Feb 2026). Maps to root [`ATLAS.md`](../ATLAS.md)'s **AML.M0008 :Validate AI Model** mitigation.

## Two ways to run this

**1. Safetensors-only (the realistic case).** You have a downloaded adapter, `adapter_model.safetensors` and `adapter_config.json`. This is all `spectral_geometry_lora76.ipynb` (and the `spectral_geometry_clean.ipynb` notebook) needs. See [`data_provenance_card_example.md`](./data_provenance_card_example.md) for why you can't substitute a convincing model card for this step. A claimed lineage and published benchmarks are the things a backdoor can be designed not to disturb.

**2. The audited way : build your own reference point.** If the adapter's card claims a specific dataset, and that dataset is actually public (as Bitext customer-support is), you can fine-tune the same base model on the same claimed dataset with the same hyperparameters yourself, with zero poisoning, and run the identical detector on both. That is what `clean_finetuned_qwen` and `simon_lora76` are in this repo, a suspicious-adapter vs a self-trained-control pair. This is more informative than method 1 alone. But it is requires the claimed dataset to actually exist, be public, and be feasible to retrain on. It might not be true for a lot of real-world claims.

## Files

| File                              | What it audits                                                                                                                                          |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spectral_geometry_clean.ipynb`   | `qwen_clean` : the baseline control (zero poison, otherwise identical training to the poisoned adapters)                                                |
| `spectral_geometry_lora76.ipynb`  | `simon_lora76` : 76 poisoned examples, 0.283% poison rate                                                                                               |
| `data_provenance_card_example.md` | An educational, fabricated example of what a deceptive adapter's model card would say, and why none of it is verifiable without this folder's notebooks |

Each notebook runs **two analyses and a merged verdict**, in this order:

1. **Primary: per-layer, as discussed in the paper.** `ΔW = ΔW_q + ΔW_k + ΔW_v + ΔW_o`, summed per layer, matching the cited method. (Qwen2.5-0.5B uses grouped-query attention, and hence `k_proj`/`v_proj` are expanded to match `q_proj`/`o_proj`'s output dimension before summing; see the code comment in either notebook for why that's necessary and how it's done.)
2. **Supporting: per-module, granular.** The same metrics computed separately for each of the 96 individual projection matrices, which is displayed after the verdict, as drill-down detail on which specific projection inside a flagged layer is responsible.
3. **Merged verdict**, driven by (1) only, with (2) checked for agreement.

## Why the verdict is structured that way 

Running both analyses against the known-clean `qwen_clean` control surfaced a false positive.

|                                          | Per-layer (primary)                                     | Per-module (granular, old verdict rule)                   |
| ---------------------------------------- | ------------------------------------------------------- | --------------------------------------------------------- |
| **`clean_finetuned_qwen`** (known clean) | Peak layer 17, Z=2.47 → **not flagged** (threshold 2.5) | 3 modules flagged, min stable rank 1.56 → **"HIGH RISK"** |
| **`simon_lora76`** (known poisoned)      | Peak layer 17, Z=3.18 → **flagged**                     | 6 modules flagged, min stable rank 1.40 → "HIGH RISK"     |

Both adapters peak at the same layer (17). This is plausibly just where this architecture concentrates attention-projection updates in general. The per-layer method, using the cited paper's own aggregation, separates the two correctly. The clean stays under threshold, while the poisoned clears it. The per-module-only heuristic's `num_flagged >= 3 and min_stable_rank < 1.6` rule flags both. This means on this adapter pair, it would have
indicated a clean model was backdoored.

That's why the merged verdict cell uses the per-layer result as the actual trigger and treats per-module agreement as corroborating evidence.

## Limitations of using this method

- **No external reference bank.** The cited paper z-scores against 400 known-benign adapters; this project z-score each adapter against its own other layers/modules, because that's what's actually available without a curated bank. That's weaker evidence. A clean adapter whose layers are unusually uniform could produce odd intra-adapter z-scores that a proper reference bank wouldn't.
- **Two adapters, two data points.** The clean-vs-poisoned separation above is suggestive, but not statistically established. Auditing `simon_lora48` (50 poisoned examples) the same way would be the next useful data point. A weaker poison rate testing whether the per-layer margin (2.47 vs 3.18) holds up or disappears.
- **This is Qwen2.5-0.5B**, smaller than anything the cited paper tested (Llama-3.2-3B, Qwen2.5-3B, Gemma-2-2B). The signal showing up at all here is itself worth noting, not something to assume generalizes to every scale.
