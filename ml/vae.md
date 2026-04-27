# Variational Autoencoders — Everything You Need to Know

> A ground-up, intuition-first guide. No hand-waving, no skipped steps.

---

## 1. The Problem: We Want to Learn a Data Distribution

You have a dataset of images — faces, digits, cats, whatever. Each image is a point **x** in some astronomically high-dimensional space (e.g., 64×64×3 = 12,288 dimensions). All your images together form a **data distribution** called **p(x)**.

Your dream: learn p(x) so well that you can **sample from it** and get new, realistic images that look like they belong to your dataset but aren't copies.

The trouble? p(x) lives in a ridiculously high-dimensional space. You can't model it directly.

---

## 2. The Latent Space Idea

Here's the key insight: real images don't actually use all 12,288 dimensions of freedom. A face image is mostly determined by a few factors — skin tone, hair style, expression, head angle. Those factors live in a much smaller **latent space** of dimension d (say, d = 128).

So the idea is:

1. Define a simple distribution **p(z)** over this latent space (we'll pick one soon).
2. Learn a mapping **p(x|z)** — "given a latent code z, produce an image x."
3. To generate a new image: sample z from p(z), then push it through p(x|z).

Mathematically, the data distribution can be written as:

```
p(x) = ∫ p(x|z) · p(z) dz
```

This integral says: "sum up the contributions of every possible latent code z, weighted by how likely that code is."

**The problem:** this integral is **intractable**. You're integrating over every possible z in a continuous, high-dimensional space. You can't compute it.

---

## 3. Why You Need an Encoder (Not Just a Decoder)

Suppose you just had a decoder — a neural network that takes z and produces x. If you sample z randomly from some distribution, the decoder has no guidance. It doesn't know which regions of latent space correspond to what. The output will be **garbage**.

You need to **organise** the latent space. That means you need to know, for a given image x, **which z produced it** — i.e., you need p(z|x), the **posterior**.

This is where the **encoder** comes in. It takes an image x and maps it to a region in latent space. The encoder and decoder must be **trained together**:

- **Encoder**: x → z (finds the latent code for this image)
- **Decoder**: z → x' (reconstructs the image from the latent code)

By training them jointly, the latent space gets organised — similar images get mapped to nearby z values, and the decoder learns to map those z values back to realistic images.

---

## 4. The Intractability Problem and the Approximation

To train the encoder, you'd ideally compute the true posterior **p(z|x)** using Bayes' theorem:

```
p(z|x) = p(x|z) · p(z) / p(x)
```

But **p(x) is intractable** (that integral from Section 2). So p(z|x) is also intractable. You're stuck.

**The VAE solution:** Don't compute p(z|x). Instead, **approximate** it with a simpler distribution:

```
q_φ(z|x) = N(μ(x), σ²(x))
```

This is a Gaussian whose mean μ and variance σ² are **output by the encoder network** (parameterised by φ). For each input image x, the encoder outputs a mean vector and a variance vector, defining a Gaussian "cloud" in latent space.

We also **assume a prior**:

```
p(z) = N(0, I)    (standard normal)
```

This is a design choice. It says: "before seeing any data, we expect latent codes to be centred at zero with unit variance." This keeps the latent space tidy.

---

## 5. Likelihood vs Log-Likelihood

Before diving into the loss function, a quick but important distinction.

### Likelihood

The **likelihood** is the probability of your data given the model parameters θ:

```
L(θ) = p(x₁|θ) · p(x₂|θ) · … · p(xₙ|θ)
```

For n independent data points, it's a **product** of probabilities.

### Log-Likelihood

Take the log:

```
ℓ(θ) = log p(x₁|θ) + log p(x₂|θ) + … + log p(xₙ|θ)
```

The product becomes a **sum**.

### Why always use log?

| Reason | Explanation |
|--------|-------------|
| **Numerical stability** | Individual probabilities like 0.0001 multiplied thousands of times → underflows to 0 in floating point. Logs keep numbers in a sane range. |
| **Nicer gradients** | Log cancels the exp in Gaussians, giving you clean quadratic forms. |
| **Same optimum** | Log is monotonic, so argmax of log p(x) = argmax of p(x). No information lost. |

### Connection to VAEs

In the ELBO (coming next), the reconstruction term is **E[log p(x|z)]**:
- If the decoder models a Gaussian → log p(x|z) becomes **MSE** (mean squared error)
- If it models a Bernoulli → becomes **binary cross-entropy**

The log is what makes these standard losses fall out naturally.

---

## 6. The ELBO — The Actual Training Objective

Since we can't maximise **log p(x)** directly (intractable), we maximise a **lower bound** on it — the **Evidence Lower BOund (ELBO)**.

### The Derivation (Intuition Version)

Start with what we want: log p(x).

We can show (via a few lines of algebra) that:

```
log p(x) = ELBO + KL(q(z|x) ‖ p(z|x))
```

where:

```
ELBO = E_{z~q(z|x)}[log p(x|z)] − KL(q(z|x) ‖ p(z))
```

Since **KL divergence is always ≥ 0**, we know:

```
log p(x) ≥ ELBO
```

So by **maximising the ELBO**, we're pushing log p(x) up (what we want) while simultaneously pushing KL(q ‖ p(z|x)) down — meaning our approximation q gets closer to the true posterior, **without ever computing the true posterior**.

### The Two Terms of the ELBO

The ELBO has two terms that pull in opposite directions:

```
ELBO = E[log p(x|z)]  −  KL(q(z|x) ‖ p(z))
         ↑                    ↑
    Reconstruction         Regularisation
```

| Term | What it does | In plain English |
|------|-------------|------------------|
| **Reconstruction** E[log p(x|z)] | Measures how well the decoder reconstructs x from z | "Can you rebuild the image from the latent code?" |
| **Regularisation** KL(q(z\|x) ‖ p(z)) | Measures how far the encoder's distribution is from N(0,1) | "Is the latent space staying organised and smooth?" |

**The tension:** The reconstruction term wants to encode everything — make each z as informative as possible. The KL term wants the latent space to look like N(0,1) — smooth, continuous, well-behaved. This push-pull is what makes VAEs produce smooth latent spaces where you can interpolate between images.

---

## 7. The Closed-Form KL Between Two Gaussians

The regularisation term KL(q(z|x) ‖ p(z)) compares:

- q(z|x) = N(μ, σ²) — the encoder's output
- p(z) = N(0, 1) — the prior

Since both are Gaussians, the KL has a **closed-form solution** (no sampling needed):

```
D_KL = ½ Σᵢ (μᵢ² + σᵢ² − log(σᵢ²) − 1)
```

where the sum is over each dimension i of the latent space.

### What each term does intuitively

| Term | What it penalises | Why |
|------|-------------------|-----|
| **μᵢ²** | Mean drifting away from 0 | Keeps latent codes centred |
| **σᵢ²** | Variance being too large | Prevents latent space from exploding |
| **−log(σᵢ²)** | Variance collapsing to 0 | If σ→0, this term →+∞, preventing point estimates |
| **−1** | Nothing (constant offset) | Makes KL = 0 when μ=0, σ=1 (perfect match) |

> [!TIP]
> You can verify: plug in μ=0, σ=1 and you get ½(0 + 1 − 0 − 1) = 0. Perfect match → zero divergence.

---

## 8. The Reparameterisation Trick

During training, the encoder outputs μ and σ, and you need to **sample** z from N(μ, σ²). But sampling is a random operation — you can't backpropagate through randomness.

**The trick:** rewrite the sampling as:

```
z = μ + σ · ε,    where ε ~ N(0, 1)
```

Now:
- ε is just random noise (not a learnable parameter)
- z is a **deterministic function** of μ, σ, and ε
- Gradients flow cleanly through μ and σ back to the encoder

This is what makes VAE training possible with standard backpropagation.

---

## 9. Forward vs Reverse KL Divergence — The Deep Dive

This is the section that trips everyone up. Let's build it from absolute zero.

### What is KL Divergence?

It answers one question: **"How different are these two distributions?"**

- KL = 0 → they're identical
- KL > 0 → they differ (bigger = more different)

### How does it measure "different"?

It looks at every point z and computes the **log ratio** of probabilities:

```
log P(z)/Q(z)
```

- If P(z) = 0.8 and Q(z) = 0.8 → log(1) = 0. They agree. No penalty.
- If P(z) = 0.8 and Q(z) = 0.01 → log(80) = big. P cares about this region, Q doesn't. Big penalty.
- If P(z) = 0.01 and Q(z) = 0.8 → log(0.0125) = negative. But this gets *weighted* by P(z) = 0.01, so it barely contributes.

That weighting is the key to everything.

### The Asymmetry: Who is the "Boss"?

KL divergence is **not symmetric**: D_KL(P‖Q) ≠ D_KL(Q‖P).

The distribution that appears **first** (the one you're taking the expectation under) is the "Boss." The Boss decides which regions matter.

---

### Forward KL: D_KL(P ‖ Q)

```
D_KL(P ‖ Q) = E_{z ~ P} [ log P(z)/Q(z) ]
```

**P is the Boss.** You sample from P (the true distribution) and ask: "Does Q explain these samples well?"

#### The penalty structure:

| P(z) | Q(z) | log ratio | Weight (P) | Contribution |
|-------|-------|-----------|------------|-------------|
| High | High | ≈ 0 | High | ≈ 0 (they agree) |
| **High** | **Low** | **Very large** | **High** | **HUGE penalty** ☠️ |
| Low | High | Negative | Low | Tiny (Boss doesn't care) |
| Low | Low | ≈ 0 | Low | ≈ 0 |

**The terrifying case:** P(z) > 0 but Q(z) ≈ 0. The log ratio blows up to infinity, and it's weighted by a large P(z). This is catastrophic.

#### What this forces Q to do:

Q **must cover every region** where P has probability mass. It cannot afford to have Q ≈ 0 anywhere P is non-zero.

#### The consequence — Mean-Seeking / Mass-Covering:

Imagine P is **bimodal** — two humps (say, one at z = −5 and one at z = +5). Q is a single Gaussian, so it can only make one hump.

What does Q do? It **stretches wide** to cover both humps. It would rather assign probability to the empty valley between the humps (where no real data lives) than risk leaving one hump uncovered and getting infinite penalty.

```
Forward KL result with bimodal P:

  P(z):     ∧         ∧
           / \       / \
          /   \     /   \
    ─────/─────\───/─────\─────→ z
        -5     0       +5

  Q(z):      ╱‾‾‾‾‾‾‾‾‾╲          ← Q spreads wide, covers both
            ╱             ╲            but also covers the empty
    ───────╱───────────────╲───→ z     valley between them
          -5       0       +5

  Problem: Q assigns probability to the VALLEY (z ≈ 0)
           where NO real data exists → blurry samples
```

> **In image generation:** Forward KL produces outputs that are **blurry averages** — a smeared combination of features from different modes, rather than a sharp commitment to one.

---

### Reverse KL: D_KL(Q ‖ P)

```
D_KL(Q ‖ P) = E_{z ~ Q} [ log Q(z)/P(z) ]
```

**Q is the Boss.** You sample from Q (your model) and ask: "Are my samples landing in high-density regions of P?"

#### The penalty structure:

| Q(z) | P(z) | log ratio | Weight (Q) | Contribution |
|-------|-------|-----------|------------|-------------|
| High | High | ≈ 0 | High | ≈ 0 (they agree) |
| **High** | **Low** | **Very large** | **High** | **HUGE penalty** ☠️ |
| Low | High | Negative | Low | Tiny (Boss doesn't care) |
| Low | Low | ≈ 0 | Low | ≈ 0 |

**The terrifying case flips:** Q(z) > 0 but P(z) ≈ 0. Q is putting mass where the true distribution says "nothing should exist here." Penalty explodes.

#### What this forces Q to do:

Q **must only exist where P also has mass.** Q is terrified of putting probability anywhere P is near zero. This is called **zero-forcing**.

#### The consequence — Mode-Seeking:

Same bimodal P. What does Q do now?

Q **picks one hump and sits on it tightly.** It would rather completely ignore the second hump than risk spreading into the valley where P ≈ 0.

```
Reverse KL result with bimodal P:

  P(z):     ∧         ∧
           / \       / \
          /   \     /   \
    ─────/─────\───/─────\─────→ z
        -5     0       +5

  Q(z):     ∧                      ← Q picks ONE mode
           /|\                        sits tightly inside it
          / | \                       completely ignores the other
    ─────/──+──\───────────────→ z
        -5     0       +5

  The other mode at +5? Completely dropped.
  But the samples Q produces are SHARP and REALISTIC.
```

> **In image generation:** Reverse KL produces outputs that are **sharp and realistic** — committed to one coherent mode. But it may miss entire categories of images (mode-dropping).

---

### Side-by-Side Summary

| | Forward KL: D_KL(P ‖ Q) | Reverse KL: D_KL(Q ‖ P) |
|---|---|---|
| **Boss** | P (true distribution) | Q (your model) |
| **Catastrophic case** | Q = 0 where P > 0 | Q > 0 where P = 0 |
| **Behaviour** | Mean-seeking / Mass-covering | Mode-seeking / Zero-forcing |
| **If P is bimodal** | Q spreads across both humps + valley | Q picks one hump, ignores the other |
| **Sample quality** | Blurry (averaged between modes) | Sharp (committed to one mode) |
| **Coverage** | Covers all modes | May drop modes |

---

### Reading the Graphs (from the slides)

The four panels in the slide show exactly these four scenarios:

```
┌─────────────────────────┬─────────────────────────┐
│  FORWARD KL — BAD       │  REVERSE KL — BAD       │
│                         │                         │
│  Q is too narrow,       │  Q leaks outside P,     │
│  misses part of P       │  assigns mass where     │
│  → KL = ∞               │  P ≈ 0 → KL = ∞        │
│                         │                         │
│  P: ████████████        │  P:    ████████         │
│  Q:      ████           │  Q: ████████████        │
│       ↑ UNCOVERED!      │     ↑ LEAKED!           │
├─────────────────────────┼─────────────────────────┤
│  FORWARD KL — GOOD      │  REVERSE KL — GOOD      │
│                         │                         │
│  Q spreads wide to      │  Q squeezes tight       │
│  cover all of P         │  under one mode of P    │
│  → KL small             │  → KL small             │
│                         │                         │
│  P: ██    ██            │  P: ██    ██            │
│  Q: ████████████        │  Q: ██                  │
│     ↑ BLURRY but safe   │     ↑ SHARP but drops   │
│                         │       the second mode   │
└─────────────────────────┴─────────────────────────┘
```

**Top-left (Forward, BAD):** Q doesn't cover all of P → infinite penalty → optimizer won't allow this.

**Top-right (Reverse, BAD):** Q extends beyond P → infinite penalty → optimizer won't allow this.

**Bottom-left (Forward, GOOD):** Q covers everything P covers, even if it wastes probability on empty regions. This is the best Q can do under forward KL. Result: blurry.

**Bottom-right (Reverse, GOOD):** Q sits entirely within one mode of P. It ignores the other mode, but every sample it produces is valid. This is the best Q can do under reverse KL. Result: sharp.

---

## 10. Why VAEs Use Reverse KL

In a VAE:

- **Q** = q(z|x) — the encoder's approximate posterior
- **P** = p(z|x) — the true (intractable) posterior

We minimise **D_KL(q(z|x) ‖ p(z|x))** — that's **reverse KL**.

Why?

1. **Sharp latent codes:** The encoder commits to a specific, well-defined region of latent space for each input x. The decoder only ever sees latent codes from "real" regions → produces sharp outputs.

2. **Zero-forcing safety:** q never assigns probability to latent regions where the true posterior is near zero. This means the decoder never receives garbage z values.

3. **Practical reason:** In the ELBO derivation, reverse KL is what naturally falls out. The intractable term KL(q‖p(z|x)) gets absorbed — you never need to compute p(z|x) directly.

### The tradeoff: Mode-Dropping

Yes, reverse KL means q(z|x) might only capture **one mode** of the true posterior. If the true posterior is multimodal (multiple valid z for a given x), the encoder will pick one and ignore the rest.

**But this is acceptable for VAEs** because:
- For each input x, you just need **one** coherent z, not all possible z's
- A sharp, realistic reconstruction is more valuable than a blurry average

### How do people fix mode-dropping?

This is an active research area:

| Approach | How it helps |
|----------|-------------|
| **Mixture of Gaussians prior** | Uses K Gaussians instead of one, allowing multiple modes |
| **VQ-VAE** | Discrete codebook — different codes capture different modes |
| **Diffusion models** | Sidestep KL entirely with a different training objective |
| **Normalising flows** | Make q(z\|x) more expressive than a simple Gaussian |

---

## 11. The Slide Confusion: KL Against p(z|x) vs N(0,1)

This is worth calling out explicitly because it's a common source of confusion.

### What the slide shows (the *motivation*):

```
Ideal goal: minimise KL(q(z|x) ‖ p_θ(z|x))
```

This says "make q match the true posterior." It's the conceptual starting point.

### What you *actually* compute in training:

```
Actual training: minimise KL(q(z|x) ‖ N(0, 1))
```

This is the closed-form KL against the prior — the regularisation term in the ELBO.

### How you get from one to the other:

```
Step 1:  Ideal goal      → minimise KL(q(z|x) ‖ p(z|x))
                              ↓
                     Can't compute this (p(z|x) is intractable)
                              ↓
Step 2:  Reformulate     → maximise ELBO
                              ↓
                     ELBO = E[log p(x|z)] − KL(q(z|x) ‖ N(0,1))
                              ↓
Step 3:  Actual training → Compute both terms (both are tractable!)
```

The magic: by maximising the ELBO, you're **implicitly** minimising KL(q ‖ p(z|x)) without ever computing p(z|x). The ELBO derivation makes the intractable term disappear.

> [!IMPORTANT]
> The closed-form KL formula (½ Σ(μ² + σ² − log σ² − 1)) is **always** the KL against N(0,1), never against p(z|x). If a slide shows that formula next to KL(q ‖ p(z|x)), the slide is being sloppy with notation.

---

## 12. The Full VAE Loss Function

Putting it all together, the VAE training loss (to minimise) is:

```
L = −ELBO = −E[log p(x|z)] + KL(q(z|x) ‖ p(z))
```

Which in practice becomes:

```
L = ‖x − x̂‖²  +  ½ Σᵢ (μᵢ² + σᵢ² − log(σᵢ²) − 1)
    ↑                ↑
    Reconstruction   KL Regularisation
    (MSE loss)       (closed-form)
```

Where x̂ = decoder(z) and z = μ + σ · ε with ε ~ N(0,1).

### Training pipeline:

```
Input x
   │
   ▼
┌──────────┐
│ Encoder  │ → outputs μ, σ
└──────────┘
   │
   ▼
z = μ + σ · ε,  ε ~ N(0,1)    ← reparameterisation trick
   │
   ▼
┌──────────┐
│ Decoder  │ → outputs x̂
└──────────┘
   │
   ▼
Loss = MSE(x, x̂) + KL(μ, σ)
   │
   ▼
Backpropagate through everything
(gradients flow through z because of the reparam trick)
```

---

## 13. Quick Reference Card

| Concept | One-liner |
|---------|-----------|
| **p(x)** | The data distribution (what we want to model, intractable) |
| **p(z)** | The prior over latent space (we choose N(0,1)) |
| **p(x\|z)** | The decoder — "given a latent code, generate an image" |
| **p(z\|x)** | The true posterior — "given an image, what latent code made it?" (intractable) |
| **q(z\|x)** | The encoder's approximation of p(z\|x) — a Gaussian N(μ, σ²) |
| **ELBO** | Lower bound on log p(x); the thing we actually optimise |
| **Reconstruction term** | E[log p(x\|z)] — "can the decoder rebuild x from z?" |
| **KL term** | KL(q(z\|x) ‖ p(z)) — "is the latent space staying organised?" |
| **Reparam trick** | z = μ + σε makes sampling differentiable |
| **Forward KL** | Mean-seeking, mass-covering → blurry outputs |
| **Reverse KL** | Mode-seeking, zero-forcing → sharp outputs, may drop modes |
| **Mode-dropping** | Reverse KL ignores some modes of P — a known limitation |

---

## 14. The One-Paragraph Summary

A VAE learns a latent representation of data by training an encoder and decoder together. The encoder maps an input x to a distribution q(z|x) = N(μ, σ²) over latent codes, and the decoder maps a sampled z back to a reconstruction x̂. Since the true posterior p(z|x) is intractable (it requires the intractable p(x)), we optimise a lower bound (the ELBO) instead. The ELBO has two terms: a reconstruction loss (how well can we rebuild x?) and a KL regulariser (how close is q(z|x) to the prior N(0,1)?). The reparameterisation trick (z = μ + σε) makes this trainable with backprop. VAEs use reverse KL, which is mode-seeking — the encoder commits to sharp, realistic latent regions rather than spreading into blurry averages, at the cost of potentially missing some modes of the true posterior.
