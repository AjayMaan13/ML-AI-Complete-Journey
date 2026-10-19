# 03: Regularization & Optimizers: Simple Notes

Covers only the topics I got confused about, merged with CS231n Lecture 3 (Regularization and Optimization).

**Map of everything:**

```
Regularization (L1, L2, AdamW's weight decay)
Gradient Descent -> Batch / SGD / Mini-batch -> Noise
   -> + Momentum -> RMSProp -> Adam -> AdamW
Learning rate decay
```

---

## 1. Regularization

**What it is:** a penalty added to the loss so the model can't fit the training data *too* perfectly (so it doesn't memorize noise).

```
Loss = data loss + λ · penalty(weights)
```

- **Data loss:** predictions should match training data.
- **Regularization:** prevent the model from doing too well on training data.
- **λ:** regularization strength (a hyperparameter). λ = 0 means no regularization. Huge λ means weights are crushed to ~0 and the model underfits.

**Why (from lecture):**
- Express preferences over weights
- Make the model simple so it works on test data (Occam's Razor: simplest explanation wins)
- Improve optimization by adding curvature

**Other regularizers:** Dropout, Batch Norm, Stochastic depth, early stopping, more data.

**Tune λ on the dev set, never the test set.**

---

## 2. L1 vs L2

| | **L2** | **L1** |
|---|---|---|
| Penalty | λ · Σ w² | λ · Σ \|w\| |
| Gradient | λ·w (proportional to weight) | λ·sign(w) (constant) |
| Effect | Shrinks all weights smoothly, never exactly 0 | Pushes many weights to **exactly 0** |
| Prefers | **Spread-out** weights | **Sparse** weights (few nonzero) |
| Also called | Ridge, weight decay | Lasso |
| Elastic Net | L1 + L2 combined | |

### Example: why L2 spreads and L1 doesn't

Two identical features, the data needs w₁ + w₂ = 1:

| (w₁, w₂) | L1 cost | L2 cost |
|---|---|---|
| (1, 0) | 1 | 1 |
| (0.5, 0.5) | 1 | **0.5** |

- **L2** picks (0.5, 0.5). Squaring makes one big weight very expensive, so it shares the load.
- **L1** sees a tie (every split costs 1). If one feature is even slightly more useful, it puts *all* weight there and zeros the other.

**Lecture's question:** "Which of w1 or w2 will L2 prefer?" Answer: the spread-out one. L1 prefers the sparse one.

**Picture:** L2 constraint is a circle (contact at a generic point). L1 constraint is a diamond with corners on the axes (contact at a corner means a weight is exactly 0).

**When to use:** L1 when only a few features matter. L2 when many features each matter a little, or features are correlated.

---

## 3. Gradient Descent (GD)

**What it is:** an algorithm that finds the minimum of the loss by repeatedly stepping downhill. It uses only the local slope (first-order, local, iterative).

```
w = w − η · gradient
```

- **gradient:** slope of the loss for each weight. Points uphill.
- **η (learning rate):** step size.
- Subtract the gradient to go **downhill**.

### Example: minimize L = w², gradient = 2w, η = 0.1, start w = 4

| step | w | gradient | new w |
|---|---|---|---|
| 0 | 4.00 | 8.0 | 3.20 |
| 1 | 3.20 | 6.4 | 2.56 |
| 2 | 2.56 | 5.12 | 2.05 |

The steps shrink on their own near the bottom because the slope flattens.

**Learning rate:** too small means slow. Too large means overshoot or diverge.

**Why it works:** near your point the loss looks like a straight line, and the best small step against a line is straight opposite the gradient.

**From the lecture:**
- *Numerical gradient:* nudge each weight by h = 0.0001 and measure the change, e.g. (1.25322 − 1.25347)/0.0001 = **−2.5**. Approximate, slow, easy to write.
- *Analytic gradient (calculus):* exact, fast, error-prone.
- **Gradient check:** always use analytic, but verify it against numerical.

---

## 4. Batch vs SGD vs Mini-batch

Same update rule. The only difference is **how much data is used to compute the gradient per step.**

| | Batch GD | SGD | Mini-batch GD |
|---|---|---|---|
| Data per step | all N | 1 example | B examples (32/64/128) |
| Updates per epoch | 1 | N | N / B |
| Gradient | exact | very noisy | a bit noisy |
| Speed / GPU use | slow | poor GPU use | best |

An **epoch** is one full pass over the data. In practice "SGD" almost always means mini-batch.

**Lecture:** the full sum is expensive when N is large, so approximate it with a minibatch.

### How SGD makes N updates in one epoch

```python
for x, y in shuffled(data):      # N loops = 1 epoch
    g = grad(x, y, w)            # gradient from THIS example
    w = w - lr * g               # update right away
```

Each example's gradient is computed at the **current** w, which already moved after the previous example. Batch GD computes all gradients at the same w, then updates once.

### Example: N = 182,000, B = 32

Updates per epoch = 182,000 / 32 = about **5,690**.

---

## 5. Noise in GD

**Noise** = the gap between the gradient from a sample and the true full-data gradient.

```
sampled gradient = true gradient + noise
```

- **Unbiased:** averaged over many batches it equals the true gradient, so SGD still works.
- **Batch GD has zero noise.**
- **Doesn't vanish at the minimum:** the true gradient is 0 there, but individual examples' gradients aren't, so weights keep jittering.

### Example

Per-example gradients: −4, −16, −4. True average = **−8**. SGD gets −4 or −16 each step. That spread is the noise.

### Why SGD has the *most* noise

With B = 1 nothing is averaged out. Noise std ∝ **1/√B**:
- B = 1: full noise
- B = 32: noise / 5.7
- B = N: zero

To halve the noise you need **4× the batch size** (diminishing returns).

**Good side of noise:** escapes saddle points and shallow bad minima, lands in flatter minima (generalize better), acts like mild regularization. This is why makemore's loss curve looks fuzzy: random batches of 32. It's not a bug.

**Lecture: 3 problems with SGD**
1. **Poor conditioning:** loss changes fast in one direction, slow in another. Result: jitter in the steep direction, slow progress in the shallow one.
2. **Local minima / saddle points:** gradient is 0, so it gets stuck. Saddle points are much more common in high dimensions.
3. **Gradient noise** from minibatches.

Momentum helps with all three.

---

## 6. SGD + Momentum

**Idea:** keep a running "velocity" of past gradients, so you keep moving in the general direction you were already going (a ball rolling downhill).

```
v = β · v + g
w = w − η · v
```

| Symbol | Meaning |
|---|---|
| g | current gradient |
| v | velocity (memory of past gradients), starts at 0 |
| β | how much old velocity you keep (0.9 or 0.99). Lecture calls it **rho** |
| η | learning rate |

**In words:** keep 90% of the old direction, add the new slope, step along that.

### Example: gradients +1, +1, +1, −1 with β = 0.9

| step | g | v = 0.9·v + g |
|---|---|---|
| 1 | 1 | 1.0 |
| 2 | 1 | 1.9 |
| 3 | 1 | 2.71 |
| 4 | −1 | 1.44 |

At step 4 the gradient flips but velocity is still positive, so it keeps moving the old way (just slower).

**Sanity checks:**
- β = 0 gives plain SGD.
- Constant gradient forever: v approaches g/(1−β) = 10g. Steps on a steady slope become 10× bigger.

**What it fixes:**
- Consistent directions add up (faster).
- Zigzag directions cancel (smoother).
- Noise averages out.

**Lecture notes:** there's an equivalent alternative formulation (same sequence of weights). **Nesterov momentum** (appendix) computes the gradient at the "look-ahead" point where velocity would take you, then mixes it in.

---

## 7. RMSProp

**Idea:** give every weight its own step size. Divide each weight's gradient by its recent typical gradient size. Lecture: "per-parameter / adaptive learning rates."

```
s = β · s + (1 − β) · g²
w = w − η · g / (√s + ε)
```

| Symbol | Meaning |
|---|---|
| s | running average of g² per weight (magnitude, not direction) |
| β | decay (0.9 or 0.99) |
| ε | tiny number (1e-8) so we never divide by 0 |

### Example (η = 0.1, √s has settled at typical gradient size)

| | gradient | √s | SGD step | RMSProp step |
|---|---|---|---|---|
| Steep weight | 10 | 10 | 1.0 | **0.1** |
| Flat weight | 0.1 | 0.1 | 0.01 | **0.1** |

SGD moves the steep weight 100× more. RMSProp equalizes them.

**Lecture:** "Progress along steep directions is damped; progress along flat directions is accelerated."

**Why a decaying average:** the older **AdaGrad** summed *all* past g², so the step size decays to zero over time. RMSProp = "leaky AdaGrad," so it can recover.

**Momentum vs RMSProp:**
- Momentum averages `g` (direction) and fixes zigzag/noise.
- RMSProp averages `g²` (magnitude) and fixes the one-size-fits-all learning rate.

**They're independent:** RMSProp is not always used with momentum (PyTorch's `RMSprop` has `momentum=0` by default). Using both gives Adam.

---

## 8. Adam

**Adam = Momentum + RMSProp + bias correction.** Lecture: "Sort of like RMSProp with momentum."

```
m = β₁ · m + (1 − β₁) · g        # momentum (direction)
s = β₂ · s + (1 − β₂) · g²       # RMSProp (magnitude)

m̂ = m / (1 − β₁ᵗ)                # bias correction
ŝ = s / (1 − β₂ᵗ)

w = w − η · m̂ / (√ŝ + ε)
```

| Symbol | Typical value |
|---|---|
| β₁ | 0.9 |
| β₂ | 0.999 |
| η | 1e-3 or 5e-4 (3e-4 for transformers) |
| ε | 1e-8 |
| t | step number (1, 2, 3 ...) |

**In words:** step = (smoothed direction) ÷ (typical size). Each weight moves about η per step regardless of its gradient scale.

**Lecture advice:** Adam with β₁ = 0.9, β₂ = 0.999, lr = 1e-3 or 5e-4 is a great starting point for many models.

**Why it's good:** per-weight step sizes, smooth direction, needs little tuning, fast early training.

**Caveats:**
- Stores 2 extra copies of the model (m and s), so it uses more memory.
- Well-tuned SGD + Momentum can match or beat it on final accuracy (esp. CNNs).

### How Adam removes the bias

"Bias" here means **initialization bias**, not the `b` in `wx + b`. m and s start at 0, so early values are too small.

If the gradient is constant (g = 1), then m = (1 − 0.9ᵗ)·g:

| t | m (no fix) | divide by (1 − 0.9ᵗ) | m̂ |
|---|---|---|---|
| 1 | 0.1 | ÷ 0.1 | **1.0** |
| 2 | 0.19 | ÷ 0.19 | **1.0** |
| 3 | 0.271 | ÷ 0.271 | **1.0** |

The fix is large at the start and fades to ~1 later.

### Lecture's question: "What happens at the first timestep?"

Without correction at t = 1: m = 0.1g, s = 0.001g², so the step is 0.1g / (0.0316·|g|) ≈ **3.16·η·sign(g)**, an oversized first step.

With correction: m̂ = g and ŝ = g², so the step is exactly **η·sign(g)**.

---

## 9. AdamW (Adam + regularization)

**Adam alone has no regularization.** You add it. There are two ways:

| | How | Problem |
|---|---|---|
| **Adam + L2** | add λw into the gradient, *before* the moments | gets divided by √s, so decay strength is uneven per weight |
| **AdamW** | subtract λw *after* the moments | every weight decays by the same proportion |

```
Adam + L2 :  g = grad + λ·w              (L2 inside the gradient)
AdamW     :  w = w − η·m̂/(√ŝ+ε) − η·λ·w (decay applied separately)
```

**Lecture:** "Standard Adam computes L2 here (used during moment calculations). AdamW adds the term after the moments." The fast.ai plot shows AdamW gets better ImageNet accuracy.

**Lecture's Q:** "How does regularization interact with the optimizer?" A: **It depends!**

```python
opt = torch.optim.AdamW(model.parameters(), lr=3e-4, weight_decay=0.1)
```

Usually skip weight decay on biases, norm layers, and embeddings (nanoGPT does this).

**Lecture "in practice":**
- Adam(W) is a good default; often works ok even with a constant learning rate.
- SGD + Momentum can outperform Adam but needs more tuning of LR and schedule.
- If you can afford full-batch updates, look beyond first-order methods.

---

## 10. Learning Rate Decay

**Why:** SGD, Momentum, RMSProp, Adam and AdamW all have a learning rate. A high LR is good early (fast progress). A low LR is good late (settle into the minimum, calm the noise). So the LR starts big and shrinks over time.

Notation: α₀ = initial LR, αₜ = LR at epoch t, T = total epochs.

| Schedule | Formula | Example (α₀ = 1, T = 100) |
|---|---|---|
| **Step** | multiply LR by 0.1 at fixed epochs (ResNets: epochs 30, 60, 90) | 1 → 0.1 at epoch 30 → 0.01 at 60 → 0.001 at 90 |
| **Cosine** | αₜ = ½ · α₀ · (1 + cos(tπ/T)) | t = 0: 1.0, t = 50: **0.5**, t = 100: **0** |
| **Linear** | αₜ = α₀ · (1 − t/T) | t = 50: **0.5**, t = 100: **0** |
| **Inverse sqrt** | αₜ = α₀ / √t | t = 1: 1.0, t = 25: 0.2, t = 100: **0.1** |

**About your screenshot:** that curve (starts at 1.0, drops fast, flattens near 0.1 at epoch 100) is **inverse sqrt**. Cosine and linear would hit 0 at the end.

**Who uses what (from the slides):** Cosine in GPT / SGDR / SlowFast, Linear in BERT, Inverse sqrt in the original Transformer ("Attention Is All You Need").

### Linear warmup

- High initial LRs can make the loss **explode**.
- Fix: increase the LR linearly from **0** over the first ~5,000 iterations, then decay.
- **Rule of thumb:** if you increase batch size by N, scale the initial LR by N too.

---

## 11. Quick cheat sheet

| Thing | One line |
|---|---|
| Regularization | Loss = data loss + λ·penalty; stops overfitting |
| L2 | shrinks all weights, prefers **spread out** |
| L1 | pushes weights to exactly 0, prefers **sparse** |
| GD | w = w − η·g, walk downhill |
| Batch / SGD / Mini-batch | all data / 1 example / small chunk per gradient |
| Noise | sampled gradient − true gradient; std ∝ 1/√B |
| Momentum | v = βv + g; smooths direction, builds speed |
| RMSProp | divide g by √(avg g²); per-weight step size |
| Adam | momentum + RMSProp + bias correction |
| Bias correction | divides by (1 − βᵗ) to fix the zero start |
| AdamW | Adam + weight decay applied outside the adaptive scaling |
| LR decay | start big, shrink later (step / cosine / linear / inv sqrt) |
| Warmup | ramp LR from 0 first to avoid exploding loss |

**Source:** Stanford CS231n, Lecture 3 (Spring 2026): https://cs231n.stanford.edu/slides/2026/lecture_3.pdf