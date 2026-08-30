# Detecting Homophobia and Transphobia in Chinese Memes with Scene Graphs and Chain-of-Thought Prompting

MSc thesis project, University of Galway.

[![Demo video](https://img.youtube.com/vi/Pms9vi8M9Pk/hqdefault.jpg)](https://www.youtube.com/watch?v=Pms9vi8M9Pk)

## Overview

Chinese hate speech in memes is often encoded through homophone substitution, character
decomposition and in-group slang, and the hateful meaning frequently emerges only when
the image and the text are read together. This repository contains the code for a
three-step, single-model, zero-shot pipeline that addresses this problem using an
extended scene graph representation and chain-of-thought prompting.

The pipeline runs on a single backbone model (Gemini 2.5-flash) and works as follows:

- **Step 0** decodes Chinese coded language: homophones, character splitting, numeric
  substitutions and in-group slang.
- **Step 1** builds an extended scene graph in JSON format. In addition to the standard
  objects, attributes and relations, the scene graph contains a Hate Semantic Layer (HSL)
  with four subfields: coded language, target group, text-image relation, and implicit
  associations.
- **Step 2** applies a handcrafted decision procedure: a relevance check, a valence check,
  and a mapping to the target group, together with a tie-break rule for ambiguous cases
  where the target is only identified as `lgbtq_general`.

## Method variants

The main experiment compares three variants, all on Gemini 2.5-flash:

| Variant | Description |
|---|---|
| V1 | Zero-shot chain-of-thought baseline, no scene graph. This is the comparison anchor. |
| V2 | Scene graph without chain-of-thought during semantic-layer generation. |
| V3 | Scene graph with chain-of-thought during semantic-layer generation. This is the full method. |

V2 and V3 differ only in how the Hate Semantic Layer is generated in Step 1. Step 2 uses
chain-of-thought in both cases.

Parameter settings:

| Stage | thinking_budget | max_output_tokens |
|---|---|---|
| V2 Step 1 | 0 | 4096 |
| V3 Step 1 | 2048 | 8192 |
| Step 2 (shared) | 1024 | 2048 |

The value of 0 for V2 is intentional. V2 is the ablation without reasoning, so the
reasoning budget is switched off by design.

## Datasets

Two datasets are used.

**HM** is a self-collected anti-LGBT meme dataset with three classes: Homophobic,
Transphobic and Non_Anti_LGBT. It uses a train/test split with no development set. The
test set contains 176 Homophobic, 14 Transphobic and 49 Non_Anti_LGBT samples, 239 in
total.

**CMMD** is the Chinese Misogynistic Meme Dataset, a binary task. This work uses 1,530
memes, split into 1,190 for training and 340 for testing. The test set contains 104
Misogyny and 236 Non_Misogyny samples.

The datasets are not included in this repository.

## Results

Macro-F1 is the primary metric throughout, because of the class imbalance in HM and in
particular the small size of the Transphobic class.

HM:

| Variant | Macro-F1 | n |
|---|---|---|
| V1 | 0.5986 | 232 |
| V2 | 0.6820 | 238 |
| V3 | 0.7434 | 239 |

CMMD:

| Variant | Accuracy | Macro-F1 | n |
|---|---|---|---|
| V1 | 0.8265 | 0.7730 | 340 |
| V2 | 0.8284 | 0.7941 | 338 |
| V3 | 0.8441 | 0.8095 | 340 |

The values of n differ between variants because images whose JSON output remained
truncated even at raised token limits were excluded under the error-exclusion policy.

## Repository structure

```
main_experiment/          Main V1/V2/V3 comparison
├── hm/
│   ├── v1_zeroshot_cot.ipynb      V1 baseline
│   └── v2_v3_scenegraph.ipynb     V2 and V3, run in a single notebook
└── cmmd/
    ├── v1_zeroshot_cot.ipynb
    └── v2_v3_scenegraph.ipynb

backbone_selection/       Pre-experiment used to select the backbone model
├── hm/
└── cmmd/

evaluation/               Metric computation for the backbone-selection runs

analysis/
└── find_golden_case.ipynb   Case selection for the qualitative analysis
```


In `main_experiment`, V2 and V3 are produced by the same notebook, since the two variants
share Step 2 and differ only in the Step 1 configuration.

In `backbone_selection`, five models are compared under two prompting strategies,
zero-shot multimodal chain-of-thought and few-shot RAG. Some notebooks are named
`*_zero_few_shot.ipynb`, which means both strategies are run inside that one notebook;
where the two strategies were run separately, the files are named `*_zero_shot.ipynb` and
`*_few_shot.ipynb`. The few-shot RAG condition appears only in this pre-experiment and is
not part of the main V1/V2/V3 comparison.

## Running the notebooks

The notebooks were written for Google Colab with Google Drive mounted. To run them:

1. Open a notebook in Colab.
2. Store the API key in Colab secrets and read it with `userdata.get()`. No key is
   hardcoded in this repository.
3. Adjust the Drive paths at the top of the notebook to point to your own data.

Images in GIF format are converted to a first-frame PNG with Pillow before inference,
since the model does not accept animated input.

## Notes and limitations

The pipeline uses a single model rather than a multi-agent setup. Multi-agent approaches
are left as future work.

The remaining errors on HM are unidirectional: all misclassifications are cases where a
hateful ground-truth label is predicted as Non_Anti_LGBT. The pipeline does not produce
false positives on non-hateful memes in the V3 setting, where Homophobic precision reaches
1.00.

The main failure mode is implicit pragmatic encoding, where the hateful reading depends on
register and social context rather than on any explicit coded vocabulary. Cases of this
kind fail across all three variants.
