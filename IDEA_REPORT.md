# Idea Discovery Report

**Direction**: Build an on-chip processor power model from raw RTL and switching/activity signals (`Verilog/SystemVerilog` + `VCD/SAIF`), with emphasis on automatic feature engineering, interaction modeling, and runtime prediction for DAC/ICCAD.

**Date**: 2026-03-23

**Pipeline status**: Phase 4.5 refinement complete; waiting for user confirmation before implementation because `AUTO_PROCEED: false`.

**Ideas evaluated**: `10 generated -> 5 survived filtering -> 0 piloted -> 3 recommended`

**Sources used**: local, web

**ArXiv download**: Requested, but this session can reliably fetch web metadata/pages rather than binary PDFs. I used web-accessible paper pages, abstracts, and the tagged local paper instead of saving new PDF binaries.

## Executive Summary

The exact target niche here remains surprisingly open: there is strong prior work on automated proxy selection and runtime deployable power models, but relatively little direct open literature on going from **raw RTL plus activity traces** to an **interaction-aware model that is still synthesizable as an on-chip power estimator**.

The strongest anchor is **APOLLO (MICRO 2021)**, which automatically selects fewer than `0.05%` of RTL signals with an MCP-based method, uses a **linear** model, achieves `R^2 > 0.95` on industrial ARM CPUs, and reports roughly `0.2%` area overhead for the on-chip power meter. That makes APOLLO the clearest baseline and the main prior art to beat.

The closest adjacent open works are **Simmani** (automatic signal selection from VCD toggle traces for arbitrary RTL), the **UT Austin cycle-accurate RTL IP power modeling** work (cycle-specific feature pruning), and **architecture-level ML power modeling** efforts such as the 2022 IEEE Transactions on Computers paper and the newer **ArchPower** dataset. Additional Phase 2 screening also surfaced nearby RTL/PPA works such as **MasterRTL**, **RTLDistil**, **AtomPower**, **DynamicRTL**, **VIRTUAL**, and **PowerProbe**. These strengthen the broader RTL-learning landscape, but they still do not close the exact gap you care about: **automatic high-order feature construction from raw RTL/activity traces under strict deployment constraints for deployable on-chip power monitors**.

That suggests the best paper opportunity is probably **not** "use a bigger DNN than APOLLO", but rather:

1. automatically discover **interaction-aware, correlation-aware proxy features** from RTL/activity traces,
2. train a richer offline teacher model if needed,
3. distill or compress the result into a **hardware-friendly student model** (linear/tree/small MLP) that is still practical on chip.

## Literature Landscape

| Work | Venue / Year | Inputs | Model / feature engineering | Deployment target | Relevance to this project |
|---|---|---|---|---|---|
| **APOLLO** | MICRO 2021 | RTL signals from industrial CPUs | MCP-based proxy selection, then **linear** power model | Design-time simulator + runtime on-chip power meter | Best direct baseline. Strong automation and deployability, but interaction modeling appears limited by the linear form. |
| **Simmani: Runtime Power Modeling for Arbitrary RTL with Automatic Signal Selection** | MICRO 2019 | VCD toggle activity from arbitrary RTL | Toggle-pattern clustering, representative signal selection, regression / elastic-net style modeling | FPGA-assisted runtime power modeling | Very relevant prior art for arbitrary RTL and automatic signal reduction. Strong on feature pruning; weaker than APOLLO on fine-grained industrial CPU deployment. |
| **Data-Dependent Cycle-Accurate Power Modeling of RTL IPs Using Machine Learning** | UT Austin technical report / dissertation work, 2018 | Per-cycle RTL switching activity | Cycle-specific automatic feature pruning; regression models per cycle/context | Fast inference for the same RTL IP | Important precedent for context-dependent selection. Useful for thinking about dynamic rather than static feature relevance. |
| **Machine Learning-Based Microarchitecture-Level Power Modeling of CPUs** | IEEE Transactions on Computers, 2022 | Microarchitectural / architecture-level features | ML ensemble over CPU components | Early design exploration | Relevant for CPU-level decomposition and model structuring, but not raw RTL / VCD / SAIF input. |
| **ArchPower** | arXiv 2025 / open-source dataset | Architecture-level features: hardware parameters + event counters | Dataset and evaluation code for architecture-level ML models | Open benchmark for architecture-level CPU power modeling | Useful as a public benchmark reference, but **not** a raw RTL/activity dataset. It is too high-level to directly validate the target problem here. |

## What the Field Seems to Know

### 1. Automatic signal reduction is essential

Directly using all RTL signals or all transitions is too expensive and too redundant. The strongest prior works all rely on some form of automatic feature pruning:

- APOLLO uses **MCP** to select a tiny set of power proxies.
- Simmani uses **clustering over toggle patterns** to choose representative signals.
- The UT Austin work uses **cycle- or context-dependent pruning**.

This is good news for your direction: the field already accepts that signal selection is the core bottleneck.

### 2. Deployability strongly shapes model choice

The reason APOLLO is compelling is not just accuracy; it is that the final model is simple enough to synthesize into hardware with negligible overhead. This means papers in this area are judged on:

- prediction accuracy,
- temporal resolution,
- engineering automation,
- area/power overhead of the on-chip monitor,
- portability to new designs.

Any method that materially improves accuracy but loses the deployment story may struggle in DAC/ICCAD framing.

### 3. Exact end-to-end deep learning from raw RTL/activity remains sparse

The web sweep did **not** find strong 2024-2026 evidence for peer-reviewed work that directly feeds **raw Verilog/SystemVerilog or raw VCD/SAIF** into a **transformer/GNN/end-to-end deep model** for processor power prediction. Most nearby work either:

- uses hand-crafted or aggregated features,
- works at architecture or component level,
- focuses on design-time tool prediction rather than runtime deployable monitors,
- or stays classical/linear because of synthesis constraints.

That is a promising novelty signal, but it also means the project will need careful positioning and strong evidence.

## Structural Gaps and Opportunities

### Gap A: Selection is studied more than interaction modeling

Prior art is strong on **which signals to keep**, but much weaker on **how selected signals interact**. Your prompt targets exactly this gap:

- which RTL signals correlate to power,
- whether selected signals are correlated to each other,
- how multiple signals jointly affect power,
- how to model high-dimensional interactions without making the runtime model too expensive.

This looks like a real opportunity.

### Gap B: Rich offline models are not yet clearly connected to lightweight on-chip implementations

There is room for a **teacher-student** story:

- offline teacher: richer model over raw RTL/activity traces,
- structured feature generation: pairwise or grouped interaction features,
- on-chip student: low-bit linear/tree/small-MLP model synthesized into RTL.

That could preserve the deployment story while addressing APOLLO's likely limitation on nonlinear interactions.

### Gap C: There is still no obvious open raw RTL/activity benchmark

**ArchPower is not enough** for this exact problem:

- it provides about `200` samples,
- it uses about `100` architecture-level features,
- it does **not** provide raw `VCD/SAIF`,
- it targets architecture-level CPU power modeling rather than signal-level RTL power modeling.

So a strong paper may need to contribute either:

- a new open benchmark from open RTL cores and signoff-like power flows, or
- a semi-open reproducible pipeline that generates the necessary waveform/power pairs.

### Gap D: Generalization is underexplored

An especially interesting question is not just "can one model fit one core?" but:

- can selected proxy sets transfer across workloads?
- can interaction features transfer across nearby configurations?
- can a model trained offline remain robust after design changes?

This can become a strong DAC/ICCAD angle if framed as **automation and portability**, not only predictive accuracy.

## Assessment of the Referenced Dataset

### ArchPower

`ArchPower` is useful background, but it is **not** a direct dataset for this project.

What it offers:

- open-source CPU power dataset,
- architecture-level hardware + event features,
- total and component-wise power labels,
- multiple CPU configurations and workloads,
- a public baseline environment for ML power modeling.

Why it is insufficient here:

- no raw RTL signal traces,
- no `VCD/SAIF`,
- abstraction level is too high,
- sample count is small for rich sequence or graph models,
- best fit is architecture-level modeling rather than signal-level deployable on-chip modeling.

Best use in this project:

- cite it as evidence that **open power-model datasets are scarce**,
- use it as a contrast point for why a **raw RTL/activity benchmark** would matter,
- possibly reuse its open CPU families as inspiration for benchmark construction.

## Recommended Scope for Phase 2

The most promising next-step scope is:

### Recommended scope

**Automatic interaction-aware proxy engineering for deployable RTL power models**

Core intuition:

- start from raw RTL/activity traces,
- discover a compact set of signals plus structured interaction features,
- explicitly penalize redundancy and multicollinearity,
- use a richer offline model only when it improves signal discovery,
- compile the final deployed model into a low-overhead on-chip implementation.

This is stronger than a plain deep model because it gives a clear DAC/ICCAD story:

- better automation than manual proxy engineering,
- better fidelity than purely linear proxies,
- better deployability than a large black-box DNN,
- clearer novelty than simply swapping regressors.

## Phase 2: Ranked Ideas

### Idea 1: Hardware-cost-aware interaction basis compiler — SELECTED AND NARROWED

- **Hypothesis**: A model that automatically constructs a small set of pairwise or grouped proxy interactions, but only when each interaction improves accuracy enough to justify its synthesized gate cost, will beat APOLLO-style raw-toggle linear models at the same monitor area budget.
- **Core mechanism**: Start from decorrelated signal families, generate candidate interaction terms (`AND`, `XOR`, gated windows, grouped counters), and select terms by **accuracy gain per hardware cost** under a hard area budget.
- **Minimum experiment**: Build an APOLLO-style sparse linear baseline and compare it against a cost-aware interaction basis on the same raw RTL/activity traces and the same synthesized monitor budget (`0.1%`, `0.2%`, `0.5%` area targets).
- **Expected outcome**: Better per-cycle power fidelity at fixed overhead, especially for bursty or cross-unit interactions that linear proxies miss.
- **Novelty**: `7.0/10 after narrowing` — broad claims were unsafe because Simmani already uses interaction / high-order terms and APOLLO / DEEP already establish deployable RTL power monitors. The defensible novelty is now **synthesis-in-the-loop compilation of structured interaction proxies under hard actual area budgets**.
- **Feasibility**: `7.5/10` — needs a benchmark generation flow, but the algorithmic path is concrete and hardware-friendly.
- **Risk**: `MEDIUM`
- **Contribution type**: `new method`
- **Pilot result**: `SKIPPED` — no raw RTL + VCD/SAIF benchmark packaged in this repo; needs benchmark construction first.
- **Reviewer's likely objection**: "Once you count the extra interaction logic, do the gains still survive at the same area budget?"
- **Why we should do this**: This is the cleanest extension beyond APOLLO without losing the DAC/ICCAD deployment story.

#### Deep novelty verdict for Idea 1

- **Proceed / caution / abandon:** `PROCEED WITH CAUTION`
- **Broad claim to avoid:** "first interaction-aware deployable RTL power model"
- **Safe thesis:** **Synthesis-in-the-loop selection and compilation of a small set of structured interaction proxies under hard post-synthesis monitor-area budgets for ultra-low-overhead on-chip power monitors.**
- **Key blocking prior work:** Simmani (interactions already exist), APOLLO + DEEP (deployable on-chip monitors already exist)
- **What now makes it paper-worthy:** real synthesized cost in the loop, hard budget enforcement, and actual accuracy/area Pareto improvement instead of generic nonlinear modeling

### Idea 2: Teacher-student deployable power model — RECOMMENDED

- **Hypothesis**: A richer offline teacher model can identify nonlinear dependencies and difficult interactions that direct sparse regression misses, while a distilled student still fits a tight on-chip logic budget.
- **Core mechanism**: Train a high-capacity offline teacher on raw traces or richer derived features, then distill its decisions into a sparse student constrained to synthesizable logic terms and quantized weights.
- **Minimum experiment**: Compare direct sparse training versus teacher-guided distillation under identical student architecture and overhead constraints.
- **Expected outcome**: The distilled student should outperform directly trained lightweight baselines under equal deployability constraints.
- **Novelty**: `8.2/10` — closest work: RTLDistil (timing distillation, not power), MasterRTL / RTL PPA predictors (power estimation without distillation).
- **Feasibility**: `7.0/10` — technically straightforward if benchmark data exists, but depends on finding a teacher that gives a meaningful student advantage.
- **Risk**: `MEDIUM`
- **Contribution type**: `new method`
- **Pilot result**: `SKIPPED` — needs benchmark data and a deployable student family definition first.
- **Reviewer's likely objection**: "If the student ends up looking like APOLLO anyway, what did distillation really buy you?"
- **Why we should do this**: It creates a strong bridge between expressive offline modeling and lightweight on-chip deployment, which current power-monitor papers do not cleanly solve.

### Idea 3: Structure-aware macro proxy synthesis — RECOMMENDED, BUT MORE CROWDED

- **Hypothesis**: Auto-derived macro-events from hierarchy, clock domains, and always blocks can give a more interpretable and routing-friendly proxy space than flat signal bags.
- **Core mechanism**: Parse RTL structure to create macro proxies such as issue-path active, LSU burst active, or vector-lane active, then learn sparse macro interactions.
- **Minimum experiment**: Compare flat sparse signal selection against structure-aware macro proxies on at least two cores with preserved hierarchy.
- **Expected outcome**: Better interpretability and possibly better portability, with similar or lower deployment complexity.
- **Novelty**: `6.6/10` — closer prior art exists than initially expected: AtomPower, PowerProbe, DynamicRTL, and VIRTUAL all crowd the structure-aware angle.
- **Feasibility**: `6.0/10`
- **Risk**: `MEDIUM-HIGH`
- **Contribution type**: `new method / diagnostic`
- **Pilot result**: `SKIPPED`
- **Reviewer's likely objection**: "Is this really new, or just another structured proxy extraction pipeline?"
- **Why we should still keep it**: It is hardware-native and explainable, but it now looks better as a backup or component than as the lead thesis.

### Backup Idea 4: Cross-core proxy template transfer

- **Hypothesis**: Interaction templates learned on one core can transfer to sibling cores if proxies are mapped by microarchitectural role instead of exact signal name.
- **Best use**: Strong evaluation axis or follow-up paper on automation/portability.
- **Main blocker**: Needs multiple related cores and a robust mapping story.

### Backup Idea 5: Phase-gated sparse experts

- **Hypothesis**: A tiny phase classifier plus a bank of sparse experts can model workload-phase-dependent power behavior better than a single global predictor.
- **Best use**: Ablation or backup direction if interactions are weak.
- **Main blocker**: Reviewers may see it as piecewise-linear engineering rather than a headline contribution.

## Eliminated Ideas

| Idea | Reason eliminated |
|---|---|
| Redundancy-suppressed family interactions | Feels too much like "better regularization" rather than a paper-level contribution. |
| Multi-timescale hybrid student | Useful engineering, but more likely a section or ablation than the main thesis. |
| Disagreement-driven trace labeling | Good for data efficiency, weak as the central idea for a runtime power-monitor paper. |
| Open benchmark only | Valuable artifact, but unlikely to be enough alone for DAC/ICCAD without a stronger method contribution. |
| Post-silicon trimmed deployable student | Interesting, but drifts into calibration and silicon adaptation instead of the raw RTL/activity modeling problem. |

## Quick Novelty Screen on the Top Ideas

- **Idea 1 looks the cleanest**: quick searches did not surface prior work that explicitly constructs **interaction proxies under a hard synthesized monitor-area budget**.
- **Idea 2 looks underexplored rather than empty**: there is adjacent RTL-level distillation for **timing** (RTLDistil), but not an obvious prior power-model paper distilling from rich offline teacher to deployable on-chip student.
- **Idea 3 got weaker during screening**: newer adjacent works such as **AtomPower**, **PowerProbe**, **DynamicRTL**, and **VIRTUAL** make the structure-aware macro-proxy direction look more crowded.

## Selected Idea Outcome

- **Idea carried into deep validation:** Idea 1 only
- **Deep novelty outcome:** viable only in a narrowed form
- **Critical review outcome:** strongest reviewer attacks are now explicit and directly converted into must-run experiments
- **Current thesis:** the paper is no longer "better interaction modeling" in general; it is a **budget-faithful compiler paper** for interaction proxies that survive real synthesis

## Refined Proposal and Experiment Handoff

- **Review summary:** `refine-logs/REVIEW_SUMMARY.md`
- **Proposal:** `refine-logs/FINAL_PROPOSAL.md`
- **Experiment plan:** `refine-logs/EXPERIMENT_PLAN.md`
- **Tracker:** `refine-logs/EXPERIMENT_TRACKER.md`

### Proposal summary

- **Problem anchor:** build an on-chip processor power monitor from raw RTL + VCD/SAIF that maximizes prediction quality under a hard synthesized monitor-area budget
- **Dominant contribution:** synthesis-in-the-loop compilation of a tiny set of structured interaction proxies under hard actual area budgets
- **Optional supporting contribution:** show that estimated feature cost misranks candidates and can erase supposed gains unless synthesis participates in selection
- **Deliberate non-claims:** not first deployable monitor, not first interaction model, not a large neural predictor

### Must-run experiment blocks

1. **Matched-budget Pareto on two RTLs** — raw-only vs generic polynomial vs structured estimated-cost vs structured synth-in-loop
2. **Estimated cost vs actual synthesized cost** — prove synthesis-in-the-loop is necessary
3. **Leave-workload-out transfer** — show the selected interaction families are not just overfitting

### First three runs to launch

1. `R001` — trace / label sanity
2. `R002` — primitive cost library synthesis
3. `R003` — fixed-point readout quantization sanity

## Suggested Execution Order

1. **Lead thesis**: Idea 1 — hardware-cost-aware interaction basis compiler
2. **Preferred training recipe**: Idea 2 — teacher-student distillation
3. **Optional component or backup**: Idea 3 — structure-aware macro proxy synthesis
4. **Later evaluation axis**: cross-core transfer

### Alternative scopes worth considering

1. **Teacher-student distillation**
   - Teacher: attention / MLP / tree ensemble over richer feature space
   - Student: synthesizable linear / tree / tiny-MLP predictor
   - Best if you want both accuracy and an on-chip story.

2. **Benchmark-first paper**
   - Build and release the first open raw RTL + VCD/SAIF + power dataset for CPU power modeling
   - Strong community value, but may dilute the modeling contribution unless tightly scoped.

3. **Dynamic feature relevance**
   - Make feature selection time- or context-dependent rather than static
   - Good fit if you want to emphasize workload phases, bursts, or per-stage processor behavior.

## Risks to Watch Early

- **Data generation cost**: signoff-quality labels are expensive.
- **Model-to-hardware gap**: a powerful offline model may not map cleanly to on-chip logic.
- **Overfitting to one core**: easy to get a nice result on a single design and weak generalization.
- **Novelty trap**: "replace linear regression with a DNN" is probably not enough.
- **Evaluation trap**: without area/latency/energy overhead analysis, the work will feel incomplete for DAC/ICCAD.

## Sources

- Local paper: `MICRO21_APOLLO.pdf`
- PowerProbe: `https://past.date-conference.com/proceedings-archive/2018/pdf/0435.pdf`
- LIPPo / UT Austin report: `https://users.ece.utexas.edu/~gerstl/publications/UT-CERC-18-01.LIPPo.pdf`
- APOLLO paper page: `https://dl.acm.org/doi/10.1145/3466752.3480064`
- DEEP: `https://zhiyaoxie.com/files/ICCAD22_DEEP.pdf`
- Sagi TCAD 2020: `https://doi.org/10.1109/TCAD.2020.3013062`
- Simmani paper: `https://simmani.github.io/assets/paper.pdf`
- UT Austin report: `https://users.ece.utexas.edu/~gerstl/publications/UT-CERC-18-01.LIPPo.pdf`
- ML-based microarchitecture-level CPU power modeling: `http://slam.ece.utexas.edu/pubs/tc22.LACPo.pdf`
- MasterRTL: `https://arxiv.org/abs/2311.08441`
- RTLDistil: `https://www.cse.cuhk.edu.hk/~byu/papers/C276-ICML2025-Net2RTL.pdf`
- DynamicRTL: `https://arxiv.org/abs/2511.09593`
- VIRTUAL: `https://www.cse.cuhk.edu.hk/~byu/papers/C294-ICCAD2025-RTLPower.pdf`
- AtomPower: `https://www.jstage.jst.go.jp/article/elex/advpub/0/advpub_23.20260004/_article`
- PowerProbe: `https://www.researchgate.net/publication/324710761_PowerProbe_Run-time_power_modeling_through_automatic_RTL_instrumentation`
- ArchPower repo: `https://github.com/hkust-zhiyao/ArchPower`
- ArchPower paper page: `https://arxiv.org/abs/2512.06854`

## Next Step

The proposal is now concrete enough for the next checkpoint:

1. proceed to implementation and dataset / flow setup for the experiment plan,
2. or adjust the proposal before writing code and building the benchmark pipeline.
