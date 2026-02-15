---
date: 2026-02-14T23:50:56+00:00
title: The Minority Guessing Game
---

I recently learnt a drinking game where you had 2 cards (red/black) for round, and people use these as answers to questions. The goal of the game is to locate who the minority answers came from, and with that came some interesting questions.

## Game Rules

1. $n$ players each hold a red card and a black card.
2. A question is posed each round (e.g., yes/no, would-you-rather, etc.). Each player secretly plays one card to reflect their answer.
3. All played cards are collected and shuffled — the total counts of each red and black are revealed, but not who played what.
4. This reveals a *minority* group of size $k$ and a majority of size $n − k$
5. The group is given $g$ guesses to identify members of the minority. The group's score is $C$, the number of correct guesses. This is the only random variable in this question; everything else is given.

*Then I had a question: what's a good number for $g$ so that the game is fun?* [^fun1] So not too easy $g \approx n$ where we'll definitely guess who the minorities are, but not too hard $g=k$ [^hard] that we'll never find the minorities. [^change]

## Modelling Our Game

Before we attempt to answer that question, let's model our game first. Specifically, a core question is *what is the probability of getting at least $c$ correct out of $g$ guesses?* Or how do we compute

$$P(C \geq c)$$

Under random guessing, you are choosing $g$ players uniformly at random from a population of $n$. $k$ are minorities in this population. This is sampling without replacement, which gives rise to the *hypergeometric distribution*.

### Hypergeometric Distribution

The total number of ways to choose $g$ players from $n$ is $\binom{n}{g}$. For exactly $c$ of those to be minorities, we need to independently choose $c$ from the $k$ minorities *and* $g - c$ from the $n - k$ majority. So the probability mass function for $C$ [^range] follows a hypergeometric distribution where:

$$P(C = c) = \frac{\binom{k}{c}\binom{n-k}{g-c}}{\binom{n}{g}}$$

| Term | Meaning |
|------|---------|
| $\binom{k}{c}$ | Ways to correctly pick $c$ minorities from the $k$ available |
| $\binom{n-k}{g-c}$ | Ways to incorrectly pick $g - c$ non-minorities from the $n - k$ majority |
| $\binom{n}{g}$ | Total ways to pick $g$ players from $n$ |

### Survival Function

We can then calculate the probability of getting at least $c$ guesses correct by summing the PMF from $c$ upward:

$$P(C \geq c) = \sum_{i=c}^{\min(g,\,k)} \frac{\binom{k}{i}\binom{n-k}{g-i}}{\binom{n}{g}}$$

### Expected Value

$$E[C] = \frac{gk}{n}$$

> This is intuitive: each guess has a $\frac{k}{n}$ chance of landing on a minority, and you make $g$ guesses.

### Variance

$$\text{Var}(C) = g \cdot \frac{k}{n} \cdot \frac{n-k}{n} \cdot \frac{n-g}{n-1}$$

> Note that when $g = n$ (you guess everyone), the variance collapses to 0 — you deterministically identify all minorities.

## Choosing $g$: What Makes the Game Fun?

Now we can return to the original question: how to choose $g$ that makes for a fun game? A natural "fun" thing to have is **finding all $k$ minorities**. 

### The Falling Factorial Simplification

The probability of a perfect guess under random guessing is $P(C = k)$. Setting $c = k$ in the PMF gives 

$$P(C = k) = \frac{\binom{k}{k}\binom{n-k}{g-k}}{\binom{n}{g}} = \frac{\binom{n-k}{g-k}}{\binom{n}{g}} = \frac{\frac{(n-k)!}{(g-k)!\,(n-g)!}}{\frac{n!}{g!\,(n-g)!}} = \frac{(n-k)! \cdot g!}{(g-k)! \cdot n!}$$

Now notice that $\frac{g!}{(g-k)!} = g(g-1)\cdots(g-k+1)$ and $\frac{n!}{(n-k)!} = n(n-1)\cdots(n-k+1)$. These are **falling factorials**, written $g^{\underline{k}}$ and $n^{\underline{k}}$ respectively. So:

$$P(C = k) = \frac{g^{\underline{k}}}{n^{\underline{k}}} = \prod_{i=0}^{k-1} \frac{g - i}{n - i}$$

Each factor $\frac{g-i}{n-i}$ has a nice interpretation: it's the probability that the $(i+1)$-th minority is among your remaining guesses, given that the previous $i$ minorities were already found.

### Deriving the Heuristic

The game is fun when guessing everyone is achievable but unlikely — say $\approx 50\%$ for a casual drinking game (below 20% feels difficult; above 70% feels trivial). Let $f$ be probability of guessing everyone correctly (a.k.a our fun probability say $f = 0.5$). The number of guesses $g$ is the main difficulty knob, and since $k$ changes every round, $g$ should depend on both $n$ and $k$. So we want to find $g$ such that:

$$\prod_{i=0}^{k-1} \frac{g - i}{n - i} = f$$

This is a degree-$k$ polynomial in $g$, so there's no general closed form.[^abel] But the product has $k$ factors that decrease from $\frac{g}{n}$ (at $i=0$) down to $\frac{g-k+1}{n-k+1}$ (at $i=k-1$). If we approximate every factor by the largest one $\frac{g}{n}$, the product becomes $\left(\frac{g}{n}\right)^k \approx f$, which gives:

$$\boxed{g \approx \text{round}\!\left(n \cdot f^{1/k}\right)}$$

This overestimates each factor (since later terms are smaller than $\frac{g}{n}$), but the error is small and — crucially — you can actually compute this after a few drinks. More accurate alternatives exist[^alternatives] but aren't worth the mental overhead.

### Verification

Tested over 676 $(n, k, f)$ triples: $n \in [5, 15]$, $k \in [1, \lfloor n/2 \rfloor]$, $f \in \{20\%, 25\%, \ldots, 80\%\}$.

The heuristic $g = \text{round}(n \cdot f^{1/k})$ is *always within $\pm 1$ of the true optimal* across the entire parameter space — it never misses by 2 or more.

**Exact match rate by target probability $f$:**

| $f$ | 20% | 30% | 40% | 50% | 60% | 70% | 80% |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Exact match | 56% | 75% | 77% | 83% | 94% | 96% | 92% |

The approximation improves as $f$ increases, since a larger $g$ means each $\frac{g-i}{n-i}$ is closer to $\frac{g}{n}$.

**Exact match rate by minority size $k$:**

| $k$ | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Exact match | 96% | 84% | 79% | 76% | 76% | 73% | 73% |

Performance degrades as $k$ grows — when $k$ is large relative to $n$, the offset terms $i$ in $\frac{g-i}{n-i}$ are no longer negligible compared to $g$ and $n$. But even at $k = 7$ (the maximum for $n = 15$), the heuristic is always within $\pm 1$.

## Conclusion

To play a fun game where around $f=50\%$ of the time we guess everyone correctly, at each round pick $g$ such that $g \approx \text{round}\!\left(n \cdot f^{1/k}\right)$.

> Crazy, I know. Especially when drunk - did I mention this is a drinking game? =D

---

[^abel]: For $k \geq 5$ there's no general closed form by the Abel–Ruffini theorem, but the approximation works well for all $k$.

[^alternatives]: Using the **middle factor** $\frac{g-(k-1)/2}{n-(k-1)/2}$ instead of the first gives $g \approx \frac{k-1}{2} + (n - \frac{k-1}{2})\,f^{1/k}$, which achieves 96.4% exact match vs the naive's 82.1%. A **geometric mean of endpoints** approach gives a quadratic $g = \frac{(k-1) + \sqrt{(k-1)^2 + 4f^{2/k}\,n(n-k+1)}}{2}$, at 94.5% exact. Both are always within $\pm 1$, but neither is worth the calculation involved in some stupid drinking game...

[^change]: Note that every round $k$ changes, so $g$ should really depend on both $n$ and $k$ for the game to be fun. So $g$ should change every round.

[^hard]: The reason why I say $g = k$ (you get exactly as many guesses as there are minorities) makes a hard game is because think about the scenario where $n$ is large and $k = 1$. Then you only have one guess to find the minority out of a huge number of people, which frankly is quite impossible! The probability of all correct guesses is $P(C = k) = \frac{1}{\binom{n}{k}}$. This drops off really rapidly. For $n = 10, k = 3$: $P = \frac{1}{120} \approx 0.83\%$.

[^fun1]: For a fun game, $g$ really should be greater than $k$, so it is actually possible to guess all the minorities. But we don't want this to be too big (close to $n$), if not we'll definitely guess all the minorities for sure and the game won't be fun.

[^range]: The valid range of $c$ is $c \in \{\max(0,\; g - n + k),\;\dots,\;\min(g,\;k)\}$. The upper bound $\min(g, k)$ is clear: you can't guess more minorities than exist ($k$), nor more than you have guesses ($g$). The lower bound $\max(0, g - n + k)$ is less obvious — it says that if you make enough guesses ($g > n - k$), you're *forced* to hit some minorities because there aren't enough majority members to fill all your guesses.