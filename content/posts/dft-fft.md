---
date: 2025-03-07T17:05:08.000Z
nocite: "[@*]"
title: The Discrete Fourier Transform and Fast Fourier Transform
---

---

We analyze [^1] the theoretical properties of both the _Discrete Fourier
Transform_ (DFT) and the optimized _Fast Fourier Transform_ (FFT), and
estimate their algorithmic complexity.

# Discrete Fourier Transform

## Theory

The DFT converts a finite-length time-domain signal into its
frequency-domain representation. For an input signal $x[n]$ of length
$N$, where $0 \le n \le N-1$, the DFT is defined as:

$$
X[k] = \sum_{n=0}^{N-1} x[n]\, W_N^{kn}, \quad 0 \le k \le N-1
\label{eq:DFT}
$$

where $W_N=e^{-j\frac{2\pi}{N}}$ is the $N$-th principal root of unity,
and $X[k]$ is component at frequency $kf_s/N$ with sampling frequency
$f_s$. We can rewrite $X[k]$ as an inner product:

$$
\begin{align}
X[k] &= \left[ 1 \quad e^{-j\frac{2\pi k}{N}} \quad \ldots \quad e^{-j\frac{2\pi k}{N}(N-1)} \right]
    \begin{bmatrix}
    x[0] \\
    x[1] \\
    \vdots \\
    x[N-1]
    \end{bmatrix} \\
&= \left[ 1 \quad W_N^k \quad \ldots \quad W_N^{(N-1)k} \right]
    \begin{bmatrix}
    x[0] \\
    x[1] \\
    \vdots \\
    x[N-1]
    \end{bmatrix}
\end{align}
$$

By varying $k$ from $0$ to $N-1$, we can collate all outputs ${X[k]}$
into a vector $\mathbf{X}$, collate all inputs $x[n]$ into a vector
$\mathbf{x}$, to get:

$$
\mathbf{X}=\mathbf{W}\mathbf{x}, \quad
\mathbf{W} =
\begin{bmatrix}
1 & 1 & 1 & \cdots & 1 \\
1 & W_N & W_N^2 & \cdots & W_N^{N-1} \\
1 & W_N^2 & W_N^4 & \cdots & W_N^{2(N-1)} \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
1 & W_N^{N-1} & W_N^{2(N-1)} & \cdots & W_N^{(N-1)(N-1)}
\end{bmatrix}
$$

where $\mathbf{W}$ is the DFT matrix.

## Complexity

{{< highlight matlab >}}
function X = dft_vectorized(x)
% Compute the Discrete Fourier Transform (DFT) of input vector x using vectorized operations
% x : input signal (vector)
% X : DFT of x

    % Convert input to a column vector for consistency
    x = x(:);
    N = length(x);       % Number of samples in the signal

    % Create index vectors: n as a row vector and k as a column vector
    n = 0:N-1;           % Time indices (row vector)
    k = n';              % Frequency indices (column vector)

    % Construct the DFT matrix using the formula: exp(-1j*2*pi*k*n/N)
    W = exp(-1j * 2 * pi * k * n / N);

    % Multiply the DFT matrix with the signal to obtain the transform
    X = W * x;

end
{{< /highlight >}}

Direct computation of $\mathbf{X}$ requires $(N-1)^2$ complex
multiplications and $N(N-1)$ complex additions, as in Algorithm
[\[alg:dft\]](#alg:dft){reference-type="ref+label" reference="alg:dft"}.
Since multiplication is more compute-intensive than addition, the
asymptotic complexity is dominated by multiplication, giving $O(N^2)$.

# Fast Fourier Transform

## Theory

The radix-2 FFT algorithms work by dividing the DFT into 2 DFTs of
length $N/2$ each, and iterating. We introduce the simplest variant,
called the _Decimation-In-Time_ (DIT) algorithm. DIT begins by splitting
$x[n]$ into two parts --- one for the even-indexed values $x[2n]$ and
one for the odd-indexed values $x[2n + 1]$. Define two $N/2$-point
signals $x_{even}[n]$ and $x_{odd}[n]$ as
$$x_{even}[n] = x[2n], \quad x_{odd}[n] = x[2n + 1], \quad 0 \leq n \leq N/2-1$$

using the DFT of these 2 splits gives us:

$$
\begin{aligned}
X[k] &= X_{even}[k] + W_N^{k} \cdot X_{odd}[k] \quad \text{for } 0 \leq k \leq \frac{N}{2} - 1 \\
X[k + N/2] &= X_{even}[k] - W_N^{k} \cdot X_{odd}[k] \quad \text{for } 0 \leq k \leq \frac{N}{2} - 1
\label{eq:fft_final}
\end{aligned}
$$

The first computation with $+W_N^k$ give us the first half of the full
DFT vector $\mathbf{X}$, while the second computation with $-W_N^k$ give
us the second half of $\mathbf{X}$. The proof is laid out in the
Appendix.

## Complexity

:::: algorithm
::: algorithmic
$N \gets \text{length}(\mathbf{x})$

$\mathbf{x}$

$\mathbf{x}_{even} \gets \mathbf{x}[0:2:N-1]$
$\mathbf{x}_{odd} \gets \mathbf{x}[1:2:N-1]$

$\mathbf{X}_{even} \gets \text{FFT}(\mathbf{x}_{even})$
$\mathbf{X}_{odd} \gets \text{FFT}(\mathbf{x}_{odd})$

$\mathbf{X} \gets \text{zeros}(N)$

$W_N^k \gets e^{-j2\pi k/N}$
$\mathbf{X}[k] \gets \mathbf{X}_{even}[k] + W_N^k \cdot \mathbf{X}_{odd}[k]$
$\mathbf{X}[k+N/2] \gets \mathbf{X}_{even}[k] - W_N^k \cdot \mathbf{X}_{odd}[k]$

$\mathbf{X}$
:::
::::

We can follow the above procedure to split an $N$-point DFT into two
$\frac{N}{2}$-point DFT, giving us Algorithm
[\[alg:fft\]](#alg:fft){reference-type="ref+label" reference="alg:fft"}.
Let $A_c(N)$ and $M_c(N)$ denote respectively the number of complex
additions and multiplications for computing the DFT of an $N$-point
complex sequence $x[n]$. Let $N$ be a power of 2, $N = 2^k$. Then, we
have
$$A_c(N) = 2 A_c(N/2) + N \quad , \quad M_c(N) = 2 M_c(N/2) + \frac{N}{2} - 1$$

as $N$ complex additions (addition of even and odd terms) and
$\frac{N}{2} - 1$ complex multiplications ($W_N^{k} \cdot X_{odd}[k]$)
are required to put the two $N/2$-point DFTs together. Note that a
2-point DFT is simply a sum and difference
$X[0] = x[0] + x[1] , \quad X[1] = x[0] - x[1]$. Hence, the starting
conditions are $A_c(2) = 2$ and $M_c(2) = 0$. Solving the recursive
equation yields
$$A_c(N) = N \log_2 N  \quad , \quad M_c(N) = \frac{N}{2} \log_2 N - N + 1$$

Our overall complexity is $O(N\log N)$.

# Experiments

## Complexity

![Execution time versus signal length $N$ for various implementations of
DFT and FFT](/figures/complex.jpg){#fig:complexity
width="40%"}

::: {#tab:R2}
Algorithm O(N) O(N²) O(N³) O(log N) O(N log N)

---

my-DFT 0.9208 0.9999 0.9864 0.4167 0.9417
my-FFT 0.9830 0.9657 0.9161 0.5634 0.9894
MATLAB-FFT 0.8491 0.7141 0.6526 0.8537 0.8287

: R^2^ values for different complexity models.
:::

To verify the theoretical complexity, we measured the execution time for
computing the DFT over a range of signal lengths. For each signal length
$N$, $N$ samples were taken from a 5Hz sine wave, and the DFT/FFT
computation time recorded. We compare our implementation of Algorithm
[\[alg:dft\]](#alg:dft){reference-type="ref+label" reference="alg:dft"}
as `my-DFT`, our implementation of Algorithm
[\[alg:fft\]](#alg:fft){reference-type="ref+label" reference="alg:fft"}
as `my-FFT`, and the `MATLAB` FFT implementation as `MATLAB-FFT`.
[1](#fig:complexity){reference-type="ref+label"
reference="fig:complexity"} shows the measured execution times on a
log-log plot. The FFT results clearly exhibit an $O(N\log N)$ scaling,
while the DFT scales as $O(N^2)$. These experimental results confirm the
significant computational advantage of using the FFT for large-scale
problems. We also note the much more efficient implementation of
`MATLAB` FFT, which uses the `FFTW` [^2] package.
[1](#tab:R2){reference-type="ref+label" reference="tab:R2"} show the how
well different models fit to data. Both FFT implementations fit well to
$O(N)$ and $O(N\log N)$ models.

## Numerical Error

![Reconstruction error versus signal length $N$ for various
implementations of DFT and
FFT](/figures/recon.jpg){#fig:recon width="40%"}

We evaluate numerical accuracy of both algorithms in
[2](#fig:recon){reference-type="ref+label" reference="fig:recon"} by
measuring reconstruction error of the reconstructed signal
$\text{RMSE} = \sqrt{\frac{1}{N}\sum_{i=1}^{N}|x_i - \hat{x}_i|^2}$
where $x_i$ is the original signal and $\hat{x}_i$ is the reconstructed
signal (using the `MATLAB` Inverse DFT function) Despite all errors
falling within the $10^{-12}$ range, indicating high overall accuracy,
the custom DFT implementation exhibits exponentially growing error with
increasing sequence length. In contrast, both FFT implementations
maintain consistently minimal error across all tested sequence lengths.
The superior numerical stability of FFT algorithms is possibly due to
its lower operation count (lower algorithmic complexity). Floating-point
rounding errors accumulate more significantly in DFT.

# Conclusion

In this report, we investigated both theoretical and practical aspects
of the DFT and the FFT. The analysis verified that the computational
complexity of the direct DFT implementation scales as $O(N^2)$, whereas
our FFT implementation achieved the expected $O(N \log N)$ complexity,
offering a significant improvement in efficiency. Moreover, the FFT
exhibited superior numerical stability. Selecting the appropriate
windowing function is also crucial for accurate frequency analysis, with
the Han or Hamming window recommended for most applications. All code
available at <https://github.com/liuzihe02/dft-fft.git>.

# Appendix: Derivation of FFT {#app:fft_proof .unnumbered}

Consider again a $N$-point signal $x[n]$ of even length. The derivation
of the DIT radix-2 FFT begins by splitting the $x[n]$ into two parts ---
one part for the even-indexed values $x[2n]$ and one part for the
odd-indexed values $x[2n + 1]$. Define two $N/2$-point signals
$x_{even}[n]$ and $x_{odd}[n]$ as
$$x_{even}[n] = x[2n], \quad x_{odd}[n] = x[2n + 1], \quad 0 \leq n \leq N/2-1$$

The DFT can be written as

$$
\begin{aligned}
X[k] &= \sum_{n=0 \atop n \text{ even}}^{N-1} x[n] W_N^{nk} + \sum_{n=0 \atop n \text{ odd}}^{N-1} x[n] W_N^{nk}\\
&= \sum_{n=0}^{N/2-1} x[2n] W_N^{2nk} + \sum_{n=0}^{N/2-1} x[2n + 1] W_N^{(2n+1)k}\\
&= \sum_{n=0}^{N/2-1} x_{even}[n] W_N^{2nk} + \sum_{n=0}^{N/2-1} x_{odd}[n] W_N^{(2n+1)k} \\
&= \sum_{n=0}^{N/2-1} x_{even}[n] W_N^{2nk} + W_N^{k} \cdot \sum_{n=0}^{N/2-1} x_{odd}[n] W_N^{2nk}\\
\end{aligned}
$$

Noting that $W_N^{2}=W_{N/2}$ or more generally
$W_N^{2nk}=W_{N/2}^{nk}$, $$\begin{aligned}
X[k]&= \sum_{n=0}^{N/2-1} x_{even}[n] W_{N/2}^{nk} + W_N^{k} \cdot \sum_{n=0}^{N/2-1} x_{odd}[n] W_{N/2}^{nk}
\end{aligned}$$

Recognizing that the $\frac{N}{2}$-point DFT of $x_{even}[n]$ and
$x_{odd}[n]$ are given by:

$$X_{even}[k] = \text{DFT}_{\frac{N}{2}}\{x_{even}[n]\} = \sum_{n=0}^{N/2-1} x_{even}[n] W_{N/2}^{nk} \quad , \quad X_{odd}[k] = \text{DFT}_{\frac{N}{2}}\{x_{odd}[n]\} = \sum_{n=0}^{N/2-1} x_{odd}[n] W_{N/2}^{nk}$$

we then obtain our core recursive equation:

$$
X[k] = X_{even}[k] + W_N^{k} \cdot X_{odd}[k]
    \label{eq:fft_int}
$$

Since $x_{even}[n]$ and $x_{odd}[n]$ are $N/2$-point signals with
$0 \leq n \leq N/2-1$, their DFT are also only $N/2$-point signals.
However, we require $0 \le k \le N-1$. We resolve this by noting their
DFT coefficients are periodic with a period of $\frac{N}{2}$

$$X_{even}[k] = X_{even}\left[k + \frac{N}{2}\right], \quad X_{odd}[k] = X_{odd}\left[k + \frac{N}{2}\right]$$

This gives us

$$
X[k] =
    \begin{cases}
    X_{even}[k] + W_N^{k} \cdot X_{odd}[k], & \text{for } k = 0,1,\ldots,\frac{N}{2}-1 \\
    X_{even}\left[k-\frac{N}{2}\right] + W_N^{k} \cdot X_{odd}\left[k-\frac{N}{2}\right], & \text{for } k = \frac{N}{2},\frac{N}{2}+1,\ldots,N-1
\end{cases}
$$

Noting $W_N^{k +\frac{N}{2}}=-W_{N}^{k}$, we write

$$
X[k] =
    \begin{cases}
    X_{even}[k] + W_N^{k} \cdot X_{odd}[k], & \text{for } k = 0,1,\ldots,\frac{N}{2}-1 \\
    X_{even}\left[k-\frac{N}{2}\right] - W_N^{k-\frac{N}{2}} \cdot X_{odd}\left[k-\frac{N}{2}\right], & \text{for } k = \frac{N}{2},\frac{N}{2}+1,\ldots,N-1
\end{cases}
$$

finally giving us
[\[eq:fft_final\]](#eq:fft_final){reference-type="ref+label"
reference="eq:fft*final"}. The multipliers $W_N^k$ are known as \_twiddle
factors*. The first computation with $+W_N^k$ give us the first half of
the full DFT vector $\mathbf{X}$, while the second computation with
$-W_N^k$ give us the second half of $\mathbf{X}$.

[^1]: This post was created for educational purposes, so much of the analysis is taken directly from the sources quoted in the references
[^2]: This package is highly optimized to each machine and is written inlow level C
