# Chapter 2: Monte Carlo Integration

<!-- interactive-nav -->
## Contents

- [Monte Carlo Integration (wgpu)](#monte-carlo-integration-wgpu)
- [2.1 Monte Carlo: Basics](#21-monte-carlo-basics)
  - [2.1.1 Background and Probability Review](#211-background-and-probability-review)
  - [2.1.2 Expected Values](#212-expected-values)
  - [2.1.3 The Monte Carlo Estimator](#213-the-monte-carlo-estimator)
  - [2.1.4 Error in Monte Carlo Estimators](#214-error-in-monte-carlo-estimators)
- [2.2 Improving Efficiency](#22-improving-efficiency)
  - [2.2.1 Stratified Sampling](#221-stratified-sampling)
  - [2.2.2 Importance Sampling](#222-importance-sampling)
  - [2.2.3 Multiple Importance Sampling](#223-multiple-importance-sampling)
  - [2.2.4 Russian Roulette](#224-russian-roulette)
  - [2.2.5 Splitting](#225-splitting)
- [2.3 Sampling Using the Inversion Method](#23-sampling-using-the-inversion-method)
  - [2.3.1 Discrete Case](#231-discrete-case)
  - [2.3.2 Continuous Case](#232-continuous-case)
- [2.4 Transforming Between Distributions](#24-transforming-between-distributions)
  - [2.4.1 Transformation in Multiple Dimensions](#241-transformation-in-multiple-dimensions)
  - [2.4.2 Sampling with Multidimensional Transformations](#242-sampling-with-multidimensional-transformations)
- [Further Reading](#further-reading)
- [Exercises](#exercises)
- [Notes](#notes)

---

## Monte Carlo Integration (wgpu)

```mermaid
flowchart LR
    A[Sample the domain] --> B[Evaluate the weighted integrand f/p]
    B --> C[Accumulate the sum]
    C --> D{More samples?}
    D -- yes --> A
    D -- no --> E[Divide by n]
    E --> F[Integral estimate]
```

Rendering is full of integration problems. In addition to the light transport equation (1.1), the following chapters show that integral equations also describe a variety of additional quantities related to light, including the sensor response in a camera, the attenuation and scattering of light in participating media, and scattering from materials like skin. These integral equations generally do not have analytic solutions, so we must turn to numerical methods. Although standard numerical integration techniques like trapezoidal integration or Gaussian quadrature are effective at solving low-dimensional smooth integrals, their rate of convergence is poor for the higher dimensional and discontinuous integrals that are common in rendering. Monte Carlo integration techniques provide one solution to this problem: they use random sampling to evaluate integrals with a convergence rate that is independent of the dimensionality of the integrand.

Monte Carlo integration[^1] has the useful property that it only requires the ability to evaluate an integrand $f(x)$ at arbitrary points in the domain in order to estimate the value of its integral $\int f(x)\,dx$. This property not only makes Monte Carlo easy to implement but also makes the technique applicable to a broad variety of integrands. It has a natural extension to multidimensional functions; in Chapter 13, we will see that the light transport algorithm implemented in `RandomWalkIntegrator` can be shown to be estimating the value of an infinite-dimensional integral.

Judicious use of randomness has revolutionized the field of algorithm design. Randomized algorithms fall broadly into two classes: Las Vegas and Monte Carlo. Las Vegas algorithms are those that use randomness but always give the same result in the end (e.g., choosing a random array entry as the pivot element in Quicksort). Monte Carlo algorithms, on the other hand, give different results depending on the particular random numbers used along the way but give the right answer on average. So, by averaging the results of several runs of a Monte Carlo algorithm (on the same input), it is possible to find a result that is statistically very likely to be close to the true answer.

The following sections discuss the basic principles of Monte Carlo integration, focusing on those that are widely used in pbrt. See also Appendix A, which has the implementations of additional Monte Carlo sampling functions that are more rarely used in the system.

> *Printed-edition index: `RandomWalkIntegrator` (p. 33).*

## 2.1 Monte Carlo: Basics

Because Monte Carlo integration is based on randomization, we start this chapter with a brief review of ideas from probability and statistics that provide the foundations of the approach. Doing so allows us to introduce the basic Monte Carlo algorithm as well as mathematical tools for evaluating its error.

### 2.1.1 Background and Probability Review

We start by defining some terms and reviewing basic ideas from probability. We assume that the reader is already familiar with basic probability concepts; readers needing a more complete introduction to this topic should consult a textbook such as Sheldon Ross's *Introduction to Probability Models* (2002).

**Random variables.** A random variable $X$ is a value chosen by some random process. We will generally use capital letters to denote random variables, with exceptions made for a few Greek symbols that represent special random variables. Random variables are always drawn from some domain, which can be either discrete (e.g., a fixed, finite set of possibilities) or continuous (e.g., the real numbers $\mathbb{R}$). Applying a function $f$ to a random variable $X$ results in a new random variable $Y=f(X)$.

**Discrete example.** The result of a roll of a die is a discrete random variable sampled from the set of events $X_i\in\{1,2,3,4,5,6\}$. Each event has probability

$$p_i=\frac{1}{6},\qquad \sum_{i=1}^{6} p_i = 1.$$

A random variable like this one that has the same probability for all potential values is said to be *uniform*. A function $p(X)$ that gives a discrete random variable's probability is termed a *probability mass function* (PMF), so we could equivalently write $p(X_i=i)=1/6$ in this case.

**Independence.** Two random variables are independent if the probability of one does not affect the probability of the other. In this case, the joint probability $p(X,Y)$ of two random variables is given by the product of their probabilities:

$$p(X,Y)=p(X)\,p(Y).$$

For example, two random variables representing random samples of the six sides of a die are independent.

**Dependence.** For dependent random variables, one's probability affects the other's. Consider a bag filled with some number of black balls and some number of white balls. If we randomly choose two balls from the bag, the probability of the second ball being white is affected by the color of the first ball, since its choice changes the number of balls of one type left in the bag. We say that the second ball's probability is *conditioned* on the choice of the first one. In this case, the joint probability for choosing two balls $X$ and $Y$ is given by

$$p(X,Y)=p(X)\,p(Y\mid X), \quad\text{(2.1)}$$

where $p(Y\mid X)$ is the conditional probability of $Y$ given a value of $X$.

In the following, it will often be the case that a random variable's probability is conditioned on many values; for example, when choosing a light source from which to sample illumination, the `BVHLightSampler` in Section 12.6.3 considers the 3D position of the receiving point and its surface normal, so the choice of light is conditioned on them. However, we will often omit the variables that a random variable is conditioned on in cases where there are many of them and where enumerating them would obscure notation.

**The canonical uniform random variable.** A particularly important random variable is the canonical uniform random variable $\xi$. It takes on all values in its domain $[0,1)$ independently and with uniform probability. This particular variable is important for two reasons. First, it is easy to generate a variable with this distribution in software—most runtime libraries have a pseudo-random number generator that does just that.[^2] Second, we can take $\xi$ and map it to a discrete random variable, choosing $X_i$ if

$$\sum_{j=1}^{i-1} p_j \;\le\; \xi \;<\; \sum_{j=1}^{i} p_j. \quad\text{(2.2)}$$

For lighting applications, we might want to define the probability of sampling illumination from each light in the scene based on its power $\Phi_i$ relative to the total power from all sources:

$$p_i=\frac{\Phi_i}{\sum_j \Phi_j}.$$

Notice that these $p_i$ values also sum to 1. Given such per-light probabilities, $\xi$ could be used to select a light source from which to sample illumination.

**The cumulative distribution function.** The *cumulative distribution function* (CDF) $P(x)$ of a random variable is the probability that a value from the variable's distribution is less than or equal to some value $x$:

$$P(x)=\Pr\{X\le x\}. \quad\text{(2.3)}$$

For the die example, $P(2)=\Pr\{X\le 2\}=2/6=1/3$, since two of the six possibilities are less than or equal to 2.

**Continuous random variables.** These take on values over ranges of continuous domains (e.g., the real numbers, directions on the unit sphere, or the surfaces of shapes in the scene). Beyond $\xi$, another example is the random variable that ranges over the real numbers between 0 and 2, where the probability of its taking on any particular value $x$ is proportional to $2-x$: it is twice as likely for this random variable to take on a value around 0 as it is to take one around 1, and so forth. Normalizing so that the density integrates to 1 gives

$$p(x)=\frac{2-x}{2},\qquad 0\le x\le 2.$$

**The probability density function.** The *probability density function* (PDF) formalizes this idea: it describes the relative probability of a random variable taking on a particular value and is the continuous analog of the PMF. The PDF $p(x)$ is the derivative of the random variable's CDF,

$$p(x)=\frac{dP(x)}{dx}=P'(x).$$

For uniform random variables, $p(x)$ is a constant; this is a direct consequence of uniformity. For $\xi$ we have $p(x)=1$ on $0\le x<1$ (and zero elsewhere).

PDFs are necessarily nonnegative and always integrate to 1 over their domains. Note that their value at a point $x$ is not necessarily less than 1, however.

Given an interval $[a,b]$ in the domain, integrating the PDF gives the probability that a random variable lies inside the interval:

$$\Pr\{a\le X\le b\}=\int_a^b p(x)\,dx = P(b)-P(a).$$

This follows directly from the first fundamental theorem of calculus and the definition of the PDF.

> *Printed-edition index: `BVHLightSampler` (p. 796).*

### 2.1.2 Expected Values

The expected value $\mathbb{E}_p[f(x)]$ of a function $f$ is defined as the average value of the function over some distribution of values $p(x)$ over its domain $D$:

$$\mathbb{E}_p[f(x)] = \int_D f(x)\,p(x)\,dx.$$

As an example, consider finding the expected value of the cosine function between 0 and $\pi$, where $p$ is uniform. Because the PDF $p(x)$ must integrate to 1 over the domain, $p(x)=1/\pi$, so[^3]

$$\mathbb{E}[\cos x]=\int_0^{\pi}\frac{\cos x}{\pi}\,dx = \frac{1}{\pi}\Big[\sin x\Big]_0^{\pi}=0,$$

which is precisely the expected result. (Consider the graph of $\cos x$ over $[0,\pi]$ to see why this is so.)

The expected value has a few useful properties that follow from its definition:

$$\mathbb{E}[c\,f(x)] = c\,\mathbb{E}[f(x)], \quad\text{(2.4)}$$

$$\mathbb{E}[f(x)+g(x)] = \mathbb{E}[f(x)] + \mathbb{E}[g(x)]. \quad\text{(2.5)}$$

We will repeatedly use these properties in derivations in the following sections.

### 2.1.3 The Monte Carlo Estimator

We can now define the Monte Carlo estimator, which approximates the value of an arbitrary integral. Suppose that we want to evaluate a 1D integral $\int_a^b f(x)\,dx$. Given a supply of independent uniform random variables $X_i\in[a,b]$, the Monte Carlo estimator says that the expected value of the estimator

$$F_n=\frac{b-a}{n}\sum_{i=1}^{n} f(X_i) \quad\text{(2.6)}$$

— that is, $\mathbb{E}[F_n]$ — is equal to the integral. This fact can be demonstrated with just a few steps. First, note that the PDF $p(x)$ corresponding to the random variable $X_i$ must be equal to $1/(b-a)$, since $p$ must not only be a constant but also integrate to 1 over the domain $[a,b]$. Algebraic manipulation using the properties from Equations (2.4) and (2.5) then shows that

$$\mathbb{E}[F_n] = \frac{b-a}{n}\sum_{i=1}^{n}\mathbb{E}[f(X_i)] = \frac{b-a}{n}\sum_{i=1}^{n}\int_a^b \frac{f(x)}{b-a}\,dx = \int_a^b f(x)\,dx.$$

Extending this estimator to multiple dimensions or complex integration domains is straightforward: $n$ independent samples $X_i$ are taken from a uniform multidimensional PDF, and the estimator is applied in the same way. For example, consider the 3D integral

$$\int_{x_0}^{x_1}\int_{y_0}^{y_1}\int_{z_0}^{z_1} f(x,y,z)\,dz\,dy\,dx.$$

If samples $X_i=(x_i,y_i,z_i)$ are chosen uniformly from the cube $[x_0,x_1]\times[y_0,y_1]\times[z_0,z_1]$, then the PDF $p(X)$ is the constant value

$$p(X)=\frac{1}{(x_1-x_0)(y_1-y_0)(z_1-z_0)},$$

and the estimator is

$$F_n=\frac{(x_1-x_0)(y_1-y_0)(z_1-z_0)}{n}\sum_{i=1}^{n} f(X_i).$$

The restriction to uniform random variables can be relaxed with a small generalization. This is an important step, since carefully choosing the PDF from which samples are drawn leads to a key technique for reducing error in Monte Carlo that will be introduced in Section 2.2.2. If the random variables $X_i$ are drawn from a PDF $p(x)$, then the estimator

$$\boxed{\;F_n=\frac{1}{n}\sum_{i=1}^{n}\frac{f(X_i)}{p(X_i)}\;} \quad\text{(2.7)}$$

can be used to estimate the integral instead. The only limitation on $p(x)$ is that it must be nonzero for all $x$ where $|f(x)|>0$.

It is similarly not too hard to see that the expected value of this estimator is the desired integral of $f$:

$$\mathbb{E}[F_n] =\frac{1}{n}\sum_{i=1}^{n}\mathbb{E}\!\left[\frac{f(X_i)}{p(X_i)}\right] =\frac{1}{n}\sum_{i=1}^{n}\int \frac{f(x)}{p(x)}\,p(x)\,dx =\int f(x)\,dx.$$

We can now understand the factor of $1/(4\pi)$ in the implementation of `RandomWalkIntegrator`: directions are uniformly sampled over the unit sphere, which has area $4\pi$. Because the PDF is normalized over the sampling domain, it must have the constant value $1/(4\pi)$. When the estimator of Equation (2.7) is applied, that value appears in the divisor.

With Monte Carlo, the number of samples $n$ can be chosen arbitrarily, regardless of the dimensionality of the integrand. This is another important advantage of Monte Carlo over traditional deterministic quadrature techniques, which typically require a number of samples that is exponential in the dimension.

### 2.1.4 Error in Monte Carlo Estimators

Showing that the Monte Carlo estimator converges to the right answer is not enough to justify its use; its rate of convergence is important too. Variance, the expected squared deviation of a function from its expected value, is a useful way to characterize Monte Carlo estimators' convergence.[^7] The variance of an estimator $F$ is defined as

$$\mathrm{V}[F]=\mathbb{E}\!\left[(F-\mathbb{E}[F])^2\right], \quad\text{(2.8)}$$

from which it follows that

$$\mathrm{V}[F]=\mathbb{E}[F^2]-\big(\mathbb{E}[F]\big)^2. \quad\text{(2.9)}$$

This property and Equation (2.5) yield an alternative expression for the variance; in particular, for the Monte Carlo estimator of Equation (2.7),

$$\mathrm{V}[F_n]=\frac{1}{n}\,\mathrm{V}\!\left[\frac{f(X)}{p(X)}\right].$$

If the estimator is a sum of independent random variables (like the Monte Carlo estimator $F_n$), then the variance of the sum is the sum of the individual random variables' variances:

$$\mathrm{V}\!\left[\sum_i X_i\right]=\sum_i \mathrm{V}[X_i]. \quad\text{(2.10)}$$

From Equation (2.10) it is easy to show that variance decreases linearly with the number of samples $n$. Because variance is squared error, the error in a Monte Carlo estimate therefore only goes down at a rate of $\mathcal{O}(n^{-1/2})$ in the number of samples. Although standard quadrature techniques converge at a faster rate in one dimension, their performance becomes exponentially worse as the dimensionality of the integrand increases, while Monte Carlo's convergence rate is independent of the dimension, making Monte Carlo the only practical numerical integration algorithm for high-dimensional integrals.

The $\mathcal{O}(n^{-1/2})$ characteristic of Monte Carlo's rate of error reduction is apparent when watching a progressive rendering of a scene where additional samples are incrementally taken in all pixels. The image improves rapidly for the first few samples when doubling the number of samples is relatively little additional work. Later on, once tens or hundreds of samples have been taken, each additional sample doubling takes much longer and remaining error in the image takes a long time to disappear.

The linear decrease in variance with increasing numbers of samples makes it easy to compare different Monte Carlo estimators. Consider two estimators, where the second has half the variance of the first but takes three times as long to compute an estimate; which of the two is better? In that case, the first is preferable: it could take three times as many samples in the time consumed by the second, in which case it would achieve a $3\times$ variance reduction. This concept can be encapsulated in the *efficiency* of an estimator $F$, which is defined as

$$\varepsilon[F]=\frac{1}{\mathrm{V}[F]\,T[F]},$$

where $\mathrm{V}[F]$ is its variance and $T[F]$ is the running time to compute its value.

Not all estimators of integrals have expected values that are equal to the value of the integral. Such estimators are said to be *biased*, where the difference

$$\mathrm{b}[F]=\mathbb{E}[F]-\int f(x)\,dx$$

is the amount of bias. Biased estimators may still be desirable if they are able to get close to the correct result more quickly than unbiased estimators. Kalos and Whitlock (1986, pp. 36–37) give the following example: consider the problem of computing an estimate of the mean value of a uniform distribution $X_i\sim p$ over the interval from 0 to 1. One could use the unbiased sample mean

$$\bar X_n=\frac{1}{n}\sum_{i=1}^{n}X_i,$$

or one could use a deliberately biased estimator that shrinks toward a known constant. The first estimator is unbiased but has variance with order $\mathcal{O}(n^{-1})$. The second estimator's expected value approaches the true mean, so it is biased, although its variance is $\mathcal{O}(n^{-2})$, which is much better. This estimator has the useful property that its error goes to 0 in the limit as the number of samples $n$ goes to infinity; such estimators are *consistent*.[^4] Most of the Monte Carlo estimators used in pbrt are unbiased, with the notable exception of the `SPPMIntegrator`, which implements a photon mapping algorithm.

Closely related to the variance is the *mean squared error* (MSE), which is defined as the expectation of the squared difference of an estimator and the true value,

$$\mathrm{MSE}[F]=\mathbb{E}\!\left[\Big(F-\int f(x)\,dx\Big)^2\right] =\mathrm{V}[F]+\big(\mathrm{b}[F]\big)^2.$$

For an unbiased estimator, MSE is equal to the variance; otherwise it is the sum of variance and the squared bias of the estimator.

It is possible to work out the variance and MSE of some simple estimators in closed form, but for most of the ones of interest in rendering, this is not possible. Yet it is still useful to be able to quantify these values. For this purpose, the sample variance can be computed using a set of independent random variables $X_i$. Equation (2.8) points at one way to compute the sample variance for a set of $n$ random variables $X_i$. If the sample mean is computed as their average, $\bar F=\frac{1}{n}\sum_i X_i$, then the sample variance is

$$s^2=\frac{1}{n-1}\sum_{i=1}^{n}(X_i-\bar F)^2.$$

The division by $n-1$ rather than $n$ is Bessel's correction, and ensures that the sample variance is an unbiased estimate of the variance. (See also Section B.2.11, where a numerically stable approach for computing the sample variance is introduced.)

The sample variance is itself an estimate of the variance, so it has variance itself. Consider, for example, a random variable that has a value of 1 99.99% of the time, and a value of one million 0.01% of the time. If we took ten random samples of it that all had the value 1, the sample variance would suggest that the random variable had zero variance even though its variance is actually much higher.

If an accurate estimate of the integral $\int f(x)\,dx$ can be computed (for example, using a large number of samples), then the mean squared error can be estimated by averaging the squared deviations of individual estimates from that reference,

$$\mathrm{MSE}\;\approx\;\frac{1}{n}\sum_{i=1}^{n}\big(F_i-F_{\mathrm{ref}}\big)^2.$$

The `imgtool` utility program that is provided in pbrt's distribution can compute an image's MSE with respect to a reference image via its `diff` option.

## 2.2 Improving Efficiency

Given an unbiased Monte Carlo estimator, we are in the fortunate position of having a reliable relationship between the number of samples taken and variance (and thus, error). If we have an unacceptably noisy rendered image, increasing the number of samples will reduce error in a predictable way, and—given enough computation—an image of sufficient quality can be generated.

However, computation takes time, and often there is not enough of it. The deadline for a movie may be at hand, or the sixtieth-of-a-second time slice in a real-time renderer may be coming to an end. Given the consequentially limited number of samples, the only option for variance reduction is to find ways to make more of the samples that can be taken. Fortunately, a variety of techniques have been developed to improve the basic Monte Carlo estimator by making the most of the samples that are taken; here we discuss the most important ones that are used in pbrt.

### 2.2.1 Stratified Sampling

A classic and effective family of techniques for variance reduction is based on the careful placement of samples in order to better capture the features of the integrand (or, more accurately, to be less likely to miss important features). These techniques are used extensively in pbrt. Stratified sampling decomposes the integration domain into regions and places samples in each one; here we analyze that approach in terms of its variance reduction properties. Later, in Section 8.2.1, we return with machinery based on Fourier analysis that provides further insights about it.

Stratified sampling subdivides the integration domain $\Lambda$ into $n$ nonoverlapping regions $\Lambda_1,\Lambda_2,\ldots,\Lambda_n$. Each region is called a *stratum*, and they must completely cover the original domain:

$$\Lambda=\bigcup_{i=1}^{n}\Lambda_i,\qquad \Lambda_i\cap\Lambda_j=\varnothing\;\;(i\ne j).$$

To draw samples from $\Lambda$, we draw $n_i$ samples from each $\Lambda_i$, according to densities $p_i$ inside each stratum. A simple example is supersampling a pixel. With stratified sampling, the area around a pixel is divided into a $k\times k$ grid, and a sample is drawn uniformly within each grid cell. This is better than taking $k^2$ random samples, since the sample locations are less likely to clump together. Here we show why this technique reduces variance.

```mermaid
flowchart LR
    A[Integration domain Λ] --> B[Partition into strata Λ_i]
    B --> C[Draw n_i samples per stratum]
    C --> D[Estimate each stratum]
    D --> E[Weight by v_i and sum]
```

Within a single stratum $\Lambda_i$, the Monte Carlo estimate is

$$F_i=\frac{1}{n_i}\sum_{j=1}^{n_i}\frac{f(X_{i,j})}{p_i(X_{i,j})},$$

where $X_{i,j}$ is the $j$th sample drawn from density $p_i$. The overall estimate is $F=\sum_i v_i F_i$, where $v_i$ is the fractional volume of stratum $i$ ($v_i\in(0,1]$).

The true value of the integrand in stratum $i$ is

$$\mu_i=\mathbb{E}[F_i]=\int_{\Lambda_i} f(x)\,p_i(x)\,dx,$$

and the variance in this stratum is

$$\sigma_i^2=\mathrm{V}\!\left[\frac{f(X_{i,j})}{p_i(X_{i,j})}\right].$$

Thus, with $n_i$ samples in the stratum, the variance of the per-stratum estimator is $\sigma_i^2/n_i$. This shows that the variance of the overall estimator is

$$\mathrm{V}[F]=\sum_{i=1}^{n} v_i^2\,\frac{\sigma_i^2}{n_i}.$$

If we make the reasonable assumption that the number of samples $n_i$ is proportional to the volume $v_i$, then we have $n_i=v_i n$, and the variance of the overall estimator is

$$\mathrm{V}[F]=\frac{1}{n}\sum_{i=1}^{n} v_i\,\sigma_i^2.$$

To compare this result to the variance without stratification, we note that choosing an unstratified sample is equivalent to choosing a random stratum $I$ according to the discrete probability distribution defined by the volumes $v_i$ and then choosing a random sample $X$ in $I$. In this sense, $X$ is chosen conditionally on $I$, so it can be shown using conditional probability that[^5]

$$\mathrm{V}[F_{\text{unstrat}}] =\frac{1}{n}\sum_{i=1}^{n} v_i\,\sigma_i^2 +\frac{1}{n}\sum_{i=1}^{n} v_i\,(\mu_i-Q)^2, \quad\text{(2.12)}$$

where $Q=\sum_i v_i\mu_i$ is the mean of $f$ over the whole domain $\Lambda$.

> *Figure 2.1 (caption from the original figure): "Variance is higher and the image noisier (a) when independent random sampling is used than (b) when a stratified distribution of sample directions is used instead. (Bunny model courtesy of the Stanford Computer Graphics Laboratory.)" The extracted page scan contained only this caption—the pictorial renderings were not recoverable—so the comparison is summarized diagrammatically below.*

```mermaid
flowchart TB
    subgraph ind["(a) Independent random sampling"]
        A1[Sample positions can clump and leave gaps] --> A2[Higher variance, noisier image]
    end
    subgraph str["(b) Stratified sampling of sample directions"]
        B1[One or more samples per stratum] --> B2[Lower variance, cleaner image]
    end
```

There are two things to notice about Equation (2.12). First, we know that the right-hand sum must be nonnegative, since variance is always nonnegative. Second, it demonstrates that stratified sampling can never increase variance. Stratification always reduces variance unless the right-hand sum is exactly 0. It can only be 0 when the function $f$ has the same mean over each stratum $\Lambda_i$. For stratified sampling to work best, we would like to maximize the right-hand sum, so it is best to make the strata have means that are as unequal as possible. This explains why compact strata are desirable if one does not know anything about the function $f$. If the strata are wide, they will contain more variation and will have $\mu_i$ closer to the true mean $Q$.

Figure 2.1 shows the effect of using stratified sampling versus an independent random distribution for sampling when rendering an image that includes glossy reflection. There is a reasonable reduction in variance at essentially no cost in running time.

The main downside of stratified sampling is that it suffers from the same "curse of dimensionality" as standard numerical quadrature. Full stratification in $D$ dimensions with $S$ strata per dimension requires $S^D$ samples, which quickly becomes prohibitive. Fortunately, it is often possible to stratify some of the dimensions independently and then randomly associate samples from different dimensions; this approach will be used in Section 8.5. Choosing which dimensions are stratified should be done in a way that stratifies dimensions that tend to be most highly correlated in their effect on the value of the integrand (Owen 1998).

### 2.2.2 Importance Sampling

Importance sampling is a powerful variance reduction technique that exploits the fact that the Monte Carlo estimator

$$F_n=\frac{1}{n}\sum_{i=1}^{n}\frac{f(X_i)}{p(X_i)}$$

converges more quickly if the samples are taken from a distribution $p(x)$ that is similar to the function $f(x)$ in the integrand. In this case, samples are more likely to be taken when the magnitude of the integrand is relatively large. Importance sampling is one of the most frequently used variance reduction techniques in rendering, since it is easy to apply and is very effective when good sampling distributions are used.

To see why such sampling distributions reduce error, first consider the effect of using a distribution $p(x)\propto f(x)$, or $p(x)=c\,f(x)$.[^6] It is trivial to show that normalization of the PDF requires that

$$c=\frac{1}{\displaystyle\int f(x)\,dx}.$$

Finding such a PDF requires that we know the value of the integral, which is what we were trying to estimate in the first place. Nonetheless, if we could sample from this distribution, each term of the sum in the estimator would have the value

$$\frac{f(X_i)}{p(X_i)}=\frac{1}{c}=\int f(x)\,dx.$$

The variance of the estimator is zero! Of course, this is ludicrous since we would not bother using Monte Carlo if we could integrate $f$ directly. However, if a density $p(x)$ can be found that is similar in shape to $f(x)$, variance is reduced.

As a more realistic example, consider a narrow Gaussian function

$$f(x)\;\propto\;\exp\!\left(-\frac{(x-1/4)^2}{2\sigma^2}\right),\qquad \sigma\ \text{small},$$

plotted in Figure 2.2(a) over $[0,1]$. Its value is close to zero over most of the domain. Samples $X$ with $X<0.2$ or $X>0.3$ are of little help in estimating the value of the integral since they give no information about the magnitude of the bump in the function's value around $1/4$. With uniform sampling and the basic Monte Carlo estimator of Equation (2.6), variance is approximately $0.0365$.

If samples are instead drawn from a piecewise-constant PDF that concentrates probability near the bump—schematically,

$$p(x)= \begin{cases} p_{\text{low}}, & x\in[0,0.15)\cup[0.35,1],\\ p_{\text{high}}, & x\in[0.15,0.35), \end{cases} \qquad 0.8\,p_{\text{low}}+0.2\,p_{\text{high}}=1,$$

which is plotted in Figure 2.2(b), and the estimator from Equation (2.7) is used instead, then variance is reduced by a factor of approximately $6.7\times$. A representative set of 6 points from this distribution is shown in Figure 2.2(c); we can see that most of the evaluations of $f(x)$ are in the interesting region where it is not nearly zero.

> *Figure 2.2 (original figure, transcribed; no page screenshot kept): (a) A narrow Gaussian function that is close to zero over most of the range $[0,1]$. The basic Monte Carlo estimator of Equation (2.6) has relatively high variance if it is used to integrate this function, since most samples have values that are close to zero. (b) A PDF that roughly approximates the function's distribution. If this PDF is used to generate samples, variance is reduced substantially. (c) A representative distribution of samples generated according to (b).*

```mermaid
flowchart LR
    A["(a) f(x): narrow bump near x = 1/4"] --> B["(b) p(x): mass concentrated near the bump"]
    B --> C["(c) samples land mostly where f(x) is large"]
    C --> D[Lower estimator variance]
```

Importance sampling can increase variance if a poorly chosen distribution is used, however. Consider instead using the distribution (recovered from the original printed page)

$$p(x)= \begin{cases} 1.2, & x\in[0,0.15),\\ 0.2, & x\in[0.15,0.35),\\ 1.2, & x\in[0.35,1], \end{cases}$$

for estimating the integral of the Gaussian function. This PDF increases the probability of sampling the function where its value is close to zero and decreases the probability of sampling it where its magnitude is larger. (The three pieces are normalized: $0.15(1.2)+0.20(0.2)+0.65(1.2)=1$.)

Not only does this PDF generate fewer samples where the integrand is large, but when it does, the magnitude of $f(x)/p(x)$ in the Monte Carlo estimator will be especially high since $p(x)=0.2$ in that region. The result is approximately $5.4\times$ higher variance than uniform sampling, and nearly $36\times$ higher variance than the better PDF above. In the context of Monte Carlo integration for rendering where evaluating the integrand generally involves the expense of tracing a ray, it is desirable to minimize the number of samples taken; using an inferior sampling distribution and making up for it by evaluating more samples is an unappealing option.

### 2.2.3 Multiple Importance Sampling

We are frequently faced with integrals that are the product of two or more functions, $\int f_a(x)f_b(x)\,dx$. It is often possible to derive separate importance sampling strategies for the individual factors, though not one that is similar to their product. This situation is especially common in the integrals involved with light transport, such as the product of BSDF, incident radiance, and a cosine factor in the light transport equation (1.1).

To understand the challenges involved with applying Monte Carlo to such products, assume for now the good fortune of having two sampling distributions $p_a$ and $p_b$ that match the distributions of $f_a$ and $f_b$ exactly. (In practice, this will not normally be the case.) With the Monte Carlo estimator of Equation (2.7), we have two options: we might draw samples using $p_a$, which gives the estimator

$$F=\frac{1}{n}\sum_{i=1}^{n}\frac{f_a(X_i)\,f_b(X_i)}{p_a(X_i)} =\frac{1}{n}\sum_{i=1}^{n} c\,f_b(X_i),$$

where $c$ is a constant equal to the integral of $f_a$, since $p_a(x)\propto f_a(x)$. The variance of this estimator is proportional to the variance of $f_b$, which may itself be high.[^7] Conversely, we might sample from $p_b$, though doing so gives us an estimator with variance proportional to the variance of $f_a$, which may similarly be high. In the more common case where the sampling distributions only approximately match one of the factors, the situation is usually even worse.

Unfortunately, the obvious solution of taking some samples from each distribution and averaging the two estimators is not much better. Because variance is additive, once variance has crept into an estimator, we cannot eliminate it by adding it to another low-variance estimator.

Multiple importance sampling (MIS) addresses exactly this issue, with an easy-to-implement variance reduction technique. The basic idea is that, when estimating an integral, we should draw samples from multiple sampling distributions, chosen in the hope that at least one of them will match the shape of the integrand reasonably well, even if we do not know which one this will be. MIS then provides a method to weight the samples from each technique that can eliminate large variance spikes due to mismatches between the integrand's value and the sampling density. Specialized sampling routines that only account for unusual special cases are even encouraged, as they reduce variance when those cases occur, with relatively little cost in general.

```mermaid
flowchart LR
    F[Integrand f = fa · fb] --> PA[Sample from pa, a good match to fa]
    F --> PB[Sample from pb, a good match to fb]
    PA --> W[Weight samples with wa, wb]
    PB --> W
    W --> S[Sum weighted contributions]
```

With two sampling distributions $p_a$ and $p_b$ and a single sample taken from each one, $X\sim p_a$ and $Y\sim p_b$, the MIS Monte Carlo estimator is

$$F=\frac{w_a(X)\,f(X)}{p_a(X)}+\frac{w_b(Y)\,f(Y)}{p_b(Y)}, \quad\text{(2.13)}$$

where $w_a$ and $w_b$ are weighting functions chosen such that the expected value of this estimator is the value of the integral of $f(x)$.

More generally, given $n$ sampling distributions $p_i$ with $n_i$ samples $X_{i,j}$ taken from the $i$th distribution, the MIS Monte Carlo estimator is

$$F=\sum_{i=1}^{n}\frac{1}{n_i}\sum_{j=1}^{n_i}\frac{w_i(X_{i,j})\,f(X_{i,j})}{p_i(X_{i,j})}.$$

(The full set of conditions on the weighting functions for the estimator to be unbiased are that they sum to 1 when $f(x)\ne 0$, $\sum_i w_i(x)=1$, and that $w_i(x)=0$ if $p_i(x)=0$.)

Setting $w_i(X)=1/n$ corresponds to the case of summing the various estimators, which we have already seen is an ineffective way to reduce variance. It would be better if the weighting functions were relatively large when the corresponding sampling technique was a good match to the integrand and relatively small when it was not, thus reducing the contribution of high-variance samples.

In practice, a good choice for the weighting functions is given by the *balance heuristic*, which attempts to fulfill this goal by taking into account all the different ways that a sample could have been generated, rather than just the particular one that was used to do so. The balance heuristic's weighting function for the $i$th sampling technique is

$$w_i(x)=\frac{n_i\,p_i(x)}{\sum_j n_j\,p_j(x)}. \quad\text{(2.14)}$$

With the balance heuristic and our example of taking a single sample from each of two sampling techniques, the estimator of Equation (2.13) works out to be

$$F=\frac{f(X)}{p_a(X)+p_b(X)}+\frac{f(Y)}{p_a(Y)+p_b(Y)}.$$

Each evaluation of $f$ is divided by the sum of all PDFs for the corresponding sample rather than just the one that generated the sample. Thus, if $p_a$ generates a sample with low probability at a point where $p_b$ has a higher probability, then dividing by $p_a(X)+p_b(X)$ reduces the sample's contribution. Effectively, such samples are downweighted when sampled from $p_a$, recognizing that the sampling technique associated with $p_b$ is more effective at the corresponding point in the integration domain. As long as just one of the sampling techniques has a reasonable probability of sampling a point where the function's value is large, the MIS weights can lead to a significant reduction in variance.

`BalanceHeuristic()` computes Equation (2.14) for the specific case of two distributions $p_a$ and $p_b$. We will not need a more general multidistribution case in pbrt. The sampling routines below share a small set of helpers; in the wgpu path `Float` is `f32` so that the same code maps cleanly onto WGSL's `f32`.

```rust
// Shared helpers used by all sampling routines in this chapter.
/// Scalar type used by the renderer (32-bit float in the wgpu backend).
pub type Float = f32;

/// Largest representable Float strictly less than 1.0.
pub const ONE_MINUS_EPSILON: Float = 1.0 - f32::EPSILON / 2.0;

#[inline]
pub fn sqr(v: Float) -> Float { v * v }

/// Next representable Float below `v` (valid for the positive, finite
/// values produced by samplers). Mirrors pbrt's `NextFloatDown()`.
#[inline]
pub fn next_float_down(v: Float) -> Float {
    Float::from_bits(v.to_bits() - 1)
}
```

```rust
// Fragment: Sampling Inline Functions
/// Balance heuristic (β = 1) for two sampling techniques.
#[inline]
pub fn balance_heuristic(nf: i32, f_pdf: Float, ng: i32, g_pdf: Float) -> Float {
    let f = nf as Float * f_pdf;
    let g = ng as Float * g_pdf;
    let sum = f + g;
    if sum == 0.0 { 0.0 } else { f / sum }
}
```

> *Printed-edition index: `Float` (p. 23).*

In practice, the power heuristic often reduces variance even further. For an exponent $\beta$, the power heuristic is

$$w_i(x)=\frac{\big(n_i\,p_i(x)\big)^\beta}{\sum_j \big(n_j\,p_j(x)\big)^\beta}.$$

Note that the power heuristic has a similar form to the balance heuristic, though it further reduces the contribution of relatively low probabilities. Our implementation has $\beta=2$ hard-coded; that parameter value usually works well in practice.

```rust
// Fragment: Sampling Inline Functions (continued)
/// Power heuristic with β = 2, the default used throughout the renderer.
#[inline]
pub fn power_heuristic(nf: i32, f_pdf: Float, ng: i32, g_pdf: Float) -> Float {
    let f = nf as Float * f_pdf;
    let g = ng as Float * g_pdf;
    let f2 = sqr(f);
    let g2 = sqr(g);
    let sum = f2 + g2;
    if sum == 0.0 { 0.0 } else { f2 / sum }
}
```

> *Printed-edition index: `Float` (p. 23), `Sqr()` (p. 1034).*

The same heuristics map directly to WGSL compute shaders, where each ray/pixel thread evaluates a single sample:

```wgsl
fn balance_heuristic(nf: f32, f_pdf: f32, ng: f32, g_pdf: f32) -> f32 {
    let f = nf * f_pdf;
    let g = ng * g_pdf;
    let sum = f + g;
    return select(0.0, f / sum, sum > 0.0);
}

fn power_heuristic(nf: f32, f_pdf: f32, ng: f32, g_pdf: f32) -> f32 {
    let f = nf * f_pdf;
    let g = ng * g_pdf;
    let f2 = f * f;
    let g2 = g * g;
    let sum = f2 + g2;
    return select(0.0, f2 / sum, sum > 0.0);
}
```

Multiple importance sampling can be applied even without sampling from all the distributions. This approach is known as the *single sample model*. We will not include the derivation here, but it can be shown that given an integrand $f(x)$, if a sampling technique $p_i$ is chosen from a set of techniques with probability $q_i$ and a sample $X$ is drawn from $p_i$, then the single sample estimator

$$F=\frac{w_i(X)\,f(X)}{p_i(X)},\qquad \text{with}\quad \sum_i q_i\,w_i(x)=1\ \text{wherever } f(x)\ne 0,$$

gives an unbiased estimate of the integral. For the single sample model, the balance heuristic is provably optimal.

One shortcoming of multiple importance sampling is that if one of the sampling techniques is a very good match to the integrand, MIS can slightly increase variance. For rendering applications, MIS is almost always worthwhile for the variance reduction it provides in cases that can otherwise have high variance.

#### MIS Compensation

Multiple importance sampling is generally applied using probability distributions that are all individually valid for importance sampling the integrand, with nonzero probability of generating a sample anywhere that the integrand is nonzero. However, when MIS is being used, it is not a requirement that all PDFs are nonzero where the function's value is nonzero; only one of them must be.

This observation led to the development of a technique called *MIS compensation*, which can further reduce variance. It is motivated by the fact that if all the sampling distributions allocate some probability to sampling regions where the integrand's value is small, it is often the case that that region of the integrand ends up being oversampled, leaving the region where the integrand is high undersampled.

MIS compensation is based on the idea of sharpening one or more (but not all) the probability distributions—for example, by adjusting them to have zero probability in areas where they earlier had low probability. A new sampling distribution $p'$ can, for example, be defined by clipping and renormalizing with a fixed value $\delta$,

$$p'(x)=\frac{\max\{p(x)-\delta,\,0\}}{\displaystyle\int \max\{p(x)-\delta,\,0\}\,dx},$$

for some fixed value $\delta$.

This technique is especially easy to apply in the case of tabularized sampling distributions. In Section 12.5, it is used to good effect for sampling environment map light sources.

### 2.2.4 Russian Roulette

Russian roulette is a technique that can improve the efficiency of Monte Carlo estimates by skipping the evaluation of samples that would make a small contribution to the final result. In rendering, we often have estimators of the form

$$F_n=\frac{1}{n}\sum_{i=1}^{n}\frac{f(X_i)\,v(X_i)}{p(X_i)},$$

where the integrand consists of some factors $f(X)$ that are easily evaluated (e.g., those that relate to how the surface scatters light) and others that are more expensive to evaluate, such as a binary visibility factor $v(X)$ that requires tracing a ray. In these cases, most of the computational expense of evaluating the estimator lies in $v$.

If $f(X)$ is zero, it is obviously worth skipping the work of evaluating $v(X)$, since its value will not affect the value of the estimator. However, if we also skipped evaluating estimators where $f(X)$ was small but nonzero, then we would introduce bias into the estimator and would systematically underestimate the value of the integrand. Russian roulette solves this problem, making it possible to also skip tracing rays when $f(X)$'s value is small but not necessarily 0, while still computing the correct value on average.

To apply Russian roulette, we select some termination probability $q$. This value can be chosen in almost any manner; for example, it could be based on an estimate of the value of the integrand for the particular sample chosen, increasing as the integrand's value becomes smaller. With probability $q$, the estimator is not evaluated for the particular sample, and some constant value $c$ is used in its place ($c=0$ is often used). With probability $1-q$, the estimator is still evaluated but is weighted by the factor $1/(1-q)$, which effectively compensates for the samples that were skipped.

```mermaid
flowchart TD
    S[Sample with cheap factor f X] --> Q{Draw ξ uniform: ξ < q?}
    Q -- yes --> C[Terminate: substitute constant c, e.g. 0]
    Q -- no --> K[Evaluate expensive factor v X, trace ray]
    K --> W[Weight contribution by 1 / 1-q]
```

We have the new estimator

$$F'= \begin{cases} c, & \text{with probability } q,\\[2pt] \dfrac{f(X)\,v(X)}{(1-q)\,p(X)}, & \text{with probability } 1-q. \end{cases}$$

It is easy to see that its expected value is the same as the expected value of the original estimator:

$$\mathbb{E}[F']=q\,c+(1-q)\,\mathbb{E}\!\left[\frac{f(X)\,v(X)}{(1-q)\,p(X)}\right] =q\,c+\mathbb{E}[F],$$

which equals $\mathbb{E}[F]$ when $c=0$.

Russian roulette never reduces variance. In fact, unless somehow $c=F$, it will always increase variance. However, it does improve Monte Carlo efficiency if the probabilities are chosen so that samples that are likely to make a small contribution to the final result are skipped.

### 2.2.5 Splitting

While Russian roulette reduces the number of samples, splitting increases the number of samples in some dimensions of multidimensional integrals in order to improve efficiency. As an example, consider an integral of the general form

$$\int_A\int_B f(x,y)\,dy\,dx. \quad\text{(2.17)}$$

With the standard importance sampling estimator, we might draw $n$ samples from independent distributions, $X_i\sim p_x$ and $Y_i\sim p_y$, and compute

$$F_n=\frac{1}{n}\sum_{i=1}^{n}\frac{f(X_i,Y_i)}{p_x(X_i)\,p_y(Y_i)}. \quad\text{(2.18)}$$

Splitting allows us to formalize the idea of taking more than one sample for the integral over $B$ for each sample taken in $A$. With splitting, we might take $m$ samples $Y_{i,j}$ for each sample $X_i$, giving the estimator

$$F_n=\frac{1}{n}\sum_{i=1}^{n}\frac{1}{p_x(X_i)} \left(\frac{1}{m}\sum_{j=1}^{m}\frac{f(X_i,Y_{i,j})}{p_y(Y_{i,j})}\right).$$

If it is possible to partially evaluate $f(X_i,\cdot)$ for each $X_i$, then we can compute a total of $nm$ samples more efficiently than we had taken $nm$ independent $X_i$ values using Equation (2.18).

For an example from rendering, an integral of the form of Equation (2.17) is evaluated to compute the color of pixels in an image: an integral is taken over the area of the pixel $A$ where at each point in the pixel $x$, a ray is traced into the scene and the reflected radiance at the intersection point is computed using an integral over the hemisphere (denoted here by $B$) for which one or more rays are traced. With splitting, we can take multiple samples for each lighting integral, improving efficiency by amortizing the cost of tracing the initial ray from the camera over them.

## 2.3 Sampling Using the Inversion Method

To evaluate the Monte Carlo estimator in Equation (2.7), it is necessary to be able to draw random samples from a chosen probability distribution. There are a variety of techniques for doing so, but one of the most important for rendering is the *inversion method*, which maps uniform samples from $[0,1)$ to a given 1D probability distribution by inverting the distribution's CDF. (In Section 2.4.2 we will see how this approach can be applied to higher-dimensional functions by considering a sequence of 1D distributions.) When used with well-distributed samples such as those generated by the samplers that are defined in Chapter 8, the inversion method can be particularly effective. Throughout the remainder of the book, we will see the application of the inversion method to generate samples from the distributions defined by BSDFs, light sources, cameras, and scattering media.

```mermaid
flowchart LR
    U[Canonical uniform ξ ∈ 0,1) --> INV[Invert the CDF: X = P⁻¹ ξ]
    INV --> OUT[Sample X drawn from p x]
```

### 2.3.1 Discrete Case

Equation (2.2) leads to an algorithm for sampling from a set of discrete probabilities using a uniform random variable. Suppose we have a process with four possible outcomes where the probabilities of each of the four outcomes are given by $p_1,p_2,p_3,p_4$, with $\sum_i p_i=1$. The corresponding PMF is shown in Figure 2.3.

| Outcome | 1 | 2 | 3 | 4 |
|:---:|:---:|:---:|:---:|:---:|
| Probability | $p_1$ | $p_2$ | $p_3$ | $p_4$ |

> *Figure 2.3 (original caption): "A PMF for Four Events, Each with a Probability $p_i$. The sum of their probabilities $\sum_i p_i$ is necessarily 1." Bar heights are shown schematically below; the original page scan contained only a broken image and was removed.*

```mermaid
xychart-beta
    title "Schematic PMF for four outcomes (illustrative heights, ∑pᵢ = 1)"
    x-axis ["p1", "p2", "p3", "p4"]
    y-axis "probability" 0 --> 0.5
    bar [0.10, 0.20, 0.30, 0.40]
```

There is a direct connection between the sums in Equation (2.2) and the definition of the CDF. The discrete CDF is given by

$$P(i)=\sum_{j=1}^{i} p_j,$$

which can be interpreted graphically by stacking the bars of the PMF on top of each other, starting at the left. This idea is shown in Figure 2.4.

> *Figure 2.4 (original caption): "A Discrete CDF, Corresponding to the PMF in Figure 2.3. Each column's height is given by the PMF for the event that it represents plus the sum of the PMFs for the previous events, $P(i)=\sum_{j\le i}p_j$." The original page scan contained only a broken image and was removed; the same construction is shown diagrammatically.*

```mermaid
flowchart LR
    Z[0] -->|p1| A[p1]
    A -->|p2| B[p1 + p2]
    B -->|p3| C[p1 + p2 + p3]
    C -->|p4| D[1]
```

The sampling operation of Equation (2.2) can be expressed as finding $i$ such that

$$\sum_{j=1}^{i-1} p_j \;\le\; \xi \;<\;\sum_{j=1}^{i} p_j. \quad\text{(2.19)}$$

This can be interpreted as inverting the CDF $P$, and thus, the name of the technique. Continuing the graphical interpretation, this sampling operation can be considered in terms of projecting the events' probabilities onto the vertical axis where they cover the range $[0,1]$ and using a random variable $\xi$ to select among them (see Figure 2.5). It should be clear that this draws from the correct distribution—the probability of the uniform sample hitting any particular bar is exactly equal to the height of that bar.

The `SampleDiscrete()` function implements this algorithm. It takes a not-necessarily normalized set of nonnegative weights, a uniform random sample $u$, and returns the index of one of the weights with probability proportional to its weight. The sampling operation it performs corresponds to finding $i$ such that

$$\sum_{j=1}^{i-1} w_j \;\le\; u\sum_j w_j \;<\;\sum_{j=1}^{i} w_j, \quad\text{(2.20)}$$

which corresponds to multiplying Equation (2.19) by $\sum_j w_j$. (Not requiring a normalized PMF is a convenience for calling code and not much more work in the function's implementation.) Two optional parameters are provided to return the value of the PMF for the sample as well as a new uniform random sample that is derived from $u$.

This function is designed for the case where only a single sample needs to be generated from the weights' distribution; if multiple samples are required, the `AliasTable`, which will be introduced in Section A.1, should generally be used instead: it generates samples in $\mathcal{O}(1)$ time after an $\mathcal{O}(n)$ preprocessing step, whereas `SampleDiscrete()` requires $\mathcal{O}(n)$ time for each sample generated.

The literate-programming fragment below assembles the following steps; the printed edition lists them as 〈Handle empty weights for discrete sampling〉 (p. 71), 〈Compute sum of weights〉 (p. 71), 〈Compute rescaled u sample〉 (p. 71), 〈Find offset in weights corresponding to u′〉 (p. 71), and 〈Compute PMF and remapped u value, if necessary〉 (p. 71).

```rust
// Fragment: Sampling Inline Functions (p. 70)
/// Sample a discrete distribution from unnormalized weights.
/// Returns the selected index (or -1 for empty weights) and optionally
/// the PMF of the sample and a remapped uniform value.
pub fn sample_discrete(
    weights: &[Float],
    u: Float,
    mut pmf: Option<&mut Float>,
    mut u_remapped: Option<&mut Float>,
) -> i32 {
    // Handle empty weights for discrete sampling.
    if weights.is_empty() {
        if let Some(p) = pmf.as_deref_mut() { *p = 0.0; }
        return -1;
    }

    // Compute sum of weights.
    let sum_weights: Float = weights.iter().copied().sum();

    // Compute rescaled u sample (Equation 2.20).
    let mut up = u * sum_weights;
    if up == sum_weights {
        up = next_float_down(up);
    }

    // Find offset in weights corresponding to u′.
    let mut offset = 0usize;
    let mut sum: Float = 0.0;
    while sum + weights[offset] <= up {
        sum += weights[offset];
        offset += 1;
    }

    // Compute PMF and remapped u value, if necessary.
    if let Some(p) = pmf.as_deref_mut() {
        *p = weights[offset] / sum_weights;
    }
    if let Some(ur) = u_remapped.as_deref_mut() {
        *ur = ((up - sum) / weights[offset]).min(ONE_MINUS_EPSILON);
    }

    offset as i32
}
```

> *Printed-edition index: `AliasTable` (p. 994), `Float` (p. 23).*

The case of weights being empty is handled first so that subsequent code can assume that there is at least one weight.

The discrete probability of sampling the $i$th element is given by `weights[i]` divided by the sum of all weight values. Therefore, the function computes that sum next.

Following Equation (2.20), the uniform sample $u$ is scaled by the sum of the weights to get a value $u'$ that will be used to sample from them. Even though the provided $u$ value should be in the range $[0,1)$, it is possible that $u\sum w_i$ will be equal to $\sum w_i$ due to floating-point round-off. In that rare case, $u'$ is bumped down to the next lower floating-point value so that subsequent code can assume that $u'<\sum w_i$.

We would now like to find the last offset in the weights array $i$ where the random sample $u'$ is greater than the sum of weights up to $i$. Sampling is performed using a linear search from the start of the array, accumulating a sum of weights until the sum would be greater than $u'$.

After the while loop terminates, the randomness in the provided sample $u$ has only been used to select an element of the array—a discrete choice. The offset of a sample between the CDF values that bracket it is itself a uniform random value that can easily be remapped to $[0,1)$. This value is returned to the caller in `u_remapped`, if requested.

One might ask: why bother? It is not too difficult to generate uniform random variables, so the benefit of providing this option may seem marginal. However, for some of the high-quality sample generation algorithms in Chapter 8, it can be beneficial to reuse samples in this way rather than generating new ones—thus, this option is provided.

A WGSL version for GPU-side tabular sampling (e.g. environment-light CDFs in a storage buffer) follows the same logic. WGSL has no "next float down" builtin, so we bias $u'$ inward by one ULP of the sum:

```wgsl
struct DiscreteSample {
    index: i32,
    pmf: f32,
    u_remapped: f32,
};

fn sample_discrete(weights: ptr<storage, array<f32>, read>,
                   n: u32, u: f32) -> DiscreteSample {
    var sum_weights = 0.0;
    for (var i = 0u; i < n; i = i + 1u) {
        sum_weights += (*weights)[i];
    }
    var up = u * sum_weights;
    if (up == sum_weights) {
        up = up - max(sum_weights * 1.1920929e-7, 1.1920929e-7);
    }
    var offset = 0u;
    var sum = 0.0;
    while (sum + (*weights)[offset] <= up) {
        sum += (*weights)[offset];
        offset = offset + 1u;
    }
    var result: DiscreteSample;
    result.index = i32(offset);
    result.pmf = (*weights)[offset] / sum_weights;
    result.u_remapped = min((up - sum) / (*weights)[offset], 0.99999994);
    return result;
}
```

> *Printed-edition index: `Float` (p. 23), `NextFloatDown()` (p. 366), `OneMinusEpsilon` (p. 470).*

> *Figure 2.5 (original caption): "To use the inversion method to draw a sample from the distribution described by the PMF in Figure 2.3, a canonical uniform random variable is plotted on the vertical axis. By construction, the horizontal extension of $\xi$ will intersect the box representing the $i$th outcome with probability $p_i$. If the corresponding event is chosen for a set of random variables $\xi$, then the resulting distribution of events will be distributed according to the PMF." The original page scan contained only a caption box, so the construction is redrawn as a diagram.*

```mermaid
flowchart LR
    XI[Draw ξ uniform on 0,1] --> AXIS[Plot ξ on the vertical axis]
    AXIS --> HIT[Extend horizontally to the stacked PMF bars]
    HIT --> PICK[Intersect bar i with probability p_i]
    PICK --> OUT[Outcome i ~ PMF]
```

### 2.3.2 Continuous Case

In order to generalize this technique to continuous distributions, consider what happens as the number of discrete possibilities approaches infinity. The PMF from Figure 2.3 becomes a PDF, and the CDF from Figure 2.4 becomes its integral. The projection process is still the same, but it has a convenient mathematical interpretation—it represents inverting the CDF and evaluating the inverse at $\xi$.

More precisely, we can draw a sample $X_i$ from a PDF $p(x)$ with the following steps:[^8]

1. Integrate the PDF to find the CDF, $P(x)=\int_{-\infty}^{x}p(t)\,dt$.
2. Obtain a uniformly distributed random number $\xi$.
3. Generate a sample by solving $\xi=P(X)$ for $X$; in other words, find $X=P^{-1}(\xi)$.

We will illustrate this algorithm with a simple example; see Section A.4 for its application to a number of additional functions.

**Sampling a Linear Function.** The function $f(x)=(1-x)a+xb$ defined over $[0,1]$ linearly interpolates between $a$ at $x=0$ and $b$ at $x=1$. Here we assume that $a,b\ge 0$; an exercise at the end of the chapter discusses the more general case.

```rust
// Fragment: Math Inline Functions
/// Linear interpolation between a (at x = 0) and b (at x = 1).
#[inline]
pub fn lerp(x: Float, a: Float, b: Float) -> Float {
    (1.0 - x) * a + x * b
}
```

The function's integral is $\int_0^1 f(x)\,dx=(a+b)/2$, which gives the normalization constant $2/(a+b)$ to define its PDF,

$$p(x)=\frac{2\big((1-x)a+xb\big)}{a+b}=\frac{2\,\operatorname{lerp}(x,a,b)}{a+b}.$$

> *Printed-edition index: `Float` (p. 23).*

```rust
// Fragment: Sampling Inline Functions
/// PDF of the linear distribution (zero outside [0, 1]).
#[inline]
pub fn linear_pdf(x: Float, a: Float, b: Float) -> Float {
    if x < 0.0 || x > 1.0 {
        return 0.0;
    }
    2.0 * lerp(x, a, b) / (a + b)
}
```

Integrating the PDF gives the CDF, which is the quadratic function

$$P(x)=\int_0^x \frac{2\big((1-t)a+tb\big)}{a+b}\,dt =\frac{2ax+(b-a)x^2}{a+b} =\frac{x\big(a(2-x)+bx\big)}{a+b}.$$

Inverting $\xi=P(X)$ gives a sampling recipe

$$X=\frac{-a+\sqrt{a^2+(b^2-a^2)\,\xi}}{b-a},$$

though note that in this form, the case $a=b$ gives an indeterminate result. The more stable formulation (Muller's method, as applied by Heitz 2020)

$$X=\frac{\xi\,(a+b)}{a+\sqrt{(1-\xi)\,a^2+\xi\,b^2}}$$

computes the same result and is implemented here.

```rust
// Fragment: Sampling Inline Functions (continued)
/// Sample the linear distribution using the numerically stable form.
#[inline]
pub fn sample_linear(u: Float, a: Float, b: Float) -> Float {
    if u == 0.0 && a == 0.0 {
        return 0.0;
    }
    let root = lerp(u, sqr(a), sqr(b)).sqrt();
    let x = u * (a + b) / (a + root);
    x.min(ONE_MINUS_EPSILON)
}
```

```wgsl
fn sample_linear(u: f32, a: f32, b: f32) -> f32 {
    if (u == 0.0 && a == 0.0) { return 0.0; }
    let root = sqrt(mix(a * a, b * b, u));
    return min(u * (a + b) / (a + root), 0.99999994);
}
```

One detail to note is the `min` call in the return statement, which ensures that the returned value is within the range $[0,1)$. Although the sampling algorithm generates values in that range given $\xi\in[0,1)$, round-off error may cause the result to be equal to 1. Because some of the code that calls the sampling routines depends on the returned values being in the specified range, the sampling routines must ensure this is so.

In addition to providing functions that sample from a distribution and compute the PDF of a sample, pbrt usually also provides functions that invert sampling operations, returning the random sample $\xi$ that corresponds to a value $x$. In the 1D case, this is equivalent to evaluating the CDF.

```rust
// Fragment: Sampling Inline Functions (continued)
/// Invert sample_linear: return the u that produces x (i.e. evaluate the CDF).
#[inline]
pub fn invert_linear_sample(x: Float, a: Float, b: Float) -> Float {
    x * (a * (2.0 - x) + b * x) / (a + b)
}
```

> *Printed-edition index: `Float` (p. 23), `Lerp()` (p. 72), `OneMinusEpsilon` (p. 470), `Sqr()` (p. 1034).*

## 2.4 Transforming Between Distributions

In describing the inversion method, we introduced a technique that generates samples according to some distribution by transforming canonical uniform random variables in a particular manner. Here, we investigate the more general question of which distribution results when we transform samples from an arbitrary distribution to some other distribution with a function $f$. Understanding the effect of such transformations is useful for a few reasons, though here we focus on how they allow us to derive multidimensional sampling algorithms.

Suppose we are given a random variable $X$ drawn from some PDF $p(x)$ with CDF $P(x)$. Given a function $f(x)$ with $y=f(x)$, if we compute $Y=f(X)$, we would like to find the distribution of the new random variable $Y$. In this case, the function $f(x)$ must be a one-to-one transformation; if multiple values of $x$ mapped to the same $y$ value, then it would be impossible to unambiguously describe the probability density of a particular $y$ value. A direct consequence of $f$ being one-to-one is that its derivative must either be strictly greater than 0 or strictly less than 0, which implies that for a given $x$,

$$\Pr\{Y\le f(x)\}=\Pr\{X\le x\}.$$

From the definition of the CDF, Equation (2.3), we can see that

$$P_f(y)=P_f(f(x))=P(x).$$

This relationship between CDFs leads directly to the relationship between their PDFs. If we assume that $f$'s derivative is greater than 0, differentiating gives

$$p_f(f(x))\,f'(x)=p(x),$$

and so

$$p_f(y)=\frac{p(x)}{f'(x)}\quad\text{with } x=f^{-1}(y).$$

In general, $f$'s derivative is either strictly positive or strictly negative, and the relationship between the densities is

$$p_y(y)=\frac{p_x(x)}{\left|\dfrac{dy}{dx}\right|}\qquad\big(x=f^{-1}(y)\big).$$

How can we use this formula? Suppose that $p(x)=2x$ over the domain $[0,1]$, and let $f(x)=\sin x$. What is the PDF of the random variable $Y=f(X)$? Because we know that $df/dx=\cos x$,

$$p_y(y)=\frac{2x}{\cos x}\bigg|_{x=\arcsin y} =\frac{2\arcsin y}{\sqrt{1-y^2}}.$$

This procedure may seem backward—usually we have some PDF that we want to sample from, not a given transformation. For example, we might have $X$ drawn from some $p(x)$ and would like to compute $Y$ from some distribution $p_f(y)$. What transformation should we use? All we need is for the CDFs to be equal, or $P_f(y)=P(x)$, which immediately gives the transformation

$$y=f(x)=P_f^{-1}\big(P(x)\big).$$

This is a generalization of the inversion method, since if $X$ were uniformly distributed over $[0,1)$ then $P(x)=x$, and we have the same procedure as was introduced previously.

### 2.4.1 Transformation in Multiple Dimensions

In the general $d$-dimensional case, a similar derivation gives the analogous relationship between different densities. We will not show the derivation here; it follows the same form as the 1D case. Suppose we have a $d$-dimensional random variable $\mathbf{X}$ with density function $p_x(\mathbf{x})$. Now let $\mathbf{Y}=T(\mathbf{X})$, where $T$ is a bijection. In this case, the densities are related by

$$p_y(\mathbf{y})=\frac{p_x(\mathbf{x})}{\left|J_T(\mathbf{x})\right|}, \quad\text{(2.21)}$$

where $|J_T|$ is the absolute value of the determinant of $T$'s Jacobian matrix, which is

$$J_T(\mathbf{x})= \begin{pmatrix} \partial T_1/\partial x_1 & \partial T_1/\partial x_2 & \cdots & \partial T_1/\partial x_d\\ \partial T_2/\partial x_1 & \partial T_2/\partial x_2 & \cdots & \partial T_2/\partial x_d\\ \vdots & \vdots & \ddots & \vdots\\ \partial T_d/\partial x_1 & \partial T_d/\partial x_2 & \cdots & \partial T_d/\partial x_d \end{pmatrix},$$

where subscripts index dimensions of $T(\mathbf{x})$ and $\mathbf{x}$.

For a 2D example of the use of Equation (2.21), the polar transformation relates Cartesian $(x,y)$ coordinates to a polar radius and angle,

$$x=r\cos\theta,\qquad y=r\sin\theta.$$

Suppose we draw samples from some density $p(r,\theta)$. What is the corresponding density $p(x,y)$? The Jacobian of this transformation is

$$J=\begin{pmatrix}\cos\theta & -r\sin\theta\\ \sin\theta & r\cos\theta\end{pmatrix},$$

and the determinant is $r(\cos^2\theta+\sin^2\theta)=r$. So, $p(x,y)=p(r,\theta)/r$. Of course, this is backward from what we usually want—typically we start with a sampling strategy in Cartesian coordinates and want to transform it to one in polar coordinates. In that case, we would have

$$p(r,\theta)=r\,p(x,y),\qquad r=\sqrt{x^2+y^2},\quad \theta=\operatorname{atan2}(y,x).$$

In 3D, given the spherical coordinate representation of directions, Equation (3.7), the Jacobian of this transformation has determinant $|J_T|=r^2\sin\theta$, so the corresponding density function is

$$p(x,y,z)=\frac{p(r,\theta,\phi)}{r^2\sin\theta}, \qquad p(r,\theta,\phi)=r^2\sin\theta\;p(x,y,z).$$

This transformation is important since it helps us represent directions as points $(x,y,z)$ on the unit sphere.

### 2.4.2 Sampling with Multidimensional Transformations

Suppose we have a 2D joint density function $p(x,y)$ that we wish to draw samples $(X,Y)$ from. If the densities are independent, they can be expressed as the product of 1D densities

$$p(x,y)=p_x(x)\,p_y(y),$$

and random variables $(X,Y)$ can be found by independently sampling $X$ from $p_x$ and $Y$ from $p_y$. Many useful densities are not separable, however, so we introduce the theory of how to sample from multidimensional distributions in the general case.

Given a 2D density function, the marginal density function $p(x)$ is obtained by "integrating out" one of the dimensions:

$$p(x)=\int p(x,y)\,dy.$$

This can be thought of as the density function for $X$ alone. More precisely, it is the average density for a particular $x$ over all possible $y$ values.

If we can draw a sample $X\sim p(x)$, then—using Equation (2.1)—we can see that in order to sample $Y$, we need to sample from the conditional probability density, $Y\sim p(y\mid x)$, which is given by

$$p(y\mid x)=\frac{p(x,y)}{p(x)}.$$

Sampling from higher-dimensional distributions can be performed in a similar fashion, integrating out all but one of the dimensions, sampling that one, and then applying the same technique to the remaining conditional distribution, which has one fewer dimension.

**Sampling the Bilinear Function.** The bilinear function

$$f(x,y)=(1-x)(1-y)w_0+x(1-y)w_1+(1-x)y\,w_2+xy\,w_3$$

interpolates between four values $w_i$ at the four corners of $[0,1]^2$ ($w_0$ at $(0,0)$, $w_1$ at $(1,0)$, $w_2$ at $(0,1)$, and $w_3$ at $(1,1)$). After integration and normalization, we find that its PDF is

$$p(x,y)=\frac{4\,f(x,y)}{w_0+w_1+w_2+w_3}.$$

```rust
// Fragment: Sampling Inline Functions
#[derive(Copy, Clone, Debug, Default)]
pub struct Point2f {
    pub x: Float,
    pub y: Float,
}

/// PDF of the bilinear distribution over [0,1]².
pub fn bilinear_pdf(p: Point2f, w: &[Float; 4]) -> Float {
    if !(0.0..=1.0).contains(&p.x) || !(0.0..=1.0).contains(&p.y) {
        return 0.0;
    }
    let sum: Float = w.iter().copied().sum();
    if sum == 0.0 {
        return 1.0;
    }
    4.0 * ((1.0 - p.x) * (1.0 - p.y) * w[0]
         + p.x * (1.0 - p.y) * w[1]
         + (1.0 - p.x) * p.y * w[2]
         + p.x * p.y * w[3]) / sum
}
```

The two dimensions of this function are not independent, so the sampling method samples a marginal distribution before sampling the resulting conditional distribution.

```rust
// Fragment: Sampling Inline Functions (continued)
/// Sample the bilinear distribution: marginal in y, then conditional in x.
pub fn sample_bilinear(u: Point2f, w: &[Float; 4]) -> Point2f {
    let mut p = Point2f::default();
    // Sample y for bilinear marginal distribution.
    p.y = sample_linear(u.y, w[0] + w[1], w[2] + w[3]);
    // Sample x for bilinear conditional distribution.
    let a = lerp(p.y, w[0], w[2]);
    let b = lerp(p.y, w[1], w[3]);
    p.x = sample_linear(u.x, a, b);
    p
}
```

We can choose either $x$ or $y$ to be the marginal distribution. If we choose $y$ and integrate out $x$, we find that

$$p(y)=\frac{2\big[(1-y)(w_0+w_1)+y(w_2+w_3)\big]}{w_0+w_1+w_2+w_3}.$$

$p(y)$ performs linear interpolation between two constant values, and so we can use `sample_linear()` to sample from the simplified proportional function since it normalizes the associated PDF.

Applying Equation (2.1) and again canceling out common factors, we have

$$p(x\mid y)=\frac{2\big[(1-x)\operatorname{lerp}(y,w_0,w_2)+x\operatorname{lerp}(y,w_1,w_3)\big]} {\operatorname{lerp}(y,w_0,w_2)+\operatorname{lerp}(y,w_1,w_3)},$$

which can also be sampled in $x$ using `sample_linear()`.

Because the bilinear sampling routine is based on the composition of two 1D linear sampling operations, it can be inverted by applying the inverses of those two operations in reverse order.

```rust
// Fragment: Sampling Inline Functions (continued)
/// Invert sample_bilinear: return the (u.x, u.y) that produces p.
pub fn invert_bilinear_sample(p: Point2f, w: &[Float; 4]) -> Point2f {
    let a = lerp(p.y, w[0], w[2]);
    let b = lerp(p.y, w[1], w[3]);
    Point2f {
        x: invert_linear_sample(p.x, a, b),
        y: invert_linear_sample(p.y, w[0] + w[1], w[2] + w[3]),
    }
}
```

See Section A.5 for further examples of multidimensional sampling algorithms, including techniques for sampling directions on the unit sphere and hemisphere, sampling unit disks, and other useful distributions for rendering.

> *Printed-edition index: `Float` (p. 23), `Point2f` (p. 92), `SampleLinear()` (p. 73).*

## Further Reading

The Monte Carlo method was introduced soon after the development of the digital computer by Stanislaw Ulam and John von Neumann (Ulam et al. 1947), though it also seems to have been independently invented by Enrico Fermi (Metropolis 1987). An early paper on Monte Carlo was written by Metropolis and Ulam (1949).

Many books have been written on Monte Carlo integration. Hammersley and Handscomb (1964), Spanier and Gelbard (1969), and Kalos and Whitlock (1986) are classic references. More recent books on the topic include those by Sobol′ (1994), Fishman (1996), and Liu (2001). We have also found Owen's in-progress book (2019) to be an invaluable resource. Motwani and Raghavan (1995) have written an excellent introduction to the broader topic of randomized algorithms.

Most of the functions of interest in rendering are nonnegative; applying importance sampling to negative functions requires special care. A straightforward option is to define a sampling distribution that is proportional to the absolute value of the function. See also Owen and Zhou (2000) for a more effective sampling approach for such functions.

Multiple importance sampling was developed by Veach and Guibas (Veach and Guibas 1995; Veach 1997). Normally, a predetermined number of samples are taken using each sampling technique; see Pajot et al. (2011) and Lu et al. (2013) for approaches to adaptively distributing the samples over strategies in an effort to reduce variance by choosing those that are the best match to the integrand. Grittmann et al. (2019) tracked the variance of each sampling technique and then dynamically adjusted the MIS weights accordingly. The MIS compensation approach was developed by Karlík et al. (2019).

Sbert and collaborators (2016, 2017, 2018) have performed further variance analysis on MIS estimators and have developed improved methods based on allocating samples according to the variance and cost of each technique. Kondapaneni et al. (2019) considered the generalization of MIS to include negative weights and derived optimal estimators in that setting. West et al. (2020) considered the case where a continuum of sampling techniques are available and derived an optimal MIS estimator for that case, and Grittmann et al. (2021) have developed improved MIS estimators when correlation is present among samples (as is the case, for example, with bidirectional light transport algorithms).

Heitz (2020) described an inversion-based sampling method that can be applied when CDF inversion of a 1D function is not possible. It is based on sampling from a second function that approximates the first and then using a second random variable to adjust the sample to match the original function's distribution. An interesting alternative to manually deriving sampling techniques was described by Anderson et al. (2017), who developed a domain-specific language for sampling where probabilities are automatically computed, given the implementation of a sampling algorithm. They showed the effectiveness of their approach with succinct implementations of a number of tricky sampling techniques.

The numerically stable sampling technique used in `SampleLinear()` is an application of Muller's method (1956) due to Heitz (2020).

In applications of Monte Carlo in graphics, the integrand is often a product of factors, where no sampling distribution is available that fits the full product. While multiple importance sampling can give reasonable results in this case, at least minimizing variance from ineffective sampling techniques, sampling the full product is still preferable. Talbot et al. (2005) applied importance resampling to this problem, taking multiple samples from some distribution and then choosing among them with probability proportional to the full integrand. More recently, Hart et al. (2020) presented a simple technique based on warping uniform samples that can be used to approximate product sampling. For more information on this topic, see also the "Further Reading" sections of Chapters 13 and 14, which discuss product sampling approaches in the context of specific light transport algorithms.

Debugging Monte Carlo algorithms can be challenging, since it is their behavior in expectation that determines their correctness: it may be difficult to tell if the program execution for a particular sample is correct. Statistical tests can be an effective approach for checking their correctness. See the papers by Subr and Arvo (2007a) and by Jung et al. (2020) for applicable techniques.

See also the "Further Reading" section in Appendix A, which has information about the sampling algorithms implemented there as well as related approaches.

> *Printed-edition index: `Float` (p. 23), `InvertLinearSample()` (p. 73), `Lerp()` (p. 72), `Point2f` (p. 92), `SampleLinear()` (p. 73).*

## Exercises

> *Printed-edition index: `SampleLinear()` (p. 73).*

**2.1 (➋)** Write a program that compares Monte Carlo and one or more alternative numerical integration techniques. Structure this program so that it is easy to replace the particular function being integrated. Verify that the different techniques compute the same result (given a sufficient number of samples for each of them). Modify your program so that it draws samples from distributions other than the uniform distribution for the Monte Carlo estimate, and verify that it still computes the correct result when the correct estimator, Equation (2.7), is used. (Make sure that any alternative distributions you use have nonzero probability of choosing any value of $x$ where $f(x)>0$.)

**2.2 (➊)** Write a program that computes unbiased Monte Carlo estimates of the integral of a given function. Compute an estimate of the variance of the estimates by performing a series of trials with successively more samples and computing the mean squared error for each one. Demonstrate numerically that variance decreases at a rate of $\mathcal{O}(n)$.

**2.3 (➋)** The algorithm for sampling the linear interpolation function in Section 2.3.2 implicitly assumes that $a,b\ge 0$ and that thus $f(x)\ge 0$. If $f$ is negative, then the importance sampling PDF should be proportional to $|f(x)|$. Generalize `SampleLinear()` and the associated PDF and inversion functions to handle the case where $f$ is always negative as well as the case where it crosses zero due to $a$ and $b$ having different signs.

> *Printed-edition index: `SampleLinear()` (p. 73).*

## Notes

[^1]: For brevity, we will refer to Monte Carlo integration simply as "Monte Carlo."

[^2]: Although the theory of Monte Carlo is based on using truly random numbers, in practice a well-written pseudo-random number generator (PRNG) is sufficient. pbrt uses a particularly high-quality PRNG that returns a sequence of pseudo-random values that is effectively as "random" as true random numbers. True random numbers, found by measuring random phenomena like atomic decay or atmospheric noise, are available from sources like www.random.org for those for whom PRNGs are not acceptable.

[^3]: When computing expected values with a uniform distribution, we will drop the subscript $p$ from $\mathbb{E}_p$.

[^4]: As a technical note, it is possible for an estimator with infinite variance to be unbiased but not consistent. Such estimators do not generally come up in rendering, however.

[^5]: See Veach (1997) for a derivation of this result.

[^6]: We will generally assume that $f(x)\ge 0$; if it is negative, we might set $p(x)\propto|f(x)|$. See the "Further Reading" section for more discussion of this topic.

[^7]: Note that the definition of variance in Equation (2.8) does not preclude computing the variance of a function itself.

[^8]: In general, the lower limit of integration should be $-\infty$, although if $p(x)=0$ for $x<0$, this equation is equivalent.

---

[↑ Back to top](#contents)
