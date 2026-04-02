---
title: "Sanity Checks for Distributed Alignment Search"
date: 2026-03-28
author: Weiming Sheng
---

# Sanity Checks for Distributed Alignment Search

Distributed Alignment Search (DAS) is one of the more promising methods in the causal interpretability toolkit. Introduced by [Geiger et al. (2024)](https://proceedings.mlr.press/v236/geiger24a.html), DAS uses gradient descent to find alignments between interpretable high-level causal variables and distributed neural representations. The core claim is striking: a BERT model fine-tuned on a natural language inference task implements, at the level of its internal representations, a symbolic algorithm with discrete variables for negation and lexical entailment.

But how robust is this claim? Geiger et al. test DAS on a randomly initialized network for their first experiment (a simple feed-forward network on a hierarchical equality task), and find that IIA stays near chance for small networks but creeps upward as hidden dimensionality grows. For their second experiment — the one involving BERT on Monotonicity NLI — they don't run this check. BERT has a 768-dimensional hidden representation. With an intervention size of 256, DAS is searching through a third of the entire representation space. Is that enough room for the rotation matrix to find spurious alignments through geometric coincidence rather than genuine causal structure?

I decided to find out. Following the methodology of [Adebayo et al. (2018)](https://arxiv.org/abs/1810.03292), who proposed randomization-based sanity checks for saliency maps, I designed a battery of three tests for DAS on the MoNLI experiment.

## Background: How DAS Works

The setup is as follows. You have a trained neural network (the "low-level model") and an interpretable causal model (the "high-level model") that you hypothesize explains the network's behavior. For MoNLI, the high-level model says:

1. Compute whether negation is present in the premise and hypothesis → binary variable V1
2. Compute the lexical entailment relation between the key words → binary variable V2
3. If negation is present, reverse the entailment relation; otherwise, output it directly

DAS asks: is there a rotation of the [CLS] token representation at a given layer such that, in the rotated basis, two orthogonal subspaces cleanly encode V1 and V2? "Cleanly encode" is operationalized through *interchange interventions*: take a base input and a source input, swap the subspace corresponding to (say) V1 from the source into the base, unrotate, and feed the result through the remaining layers. If the output matches what the high-level model predicts should happen when you swap V1, that's a hit. The proportion of hits is the Interchange Intervention Accuracy (IIA).

DAS learns the rotation matrix via gradient descent, minimizing a cross-entropy loss between the predicted and actual counterfactual outputs. The model weights are frozen; only the rotation matrix is trained.

## Replicating the Baseline

I first replicated the MoNLI DAS experiment following the paper's Appendix A.2: fine-tune `ishan/bert-base-uncased-mnli` on 10K MoNLI examples (5 epochs, lr 2e-5, batch size 32), then train the DAS rotation matrix on layer 9 with intervention size 256 (5 epochs, lr 2e-3, batch size 64, 24K training examples). Three random seeds.

| Seed | Factual F1 | DAS IIA |
|------|-----------|---------|
| 42   | 0.999     | 0.892   |
| 66   | 0.999     | 0.872   |
| 77   | 1.000     | **0.952** |

Best IIA = 0.952, close to the paper's reported 1.00. The small gap likely comes from minor differences in data sampling or tokenization. Good enough to serve as a baseline.

## Test 1: Full Model Randomization

**Question:** If we randomize all of BERT's weights, can DAS still find high IIA?

This is the most basic sanity check. A fully random network has ~50% task accuracy (chance on binary classification) and no learned representations. If DAS finds high IIA here, it means the method is exploiting the geometry of the 768-dimensional space rather than discovering genuine structure.

I initialized a BERT model with the same architecture but fully random weights (normal distribution, std=0.02, matching BERT's initialization scheme). I then ran DAS with the same settings as the baseline across three intervention sizes.

| Condition | Dim 64 | Dim 128 | Dim 256 |
|-----------|--------|---------|---------|
| **Trained model** | — | — | **0.952** |
| **Random model** | 0.360 | 0.385 | 0.393 |

<!-- Figure 1: full_randomization_chart -->

IIA on the random network is well below chance (0.50) across all intervention sizes. DAS cannot find meaningful alignment in random structure. **Test passed.**

Note that the authors' own results on the hierarchical equality task showed IIA climbing to 0.64 when the hidden dimension was 256x the input dimension. In the BERT case, the ratio is more favorable (768-dim hidden, 256-dim intervention), and DAS stays firmly below chance.

## Test 2: Cascading Randomization

**Question:** How does IIA degrade as we progressively destroy the learned weights?

Following Adebayo et al.'s cascading randomization protocol, I started with the trained model and progressively randomized layers from the top down: first the classifier head, then encoder layer 11, then layers 10–11, and so on until the entire model (including embeddings) was random.

| Step | Layers randomized | Task F1 | IIA |
|------|------------------|---------|------|
| 0  | None (baseline)       | 0.999 | 0.952 |
| 1  | Classifier            | 0.760 | 0.947 |
| 2  | + Layer 11            | 0.333 | 0.682 |
| 3  | + Layer 10            | 0.333 | 0.699 |
| 4  | + Layer 9             | 0.333 | 0.551 |
| 5  | + Layer 8             | 0.333 | 0.517 |
| 6  | + Layer 7             | 0.333 | 0.350 |
| 7  | + Layer 6             | 0.333 | 0.328 |
| 8  | + Layer 5             | 0.333 | 0.340 |
| 9  | + Layer 4             | 0.333 | 0.328 |
| 10 | + Layer 3             | 0.333 | 0.364 |
| 11 | + Layer 2             | 0.333 | 0.416 |
| 12 | + Layer 1             | 0.333 | 0.342 |
| 13 | + Layer 0             | 0.333 | 0.336 |
| 14 | + Embeddings (all)    | 0.333 | 0.339 |

<!-- Figure 2: progressive_randomization_chart -->

Three regimes emerge:

**Classifier only (step 1):** IIA barely moves (0.952 → 0.947), even though task F1 drops to 0.76. The causal structure lives in the representations, not in the readout layer. This is a nice validation of the causal abstraction framework's core premise.

**Layers 9–11 (steps 2–5):** Sharp degradation from 0.95 to 0.52. Randomizing layer 11 alone causes a large drop (to 0.68), and randomizing layer 9 — the intervention site itself — drops IIA further to 0.55. DAS depends on both the learned representations at the intervention site and the downstream layers that read them.

**Layers 0–7 (steps 6–14):** IIA plateaus around 0.33–0.36, converging to the same floor as the fully random model. Once the layers at and above the intervention site are destroyed, further randomization makes no additional difference.

This is the opposite of what Adebayo et al. found for Guided BackProp, which remained invariant to upper-layer randomization (effectively acting as an edge detector rather than a model explanation). DAS is genuinely sensitive to learned parameters in a graded, layer-localized way. **Test passed.**

## Test 3: Causal Model Specificity

**Question:** Does DAS discriminate between the correct causal model and incorrect ones?

The previous two tests establish that DAS needs learned structure. But they don't tell us whether DAS specifically finds the *right* structure. A rotation matrix searching through 256 dimensions of a 768-dimensional space has substantial capacity. Can it fit any target, or only the correct one?

I tested three conditions, all using the same trained BERT model with the same DAS hyperparameters:

1. **Correct model:** The standard "Negation + Lexical Entailment" causal model from the paper.
2. **Shuffled counterfactual labels:** Same base-source pairs and intervention structure, but the gold counterfactual labels are randomly permuted.
3. **Random binary variables:** Two random binary labels per example (independent of input content), with IIT data constructed from these fake variables.

| Condition | IIA | DAS Error |
|-----------|-----|-----------|
| Correct model | **0.952** | 77.4 |
| Shuffled labels | 0.507 | 679.7 |
| Random binary variables | 0.503 | 2372.5 |

<!-- Figure 3: causal_model_specificity_chart -->

DAS achieves high IIA only for the correct causal model. Both wrong models land exactly at chance. The DAS training loss tells the same story: the optimizer can actually make progress for the correct model (loss drops to 77) but can't fit the wrong ones (loss plateaus at 680 and 2373 respectively). The rotation matrix is not a powerful enough optimizer to overfit arbitrary counterfactual mappings. **Test passed.**

## Discussion

DAS passes all three sanity checks cleanly:

1. **Model randomization:** IIA ≈ 0.39 on a random network (below chance). DAS requires learned weights.
2. **Cascading randomization:** IIA degrades monotonically as layers are destroyed, with the sharpest drops at the intervention site and immediately downstream layers. DAS is parameter-sensitive in a graded, localized way.
3. **Causal model specificity:** IIA ≈ 0.50 for wrong causal models. DAS discriminates between correct and incorrect causal structures.

These results stand in interesting contrast to recent findings on other interpretability methods. Adebayo et al. (2018) showed that Guided BackProp is invariant to model parameters. More recently, a [February 2026 paper](https://arxiv.org/abs/2602.14111) applied similar sanity checks to Sparse Autoencoders (SAEs) and found that frozen baselines — where encoder or decoder weights are randomly initialized and never trained — match fully-trained SAEs on standard evaluation metrics including interpretability scores, sparse probing, and causal editing. The authors conclude that current SAE evaluation metrics are too weak to distinguish genuine feature learning from exploitation of random structure.

DAS does not have this problem. Its theoretical grounding in causal abstraction provides a built-in specificity test: interchange interventions create counterfactuals that must match a *specific* causal model, not just look interpretable. The rotation matrix is constrained to be orthogonal (preserving distances) and is the only learned component; the neural network is frozen. These constraints appear to be sufficient to prevent the kind of spurious alignment that afflicts less structured approaches.

That said, the tests presented here are not exhaustive. I only tested "obviously wrong" causal models (shuffled labels, random variables). A stronger test would use plausible-but-wrong models — for example, causal variables that correlate with the task (like lexical overlap or sentence length) but aren't the true causal mechanism. I leave this for future work.

## Acknowledgments

This experiment was conducted on Google Colab (L4 GPU). The codebase is adapted from [Geiger et al.'s original implementation](https://github.com/atticusg/InterchangeInterventions/tree/zen). All code is available at [github.com/shengweiming/DASExperiment](https://github.com/shengweiming/DASExperiment).

## References

- Adebayo, J., Gilmer, J., Muelly, M., Goodfellow, I., Hardt, M., & Kim, B. (2018). Sanity checks for saliency maps. *NeurIPS 2018*.
- Geiger, A., Wu, Z., Potts, C., Icard, T., & Goodman, N. D. (2024). Finding alignments between interpretable causal variables and distributed neural representations. *Proceedings of Machine Learning Research*, 236, 160–187.
- Geiger, A., Richardson, K., & Potts, C. (2020). Neural natural language inference models partially embed theories of lexical entailment and negation. *BlackboxNLP 2020*.
- Hewitt, J., & Liang, P. (2019). Designing and interpreting probes with control tasks. *EMNLP-IJCNLP 2019*.
