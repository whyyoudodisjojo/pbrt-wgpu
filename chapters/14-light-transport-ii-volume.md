# Chapter 14: Light Transport II: Volume

<!-- interactive-nav -->
## Contents

- [Volume Light Transport (wgpu)](#volume-light-transport-wgpu)
- [14.1 The Equation of Transfer](#141-the-equation-of-transfer)
  - [14.1.1 Null-Scattering Extension](#1411-null-scattering-extension)
  - [14.1.2 Evaluating the Equation of Transfer](#1412-evaluating-the-equation-of-transfer)
  - [14.1.3 Sampling the Majorant Transmittance](#1413-sampling-the-majorant-transmittance)
  - [14.1.4 Generalized Path Space](#1414-generalized-path-space)
  - [14.1.5 Evaluating the Volumetric Path Integral](#1415-evaluating-the-volumetric-path-integral)
- [14.2 Volume Scattering Integrators](#142-volume-scattering-integrators)
  - [14.2.1 A Simple Volumetric Integrator](#1421-a-simple-volumetric-integrator)
  - [14.2.2 Improving the Sampling Techniques](#1422-improving-the-sampling-techniques)
  - [14.2.3 Improved Volumetric Integrator](#1423-improved-volumetric-integrator)
- [14.3 Scattering from Layered Materials](#143-scattering-from-layered-materials)
  - [14.3.1 The One-Dimensional Equation of Transfer](#1431-the-one-dimensional-equation-of-transfer)
  - [14.3.2 Layered BxDF](#1432-layered-bxdf)
  - [14.3.3 Coated Diffuse and Coated Conductor Materials](#1433-coated-diffuse-and-coated-conductor-materials)
- [Further Reading](#further-reading)
- [Exercises](#exercises)

---

## Volume Light Transport (wgpu)

```mermaid
flowchart TD
    A[Camera ray] --> B[Intersect scene geometry]
    B --> C{Ray passes through a medium?}
    C -- no --> H[Surface / environment contribution]
    C -- yes --> D[Sample distance from majorant transmittance T_maj]
    D --> E{Select event}
    E -- absorption --> F[Add emission L_e; terminate path]
    E -- real scattering --> G[Sample phase function; spawn new ray]
    E -- null scattering --> D
    E -- past t_max --> H
    G --> B
    F --> P[Accumulate into pixel]
    H --> P
```

The abstractions for representing participating media that were introduced in Chapter 11 describe how media scatter light, but they do not provide the capability of simulating the global effects of light transport in a scene. The situation is similar to that with `BSDFs`: they describe local effects, but it was necessary to start to introduce integrators in Chapter 13 that accounted for direct lighting and interreflection in order to render images. This chapter does the same for volumetric scattering.

We begin with the introduction of the equation of transfer, which generalizes the light transport equation to describe the equilibrium distribution of radiance in scenes with participating media. Like the transmittance equations in Section 11.2, the equation of transfer has a null-scattering generalization that allows sampling of heterogeneous media for unbiased integration. We will also introduce a path integral formulation of it that generalizes the surface path integral from Section 13.1.4.

Following sections discuss implementations of solutions to the equation of transfer. Section 14.2 introduces two `Integrators` that use Monte Carlo integration to solve the full equation of transfer, making it possible to render scenes with complex volumetric effects. Section 14.3 then describes the implementation of `LayeredBxDF`, which solves a 1D specialization of the equation of transfer to model scattering from layered materials at surfaces.

> **wgpu orientation.** pbrt expresses these algorithms with C++ templates, tagged pointers, and callbacks. On the GPU there are no closures and no dynamic dispatch, so the wgpu versions below make different choices: media and phase functions are represented by tagged storage-buffer structs, a ray's majorant segments are produced by an explicit `RayMajorantIterator` index, and the "callback" of `SampleT_maj()` is inlined into the sampling loop. Radiance is shown as `vec3<f32>` (RGB) for clarity; a fully spectral implementation replaces it with a small fixed-size array of wavelength samples plus a `SampledWavelengths` mask, exactly mirroring `SampledSpectrum`.

## 14.1 THE EQUATION OF TRANSFER

The equation of transfer is the fundamental equation that governs the behavior of light in a medium that absorbs, emits, and scatters radiation. It accounts for all the volume scattering processes described in Chapter 11—absorption, emission, in scattering, and out scattering—to give an equation that describes the equilibrium distribution of radiance. The light transport equation is in fact a special case of it, simplified by the lack of participating media and specialized for scattering from surfaces. (We will equivalently refer to the equation of transfer as the volumetric light transport equation.)

Reference / index entries appearing in the page margin: `Integrator` 22, `LayeredBxDF` 895.

**Figure 14.1.** The equation of transfer gives the incident radiance $L_i(\mathrm{p},\omega)$ at a point, accounting for participating media. At each point $\mathrm{p}'$ along the ray, the source function $L_s(\mathrm{p}',-\omega)$ gives the differential radiance added there by scattering and emission; that radiance is attenuated by the beam transmittance $T_r(\mathrm{p}'\to\mathrm{p})$ from $\mathrm{p}'$ to the ray's origin. In short, $L_i(\mathrm{p},\omega)$ is the integral of all such attenuated contributions along the ray.

```mermaid
flowchart LR
    SRC["Source point p' in the medium<br/>L_s(p', -ω)"] -->|"× σ_t(p') dt'"| ADD["differential radiance added<br/>by scattering + emission"]
    ADD -->|"× T_r(p' → p)"| OBS["observed at p:<br/>L_i(p, ω) = ∫ T_r σ_t L_s dt'"]
```

In its most basic form, the equation of transfer is an integro-differential equation that describes how the radiance along a beam changes at a point in space. It can be derived by subtracting the effects of the scattering processes that reduce energy along a beam (absorption and out scattering) from the processes that increase energy along it (emission and in scattering). To start, recall the source function $L_s$ from Section 11.1.4: it gives the change in radiance at a point $\mathrm{p}$ in a direction $\omega$ due to emission and in-scattered light from other points in the medium:

$$
L_s(\mathrm{p}, \omega)
= L_e(\mathrm{p}, \omega)
+ \sigma_s(\mathrm{p}, \omega) \int_{\mathcal{S}^2} p(\omega, \omega')\, L_i(\mathrm{p}, \omega')\, \mathrm{d}\omega'.
\tag{11.3}
$$

The source function accounts for all the processes that add radiance to a ray. The attenuation coefficient, $\sigma_t(\mathrm{p},\omega)$, accounts for all processes that reduce radiance at a point: absorption and out scattering. The differential equation that describes its effect, Equation (11.4), is

$$
\mathrm{d}L_o(\mathrm{p},\omega) = -\,\sigma_t(\mathrm{p},\omega)\, L_i(\mathrm{p},-\omega)\, \mathrm{d}t.
\tag{11.4}
$$

The overall differential change in radiance at a point $\mathrm{p}' = \mathrm{p} + t\omega$ along a ray is found by adding these two effects together to get the integro-differential form of the equation of transfer:[^14-1]

$$
\frac{\mathrm{d}L(\mathrm{p},\omega)}{\mathrm{d}t}
= -\sigma_t(\mathrm{p},\omega)\, L(\mathrm{p},\omega)
  + \sigma_t(\mathrm{p},\omega)\, L_s(\mathrm{p},\omega).
\tag{14.1}
$$

(The $\sigma_t$ modulation of the source function accounts for the medium's density at the point.)

With suitable boundary conditions, this equation can be transformed to a pure integral equation that describes the effect of participating media from the infinite number of points along a ray. For example, if we assume that there are no surfaces in the scene so that the rays are never blocked and have an infinite length, the integral equation of transfer is

$$
L_i(\mathrm{p}, \omega)
= \int_0^{\infty} T_r(\mathrm{p}' \to \mathrm{p})\, \sigma_t(\mathrm{p}')\, L_s(\mathrm{p}', -\omega)\, \mathrm{d}t',
\qquad \mathrm{p}' = \mathrm{p} + t'\omega.
\tag{14.2a}
$$

(See Figure 14.1.) The meaning of this equation is reasonably intuitive: it just says that the radiance arriving at a point from a given direction is determined by accumulating the radiance added at all points along the ray. The amount of added radiance at each point along the ray that reaches the ray's origin is reduced by the beam transmittance to the point.

**Figure 14.2.** For a finite ray that intersects a surface, the incident radiance $L_i(\mathrm{p},\omega)$ equals the outgoing surface radiance $L_o(\mathrm{p}_s,-\omega)$ times the beam transmittance to the surface, plus the radiance added by scattering and emission at all points from $\mathrm{p}$ to $\mathrm{p}_s$.

```mermaid
flowchart LR
    SURF["Surface point p_s<br/>L_o(p_s, -ω)"] -->|"× T_r(p_s → p)"| OBS["L_i(p, ω) at ray origin p"]
    MED["Medium points p' ∈ [p, p_s]<br/>L_s(p', -ω)"] -->|"∫₀ᵗ T_r(p' → p) σ_t L_s dt'"| OBS
    BEYOND["Points beyond p_s"] -.->|"blocked — no contribution"| BLOCKED["✗"]
```

More generally, if there are reflecting or emitting surfaces in the scene, rays do not necessarily have infinite length and the first surface that a ray hits affects its radiance, adding outgoing radiance from the surface at the point and preventing radiance from points along the ray beyond the intersection point from contributing to radiance at the ray's origin. If a ray $(\mathrm{p},\omega)$ intersects a surface at some point $\mathrm{p}_s$ at a parametric distance $t$ along the ray, then the integral equation of transfer is

$$
L_i(\mathrm{p}, \omega)
= T_r(\mathrm{p}_s \to \mathrm{p})\, L_o(\mathrm{p}_s, -\omega)
  + \int_0^{t} T_r(\mathrm{p}' \to \mathrm{p})\, \sigma_t(\mathrm{p}')\, L_s(\mathrm{p}', -\omega)\, \mathrm{d}t',
\tag{14.2}
$$

where $\mathrm{p}' = \mathrm{p} + t'\omega$ are points along the ray (Figure 14.2). This equation describes the two effects that contribute to radiance along the ray. First, reflected radiance back along the ray from the surface is given by

$$
T_r(\mathrm{p}_s \to \mathrm{p})\, L_o(\mathrm{p}_s, -\omega),
$$

the $L_o$ term, which gives the emitted and reflected radiance from the surface. This radiance may be attenuated by the participating media; the beam transmittance from the ray origin to the point $\mathrm{p}_s$ accounts for this. The second term accounts for the added radiance along the ray due to volumetric scattering and emission up to the point where the ray intersects the surface; points beyond that one do not affect the radiance along the ray.

### 14.1.1 NULL-SCATTERING EXTENSION

In Section 11.2.1 we saw the value of null scattering, which made it possible to sample from a modified transmittance equation and to compute unbiased estimates of the transmittance between two points using algorithms like delta tracking and ratio tracking. Null scattering can be applied in a similar way to the equation of transfer, giving similar benefits. In order to simplify notation in the following, we will assume that the various scattering coefficients $\sigma$ do not vary as a function of direction. As before, we will also assume that the null-scattering coefficient $\sigma_n$ is nonnegative and has been set to homogenize the medium's density to a fixed majorant $\sigma_{\mathrm{maj}} = \sigma_n + \sigma_t$. Neither of these simplifications affect the course of the following derivations; both generalizations could be easily reintroduced. A null-scattering generalization of the equation of transfer can be found using the relationship $\sigma_t = \sigma_{\mathrm{maj}} - \sigma_n$ from Equation (11.11). If that substitution is made in the integro-differential equation of transfer, Equation (14.1), and the boundary condition of a surface at distance $t$ along the ray is applied, then the result can be transformed into the pure integral equation

$$
L_i(\mathrm{p}, \omega)
= T_{\mathrm{maj}}(\mathrm{p}_s \to \mathrm{p})\, L_o(\mathrm{p}_s, -\omega)
  + \int_0^{t} T_{\mathrm{maj}}(\mathrm{p}' \to \mathrm{p})\, \sigma_{\mathrm{maj}}\, L_n(\mathrm{p}', -\omega)\, \mathrm{d}t',
\tag{14.3}
$$

where $\mathrm{p}' = \mathrm{p} + t'\omega$, as before, and we have introduced $T_{\mathrm{maj}}$ to denote the majorant transmittance that accounts for both regular attenuation and null scattering. Using the same convention as before that $d = \|\mathrm{p} - \mathrm{p}'\|$ is the distance between points $\mathrm{p}$ and $\mathrm{p}'$, it is

$$
T_{\mathrm{maj}}(\mathrm{p} \to \mathrm{p}')
= \exp\!\left(-\int_0^{d} \sigma_{\mathrm{maj}}(\mathrm{p} + s\omega)\, \mathrm{d}s\right).
\tag{14.4}
$$

The null-scattering source function $L_n$ is the source function $L_s$ from Equation (11.3) plus a new third term:

$$
\begin{aligned}
L_n(\mathrm{p}, \omega)
&= L_s(\mathrm{p},\omega) + \sigma_n(\mathrm{p})\, L_i(\mathrm{p},\omega) \\
&= L_e(\mathrm{p},\omega)
   + \sigma_s(\mathrm{p}) \int_{\mathcal{S}^2} p(\omega,\omega')\, L_i(\mathrm{p},\omega')\, \mathrm{d}\omega'
   + \sigma_n(\mathrm{p})\, L_i(\mathrm{p},\omega).
\end{aligned}
\tag{14.5}
$$

Because it includes attenuation due to null scattering, $T_{\mathrm{maj}}$ is always less than or equal to the actual transmittance. Thus, the product $T_{\mathrm{maj}}L_o$ in Equation (14.3) may be less than the actual contribution of radiance leaving the surface, $T_r L_o$. However, any such deficiency is made up for by the last term of Equation (14.5).

### 14.1.2 EVALUATING THE EQUATION OF TRANSFER

The $T_{\mathrm{maj}}$ factor in the null-scattering equation of transfer gives a convenient distribution for sampling distances $t$ along the ray in the medium that leads to the volumetric path-tracing algorithm, which we will now describe. (The algorithm we will arrive at is sometimes described as using delta tracking to solve the equation of transfer, since that is the sampling technique it uses for finding the locations of absorption and scattering events.) If we assume for now that there is no geometry in the scene, then the null-scattering equation of transfer, Equation (14.3), simplifies to

$$
L_i(\mathrm{p},\omega)
= \int_0^{\infty} \sigma_{\mathrm{maj}}\, T_{\mathrm{maj}}(\mathrm{p}'\to\mathrm{p})\, L_n(\mathrm{p}',-\omega)\, \mathrm{d}t'.
\tag{14.6}
$$

Thanks to null scattering having made the majorant medium homogeneous, $\sigma_{\mathrm{maj}}T_{\mathrm{maj}}$ can be sampled exactly. The first step in the path-tracing algorithm is to sample a point $\mathrm{p}'$ from its distribution, giving the estimator

$$
\left\langle L_i(\mathrm{p},\omega)\right\rangle
= \frac{\sigma_{\mathrm{maj}}\, T_{\mathrm{maj}}(\mathrm{p}'\to\mathrm{p})\, L_n(\mathrm{p}',-\omega)}
        {p(\mathrm{p}')},
\qquad \mathrm{p}' \sim \sigma_{\mathrm{maj}}T_{\mathrm{maj}}.
$$

From Section A.4.2, we know that the probability density function (PDF) for sampling a distance $t$ from the exponential distribution is $p(t) = \sigma_{\mathrm{maj}} e^{-\sigma_{\mathrm{maj}} t} = \sigma_{\mathrm{maj}}T_{\mathrm{maj}}$, and so the estimator simplifies to

$$
\left\langle L_i(\mathrm{p},\omega)\right\rangle = L_n(\mathrm{p}',-\omega).
$$

What is left is to evaluate $L_n$.

Because $\sigma_{\mathrm{maj}} = \sigma_a + \sigma_s + \sigma_n$, the initial $\sigma$ factors in each term of Equation (14.5) can be considered to be three probabilities that sum to 1. If one of the three terms is randomly selected according to its probability and the rest of the term is evaluated without that factor, the expected value of the result is equal to $L_n$. Considering how to evaluate each of the terms:

- If the $\sigma_a$ term is chosen, then the emission at $L_e(\mathrm{p}',\omega)$ is returned and sampling terminates.
- For the $\sigma_s$ term, the integral over the sphere of directions must be estimated. A direction $\omega'$ is sampled from some distribution and recursive evaluation of $L_i(\mathrm{p}',\omega')$ then proceeds, weighted by the ratio of the phase function and the probability of sampling the direction $\omega'$.
- If the null-scattering term is selected, $L_i(\mathrm{p}',\omega)$ is to be evaluated, which can be handled recursively as well.

For the full equation of transfer that includes scattering from surfaces, both the surface-scattering term and the integral over the ray's extent lead to recursive evaluation of the equation of transfer. In the context of path tracing, however, we would like to only evaluate one of the terms in order to avoid an exponential increase in work. We will therefore start by defining a probability $q$ of estimating the surface-scattering term; volumetric scattering is evaluated otherwise. Given such a $q$, the Monte Carlo estimator

$$
L_i(\mathrm{p},\omega) \approx
\begin{cases}
\dfrac{T_{\mathrm{maj}}(\mathrm{p}_s\to\mathrm{p})}{q}\, L_o(\mathrm{p}_s,-\omega),
    & \text{with probability } q,\\[6pt]
\dfrac{1}{1-q}\displaystyle\int_0^{t}
    \sigma_{\mathrm{maj}} T_{\mathrm{maj}}(\mathrm{p}'\to\mathrm{p})\,
    L_n(\mathrm{p}',-\omega)\,\mathrm{d}t',
    & \text{otherwise},
\end{cases}
$$

gives $L_i(\mathrm{p},\omega)$ in expectation. A good choice for $q$ is that it be equal to $T_{\mathrm{maj}}(\mathrm{p}_s\to\mathrm{p})$. Surface scattering is then evaluated with a probability proportional to the transmittance to the surface and the ratio $T_{\mathrm{maj}}/q$ is equal to 1, leaving just the $L_o$ factor. Furthermore, a sampling trick can be used to choose between the two terms: if a sample $t' \in [0,\infty)$ is taken from $\sigma_{\mathrm{maj}}T_{\mathrm{maj}}$'s distribution, then the probability that $t' > t$ is equal to $T_{\mathrm{maj}}(\mathrm{p}_s\to\mathrm{p})$. (This can be shown by integrating $T_{\mathrm{maj}}$'s PDF to find its cumulative distribution function (CDF) and then considering the value of its CDF at $t$.) Using this technique and

$$
q = T_{\mathrm{maj}}(\mathrm{p}_s\to\mathrm{p}),
$$

then making the same simplifications that brought us to Equation (14.6), we arrive at the estimator

$$
L_i(\mathrm{p},\omega) \approx
\begin{cases}
L_o(\mathrm{p}_s,-\omega), & t' > t \quad\text{(surface)},\\[4pt]
L_n(\mathrm{p}',-\omega),   & t' \le t \quad\text{(medium)},
\end{cases}
\qquad \mathrm{p}' = \mathrm{p} + t'\omega,\quad t'\sim \sigma_{\mathrm{maj}}T_{\mathrm{maj}}.
\tag{14.7}
$$

From this point, outgoing radiance from a surface can be estimated using techniques that were introduced in Chapter 13, and $L_n$ can be estimated as described earlier.

### 14.1.3 SAMPLING THE MAJORANT TRANSMITTANCE

We have so far presented volumetric path tracing with the assumption that $\sigma_{\mathrm{maj}}$ is constant along the ray and thus that $T_{\mathrm{maj}}$ is a single exponential function. However, those assumptions are not compatible with the segments of piecewise-constant majorants that `Medium` implementations provide with their `RayMajorantIterator`s. We will now resolve this incompatibility.

**Figure 14.3.** Given piecewise-constant majorants defined over segments along a ray, the corresponding optical thickness $\tau$ is a piecewise-linear function; exponentiating $-\tau$ gives the transmittance at each point along the ray. The transmittance function is continuous and strictly decreasing, with a first-derivative discontinuity at transitions between segments. If integration starts at $t=0$ and the $i$th segment's majorant is denoted $\sigma_{\mathrm{maj},i}$ with endpoint $\mathrm{p}_i$, then

$$
T_{\mathrm{maj}}(0 \to \mathrm{p}') =
\prod_{i=1}^{n-1} T_{\mathrm{maj},i}\;\cdot\;T_{\mathrm{maj},n}(\mathrm{p}_n \to \mathrm{p}'),
$$

where $T_{\mathrm{maj},i}$ is the transmittance function for the $i$th segment and $\mathrm{p}'$ is the endpoint of the $n$th segment. This relationship uses the multiplicative property of transmittance from Equation (11.6).

Reference / index entries appearing in the page margin: `Medium` 714.

The two original panels redrawn as charts (values approximate the book's illustration):

```mermaid
xychart-beta
    title " (a) Piecewise-constant σ_maj (blue) and optical thickness τ (red)"
    x-axis "t" [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
    y-axis "value" 0 --> 4
    line [4.0, 4.0, 4.0, 0.5, 0.5, 0.5, 2.6, 2.6, 2.6, 1.6, 1.6]
    line [0.0, 0.8, 0.9, 1.0, 1.2, 1.4, 1.6, 1.8, 2.0, 2.2, 2.4]
```

```mermaid
xychart-beta
    title " (b) Majorant transmittance T_maj = exp(-τ): continuous, decreasing"
    x-axis "t" [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
    y-axis "T_maj" 0 --> 1
    line [1.00, 0.75, 0.50, 0.46, 0.44, 0.43, 0.36, 0.26, 0.19, 0.13, 0.09]
```

The key relations behind the curves are $\tau(t) = \int_0^t \sigma_{\mathrm{maj}}(s)\,\mathrm{d}s$ (piecewise linear) and $T_{\mathrm{maj}}(t) = e^{-\tau(t)}$ (continuous and decreasing, with derivative discontinuities where $\sigma_{\mathrm{maj}}$ jumps).

Given the general task of estimating an integral of the form

$$
\int_0^{t} f(\mathrm{p}')\, \sigma_{\mathrm{maj}}(\mathrm{p}')\,
        T_{\mathrm{maj}}(\mathrm{p}'\to\mathrm{p})\, \mathrm{d}t',
\qquad \mathrm{p}' = \mathrm{p} + t'\omega,
$$

with $\mathrm{p}' = \mathrm{p} + t'\omega$ and $f$ the remaining integrand, it is useful to rewrite the integral to be over the individual majorant segments, which gives

$$
\int_0^{t}\! f\,\sigma_{\mathrm{maj}}T_{\mathrm{maj}}\,\mathrm{d}t'
= \sum_{i=1}^{n} \left(\prod_{j=1}^{i-1} T_{\mathrm{maj},j}\right)
    \int_{t_{i-1}}^{t_i} \sigma_{\mathrm{maj},i}\,
      T_{\mathrm{maj},i}(\mathrm{p}'\to\mathrm{p}_i)\, f(\mathrm{p}')\, \mathrm{d}t'
  + \left(\prod_{j=1}^{n} T_{\mathrm{maj},j}\right) f(\mathrm{p}_n).
\tag{14.8}
$$

Note that each term's contribution is modulated by the transmittances and majorants from the previous segments. The form of Equation (14.8) hints at a sampling strategy: we start by sampling a value from distribution $p_1$, where

$$
p_1(\mathrm{p}') = \sigma_{\mathrm{maj},1}\, T_{\mathrm{maj},1}(\mathrm{p}'\to\mathrm{p}_1),
$$

and if $t'$ is less than $t_1$, then we evaluate the estimator at the sampled point $\mathrm{p}'$:

$$
\frac{f(\mathrm{p}')}{p_1(\mathrm{p}')} = f(\mathrm{p}').
$$

Applying the same ideas that led to Equation (14.7), we otherwise continue and consider the second term, drawing a sample starting at $t_1$ from the distribution

$$
p_2(\mathrm{p}') = \sigma_{\mathrm{maj},2}\, T_{\mathrm{maj},2}(\mathrm{p}'\to\mathrm{p}_2).
$$

If the sampled point is before the segment's endpoint, $t' < t_2$, then we have the estimator

$$
\frac{f(\mathrm{p}')}{p_2(\mathrm{p}')} = f(\mathrm{p}').
$$

Because the probability that $t' < t_2$ is equal to $\prod_{j=1}^{1} T_{\mathrm{maj},j}$, the estimator for the second term again simplifies to $f(\mathrm{p}')$. Otherwise, following this sampling strategy for subsequent segments similarly leads to the same simplified estimator in the end. It can furthermore be shown that the probability that no sample is generated in any of the segments is equal to the full majorant transmittance from $0$ to $t$, which is exactly the probability required for the surface/volume estimator of Equation (14.7).

`SampleT_maj()` implements this sampling strategy, handling the details of iterating over `RayMajorantSegment`s and sampling them. Its functionality will be used repeatedly in the following volumetric integrators. On the CPU it is a template over a callback; on the GPU we make the segment iterator explicit and inline the event logic.

**Host-side signature (Rust).** The generic callback becomes a trait closure on the host; in the wgpu shader it is expanded into a switch on the sampled event mode.

```rust
// 〈Medium Sampling Functions〉
fn sample_t_maj<F>(
    ray: Ray,
    t_max: f32,
    mut u: f32,
    rng: &mut Rng,
    lambda: &SampledWavelengths,
    mut callback: F,
) -> SampledSpectrum
where
    F: FnMut(Point3f, MediumProperties, SampledSpectrum, SampledSpectrum) -> bool,
{
    // Iterate over RayMajorantSegments and sample each one; the callback
    // returns true when sampling should restart at the sampled point.
    // ...
    SampledSpectrum::from(1.0)
}
```

In addition to a ray and an endpoint along it specified by `t_max`, `sample_t_maj()` takes a single uniform sample and an `Rng` to use for generating any necessary additional samples. This allows it to use a well-distributed value from a `Sampler` for the first sampling decision along the ray while it avoids consuming a variable and unbounded number of sample dimensions if more are needed (recall the discussion of the importance of consistency in sample dimension consumption in Section 8.3). The provided `SampledWavelengths` play their usual role, though the first of them has additional meaning: for media with scattering properties that vary with wavelength, the majorant at the first wavelength is used for sampling. The alternative would be to sample each wavelength independently, though that would cause an explosion in samples to be evaluated in the context of algorithms like path tracing. Sampling a single wavelength can work well for evaluating all wavelengths' contributions if multiple importance sampling (MIS) is used; this topic is discussed further in Section 14.2.2.

A callback function is the last parameter passed to `SampleT_maj()`. This is a significant difference from pbrt's other sampling methods, which all generate a single sample (or sometimes, no sample) each time they are called. When sampling media that has null scattering, however, often a succession of samples are needed along the same ray. (Delta tracking, described in Section 11.2.1, is an example of such an algorithm.) The provided callback function is therefore invoked by `SampleT_maj()` each time a sample is taken. After the callback processes the sample, it should return a Boolean value that indicates whether sampling should recommence starting from the provided sample. With this implementation approach, `SampleT_maj()` can maintain state like the `RayMajorantIterator` between samples along a ray, which improves efficiency. The signature of the callback function should be the following:

```rust
type MediumCallback =
    fn(Point3f, MediumProperties, SampledSpectrum, SampledSpectrum) -> bool;
```

Each invocation of the callback is passed a sampled point along the ray, the associated `MediumProperties` and $\sigma_{\mathrm{maj}}$ for the medium at that point, and the majorant transmittance $T_{\mathrm{maj}}$. The first time callback is invoked, the majorant transmittance will be from the ray origin to the sample; any subsequent invocations give the transmittance from the previous sample to the current one. After sampling concludes, `SampleT_maj()` returns the majorant transmittance $T_{\mathrm{maj}}$ from the last sampled point in the medium (or the ray origin, if no samples were generated) to the ray's endpoint (see Figure 14.4).

**Figure 14.4.** In addition to calling a provided callback function at sampled points in the medium (filled circles below), `SampleT_maj()` returns the majorant transmittance $T_{\mathrm{maj}}$ from the last sampled point to the provided `t_max` value.

```mermaid
flowchart LR
    O["ray origin ○"] --> S1["sample p₁ ●"]
    S1 --> S2["sample p₂ ●"]
    S2 --> S3["sample p₃ ●"]
    S3 --> T["t_max ○"]
    O -. "T_maj" .-> S1
    S1 -. "T_maj" .-> S2
    S2 -. "T_maj" .-> S3
    S3 -. "returned T_maj" .-> T
```

As if all of this was not sufficiently complex, the implementation of `SampleT_maj()` starts out with some tricky C++ code. There is a second variant of `SampleT_maj()` we will introduce shortly that is templated based on the concrete type of `Medium` being sampled. In order to call the appropriate template specialization, we must determine which type of `Medium` the ray is passing through. Conceptually, we would like to do something like the following, using pbrt's `TaggedPointer::Is()` method:

```rust
// Conceptual dispatch on the concrete medium type (host side).
if let Medium::Homogeneous(m) = &ray.medium {
    sample_t_maj_typed::<HomogeneousMedium>(ray, t_max, u, rng, lambda, func);
} else if let Medium::UniformGrid(m) = &ray.medium {
    // ...
}
```

Reference / index entries appearing in the page margin: `Float` 23, `Medium` 714, `MediumProperties` 718, `Ray` 95, `RayMajorantSegment` 718, `RNG` 1054, `SampledSpectrum` 171, `SampledWavelengths` 173, `Sampler` 469.

However, enumerating all the media that are implemented in pbrt in the `SampleT_maj()` function is undesirable: that would add an unexpected and puzzling additional step for users who wanted to extend the system with a new `Medium`. Therefore, the first `SampleT_maj()` function uses the dynamic dispatch capability of the `Medium`'s `TaggedPointer` along with a generic lambda function, `sample`, to determine the `Medium`'s type. `TaggedPointer::Dispatch()` ends up passing the `Medium` pointer back to `sample`; because the parameter is declared as `auto`, it then takes on the actual type of the medium when it is invoked. Thus, the following function has equivalent functionality to the code above but naturally handles all the media that are listed in the `Medium` class declaration without further modification.

```rust
// 〈Medium Sampling Function Definitions〉 — generic dispatch, host side.
fn sample_t_maj<F>(
    ray: Ray,
    t_max: f32,
    mut u: f32,
    rng: &mut Rng,
    lambda: &SampledWavelengths,
    callback: F,
) -> SampledSpectrum
where
    F: FnMut(Point3f, MediumProperties, SampledSpectrum, SampledSpectrum) -> bool,
{
    ray.medium.dispatch(|medium| {
        sample_t_maj_typed(ray, t_max, u, rng, lambda, callback, medium)
    })
}
```

With the concrete type of the medium available, we can proceed with the second instance of `SampleTmaj()`, which can now be specialized based on that type.

```rust
// 〈Medium Sampling Function Definitions〉 — typed implementation.
fn sample_t_maj_typed<M: Medium, F>(
    ray: Ray,
    t_max: f32,
    mut u: f32,
    rng: &mut Rng,
    lambda: &SampledWavelengths,
    callback: F,
) -> SampledSpectrum
where
    F: FnMut(Point3f, MediumProperties, SampledSpectrum, SampledSpectrum) -> bool,
{
    // 〈Normalize ray direction and update t_max accordingly〉
    // 〈Initialize MajorantIterator for ray majorant sampling〉
    // 〈Generate ray majorant samples until termination〉
    // ...
    SampledSpectrum::from(1.0)
}
```

Reference / index entries: `Float` 23, `Medium` 714, `Ray` 95, `Ray::medium` 95, `RNG` 1054, `SampledSpectrum` 171, `SampledWavelengths` 173, `SampleT_maj()` 859, `TaggedPointer` 1073, `TaggedPointer::Dispatch()` 1075, `TaggedPointer::Is()` 1074.

The function starts by normalizing the ray's direction so that parametric distance along the ray directly corresponds to distance from the ray's origin. This simplifies subsequent transmittance computations in the remainder of the function. Since normalization scales the direction's length, the `t_max` endpoint must also be updated so that it corresponds to the same point along the ray.

```rust
// 〈Normalize ray direction and update t_max accordingly〉
t_max *= ray.d.length();
ray.d = ray.d.normalize();
```

Since the actual type of the medium is known and because all `Medium` implementations must define a `MajorantIterator` type (recall Section 11.4.1), the medium's iterator type can be directly declared as a stack-allocated variable. This gives a number of benefits: not only is the expense of dynamic allocation avoided, but subsequent calls to the iterator's `next()` method in this function are regular method calls that can even be expanded inline by the compiler; no dynamic dispatch is necessary for them. An additional benefit of knowing the medium's type is that the appropriate `sample_ray()` method can be called directly without incurring the cost of dynamic dispatch here.

```rust
// 〈Initialize MajorantIterator for ray majorant sampling〉
let medium: &ConcreteMedium = ray.medium.cast::<ConcreteMedium>();
let mut iter: MajorantIterator<ConcreteMedium> = medium.sample_ray(ray, t_max, lambda);
```

**wgpu version of the iterator.** A WGSL compute shader cannot hold a generic iterator or a closure; instead the majorant segments live in a storage buffer and the iterator is an integer cursor. The sampled event is represented by an enum so the event logic can be inlined.

```wgsl
struct MajorantSegment {
    t_min: f32,
    t_max: f32,
    sigma_maj: vec3<f32>,   // majorant extinction per wavelength
};

struct MajorantIterator {
    segments: ptr<storage, array<MajorantSegment>, read>,
    index: u32,
    count: u32,
};

fn iter_next(iter: ptr<function, MajorantIterator>) -> MajorantSegment {
    if ((*iter).index >= (*iter).count) {
        return MajorantSegment(0.0, -1.0, vec3<f32>(0.0)); // sentinel: done
    }
    let seg = (*iter).segments[(*iter).index];
    (*iter).index += 1u;
    return seg;
}

// Event modes replace the host-side callback.
const EVENT_ABSORB: u32 = 0u;
const EVENT_SCATTER: u32 = 1u;
const EVENT_NULL: u32 = 2u;
```

With an iterator initialized, sampling along the ray can proceed. The `t_maj` variable declared here tracks the accumulated majorant transmittance from the ray origin or the previous sample along the ray (depending on whether a sample has yet been generated).

```rust
// 〈Generate ray majorant samples until termination〉
let mut t_maj = SampledSpectrum::from(1.0);
let mut done = false;
while !done {
    // 〈Get next majorant segment from iterator and sample it〉
}
// The final `t_maj` holds the remaining transmittance to t_max.
t_maj
```

If the iterator has no further majorant segments to provide, then sampling is complete. In this case, it is important to return any majorant transmittance that has accumulated in `t_maj`; that represents the remaining transmittance to the ray's endpoint. Otherwise, a few details are attended to before sampling proceeds along the segment.

```rust
// 〈Get next majorant segment from iterator and sample it〉
let seg = match iter.next() {
    Some(s) => s,
    None => return t_maj,
};
// 〈Handle zero-valued majorant for current segment〉
// 〈Generate samples along current majorant segment〉
```

If the majorant has the value 0 in the first wavelength, then there is nothing to sample along the segment. It is important to handle this case, since otherwise the subsequent call to `sample_exponential()` in this function would return an infinite value that would subsequently lead to not-a-number values. Because the other wavelengths may not themselves have zero-valued majorants, we must still update `t_maj` for the segment's majorant transmittance even though the transmittance for the first wavelength is unchanged.

Reference / index entries: `Length()` 88, `Medium::SampleRay()` 719, `Normalize()` 88, `Ray::d` 95, `Ray::medium` 95, `RayMajorantIterator::Next()` 719, `RayMajorantSegment` 718, `SampledSpectrum` 171.

```rust
// 〈Handle zero-valued majorant for current segment〉
if seg.sigma_maj[0] == 0.0 {
    let mut dt = seg.t_max - seg.t_min;
    // 〈Handle infinite dt for ray majorant segment〉
    t_maj *= fast_exp(-dt * seg.sigma_maj);
    continue;
}
```

One edge case must be attended to before the exponential function is called. If `t_max` holds the IEEE floating-point infinity value, then `dt` will as well; it then must be bumped down to the largest finite `Float`. This is necessary because with floating-point arithmetic, zero times infinity gives a not-a-number value (whereas any nonzero value times infinity gives infinity). Otherwise, for any wavelengths with zero-valued `sigma_maj`, not-a-number values would be passed to `fast_exp()`.

```rust
// 〈Handle infinite dt for ray majorant segment〉
if dt.is_infinite() {
    dt = f32::MAX;
}
```

The implementation otherwise tries to generate a sample along the current segment. This work is inside a while loop so that multiple samples may be generated along the segment.

```rust
// 〈Generate samples along current majorant segment〉
let mut t_min = seg.t_min;
loop {
    // 〈Try to generate sample along current majorant segment〉
}
```

In the usual case, a distance is sampled according to the PDF $p(t')=\sigma_{\mathrm{maj},0} e^{-\sigma_{\mathrm{maj},0}t'}$. Separate cases handle a sample that is within the current majorant segment and one that is past it. One detail to note in this fragment is that as soon as the uniform sample `u` has been used, a replacement is immediately generated using the provided `Rng`. In this way, the method maintains the invariant that `u` is always a valid independent sample value. While this can lead to a single excess call to `Rng::uniform()` each time `sample_t_maj()` is called, it ensures the initial `u` value provided to the method is used only once.

```rust
// 〈Try to generate sample along current majorant segment〉
let mut t = t_min + sample_exponential(u, seg.sigma_maj[0]);
u = rng.uniform::<f32>();
if t < seg.t_max {
    // 〈Call callback function for sample within segment〉
} else {
    // 〈Handle sample past end of majorant segment〉
}
```

For a sample within the segment's extent, the final majorant transmittance to be passed to the callback is found by accumulating the transmittance from `t_min` to the sample point. The rest of the necessary medium properties can be found using `sample_point()`. If the callback function returns false to indicate that sampling should conclude, then we have a doubly nested while loop to break out of; a break statement takes care of the inner one, and setting `done` to true causes the outer one to terminate.

Reference / index entries: `FastExp()` 1036, `Float` 23, `IsInf()` 363, `RayMajorantSegment::sigma_maj` 718, `RayMajorantSegment::t_max` 718, `RayMajorantSegment::t_min` 718, `RNG::Uniform<Float>()` 1056, `SampleExponential()` 1003.

If true is returned by the callback, indicating that sampling should restart at the sample that was just generated, then the accumulated transmittance is reset to 1 and `t_min` is updated to be at the just-taken sample's position.

```rust
// 〈Call callback function for sample within segment〉
t_maj *= fast_exp(-(t - t_min) * seg.sigma_maj);
let mp = medium.sample_point(ray.at(t), lambda);
if !callback(ray.at(t), mp, seg.sigma_maj, t_maj) {
    done = true;
    break;
}
t_maj = SampledSpectrum::from(1.0);
t_min = t;
```

If the sampled distance `t` is past the end of the segment, then there is no medium interaction along it and it is on to the next segment, if any. In this case, majorant transmittance up to the end of the segment must be accumulated into `t_maj` so that the complete majorant transmittance along the ray is provided with the next valid sample (if any).

```rust
// 〈Handle sample past end of majorant segment〉
let dt = seg.t_max - t_min;
t_maj *= fast_exp(-dt * seg.sigma_maj);
break;
```

The same loop expressed as a WGSL function, with the event logic inlined, looks like this:

```wgsl
// Sample the medium and return both the final majorant transmittance and
// the state of the event that was selected (for the integrator to act on).
fn sample_t_maj_wgsl(
    ray_o: vec3<f32>,
    ray_d: vec3<f32>,
    t_max: f32,
    u_initial: f32,
    lambda0: u32,
    iter: ptr<function, MajorantIterator>,
) -> MajorantSample {
    var t_maj = vec3<f32>(1.0);
    var t_min = 0.0;
    var u = u_initial;
    var result: MajorantSample;
    result.t_maj = vec3<f32>(1.0);
    result.event = EVENT_NULL;

    loop {
        let seg = iter_next(iter);
        if (seg.t_max <= seg.t_min) { break; }          // no more segments

        if (seg.sigma_maj[lambda0] == 0.0) {
            let dt = min(seg.t_max - t_min, 1.0e30);
            t_maj *= exp(-dt * seg.sigma_maj);
            t_min = seg.t_max;
            continue;
        }

        loop {
            let t = t_min + sample_exponential(u, seg.sigma_maj[lambda0]);
            u = next_uniform();  // replacement draw, invariant maintained
            if (t < seg.t_max) {
                t_maj *= exp(-(t - t_min) * seg.sigma_maj);
                result.p = ray_o + ray_d * t;
                result.t_maj = t_maj;
                result.sigma_maj = seg.sigma_maj;
                result.mode = select_event(seg, u);      // absorb / scatter / null
                if (result.mode != EVENT_NULL) { return result; }
                t_maj = vec3<f32>(1.0);                   // restart at this sample
                t_min = t;
            } else {
                let dt = seg.t_max - t_min;
                t_maj *= exp(-dt * seg.sigma_maj);
                t_min = seg.t_max;
                break;
            }
        }
    }
    result.t_maj = t_maj;   // transmittance from last sample (or origin) to t_max
    result.mode = EVENT_NULL;
    return result;
}
```

### 14.1.4 GENERALIZED PATH SPACE

Just as it was helpful to express the light transport equation (LTE) as a sum over paths of scattering events, it is also helpful to express the null-scattering integral equation of transfer in this form. Doing so makes it possible to apply variance reduction techniques like multiple importance sampling and is a prerequisite for constructing participating medium-aware bidirectional integrators. Recall how, in Section 13.1.4, the surface form of the LTE was repeatedly substituted into itself to derive the path space contribution function for a path of length $n$

$$
P(\mathrm{p}_n)
= \left[\prod_{i=1}^{n-1} f(\mathrm{p}_{i+1}\to\mathrm{p}_i\to\mathrm{p}_{i-1})\,
      G(\mathrm{p}_{i+1}\leftrightarrow\mathrm{p}_i)\right]
  L_e(\mathrm{p}_n\to\mathrm{p}_{n-1}),
$$

where the throughput $T(\mathrm{p}_n)$ was defined as

$$
T(\mathrm{p}_n)
= \prod_{i=1}^{n-1} f(\mathrm{p}_{i+1}\to\mathrm{p}_i\to\mathrm{p}_{i-1})\,
      G(\mathrm{p}_{i+1}\leftrightarrow\mathrm{p}_i).
$$

This previous definition only works for surfaces, but using a similar approach of substituting the integral equation of transfer, a medium-aware path integral can be derived. The derivation is laborious and we will just present the final result here. (The "Further Reading" section has a pointer to the full derivation.)

Previously, integration occurred over a Cartesian product of surface locations $\mathcal{A}^n$. Now, we will need a formal way of writing down an integral over an arbitrary sequence of each of 2D surface locations $\mathcal{A}$, 3D positions in a participating medium $\mathcal{V}$ where actual scattering occurs, and 3D positions in a participating medium $\mathcal{V}_\varnothing$ where null scattering occurs. (The two media $\mathcal{V}$ and $\mathcal{V}_\varnothing$ represent the same volume of space with the same scattering properties, but we will find it handy to distinguish between them in the following.) First, we will focus only on a specific arrangement of $n$ surface and medium vertices encoded in a configuration vector $\mathbf c$. The associated set of paths is given by a Cartesian product of surface locations and medium locations,

$$
\mathcal{P}_n^{\mathbf c}
= \mathcal{X}_{c_0}\times\mathcal{X}_{c_1}\times\cdots\times\mathcal{X}_{c_n},
\qquad \mathcal{X}_{c_i}\in\{\mathcal{A},\,\mathcal{V},\,\mathcal{V}_\varnothing\}.
$$

The set of all paths of length $n$ is the union of the above sets over all possible configuration vectors:

$$
\mathcal{P}_n = \bigcup_{\mathbf c}\,\mathcal{P}_n^{\mathbf c}.
$$

Next, we define a measure, which provides an abstract notion of the "volume" of a subset $D\subseteq\mathcal{P}_n$ that is essential for integration. The measure we will use simply sums up the product of surface area and volume associated with the individual vertices in each of the path spaces of specific configurations:

$$
\mu(D) = \sum_{\mathbf c}\int_{\mathcal{P}_n^{\mathbf c}\,\cap\, D}
         \mathrm{d}\mu(\mathbf c),
\qquad
\mathrm{d}\mu(\mathbf c) = \prod_{i=0}^{n}\mathrm{d}\mathcal{X}_{c_i}.
$$

The measure for null-scattering vertices $\mathrm{d}V_\varnothing$ incorporates a Dirac delta distribution to limit integration to be along the line between successive real-scattering vertices:

$$
\mathrm{d}V_\varnothing(\mathrm{p})
= \delta^{\perp}\!\left(\mathrm{p} - \overline{\mathrm{r_i r_{i+1}}}\right)
  \,\mathrm{d}V(\mathrm{p}),
$$

where $\delta^\perp$ constrains $\mathrm{p}$ to the segment connecting two successive real-scattering vertices.

The generalized path contribution can now be written as

$$
F(\mathrm{p}_n)
= \int_{\mathcal{P}_n} \left[\prod_{i=1}^{n}\bar S_i\,\hat G_i\right]
  L_e^{\mathrm{path}}(\mathrm{p}_{n+1}\to\mathrm{p}_n)\,
  \mathrm{d}\mu(\mathrm{p}_n),
$$

where

$$
L_e^{\mathrm{path}}(\mathrm{p}\to\mathrm{q}) = \sigma_a(\mathrm{p})\, L_e(\mathrm{p}\to\mathrm{q}).
\tag{14.10}
$$

Due to the measure defined earlier, the generalized path contribution is a sum of many integrals considering all possible sequences of surface, volume, and null-scattering events. The full set of path vertices $\mathrm{p}_i$ include both null- and real-scattering events. We will find it useful to use $\mathrm{r}_i$ to denote the subset of them that represent real scattering (see Figure 14.5). Note a given real-scattering vertex $\mathrm{r}_i$ will generally have a different index value in the full path.

The path throughput function can then be defined as:

$$
T(\mathrm{p}_n)
= \left[\prod_{i=1}^{n}\bar S(\mathrm{p}_{i+1}\to\mathrm{p}_i\to\mathrm{p}_{i-1})\right]
  \left[\prod_{i=1}^{n}\hat G(\mathrm{p}_{i+1}\leftrightarrow\mathrm{p}_i)\right]
  \left[\prod_{i=1}^{m} \sigma_a(\mathrm{r}_i)\, L_e(\mathrm{r}_i\to\mathrm{r}_{i-1})\right].
\tag{14.11}
$$

It now refers to a generalized scattering distribution function and generalized geometric term $\hat G$. The former simply falls back to the `BSDF`, phase function (multiplied by $\sigma_s$), or a factor that enforces the ordering of null-scattering vertices, depending on the type of the vertex $\mathrm{p}_i$. Note that the first two products in Equation (14.11) are over all vertices but the third is only over real-scattering vertices.

**Figure 14.5 — volume path diagram.** In the path space framework, a path is defined by a set of $n$ vertices $\mathrm{p}_i$ with an emitter at one endpoint and a sensor at the other; intermediate vertices represent scattering events, including null scattering. The subset of $m$ vertices that represent real scattering events are labeled $\mathrm{r}_i$. The figure shows the camera $\mathrm{p}_0{=}\mathrm{r}_0$, a surface vertex $\mathrm{p}_1{=}\mathrm{r}_1$, a real medium scatter $\mathrm{p}_2{=}\mathrm{r}_2$, a null-scattering vertex $\mathrm{p}_3$, and the light vertex $\mathrm{p}_4{=}\mathrm{r}_3$.

```mermaid
flowchart LR
    CAM["p₀ = r₀<br/>(camera / sensor)"] -->|segment| SURF["p₁ = r₁<br/>(surface vertex)"]
    SURF -->|"segment through medium"| R2["p₂ = r₂<br/>(real volume scatter)"]
    R2 -->|segment| NULL["p₃<br/>(null scatter, V∅)"]
    NULL -->|segment| LIGHT["p₄ = r₃<br/>(light / emitter)"]
    classDef real fill:#dff,stroke:#393,stroke-width:1px;
    classDef nullv fill:#fdd,stroke:#933,stroke-width:1px;
    class CAM,SURF,R2,LIGHT real;
    class NULL nullv;
```

The scattering distribution function is defined by

$$
\bar S(\mathrm{p}_{i+1}\to\mathrm{p}_i\to\mathrm{p}_{i-1}) =
\begin{cases}
f(\mathrm{p}_{i+1}\to\mathrm{p}_i\to\mathrm{p}_{i-1}),
    & \mathrm{p}_i\in\mathcal A,\\[4pt]
\sigma_s(\mathrm{p}_i)\, p(\omega_{i-1},\omega_i),
    & \mathrm{p}_i\in\mathcal V,\\[4pt]
\sigma_n(\mathrm{p}_i)\, H\!\left(
    (\mathrm{p}_i-\mathrm{p}_{i-1})\cdot(\mathrm{p}_{i+1}-\mathrm{p}_i)\right),
    & \mathrm{p}_i\in\mathcal V_\varnothing.
\end{cases}
\tag{14.12}
$$

Here, $H$ is the Heaviside function, which is 1 if its parameter is positive and 0 otherwise. Equation (13.2) in Section 13.1.3 originally defined the geometric term $G$

$$
G(\mathrm{p}\leftrightarrow\mathrm{q})
= V(\mathrm{p}\leftrightarrow\mathrm{q})\,
  \frac{|\cos\theta_{\mathrm p}|\,|\cos\theta_{\mathrm q}|}{\|\mathrm{p}-\mathrm{q}\|^2}.
$$

A generalized form of this geometric term is given by

$$
\hat G(\mathrm{p}\leftrightarrow\mathrm{q})
= \frac{C_{\mathrm p}\, C_{\mathrm q}}{\|\mathrm{p}-\mathrm{q}\|^2}\,
  V(\mathrm{p}\leftrightarrow\mathrm{q}),
\tag{14.13}
$$

where

$$
C_{\mathrm p} =
\begin{cases}
|\cos\theta_{\mathrm p}|, & \mathrm{p}\in\mathcal A,\\
1,                        & \mathrm{p}\in\mathcal V,
\end{cases}
$$

incorporates the absolute angle cosine between the connection segment and the normal direction when the underlying vertex $\mathrm{p}$ is located on a surface. Note that $C_{\mathrm p}$ is only evaluated for real-scattering vertices $\mathrm{r}_i$, so the case of $\mathrm{p}\in\mathcal V_\varnothing$ does not need to be considered.

Similar to integrating over the path space for surface scattering, the Monte Carlo estimator for the path contribution function can be defined for a path $\mathrm{p}_n$ of $n$ path vertices $\mathrm{p}_i$. The resulting Monte Carlo estimator is

$$
\left\langle F(\mathrm{p}_n)\right\rangle
= \frac{F(\mathrm{p}_n)}{p(\mathrm{p}_n)},
\tag{14.14}
$$

where $p(\mathrm{p}_n)$ is the probability of sampling the path $\mathrm{p}_n$ with respect to the generalized path space measure. Following Equation (13.8), we will also find it useful to define the volumetric path throughput weight

$$
\beta(\mathrm{p}_n)
= \frac{1}{p(\mathrm{p}_n)}
  \prod_{i=1}^{n}\bar S_i\,\hat G_i.
\tag{14.15}
$$

Reference / index entries from the page margin: `FastExp()` 1036, `Float` 23, `Medium::SamplePoint()` 717, `MediumProperties` 718, `RayMajorantSegment::sigma_maj` 718, `RayMajorantSegment::t_max` 718, `SampledSpectrum` 171.

### 14.1.5 EVALUATING THE VOLUMETRIC PATH INTEGRAL

The Monte Carlo estimator of the null-scattering path integral from Equation (14.14) allows sampling path vertices in a variety of ways; it is not necessary to sample them incrementally from the camera as in path tracing, for example. We will now reconsider sampling paths via path tracing under the path integral framework to show its use. For simplicity, we will consider scenes that have only volumetric scattering here. The volumetric path-tracing algorithm from Section 14.1.2 is based on three sampling operations: sampling a distance along the current ray to a scattering event, choosing which type of interaction happens at that point, and then sampling a new direction from that point if the path has not been terminated. We can write the corresponding Monte Carlo estimator for the generalized path contribution function from Equation (14.14) with the path probability $p(\mathrm{p}_n)$ expressed as the product of three probabilities:

- $p_{\mathrm{maj}}(\mathrm{p}_{i+1}\mid\mathrm{p}_i,\omega_i)$: the probability of sampling the point $\mathrm{p}_{i+1}$ along the direction $\omega_i$ from the point $\mathrm{p}_i$.
- $p_e(\mathrm{p}_i)$: the discrete probability of sampling the type of scattering event—absorption, real-, or null-scattering—that was chosen at $\mathrm{p}_i$.
- $p_\omega(\omega'\mid\mathrm{r}_i,\omega_i)$: the probability of sampling the direction $\omega'$ after a regular scattering event at point $\mathrm{r}_i$ with incident direction $\omega_i$.

For an $n$ vertex path with $m$ real-scattering vertices, the resulting estimator is

$$
\frac{F(\mathrm{p}_n)}{p(\mathrm{p}_n)}
= \frac{\left[\prod_{i=1}^{n}\bar S_i\,\hat G_i\right]
        L_e^{\mathrm{path}}(\mathrm{p}_{n+1}\to\mathrm{p}_n)}
       {\left[\prod_{i=1}^{n} p_{\mathrm{maj},i}\, p_{e,i}\right]
        \left[\prod_{i=1}^{m} p_{\omega,i}\right]}.
\tag{14.16}
$$

Here $\omega_i$ denotes the direction from $\mathrm{p}_i$ to $\mathrm{p}_{i+1}$, and the $\hat G$ factor in the denominator accounts for the change of variables from sampling with respect to solid angle to sampling with respect to the path space measure. We consider each of the three sampling operations in turn, starting with distance sampling, which has density $p_{\mathrm{maj}}$. Assuming a single majorant $\sigma_{\mathrm{maj}}$, we find that $p_{\mathrm{maj}}$ has density

$$
p_{\mathrm{maj}}(\mathrm{p}_{i+1}\mid\mathrm{p}_i,\omega_i)
= \sigma_{\mathrm{maj}}\, e^{-\sigma_{\mathrm{maj}} d_i},
\qquad d_i = \|\mathrm{p}_{i+1}-\mathrm{p}_i\|,
$$

and the exponential factors cancel out the $T_{\mathrm{maj}}$ factors in $\hat G$, each one leaving behind a $1/\sigma_{\mathrm{maj}}$ factor. Expanding out and simplifying, including eliminating the $\hat G$ factors, all of which also cancel out, we have the estimator

$$
\frac{F(\mathrm{p}_n)}{p(\mathrm{p}_n)}
= \frac{\sigma_{\mathrm{maj}}^{n}\, L_e^{\mathrm{path}}(\mathrm{p}_{n+1}\to\mathrm{p}_n)}
       {\left[\prod_{i=1}^{n} p_{e,i}\right]\left[\prod_{i=1}^{m} p_{\omega,i}\right]}
  \prod_{i=1}^{m}\frac{p(\omega_i)}{\sigma_{\mathrm{maj}}}.
\tag{14.17}
$$

Consider next the discrete choice among the three types of scattering event. The probabilities $p_e$ are all of the form $\sigma_{\{a,s,n\}}/\sigma_{\mathrm{maj}}$, according to which type of scattering event was chosen at each vertex. The $(\sigma_{\mathrm{maj}})^n$ factor in Equation (14.17) cancels, leaving us with

$$
\frac{F(\mathrm{p}_n)}{p(\mathrm{p}_n)}
= \frac{L_e^{\mathrm{path}}(\mathrm{p}_{n+1}\to\mathrm{p}_n)}
       {\left[\prod_{i=1}^{n} \sigma_{\{a,s,n\}}(\mathrm{p}_i)\right]
        \left[\prod_{i=1}^{m} p_{\omega,i}\right]}
  \prod_{i=1}^{m} p(\omega_i).
\tag{14.18}
$$

The first $n-1$ $\sigma_{\{a,s,n\}}$ factors must be either real or null scattering, and the last must be $\sigma_a$, given how the path was sampled. Thus, the estimator is equivalent to

$$
\frac{F(\mathrm{p}_n)}{p(\mathrm{p}_n)}
= \frac{L_e(\mathrm{p}_{n+1}\to\mathrm{p}_n)}
       {\left[\prod_{i=1}^{n-1} \sigma_{\{s,n\}}(\mathrm{p}_i)\right]
        \left[\prod_{i=1}^{m} p_{\omega,i}\right]}
  \prod_{i=1}^{m} p(\omega_i).
$$

Because we are for now neglecting surface scattering, $\bar S$ represents either regular volumetric scattering or null scattering. Recall from Equation (14.12) that $\bar S$ includes a $\sigma_s$ or $\sigma_n$ factor in those respective cases, which cancels out all the corresponding factors in the $\sigma_{\{s,n\}}$ product in the denominator. Further, note that the Heaviside function for null scattering's $\bar S$ function is always 1 given how vertices are sampled with path tracing, so we can also restrict ourselves to the remaining $m$ real-scattering events in the numerator. Our estimator simplifies to

$$
\frac{F(\mathrm{p}_n)}{p(\mathrm{p}_n)}
= L_e(\mathrm{p}_n\to\mathrm{p}_{n-1})
  \prod_{i=1}^{m}\frac{p(\omega_i)}{p_{\omega}(\omega_i)}.
\tag{14.19}
$$

The $\sigma_a$ factor in the path space emission function, Equation (14.10), cancels the remaining $\sigma_a(\mathrm{p}_n)$. We are left with the emission $L_e(\mathrm{p}_n\to\mathrm{p}_{n-1})$ at the last vertex scaled by the product of ratios of phase function values and sampling probabilities as the estimator's value, just as we saw in Section 14.1.2.

## 14.2 VOLUME SCATTERING INTEGRATORS

The path space expression of the null-scattering equation of transfer allows a variety of sampling techniques to be applied to the light transport problem. This section defines two integrators that are based on path tracing starting from the camera.

First is the `SimpleVolPathIntegrator`, which uses simple sampling techniques, giving an implementation that is short and easily verified. This integrator is particularly useful for computing ground-truth results when debugging more sophisticated volumetric sampling and integration algorithms. The `VolPathIntegrator` is defined next. This integrator is fairly complex, but it applies state-of-the-art sampling techniques to volume light transport while handling surface scattering similarly to the `PathIntegrator`. It is pbrt's default integrator and is also the template for the wavefront integrator in Chapter 15.

Reference / index entries: `PathIntegrator` 833, `RandomWalkIntegrator` 33, `VolPathIntegrator` 877.

### 14.2.1 A SIMPLE VOLUMETRIC INTEGRATOR

The `SimpleVolPathIntegrator` implements a basic volumetric path tracer, following the sampling approach described in Section 14.1.2. Its `Li()` method is under 100 lines of code, none of them too tricky. However, with this simplicity comes a number of limitations. First, like the `RandomWalkIntegrator`, it does not perform any explicit light sampling, so it requires that rays are able to randomly intersect the lights in the scene. Second, it does not handle scattering from surfaces. An error message is therefore issued if it is used with a scene that contains delta distribution light sources or has surfaces with nonzero-valued `BSDFs`. (These defects are all addressed in the `VolPathIntegrator` discussed in Section 14.2.2.) Nevertheless, this integrator is capable of rendering complex volumetric effects; see Figure 14.6.

**Figure 14.6 (pictorial render, kept).** *Explosion rendered using the `SimpleVolPathIntegrator`.* With 256 samples per pixel, this integrator gives a reasonably accurate rendering of the volumetric model, though there are variance spikes in some pixels (especially visible toward the bottom of the volume) due to error from the integrator not directly sampling the scene's light sources. The `VolPathIntegrator`, which uses more sophisticated sampling strategies, renders this scene with 1,288 times lower MSE; it is discussed in Section 14.2.2. (Scene courtesy of Jim Price.)

<details>
<summary>Figure 14.6 — explosion render (pictorial)</summary>

![Figure 14.6](images/img-1375-322.png)

</details>

The integrator's definition:

```rust
// 〈SimpleVolPathIntegrator Definition〉
struct SimpleVolPathIntegrator {
    // 〈SimpleVolPathIntegrator Public Methods〉
    // 〈SimpleVolPathIntegrator Private Members〉
    max_depth: i32,
}
```

Reference / index entries: `RayIntegrator` 28, `SimpleVolPathIntegrator` 868, `VolPathIntegrator` 877.

This integrator's only parameter is the maximum path length, which is set via a value passed to the constructor (not included here).

```rust
// 〈SimpleVolPathIntegrator Private Members〉
max_depth: i32,
```

The general form of the `Li()` method follows that of the `PathIntegrator`.

```rust
// 〈SimpleVolPathIntegrator Method Definitions〉
fn li(
    &self,
    ray: RayDifferential,
    lambda: &mut SampledWavelengths,
    sampler: &mut Sampler,
    buf: &mut ScratchBuffer,
    _visible_surface: Option<&mut VisibleSurface>,
) -> SampledSpectrum {
    // 〈Declare local variables for delta tracking integration〉
    // 〈Terminate secondary wavelengths before starting random walk〉
    loop {
        // 〈Estimate radiance for ray path using delta tracking〉
    }
    l
}
```

A few familiar variables track the path state, including `l` to accumulate the radiance estimate for the path. For this integrator, `beta`, which tracks the path throughput weight, is just a single `Float` value, since the product of ratios of phase function values and sampling PDFs from Equation (14.19) is a scalar value.

```rust
// 〈Declare local variables for delta tracking integration〉
let mut l = SampledSpectrum::from(0.0);
let mut beta = 1.0_f32;
let mut depth = 0;
```

Media with scattering properties that vary according to wavelength introduce a number of complexities in sampling and evaluating Monte Carlo estimators. We will defer addressing them until we cover the `VolPathIntegrator`. The `SimpleVolPathIntegrator` instead estimates radiance at a single wavelength by terminating all but the first wavelength sample. Here is a case where we have chosen simplicity over efficiency for this integrator's implementation: we might instead have accounted for all wavelengths until the point that spectrally varying scattering properties were encountered, enjoying the variance reduction benefits of estimating all of them for scenes where doing so is possible. However, doing this would have led to a more complex integrator implementation.

Reference / index entries: `Float` 23, `PathIntegrator` 833, `RayDifferential` 96, `SampledSpectrum` 171, `SampledWavelengths` 173, `SampledWavelengths::TerminateSecondary()` 174, `Sampler` 469, `SampleT_maj()` 859, `ScratchBuffer` 1078, `VisibleSurface` 245, `VolPathIntegrator` 877.

```rust
// 〈Terminate secondary wavelengths before starting random walk〉
lambda.terminate_secondary();
```

The first step in the loop is to find the ray's intersection with the scene geometry, if any. This gives the parametric distance `t` beyond which no samples should be taken for the current ray, as the intersection either represents a transition to a different medium or a surface that occludes farther-away points. The `scattered` and `terminated` variables declared here will allow the lambda function that is passed to `SampleT_maj()` to report back the state of the path after sampling terminates.

```rust
// 〈Estimate radiance for ray path using delta tracking〉
let si: Option<ShapeIntersection> = self.intersect(ray);
let mut scattered = false;
let mut terminated = false;
if ray.medium.is_some() {
    // 〈Initialize RNG for sampling the majorant transmittance〉
    // 〈Sample medium using delta tracking〉
}
// 〈Handle terminated and unscattered rays after medium sampling〉
```

An `Rng` is required for the call to the `SampleT_maj()` function. We derive seeds for it based on two random values from the sampler, hashing them to convert `Float`s into integers.

```rust
// 〈Initialize RNG for sampling the majorant transmittance〉
let hash0 = hash(sampler.get_1d());
let hash1 = hash(sampler.get_1d());
let mut rng = Rng::new(hash0, hash1);
```

With that, a call to `SampleT_maj()` starts the generation of samples according to $\sigma_{\mathrm{maj}}T_{\mathrm{maj}}$. The `Sampler` is used to generate the first uniform sample `u` that is passed to the method; recall from Section 14.1.3 that subsequent ones will be generated using the provided `Rng`. In a similar fashion, the `Sampler` is used for the initial value of `u_mode` here. It will be used to choose among the three types of scattering event at the first sampled point. For `u_mode` as well, the `Rng` will provide subsequent values. In this case, the transmittance that `SampleT_maj()` returns for the final segment is unneeded, so it is ignored.

```rust
// 〈Sample medium using delta tracking〉
let t_max = si.as_ref().map_or(INFINITY, |i| i.t_hit);
let u = sampler.get_1d();
let mut u_mode = sampler.get_1d();
sample_t_maj(ray, t_max, u, &mut rng, lambda,
    |p: Point3f, mp: MediumProperties,
     sigma_maj: SampledSpectrum, t_maj: SampledSpectrum| {
        // 〈Compute medium event probabilities for interaction〉
        // 〈Randomly sample medium scattering event for delta tracking〉
    });
```

For each sample returned by `SampleT_maj()`, it is necessary to select which type of scattering it represents. The first step is to compute the probability of each possibility. Because we have specified $\sigma_n$ such that it is nonnegative and $\sigma_a+\sigma_s+\sigma_n=\sigma_{\mathrm{maj}}$, the null-scattering probability can be found as one minus the other two probabilities. A call to `max()` ensures that any slightly negative values due to floating-point round-off error are clamped at zero.

Reference / index entries: `Float` 23, `Hash()` 1042, `Infinity` 361, `Integrator::Intersect()` 23, `MediumProperties` 718, `MediumProperties::sigma_a` 718, `MediumProperties::sigma_s` 718, `Point3f` 92, `Ray::medium` 95, `RNG` 1054, `SampledSpectrum` 171, `Sampler` 469, `Sampler::Get1D()` 470, `SampleT_maj()` 859, `ShapeIntersection` 266, `ShapeIntersection::tHit` 266.

```rust
// 〈Compute medium event probabilities for interaction〉
let p_absorb = mp.sigma_a[0] / sigma_maj[0];
let p_scatter = mp.sigma_s[0] / sigma_maj[0];
let p_null = (1.0 - p_absorb - p_scatter).max(0.0);
```

A call to `SampleDiscrete()` then selects one of the three terms of $L_n$ using the specified probabilities.

```rust
// 〈Randomly sample medium scattering event for delta tracking〉
let mode = sample_discrete(&[p_absorb, p_scatter, p_null], u_mode);
if mode == 0 {
    // 〈Handle absorption event for medium sample〉
} else if mode == 1 {
    // 〈Handle regular scattering event for medium sample〉
} else {
    // 〈Handle null-scattering event for medium sample〉
}
```

If absorption is chosen, the path terminates. Any emission is added to the radiance estimate, and evaluation of Equation (14.19) is complete. The fragment therefore sets `terminated` to indicate that the path is finished and returns false from the lambda function so that no further samples are generated along the ray.

```rust
// 〈Handle absorption event for medium sample〉
l += beta * mp.le;
terminated = true;
return false;
```

For a scattering event, `beta` is updated according to the ratio of phase function and its directional sampling probability from Equation (14.19).

```rust
// 〈Handle regular scattering event for medium sample〉
// 〈Stop path sampling if maximum depth has been reached〉
// 〈Sample phase function for medium scattering event〉
// 〈Update state for recursive evaluation of Li〉
```

The counter for the number of scattering events is only incremented for real-scattering events; we do not want the number of null-scattering events to affect path termination. If this scattering event causes the limit to be reached, the path is terminated.

```rust
// 〈Stop path sampling if maximum depth has been reached〉
depth += 1;
if depth - 1 >= self.max_depth {
    terminated = true;
    return false;
}
```

If the path is not terminated, then a new direction is sampled from the phase function's distribution.

```rust
// 〈Sample phase function for medium scattering event〉
let u2 = Point2f::new(rng.uniform::<f32>(), rng.uniform::<f32>());
let ps = match mp.phase.sample_p(-ray.d, u2) {
    Some(s) => s,
    None => { terminated = true; return false; }
};
```

Reference / index entries: `Float` 23, `MediumProperties::Le` 718, `MediumProperties::phase` 718, `PhaseFunction::Sample_p()` 711, `PhaseFunctionSample` 711, `Point2f` 92, `Ray::d` 95, `RNG::Uniform<Float>()` 1056, `SampleDiscrete()` 70, `SimpleVolPathIntegrator::maxDepth` 869.

Given a sampled direction, the `beta` factor must be updated. Volumetric path-tracing implementations often assume that the phase function sampling distribution matches the phase function's actual distribution and dispense with `beta` entirely since it is always equal to 1. This variation is worth pausing to consider: in that case, emitted radiance at the end of the path is always returned, unscaled. All of the effect of transmittance, phase functions, and so forth is entirely encapsulated in the distribution of how often various terms are evaluated and in the distribution of scattered ray directions. pbrt does not impose the requirement on phase functions that their importance sampling technique be perfect, though this is the case for the Henyey–Greenstein phase function in pbrt.

Be it with `beta` or without, there is no need to do any further work along the current ray after a scattering event, so after the following code updates the path state to account for scattering, it too returns false to direct that no further samples should be taken along the ray.

```rust
// 〈Update state for recursive evaluation of Li〉
beta *= ps.p / ps.pdf;
ray.o = p;
ray.d = ps.wi;
scattered = true;
return false;
```

Null-scattering events are ignored, so there is nothing to do but to return true to indicate that additional samples along the current ray should be taken. Similar to the real-scattering case, this can be interpreted as starting a recursive evaluation of Equation (14.3) from the current sampled position without incurring the overhead of actually doing so. Since this is the only case that may lead to another invocation of the lambda function, `u_mode` must be refreshed with a new uniform sample value in case another sample is generated.

```rust
// 〈Handle null-scattering event for medium sample〉
u_mode = rng.uniform::<f32>();
return true;
```

If the path was terminated due to absorption, then there is no more work to do in the `Li()` method; the final radiance value can be returned. Further, if the ray was scattered, then there is nothing more to do but to restart the while loop and start sampling the scattered ray. Otherwise, the ray either underwent no scattering events or only underwent null scattering.

```rust
// 〈Handle terminated and unscattered rays after medium sampling〉
if terminated { return l; }
if scattered { continue; }
// 〈Add emission to surviving ray〉
// 〈Handle surface intersection along ray path〉
```

If the ray is unscattered and unabsorbed, then any emitters it interacts with contribute radiance to the path. Either surface emission or emission from infinite light sources is accounted for, depending on whether an intersection with a surface was found. Further, if the ray did not intersect a surface, then the path is finished and the radiance estimate can be returned.

```rust
// 〈Add emission to surviving ray〉
if let Some(hit) = &si {
    l += beta * hit.intr.le(-ray.d, lambda);
} else {
    for light in &self.infinite_lights {
        l += beta * light.le(ray, lambda);
    }
    return l;
}
```

Reference / index entries: `Float` 23, `Integrator::infiniteLights` 23, `Light::Le()` 743, `PhaseFunctionSample::p` 711, `PhaseFunctionSample::pdf` 711, `PhaseFunctionSample::wi` 711, `Ray::d` 95, `Ray::o` 95, `RNG::Uniform<Float>()` 1056, `ShapeIntersection::intr` 266, `SurfaceInteraction::Le()` 762.

It is still necessary to consider surface intersections, even if scattering from them is not handled by this integrator. There are three cases to consider: If the surface has no `BSDF`, it represents a transition between different types of participating media. A call to `SkipIntersection()` moves the ray past the intersection and updates its medium appropriately. If there is a valid `BSDF` and that `BSDF` also returns a valid sample from `Sample_f()`, then we have a `BSDF` that scatters; an error is issued and rendering stops. A valid but zero-valued `BSDF` is allowed; such a `BSDF` should be assigned to area light sources in scenes to be rendered using this integrator.

```rust
// 〈Handle surface intersection along ray path〉
let hit = si.as_ref().unwrap();
let bsdf = hit.intr.get_bsdf(ray, lambda, camera, buf, sampler);
if bsdf.is_none() {
    hit.intr.skip_intersection(&mut ray, hit.t_hit);
} else {
    // 〈Report error if BSDF returns a valid sample〉
}
```

The complete delta-tracking loop in WGSL, with the callback inlined, makes the data flow explicit for a GPU implementation:

```wgsl
fn simple_vol_path_li(
    ray: ptr<function, Ray>,
    lambda: ptr<function, SampledWavelengths>,
    max_depth: u32,
) -> vec3<f32> {
    var l = vec3<f32>(0.0);
    var beta = 1.0;
    var depth = 0u;

    loop {
        let si = intersect_scene(ray);
        if ((*ray).medium == MEDIUM_NONE) {
            // No medium: emit from the surface or an infinite light.
            if (si.hit) { l += beta * surface_le(si); }
            else { l += beta * infinite_le(ray); }
            break;
        }

        var iter = make_majorant_iterator((*ray), si.t_hit);
        var u = sampler_1d();
        var u_mode = sampler_1d();
        let ms = sample_t_maj_wgsl(
            (*ray).o, (*ray).d, si.t_hit, u, lambda.lambda[0], &iter);

        let p_absorb = ms.mp.sigma_a[0] / ms.sigma_maj[0];
        let p_scatter = ms.mp.sigma_s[0] / ms.sigma_maj[0];
        let p_null = max(0.0, 1.0 - p_absorb - p_scatter);
        let mode = sample_discrete3(p_absorb, p_scatter, p_null, u_mode);

        if (mode == EVENT_ABSORB) {
            l += beta * ms.mp.le;
            break;
        } else if (mode == EVENT_SCATTER) {
            depth += 1u;
            if (depth > max_depth) { break; }
            let ps = sample_hg(ms.mp.g, -(*ray).d, vec2f(next_uniform(), next_uniform()));
            beta *= ps.p / ps.pdf;
            (*ray).o = ms.p;
            (*ray).d = ps.wi;
            continue;
        }
        // EVENT_NULL: keep sampling the same ray (loop repeats with new iter state).
    }
    return l;
}
```

### 14.2.2 IMPROVING THE SAMPLING TECHNIQUES

The `VolPathIntegrator` adds three significant improvements to the approach implemented in `SimpleVolPathIntegrator`: it supports scattering from surfaces as well as from volumes; it handles spectrally varying medium scattering properties without falling back to sampling a single wavelength; and it samples lights directly, using multiple importance sampling to reduce variance when doing so. The first improvement—including surface scattering—is mostly a matter of applying the ideas of Equation (14.7), sampling distances in volumes but then choosing surface scattering if the sampled distance is past the closest intersection. For the other two, we will here discuss the underlying foundations before turning to their implementation.

**Chromatic Media.** We have thus far glossed over some of the implications of spectrally varying medium properties. Because pbrt uses point-sampled spectra, they introduce no complications in terms of evaluating things like the modified path throughput or the path throughput weight $\beta(\mathrm{p}_n)$: given a set of path vertices, such quantities can be evaluated for all the wavelength samples simultaneously using the `SampledSpectrum` class.

The problem with spectrally varying medium properties comes from sampling. Consider a wavelength-dependent function $f_\lambda(x)$ that we would like to integrate at $n$ wavelengths $\lambda_i$. If we draw samples from a wavelength-dependent PDF based on the first wavelength and then evaluate $f$ at all the wavelengths, we have the estimators

$$
\left\langle I_{\lambda_i}\right\rangle = \frac{f_{\lambda_i}(x)}{p_{\lambda_1}(x)}.
$$

Even if the PDF that was used for sampling matches well, it may be a poor match for $f$ at the other wavelengths. It may not even be a valid PDF for them, if it is zero-valued where the function is nonzero. However, falling back to integrating a single wavelength at a time would be unfortunately inefficient, as shown in Section 4.5.4.

Reference / index entries: `BSDF` 544, `SampledSpectrum` 171, `ShapeIntersection::intr` 266, `ShapeIntersection::tHit` 266, `SimpleVolPathIntegrator` 868, `SurfaceInteraction::GetBSDF()` 682, `SurfaceInteraction::SkipIntersection()` 643.

This problem of a single sampling PDF possibly mismatched with a wavelength-dependent function comes up repeatedly in volumetric path tracing. For example, sampling the majorant transmittance at one wavelength may be a poor approach for sampling it at others. That could be handled by selecting a majorant that bounds all wavelengths' extinction coefficients, but such a majorant would lead to many null-scattering events at wavelengths that could have used a much lower majorant, which would harm performance. The path tracer's choice among absorption, real scattering, and null scattering at a sampled point cannot be sidestepped in a similar way: different wavelengths may have quite different probabilities for each of these types of medium interaction, yet with path tracing the integrator must choose only one of them. Splitting up the computation to let each wavelength choose individually would be nearly as inefficient as only considering a single wavelength at a time. However, if a single type of interaction is chosen based on a single wavelength and we evaluate the modified path contribution function for all wavelengths, we could have arbitrarily high variance in the other wavelengths. To see why, note how all the $\sigma_{\{s,n\}}$ factors that came from the $p_e(\mathrm{p}_i)$ factors in Equation (14.18) canceled out to give the delta-tracking estimator, Equation (14.19). In the spectral case, if, for example, real scattering is chosen based on a wavelength $\lambda$'s scattering coefficient $\sigma_{s,\lambda}$ and if a wavelength $\lambda'$ has scattering coefficient $\sigma_{s,\lambda'}$, then the final estimator for $\lambda'$ will include a factor of $\sigma_{s,\lambda'}/\sigma_{s,\lambda}$ that can be arbitrarily large. The fact that `SampleT_maj()` nevertheless samples according to a single wavelength's majorant transmittance suggests that there is a solution to this problem. That solution, yet again, is multiple importance sampling. In this case, we are using a single sampling technique rather than MIS-weighting multiple techniques, so we use the single-sample MIS estimator from Equation (2.16), which here gives

$$
\left\langle I_\lambda\right\rangle = \frac{1}{q}\,\frac{f_\lambda(x)}{p_{\lambda_1}(x)},
$$

where $q$ is the discrete probability of sampling using the wavelength $\lambda_1$, here uniform at $1/n$ with $n$ the number of spectral samples.

The balance heuristic is optimal for single-sample MIS. It gives the MIS weight

$$
w_{\lambda_1}(x) = \frac{q\,p_{\lambda_1}(x)}{\sum_{j=1}^{n} q_j\, p_{\lambda_j}(x)}
                = \frac{p_{\lambda_1}(x)}{\sum_{j=1}^{n} p_{\lambda_j}(x)},
$$

which gives the estimator

$$
\left\langle I_\lambda\right\rangle
= \frac{1}{n}\sum_{i=1}^{n} \frac{p_{\lambda_i}(x)}{\sum_j p_{\lambda_j}(x)}\,
  \frac{f_\lambda(x)}{p_{\lambda_i}(x)}
= \frac{f_\lambda(x)}{\frac{1}{n}\sum_{j=1}^{n} p_{\lambda_j}(x)}.
$$

See Figure 14.7 for an example that shows the benefits of MIS for chromatic media.

**Figure 14.7 (pictorial render, kept).** *Chromatic Volumetric Media.* (a) When rendered without spectral MIS, variance is high. (b) Results are much better with spectral MIS, as implemented in the `VolPathIntegrator`. For this scene, MSE is reduced by a factor of 149. (Scene courtesy of Jim Price.)

<details>
<summary>Figure 14.7 — chromatic media comparison (a) without and (b) with spectral MIS</summary>

![Figure 14.7](images/img-1388-323.png)

</details>

**Direct Lighting.** Multiple importance sampling is also at the heart of how the `VolPathIntegrator` samples direct illumination. As with the `PathIntegrator`, we would like to combine the strategies of sampling the light sources with sampling the `BSDF` or phase function to find light-carrying paths and then to weight the contributions of each sampling technique using MIS. Doing so is more complex than it is in the absence of volumetric scattering, however, because not only does the sampling distribution used at the last path vertex differ (as before) but the `VolPathIntegrator` also uses ratio tracking to estimate the transmittance along the shadow ray. That is a different distance sampling technique than the delta-tracking approach used when sampling ray paths, and so it leads to a different path PDF.

Reference / index entries: `PathIntegrator` 833, `SampleT_maj()` 859, `VolPathIntegrator` 877.

In the following, we will say that the two path-sampling techniques used in the `VolPathIntegrator` are *unidirectional path sampling* and *light path sampling*; we will write their respective path PDFs as $p_u$ and $p_l$. The first corresponds to the sampling approach from Section 14.1.5, with delta tracking used to find real-scattering vertices and with the phase function or `BSDF` sampled to find the new direction at each vertex. Light path sampling follows the same approach up to the last real-scattering vertex before the light vertex; there, the light selects the direction and then ratio tracking gives the transmittance along the last path segment. (See Figure 14.8.)

Given a path $\mathrm{p}_{n-1}$, both approaches share the same path throughput weight $\beta$ up to the vertex $\mathrm{p}_{n-1}$ and the same path PDF up to that vertex, $p_u(\mathrm{p}_{n-1}).$^2

**Figure 14.8 — direct-lighting path diagram.** In the direct lighting calculation, at each path vertex a point is sampled on a light source and a shadow ray (dotted line below) is traced. The `VolPathIntegrator` uses ratio tracking to compute the transmittance along the ray by accumulating the product $\sigma_n/\sigma_{\mathrm{maj}}$ at sampled points along the ray (open circles). For the MIS weight, it is necessary to be able not only to compute the PDF for sampling the corresponding direction at the last path vertex but also to compute the probability of generating these samples using delta tracking, since that is how the path would be sampled with unidirectional path sampling.

```mermaid
flowchart LR
    CAM["camera"] --> V["real-scatter vertex p_(n-1)"]
    V -->|"BSDF / phase sample"| LIGHT_A["light p_n (unidirectional path)"]
    V -->|"shadow ray: ratio tracking"| N1["null sample ○"]
    N1 --> N2["null sample ○"]
    N2 -->|"∏(σ_n / σ_maj) · T_maj"| LIGHT_B["light p_n (light path)"]
```

For the full PDF for unidirectional path sampling, at the last scattering vertex we have the probability of scattering, $\sigma_s(\mathrm{p}_{n-1})/\sigma_{\mathrm{maj}}$, times the directional probability for sampling the new direction $p_\omega(\omega_n)$, which is given by the sampling strategy used for the `BSDF` or phase function. Then, for the path to find an emitter at the vertex $\mathrm{p}_n$, it must have only sampled null-scattering vertices between $\mathrm{p}_{n-1}$ and $\mathrm{p}_n$; absorption or a real-scattering vertex preclude making it to $\mathrm{p}_n$. Using the results from Section 14.1.5, we can find that the path PDF between two points $\mathrm{p}_i$ and $\mathrm{p}_j$ with $m$ intermediate null-scattering vertices indexed by $k$ is given by the product of $\sigma_n(\mathrm{p}_k)/\sigma_{\mathrm{maj}}$ for all null-scattering vertices. The $\sigma_{\mathrm{maj}}$ factors cancel and the null-scattering path probability is

$$
p_{\mathrm{null}}(\mathrm{p}_i\to\mathrm{p}_j)
= \left[\prod_{k=1}^{m} \sigma_n(\mathrm{p}_k)\right]\,
  T_{\mathrm{maj}}(\mathrm{p}_j\to\mathrm{p}_i).
\tag{14.20}
$$

The full unidirectional path probability is then given by

$$
p_u(\mathrm{p}_n)
= p_u(\mathrm{p}_{n-1})\,
  \frac{\sigma_s(\mathrm{p}_{n-1})}{\sigma_{\mathrm{maj}}}\,
  p_\omega(\omega_n)\,
  p_{\mathrm{null}}(\mathrm{p}_{n-1}\to\mathrm{p}_n).
\tag{14.21}
$$

For light sampling, we again have the discrete probability $\sigma_s(\mathrm{p}_{n-1})/\sigma_{\mathrm{maj}}$ for scattering at $\mathrm{p}_{n-1}$ but the directional PDF at the vertex is determined by the light's sampling distribution, which we will denote by $p_{l,\omega}(\omega_n)$. The only missing piece is the PDF of the last segment (the shadow ray), where ratio tracking is used. In that case, points are sampled according to the majorant transmittance and so the PDF for a path sampled between points $\mathrm{p}_i$ and $\mathrm{p}_j$ with $m$ intermediate vertices is

$$
p_{\mathrm{ratio}}(\mathrm{p}_i\to\mathrm{p}_j)
= \left[\prod_{k=1}^{m} \frac{\sigma_n(\mathrm{p}_k)}{\sigma_{\mathrm{maj}}}\right]\,
  T_{\mathrm{maj}}(\mathrm{p}_j\to\mathrm{p}_i)\,\sigma_{\mathrm{maj}},
\tag{14.22}
$$

and the full light sampling path PDF is given by

$$
p_l(\mathrm{p}_n)
= p_u(\mathrm{p}_{n-1})\,
  \frac{\sigma_s(\mathrm{p}_{n-1})}{\sigma_{\mathrm{maj}}}\,
  p_{l,\omega}(\omega_n)\,
  p_{\mathrm{ratio}}(\mathrm{p}_{n-1}\to\mathrm{p}_n).
\tag{14.23}
$$

The `VolPathIntegrator` samples both types of paths according to the first wavelength $\lambda_1$ but evaluates these PDFs at all wavelengths so that MIS over wavelengths can be used. Given the path $\mathrm{p}_n$ sampled using unidirectional path sampling and then the path sampled using light path sampling, the two-sample MIS estimator is

$$
I = \frac{F(\mathrm{p}_n)\, w_u(\mathrm{p}_n)}{p_u(\mathrm{p}_n)}
  + \frac{F(\mathrm{p}_n^l)\, w_l(\mathrm{p}_n^l)}{p_l(\mathrm{p}_n^l)}.
\tag{14.24}
$$

Note that because the paths share the same vertices for all of $\mathrm{p}_{n-1}$, not only do the two factors share common factors, but $p_u$ and $p_l$ do as well, following Equations (14.21) and (14.23).

In this case, the MIS weights can account not only for the differences between unidirectional and light path sampling but also for the different per-wavelength probabilities for each sampling strategy. For example, with the balance heuristic, the MIS weight for the unidirectional strategy works out to be

$$
w_u(\mathrm{p}_n) = \frac{p_u(\mathrm{p}_n)}
                         {p_u(\mathrm{p}_n) + p_l(\mathrm{p}_n)},
\tag{14.25}
$$

with $m$ the number of spectral samples. The MIS weight for light sampling is equivalent, but with the function in the numerator replaced with $p_l$.

### 14.2.3 IMPROVED VOLUMETRIC INTEGRATOR

The `VolPathIntegrator` pulls together all of these ideas to robustly handle both surface and volume transport. See Figures 14.9 and 14.10 for images rendered with this integrator that show off the visual complexity that comes from volumetric emission, chromatic media, and multiple scattering in participating media.

```rust
// 〈VolPathIntegrator Definition〉
struct VolPathIntegrator {
    // 〈VolPathIntegrator Public Methods〉
    // 〈VolPathIntegrator Private Methods〉
    // 〈VolPathIntegrator Private Members〉
    max_depth: i32,
    light_sampler: LightSampler,
    regularize: bool,
}
```

As with the other `Integrator` constructors that we have seen so far, the `VolPathIntegrator` constructor does not perform any meaningful computation, but just initializes member variables with provided values. These three are all equivalent to their parallels in the `PathIntegrator`.

```rust
// 〈VolPathIntegrator Private Members〉
max_depth: i32,
light_sampler: LightSampler,
regularize: bool,
```

Reference / index entries: `LightSampler` 781, `PathIntegrator` 833, `RayIntegrator` 28, `VolPathIntegrator` 877.

**Figure 14.9 (pictorial render, kept).** *Volumetric Emission inside Lightbulbs.* The flames in each lightbulb are modeled with participating media and rendered with the `VolPathIntegrator`. (Scene courtesy of Jim Price.)

<details>
<summary>Figure 14.9 — lightbulbs with volumetric flames (pictorial)</summary>

![Figure 14.9](images/img-1392-325.png)

</details>

**Figure 14.10 (pictorial render, kept).** *Volumetric Scattering in Liquid.* Scattering in the paint-infused water is modeled with participating media and rendered with the `VolPathIntegrator`. (Scene courtesy of Angelo Ferretti.)

<details>
<summary>Figure 14.10 — paint-infused water (pictorial)</summary>

![Figure 14.10](images/img-1393-326.png)

</details>

```rust
// 〈VolPathIntegrator Method Definitions〉
fn li(
    &self,
    mut ray: RayDifferential,
    lambda: &mut SampledWavelengths,
    sampler: &mut Sampler,
    scratch: &mut ScratchBuffer,
    visible_surf: Option<&mut VisibleSurface>,
) -> SampledSpectrum {
    // 〈Declare state variables for volumetric path sampling〉
    loop {
        // 〈Sample segment of volumetric scattering path〉
    }
    l
}
```

Reference / index entries: `RayDifferential` 96, `SampledSpectrum` 171, `SampledWavelengths` 173, `Sampler` 469, `ScratchBuffer` 1078, `VisibleSurface` 245, `VolPathIntegrator` 877.

There is a common factor of $p_u(\mathrm{p}_n)$ in the denominator of the first term of the two-sample MIS estimator, Equation (14.24), and the numerator of the MIS weights, Equation (14.25). There is a corresponding factor in the second term of the estimator and in the $w_l$ weight. It is tempting to cancel these out; in that case, the path state to be tracked by the integrator would consist of $F(\mathrm{p}_n)/p_u(\mathrm{p}_n)$ and the wavelength-dependent probabilities $p_u(\mathrm{p}_n)$ and $p_l(\mathrm{p}_n)$. Doing so is mathematically valid and would provide all the quantities necessary to evaluate Equation (14.24), but suffers from the problem that the quantities involved may overflow or underflow the range of representable floating-point values. To understand the problem, consider a highly specular surface—the `BSDF` will have a large value for directions around its peak, but the PDF for sampling those directions will also be large. That causes no problems in the `PathIntegrator`, since its `beta` variable tracks their ratio, which ends up being close to 1. However, with $F/p_u$ maintained independently, a series of specular bounces could lead to overflow. (Many null-scattering events along a path can cause similar problems.) Therefore, the `VolPathIntegrator` tracks the path throughput weight for the sampled path

$$
\beta(\mathrm{p}_n) = \frac{F(\mathrm{p}_n)}
                           {L_e^{\mathrm{path}}\, p_{\mathrm{path}}(\mathrm{p}_n)},
$$

which is numerically well behaved. Directly tracking the probabilities $p_u(\mathrm{p}_n)$ and $p_l(\mathrm{p}_n)$ would also stress the range of floating-point numbers, so instead it tracks the rescaled path probabilities

$$
r_u(\mathrm{p}_n) = \beta(\mathrm{p}_n)\, p_u(\mathrm{p}_n),
\qquad
r_l(\mathrm{p}_n) = \beta(\mathrm{p}_n)\, p_l(\mathrm{p}_n).
\tag{14.26}
$$

Here $p_{\mathrm{path}}(\mathrm{p}_n)$ is the probability for sampling the current path. It is equal to the light path probability for paths that end with a shadow ray from light path sampling and the unidirectional path probability otherwise. (Later in the implementation, we will take advantage of the fact that these two probabilities are the same until the last scattering vertex, which in turn implies that whichever of them is chosen for $p_{\mathrm{path}}$ does not affect the values of $r_u$ and $r_l$.) These rescaled path probabilities are all easily incrementally updated during path sampling. If $\beta=0$, then MIS weights like those in Equation (14.25) can be found with

$$
w_u = \frac{r_u}{r_u + r_l},
\tag{14.27}
$$

and similarly for $w_l$ when $r_l=0$.

The remaining variables in the following fragment have the same function as the variables of the same names in the `PathIntegrator`.

```rust
// 〈Declare state variables for volumetric path sampling〉
let mut l = SampledSpectrum::from(0.0);
let mut beta = SampledSpectrum::from(1.0);
let mut r_u = SampledSpectrum::from(1.0);
let mut r_l = SampledSpectrum::from(1.0);
let mut specular_bounce = false;
let mut any_non_specular_bounces = false;
let mut prev_intr_context: LightSampleContext = LightSampleContext::default();
let mut depth = 0;
let mut eta_scale = 1.0;
```

The while loop for each ray segment starts out similarly to the corresponding loop in the `SimpleVolPathIntegrator`: the integrator traces a ray to find a `t_max` value at the closest surface intersection before sampling the medium, if any, between the ray origin and the intersection point.

```rust
// 〈Sample segment of volumetric scattering path〉
let si: Option<ShapeIntersection> = self.intersect(ray);
if ray.medium.is_some() {
    // 〈Sample the participating medium〉
}
// 〈Handle surviving unscattered rays〉
```

The form of the fragment for sampling the medium is similar as well: `t_max` is set using the ray intersection `t`, if available, and an `Rng` is prepared before medium sampling proceeds. If the path is terminated or undergoes real scattering in the medium, then no further work is done to sample surface scattering at a ray intersection point.

```rust
// 〈Sample the participating medium〉
let mut scattered = false;
let mut terminated = false;
let t_max = si.as_ref().map_or(INFINITY, |i| i.t_hit);
// 〈Initialize RNG for sampling the majorant transmittance〉
let mut rng = Rng::new(hash(sampler.get_1d()), hash(sampler.get_1d()));
let t_maj = sample_t_maj(ray, t_max, sampler.get_1d(), &mut rng, lambda,
    |p: Point3f, mp: MediumProperties,
     sigma_maj: SampledSpectrum, t_maj: SampledSpectrum| {
        // 〈Handle medium scattering event for ray〉
    });
// 〈Handle terminated, scattered, and unscattered medium rays〉
```

Given a sampled point $\mathrm{p}'$ in the medium, the lambda function's task is to evaluate the $L_n$ source function, taking care of the second case of Equation (14.7).

```rust
// 〈Handle medium scattering event for ray〉
// 〈Add emission from medium scattering event〉
// 〈Compute medium event probabilities for interaction〉
// 〈Sample medium scattering event type and update path〉
```

A small difference from the `SimpleVolPathIntegrator` is that volumetric emission is added at every point that is sampled in the medium rather than only when the absorption case is sampled. There is no reason not to do so, since emission is already available via the `MediumProperties` passed to the lambda function.

```rust
// 〈Add emission from medium scattering event〉
if depth < self.max_depth && mp.le != 0.0 {
    // 〈Compute β′ at new path vertex〉
    // 〈Compute rescaled path probability for absorption at path vertex〉
    // 〈Update L for medium emission〉
}
```

Here we use the notation $[\mathrm{p}_n + \mathrm{p}']$ to denote the path $\mathrm{p}_n$ with the vertex $\mathrm{p}'$ appended to it. Thus, for example, $\mathrm{p}_n = [\mathrm{p}_{n-1} + \mathrm{p}_n]$. The estimator that gives the contribution for volumetric emission at $\mathrm{p}'$ is then

$$
\left\langle L_e\right\rangle
= \frac{\beta'\,\sigma_a(\mathrm{p}')\, L_e(\mathrm{p}')}
        {r_e},\qquad
\beta' = \beta\,\frac{T_{\mathrm{maj}}}{p_{\mathrm{maj}}}.
\tag{14.28}
$$

Reference / index entries: `Float` 23, `Infinity` 361, `MediumProperties` 718, `MediumProperties::Le` 718, `Point3f` 92, `RNG` 1054, `SampledSpectrum` 171, `Sampler::Get1D()` 470, `SampleT_maj()` 859, `ShapeIntersection::tHit` 266, `SimpleVolPathIntegrator` 868, `VolPathIntegrator::maxDepth` 877.

`beta` holds $\beta(\mathrm{p}_n)$, so we can incrementally compute $\beta([\mathrm{p}_n + \mathrm{p}'])$ by

$$
\beta' = \beta\,\frac{T_{\mathrm{maj}}}{p_{\mathrm{maj}}}.
$$

From Section 14.1.5, we know that $p_{\mathrm{maj}} = \sigma_{\mathrm{maj}}T_{\mathrm{maj}}$. Because we are always sampling absorption (at least as far as including emission goes), $p_e$ is 1 here.

```rust
// 〈Compute β′ at new path vertex〉
let pdf = sigma_maj[0] * t_maj[0];
let betap = beta * t_maj / pdf;
```

Even though this is the only sampling technique for volumetric emission, different wavelengths may sample this vertex with different probabilities, so it is worthwhile to apply MIS over the wavelengths' probabilities. With `r_u` storing the rescaled unidirectional probabilities up to the previous path vertex, the rescaled path probabilities for sampling the emissive vertex, `r_e`, can be found by multiplying `r_u` by the per-wavelength $p_{\mathrm{maj}}$ probabilities and dividing by the probability for the wavelength that was used for sampling $\mathrm{p}'$, which is already available in `pdf`. (Note that in monochromatic media, these ratios are all 1.)

```rust
// 〈Compute rescaled path probability for absorption at path vertex〉
let r_e = r_u * sigma_maj * t_maj / pdf;
```

Here we have a single-sample MIS estimator with balance heuristic weights given by

$$
w_e = \frac{r_e}{\frac{1}{n}\sum_{j=1}^{n} r_{e,j}}.
\tag{14.29}
$$

The absorption coefficient and emitted radiance for evaluating Equation (14.28) are available in `MediumProperties` and the `SampledSpectrum::average()` method conveniently computes the average of rescaled probabilities in the denominator of Equation (14.29).

```rust
// 〈Update L for medium emission〉
if r_e != 0.0 {
    l += betap * mp.sigma_a * mp.le / r_e.average();
}
```

Briefly returning to the initialization of `betap` and `r_e` in the previous fragments: it may seem tempting to cancel out the $T_{\mathrm{maj}}$ factors from them, but note how the final estimator does not perform a component-wise division of these two quantities but instead divides by the average of the rescaled probabilities when computing the MIS weight. Thus, performing such cancellations would lead to incorrect results.[^14-3] After emission is handled, the next step is to determine which term of $L_n$ to evaluate; this follows the same approach as in the `SimpleVolPathIntegrator`.

```rust
// 〈Sample medium scattering event type and update path〉
let u_m = rng.uniform::<f32>();
let mode = sample_discrete(&[p_absorb, p_scatter, p_null], u_m);
if mode == 0 {
    // 〈Handle absorption along ray path〉
} else if mode == 1 {
    // 〈Handle scattering along ray path〉
} else {
    // 〈Handle null scattering along ray path〉
}
```

As before, the ray path is terminated in the event of absorption. Since any volumetric emission at the sampled point has already been added, there is nothing to do but handle the details associated with ending the path.

```rust
// 〈Handle absorption along ray path〉
terminated = true;
return false;
```

For a real-scattering event, a shadow ray is traced to a light to sample direct lighting, and the path state is updated to account for the new ray. A false value returned from the lambda function prevents further sample generation along the current ray.

```rust
// 〈Handle scattering along ray path〉
// 〈Stop path sampling if maximum depth has been reached〉
// 〈Update beta and r_u for real-scattering event〉
if beta != 0.0 && r_u != 0.0 {
    // 〈Sample direct lighting at volume-scattering event〉
    // 〈Sample new direction at real-scattering event〉
}
return false;
```

The PDF for real scattering at this vertex is the product of the PDF for sampling its distance along the ray, $p_{\mathrm{maj}} = \sigma_{\mathrm{maj}}T_{\mathrm{maj}}$, and the probability for sampling real scattering, $\sigma_s(\mathrm{p}')/\sigma_{\mathrm{maj}}$. The $\sigma_{\mathrm{maj}}$ values cancel.

Given the PDF value, `beta` can be updated to include $T_{\mathrm{maj}}$ along the segment up to the new vertex divided by the PDF. The rescaled probabilities are computed in the same way as the path sampling PDF before being divided by it, following Equation (14.26). The rescaled light path probabilities will be set shortly, after a new ray direction is sampled.

```rust
// 〈Update beta and r_u for real-scattering event〉
let pdf = t_maj[0] * mp.sigma_s[0];
beta *= t_maj * mp.sigma_s / pdf;
r_u *= t_maj * mp.sigma_s / pdf;
```

Direct lighting is handled by the `SampleLd()` method, which we will defer until later in this section.

```rust
// 〈Sample direct lighting at volume-scattering event〉
let intr = MediumInteraction::new(p, -ray.d, ray.time, ray.medium, mp.phase);
l += self.sample_ld(&intr, None, lambda, sampler, beta, r_u);
```

Sampling the phase function gives a new direction at the scattering event.

```rust
// 〈Sample new direction at real-scattering event〉
let u = sampler.get_2d();
let ps = match intr.phase.sample_p(-ray.d, u) {
    Some(s) if s.pdf != 0.0 => s,
    _ => { terminated = true; return false; }
};
// 〈Update ray path state for indirect volume scattering〉
```

There is a bit of bookkeeping to take care of after a real-scattering event. We can now incorporate the phase function value into `beta`, which completes the contribution of $\bar S$ from Equation (14.12). Because both unidirectional path sampling and light path sampling use the same set of sampling operations up to a real-scattering vertex, an initial value for the rescaled light path sampling probabilities `r_l` comes from the value of the rescaled unidirectional probabilities before scattering. It is divided by the directional PDF from $p_\omega$ for this vertex here. The associated directional PDF for light sampling at this vertex will be incorporated into `r_l` later. There is no need to update `r_u` here, since the scattering direction's probability is the same for all wavelengths and so the update factor would always be 1. At this point, the integrator also updates various variables that record the scattering history and updates the current ray.

```rust
// 〈Update ray path state for indirect volume scattering〉
beta *= ps.p / ps.pdf;
r_l = r_u / ps.pdf;
prev_intr_context = LightSampleContext::from(&intr);
scattered = true;
ray.o = p;
ray.d = ps.wi;
specular_bounce = false;
any_non_specular_bounces = true;
```

If the ray intersects a light source, the `LightSampleContext` from the previous path vertex will be needed to compute MIS weights; `prev_intr_context` is updated to store it after each scattering event, whether in a medium or on a surface.

```rust
// 〈Declare state variables for volumetric path sampling〉 +≡
prev_intr_context: LightSampleContext,
```

If null scattering is selected, the updates to `beta` and the rescaled path sampling probabilities follow the same form as we have seen previously: the former is given by Equation (14.11) and the latter with a $p_e = \sigma_n/\sigma_{\mathrm{maj}}$ factor where, as with real scattering, the $\sigma_{\mathrm{maj}}$ cancels with a corresponding factor from the $p_{\mathrm{maj}}$ probability (Section 14.1.5).

In this case, we also must update the rescaled path probabilities for sampling this path vertex via light path sampling, which samples path vertices according to $p_{\mathrm{maj}}$. This fragment concludes the implementation of the lambda function that is passed to the `SampleT_maj()` function.

```rust
// 〈Handle null scattering along ray path〉
let sigma_n = clamp_zero(sigma_maj - mp.sigma_a - mp.sigma_s);
let pdf = t_maj[0] * sigma_n[0];
beta *= t_maj * sigma_n / pdf;
if pdf == 0.0 { beta = SampledSpectrum::from(0.0); }
r_u *= t_maj * sigma_n / pdf;
r_l *= t_maj * sigma_maj / pdf;
return beta != 0.0 && r_u != 0.0;
```

Returning to the `Li()` method immediately after the `SampleT_maj()` call, if the path terminated due to absorption, it is only here that we can break out and return the radiance estimate to the caller of the `Li()` method. Further, it is only here that we can jump back to the start of the while loop for rays that were scattered in the medium.

```rust
// 〈Handle terminated, scattered, and unscattered medium rays〉
if terminated || beta == 0.0 || r_u == 0.0 { return l; }
if scattered { continue; }
```

With those cases taken care of, we are left with rays that either underwent no scattering events in the medium or only underwent null scattering. For those cases, both the path throughput weight $\beta$ and the rescaled path probabilities must be updated. $\beta$ takes a factor of $T_{\mathrm{maj}}$ to account for the transmittance from either the last null-scattering event or the ray's origin to the ray's `t_max` position. The rescaled unidirectional and light sampling probabilities also take the same $T_{\mathrm{maj}}$, which corresponds to the final factors outside of the parenthesis in the definitions of $p_{\mathrm{null}}$ and $p_{\mathrm{ratio}}$.

```rust
// 〈Handle terminated, scattered, and unscattered medium rays〉 +≡
beta *= t_maj / t_maj[0];
r_u *= t_maj / t_maj[0];
r_l *= t_maj / t_maj[0];
```

There is much more to do for rays that have either escaped the scene or have intersected a surface without medium scattering or absorption. We will defer discussion of the first following fragment, 〈Add emitted light at volume path vertex or from the environment〉, until later in the section when we discuss the direct lighting calculation. A few of the others are implemented reusing fragments from earlier integrators.

```rust
// 〈Handle surviving unscattered rays〉
// 〈Add emitted light at volume path vertex or from the environment〉
// 〈Get BSDF and skip over medium boundaries〉
// 〈Initialize visibleSurf at first intersection〉
// 〈Terminate path if maximum depth reached〉
// 〈Possibly regularize the BSDF〉
// 〈Sample illumination from lights to find attenuated path contribution〉
// 〈Sample BSDF to get new volumetric path direction〉
// 〈Account for attenuated subsurface scattering, if applicable〉
// 〈Possibly terminate volumetric path with Russian roulette〉
```

As with the `PathIntegrator`, path termination due to reaching the maximum depth only occurs after accounting for illumination from any emissive surfaces that are intersected.

```rust
// 〈Terminate path if maximum depth reached〉
depth += 1;
if depth - 1 >= self.max_depth { return l; }
```

Sampling the light source at a surface intersection is handled by the same `SampleLd()` method that is called for real-scattering vertices in the medium. As with medium scattering, the `LightSampleContext` corresponding to this scattering event is recorded for possible later use in MIS weight calculations.

```rust
// 〈Sample illumination from lights to find attenuated path contribution〉
if bsdf.flags().is_non_specular() {
    l += self.sample_ld(&isect, Some(&bsdf), lambda, sampler, beta, r_u);
}
prev_intr_context = LightSampleContext::from(&isect);
```

Reference / index entries: `BSDF::Flags()` 544, `BSDF::Sample_f()` 545, `BSDFSample` 541, `BxDFFlags::IsNonSpecular()` 539, `Float` 23, `Interaction::wo` 137, `LightSampleContext` 741, `PathIntegrator` 833, `Sampler::Get1D()` 470, `Sampler::Get2D()` 470, `Vector3f` 86, `VolPathIntegrator::maxDepth` 877, `VolPathIntegrator::SampleLd()` 886.

The logic for sampling scattering at a surface is very similar to the corresponding logic in the `PathIntegrator`.

```rust
// 〈Sample BSDF to get new volumetric path direction〉
let wo = isect.wo;
let u = sampler.get_1d();
let bs = match bsdf.sample_f(wo, u, sampler.get_2d()) {
    Some(s) => s,
    None => break,
};
// 〈Update beta and rescaled path probabilities for BSDF scattering〉
// 〈Update volumetric integrator path state after surface scattering〉
```

Given a `BSDF` sample, $\beta$ is first multiplied by the value of the `BSDF`, which takes care of $\bar S$ from Equation (14.12). This is also a good time to incorporate the cosine factor from the $C_{\mathrm p}$ factor of the generalized geometric term, Equation (14.13). Updates to the rescaled path probabilities follow how they were done for medium scattering: first, there is no need to update `r_u` since the probabilities are the same over all wavelengths. The rescaled light path sampling probabilities are also initialized from `r_u`, here also with only the cosine factor included. The other factors in `r_l` will only be computed and included if the ray intersects an emitter; otherwise `r_l` is unused. One nit in updating `r_l` is that the `BSDF` and PDF value returned in the `BSDFSample` may only be correct up to a (common) scale factor. This case comes up with sampling techniques like the random walk used by the `LayeredBxDF` that is described in Section 14.3.2. In that case, a call to `BSDF::PDF()` gives an independent value for the PDF that can be used.

```rust
// 〈Update beta and rescaled path probabilities for BSDF scattering〉
beta *= bs.f * abs_dot(bs.wi, isect.shading.n) / bs.pdf;
if bs.pdf_is_proportional {
    r_l = r_u / bsdf.pdf(wo, bs.wi);
} else {
    r_l = r_u / bs.pdf;
}
```

A few additional state variables must be updated after surface scattering, as well.

```rust
// 〈Update volumetric integrator path state after surface scattering〉
specular_bounce = bs.is_specular();
any_non_specular_bounces |= !bs.is_specular();
if bs.is_transmission() { eta_scale *= sqr(bs.eta); }
ray = isect.spawn_ray(ray, &bsdf, bs.wi, bs.flags, bs.eta);
```

Russian roulette follows the same general approach as before, though we scale `beta` by the accumulated effect of radiance scaling for transmission that is encoded in `eta_scale` and use the balance heuristic over wavelengths. If the Russian roulette test passes, `beta` is updated with a factor that accounts for the survival probability, $1-q$.

```rust
// 〈Possibly terminate volumetric path with Russian roulette〉
let rr_beta = beta * eta_scale / r_u.average();
let u_rr = sampler.get_1d();
if rr_beta.max_component_value() < 1.0 && depth > 1 {
    let q = (1.0 - rr_beta.max_component_value()).max(0.0);
    if u_rr < q { break; }
    beta /= 1.0 - q;
}
```

**Estimating Direct Illumination.** All that remains in the `VolPathIntegrator`'s implementation is direct illumination. We will start with the `SampleLd()` method, which is called to estimate scattered radiance due to direct illumination by sampling a light source, both at scattering points in media and on surfaces. (It is responsible for computing the second term of Equation (14.24).) The purpose of most of its parameters should be evident. The last, `r_p`, gives the rescaled path probabilities up to the vertex `intr`. (A separate variable named `r_u` will be used in the function's implementation, so a new name is needed here.)

```rust
// 〈VolPathIntegrator Method Definitions〉 +≡
fn sample_ld(
    &self,
    intr: &Interaction,
    bsdf: Option<&dyn BxDF>,
    lambda: &mut SampledWavelengths,
    sampler: &mut Sampler,
    beta: SampledSpectrum,
    r_p: SampledSpectrum,
) -> SampledSpectrum {
    // 〈Estimate light-sampled direct illumination at intr〉
    // ...
    SampledSpectrum::from(0.0)
}
```

The overall structure of this method's implementation is similar to the `PathIntegrator`'s `SampleLd()` method: a light source and a point on it are sampled, the vertex's scattering function is evaluated, and then the light's visibility is determined. Here we have the added complexity of needing to compute the transmittance between the scattering point and the point on the light rather than finding a binary visibility factor, as well as the need to compute spectral path sampling weights for MIS.

```rust
// 〈Estimate light-sampled direct illumination at intr〉
// 〈Initialize LightSampleContext for volumetric light sampling〉
// 〈Sample a light source using lightSampler〉
// 〈Sample a point on the light source〉
// 〈Evaluate BSDF or phase function for light sample direction〉
// 〈Declare path state variables for ray to light source〉
loop {
    // 〈Trace ray through media to estimate transmittance〉
}
// 〈Return path contribution function estimate for direct lighting〉
```

Because it is called for both surface and volumetric scattering path vertices, `SampleLd()` takes a plain `Interaction` to represent the scattering point. Some extra care is therefore needed when initializing the `LightSampleContext`: if scattering is from a surface, it is important to interpret that interaction as the `SurfaceInteraction` that it is so that the shading normal is included in the `LightSampleContext`. This case also presents an opportunity, as was done in the `PathIntegrator`, to shift the light sampling point to avoid incorrectly sampling self-illumination from area lights.

```rust
// 〈Initialize LightSampleContext for volumetric light sampling〉
let ctx = if let Some(b) = bsdf {
    let mut c = LightSampleContext::from(intr.as_surface());
    // 〈Try to nudge the light sampling position to correct side of the surface〉
    c
} else {
    LightSampleContext::from(intr)
};
```

Sampling a point on the light follows in the usual way. Note that the implementation is careful to consume the two sample dimensions from the `Sampler` regardless of whether sampling a light was successful, in order to keep the association of sampler dimensions with integration dimensions fixed across pixel samples.

```rust
// 〈Sample a light source using lightSampler〉
let u = sampler.get_1d();
let sampled_light = self.light_sampler.sample(&ctx, u);
let u_light = sampler.get_2d();
let sampled_light = match sampled_light { Some(s) => s, None => return 0.0.into() };
let light = sampled_light.light;
```

The light samples a direction from the reference point in the usual manner. The true value passed for the `allow_incomplete_pdf` parameter of `Light::SampleLi()` indicates the use of MIS here.

```rust
// 〈Sample a point on the light source〉
let ls = match light.sample_li(&ctx, u_light, lambda, true) {
    Some(s) if s.l != 0.0 && s.pdf != 0.0 => s,
    _ => return SampledSpectrum::from(0.0),
};
let light_pdf = sampled_light.p * ls.pdf;
```

As in `PathIntegrator::SampleLd()`, it is worthwhile to evaluate the `BSDF` or phase function before tracing the shadow ray: if it turns out to be zero-valued for the direction to the light source, then it is possible to exit early and perform no further work.

```rust
// 〈Evaluate BSDF or phase function for light sample direction〉
let mut scatter_pdf: f32;
let mut f_hat: SampledSpectrum;
let wo = intr.wo;
let wi = ls.wi;
if bsdf.is_some() {
    // 〈Update f_hat and scatterPDF accounting for the BSDF〉
} else {
    // 〈Update f_hat and scatterPDF accounting for the phase function〉
}
if f_hat == 0.0 { return SampledSpectrum::from(0.0); }
```

The `f_hat` variable that holds the value of the scattering function is slightly misnamed: it also includes the cosine factor for scattering from surfaces and does not include the $\sigma_s$ for scattering from participating media, as that has already been included in the provided value of `beta`.

```rust
// 〈Update f_hat and scatterPDF accounting for the BSDF〉
if let Some(b) = bsdf {
    f_hat = b.f(wo, wi) * abs_dot(wi, intr.as_surface().shading.n);
    scatter_pdf = b.pdf(wo, wi);
}
// 〈Update f_hat and scatterPDF accounting for the phase function〉
let phase = intr.as_medium().phase;
f_hat = SampledSpectrum::from(phase.p(wo, wi));
scatter_pdf = phase.pdf(wo, wi);
```

A handful of variables keep track of some useful quantities for the ray-tracing and medium sampling operations that are performed to compute transmittance. `T_ray` holds the transmittance along the ray and `r_u` and `r_l` respectively hold the rescaled path probabilities for unidirectional sampling and light sampling, though only along the ray. Maintaining these values independently of the full path contribution and PDFs facilitates the use of Russian roulette in the transmittance computation.

```rust
// 〈Declare path state variables for ray to light source〉
let mut light_ray = intr.spawn_ray_to(ls.p_light);
let mut t_ray = SampledSpectrum::from(1.0);
let mut r_l = SampledSpectrum::from(1.0);
let mut r_u = SampledSpectrum::from(1.0);
let mut rng = Rng::new(hash(light_ray.o), hash(light_ray.d));
```

`SampleLd()` successively intersects the shadow ray with the scene geometry, returning zero contribution if an opaque surface is found and otherwise sampling the medium to estimate the transmittance up to the intersection. For intersections that represent transitions between different media, this process repeats until the ray reaches the light source. For some scenes, it could be more efficient to instead first check that there are no intersections with opaque surfaces before sampling the media to compute the transmittance. With the current implementation, it is possible to do wasted work estimating transmittance before finding an opaque surface farther along the ray.

```rust
// 〈Trace ray through media to estimate transmittance〉
let si = self.intersect(&light_ray, 1.0 - SHADOW_EPSILON);
// 〈Handle opaque surface along ray's path〉
// 〈Update transmittance for current ray segment〉
// 〈Generate next ray segment or return final transmittance〉
```

If an intersection is found with a surface that has a non-null `Material`, the visibility term is zero and the method can return immediately.

```rust
// 〈Handle opaque surface along ray's path〉
if si.as_ref().map_or(false, |h| h.intr.material.is_some()) {
    return SampledSpectrum::from(0.0);
}
```

Otherwise, if participating media is present, `SampleT_maj()` is called to sample it along the ray up to whichever is closer—the surface intersection or the sampled point on the light.

```rust
// 〈Update transmittance for current ray segment〉
if light_ray.medium.is_some() {
    let t_max = si.as_ref().map_or(1.0 - SHADOW_EPSILON, |h| h.t_hit);
    let u = rng.uniform::<f32>();
    let t_maj = sample_t_maj(light_ray, t_max, u, &mut rng, lambda,
        |p: Point3f, mp: MediumProperties,
         sigma_maj: SampledSpectrum, t_maj: SampledSpectrum| {
            // 〈Update ray transmittance estimate at sampled point〉
            // 〈Possibly terminate transmittance computation using Russian roulette〉
            true
        });
    // 〈Update transmittance estimate for final segment〉
}
```

For each sampled point in the medium, the transmittance and rescaled path probabilities are updated before Russian roulette is considered.

In the context of the equation of transfer, using ratio tracking to compute transmittance can be seen as sampling distances along the ray according to the majorant transmittance and then only including the null-scattering component of the source function $L_n$ to correct any underestimate of transmittance from $T_{\mathrm{maj}}$. Because only null-scattering vertices are sampled along transmittance rays, the logic for updating the transmittance and rescaled path probabilities at each vertex exactly follows that in the 〈Handle null scattering along ray path〉 fragment.

```rust
// 〈Update T_ray and PDFs using ratio-tracking estimator〉
let sigma_n = clamp_zero(sigma_maj - mp.sigma_a - mp.sigma_s);
let pdf = t_maj[0] * sigma_maj[0];
t_ray *= t_maj * sigma_n / pdf;
r_l *= t_maj * sigma_maj / pdf;
r_u *= t_maj * sigma_n / pdf;
```

Russian roulette is used to randomly terminate rays with low transmittance. A natural choice might seem to be setting the survival probability equal to the transmittance—along the lines of how Russian roulette is used for terminating ray paths from the camera according to $\beta$. However, doing so would effectively transform ratio tracking to delta tracking, with the transmittance always equal to zero or one. The implementation therefore applies a less aggressive termination probability, only to highly attenuated rays. In the computation of the transmittance value used for the Russian roulette test, note that an MIS weight that accounts for both the unidirectional and light sampling strategies is used, along the lines of Equation (14.27).

```rust
// 〈Possibly terminate transmittance computation using Russian roulette〉
let tr = t_ray / (r_l + r_u).average();
if tr.max_component_value() < 0.05 {
    let q = 0.75;
    if rng.uniform::<f32>() < q {
        t_ray = SampledSpectrum::from(0.0);
    } else {
        t_ray /= 1.0 - q;
    }
}
```

After the `SampleT_maj()` call returns, the transmittance and rescaled path probabilities all must be multiplied by the `T_maj` returned from `SampleT_maj()` for the final ray segment.

```rust
// 〈Update transmittance estimate for final segment〉
t_ray *= t_maj / t_maj[0];
r_l *= t_maj / t_maj[0];
r_u *= t_maj / t_maj[0];
```

If the transmittance is zero (e.g., due to Russian roulette termination), it is possible to return immediately. Furthermore, if there is no surface intersection, then there is no further medium sampling to be done and we can move on to computing the scattered radiance from the light. Alternatively, if there is an intersection, it must be with a surface that represents the boundary between two media; the `SpawnRayTo()` method call returns the continuation ray on the other side of the surface, with its medium member variable set appropriately.

```rust
// 〈Generate next ray segment or return final transmittance〉
if t_ray == 0.0 { return SampledSpectrum::from(0.0); }
if si.is_none() { break; }
light_ray = si.as_ref().unwrap().intr.spawn_ray_to(ls.p_light);
```

After the while loop terminates, we can compute the final rescaled path probabilities, compute MIS weights, and return the final estimate of the path contribution function for the light sample. The `r_p` variable passed in to `SampleLd()` stores the rescaled path probabilities for unidirectional sampling of the path up to the vertex where direct lighting is being computed—though here, `r_u` and `r_l` have been rescaled using the light path sampling probability, since that is how the vertices were sampled along the shadow ray. However, recall from Equations (14.21) and (14.23) that for the path up to the scattering vertex $p_u = p_l$. Thus, `r_p` can be interpreted as being rescaled using $p_u$. This allows multiplying `r_l` and `r_u` by `r_p` to compute final rescaled path probabilities. If the light source is described by a delta distribution, only the light sampling technique is applicable; there is no chance of intersecting such a light via sampling the `BSDF` or phase function. In that case, we still apply MIS using all the wavelengths' individual path PDFs in order to reduce variance in chromatic media. For area lights, we are able to use both light source and the scattering function samples, giving two primary sampling strategies, each of which has a separate weight for each wavelength.

```rust
// 〈Return path contribution function estimate for direct lighting〉
r_l *= r_p * light_pdf;
r_u *= r_p * scatter_pdf;
if is_delta_light(light.light_type()) {
    return beta * f_hat * t_ray * ls.l / r_l.average();
} else {
    return beta * f_hat * t_ray * ls.l / (r_l + r_u).average();
}
```

With `SampleLd()` implemented, we will return to the fragments in the `Li()` method that handle the cases where a ray escapes from the scene and possibly finds illumination from infinite area lights, as well as where a ray intersects an emissive surface. These handle the first term in the direct lighting MIS estimator, Equation (14.24).

```rust
// 〈Add emitted light at volume path vertex or from the environment〉
if si.is_none() {
    // 〈Accumulate contributions from infinite light sources〉
    break;
}
let isect = &si.as_ref().unwrap().intr;
if let Some(le) = isect.le(-ray.d, lambda) {
    // 〈Add contribution of emission from intersected surface〉
}
```

As with the `PathIntegrator`, if the previous scattering event was due to a delta-distribution scattering function, then sampling the light is not a useful strategy. In that case, the MIS weight is only based on the per-wavelength PDFs for the unidirectional sampling strategy.

Reference / index entries: `Integrator::infiniteLights` 23, `IsDeltaLight()` 741, `Light::Le()` 743, `Light::Type()` 740, `LightLiSample::L` 743, `Ray::d` 95, `SampledSpectrum` 171, `SampledSpectrum::Average()` 172, `ShapeIntersection::intr` 266, `SurfaceInteraction` 138, `SurfaceInteraction::Le()` 762.

```rust
// 〈Accumulate contributions from infinite light sources〉
for light in &self.infinite_lights {
    if let Some(le) = light.le(ray, lambda) {
        if depth == 0 || specular_bounce {
            l += beta * le / r_u.average();
        } else {
            // 〈Add infinite light contribution using both PDFs with MIS〉
        }
    }
}
```

Otherwise, the MIS weight should account for both sampling techniques. At this point, `r_l` has everything but the probabilities for sampling the light itself. (Recall that we deferred that when initializing `r_l` at the real-scattering vertex earlier.) After incorporating that factor, all that is left is to compute the final weight, accounting for both sampling strategies.

```rust
// 〈Add infinite light contribution using both PDFs with MIS〉
let light_pdf = self.light_sampler.pmf(&prev_intr_context, light)
              * light.pdf_li(&prev_intr_context, ray.d, true);
r_l *= light_pdf;
l += beta * le / (r_u + r_l).average();
```

The work done in the 〈Add contribution of emission from intersected surface〉 fragment is very similar to that done for infinite lights, so it is not included here.

## 14.3 SCATTERING FROM LAYERED MATERIALS

In addition to describing scattering from larger-scale volumetric media like clouds or smoke, the equation of transfer can be used to model scattering at much smaller scales. The `LayeredBxDF` applies it to this task, implementing a reflection model that accounts for scattering from two interfaces that are represented by surfaces with independent `BSDFs` and with a medium between them. Monte Carlo can be applied to estimating the integrals that describe the aggregate scattering behavior, in a way similar to what is done in light transport algorithms. This approach is effectively the generalization of the technique used to sum up aggregate scattering from a pair of perfectly smooth dielectric interfaces in the `ThinDielectricBxDF` in Section 9.5.1.

Modeling surface reflectance as a composition of layers makes it possible to describe a variety of surfaces that are not well modeled by the `BSDFs` presented in Chapter 9. For example, automobile paint generally has a smooth reflective "clear coat" layer applied on top of it; the overall appearance of the paint is determined by the combination of light scattering from the layer's interface as well as light scattering from the paint. (See Figure 14.11.) Tarnished metal can be modeled by an underlying metal `BRDF` with a thin scattering medium on top of it; it is again the aggregate result of a variety of light scattering paths that determines the overall appearance of the surface.

**Figure 14.11 — layered-surface scattering diagram.** Surface reflection can be modeled with a series of layers, where each interface between media is represented with a `BSDF` and where the media between layers may itself both absorb and scatter light. The aggregate scattering from such a configuration can be determined by finding solutions to the equation of transfer. Light from the environment and from the camera may scatter multiple times between the two interfaces before exiting.

```mermaid
flowchart TB
    CAM["camera (ω_o)"] -->|"reflect / transmit"| TOP["top interface (BSDF)"]
    SUN["environment illumination"] --> TOP
    TOP -->|"transmit into slab"| MED["medium: absorption + scattering"]
    MED --> BOT["bottom interface (BSDF)"]
    BOT -->|"reflect back up"| MED
    MED -->|"transmit back out (ω_i)"| TOP
    TOP -->|"outgoing radiance"| CAM
```

Reference / index entries: `Float` 23, `Light::PDF_Li()` 743, `LightSampler::PMF()` 782, `Ray::d` 95, `SampledSpectrum::Average()` 172, `ThinDielectricBxDF` 567, `VolPathIntegrator::lightSampler` 877.

**Figure 14.12 — setting for the one-dimensional equation of transfer.** If the properties of the medium only vary in one dimension and if the incident illumination is uniform over its boundary, then the equilibrium radiance distribution varies only with depth $z$ and direction $\omega$.

```mermaid
flowchart TB
    SUN["uniform incident illumination"] -->|"parallel rays"| TOP["boundary of slab"]
    TOP --> SLAB["plane-parallel medium<br/>properties vary only with depth z"]
    P["point at depth z:<br/>L(z, ω)"] --- SLAB
```

**Figure 14.13 — transmittance in 1D.** The distance between two depths is the $z$ distance between them divided by the cosine of the ray's angle with respect to the $z$ axis, $\theta$; hence $d = z/\omega_z$ and the transmittance follows.

```mermaid
flowchart TB
    A["depth z₀"] -->|"vertical drop z = d·ω_z"| B["depth z₁"]
    A -->|"ray segment d at angle θ"| B
    NOTE["d = z / |ω_z|  ⇒  T(z₀ → z₁) = exp(−σ_t · d)"]
```

With general layered media, light may exit the surface at a different point than that at which it entered it. The `LayeredBxDF` does not model this effect but instead assumes that light enters and exits at the same point on the surface. (As a `BxDF`, it is unable to express any other sort of scattering, anyway.) This is a reasonable approximation if the distance between the two interfaces is relatively small. This approximation makes it possible to use a simplified 1D version of the equation of transfer. After deriving this variant, we will show its application to evaluating and sampling such `BSDFs`.

### 14.3.1 THE ONE-DIMENSIONAL EQUATION OF TRANSFER

Given plane-parallel 3D scattering media where the scattering properties are homogeneous across planes and only vary in depth, and where the incident illumination does not vary as a function of position over the medium's planar boundary, the equations that describe scattering can be written in terms of 1D functions over depth (see Figure 14.12).

In this setting, the various quantities are more conveniently expressed as functions of depth $z$ rather than of distance $t$ along a ray. For example, if the extinction coefficient is given by $\sigma_t(z)$, then the transmittance between two depths $z_0$ and $z_1$ for a ray with direction $\omega$ is

$$
T(z_0\to z_1)
= \exp\!\left(-\int_{z_0}^{z_1}\frac{\sigma_t(z)}{|\omega_z|}\,\mathrm{d}z\right).
\tag{14.30}
$$

Reference / index entries: `BxDF` 538.

See Figure 14.13. This definition uses the fact that if a ray with direction $\omega$ travels a distance $t$, then the change in $z$ is $t\omega_z$.

**Figure 14.14 — the 1D equation of transfer.** The 1D specialization of the equation of transfer expresses the incident radiance $L_i(z,\omega)$ at a depth $z$ as the sum of attenuated radiance $L_o(z_i,-\omega)$ from the interface visible along the ray and the transmission-modulated source function $L_s(z',\omega)$ integrated over $z$.

```mermaid
flowchart LR
    SUN["scene illumination"] --> IF["visible interface at z_i:<br/>L_o(z_i, -ω)"]
    SRC["source at z': L_s(z', ω)"] -->|"σ_t(z') T(z'→z) dz'"| OBS["L_i(z, ω)"]
    IF -->|"T(z → z_i)"| OBS
```

In the case of a homogeneous medium,

$$
T(z_0\to z_1) = \exp\!\left(-\frac{\sigma_t\,|z_1-z_0|}{|\omega_z|}\right).
$$

The 1D equation of transfer can be found in a similar fashion. It says that at points inside the medium the incident radiance at a depth $z$ in direction $\omega$ is given by

$$
L_i(z,\omega)
= T(z\to z_i)\, L_o(z_i,\omega)
  + \int_{z}^{z_i} \sigma_t(z')\, T(z'\to z)\, L_s(z',\omega)\, \mathrm{d}z',
\tag{14.31}
$$

where $z_i$ is the depth of the medium interface that the ray from $z$ in direction $\omega$ intersects. (See Figure 14.14.) At boundaries, the incident radiance function is given by Equation (14.31) for directions $\omega$ that point into the medium. For directions that point outside it, incident radiance is found by integrating illumination from the scene.

The scattering from an interface at a boundary of the medium is given by the incident radiance modulated by the boundary's `BSDF`,

$$
L_o(\omega_o)
= \int_{\mathcal{H}^2} f(\omega_o,\omega_i)\,
    L_i(\omega_i)\, |\cos\theta_i|\, \mathrm{d}\omega_i.
\tag{14.32}
$$

If we also assume there is no volumetric emission (as we will do in the `LayeredBxDF`), the source function in Equation (14.31) simplifies to

$$
L_s(z,\omega)
= \sigma_s(z) \int_{\mathcal{S}^2} p(\omega,\omega')\,
    L_i(z,\omega')\, \mathrm{d}\omega'.
\tag{14.33}
$$

The `LayeredBxDF` further assumes that $\sigma_t$ is constant over all wavelengths in the medium, which means that null scattering is not necessary for sampling distances in the medium. Null scattering is easily included in the 1D simplification of the equation of transfer if necessary, though we will not do so here. For similar reasons, we will also not derive its path integral form in 1D, though it too can be found with suitable simplifications to the approach that was used in Section 14.1.4. The "Further Reading" section has pointers to more details.

Reference / index entries: `LayeredBxDF` 895.

### 14.3.2 LAYERED BxDF

The equation of transfer describes the equilibrium distribution of radiance, though our interest here is in evaluating and sampling the `BSDF` that represents all the scattering from the layered medium. Fortunately, these two things can be connected. If we would like to evaluate the `BSDF` for a pair of directions $\omega_o$ and $\omega_i$, then we can define an incident radiance function from a virtual light source from $\omega_i$ as

$$
L_i(\omega') = \frac{\delta(\omega' - \omega_i)}{|\cos\theta_i|}.
\tag{14.34}
$$

**Figure 14.15 — virtual light source.** If a medium is illuminated with a virtual light source of the form of Equation (14.34), then the radiance leaving the surface in the direction $\omega_o$ is equivalent to the layered surface's `BSDF`, $f(\omega_o,\omega_i)$.

```mermaid
flowchart LR
    VL["virtual light L_i(ω') = δ(ω'−ω_i) / |cos θ_i|"] -->|"illuminates slab"| S["layered surface"]
    S -->|"L_o(ω_o) = f(ω_o, ω_i)"| CAM["camera"]
```

If a 1D medium is illuminated by such a light, then the outgoing radiance $L_o(\omega_o)$ at the medium's interface is equivalent to the value of the `BSDF`, $f(\omega_o,\omega_i)$ (see Figure 14.15). One way to understand why this is so is to consider using such a light with the surface reflection equation:

$$
L_o(\omega_o) = \int_{\mathcal{H}^2} f(\omega_o,\omega_i)\,
    L_i(\omega_i)\,|\cos\theta_i|\, \mathrm{d}\omega_i
              = f(\omega_o,\omega_i).
$$

Thus, integrating the equation of transfer with such a light allows us to evaluate and sample the corresponding `BSDF`. However, this means that unlike all the `BxDF` implementations from Chapter 9, the values that `LayeredBxDF` returns from methods like `f()` and `PDF()` are stochastic. This is perfectly fine in the context of all of pbrt's Monte Carlo–based techniques and does not affect the correctness of other estimators that use these values; it is purely one more source of error that can be controlled in a predictable way by increasing the number of samples. The `LayeredBxDF` allows the specification of only two interfaces and a homogeneous participating medium between them. Figure 14.16 illustrates the geometric setting. Surfaces with more layers can be modeled using a `LayeredBxDF` where one or both of its layers are themselves `LayeredBxDF`s. (An exercise at the end of the chapter discusses a more efficient approach for supporting additional layers.) The types of `BxDF`s at both interfaces can be provided as template parameters. While the user of a `LayeredBxDF` is free to provide a `BxDF` for both of these types (in which case pbrt's regular dynamic dispatch mechanism will be used), performance is better if they are specific `BxDF`s and the compiler can generate a specialized implementation. This approach is used for the `CoatedDiffuseBxDF` and the `CoatedConductorBxDF` that are defined in Section 14.3.3. (The meaning of the `twoSided` template parameter will be explained in a few pages, where it is used.)

Reference / index entries: `BxDF` 538, `CoatedConductorBxDF` 909, `CoatedDiffuseBxDF` 909.

**Figure 14.16 — geometric setting for the `LayeredBxDF`.** Scattering is specified by two interfaces with associated `BSDF`s, where the bottom one is at $z=0$ and there is a medium of user-specified thickness between the two interfaces.

```mermaid
flowchart TB
    TOP["z = thickness: TopBxDF interface"]
    MED["homogeneous medium: σ_t = 1, albedo, g"]
    BOT["z = 0: BottomBxDF interface"]
    TOP --- MED
    MED --- BOT
    RAY["incident ray ω_o"] --> TOP
```

**Figure 14.17 (pictorial renders, kept).** *Effect of Varying `Medium` Thickness with the `LayeredBxDF`.* (a) Dragon with surface reflectance modeled by a smooth conductor base layer and a dielectric interface above it. (b) With a scattering layer with albedo 0.7 and thickness 0.15 between the interface and the conductor, the reflection of the conductor is slightly dimmed. (c) With a thicker scattering layer of thickness 0.5, the conductor is much more attenuated and the overall reflection is more diffuse. (Dragon model courtesy of the Stanford Computer Graphics Laboratory.)

<details>
<summary>Figure 14.17 — three dragons, varying layer thickness (a, b, c)</summary>

![Figure 14.17](images/img-1423-333.png)

</details>

```rust
// 〈LayeredBxDF Definition〉
struct LayeredBxDF<TopBxDF, BottomBxDF, const TWO_SIDED: bool> {
    // 〈LayeredBxDF Public Methods〉
    // 〈LayeredBxDF Private Methods〉
    // 〈LayeredBxDF Private Members〉
    top: TopBxDF,
    bottom: BottomBxDF,
    thickness: f32,
    g: f32,
    albedo: SampledSpectrum,
    max_depth: i32,
    n_samples: i32,
}
```

In addition to `BxDF`s for the two interfaces, the `LayeredBxDF` maintains member variables that describe the medium between them. Rather than have the user specify scattering coefficients, which can be unintuitive to set manually, it assumes a medium with $\sigma_t = 1$ and leaves it to the user to specify both the thickness of the medium and its scattering albedo. Figure 14.17 shows the effect of varying the thickness of the medium between a conductor base layer and a dielectric interface.

Reference / index entries: `BottomBxDF` 895, `BxDF` 538, `Float` 23, `SampledSpectrum` 171, `TopBxDF` 895.

```rust
// 〈LayeredBxDF Private Members〉
top: TopBxDF,
bottom: BottomBxDF,
thickness: f32,
g: f32,
albedo: SampledSpectrum,
```

**Figure 14.18 — top/bottom convention.** If the incident ray intersects the layer at $z=\text{thickness}$, then the top layer is the same as is specified in the `LayeredBxDF::top` member variable. However, if it intersects the surface from the other direction at $z=0$, we treat the $z=0$ layer as the top one and the other as the bottom. The `TopOrBottomBxDF` class helps with the related bookkeeping.

```mermaid
flowchart TB
    TOP["z = thickness (top interface)"]
    BOT["z = 0 (bottom interface)"]
    R1["ray entering at z = thickness"] -->|"top = top member"| TOP
    R2["ray entering at z = 0"] -->|"swap: treat z=0 as top"| BOT
```

Two parameters control the Monte Carlo estimates. `max_depth` has its usual role in setting a maximum number of scattering events along a path and `n_samples` controls the number of independent samples of the estimators that are averaged. Because additional samples in this context do not require tracing more rays or evaluating textures, it is more efficient to reduce any noise due to the stochastic `BSDF` by increasing this sampling rate rather than increasing the pixel sampling rate if a higher pixel sampling rate is not otherwise useful.

```rust
// 〈LayeredBxDF Private Members〉 +≡
max_depth: i32,
n_samples: i32,
```

We will find it useful to have a helper function `Tr()` that returns the transmittance for a ray segment in the medium with given direction `w` that passes through a distance `dz` in `z`, following Equation (14.30) with $\sigma_t = 1$.

```rust
// 〈LayeredBxDF Private Methods〉
fn tr(dz: f32, w: Vector3f) -> f32 {
    fast_exp(-(dz / w.z).abs())
}
```

Although the `LayeredBxDF` is specified in terms of top and bottom interfaces, we will find it useful to exchange the "top" and "bottom" as necessary to have the convention that the interface that the incident ray intersects is defined to be the top one. (See Figure 14.18.) A helper class, `TopOrBottomBxDF`, manages the logic related to these possibilities. As its name suggests, it stores a pointer to one (and only one) of two `BxDF` types that are provided as template parameters.

```rust
// 〈TopOrBottomBxDF Definition〉
struct TopOrBottomBxDF<'a, TopBxDF, BottomBxDF> {
    // 〈TopOrBottomBxDF Public Methods〉
    top: Option<&'a TopBxDF>,
    bottom: Option<&'a BottomBxDF>,
}
```

`TopOrBottomBxDF` provides the implementation of a number of `BxDF` methods like `f()`, where it calls the corresponding method of whichever of the two `BxDF` types has been provided. In addition to `f()`, it has similar straightforward `sample_f()`, `pdf()`, and `flags()` methods, which we will not include here.

```rust
// 〈TopOrBottomBxDF Public Methods〉
fn f(&self, wo: Vector3f, wi: Vector3f, mode: TransportMode) -> SampledSpectrum {
    match (self.top, self.bottom) {
        (Some(t), _) => t.f(wo, wi, mode),
        (_, Some(b)) => b.f(wo, wi, mode),
        _           => SampledSpectrum::from(0.0),
    }
}
```

**BSDF Evaluation.** The `BSDF` evaluation method `f()` can now be implemented; it returns an average of the specified number of independent samples.

```rust
// 〈LayeredBxDF Public Methods〉
fn f(&self, mut wo: Vector3f, mut wi: Vector3f, mode: TransportMode) -> SampledSpectrum {
    let mut f = SampledSpectrum::from(0.0);
    // 〈Estimate LayeredBxDF value f using random sampling〉
    f / self.n_samples as f32
}
```

There is some preliminary computation that is independent of each sample taken to estimate the `BSDF`'s value. A few fragments take care of it before the random estimation begins.

```rust
// 〈Estimate LayeredBxDF value f using random sampling〉
// 〈Set wo and wi for layered BSDF evaluation〉
// 〈Determine entrance interface for layered BSDF〉
// 〈Determine exit interface and exit z for layered BSDF〉
// 〈Account for reflection at the entrance interface〉
// 〈Declare RNG for layered BSDF evaluation〉
for _ in 0..self.n_samples {
    // 〈Sample random walk through layers to estimate BSDF value〉
}
```

With this `BSDF`, layered materials can be specified as either one- or two-sided via the `twoSided` template parameter. If a material is one-sided, then the shape's surface normal is used to determine which interface an incident ray enters. If it is in the same hemisphere as the surface normal, it enters the top interface and otherwise it enters the bottom. This configuration is especially useful when both interfaces are transmissive and have different `BSDFs`. For two-sided materials, the ray always enters the top interface. This option is useful when the bottom interface is opaque as is the case with the `CoatedDiffuseBxDF`, for example. In this case, it is usually desirable for scattering from both layers to be included, no matter which side the ray intersects.

One way to handle these options in the `f()` method would be to negate both directions and make a recursive call to `f()` if $\omega_o$ points below the surface and the material is two-sided. However, that solution is not a good one for the GPU, where it is likely to introduce thread divergence. (This topic is discussed in more detail in Section 15.1.1.) Therefore, both directions are negated at the start of the method and no recursive call is made in this case, which gives an equivalent result.

```rust
// 〈Set wo and wi for layered BSDF evaluation〉
if TWO_SIDED && wo.z < 0.0 { wo = -wo; wi = -wi; }
```

The next step is to determine which of the two `BxDF`s is the one that is encountered first by the incident ray. The sign of $\omega_o$'s $z$ component in the reflection coordinate system gives the answer.

```rust
// 〈Determine entrance interface for layered BSDF〉
let mut enter_interface: TopOrBottomBxDF<TopBxDF, BottomBxDF>;
let entered_top = TWO_SIDED || wo.z > 0.0;
if entered_top { enter_interface = TopOrBottomBxDF { top: Some(&self.top), bottom: None }; }
else           { enter_interface = TopOrBottomBxDF { top: None, bottom: Some(&self.bottom) }; }
```

It is also necessary to determine which interface $\omega_i$ exits. This is determined both by which interface $\omega_o$ enters and by whether $\omega_o$ and $\omega_i$ are in the same hemisphere. We end up with an unusual case where the EXCLUSIVE-OR operator comes in handy. Along the way, the method also stores which interface is the one that $\omega_i$ does not exit from. As random paths are sampled through the layers and medium, the implementation will always choose reflection from this interface and not transmission, as choosing the latter would end the path without being able to scatter out in the $\omega_i$ direction. The same logic then covers determining the $z$ depth at which the ray path will exit the surface.

```rust
// 〈Determine exit interface and exit z for layered BSDF〉
let (exit_interface, non_exit_interface, exit_z) =
    if same_hemisphere(wo, wi) ^ entered_top {
        (TopOrBottomBxDF { top: None, bottom: Some(&self.bottom) },
         TopOrBottomBxDF { top: Some(&self.top), bottom: None },
         0.0)
    } else {
        (TopOrBottomBxDF { top: Some(&self.top), bottom: None },
         TopOrBottomBxDF { top: None, bottom: Some(&self.bottom) },
         self.thickness)
    };
```

If both directions are on the same side of the surface, then part of the `BSDF`'s value is given by reflection at the entrance interface. This can be evaluated directly by calling the interface's `BSDF`'s `f()` method. The resulting value must be scaled by the total number of samples taken to estimate the `BSDF` in this method, since the final returned value is divided by `n_samples`.

```rust
// 〈Account for reflection at the entrance interface〉
if same_hemisphere(wo, wi) {
    f = self.n_samples as f32 * enter_interface.f(wo, wi, mode);
}
```

pbrt's `BxDF` interface does not include any uniform sample values as parameters to the `f()` method; there is no need for them for any of the other `BxDF`s in the system. In any case, an unbounded number of uniform random numbers are required for sampling operations when evaluating layered `BSDFs`. Therefore, `f()` initializes an `Rng` and defines a convenience lambda function that returns uniform random sample values. This does mean that the benefits of sampling with well-distributed point sets are not present here; an exercise at the end of the chapter returns to this issue. The `RNG` is seeded carefully: it is important that calls to `f()` with different directions have different seeds so that there is no risk of errors due to correlation between the `RNG`s used for multiple samples in a pixel or across nearby pixels. However, we would also like the samples to be deterministic so that any call to `f()` with the same two directions always has the same set of random samples. This sort of reproducibility is important for debugging so that errors appear consistently across multiple runs of the program. Hashing the two provided directions along with the system-wide seed addresses all of these concerns.

**Figure 14.19 — cosine-weighted BTDF integration.** The effect of light that scatters between the interface layers is found by integrating the product of the cosine-weighted `BTDF` at the entrance interface with the incident radiance from the medium, Equation (14.35).

```mermaid
flowchart TB
    CAM["camera: L_o(ω_o)"] --> IF["exit interface<br/>cosine-weighted BTDF"]
    IF --- HEMI["hemisphere inside the medium"]
    HEMI -->|"directions ω'"| LI["incident radiance L_i(z, ω')"]
```

```rust
// 〈Declare RNG for layered BSDF evaluation〉
let mut rng = Rng::new(hash(get_options().seed, wo), hash(wi));
let r = || std::cmp::min(rng.uniform::<f32>(), ONE_MINUS_EPSILON);
```

In order to find the radiance leaving the interface in the direction $\omega_o$, we need to integrate the product of the cosine-weighted `BTDF` at the interface with the incident radiance from inside the medium,

$$
L_o(\omega_o)
= \int_{\mathcal{H}^2} f_{BTDF}(\omega_o,\omega')\,
    L_i(\omega')\,|\cos\theta'|\, \mathrm{d}\omega',
\tag{14.35}
$$

where $\mathcal{H}^2$ is the hemisphere inside the medium (see Figure 14.19). The implementation uses the standard Monte Carlo estimator, taking a sample $\omega'$ from the `BTDF` and then proceeding to estimate $L_i$.

```rust
// 〈Sample random walk through layers to estimate BSDF value〉
// 〈Sample transmission direction through entrance interface〉
// 〈Sample BSDF for virtual light from wi〉
// 〈Declare state for random walk through BSDF layers〉
for _ in 0..self.max_depth {
    // 〈Sample next event for layered BSDF evaluation random walk〉
}
```

Sampling the direction $\omega'$ is a case where it is useful to be able to specify to `Sample_f()` that only transmission should be sampled.

**Figure 14.20 — illumination contribution from the virtual `Light` source.** At a path vertex, the contribution of the virtual light source is given by the product of the path throughput weight $\beta$ that accounts for previous scattering along the path, the scattering at the vertex, the transmittance $T_r$ to the exit interface, and the effect of the `BTDF` at the interface.

```mermaid
flowchart LR
    V["path vertex at depth z<br/>throughput β"] -->|"× T_r(z → z_exit)"| E["exit interface<br/>× BTDF value"]
    PHASE["phase function / interface BSDF<br/>p(−w, −w_i)"] --> V
    E --> VL["virtual-light contribution to L_o"]
```

```rust
// 〈Sample transmission direction through entrance interface〉
let uc = r();
let wos = match enter_interface.sample_f(
        wo, uc, Point2f::new(r(), r()), mode, BxDFReflTransFlags::Transmission) {
    Some(s) if s.f != 0.0 && s.pdf != 0.0 && s.wi.z != 0.0 => s,
    _ => continue,
};
```

The task now is to compute a Monte Carlo estimate of the 1D equation of transfer, Equation (14.31). Before discussing how it is sampled, however, we will first consider some details related to the lighting calculation with the virtual light source. At each vertex of the path, we will want to compute the incident illumination due to the light. As shown in Figure 14.20, there are three factors in the light's contribution: the value of the phase function or interface `BSDF` for a direction $\omega$, the transmittance between the vertex and the exit interface, and the value of the interface's `BTDF` for the direction from $-\omega$ to $\omega_i$.

Each of these three factors could be used for sampling; as before, one may sometimes be much more effective than the others. The `LayeredBxDF` implementation uses two of the three—sampling the phase function or `BRDF` at the current path vertex (as appropriate) and sampling the `BTDF` at the exit interface—and then weights the results using MIS. There is no reason to repeatedly sample the exit interface `BTDF` at each path vertex since the direction $\omega_i$ is fixed. Therefore, the following fragment samples it once and holds on to the resulting `BSDFSample`. Note that the negation of the `TransportMode` parameter value `mode` is taken for the call to `Sample_f()`, which is important to reflect the fact that this sampling operation is following the reverse path with respect to sampling in terms of $\omega_o$. This is an important detail so that the underlying `BxDF` can correctly account for non-symmetric scattering; see Section 9.5.2.

```rust
// 〈Sample BSDF for virtual light from wi〉
let uc = r();
let wis = match exit_interface.sample_f(
        wi, uc, Point2f::new(r(), r()), !mode, BxDFReflTransFlags::Transmission) {
    Some(s) if s.f != 0.0 && s.pdf != 0.0 && s.wi.z != 0.0 => s,
    _ => continue,
};
```

Moving forward to the random walk estimation of the equation of transfer, the implementation maintains the current path throughput weight `beta`, the depth `z` of the last scattering event, and the ray direction `w`.

```rust
// 〈Declare state for random walk through BSDF layers〉
let mut beta = wos.f * abs_cos_theta(wos.wi) / wos.pdf;
let mut z = if entered_top { self.thickness } else { 0.0 };
let mut w = wos.wi;
let phase = HGPhaseFunction::new(self.g);
```

We can now move to the body of the inner loop over scattering events along the path. After a Russian roulette test, a distance is sampled along the ray to determine the next path vertex either within the medium or at whichever interface the ray intersects.

```rust
// 〈Sample next event for layered BSDF evaluation random walk〉
// 〈Possibly terminate layered BSDF random walk with Russian roulette〉
// 〈Account for media between layers and possibly scatter〉
// 〈Account for scattering at appropriate interface〉
```

It is worth considering terminating the path as the path throughput weight becomes low, though here the termination probability is set less aggressively than it was in the `PathIntegrator` and `VolPathIntegrator`. This reflects the fact that each bounce here is relatively inexpensive, so doing more work to improve the accuracy of the estimate is worthwhile.

```rust
// 〈Possibly terminate layered BSDF random walk with Russian roulette〉
if depth > 3 && beta.max_component_value() < 0.25 {
    let q = (1.0 - beta.max_component_value()).max(0.0);
    if r() < q { break; }
    beta /= 1.0 - q;
}
```

The common case of no scattering in the medium is handled separately since it is much simpler than the case where volumetric scattering must be considered.

```rust
// 〈Account for media between layers and possibly scatter〉
if self.albedo == 0.0 {
    // 〈Advance to next layer boundary and update beta for transmittance〉
} else {
    // 〈Sample medium scattering for layered BSDF evaluation〉
}
```

If there is no medium scattering, then only the first term of Equation (14.31) needs to be evaluated. The path vertices alternate between the two interfaces. Here `beta` is multiplied by the transmittance for the ray segment through the medium; the $L_o$ factor is found by estimating Equation (14.32), which will be handled shortly.

```rust
// 〈Advance to next layer boundary and update beta for transmittance〉
z = if z == self.thickness { 0.0 } else { self.thickness };
beta *= self.tr_fn(self.thickness, w);   // Tr(thickness, w)
```

If the medium is scattering, we only sample one of the two terms of the 1D equation of transfer, choosing between taking a sample inside the medium and scattering at the other interface. A change in depth $\Delta z$ can be perfectly sampled from the 1D beam transmittance, Equation (14.30). Since $\sigma_t = 1$, the PDF is

$$
p(\Delta z) = \frac{1}{|\omega_z|}\,
             \exp\!\left(-\frac{\Delta z}{|\omega_z|}\right).
$$

Given a depth $z'$ found by adding or subtracting $\Delta z$ from the current depth $z$ according to the ray's direction, medium scattering is chosen if $z'$ is inside the medium and surface scattering is chosen otherwise. (The sampling scheme is thus similar to how the `VolPathIntegrator` chooses between medium and surface scattering.) In the case of scattering from an interface, the `Clamp()` call effectively forces $z$ to lie on whichever of the two interfaces the ray intersects next.

```rust
// 〈Sample medium scattering for layered BSDF evaluation〉
let sigma_t = 1.0;
let dz = sample_exponential(r(), sigma_t / w.z.abs());
let zp = if w.z > 0.0 { z + dz } else { z - dz };
if 0.0 < zp && zp < self.thickness {
    // 〈Handle scattering event in layered BSDF medium〉
    continue;
}
z = clamp(zp, 0.0, self.thickness);
```

If $z'$ is inside the medium, we have the estimator

$$
\left\langle L_s(z',\omega)\, T(z'\to z)\, \frac{1}{p(\Delta z)}\right\rangle
= L_s(z',\omega),
$$

since the exponential factors and $|\omega_z|$ factors in $T$ and $p(\Delta z)$ cancel, leaving simply the source function $L_s(z',-\omega)$, which should be scaled by the path throughput. The following fragment adds an estimate of its value to the sum in `f`.

```rust
// 〈Handle scattering event in layered BSDF medium〉
// 〈Account for scattering through exitInterface using wis〉
// 〈Sample phase function and update layered path state〉
// 〈Possibly account for scattering through exitInterface〉
```

For a scattering event inside the medium, it is necessary to add the contribution of the virtual light source to the path radiance estimate and to sample a new direction to continue the path. For the MIS lighting sample based on sampling the interface's `BTDF`, the outgoing direction from the path vertex is predetermined by the `BTDF` sample `wis`; all the factors of the path contribution are easily evaluated and the MIS weight is found using the PDF for the other sampling technique, sampling the phase function.

```rust
// 〈Account for scattering through exitInterface using wis〉
let mut wt = 1.0;
if !exit_interface.flags().is_specular() {
    wt = power_heuristic(1.0, wis.pdf, 1.0, phase.pdf(-w, -wis.wi));
}
f += beta * self.albedo * phase.p(-w, -wis.wi) * wt
   * self.tr_fn(zp - exit_z, wis.wi) * wis.f / wis.pdf;
```

The second sampling strategy for the virtual light is based on sampling the phase function and then connecting to the virtual light source through the exit interface. Doing so shares some common work with sampling a new direction for the path, so the implementation takes the opportunity to update the path state after sampling the phase function here.

```rust
// 〈Sample phase function and update layered path state〉
let u = Point2f::new(r(), r());
let ps = match phase.sample_p(-w, u) {
    Some(s) if s.pdf != 0.0 && s.wi.z != 0.0 => s,
    _ => continue,
};
beta *= self.albedo * ps.p / ps.pdf;
w = ps.wi;
z = zp;
```

There is no reason to try connecting through the exit interface if the current ray direction is pointing away from it or if its `BSDF` is perfect specular.

```rust
// 〈Possibly account for scattering through exitInterface〉
if ((z < exit_z && w.z > 0.0) || (z > exit_z && w.z < 0.0))
   && !exit_interface.flags().is_specular() {
    // 〈Account for scattering through exitInterface〉
}
```

If there is transmission through the interface, then because `beta` has already been updated to include the effect of scattering at $z'$, only the transmittance to the exit, MIS weight, and `BTDF` value need to be evaluated to compute the light's contribution. One important detail in the following code is the ordering of arguments to the call to `f()` in the first line: due to the non-reciprocity of BTDFs, swapping these would lead to incorrect results.[^14-4]

```rust
// 〈Account for scattering through exitInterface〉
let f_exit = exit_interface.f(-w, wi, mode);
if f_exit != 0.0 {
    let exit_pdf = exit_interface.pdf(-w, wi, mode, BxDFReflTransFlags::Transmission);
    let wt = power_heuristic(1.0, ps.pdf, 1.0, exit_pdf);
    f += beta * self.tr_fn(zp - exit_z, ps.wi) * f_exit * wt;
}
```

If no medium scattering event was sampled, the next path vertex is at an interface. In this case, the transmittance along the ray can be ignored: as before, the probability of evaluating the first term of Equation (14.31) has probability equal to $T$ and thus the two $T$ factors cancel, leaving us only needing to evaluate scattering at the boundary, Equation (14.32). The details differ depending on which interface the ray intersected.

```rust
// 〈Account for scattering at appropriate interface〉
if z == exit_z {
    // 〈Account for reflection at exitInterface〉
} else {
    // 〈Account for scattering at nonExitInterface〉
}
```

If the ray intersected the exit interface, then it is only necessary to update the path throughput: no connection is made to the virtual light source since transmission through the exit interface to the light is accounted for by the lighting computation at the previous vertex. This fragment samples only the reflection component of the path here, since a ray that was transmitted outside the medium would end the path.

```rust
// 〈Account for reflection at exitInterface〉
let uc = r();
let bs = match exit_interface.sample_f(
        -w, uc, Point2f::new(r(), r()), mode, BxDFReflTransFlags::Reflection) {
    Some(s) if s.f != 0.0 && s.pdf != 0.0 && s.wi.z != 0.0 => s,
    _ => break,
};
beta *= bs.f * abs_cos_theta(bs.wi) / bs.pdf;
w = bs.wi;
```

The 〈Account for scattering at nonExitInterface〉 fragment handles scattering from the other interface. It applies MIS to compute the contribution of the virtual light and samples a new direction with a form very similar to the case of scattering within the medium, just with the phase function replaced by the `BRDF` for evaluation and sampling. Therefore, we have not included its implementation here.

**BSDF Sampling.** The implementation of `Sample_f()` is generally similar to `f()`, so we will not include its implementation here, either. Its task is actually simpler: given the initial direction $\omega_o$ at one of the layer's boundaries, it follows a random walk of scattering events through the layers and the medium, maintaining both the path throughput and the product of PDFs for each of the sampling decisions. When the random walk exits the medium, the outgoing direction is the sampled direction that is returned in the `BSDFSample`.

With this approach, it can be shown that the ratio of the path throughput to the PDF is equal to the ratio of the actual value of the `BSDF` and its PDF for the sampled direction (see the "Further Reading" section for details). Therefore, when the weighted path throughput is multiplied by the ratio of `BSDFSample::f` and `BSDFSample::pdf`, the correct weighting term is applied. (Review, for example, the fragment 〈Update path state variables after surface scattering〉 in the `PathIntegrator`.) However, an implication of this is that the PDF value returned by `Sample_f()` cannot be used to compute the multiple importance sampling weight if the sampled ray hits an emissive surface; in that case, an independent estimate of the PDF must be computed via a call to the `PDF()` method. The `BSDFSample::pdfIsProportional` member variable flags this case and is set by `Sample_f()` here.

**PDF Evaluation.** The PDF $p(\omega_o,\omega_i)$ that corresponds to a `LayeredBxDF`'s `BSDF` can be expressed as an infinite sum. For example, consider the case of having a bottom layer that reflects light with `BRDF` $f_r^b$ and a top layer that both reflects light with `BRDF` $f_r^t$ and transmits it with `BTDF` $f_t^t$, with an overall `BSDF` $f$. If those `BSDFs` have associated PDFs $p$, and if scattering in the medium is neglected, then the overall PDF is

$$
\begin{aligned}
p(\omega_o,\omega_i)
&= p_r^t(\omega_o,\omega_i)\\
&\quad + \int_{\mathcal{H}^2}\!\!\int_{\mathcal{H}^2}
   p_t^t(\omega_o,\omega')\,
   p_r^b(\omega',\omega'')\,
   p_t^t(\omega'',\omega_i)\,
   \mathrm{d}\omega''\,\mathrm{d}\omega'
   + \cdots
\end{aligned}
\tag{14.36}
$$

The first term gives the contribution for the PDF at the top interface and the second is the PDF for directions $\omega_i$ that were sampled via transmission through the top interface, scattering at the bottom, and then transmission back out in direction $\omega_i$. Note the coupling of directions between the PDF factors in the integrand: the negation of the initial transmitted direction $\omega'$ gives the first direction for evaluation of the base PDF, and so forth (see Figure 14.21). Subsequent terms of this sum account for light that is reflected back downward at the top interface instead of exiting the layers, and expressions of a similar form can be found for the PDF if the base layer is also transmissive.

**Figure 14.21 — the first two terms of the infinite PDF sum.** (a) The PDF of the reflection component of the interface's `BSDF` accounts for light that scatters without entering the layers. (b) The second term is given by a double integral over directions: a direction $\omega'$ pointing into the medium is sampled; it gives the second direction for the interface's `BTDF` PDF and its negation gives one of the two directions for $p_r^b$. A second direction $\omega''$ is used for $p_r^b$ as well as for a second evaluation of $p_t^t$.

```mermaid
flowchart TB
    subgraph first["(a) First term — reflection at top interface"]
        WO["ω_o"] -->|"p_r^t(ω_o, ω_i)"| WI["ω_i"]
    end
    subgraph second["(b) Second term — T·R·T double integral"]
        WO2["ω_o"] -->|"p_t^t(ω_o, ω')"| A["direction ω' into medium"]
        A -->|"p_r^b(−ω', ω'')"| B["direction ω'' at bottom"]
        B -->|"p_t^t(−ω'', ω_i)"| WI2["ω_i"]
    end
```

It is possible to compute a stochastic estimate of the PDF by applying Monte Carlo to the integrals and by terminating the infinite sum using Russian roulette. For example, for the integral in Equation (14.36), we have

$$
\langle p\rangle
= \frac{p_t^t(\omega_o,\omega')\,p_r^b(\omega',\omega'')\,p_t^t(\omega'',\omega_i)}
       {p_1(\omega')\,p_2(\omega'')},
\tag{14.37}
$$

the estimator where $\omega'$ is sampled from some distribution $p_1$ and $\omega''$ from a distribution $p_2$. There is great freedom in choosing the distributions $p_1$ and $p_2$. However, as with algorithms like path tracing, care must be taken if some of the PDFs are Dirac delta distributions. For example, if the bottom layer is perfect specular, then $p_r^b$ will always be zero unless $\omega''$ was sampled according to its PDF.

Consider what happens if $\omega'$ is sampled using the top interface's sampling method, conditioned on $\omega_o$, and if $\omega''$ is sampled using the bottom interface's sampling method, conditioned on $\omega_i$: the first and last probabilities in the numerator cancel with the probabilities in the denominator, and we are left simply with $p_r^b(-\omega',-\omega'')$ as the estimate; the effect of $p_r^b$ in the PDF is fully encapsulated by the distribution of directions used to evaluate it. A stochastic estimate of the PDF can be computed by following a random walk in a manner similar to the `f()` method, just with phase function and `BSDF` evaluation replaced with evaluations of the corresponding PDFs. However, because the `PDF()` method is only called to compute PDF values that are used for computing MIS weights, the implementation here will return an approximate PDF; doing so does not invalidate the MIS estimator.[^14-5]

```rust
// 〈LayeredBxDF Public Methods〉 +≡
fn pdf(&self, wo: Vector3f, wi: Vector3f, mode: TransportMode,
       sample_flags: BxDFReflTransFlags) -> f32 {
    // 〈Set wo and wi for layered BSDF evaluation〉
    // 〈Declare RNG for layered PDF evaluation〉
    // 〈Update pdfSum for reflection at the entrance layer〉
    for _ in 0..self.n_samples {
        // 〈Evaluate layered BSDF PDF sample〉
    }
    // 〈Return mixture of PDF estimate and constant PDF〉
    // ...
}
```

It is important that the `Rng` for the `PDF()` method is seeded differently than it is for the `f()` method, since it will often be called with the same pair of directions as are passed to `f()`, and we would like to be certain that there is no correlation between the results returned by the two of them.

```rust
// 〈Declare RNG for layered PDF evaluation〉
let mut rng = Rng::new(hash(get_options().seed, wi), hash(wo));
let r = || std::cmp::min(rng.uniform::<f32>(), ONE_MINUS_EPSILON);
```

If both directions are on the same side of the surface, then part of the full PDF is given by the PDF for reflection at the interface (this was the first term of Equation (14.36)). This component can be evaluated non-stochastically, assuming that the underlying `PDF()` methods are not themselves stochastic.

```rust
// 〈Update pdfSum for reflection at the entrance layer〉
let entered_top = TWO_SIDED || wo.z > 0.0;
let mut pdf_sum = 0.0;
if same_hemisphere(wo, wi) {
    let refl_flag = BxDFReflTransFlags::Reflection;
    pdf_sum += if entered_top {
        self.n_samples as f32 * self.top.pdf(wo, wi, mode, refl_flag)
    } else {
        self.n_samples as f32 * self.bottom.pdf(wo, wi, mode, refl_flag)
    };
}
```

The more times light has been scattered, the more isotropic its directional distribution tends to become. We can take advantage of this fact by evaluating only the first term of the stochastic PDF estimate and modeling the remaining terms with a uniform distribution. We further neglect the effect of scattering in the medium, again under the assumption that if it is significant, a uniform distribution will be a suitable approximation.

```rust
// 〈Evaluate layered BSDF PDF sample〉
if same_hemisphere(wo, wi) {
    // 〈Evaluate TRT term for PDF estimate〉
} else {
    // 〈Evaluate TT term for PDF estimate〉
}
```

If both directions are on the same side of the interface, then the remaining PDF integral is the double integral of the product of three PDFs that we considered earlier. We use the shorthand "TRT" for this case, corresponding to transmission, then reflection, then transmission.

```rust
// 〈Evaluate TRT term for PDF estimate〉
let (mut r_interface, mut t_interface) =
    if entered_top {
        (TopOrBottomBxDF { top: None, bottom: Some(&self.bottom) },
         TopOrBottomBxDF { top: Some(&self.top), bottom: None })
    } else {
        (TopOrBottomBxDF { top: Some(&self.top), bottom: None },
         TopOrBottomBxDF { top: None, bottom: Some(&self.bottom) })
    };
// 〈Sample tInterface to get direction into the layers〉
// 〈Update pdfSum accounting for TRT scattering events〉
```

We will apply two sampling strategies. The first is sampling both directions via `tInterface`, once conditioned on $\omega_o$ and once on $\omega_i$—effectively a bidirectional approach. The second is sampling one direction via `tInterface` conditioned on $\omega_o$ and the other via `rInterface` conditioned on the first sampled direction. These are then combined using multiple importance sampling. After canceling factors and introducing an MIS weight $w(\omega'')$, Equation (14.37) simplifies to

$$
\langle p\rangle \approx p_r^t(-\omega',-\omega'')\, w(\omega''),
\tag{14.38}
$$

which is the estimator for both strategies.

```rust
// 〈Sample tInterface to get direction into the layers〉
let trans = BxDFReflTransFlags::Transmission;
let wos = t_interface.sample_f(wo, r(), Point2f::new(r(), r()), mode, trans);
let wis = t_interface.sample_f(wi, r(), Point2f::new(r(), r()), !mode, trans);
```

Both sampling methods will use the `wos` sample while only one uses `wis`. If `tInterface` is perfect specular, then there is no need to try sampling or to apply MIS. The PDF is all that remains from Equation (14.38).

```rust
// 〈Update pdfSum accounting for TRT scattering events〉
if let (Some(wos), Some(wis)) = (&wos, &wis) {
    if wos.f != 0.0 && wos.pdf > 0.0 && wis.f != 0.0 && wis.pdf > 0.0 {
        if !t_interface.flags().is_non_specular() {
            pdf_sum += r_interface.pdf(-wos.wi, -wis.wi, mode);
        } else {
            // 〈Use multiple importance sampling to estimate PDF product〉
        }
    }
}
```

Otherwise, we sample from `rInterface` as well. If that sample is from a perfect specular component, then again there is no need to use MIS and the estimator is just `tInterface.PDF()`.

```rust
// 〈Use multiple importance sampling to estimate PDF product〉
let rs = r_interface.sample_f(-wos.wi, r(), Point2f::new(r(), r()), mode);
if let Some(rs) = rs {
    if rs.f != 0.0 && rs.pdf > 0.0 {
        if !r_interface.flags().is_non_specular() {
            pdf_sum += t_interface.pdf(-rs.wi, wi, mode);
        } else {
            // 〈Compute MIS-weighted estimate of Equation (14.38)〉
        }
    }
}
```

If neither interface has a specular sample, then both are combined. For the first sampling technique, the second factor cancels out as well and the estimator is $p_r^b$ times the MIS weight.

```rust
// 〈Compute MIS-weighted estimate of Equation (14.38)〉
let r_pdf = r_interface.pdf(-wos.wi, -wis.wi, mode);
let mut wt = power_heuristic(1.0, wis.pdf, 1.0, r_pdf);
pdf_sum += wt * r_pdf;
```

Similarly, for the second sampling technique, we are left with a PDF to evaluate and then weight using MIS.

```rust
// 〈Compute MIS-weighted estimate of Equation (14.38)〉 +≡
let t_pdf = t_interface.pdf(-rs.wi, wi, mode);
wt = power_heuristic(1.0, rs.pdf, 1.0, t_pdf);
pdf_sum += wt * t_pdf;
```

The 〈Evaluate TT term for PDF estimate〉 fragment is of a similar form, so it is not included here. The final returned PDF value has the PDF for uniform spherical sampling, $1/4\pi$, mixed with the estimate to account for higher-order terms.

```rust
// 〈Return mixture of PDF estimate and constant PDF〉
lerp(0.9, 1.0 / (4.0 * PI), pdf_sum / self.n_samples as f32)
```

A WGSL version of the layered random walk keeps the interface selection branch-free (no recursion) and processes the two interfaces explicitly, which maps cleanly onto a compute shader:

```wgsl
// One step of the 1D layered random walk (no recursion, no closures).
fn layered_walk_step(
    state: ptr<function, LayerState>,
    thick: f32, albedo: vec3<f32>, g: f32, exit_z: f32,
) -> vec3<f32> {
    var contribution = vec3<f32>(0.0);
    let sigma_t = 1.0;
    let dz = sample_exponential(next_uniform(), sigma_t / abs((*state).w.z));
    let zp = select((*state).z - dz, (*state).z + dz, (*state).w.z > 0.0);

    if (zp > 0.0 && zp < thick) {
        // Real volume scatter: source function L_s(z', -w).
        let p = hg_p(g, -(*state).w, -(*state).w);
        contribution += (*state).beta * albedo * p;
        let ps = sample_hg(g, -(*state).w, vec2f(next_uniform(), next_uniform()));
        (*state).beta *= albedo * ps.p / ps.pdf;
        (*state).w = ps.wi;
        (*state).z = zp;
    } else {
        // No medium scatter: advance to the interface and apply its BSDF.
        (*state).z = select(0.0, thick, (*state).z != thick);
        (*state).beta *= exp(-thick / abs((*state).w.z));
        contribution += interface_scatter(state, exit_z);
    }
    return contribution;
}
```

### 14.3.3 COATED DIFFUSE AND COATED CONDUCTOR MATERIALS

Adding a dielectric interface on top of both diffuse materials and conductors is often useful to model surface reflection. For example, plastic can be modeled by putting such an interface above a diffuse material, and coated metals can be modeled by adding such an interface as well. In both cases, introducing a scattering layer can model effects like tarnish or weathering. Figure 14.22 shows the dragon model with a few variations of these.

pbrt provides both the `CoatedDiffuseBxDF` and the `CoatedConductorBxDF` for such uses. There is almost nothing to their implementations other than a public inheritance from `LayeredBxDF` with the appropriate types for the two interfaces.

```rust
// 〈CoatedDiffuseBxDF Definition〉 — pbrt uses public inheritance from
// LayeredBxDF; in Rust this is expressed as a type alias over the generic type.
type CoatedDiffuseBxDF = LayeredBxDF<DielectricBxDF, DiffuseBxDF, true>;
// 〈CoatedDiffuseBxDF Public Methods〉

// 〈CoatedConductorBxDF Definition〉
type CoatedConductorBxDF = LayeredBxDF<DielectricBxDF, ConductorBxDF, true>;
// 〈CoatedConductorBxDF Public Methods〉
```

Reference / index entries: `BSDFSample::pdf` 541, `BSDFSample::wi` 541, `BxDF::PDF()` 541, `Float` 23, `LayeredBxDF` 895, `LayeredBxDF::nSamples` 896, `Lerp()` 72, `Pi` 1033, `PowerHeuristic()` 66.

There are also corresponding `Material` implementations, `CoatedDiffuseMaterial` and `CoatedConductorMaterial`. Their implementations follow the familiar pattern of evaluating textures and then initializing the corresponding `BxDF`, and they are therefore not included here.

**Figure 14.22 (pictorial renders, kept).** *A Variety of Effects That Can Be Achieved Using Layered `Materials`.* (a) Dragon model with a blue diffuse `BRDF`. (b) The effect of adding a smooth dielectric interface on top of the diffuse `BRDF`. In addition to the specular highlights, note how the color has become more saturated, which is due to multiple scattering from paths that reflected back into the medium from the exit interface. (c) The effect of roughening the interface. The surface appears less shiny, but the blue remains more saturated. (Dragon model courtesy of the Stanford Computer Graphics Laboratory.)

<details>
<summary>Figure 14.22 — three blue dragons, coating variations (a, b, c)</summary>

![Figure 14.22](images/img-1447-338.png)

</details>

## FURTHER READING

Lommel (1889) first derived the equation of transfer. Not only did he derive this equation, but he also solved it in some simplified cases in order to estimate reflection functions from real-world surfaces (including marble and paper), and he compared his solutions to measured reflectance data from these surfaces. The equation of transfer was independently found by Khvolson (1890) soon afterward; see Mishchenko (2013) for a history of early work in the area. Seemingly unaware of Lommel's work, Schuster (1905) was the next researcher in radiative transfer to consider the effect of multiple scattering. He used the term self-illumination to describe the fact that each part of the medium is illuminated by every other part of the medium, and he derived differential equations that described reflection from a slab along the normal direction, assuming the presence of isotropic scattering. The conceptual framework that he developed remains essentially unchanged in the field of radiative transfer. Soon thereafter, Schwarzschild (1906) introduced the concept of radiative equilibrium, and Jackson (1910) expressed Schuster's equation in integral form, also noting that "the obvious physical mode of solution is Liouville's method of successive substitutions" (i.e., a Neumann series solution).

Finally, King (1913) completed the rediscovery of the equation of transfer by expressing it in the general integral form. Books by Chandrasekhar (1960), Preisendorfer (1965, 1976), and van de Hulst (1980) cover volume light transport in depth. D'Eon's book (2016) extensively discusses scattering problems, including both analytic and Monte Carlo solutions, and contains many references to related work in other fields. The equation of transfer was introduced to graphics by Kajiya and Von Herzen (1984). Arvo (1993) made essential connections between previous formalizations of light transport in graphics and the equation of transfer as well as to the field of radiative transfer in general. Pauly et al. (2000) derived the generalization of the path integral form of the light transport equation for the volume-scattering case; see also Chapter 3 of Jakob's Ph.D. thesis (2013) for a full derivation.

The integral null-scattering volume light transport equation was derived by Galtier et al. (2013) in the field of radiative transfer; Eymet et al. (2013) described the generalization to include scattering from surfaces. This approach was introduced to graphics by Novák et al. (2014). Miller et al. (2019) derived its path integral form, which made it possible to apply powerful variance reduction techniques based on multiple importance sampling.

**Volumetric Path Tracing.** von Neumann's original description of the Monte Carlo algorithm was in the context of neutron transport problems (Ulam et al. 1947); his technique included the algorithm for sampling distances from an exponential distribution (our Equation (A.2)), uniformly sampling 3D directions via uniform sampling of $\cos\theta$ (as implemented in `SampleUniformSphere()`), and randomly choosing among scattering events as described in Section 14.1.2.

Rushmeier (1988) was the first to use Monte Carlo to solve the volumetric light transport equation in a general setting. Szirmay-Kalos et al. (2005) precomputed interactions between sample points in the medium in order to more quickly compute multiple scattering. Kulla and Fajardo (2012) proposed a specialized sampling technique that is effective for light sources inside participating media. (This technique was first introduced in the field of neutron transport by Kalli and Cashwell (1977).) Georgiev et al. (2013) made the observation that incremental path sampling can generate particularly bad paths in participating media. They proposed new multi-vertex sampling methods that better account for all the relevant terms in the equation of transfer. Sampling direct illumination from lights at points inside media surrounded by an interface is challenging; traditional direct lighting algorithms are not applicable at points inside the medium, as refraction through the interface will divert the shadow ray's path. Walter et al. (2009) considered this problem and developed algorithms to efficiently find paths to lights accounting for this refraction. More recent work on this topic was done by Holzschuch (2015) and Koerner et al. (2016). Weber et al. (2017) developed an approach for more effectively sampling direct lighting in forward scattering media by allowing multiple scattering events along the path to the light. Szirmay-Kalos et al. (2017) first showed the use of the integral null-scattering volume light transport equation for rendering scattering inhomogeneous media. Kutz et al. (2017) subsequently applied it to efficient rendering of spectral media and Szirmay-Kalos et al. (2018) developed improved algorithms for sampling multiple scattering. After deriving the path integral formulation, Miller et al. (2019) used it to show the effectiveness of combining a variety of sampling techniques using multiple importance scattering, including bidirectional path tracing. The visual appearance of high-albedo objects like clouds is striking, but many bounces may be necessary for good results. Wrenninge et al. (2013) described an approximation where after the first few bounces, the scattering coefficient, the attenuation coefficient for shadow rays, and the eccentricity of the phase function are all progressively reduced. Kallweit et al. (2017) applied neural networks to store precomputed multiple scattering solutions for use in rendering highly scattering clouds.

Index entry: `SampleUniformSphere()` 1016.

Pegoraro et al. (2008b) developed a Monte Carlo sampling approach for rendering participating media that used information from previous samples to guide future sampling. More recent work in volumetric path guiding by Herholz et al. applied product sampling based on the phase function and an approximation to the light distribution in the medium (Herholz et al. 2019). Wrenninge and Villemin (2020) developed a volumetric product sampling approach based on adapting the majorant to account for important regions of the integrand and then randomly selecting among candidate samples based on weights that account for factors beyond transmittance. Villeneuve et al. (2021) have also developed algorithms for product sampling in media, accounting for the surface normal at area light sources, transmittance along the ray, and the phase function. Volumetric emission is not handled efficiently by the `VolPathIntegrator`, as there is no specialized sampling technique to account for it. Villemin and Hery (2013) precomputed tabularized CDFs for sampling volumetric emission, and Simon et al. (2017) developed further improvements, including integrating emission along rays and using the sampled point in the volume solely to determine the initial sampling direction, which gives better results in dense media. The one-dimensional volumetric light transport algorithms implemented in `LayeredBxDF` are based on Guo et al.'s approach (2018).

**Other `Light` Transport Algorithms.** Blinn (1982b) first used basic volume scattering algorithms for computer graphics. Rushmeier and Torrance (1987) used finite-element methods for rendering participating media. Other early work in volume scattering for computer graphics includes work by Max (1986); Nishita, Miyawaki, and Nakamae (1987); Bhate and Tokuta's approach based on spherical harmonics (Bhate and Tokuta 1992), and Blasi et al.'s two-pass Monte Carlo algorithm, where the first pass shoots energy from the lights and stores it in a grid and the second pass does final rendering using the grid to estimate illumination at points in the scene (Blasi, Saëc, and Schlick 1993). Glassner (1995) provided a thorough overview of this topic and early applications of it in graphics, and Max's survey article (Max 1995) also covers early work well. See Cerezo et al. (2005) for an extensive survey of approaches to rendering participating media up through 2005. One important application of volume scattering algorithms in computer graphics has been simulating atmospheric scattering. Work in this area includes early papers by Klassen (1987) and Preetham et al. (1999), who introduced a physically rigorous and computationally efficient atmospheric and sky-lighting model. Haber et al. (2005) described a model for twilight, and Hošek and Wilkie (2012, 2013) developed a comprehensive model for sky- and sunlight. Bruneton evaluated the accuracy and efficiency of a number of models for atmospheric scattering (Bruneton 2017). A sophisticated model that accurately accounts for polarization, observers at arbitrary altitudes, and the effect of atmospheric scattering for objects at finite distances was recently introduced by Wilkie et al. (2021). Jarosz et al. (2008a) first extended the principles of irradiance caching to participating media. Marco et al. (2018) described a state-of-the-art algorithm for volumetric radiance caching based on Schwarzhaupt et al.'s surface-based second-order derivatives (Schwarzhaupt et al. 2012). Jensen and Christensen (1998) were the first to generalize the photon-mapping algorithm to participating media. Knaus and Zwicker (2011) showed how to render participating media using stochastic progressive photon mapping (SPPM). Jarosz et al. (2008b) had the important insight that expressing the scattering integral over a beam through the medium as the measurement to be evaluated could make photon mapping's rate of convergence much higher than if a series of point photon estimates was instead taken along each ray. Section 5.6 of Hachisuka's thesis (2011) and Jarosz et al. (2011a, 2011b) showed how to apply this approach progressively. For another representation, see Jakob et al. (2011), who fit a sum of anisotropic Gaussians to the equilibrium radiance distribution in participating media.

Many of the other bidirectional light transport algorithms discussed in the "Further Reading" section of Chapter 13 also have generalizations to account for participating media. See also Jarosz's thesis (2008), which has extensive background on this topic and includes a number of important contributions. Some researchers have had success in deriving closed-form expressions that describe scattering along unoccluded ray segments in participating media; these approaches can be substantially more efficient than integrating over a series of point samples. See Sun et al. (2005), Pegoraro and Parker (2009), and Pegoraro et al. (2009, 2010, 2011) for examples of such methods. (Remarkably, Pegoraro and collaborators' work provides a closed-form expression for scattering from a point light source along a ray passing through homogeneous participating media with anisotropic phase functions.)

**Subsurface Scattering.** Subsurface scattering models based on volumetric light transport were first introduced to graphics by Hanrahan and Krueger (1993), although their approach did not attempt to simulate light that entered the object at points other than at the point being shaded. Dorsey et al. (1999) applied photon maps to simulating subsurface scattering that did include this effect, and Pharr and Hanrahan (2000) introduced an approach based on computing BSSRDFs for arbitrary scattering media with an integral over the medium's depth. The diffusion approximation has been shown to be an effective way to model highly scattering media for rendering. It was first introduced to graphics by Kajiya and Von Herzen (1984), though Stam (1995) was the first to clearly identify many of its advantages for rendering. A solution of the diffusion approximation based on dipoles was developed by Farrell et al. (1992); that approach was applied to BSSRDF modeling for rendering by Jensen et al. (2001b). Subsequent work by Jensen and Buhler (2002) improved the efficiency of that method. A more accurate solution based on photon beam diffusion was developed by Habel et al. (2013). (The online edition of this book includes the implementation of a BSSRDF model based on photon beam diffusion as well as many more references to related work.) Rendering realistic human skin is a challenging problem; this problem has driven the development of a number of new methods for rendering subsurface scattering after the initial dipole work as issues of modeling the layers of skin and computing more accurate simulations of scattering between layers have been addressed. For a good overview of these issues, see Igarashi et al.'s (2007) survey on the scattering mechanisms inside skin and approaches for measuring and rendering skin. Notable research in this area includes papers by Donner and Jensen (2006), d'Eon et al. (2007), Ghosh et al. (2008), and Donner et al. (2008). Donner's thesis includes a discussion of the importance of accurate spectral representations for high-quality skin rendering (Donner 2006, Section 8.5). See Gitlina et al. (2020) for recent work in the measurement of the scattering properties of skin and fitting it to a BSSRDF model. An alternative to BSSRDF-based approaches to subsurface scattering is to apply the same volumetric Monte Carlo path-tracing techniques that are used for other scattering media. This approach is increasingly used in production (Chiang et al. 2016b). See Wrenninge et al. (2017) for a discussion of such a model designed for artistic control and expressiveness. Křivánek and d'Eon introduced the theory of zero-variance random walks for path-traced subsurface scattering, applying Dwivedi's sampling technique (1982a; 1982b) to guide paths to stay close to the surface while maintaining an unbiased estimator (Křivánek and d'Eon 2014). Meng et al. (2016) developed further improvements to this approach, including strategies that handle back-lit objects more effectively. More recent work on zero-variance theory by d'Eon and Křivánek (2020) includes improved results with isotropic scattering and new sampling schemes that further reduce variance. Leonard et al. (2021) applied machine learning to subsurface scattering, training conditional variational auto-encoders to sample scattering, to model absorption probabilities, and to sample the positions of ray paths in spherical regions. They then used these capabilities to implement an efficient sphere-tracing algorithm.

**Generalizations.** Moon et al. (2007) made the important observation that some of the assumptions underlying the use of the equation of transfer—that the scattering particles in the medium are not too close together so that scattering events can be considered to be statistically independent—are not in fact true for interesting scenes that include small crystals, ice, or piles of many small glass objects. They developed a new light transport algorithm for these types of discrete random media based on composing precomputed scattering solutions. (See also concurrent work by Lee and O'Sullivan (2007) on composing scattering solutions.) Further work on rendering such materials was done by Müller et al. (2016), Guo et al. (2019), and Zhang and Zhao (2020). Non-exponential media have distributions of interactions that are not described by an exponential distribution. They arise from media that have correlation in the distribution of their particles. The assumption of uncorrelated media that we adopted in Chapter 11 and have used throughout this chapter can immediately be understood to be at minimum not quite right by considering the fact that there must be a minimum distance between any two particles; thus, the distribution cannot be perfectly uncorrelated. In practice, media with even more significant correlations are common; a variety of physical effects that lead to them are described by Bitterli et al. (2018b). Both d'Eon (2018) and Jarabo et al. (2018) developed generalizations of the equation of transfer that allow non-exponential media. Bitterli et al. (2018b) presented a more general path integral form of it that maintains reciprocity and allows heterogeneous media. Jakob et al. (2010) derived a generalized transfer equation that describes scattering by distributions of oriented particles. They proposed a microflake scattering model as a specific example of a particle distribution (where a microflake is the volumetric analog of a micro-facet on a surface) and showed a number of ways of solving this equation based on Monte Carlo, finite elements, and a dipole model. More recently, Heitz et al. (2015) derived a generalized microflake distribution, which is considerably more efficient to sample and evaluate. Their model quantifies the local scattering properties using projected areas observed from different directions, which adds a well-defined notion of volumetric level of detail. Zhao et al. (2016) and Loubet and Neyret (2018) developed techniques for downsampling microflake distributions while still maintaining their visual appearance.

The equation of transfer assumes that the index of refraction of a medium will only change at discrete boundaries, though many actual media have continuously varying indices of refraction. Ament et al. (2014) derived a variant of the equation of transfer that allows for this case and applied photon mapping to render images with it. Pediredla et al. (2020) further investigated this topic and developed an unbiased rendering algorithm for such media based on path tracing. Handling fluorescence in the context of volumetric scattering introduces a number of complexities discussed by Mojzík et al. (2018), who also derived a fluorescence-aware sampling algorithm.

## EXERCISES

➋ **14.1** Replace ratio tracking in the `VolPathIntegrator::SampleLd()` method with delta tracking. After you confirm that your changes converge to the correct result, measure the difference in performance and MSE in order to compare the Monte Carlo efficiency of the two approaches for a variety of volumetric data sets. Do you find any cases where delta tracking is more efficient? If so, can you explain why?

➋ **14.2** Residual ratio tracking can compute transmittance more efficiently than ratio tracking in dense media; it is based on finding lower bounds of $\sigma_t$ in regions of space, analytically computing that portion of the transmittance, and then using ratio tracking for the remaining variation (Novák et al. 2014). Implement this approach in pbrt and measure its effectiveness. Note that you will need to make modifications to both the `Medium`'s `RayMajorantSegment` representation and the implementation of the `VolPathIntegrator` in order to do so.

➋ **14.3** The current implementation of `SampleT_maj()` consumes a new uniform random value for each `RayMajorantSegment` returned by the medium's iterator. Its sampling operation can alternatively be implemented using a single uniform value to sample a total optical thickness and then finding the point along the ray where that optical thickness has been accumulated. Modify `SampleT_maj()` to implement that approach and measure rendering performance. Is there a benefit compared to the current implementation?

➌ **14.4** It is not possible to directly sample emission in volumes with the current `Medium` interface. Thus, integrators are left to include emission only when their random walk through a medium happens to find a part of it that is emissive. This approach can be quite inefficient, especially for localized bright emission. Add methods to the `Medium` interface that allow for sampling emission and modify the direct lighting calculation in the `VolPathIntegrator` to use them. For inspiration, it may be worthwhile to read the papers by Villemin and Hery (2013) and Simon et al. (2017) on Monte Carlo sampling of 3D emissive volumes. Measure the improvement in efficiency with your approach. Are there any cases where it hurts performance?

➌ **14.5** While sampling distances in participating media according to the majorant is much more effective than sampling uniformly, it does not account for other factors that vary along the ray, such as the scattering coefficient and phase function or variation in illumination from light sources. Implement the approach described by Wrenninge and Villemin (2020) on product sampling based on adapting the majorant to account for multiple factors in the integrand and then randomly selecting among weighted sample points. (You may find weighted reservoir sampling (Section A.2) a useful technique to apply in order to avoid the storage costs of maintaining the candidate samples.) Measure the performance of your implementation as well as how much it improves image quality for tricky volumetric scenes.

➋ **14.6** Add the capability to specify a bump or normal map for the bottom interface in the `LayeredBxDF`. (The current implementation applies bump mapping at the top interface only.) Render images that show the difference between perturbing the normal at the top interface and having a smooth bottom interface and vice versa.

➋ **14.7** Investigate the effect of improving the sampling patterns used in the `LayeredBxDF`—for example, by replacing the uniform random numbers used with low-discrepancy points. You may need to pass further information through the `BSDF` evaluation routines to do so, such as the current pixel, pixel sample, and current ray depth. Measure how much error is reduced by your changes as well as their performance impact.

Index / reference entries: `LayeredBxDF` 895, `Medium` 714, `RayMajorantSegment` 718, `SampleT_maj()` 859, `VolPathIntegrator` 877, `VolPathIntegrator::SampleLd()` 886.

➌ **14.8** Generalize the `LayeredBxDF` to allow the specification of an arbitrary number of layers with different media between them. You may want to review the improved sampling techniques for this case that were introduced by Gamboa et al. (2020). Verify that your implementation gives equivalent results to nested application of the `LayeredBxDF` and measure the efficiency difference between the two approaches.

---

[^14-1]: It is an integro-differential equation due to the integral over the sphere in the source function.
[^14-2]: Strictly speaking, two such paths ending at the same point on a light may have a different number of vertices due to different numbers of null-scattering vertices along the last segment. To simplify notation, we will here describe both as $n$ vertex paths with $\mathrm{p}_n$ the point on the light and $\mathrm{p}_{n-1}$ the scattering vertex immediately before it; we will index intermediate vertices on the last segment independently.
[^14-3]: This misconception periodically played a role in our initial development of this integrator.
[^14-4]: As was learned, painfully, during the implementation of this `BxDF`.
[^14-5]: It is admittedly unfriendly to provide an implementation of a method with a name that very clearly indicates that it should return a valid PDF and yet does not in fact do that, and to justify this with the fact that doing so is fine due to the current usage of the function. This represents a potentially gnarly bug lying in wait for someone in the future who might not expect this when extending the system. For that, our apologies in advance.

---

[↑ Back to top](#contents)
