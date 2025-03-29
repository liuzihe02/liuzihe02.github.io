---
title: "My First Post"
date: 2022-10-17T22:45:36+11:00
#use this to hide a document
draft: true
math: true
---

This is a post about latex.

An equation:
$$\int_{-\infty}^{\infty} e^{-x^2} dx$$. <!-- works -->

inline example: $\sum_{i = 0}^N 2i = y$ <!-- works -->

One overbrace:

$${a}^{b} - \overbrace{c}^{d}$$ <!-- works-->

Two overbraces:
$$\underbrace{a}_{b} - \underbrace{c}_{d}$$ <!--does not work -->

None of these below works properly:

$$
\begin{aligned}
        equation &= 16 \\
        other &= 26 + 13
\end{aligned}
$$

$$
\begin{pmatrix}
   a & b \\
      c & d
      \end{pmatrix}
$$
