# OncoSentry: Next-Generation Engineering & Scientific Roadmap

This roadmap outlines the multi-phase engineering plan to upgrade and surpass every benchmark parameter established by the TriboGuard / TriboVision baseline.

---

## Benchmark Scorecard: Baseline vs. Target

| Subsystem / Metric | Baseline Floor (TriboGuard / TriboVision) | Upgraded Target (OncoSentry) | Target Phase |
|---|---|---|---|
| **Semantic Macro Dice** | `0.9515` (Macro) / `0.9563` (Micro) | **$\ge 0.975$** Native Multi-Scale | Phase 1 |
| **Semantic Macro IoU** | `0.9081` (Macro) | **$\ge 0.940$** | Phase 1 |
| **Instance Matching (0.50:0.95)** | `0.0455` (Binary) / `0.2078` (3-Class) | **$\ge 0.550$** (surpasses Cellpose `0.3893`) | Phase 1 |
| **Damage Blindness Recovery** | Detection collapses from 85% to **$< 5\%$** | **$\ge 80\%$** detection on dying/apoptotic cells | Phase 2 |
| **Identifiability Width ($g_{width}$)** | Expands to `[0.0, 1 - R]` ($100\%$ unidentifiable) | Constrained to **$\le 15\%$** interval width | Phase 2 |
| **Temporal Cell Tracking** | **Absent (0%)** — Static population averages only | **Full single-cell lineage tracking (MOTA $\ge 0.85$)** | Phase 3 |
| **Turnover Ratio Uncertainty ($N=3$)** | Factor of **$146\times$** uncertainty | **$< 5\times$** at $N=3$ wells | Phase 3 |
| **Sample Size to Factor-of-2 Certainty** | **$66$ wells** under bulk destructive MTS | **$\le 8$ wells** under automated live tracking | Phase 3 & 4 |
| **Tissue Mechanics Recovery ($\rho$)** | A172: `0.767` \| SkBr3: `0.817` \| MCF7: `0.388` | **$\rho \ge 0.850$ on ALL lines** | Phase 1 & 4 |
| **Viability Prediction Skill ($R^2$)** | Neural cross-domain transfer: $R^2 = -1.67$ | **$R^2 \ge 0.85$** leave-one-day-out prediction | Phase 4 |

---

## Phase 1: High-Resolution Vector-Flow Instance Segmentation

### Goal
Break through the binary representation ceiling (`0.168`) and three-class watershed ceiling (`0.208`) to achieve an instance matching score of **$\ge 0.550$**, surpassing zero-shot Cellpose (`0.3893`).

### Key Innovations
1. **Multi-Scale Distance & Vector-Flow Representation**:
   - Replace discrete 3-class boundary maps with continuous 2D horizontal/vertical distance-gradient flow fields ($\mathbf{v} = (\partial_x D, \partial_y D)$).
   - Invertible flow tracking moves pixels directly toward cell centroids, avoiding oversegmentation in dense clusters.
2. **Sub-Pixel Polygon & Anti-Aliased Perimeter Extractor**:
   - Formulate continuous marching squares on distance maps to eliminate the $24\%$ perimeter overestimation artifact (crack perimeter bias), restoring true circularity metrics.
3. **Cross-Lineage Diversity Pre-training**:
   - Train on balanced multi-lineage datasets (A172, MCF7, SkBr3, SHSY5Y, BV2, Huh7) to cut cross-cell-line transfer degradation from $49\%$ to $< 15\%$.

### Deliverables
- `src/oncosentry/vision/flow_model.py`: Multi-head flow and boundary neural network.
- `src/oncosentry/vision/flow_decoder.py`: Gradient flow tracking integrator.
- `src/oncosentry/vision/subpixel_contour.py`: Anti-aliased perimeter & area calculator.
- `tests/test_flow_instance.py`: Unit and benchmark integration tests.

---

## Phase 2: Resolving the "Damage Blindness" Trap

### Goal
Eliminate the segmenter's blind spot on damaged and dying cells, raising detection rates from $< 5\%$ to **$\ge 80\%$** under severe cellular damage.

### Key Innovations
1. **Distressed Cell Phenotype Detector**:
   - Train an auxiliary classification and segmentation head specifically sensitized to apoptotic and necrotic morphology:
     - Phase halo densification and cell retraction.
     - Nuclear pyknosis, chromatin condensation, and karyorrhexis.
     - Cytoplasmic vacuolation and membrane blebbing.
2. **Identifiability Shrinkage**:
   - Dynamically compute blindness $b = 1 - d_{det}/h$.
   - Collapse the unidentifiable gap $[g_{min}, g_{max}]$ so that automated counts reliably separate cell clearance from invisible dying cells.
3. **Pre-Registration Re-Evaluation**:
   - Run against the pre-registered benchmark (`preregistration/damage_blindness_v1.json`) across all four cell lines (A172, MCF7, SHSY5Y, SkBr3) to verify that damaged cells are detected with statistical parity.

### Deliverables
- `src/oncosentry/vision/damage_detector.py`: Dying/distressed morphology classifier.
- `src/oncosentry/audit/blindness_shrinkage.py`: Dynamic identifiability interval solver.
- `scripts/evaluate_damage_upgraded.py`: Automated benchmarking script.

---

## Phase 3: Spatio-Temporal Single-Cell Lineage Tracking

### Goal
Move beyond static image slices to automated continuous lineage tracking, directly identifying division events ($b$) and death events ($d$) to compress required well sample sizes from **66 wells to $\le 8$ wells**.

### Key Innovations
1. **Lineage Graph Construction**:
   - Spatio-temporal matching across time-lapse frames using a modified Kalman-filter and Hungarian / ByteTrack graph matching algorithm.
   - Track persistence, morphology trajectory, and cell lineage trees.
2. **Direct Event Counting**:
   - Automatically flag **mitotic cleavage** events (one mother cell dividing into two daughter cells) to count $b$ directly.
   - Automatically flag **lysis/apoptosis/detachment** events to count $d$ directly.
3. **Decoupling Without Variance**:
   - By counting events directly on individual wells, the mathematical degeneracy of bulk averages is bypassed entirely.

### Deliverables
- `src/oncosentry/kinetics/tracker.py`: Graph-based spatio-temporal tracker.
- `src/oncosentry/kinetics/event_detector.py`: Mitosis and apoptosis event identifier.
- `src/oncosentry/kinetics/lineage_tree.py`: Cell family lineage tree visualizer.
- `tests/test_tracking.py`: Synthetic and real time-lapse validation suite.

---

## Phase 4: Hierarchical Bayesian MCMC Kinetics & Active Design

### Goal
Provide rigorous uncertainty quantification pooling across wells, doses, and timepoints, achieving $R^2 \ge 0.85$ cross-validated viability prediction.

### Key Innovations
1. **Hierarchical Bayesian Inference**:
   - Model plate, well, and field random effects in a unified MCMC framework with physical prior constraints ($b \ge 0, d \ge 0, q \ge 3.54$).
   - Return true posterior distributions rather than loose chi-squared approximations.
2. **Active Experimental Design Planner**:
   - Compute Expected Information Gain (EIG).
   - Inform the investigator in real time whether the next unit of research budget is best spent adding a replicate well, a timepoint, or an additional concentration.
3. **Leave-One-Day-Out Viability Predictor**:
   - Calibrate morphology-derived kinetics against orthogonal viability ground truth using cross-day permutation tests.

### Deliverables
- `src/oncosentry/kinetics/bayes_solver.py`: Hierarchical MCMC sampler.
- `src/oncosentry/audit/active_planner.py`: Information-theoretic design optimizer.
- `tests/test_bayes_kinetics.py`: Convergence and calibration test suite.

---

## Phase 5: Next-Generation Reactive Web Platform

### Goal
Deliver an ultra-premium, interactive browser dashboard for researchers, biophysicists, and oncologists.

### Key Innovations
1. **Interactive Simulation & Assay Sandbox**:
   - Live browser-based Monte Carlo simulation comparing destructive MTS bulk assays against non-destructive live imaging.
   - Real-time birth-death parameter sliders with instantaneous sample size and cost calculations.
2. **Client-Side Computer Vision Inference**:
   - WebGPU / ONNX Runtime Web integration allowing users to drag and drop phase-contrast micrographs and receive instant instance segmentation, shape index $q$, and damage assessments.
3. **State-of-the-Art Aesthetic Design**:
   - Modern dark mode with tailored HSL color palettes, glassmorphism, responsive data grids, and smooth interactive charts via Recharts / Canvas.

### Deliverables
- `web/app/`: Next-generation interactive application pages.
- `web/components/simulator/`: Interactive birth-death Monte Carlo simulator.
- `web/components/vision/`: Live micrograph dropzone & inspection canvas.

---

## Implementation Schedule

```
Month 1: Phase 1 (Vector-Flow Segmentation & Sub-Pixel Geometry)
Month 2: Phase 2 (Damage Blindness Resolution & Distress Head)
Month 3: Phase 3 (Single-Cell Spatio-Temporal Lineage Tracking)
Month 4: Phase 4 (Hierarchical Bayesian Solver & Active Planning)
Month 5: Phase 5 (Reactive Web Application & Live Simulation Platform)
```
