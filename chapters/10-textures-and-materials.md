# Chapter 10: Textures and Materials

## Textures and Materials (wgpu)

```mermaid
flowchart LR
    A[UV Coordinates] --> B[Texture Lookup]
    B --> C[Material Parameters]
    C --> D[BSDF]
```


The BRDFs and BTDFs introduced in the previous chapter address only part of the problem of describing how a surface scatters light. Although they describe how light is scattered at a particular point on a surface, the renderer needs to determine which BRDFs and BTDFs are present at a point on a surface and what their parameters are. In this chapter, we describe a procedural shading mechanism that addresses this issue.

There are two components to pbrt’s approach: textures, which describe the spatial variation of some scalar or spectral value over a surface, and materials, which evaluate textures at points on surfaces in order to determine the parameters for their associated BSDFs. Separating the pattern generation responsibilities of textures from the implementations of reflection models via materials makes it easy to combine them in arbitrary ways, thereby making it easier to create a wide variety of appearances.

In pbrt, a texture is a fairly general concept: it is a function that maps points in some domain (e.g., a surface’s (u, v) parametric space or (x, y, z) object space) to values in some other domain (e.g., spectra or the real numbers). A variety of implementations of texture classes are available in the system. For example, pbrt has textures that represent zero-dimensional functions that return a constant in order to accommodate surfaces that have the same parameter value everywhere. Image map textures are two-dimensional functions that use a 2D array of pixel values to compute texture values at particular points (they are described in Section 10.4). There are even texture functions that compute values based on the values computed by other texture functions.

Textures may be a source of high-frequency variation in the final image. Figure 10.1 shows an image with severe aliasing due to a texture. Although the visual impact of this aliasing can be reduced with the nonuniform sampling techniques from Chapter 8, a better solution to this problem is to implement texture functions that adjust their frequency content based on the rate at which they are being sampled. For many texture functions, computing a reasonable approximation to the frequency content and antialiasing in this manner are not too difficult and are substantially more efficient than reducing aliasing by increasing the image sampling rate.

The first section of this chapter will discuss the problem of texture aliasing and general approaches to solving it. We will then describe the basic texture interface and illustrate its use with a variety of texture functions. After the textures have been defined, the last section, 10.5, introduces the Material interface and a number of implementations.

![Figure 10.1](images/img-985-209.png)

*Figure 10.1: Texture Aliasing. Both spheres have the same grid texture applied and each pixel is sampled at its center. No antialiasing is performed on the left sphere; because the texture has higher frequencies than the pixel sampling rate, severe aliasing results. On the right sphere, the texture function has taken into account the image sampling rate to prefilter its function and remove high-frequency detail, giving an antialiased result even with a single sample in each pixel.*

## 10.1 TEXTURE SAMPLING AND ANTIALIASING

The sampling task from Chapter 8 was a frustrating one since the aliasing problem was known to be unsolvable from the start. The infinite frequency content of geometric edges and hard shadows guarantees aliasing in the final images, no matter how high the image sampling rate. (Our only consolation is that the visual impact of this remaining aliasing can be reduced to unobjectionable levels with a sufficient number of well-placed samples.) Fortunately, things are not this difficult from the start for textures: either there is often a convenient analytic form of the texture function available, which makes it possible to remove excessively high frequencies before sampling it, or it is possible to be careful when evaluating the function so as not to introduce high frequencies in the first place. When this problem is carefully addressed in texture implementations, as is done through the rest of this chapter, there is usually no need for more than one sample per pixel in order to render an image without texture aliasing. (Of course, sufficiently reducing Monte Carlo noise from lighting calculations may be another matter.) Two problems must be addressed in order to remove aliasing from texture functions:

1. The sampling rate in texture space must be computed. The screen-space sampling rate is known from the image resolution and pixel sampling rate, but here we need to determine the resulting sampling rate on a surface in the scene in order to find the rate at which the texture function is being sampled.
2. Given the texture sampling rate, sampling theory must be applied to guide the computation of a texture value that does not have higher frequency variation than can be represented by the sampling rate (e.g., by removing excess frequencies beyond the Nyquist limit from the texture function).

These two issues will be addressed in turn throughout the rest of this section.

### 10.1.1 FINDING THE TEXTURE SAMPLING RATE

Consider an arbitrary texture function that is a function of position, T (p), defined on a surface in the scene. If we ignore the complications introduced by visibility—the possibility that another object may occlude the surface at nearby image samples or that the surface may have a limited extent on the image plane—this texture function can also be expressed as a function over points (x, y) on the image plane, T (f (x, y)), where f (x, y) is the function that maps image points to points on the surface. Thus, T (f (x, y)) gives the value of the texture function as seen at image position (x, y).

As a simple example of this idea, consider a 2D texture function T (s, t) applied to a quadrilateral that is perpendicular to the z axis and has corners at the world-space points (0, 0, 0), (1, 0, 0), (1, 1, 0), and (0, 1, 0). If an orthographic camera is placed looking down the z axis such that the quadrilateral precisely fills the image plane and if points p on the quadrilateral are mapped to 2D (s, t) texture coordinates by s = px t = py,

then the relationship between (s, t) and screen (x, y) pixels is straightforward:

where the overall image resolution is (xr, yr) (Figure 10.2). Thus, given a sample spacing of one pixel in the image plane, the sample spacing in (s, t) texture parameter space is (1/xr, 1/yr), and the texture function must remove any detail at a higher frequency than can be represented at that sampling rate.

![Figure 10.2](images/img-987-210.png)

*Figure 10.2: If a quadrilateral is viewed with an orthographic perspective such that the quadrilateral precisely fills the image plane, it is easy to compute the relationship between the sampling rate in (x, y) pixel coordinates and the texture sampling rate.*

This relationship between pixel coordinates and texture coordinates, and thus the relationship between their sampling rates, is the key bit of information that determines the maximum frequency content allowable in the texture function. As a slightly more complex example, given a triangle with (u, v) texture coordinates at its vertices and viewed with a perspective projection, it is possible to analytically find the differences in u and v across the sample points on the image plane. This approach was the basis of texture antialiasing in graphics processors before they became programmable.

For more complex scene geometry, camera projections, and mappings to texture coordinates, it is much more difficult to precisely determine the relationship between image positions and texture parameter values. Fortunately, for texture antialiasing, we do not need to be able to evaluate f (x, y) for arbitrary (x, y) but just need to find the relationship between changes in pixel sample position and the resulting change in texture sample position at a particular point on the image. This relationship is given by the partial derivatives of this function, ∂f/∂x and ∂f/∂y. For example, these can be used to

find a first-order approximation to the value of f,

If these partial derivatives are changing slowly with respect to the distances x′ − x and y′ − y, this is a reasonable approximation. More importantly, the values of these partial derivatives give an approximation to the change in texture sample position for a shift of one pixel in the x and y directions, respectively, and thus directly yield the texture sampling rate. For example, in the previous quadrilateral example, ds/dx = 1/xr, ds/dy = 0, dt/dx = 0, and dt/dy = 1/yr.

The key to finding the values of these derivatives in the general case lies in values from the RayDifferential structure, which was defined in Section 3.6.1. This structure is initialized for each camera ray by the Camera::GenerateRayDifferential() method; it contains not only the ray being traced through the scene but also two additional rays, one offset horizontally one pixel sample from the camera ray and the other offset vertically by one pixel sample. All the geometric ray intersection routines use only the main camera ray for their computations; the auxiliary rays are ignored (this is easy to do because RayDifferential is a subclass of Ray).

We can use the offset rays to estimate the partial derivatives of the mapping p(x, y) from image position to world-space position and the partial derivatives of the mappings u(x, y) and v(x, y) from (x, y) to (u, v) parametric coordinates, giving the partial derivatives of rendering-space positions ∂p/ ∂x and ∂p/∂y and the derivatives of (u, v) parametric coordinates du/dx, dv/dx, du/dy, and dv/dy. In Section 10.2, we will see how these can be used to compute the screen-space derivatives of arbitrary quantities based on p or (u, v) and consequently the sampling rates of these quantities. The values of these derivatives at the intersection point are stored in the SurfaceInteraction structure.

〈SurfaceInteraction Public Members〉 +≡ 138

```rust
   Vector3f dpdx, dpdy;
   Float dudx = 0, dvdx = 0, dudy = 0, dvdy = 0;
```

The SurfaceInteraction::ComputeDifferentials() method computes these values. It is called by SurfaceInteraction::GetBSDF() before the Material’s GetBxDF() method is called so that these values will be available for any texture evaluation routines that are called by the material.

Ray differentials are not available for all rays traced by the system—for example, rays starting from light sources traced for photon mapping or bidirectional path tracing. Further, although we will see how to compute ray differentials after rays undergo specular reflection and transmission in Section 10.1.3, how to compute ray differentials after diffuse reflection is less clear. In cases like those as well as the corner case where one of the differentials’ directions is perpendicular to the surface normal, which leads to undefined numerical values in the following, an alternative approach based on approximating the ray differentials of a ray from the camera to the intersection point is used.

```text
    Camera::GenerateRayDifferential() 207
    Float 23
    Material 674
    Material::GetBxDF() 674
    Ray 95
    RayDifferential 96
    SurfaceInteraction 138
    SurfaceInteraction::ComputeDifferentials() 637
    SurfaceInteraction::GetBSDF() 682
    Vector3f 86
```

〈SurfaceInteraction Method Definitions〉 ≡ void SurfaceInteraction::ComputeDifferentials(const RayDifferential &ray,

```rust
             Camera camera, int samplesPerPixel) {
         if (ray.hasDifferentials && Dot(n, ray.rxDirection) != 0 &&
             Dot(n, ray.ryDirection) != 0) {
             〈Estimate screen-space change in p using ray differentials 637〉
         } else {
             〈Approximate screen-space change in p based on camera projection 641〉
         }
         〈Estimate screen-space change in (u, v) 641〉
   }
```

The key to estimating the derivatives is the assumption that the surface is locally flat with respect to the sampling rate at the point being shaded. This is a reasonable approximation in practice, and it is hard to do much better. Because ray tracing is a point-sampling technique, we have no additional information about the scene in between the rays we have traced. For highly curved surfaces or at silhouette edges, this approximation can break down, though this is rarely a source of noticeable error.

For this approximation, we need the plane through the point p intersected by the main ray that is tangent to the surface. This plane is given by the implicit equation ax + by + cz + d = 0,

where a = nx, b = ny, c = nz, and d = −(n · p). We can then compute the intersection points px and py between the auxiliary rays rx and ry and this plane (Figure 10.3). These new points give an approximation to the partial derivatives of position on the surface ∂p/∂x and ∂p/∂y, based on forward

differences:

Because the differential rays are offset one pixel sample in each direction, there is no need to divide these differences by a Δ value, since Δ = 1.

〈Estimate screen-space change in p using ray differentials〉 ≡ 637

```rust
   〈Compute auxiliary intersection points with plane, px and py 638〉
   dpdx = px - p();
   dpdy = py - p();
```

![Figure 10.3](images/img-990-211.png)

*Figure 10.3: By approximating the local surface geometry at the intersection point with the tangent plane through p, approximations to the points at which the auxiliary rays rx and ry would intersect the surface can be found by finding their intersection points with the tangent plane px and py.*

```text
    Camera 206
    Dot() 89
    Interaction::n 137
    Interaction::p() 137
    RayDifferential 96
    RayDifferential:: hasDifferentials 96
    RayDifferential::rxDirection 96
    RayDifferential::ryDirection 96
    SurfaceInteraction::dpdx 636
    SurfaceInteraction::dpdy 636
```

The ray–plane intersection algorithm described in Section 6.1.2 gives the t value where a ray described by origin o and direction d intersects a plane described by ax + by + cz + d = 0:

To compute this value for the two auxiliary rays, the plane’s d coefficient is computed first. It is not necessary to compute the a, b, and c coefficients, since they are available in n. We can then apply the formula directly.

〈Compute auxiliary intersection points with plane, px and py〉 ≡ 637

```rust
   Float d = -Dot(n, Vector3f(p()));
   Float tx = (-Dot(n, Vector3f(ray.rxOrigin)) - d) /
               Dot(n, ray.rxDirection);
   Point3f px = ray.rxOrigin + tx * ray.rxDirection;
```

```rust
    Float ty = (-Dot(n, Vector3f(ray.ryOrigin)) - d) /
               Dot(n, ray.ryDirection);
    Point3f py = ray.ryOrigin + ty * ray.ryDirection;
```

For cases where ray differentials are not available, we will add a method to the Camera interface that returns approximate values for ∂p/∂x and ∂p/∂y at a point on a surface in the scene. These should be a reasonable approximation to the differentials of a ray from the camera that found an intersection at the given point. Cameras’ implementations of this method must return reasonable results even for points outside of their viewing volumes for which they cannot actually generate rays.

〈Camera Interface〉 +≡ 206

```rust
   void Approximate_dp_dxy(Point3f p, Normal3f n, Float time,
       int samplesPerPixel, Vector3f *dpdx, Vector3f *dpdy) const;
```

CameraBase provides an implementation of an approach to approximating these differentials that is based on the minimum of the camera ray differentials across the entire image. Because all of pbrt’s current camera implementations inherit from CameraBase, the following method takes care of all of them.

〈CameraBase Public Methods〉 +≡ 212

```rust
   void Approximate_dp_dxy(Point3f p, Normal3f n, Float time,
           int samplesPerPixel, Vector3f *dpdx, Vector3f *dpdy) const {
       〈Compute tangent plane equation for ray differential intersections 640〉
       〈Find intersection points for approximated camera differential rays 640〉
       〈Estimate ∂p/∂x and ∂p/∂y in tangent plane at intersection point 641〉
   }
```

This method starts by orienting the camera so that the camera-space z axis is aligned with the vector from the camera position to the intersection point. It then uses lower bounds on the spread of rays over the image that are provided by the camera to find approximate differential rays. It then intersects these rays with the tangent plane at the intersection point. (See Figure 10.4.) CameraBase 212 Dot() 89 Float 23 Interaction::n 137 Interaction::p() 137 Normal3f 94 Point3f 92 RayDifferential::rxDirection 96 RayDifferential::rxOrigin 96 RayDifferential::ryDirection 96

RayDifferential::ryOrigin 96 Vector3f 86

There are a number of sources of error in this approximation. Beyond the fact that it does not account for how light was scattered at intermediate surfaces for multiple-bounce ray paths, there is also the fact that it is based on the minimum of the camera’s differentials for all rays. In general, it tries to underestimate those derivatives rather than overestimate them, as we prefer aliasing over blurring here. The former error can at least be addressed with additional pixel samples. In order to give a sense of the impact of some of these approximations, Figure 10.5 has visualization that compares the local area estimated by those derivatives at intersections to the area computed using the actual ray differentials generated by the camera.

![Figure 10.4](images/img-992-212.png)

*Figure 10.4: CameraBase::Approximate_dp_dxy() effectively reorients the camera to point at the provided intersection point. In camera space, the ray to the intersection then has origin (0, 0, 0) and direction (0, 0, 1). The extent of ray differentials on the tangent plane defined by the surface normal at the intersection point can then be found.*

![Figure 10.5](images/img-993-213.png)

*Figure 10.5: Visualization of the Ratio of Filter Areas Estimated by Regular Ray Differentials to Areas Estimated by CameraBase::Approximate_dp_dxy(). We represent the filter area as the product max(|du/dx|, |du/dy|) max(|dv/dx|, |dv/dy|) and visualize the base 2 logarithm of the ratio of areas computed by the two techniques. Log 2 ratios greater than 0 indicate that the camera-based approximation estimated a larger filter area.*

```text
   CameraBase::Approximate_dp_dxy() 638
```

For the first step of the algorithm, we have an intersection point in rendering space p that we would like to transform into a coordinate system where it is along the z axis with the camera at the origin. Transforming to camera space gets us started and an additional rotation that transforms the vector from the origin to the intersection point to be aligned with z finishes the job. The d coefficient of the plane equation can then be found by taking the dot product of the transformed point and surface normal. Because the x and y components of the transformed point are equal to 0, the dot product can be optimized to be a single multiply.

〈Compute tangent plane equation for ray differential intersections〉 ≡ 638

Point3f pCamera = CameraFromRender(p, time); Transform DownZFromCamera = RotateFromTo(Normalize(Vector3f(pCamera)), Vector3f(0, 0, 1)); Point3f pDownZ = DownZFromCamera(pCamera); Normal3f nDownZ = DownZFromCamera(CameraFromRender(n, time)); Float d = nDownZ.z * pDownZ.z;

Camera implementations that inherit from CameraBase and use this method must initialize the following member variables with values that are lower bounds on each of the respective position and direction differentials over all the pixels in the image. 〈CameraBase Protected Members〉 +≡ 212

Vector3f minPosDifferentialX, minPosDifferentialY; Vector3f minDirDifferentialX, minDirDifferentialY;

The main ray in this coordinate system has origin (0, 0, 0) and direction (0, 0, 1). Adding the position and direction differential vectors to those gives the origin and direction of each differential ray. Given those, the same calculation as earlier gives us the t values for the ray–plane intersections for the differential rays and thence the intersection points. 〈Find intersection points for approximated camera differential rays〉 ≡ 638

Ray xRay(Point3f(0,0,0) + minPosDifferentialX, Vector3f(0,0,1) + minDirDifferentialX); Float tx = -(Dot(nDownZ, Vector3f(xRay.o)) - d) / Dot(nDownZ, xRay.d); Ray yRay(Point3f(0,0,0) + minPosDifferentialY, Vector3f(0,0,1) + minDirDifferentialY); Float ty = -(Dot(nDownZ, Vector3f(yRay.o)) - d) / Dot(nDownZ, yRay.d); Point3f px = xRay(tx), py = yRay(ty);

For an orthographic camera, these differentials can be computed directly. There is no change in the direction vector, and the position differentials are the same at every pixel. Their values are already computed in the OrthographicCamera constructor, so can be used directly to initialize the base class’s member variables. 〈Compute minimum differentials for orthographic camera〉 ≡ 218

minDirDifferentialX = minDirDifferentialY = Vector3f(0, 0, 0); minPosDifferentialX = dxCamera; minPosDifferentialY = dyCamera;

All the other cameras call FindMinimumDifferentials(), which estimates these values by sampling at many points across the diagonal of the image and storing the minimum of all the differentials encountered. That function is not very interesting, so it is not included here. 〈Compute minimum differentials for PerspectiveCamera〉 ≡ 220

FindMinimumDifferentials(this);

```text
   CameraBase::CameraFromRender() 214
   CameraBase::FindMinimumDifferentials() 640
   CameraBase:: minDirDifferentialX 640
   CameraBase:: minDirDifferentialY 640
   CameraBase:: minPosDifferentialX 640
   CameraBase:: minPosDifferentialY 640
   Dot() 89
   Float 23
   Normal3f 94
   OrthographicCamera 217
   OrthographicCamera::dxCamera 218
   OrthographicCamera::dyCamera 218
   Point3f 92
   Ray 95
   RotateFromTo() 127
   Transform 120
   Vector3f 86
```

Given the intersection points px and py, ∂p/∂x and ∂p/∂y can now be estimated by taking their differences with the main intersection point. To get final estimates of the partial derivatives, these vectors must be transformed back out into rendering space and scaled to account for the actual pixel sampling rate. As with the initial ray differentials that were generated in the 〈Scale camera ray differentials based on image sampling rate〉 fragment, these are scaled to account for the pixel sampling rate. 〈Estimate ∂p/∂x and ∂p/∂y in tangent plane at intersection point〉 ≡ 638

Float sppScale = GetOptions().disablePixelJitter ? 1 : std::max<Float>(.125, 1 / std::sqrt((Float)samplesPerPixel)); *dpdx = sppScale * RenderFromCamera(DownZFromCamera.ApplyInverse(px - pDownZ), time); *dpdy = sppScale * RenderFromCamera(DownZFromCamera.ApplyInverse(py - pDownZ), time);

A call to this method takes care of computing the ∂p/∂x and ∂p/∂y differentials in the ComputeDifferentials() method. 〈Approximate screen-space change in p based on camera projection〉 ≡ 637

camera.Approximate_dp_dxy(p(), n, time, samplesPerPixel, &dpdx, &dpdy);

We now have both the partial derivatives ∂p/∂u and ∂p/∂v as well as, one way or another, ∂p/∂x and ∂p/∂y. From them, we would now like to compute du/dx, du/dy, dv/dx, and dv/dy. Using the chain rule, we can find that (∂p/∂y has a similar expression with du/dx replaced by du/dy and dv/dx replaced by dv/dy.) Equation (10.1) can be written as a matrix equation where the two following matrices that include ∂p have three rows, one for

each of p’s x, y, and z components: This is an overdetermined linear system since there are three equations but only two unknowns, du/dx and dv/dx. An effective solution approach in this case is to apply linear least squares, which says that for a linear system of the form Ax = b with A and b known, the least-squares solution for x is given by

In this case, A = ( ∂p/∂u ∂p/∂v ), b = ( ∂p/∂x ), and x = ( du/dx dv/dx )T. 〈Estimate screen-space change in (u, v)〉 ≡ 637, 964

〈Compute AT A and its determinant 642〉 〈Compute AT b for x and y 642〉 〈Compute u and v derivatives with respect to x and y 642〉 〈Clamp derivatives of u and v to reasonable values 642〉

```text
   BasicPBRTOptions:: disablePixelJitter 1031
   Camera::Approximate_dp_dxy() 638
   CameraBase::RenderFromCamera() 214
   Float 23
   GetOptions() 1032
   Interaction::n 137
   Interaction::p() 137
   Interaction::time 137
```

```text
   SurfaceInteraction::dpdx 636
   SurfaceInteraction::dpdy 636
   Transform::ApplyInverse() 130
```

AT A is a 2 × 2 matrix with elements given by dot products of partial

derivatives of position:

Its inverse is

Note that in both matrices the two off-diagonal entries are equal. Thus, the fragment that computes the entries of AT A only needs to compute three values. The inverse of the matrix determinant is computed here as well. If its value is infinite, the linear system cannot be solved; setting invDet to 0 causes the subsequently computed derivatives to be 0, which leads to point-sampled textures, the best remaining option in that case.

〈Compute AT A and its determinant〉 ≡

Float ata00 = Dot(dpdu, dpdu), ata01 = Dot(dpdu, dpdv); Float ata11 = Dot(dpdv, dpdv); Float invDet = 1 / DifferenceOfProducts(ata00, ata11, ata01, ata01); invDet = IsFinite(invDet) ? invDet : 0.f;

The AT b portion of the solution is easily computed. For the derivatives with respect to screen-space x, we have the two-element matrix

The solution for screen-space y is analogous.

〈Compute AT b for x and y〉 ≡

Float atb0x = Dot(dpdu, dpdx), atb1x = Dot(dpdv, dpdx); Float atb0y = Dot(dpdu, dpdy), atb1y = Dot(dpdv, dpdy);

The solution to Equation (10.2) for each partial derivative can be found by taking the product of Equations (10.3) and (10.4). We will gloss past the algebra; its result can be directly expressed in terms of the values computed so far. 〈Compute u and v derivatives with respect to x and y〉 ≡ 641

dudx = DifferenceOfProducts(ata11, atb0x, ata01, atb1x) * invDet; dvdx = DifferenceOfProducts(ata00, atb1x, ata01, atb0x) * invDet; dudy = DifferenceOfProducts(ata11, atb0y, ata01, atb1y) * invDet; dvdy = DifferenceOfProducts(ata00, atb1y, ata01, atb0y) * invDet;

In certain tricky cases (e.g., with highly distorted (u, v) parameterizations or at object silhouette edges), the estimated partial derivatives may be infinite or have very large magnitudes. It is worth clamping them to reasonable values in that case to prevent overflow and not-a-number values in subsequent computations that are based on them. 〈Clamp derivatives of u and v to reasonable values〉 ≡ 641

dudx = IsFinite(dudx) ? Clamp(dudx, -1e8f, 1e8f) : 0.f; dvdx = IsFinite(dvdx) ? Clamp(dvdx, -1e8f, 1e8f) : 0.f; dudy = IsFinite(dudy) ? Clamp(dudy, -1e8f, 1e8f) : 0.f; dvdy = IsFinite(dvdy) ? Clamp(dvdy, -1e8f, 1e8f) : 0.f;

### 10.1.2 RAY DIFFERENTIALS AT MEDIUM TRANSITIONS

Now is a good time to take care of another detail related to ray differentials: recall from Section 9.1.5 that materials may return an unset BSDF to indicate an interface between two scattering media that does not itself scatter light. In this case, it is necessary to spawn a new ray in the same direction, but past the intersection on the surface. In this case we would like the effect of the ray differentials to be the same as if no scattering had occurred. This can be achieved by setting the differential origins to the points given by evaluating the ray equation at the intersection t (see Figure 10.6).

```text
    BSDF 544
    Clamp() 1033
    DifferenceOfProducts() 1044
    Dot() 89
    Float 23
    IsFinite() 364
```

![Figure 10.6](images/img-1000-214.png)

*Figure 10.6: When a ray intersects a surface that delineates the boundary between two media, a new ray is spawned on the other side of the boundary. If the origins of this ray’s differentials are set by evaluating the ray equation for the original differentials at the intersection t, then the new ray will represent the same footprint as the original one when it subsequently intersects a surface.*

〈SurfaceInteraction Method Definitions〉 +≡ void SurfaceInteraction::SkipIntersection(RayDifferential *ray, Float t) const { *((Ray *)ray) = SpawnRay(ray->d); if (ray->hasDifferentials) { ray->rxOrigin = ray->rxOrigin + t * ray->rxDirection; ray->ryOrigin = ray->ryOrigin + t * ray->ryDirection; } }

⋆ 10.1.3 RAY DIFFERENTIALS FOR SPECULAR REFLECTION AND TRANSMISSION

Given the effectiveness of ray differentials for finding filter regions for texture antialiasing for camera rays, it is useful to extend the method to make it possible to determine texture-space sampling rates for objects that are seen indirectly via specular reflection or refraction; objects seen in mirrors, for example, should not have texture aliasing, identical to the case for directly visible objects. Igehy (1999) developed an elegant solution to

the problem of how to find the appropriate differential rays for specular reflection and refraction, which is the approach used in pbrt.1 Figure 10.7 illustrates the difference that proper texture filtering for specular reflection and transmission can make: it shows a glass ball and a mirrored ball on a plane with a texture map containing high-frequency components. Ray differentials ensure that the images of the texture seen via reflection and refraction from the balls are free of aliasing artifacts. Here, ray differentials eliminate aliasing without excessively blurring the texture.

```text
   Float 23
   Ray 95
   Ray::d 95
   RayDifferential 96
   RayDifferential:: hasDifferentials 96
   RayDifferential::rxDirection 96
   RayDifferential::rxOrigin 96
   RayDifferential::ryDirection 96
   RayDifferential::ryOrigin 96
   SurfaceInteraction::SpawnRay() 645
```

To compute the reflected or transmitted ray differentials at a surface intersection point, we need an approximation to the rays that would have been traced at the intersection points for the two offset rays in the ray differential that hit the surface (Figure 10.8). The new ray for the main ray is found by sampling the BSDF, so here we only need to compute the outgoing rays for the rx and ry differentials. This task is handled by another SurfaceInteraction::SpawnRay() variant that takes an incident ray differential as well as information about the BSDF and the type of scattering that occurred.

![Figure 10.7](images/img-1002-215.png)

*Figure 10.7: (a) Tracking ray differentials for reflected and refracted rays ensures that the image map texture seen in the balls is filtered to avoid aliasing. The left ball is glass, exhibiting reflection and refraction, and the right ball is a mirror, just showing reflection. Note that the texture is well filtered over both of the balls. (b) shows the aliasing artifacts that are present if ray differentials are not used.*

![Figure 10.8](images/img-1003-216.png)

*Figure 10.8: The specular reflection formula gives the direction of the reflected ray at a point on a surface. An offset ray for a ray differential r′ (dashed line) will generally intersect the surface at a different point and be reflected in a different direction. The new direction is affected by the different surface normal at the point as well as by the offset ray’s different incident direction. The computation to find the reflected direction for the offset ray in pbrt estimates the change in reflected direction as a function of image-space position and approximates the ray differential’s direction with the main ray’s direction added to the estimated change in direction.*

〈SurfaceInteraction Method Definitions〉 +≡ RayDifferential SurfaceInteraction::SpawnRay( const RayDifferential &rayi, const BSDF &bsdf, Vector3f wi, int flags, Float eta) const { RayDifferential rd(SpawnRay(wi)); if (rayi.hasDifferentials) { 〈Compute ray differentials for specular reflection or transmission 645〉 } 〈Squash potentially troublesome differentials〉 return rd; }

It is not well defined what the ray differentials should be in the case of non-specular scattering. Therefore, this method handles the two types of specular scattering only; for all other types of rays, approximate differentials will be computed at their subsequent intersection points with Camera::Approximate_dp_dxy().

〈Compute ray differentials for specular reflection or transmission〉 ≡ 645

〈Compute common factors for specular ray differentials 645〉 if (flags == BxDFFlags::SpecularReflection) { 〈Initialize origins of specular differential rays 645〉 〈Compute differential reflected directions 646〉 } else if (flags == BxDFFlags::SpecularTransmission) { 〈Initialize origins of specular differential rays 645〉 〈Compute differential transmitted directions 646〉 }

```text
   BSDF 544
   BxDFFlags::SpecularReflection 539
   BxDFFlags::SpecularTransmission 539
   Camera::Approximate_dp_dxy() 638
   Float 23
   Interaction::p() 137
   Interaction::SpawnRay() 383
   Normal3f 94
   RayDifferential 96
   RayDifferential:: hasDifferentials 96
   RayDifferential::rxDirection 96
   RayDifferential::ryDirection 96
   SurfaceInteraction::dpdx 636
   SurfaceInteraction::dpdy 636
   SurfaceInteraction::dudx 636
   SurfaceInteraction::dudy 636
   SurfaceInteraction::dvdx 636
   SurfaceInteraction::dvdy 636
   SurfaceInteraction::shading 139
   SurfaceInteraction:: shading::dndu 139
   SurfaceInteraction:: shading::dndv 139
   SurfaceInteraction:: shading::n 139
   Vector3f 86
```

A few variables will be used for both types of scattering, including the partial derivatives of the surface normal with respect to x and y on the image and ∂n/∂x and ∂n/∂y, which are computed using the chain rule. 〈Compute common factors for specular ray differentials〉 ≡ 645

Normal3f n = shading.n; Normal3f dndx = shading.dndu * dudx + shading.dndv * dvdx; Normal3f dndy = shading.dndu * dudy + shading.dndv * dvdy; Vector3f dwodx = -rayi.rxDirection - wo, dwody = -rayi.ryDirection - wo;

For both reflection and transmission, the origin of each differential ray can be found using the already-computed approximations of how much the surface position changes with respect to (x, y) position on the image plane ∂p/∂x and ∂p/∂y. 〈Initialize origins of specular differential rays〉 ≡ 645

```rust
   rd.hasDifferentials = true;
   rd.rxOrigin = p() + dpdx;
   rd.ryOrigin = p() + dpdy;
```

Finding the directions of these rays is slightly trickier. If we know how much the reflected direction ωi changes with respect to a shift of a pixel sample in the x and y directions on the image plane, we can use this information to approximate the direction of the offset rays. For example, the direction for the ray offset in x is Recall from Equation (9.1) that for a normal n and outgoing direction ωo the direction for perfect specular reflection is ωi = −ωo + 2(ωo · n)n.

The partial derivatives of this expression are easily computed:

Using the properties of the dot product, it can further be shown that

The value of ∂ωo/∂x has already been computed from the difference between the direction of the ray differential’s main ray and the direction of the rx offset ray, and all the other necessary quantities are readily available from the SurfaceInteraction. 〈Compute differential reflected directions〉 ≡ 645

Float dwoDotn_dx = Dot(dwodx, n) + Dot(wo, dndx); Float dwoDotn_dy = Dot(dwody, n) + Dot(wo, dndy); rd.rxDirection = wi - dwodx +

```rust
       2 * Vector3f(Dot(wo, n) * dndx + dwoDotn_dx * n);
   rd.ryDirection = wi - dwody +
       2 * Vector3f(Dot(wo, n) * dndy + dwoDotn_dy * n);
```

A similar process of differentiating the equation for the direction of a specularly transmitted ray, Equation (9.4), gives the equation to find the differential change in the transmitted direction. pbrt computes refracted

rays as where n is flipped if necessary to lie in the same hemisphere as ωo, and where η is the relative index of refraction from ωo’s medium to ωi’s medium. If we denote the term in brackets by μ, then we have ωi = −(1/η)ωo + μn. Taking the partial derivative in x, we have

Using some of the values found from computing specularly reflected ray differentials, we can find that we already know how to compute all of these values except for ∂μ/∂x. 〈Compute differential transmitted directions〉 ≡ 645

〈Find oriented surface normal for transmission 647〉 〈Compute partial derivatives of μ 647〉 rd.rxDirection = wi - eta * dwodx + Vector3f(mu * dndx + dmudx * n); rd.ryDirection = wi - eta * dwody + Vector3f(mu * dndy + dmudy * n);

Before we get to the computation of μ’s partial derivatives, we will start by reorienting the surface normal if necessary so that it lies on the same side of the surface as ωo. This matches pbrt’s computation of refracted ray directions.

```text
   Dot() 89
   Float 23
   Vector3f 86
```

〈Find oriented surface normal for transmission〉 ≡ 646

if (Dot(wo, n) < 0) {

```rust
       n = -n;
       dndx = -dndx;
       dndy = -dndy;
   }
```

Returning to μ and considering ∂μ/∂x, we have

Its first term can be evaluated with already known values. For the second

term, we will start with Snell’s law, which gives If we square both sides of the equation and take the partial derivative ∂/∂x, we find

We now can solve for ∂ cos θi/∂x:

Putting it all together and simplifying, we have

The partial derivative in y is analogous and the implementation follows. 〈Compute partial derivatives of μ〉 ≡ 646

Float dwoDotn_dx = Dot(dwodx, n) + Dot(wo, dndx); Float dwoDotn_dy = Dot(dwody, n) + Dot(wo, dndy); Float mu = Dot(wo, n) / eta - AbsDot(wi, n); Float dmudx = dwoDotn_dx * (1/eta + 1/Sqr(eta) * Dot(wo, n) / Dot(wi, n)); Float dmudy = dwoDotn_dy * (1/eta + 1/Sqr(eta) * Dot(wo, n) / Dot(wi, n));

If a ray undergoes many specular bounces, ray differentials sometimes drift off to have very large magnitudes, which can leave a trail of infinite and not-a-number values in their wake when they are used for texture filtering calculations. Therefore, the final fragment in this SpawnRay() method computes the squared length of all the differentials. If any is greater than 1016, the ray differentials are discarded and the RayDifferential hasDifferentials value is set to false. The fragment that handles this, 〈Squash potentially troublesome differentials〉, is simple and thus not included here.

```text
    AbsDot() 90
    Dot() 89
    Float 23
    Sqr() 1034
```

### 10.1.4 FILTERING TEXTURE FUNCTIONS

To eliminate texture aliasing, it is necessary to remove frequencies in texture functions that are past the Nyquist limit for the texture sampling rate. The goal is to compute, with as few approximations as possible, the result of the ideal texture resampling process, which says that in order to evaluate a texture function T at a point (x, y) on the image without aliasing, we must first band-limit it, removing frequencies beyond the Nyquist limit by convolving it with the sinc filter:

where, as in Section 10.1.1, f (x, y) maps pixel locations to points in the texture function’s domain. The band-limited function Tb in turn should then be convolved with the pixel filter g(x, y) centered at the (x, y) point on the screen at which we want to evaluate the texture function:

This gives the theoretically perfect value for the texture as projected onto the screen.2 In practice, there are many simplifications that can be made to this process. For example, a box filter may be used for the band-limiting step, and the

second step is usually ignored completely, effectively acting as if the pixel filter were a box filter, which makes it possible to do the antialiasing work completely in texture space. (The EWA filtering algorithm in Section 10.4.4 is a notable exception in that it assumes a Gaussian pixel filter.) Assuming box filters then if, for example, the texture function is defined over parametric (u, v) coordinates, the filtering task is to average it over a region

in (u, v): The extent of the filter region can be determined using the derivatives from the previous sections—for example, setting

and similarly for v0 and v1 to conservatively specify the box’s extent.

The box filter is easy to use, since it can be applied analytically by computing the average of the texture function over the appropriate region. Intuitively, this is a reasonable approach to the texture filtering problem, and it can be computed directly for many texture functions. Indeed, through the rest of this chapter, we will often use a box filter to average texture function values between samples and informally use the term filter region to describe the area being averaged over. This is the most common approach when filtering texture functions. Even the box filter, with all of its shortcomings, gives acceptable results for texture filtering in many cases. One factor that helps is the fact that a number of samples are usually taken in each pixel. Thus, even if the filtered texture values used in each one are suboptimal, once they are filtered by the pixel reconstruction filter, the end result generally does not suffer too much. An alternative to using the box filter to filter texture functions is to use the observation that the effect of the ideal sinc filter is to let frequency components below the Nyquist limit pass through unchanged but to remove frequencies past it. Therefore, if we know the frequency content of the texture function (e.g., if it is a sum of terms, each one with known frequency content), then if we replace the high-frequency terms with their average values, we are effectively doing the work of the sinc prefilter. This approach is known as clamping.

Finally, for texture functions where none of these techniques is easily applied, a final option is supersampling—the function is evaluated and filtered at multiple locations near the main evaluation point, thus increasing the sampling rate in texture space. If a box filter is used to filter these sample values, this is equivalent to averaging the value of the function. This approach can be expensive if the texture function is complex to evaluate, and as with image sampling, a very large number of samples may be needed to remove aliasing. Although this is a brute-force solution, it is still more efficient than increasing the image sampling rate, since it does not incur the cost of tracing more rays through the scene.

## 10.2 TEXTURE COORDINATE GENERATION

Almost all the textures in this chapter are functions that take a 2D or 3D coordinate and return a texture value. Sometimes there are obvious ways to choose these texture coordinates; for parametric surfaces, such as the quadrics in Chapter 6, there is a natural 2D (u, v) parameterization of the surface, and for all types of surfaces the shading point p is a natural choice for a 3D coordinate. In other cases, there is no natural parameterization, or the natural parameterization may be undesirable. For instance, the (u, v) values near the poles of spheres are severely distorted. Therefore, this section introduces classes that provide an interface to different techniques for generating these parameterizations as well as a number of implementations of them. The Texture implementations later in this chapter store a tagged pointer to a 2D or 3D mapping function as appropriate and use it to compute the texture coordinates at each point at which they are evaluated. Thus, it is easy to add new mappings to the system without having to modify all the Texture implementations, and different mappings can be used for different textures associated with the same surface. In pbrt, we will use the convention that 2D texture coordinates are denoted by (s, t); this helps make clear the distinction between the intrinsic (u, v) parameterization of the underlying surface and the possibly different coordinate values used for texturing.

TextureMapping2D defines the interface for 2D texture coordinate generation. It is defined in the file base/texture.rs. The implementations of the texture mapping classes are in textures.rs and textures.rs.

〈TextureMapping2D Definition〉 ≡ class TextureMapping2D : public TaggedPointer<UVMapping, SphericalMapping, CylindricalMapping, PlanarMapping> { public: 〈TextureMapping2D Interface 650〉 };

```text
   CylindricalMapping 653
   PlanarMapping 654
   SphericalMapping 652
   TaggedPointer 1073
   TextureEvalContext 650
   TextureMapping2D 649
   UVMapping 650
```

The TextureMapping2D interface consists of a single method, Map(). It is given a TextureEval Context that stores relevant geometric information at the shading point and returns a small structure, TexCoord2D, that stores the (s, t) texture coordinates and estimates for the change in (s, t) with respect to pixel x and y coordinates so that textures that use the mapping can determine the (s, t) sampling rate and filter accordingly. 〈TextureMapping2D Interface〉 ≡ 649

TexCoord2D Map(TextureEvalContext ctx) const; 〈TexCoord2D Definition〉 ≡ struct TexCoord2D { Point2f st; Float dsdx, dsdy, dtdx, dtdy; };

In previous versions of pbrt, the Map() interface was defined to take a complete Surface Interaction; the TextureEvalContext structure did not exist. For this version, we have tightened up the interface to only include specific values that are useful for texture coordinate generation.

This change was largely motivated by the GPU rendering path: with the CPU renderer, all the relevant information is already at hand in the functions that call the Map() methods; most likely the SurfaceInteraction is already in the CPU cache. On the GPU, the necessary values have to be read from off-chip memory. TextureEvalContext makes it possible for the GPU renderer to only read the necessary values from memory, which in turn has measurable performance benefits. TextureEvalContext provides three constructors, not included here. Two initialize the various fields using corresponding values from either an Interaction or a SurfaceInteraction and the third allows specifying them directly. 〈TextureEvalContext Definition〉 ≡ struct TextureEvalContext { 〈TextureEvalContext Public Methods〉 Point3f p; Vector3f dpdx, dpdy; Normal3f n; Point2f uv; Float dudx = 0, dudy = 0, dvdx = 0, dvdy = 0; };

### 10.2.1 (u, v) MAPPING

UVMapping uses the (u, v) coordinates in the TextureEvalContext to compute the texture coordinates, optionally scaling and offsetting their values in each dimension. 〈UVMapping Definition〉 ≡ class UVMapping { public: 〈UVMapping Public Methods 650〉 private: Float su, sv, du, dv; }; 〈UVMapping Public Methods〉 ≡ 650

```rust
   UVMapping(Float su = 1, Float sv = 1, Float du = 0, Float dv = 0)
       : su(su), sv(sv), du(du), dv(dv) {}
```

```text
    Float 23
    Interaction 136
    Normal3f 94
    Point2f 92
    Point3f 92
    SurfaceInteraction 138
    TexCoord2D 650
    TextureEvalContext 650
    UVMapping 650
    Vector3f 86
```

The scale-and-shift computation to compute (s, t) coordinates is straightforward: 〈UVMapping Public Methods〉 +≡ 650

TexCoord2D Map(TextureEvalContext ctx) const { 〈Compute texture differentials for 2D (u, v) mapping〉 Point2f st(su * ctx.uv[0] + du, sv * ctx.uv[1] + dv); return TexCoord2D{st, dsdx, dsdy, dtdx, dtdy}; }

For a general 2D mapping function f (u, v) → (s, t), the screen-space derivatives of s and t are given by the chain rule:

Note that the TextureEvalContext provides the values ∂(u, v)/∂(x, y). In this case, f (u, v) = (suu + du, svv + dv) and so

We will skip past the straightforward fragment that implements Equation (10.7) to initialize dsdx, dsdy, dtdx, and dtdy.

### 10.2.2 SPHERICAL MAPPING

Another useful mapping effectively wraps a sphere around the object. Each point is projected along the vector from the sphere’s center through the point on to the sphere’s surface. Since this mapping is based on spherical coordinates, Equation (3.8) can be applied, with the angles it returns remapped to [0, 1]:

*Figure 10.9 shows the use of this mapping with an object in the Kroken scene. The SphericalMapping further stores a transformation that is applied to points before this mapping is performed; this effectively allows the mapping sphere to be arbitrarily positioned and oriented with respect to the object.*

![Figure 10.9](images/img-1014-218.png)

*Figure 10.9: Use of the SphericalMapping in the Kroken Scene. (a) Visualization of the resulting (u, v) parameterization. (b) Effect of using the SphericalMapping to apply a texture. Note that although the shape is spherical, it is modeled with a triangle mesh, to which the SphericalMapping is applied. (Scene courtesy of Angelo Ferretti.)*

```text
   Point2f 92
   SphericalMapping 652
   TexCoord2D 650
   TextureEvalContext 650
   TextureEvalContext::uv 650
   UVMapping::du 650
   UVMapping::dv 650
```

```text
   UVMapping::su 650
   UVMapping::sv 650
```

〈SphericalMapping Definition〉 ≡ class SphericalMapping { public: 〈SphericalMapping Public Methods 652〉 private: 〈SphericalMapping Private Members 652〉 }; 〈SphericalMapping Private Members〉 ≡ 652

Transform textureFromRender;

The Map() function starts by computing the texture-space point pt.

〈SphericalMapping Public Methods〉 ≡ 652

TexCoord2D Map(TextureEvalContext ctx) const { Point3f pt = textureFromRender(ctx.p); 〈Compute ∂ s/∂ p and ∂ t/∂ p for spherical mapping 652〉 〈Compute texture coordinate differentials for spherical mapping 652〉 〈Return (s, t) texture coordinates and differentials based on spherical mapping 653〉 }

For a mapping function based on a 3D point p, the generalization of Equation (10.6) is

Taking the partial derivatives of the mapping function, Equation (10.8), we can find

These quantities are computed using the texture-space position pt. 〈Compute ∂ s/∂ p and ∂ t/∂ p for spherical mapping〉 ≡ 652

Float x2y2 = Sqr(pt.x) + Sqr(pt.y);

```rust
   Float sqrtx2y2 = std::sqrt(x2y2);
   Vector3f dsdp = Vector3f(-pt.y, pt.x, 0) / (2 * Pi * x2y2);
   Vector3f dtdp = 1 / (Pi * (x2y2 + Sqr(pt.z))) *
       Vector3f(pt.x * pt.z / sqrtx2y2, pt.y * pt.z / sqrtx2y2, -sqrtx2y2);
```

The final differentials are then found using the four dot products from Equation (10.9). 〈Compute texture coordinate differentials for spherical mapping〉 ≡ 652

Vector3f dpdx = textureFromRender(ctx.dpdx); Vector3f dpdy = textureFromRender(ctx.dpdy); Float dsdx = Dot(dsdp, dpdx), dsdy = Dot(dsdp, dpdy); Float dtdx = Dot(dtdp, dpdx), dtdy = Dot(dtdp, dpdy);

Finally, previously defined spherical geometry utility functions compute the mapping of Equation (10.8).

```text
   Dot() 89
   Float 23
   Pi 1033
   Point3f 92
   SphericalMapping:: textureFromRender 652
   Sqr() 1034
   TexCoord2D 650
   TextureEvalContext 650
   TextureEvalContext::dpdx 650
   TextureEvalContext::dpdy 650
   TextureEvalContext::p 650
   Transform 120
   Vector3f 86
```

![Figure 10.10](images/img-1017-219.png)

*Figure 10.10: Use of the Cylindrical Texture Mapping. (a) Visualization of the (u, v) mapping from the CylindricalMapping. (b) Kettle with texture maps applied. (c) Scratch texture that is applied using the cylindrical texture mapping. (Scene courtesy of Angelo Ferretti.)*

〈Return (s, t) texture coordinates and differentials based on spherical mapping〉 ≡ 652

Vector3f vec = Normalize(pt - Point3f(0,0,0)); Point2f st(SphericalTheta(vec) * InvPi, SphericalPhi(vec) * Inv2Pi); return TexCoord2D{st, dsdx, dsdy, dtdx, dtdy};

### 10.2.3 CYLINDRICAL MAPPING

The cylindrical mapping effectively wraps a cylinder around the object and then uses the cylinder’s parameterization.

See Figure 10.10 for an example of its use. Note that the t texture coordinate it returns is not necessarily between 0 and 1; the mapping should either be scaled in z so that the object being textured has t ∈ [0, 1] or the texture being used should return results for coordinates outside that range that match the desired result. 〈CylindricalMapping Definition〉 ≡ class CylindricalMapping {

```rust
     public:
       〈CylindricalMapping Public Methods〉
     private:
       〈CylindricalMapping Private Members 653〉
   };
```

CylindricalMapping also supports a transformation to orient the mapping cylinder. 〈CylindricalMapping Private Members〉 ≡ 653

Transform textureFromRender;

Because the s texture coordinate is computed in the same way as it is with the spherical mapping, the cylindrical mapping’s ∂s/∂p matches the sphere’s in Equation (10.10). The partial derivative in t can easily be seen to be ∂t/∂p = (0, 0, 1).

```text
    CylindricalMapping 653
    Inv2Pi 1033
    InvPi 1033
    Normalize() 88
    Point2f 92
    Point3f 92
    SphericalPhi() 107
    SphericalTheta() 107
    TexCoord2D 650
    Transform 120
    Vector3f 86
```

Since the cylindrical mapping function and derivative computation are only slight variations on the spherical mapping’s, we will not include the implementation of its Map() function here.

### 10.2.4 PLANAR MAPPING

Another classic mapping method is planar mapping. The point is effectively projected onto a plane; a 2D parameterization of the plane then gives texture coordinates for the point. For example, a point p might be projected onto the z = 0 plane to yield texture coordinates given by s = px and t = py.

One way to define such a parameterized plane is with two nonparallel vectors vs and vt and offsets ds and dt. The texture coordinates are given by the coordinates of the point with respect to the plane’s coordinate system, which are computed by taking the dot product of the vector from the point to the origin with each vector vs and vt and then adding the corresponding offset: 〈PlanarMapping Definition〉 ≡ class PlanarMapping { public: 〈PlanarMapping Public Methods 654〉 private: 〈PlanarMapping Private Members 654〉 };

A straightforward constructor, not included here, initializes the following member variables. 〈PlanarMapping Private Members〉 ≡ 654

Transform textureFromRender; Vector3f vs, vt; Float ds, dt; 〈PlanarMapping Public Methods〉 ≡ 654

TexCoord2D Map(TextureEvalContext ctx) const { Vector3f vec(textureFromRender(ctx.p)); 〈Initialize partial derivatives of planar mapping (s, t) coordinates 654〉 Point2f st(ds + Dot(vec, vs), dt + Dot(vec, vt)); return TexCoord2D{st, dsdx, dsdy, dtdx, dtdy}; }

The planar mapping differentials can be computed directly using the partial derivatives of the mapping function, which are easily found. For example, the partial derivative of the s texture coordinate with respect to screen-space x is just ∂s/∂x = (vs · ∂p/∂x).

〈Initialize partial derivatives of planar mapping (s, t) coordinates〉 ≡ 654

```rust
   Vector3f dpdx = textureFromRender(ctx.dpdx);
   Vector3f dpdy = textureFromRender(ctx.dpdy);
   Float dsdx = Dot(vs, dpdx), dsdy = Dot(vs, dpdy);
```

```rust
   Float dtdx = Dot(vt, dpdx), dtdy = Dot(vt, dpdy);
```

```text
    Dot() 89
    Float 23
    PlanarMapping::ds 654
    PlanarMapping::dt 654
    PlanarMapping:: textureFromRender 654
    PlanarMapping::vs 654
    PlanarMapping::vt 654
    Point2f 92
    TexCoord2D 650
    TextureEvalContext 650
    TextureEvalContext::dpdx 650
    TextureEvalContext::dpdy 650
    TextureEvalContext::p 650
    TextureMapping3D 655
    Transform 120
    Vector3f 86
```

### 10.2.5 3D MAPPING

We will also define a TextureMapping3D class that defines the interface for generating 3D texture coordinates. 〈TextureMapping3D Definition〉 ≡ class TextureMapping3D : public TaggedPointer<PointTransformMapping> { public: 〈TextureMapping3D Interface 655〉 };

The Map() method it specifies returns a 3D point and partial derivative vectors in the form of a TexCoord3D structure. 〈TextureMapping3D Interface〉 ≡ 655

TexCoord3D Map(TextureEvalContext ctx) const;

TexCoord3D parallels TexCoord2D, storing both the point and its screen-space derivatives. 〈TexCoord3D Definition〉 ≡ struct TexCoord3D {

Point3f p; Vector3f dpdx, dpdy; };

The natural 3D mapping takes the rendering-space coordinate of the point and applies a linear transformation to it. This will often be a transformation that takes the point back to the primitive’s object space. Such a mapping is implemented by the PointTransformMapping class.

〈PointTransformMapping Definition〉 ≡ class PointTransformMapping { public: 〈PointTransformMapping Public Methods 655〉 private: Transform textureFromRender; };

Because it applies a linear transformation, the differential change in texture coordinates can be found by applying the same transformation to the partial derivatives of position. 〈PointTransformMapping Public Methods〉 ≡ 655

TexCoord3D Map(TextureEvalContext ctx) const { return TexCoord3D{textureFromRender(ctx.p), textureFromRender(ctx.dpdx), textureFromRender(ctx.dpdy)}; }

```text
   Point3f 92
   PointTransformMapping 655
   PointTransformMapping:: textureFromRender 655
   TaggedPointer 1073
   TexCoord2D 650
   TexCoord3D 655
   TextureEvalContext 650
   TextureEvalContext::dpdx 650
   TextureEvalContext::dpdy 650
   TextureEvalContext::p 650
   Transform 120
   Vector3f 86
```

## 10.3 TEXTURE INTERFACE AND BASIC TEXTURES

Given a variety of ways to generate 2D and 3D texture coordinates, we will now define the general interfaces for texture functions. As mentioned earlier, pbrt supports two types of Textures: scalar Float-valued, and spectral-valued. For the first, there is FloatTexture, which is defined in base/texture.rs. There are currently 14 implementations of this interface in pbrt, which leads to a lengthy list of types for the TaggedPointer template class. Therefore, we have gathered them into a fragment, 〈FloatTextures〉, that is not included here. 〈FloatTexture Definition〉 ≡ class FloatTexture : public TaggedPointer<〈FloatTextures〉> { public: 〈FloatTexture Interface 656〉 };

A FloatTexture takes a TextureEvalContext and returns a Float value.

〈FloatTexture Interface〉 ≡ 656

Float Evaluate(TextureEvalContext ctx) const;

SpectrumTexture plays an equivalent role for spectral textures. It also has so many implementations that we have elided their enumeration from the text. It, too, is defined in base/texture.rs. 〈SpectrumTexture Definition〉 ≡ class SpectrumTexture : public TaggedPointer< 〈SpectrumTextures〉> { public: 〈SpectrumTexture Interface 656〉 };

For the reasons that were discussed in Section 4.5.4, the SpectrumTexture evaluation routine does not return a full spectral distribution (e.g., an

implementation of the Spectrum interface from Section 4.5.1). Rather, it takes a set of wavelengths of interest and returns the texture’s value at just those wavelengths. 〈SpectrumTexture Interface〉 ≡ 656

SampledSpectrum Evaluate(TextureEvalContext ctx, SampledWavelengths lambda) const;

### 10.3.1 CONSTANT TEXTURE

The constant textures return the same value no matter where they are evaluated. Because they represent constant functions, they can be accurately reconstructed with any sampling rate and therefore need no antialiasing. Although these two textures are trivial, they are actually quite useful. By providing these classes, all parameters to all Materials can be represented as Textures, whether they are spatially varying or not. For example, a red diffuse object will have a SpectrumConstantTexture that always returns red as the diffuse color of the material. This way, the material system always evaluates a texture to get the surface properties at a point, avoiding the need for separate textured and nontextured versions of materials. Such an approach would grow increasingly unwieldy as the number of material parameters increased. FloatConstantTexture, like all the following texture implementations, is defined in the files texture.rs and texture.rs.

〈FloatConstantTexture Definition〉 ≡ class FloatConstantTexture { public: FloatConstantTexture(Float value) : value(value) {} Float Evaluate(TextureEvalContext ctx) const { return value; } private: Float value; };

```text
    Float 23
    FloatConstantTexture 656
```

```text
    FloatConstantTexture::value 656
    Material 674
    SampledSpectrum 171
    SampledWavelengths 173
    SpectrumConstantTexture 657
    TaggedPointer 1073
    TextureEvalContext 650
```

The spectrum constant texture, SpectrumConstantTexture, is similarly simple. Here is its Evaluate() method; the rest of its structure parallels FloatConstantTexture and so is not included here.

〈SpectrumConstantTexture Public Methods〉 ≡ SampledSpectrum Evaluate(TextureEvalContext ctx, SampledWavelengths lambda) const { return value.Sample(lambda); }

### 10.3.2 SCALE TEXTURE

We have defined the texture interface in a way that makes it easy to use the output of one texture function when computing another. This is useful since it lets us define generic texture operations using any of the other texture types. The FloatScaledTexture takes two Float-valued textures and returns the product of their values. 〈FloatScaledTexture Definition〉 ≡ class FloatScaledTexture { public: 〈FloatScaledTexture Public Methods 657〉 private: FloatTexture tex, scale; };

FloatScaledTexture ignores antialiasing, leaving it to its two subtextures to antialias themselves but not making an effort to antialias their product. While it is easy to show that the product of two band-limited functions is also band limited, the maximum frequency present in the product may be

greater than that of either of the two terms individually. Thus, even if the scale and value textures are perfectly antialiased, the result might not be. Fortunately, the most common use of this texture is to scale another texture by a constant, in which case the other texture’s antialiasing is sufficient. One thing to note in the implementation of its Evaluate() method is that it skips evaluating the tex texture if the scale texture returns 0. It is worthwhile to avoid incurring the cost of this computation if it is unnecessary. 〈FloatScaledTexture Public Methods〉 ≡ 657

Float Evaluate(TextureEvalContext ctx) const { Float sc = scale.Evaluate(ctx); if (sc == 0) return 0; return tex.Evaluate(ctx) * sc; }

SpectrumScaledTexture is the straightforward variant and is therefore not included here. An example of its use is shown in Figure 10.11.

### 10.3.3 MIX TEXTURES

The mix textures are more general variations of the scale textures. They blend between two textures of the same type based on a scalar blending factor. Note that a constant texture could be used for the blending factor to achieve a uniform blend, or a more complex Texture could be used to blend in a spatially nonuniform way. Figure 10.12 shows the use of the SpectrumMixTexture where an image is used to blend between two constant RGB colors.

```text
    Float 23
    FloatConstantTexture 656
    FloatScaledTexture 657
    FloatScaledTexture::scale 657
    FloatScaledTexture::tex 657
    FloatTexture 656
    FloatTexture::Evaluate() 656
    SampledSpectrum 171
    SampledWavelengths 173
    Spectrum::Sample() 175
    SpectrumMixTexture 659
```

TextureEvalContext 650

![Figure 10.11](images/img-1026-220.png)

*Figure 10.11: Use of the SpectrumScaledTexture in the Watercolor Scene. The product of (a) a texture of paint strokes and (b) a mask representing splotches gives (c) colorful splotches. (d) When applied to the surface of a table, a convincing paint spill results. (Scene courtesy of Angelo Ferretti.)*

![Figure 10.12](images/img-1027-221.png)

*Figure 10.12: Use of the Mix Texture in the Kroken Scene. (a) The SpectrumMixTexture is used to find the color at each point on the bottom two cups. (b) Two fixed RGB colors are modulated using this image texture. (Scene courtesy of Angelo Ferretti.)*

```text
   SpectrumMixTexture 659
   SpectrumScaledTexture 657
```

〈FloatMixTexture Definition〉 ≡ class FloatMixTexture { public: 〈FloatMixTexture Public Methods 659〉 private: FloatTexture tex1, tex2; FloatTexture amount; };

To evaluate the mixture, the three textures are evaluated and the floating-point value is used to linearly interpolate between the two. When the blend amount amt is zero, the first texture’s value is returned, and when it is one the second one’s value is returned. The Evaluate() method here makes sure not to evaluate textures unnecessarily if the blending amount implies

that only one of their values is necessary. (Section 15.1.1 has further discussion about why the logic for that is written just as it is here, rather than with, for example, cascaded if tests that each directly return the appropriate value.) We will generally assume that amt will be between zero and one, but this behavior is not enforced, so extrapolation is possible as well. As with the scale textures, antialiasing is ignored, so the introduction of aliasing here is a possibility. 〈FloatMixTexture Public Methods〉 ≡ 659

Float Evaluate(TextureEvalContext ctx) const { Float amt = amount.Evaluate(ctx); Float t1 = 0, t2 = 0; if (amt != 1) t1 = tex1.Evaluate(ctx); if (amt != 0) t2 = tex2.Evaluate(ctx); return (1 - amt) * t1 + amt * t2; }

We will not include the implementation of SpectrumMixTexture here, as it parallels that of FloatMixTexture.

It can also be useful to blend between two textures based on the surface’s orientation. The FloatDirectionMixTexture and SpectrumDirectionMixTexture use the dot product of the surface normal with a specified direction to compute such a weight. As they are very similar, we will only discuss SpectrumDirectionMixTexture here.

〈SpectrumDirectionMixTexture Definition〉 ≡ class SpectrumDirectionMixTexture { public: 〈SpectrumDirectionMixTexture Public Methods 660〉 private: 〈SpectrumDirectionMixTexture Private Members 659〉 }; 〈SpectrumDirectionMixTexture Private Members〉 ≡ 659

SpectrumTexture tex1, tex2; Vector3f dir;

```text
   Float 23
   FloatMixTexture::amount 659
```

```text
   FloatMixTexture::tex1 659
   FloatMixTexture::tex2 659
   FloatTexture 656
   FloatTexture::Evaluate() 656
   SpectrumTexture 656
   TextureEvalContext 650
   Vector3f 86
```

![Figure 10.13](images/img-1029-222.png)

*Figure 10.13: Use of the SpectrumDirectionMixTexture in the Kroken Scene. The texture is used to select between one image texture that uses a planar projection to specify the cover image of the magazine and another that uses a different planar projection for its spine. (Scene courtesy of Angelo Ferretti.)*

If the normal is coincident with the specified direction, tex1 is returned; if it is perpendicular, then tex2 is. Otherwise, the two textures are blended. Figure 10.13 shows an example of the use of this texture. 〈SpectrumDirectionMixTexture Public Methods〉 ≡ 659

SampledSpectrum Evaluate(TextureEvalContext ctx, SampledWavelengths lambda) const { Float amt = AbsDot(ctx.n, dir);

```rust
        SampledSpectrum t1, t2;
        if (amt != 0) t1 = tex1.Evaluate(ctx, lambda);
        if (amt != 1) t2 = tex2.Evaluate(ctx, lambda);
        return amt * t1 + (1 - amt) * t2;
   }
```

## 10.4 IMAGE TEXTURE

Image textures store 2D arrays of point-sampled values of a texture function. They use these samples to reconstruct a continuous image function that can be evaluated at an arbitrary (s, t) position.3 These sample values are often called texels, since they are similar to pixels in an image but are used in the context of a texture. Image textures are the most widely used type of texture in computer graphics; digital photographs, scanned artwork, images created with image-editing programs, and images generated by renderers are all extremely useful sources of data for this particular texture representation (Figure 10.14).

```text
   AbsDot() 90
   Float 23
   SampledSpectrum 171
   SampledWavelengths 173
   SpectrumDirectionMixTexture 659
   SpectrumDirectionMixTexture:: dir 659
   SpectrumDirectionMixTexture:: tex1 659
   SpectrumDirectionMixTexture:: tex2 659
   SpectrumTexture::Evaluate() 656
   TextureEvalContext 650
   TextureEvalContext::n 650
```

![Figure 10.14](images/img-1031-223.png)

*Figure 10.14: An Example of Image Textures. Image textures are used throughout the Watercolor scene to represent spatially varying surface appearance properties. (a) Scene rendered with image textures. (b) Each image texture has been replaced with its average value. Note how much visual richness is lost. (Scene courtesy of Angelo Ferretti.)*

As with most of the other types of texture, pbrt provides both Float and spectral variants. Both implementations inherit from ImageTextureBase, which provides some common functionality. 〈ImageTextureBase Definition〉 ≡ class ImageTextureBase { public: 〈ImageTextureBase Public Methods 662〉 protected: 〈ImageTextureBase Protected Members 662〉 private: 〈ImageTextureBase Private Members 663〉

```rust
   };
```

In the following, we will present the implementation of SpectrumImageTexture; FloatImage Texture is analogous and does not add anything new. 〈SpectrumImageTexture Definition〉 ≡ class SpectrumImageTexture : public ImageTextureBase { public: 〈SpectrumImageTexture Public Methods 662〉 private: 〈SpectrumImageTexture Private Members 662〉 };

```text
    ImageTextureBase 661
```

### 10.4.1 TEXTURE MEMORY MANAGEMENT

The caller of SpectrumImageTexture’s constructor provides a texture mapping function, the filename of an image, various parameters that control the filtering of the image map, how boundary conditions are managed, and how colors are converted to spectral samples. All the necessary initialization is handled by ImageTextureBase. 〈SpectrumImageTexture Public Methods〉 ≡ 661

SpectrumImageTexture(TextureMapping2D mapping, std::string filename, MIPMapFilterOptions filterOptions, WrapMode wrapMode, Float scale, bool invert, ColorEncoding encoding, SpectrumType spectrumType, Allocator alloc) : ImageTextureBase(mapping, filename, filterOptions, wrapMode, scale, invert, encoding, alloc), spectrumType(spectrumType) {}

As was discussed in Section 4.6.6, RGB colors are transformed into spectra differently depending on whether or not they represent reflectances. The spectrumType records what type of RGB a texture represents.

〈SpectrumImageTexture Private Members〉 ≡ 661

SpectrumType spectrumType;

The contents of the image file are used to create an instance of the MIPMap class that stores the texels in memory and handles the details of reconstruction and filtering to reduce aliasing. 〈ImageTextureBase Public Methods〉 ≡ 661

ImageTextureBase(TextureMapping2D mapping, std::string filename, MIPMapFilterOptions filterOptions, WrapMode wrapMode, Float scale, bool invert, ColorEncoding encoding, Allocator alloc) : mapping(mapping), filename(filename), scale(scale), invert(invert) { 〈Get MIPMap from texture cache if present 663〉 〈Create MIPMap for filename and add to texture cache 663〉 }

A floating-point scale can be specified with each texture; it is applied to the values returned by the Evaluate() method. Further, a true value for the invert parameter causes the texture value to be subtracted from 1 before it is returned. While the same functionality can be achieved with scale and mix textures, it is easy to also provide that functionality directly in the texture here. Doing so can lead to more efficient texture evaluation on GPUs, as is discussed further in Section 15.3.9. 〈ImageTextureBase Protected Members〉 ≡ 661

TextureMapping2D mapping; std::string filename; Float scale; bool invert; MIPMap *mipmap;

Each MIP map may require a meaningful amount of memory, and a complex scene may have thousands of image textures. Because an on-disk image may be reused for multiple textures in a scene, pbrt maintains a table of MIP maps that have been loaded so far so that they are only loaded into memory once even if they are used in more than one image texture.

```text
   Allocator 40
   ColorEncoding 1094
   Float 23
   ImageTextureBase 661
   MIPMap 665
   MIPMapFilterOptions 667
   SpectrumImageTexture 661
   SpectrumType 1125
```

```text
   TextureMapping2D 649
   WrapMode 1082
```

pbrt loads textures in parallel after the scene description has been parsed; doing so reduces startup time before rendering begins. Therefore, a mutex is used here to ensure that only one thread accesses the texture cache at a time. Note that if the MIPMap is not found in the cache, the lock is released before it is read so that other threads can access the cache in the meantime. 〈Get MIPMap from texture cache if present〉 ≡ 662

TexInfo texInfo(filename, filterOptions, wrapMode, encoding); std::unique_lock<std::mutex> lock(textureCacheMutex); if (auto iter = textureCache.find(texInfo); iter != textureCache.end()) { mipmap = iter->second; return; } lock.unlock();

The texture cache itself is managed with a std::map.

〈ImageTextureBase Private Members〉 ≡ 661

static std::mutex textureCacheMutex; static std::map<TexInfo, MIPMap *> textureCache;

TexInfo is a simple structure that acts as a key for the texture cache std::map. It holds all the specifics that must match for a MIPMap to be reused in another image texture. 〈TexInfo Definition〉 ≡ struct TexInfo { 〈TexInfo Public Methods 663〉 std::string filename; MIPMapFilterOptions filterOptions; WrapMode wrapMode; ColorEncoding encoding; };

The TexInfo constructor, not included here, sets its member variables with provided values. Its only other method is a comparison operator, which is

required by std::map. 〈TexInfo Public Methods〉 ≡ 663

bool operator<(const TexInfo &t) const { return std::tie(filename, filterOptions, encoding, wrapMode) < std::tie(t.filename, t.filterOptions, t.encoding, t.wrapMode); }

If the texture has not yet been loaded, a call to CreateFromFile() yields a MIPMap for it. If the file is not found or there is an error reading it, pbrt exits with an error message, so a nullptr return value does not need to be handled here. 〈Create MIPMap for filename and add to texture cache〉 ≡ 662

mipmap = MIPMap::CreateFromFile(filename, filterOptions, wrapMode, encoding, alloc); lock.lock(); textureCache[texInfo] = mipmap;

```text
    ColorEncoding 1094
    ImageTextureBase::mipmap 662
    ImageTextureBase:: textureCache 663
    ImageTextureBase:: textureCacheMutex 663
    MIPMap 665
    MIPMapFilterOptions 667
    SpectrumImageTexture 661
    TexInfo 663
    TexInfo::encoding 663
    TexInfo::filename 663
    TexInfo::filterOptions 663
    TexInfo::wrapMode 663
    WrapMode 1082
```

### 10.4.2 IMAGE TEXTURE EVALUATION

Before describing the MIPMap implementation, we will discuss the SpectrumImageTexture Evaluate() method.

〈SpectrumImageTexture Method Definitions〉 ≡ SampledSpectrum SpectrumImageTexture::Evaluate( TextureEvalContext ctx, SampledWavelengths lambda) const {

〈Apply texture mapping and flip t coordinate for image texture lookup 664〉 〈Lookup filtered RGB value in MIPMap 664〉 〈Return SampledSpectrum for RGB image texture value 664〉 }

It is easy to compute the (s, t) texture coordinates and their derivatives for filtering with the TextureMapping2D’s Map() method. However, the t coordinate must be flipped, because pbrt’s Image class (and in turn, MIPMap, which is based on it) defines (0, 0) to be the upper left corner of the image, while image textures have (0, 0) at the lower left. (These are the typical conventions for indexing these entities in computer graphics.) 〈Apply texture mapping and flip t coordinate for image texture lookup〉 ≡ 664

TexCoord2D c = mapping.Map(ctx); c.st[1] = 1 - c.st[1];

The MIPMap’s Filter() method provides the filtered value of the image texture over the specified region; any specified scale or inversion is easily applied to the value it returns. A call to ClampZero() here ensures that no negative values are returned after inversion. 〈Lookup filtered RGB value in MIPMap〉 ≡ 664

RGB rgb = scale * mipmap->Filter<RGB>(c.st, {c.dsdx, c.dtdx}, {c.dsdy, c.dtdy}); rgb = ClampZero(invert ? (RGB(1, 1, 1) - rgb) : rgb);

As discussed in Section 4.6.2, an RGB color space is necessary in order to interpret the meaning of an RGB color value. Normally, the code that reads image file formats from disk returns an RGBCcolorSpace with the read image. Most RGB image formats default to sRGB, and some allow specifying an alternative color space. (For example, OpenEXR allows specifying the primaries of an arbitrary RGB color space in the image file’s metadata.) A color space and the value of spectrumType make it possible to create the appropriate type RGB spectrum, and in turn, its Spectrum::Sample() can be called to get the SampledSpectrum that will be returned.

If the MIPMap has no associated color space, the image is assumed to have the same value in all channels and a constant value is returned for all the spectrum samples. This assumption is verified by a DCHECK() call in non-optimized builds. 〈Return SampledSpectrum for RGB image texture value〉 ≡ 664

if (const RGBColorSpace *cs = mipmap->GetRGBColorSpace(); cs) { if (spectrumType == SpectrumType::Unbounded) return RGBUnboundedSpectrum(*cs, rgb).Sample(lambda); else if (spectrumType == SpectrumType::Albedo) return RGBAlbedoSpectrum(*cs, Clamp(rgb, 0, 1)).Sample(lambda);

```rust
         else
            return RGBIlluminantSpectrum(*cs, rgb).Sample(lambda);
   }
   DCHECK(rgb[0] == rgb[1] && rgb[1] == rgb[2]);
   return SampledSpectrum(rgb[0]);
```

```text
    Clamp() 1033
    DCHECK() 1066
    Image 1079
    ImageTextureBase::invert 662
    ImageTextureBase::mapping 662
    ImageTextureBase::mipmap 662
    ImageTextureBase::scale 662
    MIPMap 665
    MIPMap::Filter() 668
    MIPMap::GetRGBColorSpace() 667
    RGB 182
    RGBAlbedoSpectrum 197
    RGBColorSpace 183
    RGBIlluminantSpectrum 199
    RGBIlluminantSpectrum::Sample() 200
    RGBUnboundedSpectrum 198
    RGBUnboundedSpectrum::Sample() 199
    SampledSpectrum 171
    SampledWavelengths 173
    Spectrum::Sample() 175
    SpectrumImageTexture:: spectrumType 662
    SpectrumType::Albedo 1125
    SpectrumType::Unbounded 1125
    TexCoord2D 650
    TexCoord2D::dsdx 650
    TexCoord2D::dsdy 650
    TexCoord2D::dtdx 650
    TexCoord2D::dtdy 650
    TexCoord2D::st 650
    TextureEvalContext 650
    TextureMapping2D 649
    TextureMapping2D::Map() 650
```

### 10.4.3 MIP MAPS

As always, if the image texture function has higher frequency detail than can be represented by the texture sampling rate, aliasing will be present in the final image. Any frequencies higher than the Nyquist limit must be removed by prefiltering before the function is evaluated. Figure 10.15

shows the basic problem we face: an image texture has texels that are samples of some image function at a fixed frequency. The filter region for the lookup is given by its (s, t) center point and offsets to the estimated texture coordinate locations for the adjacent image samples. Because these offsets are estimates of the texture sampling rate, we must remove any frequencies higher than twice the distance to the adjacent samples in order to satisfy the Nyquist criterion.

![Figure 10.15](images/img-1039-224.png)

*Figure 10.15: Given a point at which to perform an image map lookup (denoted by the solid point in the center) and estimates of the texture-space sampling rate (denoted by adjacent solid points), it may be necessary to filter the contributions of a large number of texels in the image map (denoted by open points).*

The texture sampling and reconstruction process has a few key differences from the image sampling process discussed in Chapter 8. These differences make it possible to address the antialiasing problem with more effective and less computationally expensive techniques. For example, here it is inexpensive to get the value of a sample—only an array lookup is necessary (as opposed to having to trace a number of rays to compute radiance). Further, because the texture image function is fully defined by the set of samples and there is no mystery about what its highest frequency could be, there is no uncertainty related to the function’s behavior between samples. These differences make it possible to remove detail from the texture before sampling, thus eliminating aliasing. However, the texture sampling rate will typically change from pixel to pixel. The sampling rate is determined by scene geometry and its orientation, the texture coordinate mapping function, and the camera

projection and image sampling rate. Because the texture sampling rate is not fixed, texture filtering algorithms need to be able to filter over arbitrary regions of texture samples efficiently. The MIPMap class implements a number of methods for texture filtering with spatially varying filter widths. It can be found in the files util/mipmap.rs and util/mipmap.rs. The filtering algorithms it offers range from simple point sampling to bilinear interpolation and trilinear interpolation, which is fast and easy to implement and was widely used for texture filtering in early graphics hardware, to elliptically weighted averaging, which is more complex but returns extremely high-quality results. Figure 10.16 compares the result of texture filtering using trilinear interpolation and the EWA algorithm. 〈MIPMap Definition〉 ≡ class MIPMap { public: 〈MIPMap Public Methods 667〉 private: 〈MIPMap Private Methods〉 〈MIPMap Private Members 667〉 };

```text
   MIPMap 665
```

![Figure 10.16](images/img-1041-225.png)

*Figure 10.16: Filtering the image map properly substantially improves the image. Trilinear interpolation is used for the sphere on the left and the EWA algorithm is used for the sphere on the right. Both of these approaches give a much better result than the unfiltered image map on the sphere on the left in Figure 10.1. Trilinear interpolation is not as effective as EWA at handling strongly anisotropic filter footprints, which is why the lines on the left sphere are blurrier. In regions with highly anisotropic filter footprints such as the pole of the sphere and toward the edges, EWA resolves much more detail.*

If an RGB image is provided to the MIPMap constructor, its channels should be stored in R, G, B order in memory; for efficiency, the following code assumes that this is the case. All the code that currently uses MIPMaps in pbrt ensures that this is so.

〈MIPMap Method Definitions〉 ≡ MIPMap::MIPMap( Image image, const RGBColorSpace *colorSpace, WrapMode wrapMode, Allocator alloc, const MIPMapFilterOptions &options) : colorSpace(colorSpace), wrapMode(wrapMode), options(options) { pyramid = Image::GeneratePyramid(std::move(image), wrapMode, alloc); }

To limit the potential number of texels that need to be accessed, these filtering methods use an image pyramid of increasingly lower resolution prefiltered versions of the original image to accelerate their operation.4 The original image texels are at the bottom level of the pyramid, and the image at each level is half the resolution of the previous level, up to the top level, which has a single texel representing the average of all the texels in the original image. This collection of images needs at most 1/3 more memory than storing the most detailed level alone and can be used to quickly find filtered values over large regions of the original image. The basic idea behind the pyramid is that if a large area of texels needs to be filtered, a reasonable approximation is to use a higher level of the pyramid and do the filtering over the same area there, accessing many fewer texels. The MIPMap’s image pyramid is represented by a vector of Images. See Section B.5 for the implementation of Image and Section B.5.5 for its GeneratePyramid() method, which generates image pyramids.

```text
   Allocator 40
   Image 1079
   Image::GeneratePyramid() 1090
   MIPMap 665
   MIPMapFilterOptions 667
   RGBColorSpace 183
   WrapMode 1082
```

〈MIPMap Private Members〉 ≡ 665

pstd::vector<Image> pyramid; const RGBColorSpace *colorSpace; WrapMode wrapMode; MIPMapFilterOptions options;

The choice of filtering algorithm and a parameter used by the EWA method are represented by MIPMapFilterOptions. 〈MIPMapFilterOptions Definition〉 ≡ struct MIPMapFilterOptions { FilterFunction filter = FilterFunction::EWA; Float maxAnisotropy = 8.f; };

〈FilterFunction Definition〉 ≡ enum class FilterFunction { Point, Bilinear, Trilinear, EWA };

A few simple utility methods return information about the image pyramid and the MIPMap’s color space. 〈MIPMap Public Methods〉 ≡ 665

Point2i LevelResolution(int level) const { return pyramid[level].Resolution(); } int Levels() const { return int(pyramid.size()); } const RGBColorSpace *GetRGBColorSpace() const { return colorSpace; } const Image &GetLevel(int level) const { return pyramid[level]; }

Given the image pyramid, we will define some utility MIPMap methods that retrieve the texel value at a specified pyramid level and discrete integer pixel coordinates. For the RGB variant, there is an implicit assumption that the image channels are laid out in R, G, B (and maybe A) order. 〈MIPMap Method Definitions〉 +≡ template <> RGB MIPMap::Texel(int level, Point2i st) const { if (int nc = pyramid[level].NChannels(); nc == 3 || nc == 4) return RGB(pyramid[level].GetChannel(st, 0, wrapMode), pyramid[level].GetChannel(st, 1, wrapMode), pyramid[level].GetChannel(st, 2, wrapMode)); else { Float v = pyramid[level].GetChannel(st, 0, wrapMode); return RGB(v, v, v); } }

The Float specialization of Texel(), not included here, is analogous.

```text
    FilterFunction 667
    FilterFunction::EWA 667
    Float 23
    Image 1079
    Image::GetChannel() 1081
    Image::NChannels() 1080
    Image::Resolution() 1080
    MIPMap 665
    MIPMap::colorSpace 667
    MIPMap::pyramid 667
    MIPMapFilterOptions 667
    Point2i 92
    RGB 182
    RGBColorSpace 183
    WrapMode 1082
```

### 10.4.4 IMAGE MAP FILTERING

The MIPMap Filter() method returns a filtered image function value at the provided (s, t) coordinates. It takes two derivatives that give the change in (s, t) with respect to image pixel samples. 〈MIPMap Method Definitions〉 +≡ template <typename T> T MIPMap::Filter(Point2f st, Vector2f dst0, Vector2f dst1) const { if (options.filter != FilterFunction::EWA) { 〈Handle non-EWA MIP Map filter 668〉 } 〈Compute EWA ellipse axes 670〉 〈Clamp ellipse vector ratio if too large 671〉 〈Choose level of detail for EWA lookup and perform EWA filtering 671〉

}

The EWA filtering technique to be described shortly uses both derivatives of (s, t) to compute an anisotropic filter—one that filters by different amounts in the different dimensions. The other three use an isotropic filter that filters both equally. The isotropic filters are more computationally efficient than the anisotropic filter, though they do not give results that are as good. For

them, only a single value is needed to specify the width of the filter. The width here is conservatively chosen to avoid aliasing in both the s and t directions, though this choice means that textures viewed at an oblique angle will appear blurry, since the required sampling rate in one direction will be very different from the sampling rate along the other in this case. 〈Handle non-EWA MIP Map filter〉 ≡ 668

Float width = 2 * std::max({std::abs(dst0[0]), std::abs(dst0[1]), std::abs(dst1[0]), std::abs(dst1[1])}); 〈Compute MIP Map level for width and handle very wide filter 669〉 if (options.filter == FilterFunction::Point) { 〈Return point-sampled value at selected MIP level 669〉 } else if (options.filter == FilterFunction::Bilinear) { 〈Return bilinear-filtered value at selected MIP level 669〉 } else { 〈Return trilinear-filtered value at selected MIP level 670〉 }

Because filtering over many texels for wide filter widths would be inefficient, this method chooses a MIP map level from the pyramid such that the filter region at that level would cover four texels at that level. Figure 10.17 illustrates this idea.

![Figure 10.17](images/img-1045-226.png)

*Figure 10.17: Choosing a MIP Map Level for the Triangle Filter. The MIPMap chooses a level such that the filter covers four texels.*

```text
   Filter 515
   FilterFunction::Bilinear 667
   FilterFunction::EWA 667
   FilterFunction::Point 667
   Float 23
```

```text
   MIPMap::options 667
   MIPMapFilterOptions::filter 667
   Point2f 92
   Vector2f 86
```

Since the resolutions of the levels of the pyramid are all powers of two, the resolution of level l is 2nLevels−1−l. Therefore, to find the level with a texel spacing width w requires solving for l. In general, this will be a floating-point value between two MIP map levels. Values of l greater than the number of pyramid levels correspond to a filter width wider than the image, in which case the single pixel at the top level is returned.

〈Compute MIP Map level for width and handle very wide filter〉 ≡ 668

int nLevels = Levels(); Float level = nLevels - 1 + Log2(std::max<Float>(width, 1e-8)); if (level >= Levels() - 1) return Texel<T>(Levels() - 1, Point2i(0, 0)); int iLevel = std::max(0, int(pstd::floor(level)));

For a point-sampled texture lookup, it is only necessary to convert the continuous texture coordinates over [0, 1] to discrete coordinates over the image resolution and to retrieve the appropriate texel value via the MIPMap’s Texel() method. 〈Return point-sampled value at selected MIP level〉 ≡ 668

Point2i resolution = LevelResolution(iLevel); Point2i sti(pstd::round(st[0] * resolution[0] - 0.5f), pstd::round(st[1] * resolution[1] - 0.5f)); return Texel<T>(iLevel, sti);

Bilinear filtering, which is equivalent to filtering using a triangle filter, is easily implemented via a call to Bilerp(). 〈Return bilinear-filtered value at selected MIP level〉 ≡ 668

return Bilerp<T>(iLevel, st);

Bilinear interpolation is provided in a separate method so that it can also be used for trilinear filtering.

〈MIPMap Method Definitions〉 +≡ template <> RGB MIPMap::Bilerp(int level, Point2f st) const { if (int nc = pyramid[level].NChannels(); nc == 3 || nc == 4) return RGB(pyramid[level].BilerpChannel(st, 0, wrapMode), pyramid[level].BilerpChannel(st, 1, wrapMode), pyramid[level].BilerpChannel(st, 2, wrapMode)); else { Float v = pyramid[level].BilerpChannel(st, 0, wrapMode); return RGB(v, v, v); } }

As shown by Figure 10.17, applying a triangle filter to the four texels around the sample point will either filter over too small a region or too large a region (except for very carefully selected filter widths). Therefore, the Trilinear filtering option applies the triangle filter at both of these levels and blends between them according to how close level is to each of them. This helps hide the transitions from one MIP map level to the next at nearby pixels in the final image. While applying a triangle filter to four texels at two levels in this manner does not generally give exactly the same result as applying a triangle filter to the original pixels, the difference is not too bad in practice, and the efficiency of this approach is worth this penalty. In any case, the following elliptically weighted average filtering approach should be used when texture quality is important.

```text
   Float 23
   Image::BilerpChannel() 1082
   Image::NChannels() 1080
   Log2() 1035
   MIPMap::Bilerp() 669
   MIPMap::LevelResolution() 667
```

```text
   MIPMap::Levels() 667
   MIPMap::pyramid 667
   MIPMap::Texel() 667
   Point2f 92
   Point2i 92
   RGB 182
```

![Figure 10.18](images/img-1048-227.png)

*Figure 10.18: The EWA filter applies a Gaussian filter to the texels in an elliptical area around the evaluation point. The extent of the ellipse is such that its edge passes through the positions of the adjacent texture samples as estimated by the texture coordinate partial derivatives.*

〈Return trilinear-filtered value at selected MIP level〉 ≡ 668

if (iLevel == 0) return Bilerp<T>(0, st);

```rust
   else
          return Lerp(level - iLevel, Bilerp<T>(iLevel, st),
                     Bilerp<T>(iLevel + 1, st));
```

The elliptically weighted average (EWA) algorithm fits an ellipse to the two differential vectors in texture space and then filters the texture with a Gaussian filter function (Figure 10.18). It is widely regarded as one of the best texture filtering algorithms in graphics and has been carefully derived from the basic principles of sampling theory. Unlike the triangle filter, it can filter over arbitrarily oriented regions of the texture, with different filter extents in different directions. The quality of its results is improved by it being an anisotropic filter, since it can adapt to different sampling rates along the two image axes. We will not show the full derivation of this filter here, although we do note that it is distinguished by being a unified resampling filter: it simultaneously computes the result of a Gaussian filtered texture function convolved with a Gaussian reconstruction filter in image space. This is in contrast to many other texture filtering methods that ignore the effect of the image-space filter or equivalently assume that it is a box. Even if a Gaussian is not being used for filtering the samples for the image being rendered, taking some account of the spatial variation of the image filter improves the results, assuming that the filter being used is somewhat similar in shape to the Gaussian, as the Mitchell and windowed sinc filters are. The screen-space partial derivatives of the texture coordinates define the ellipse. The lookup method starts out by determining which of the two axes is the longer of the two, swapping them if needed so that dst0 is the longer vector. The length of the shorter vector will be used to select a MIP map level. 〈Compute EWA ellipse axes〉 ≡ 668

if (LengthSquared(dst0) < LengthSquared(dst1)) pstd::swap(dst0, dst1); Float longerVecLength = Length(dst0), shorterVecLength = Length(dst1);

```text
   Float 23
   Length() 88
   LengthSquared() 87
```

```text
   Lerp() 72
   MIPMap::Bilerp() 669
```

Next the ratio of the length of the longer vector to the length of the shorter one is considered. A large ratio indicates a very long and skinny ellipse. Because this method filters texels from a MIP map level chosen based on the length of the shorter differential vector, a large ratio means that a large number of texels need to be filtered. To avoid this expense (and to ensure that any EWA lookup takes a bounded amount of time), the length of the shorter vector may be increased to limit this ratio. The result may be an increase in blurring, although this effect usually is not noticeable in practice. 〈Clamp ellipse vector ratio if too large〉 ≡ 668

if (shorterVecLength * options.maxAnisotropy < longerVecLength && shorterVecLength > 0) { Float scale = longerVecLength / (shorterVecLength * options.maxAnisotropy); dst1 *= scale; shorterVecLength *= scale; } if (shorterVecLength == 0) return Bilerp<T>(0, st);

Like the triangle filter, the EWA filter uses the image pyramid to reduce the number of texels to be filtered for a particular texture lookup, choosing a MIP map level based on the length of the shorter vector. Given the limited ratio from the clamping above, the total number of texels used is thus bounded. Given the length of the shorter vector, the computation to find the appropriate pyramid level is the same as was used for the triangle filter. Similarly, the implementation here blends between the filtered results at the two levels around the computed level of detail, again to reduce artifacts from transitions from one level to another. 〈Choose level of detail for EWA lookup and perform EWA filtering〉 ≡ 668

Float lod = std::max<Float>(0, Levels() - 1 + Log2(shorterVecLength)); int ilod = pstd::floor(lod); return Lerp(lod - ilod, EWA<T>(ilod, st, dst0, dst1), EWA<T>(ilod + 1, st, dst0, dst1));

The MIPMap::EWA() method actually applies the filter at a particular level.

〈MIPMap Method Definitions〉 +≡ template <typename T> T MIPMap::EWA(int level, Point2f st, Vector2f dst0, Vector2f dst1) const { if (level >= Levels()) return Texel<T>(Levels() - 1, {0, 0}); 〈Convert EWA coordinates to appropriate scale for level 672〉 〈Find ellipse coefficients that bound EWA filter region 672〉 〈Compute the ellipse’s (s, t) bounding box in texture space 672〉 〈Scan over ellipse bound and evaluate quadratic equation to filter image 673〉 }

This method first converts from texture coordinates in [0, 1] to coordinates and differentials in terms of the resolution of the chosen MIP map level. It also subtracts 0.5 from the continuous position coordinate to align the sample point with the discrete texel coordinates, as was done in MIPMap::Bilerp().

```text
   Float 23
   Lerp() 72
   Log2() 1035
   MIPMap::Bilerp() 669
   MIPMap::EWA() 671
   MIPMap::Levels() 667
   MIPMap::options 667
   MIPMap::Texel() 667
   MIPMapFilterOptions:: maxAnisotropy 667
   Point2f 92
   Vector2f 86
```

〈Convert EWA coordinates to appropriate scale for level〉 ≡ 671

Point2i levelRes = LevelResolution(level); st[0] = st[0] * levelRes[0] - 0.5f; st[1] = st[1] * levelRes[1] - 0.5f; dst0[0] *= levelRes[0]; dst0[1] *= levelRes[1]; dst1[0] *= levelRes[0]; dst1[1] *= levelRes[1];

It next computes the coefficients of the implicit equation for the ellipse centered at the origin that is defined by the vectors (ds0,dt0) and (ds1,dt1). Placing the ellipse at the origin rather than at (s, t) simplifies the implicit equation and the computation of its coefficients and can be easily corrected for when the equation is evaluated later. The general form of the implicit equation for all points (s, t) inside such an ellipse is e(s, t) = As2 + Bst + Ct2 < F, although it is more computationally efficient to divide through by F and express this as

We will not derive the equations that give the values of the coefficients, although the interested reader can easily verify their correctness.5 〈Find ellipse coefficients that bound EWA filter region〉 ≡ 671

Float A = Sqr(dst0[1]) + Sqr(dst1[1]) + 1; Float B = -2 * (dst0[0] * dst0[1] + dst1[0] * dst1[1]); Float C = Sqr(dst0[0]) + Sqr(dst1[0]) + 1; Float invF = 1 / (A * C - Sqr(B) * 0.25f); A *= invF; B *= invF; C *= invF;

The next step is to find the axis-aligned bounding box in discrete integer texel coordinates of the texels that are potentially inside the ellipse. The EWA algorithm loops over all of these candidate texels, filtering the contributions of those that are in fact inside the ellipse. The bounding box is found by determining the minimum and maximum values that the ellipse takes in the s and t directions. These extrema can be calculated by finding the partial derivatives ∂e/∂s and ∂e/∂t, finding their solutions for s = 0 and t = 0, and adding the offset to the ellipse center. For brevity, we will not include the derivation for these expressions here. 〈Compute the ellipse’s (s, t) bounding box in texture space〉 ≡ 671

Float det = -Sqr(B) + 4 * A * C; Float invDet = 1 / det; Float uSqrt = SafeSqrt(det * C), vSqrt = SafeSqrt(A * det); int s0 = pstd::ceil(st[0] - 2 * invDet * uSqrt);

```rust
   int s1 = pstd::floor(st[0] + 2 * invDet * uSqrt);
   int t0 = pstd::ceil(st[1] - 2 * invDet * vSqrt);
   int t1 = pstd::floor(st[1] + 2 * invDet * vSqrt);
```

```text
   Float 23
   MIPMap::LevelResolution() 667
   Point2i 92
   SafeSqrt() 1034
   Sqr() 1034
```

![Figure 10.19](images/img-1053-228.png)

*Figure 10.19: Finding the r2 Ellipse Value for the EWA Filter Table Lookup.*

Now that the bounding box is known, the EWA algorithm loops over the texels, transforming each one to the coordinate system where the texture lookup point (s, t) is at the origin with a translation. It then evaluates the ellipse equation to see if the texel is inside the ellipse (Figure 10.19) and computes the filter weight for the texel if so. The final filtered value returned is a weighted sum over texels T (s′, t′) inside the ellipse, where f is

the Gaussian filter function: 〈Scan over ellipse bound and evaluate quadratic equation to filter image〉 ≡ 671

T sum{}; Float sumWts = 0; for (int it = t0; it <= t1; ++it) { Float tt = it - st[1]; for (int is = s0; is <= s1; ++is) { Float ss = is - st[0];

```rust
                〈Compute squared radius and filter texel if it is inside the ellipse 673〉
       }
   }
   return sum / sumWts;
```

A nice feature of the implicit equation e(s, t) is that its value at a particular texel is the squared ratio of the distance from the center of the ellipse to the texel to the distance from the center of the ellipse to the ellipse boundary along the line through that texel (Figure 10.19). This value can be used to index into a precomputed lookup table of Gaussian filter function values. 〈Compute squared radius and filter texel if it is inside the ellipse〉 ≡ 673

Float r2 = A * Sqr(ss) + B * ss * tt + C * Sqr(tt); if (r2 < 1) { int index = std::min<int>(r2 * MIPFilterLUTSize, MIPFilterLUTSize - 1); Float weight = MIPFilterLUT[index]; sum += weight * Texel<T>(level, {is, it}); sumWts += weight; }

```text
   Float 23
   GaussianFilter 522
   MIPFilterLUTSize 674
   Sqr() 1034
```

The lookup table is precomputed and available as a constant array. Similar to the Gaussian Filter used for image reconstruction, the filter function is offset so that it goes to zero at the end of its extent rather than having an abrupt step. It is α = 2 was used for the table in pbrt. Because the table is indexed with squared distances from the filter center r2, each entry stores a value e−αr, rather than . 〈MIPMap EWA Lookup Table Definition〉 ≡ static constexpr int MIPFilterLUTSize = 128; static PBRT_CONST Float MIPFilterLUT[MIPFilterLUTSize] = { 〈MIPMap EWA Lookup Table Values〉 };

## 10.5 MATERIAL INTERFACE AND IMPLEMENTATIONS

With a variety of textures available, we will turn to materials, first introducing the material interface and then a few material implementations. pbrt’s materials all follow a similar form, evaluating textures to get parameter values that are used to initialize their particular BSDF model. Therefore, we will only include a few of their implementations in the text here.

The Material interface is defined by the Material class, which can be found in the file base/material.rs. pbrt includes the implementations of 11 materials; these are enough that we have collected all of their type names in a fragment that is not included in the text.

〈Material Definition〉 ≡ class Material : public TaggedPointer<〈Material Types〉> { public: 〈Material Interface 676〉 };

One of the most important methods that Material implementations must provide is GetBxDF(). It has the following signature: template <typename TextureEvaluator> ConcreteBxDF GetBxDF(TextureEvaluator texEval, MaterialEvalContext ctx, SampledWavelengths &lambda) const;

There are a few things to notice in its declaration. First, it is templated based on a type TextureEvaluator. This class is used by materials to, unsurprisingly, evaluate their textures. We will discuss it further in a page or two, as well as MaterialEvalContext, which serves a similar role to TextureEvalContext.

Most importantly, note the return type, ConcreteBxDF. This type is specific to each Material and should be replaced with the actual BxDF type that the material uses. (For example, the DiffuseMaterial returns a DiffuseBxDF.) Different materials thus have different signatures for their GetBxDF() methods. This is unusual for an interface method in C++ and is not usually allowed with regular C++ virtual functions, though we will see shortly how pbrt handles the variety of them.

Each Material is also responsible for defining the type of BxDF that it returns from its GetBxDF() method with a local type definition for the type BxDF. For example, Diffuse Material has using BxDF = DiffuseBxDF; in the body of its definition.

```text
   BxDF 538
   DiffuseBxDF 546
   DiffuseMaterial 678
   Float 23
   Material 674
   MaterialEvalContext 676
   MIPFilterLUTSize 674
   PBRT_CONST 929
   TaggedPointer 1073
   TextureEvalContext 650
```

The value of defining the interface in this way is that doing so makes it possible to write generic BSDF evaluation code that is templated on the type of material. Such code can then allocate storage for the BxDF on the stack, for whatever type of BxDF the material uses. pbrt’s wavefront renderer, which is described in Chapter 15, takes advantage of this opportunity. (Further details and discussion of its use there are in Section 15.3.9.) A disadvantage of this design is that materials cannot return different BxDF types depending on their parameter values; they are limited to the one that they declare. The Material class provides a GetBSDF() method that handles the variety of material BxDF return types. It requires some C++ arcana, though it centralizes the complexity of handling the diversity of types returned from the GetBxDF() methods. Material::GetBSDF() has the same general form of most of the dynamic dispatch method implementations in pbrt. (We have elided almost all of them from the text since most of them are boilerplate code.) Here we define a lambda function, getBSDF, and call the Dispatch() method that Material inherits from TaggedPointer. Recall that Dispatch() uses type information encoded in a 64-bit pointer to determine which concrete material type the Material points to before casting the pointer to that type and passing it to the lambda. 〈Material Inline Method Definitions〉 ≡ template <typename TextureEvaluator> BSDF Material::GetBSDF(

```rust
         TextureEvaluator texEval, MaterialEvalContext ctx,
         SampledWavelengths &lambda, ScratchBuffer
         &scratchBuffer) const {
       〈Define getBSDF lambda function for Material::GetBSDF() 675〉
       return Dispatch(getBSDF); }
```

getBSDF is a C++ generic lambda: when it is called, the auto mtl parameter will have a concrete type, that of a reference to a pointer to one of the materials enumerated in the 〈Material Types〉 fragment. Given mtl, then, we can find the concrete type of its material and thence the type of its BxDF. If a material does not return a BxDF, it should use void for its BxDF type definition. In that case, an unset BSDF is returned. (The MixMaterial is the only such Material in pbrt.)

〈Define getBSDF lambda function for Material::GetBSDF()〉 ≡ 675

auto getBSDF = [&](auto mtl) -> BSDF { using ConcreteMtl = typename std::remove_reference_t<decltype(*mtl)>; using ConcreteBxDF = typename ConcreteMtl::BxDF; if constexpr (std::is_same_v<ConcreteBxDF, void>) return BSDF(); else { 〈Allocate memory for ConcreteBxDF and return BSDF for material 676〉 } };

The provided ScratchBuffer is used to allocate enough memory to store the material’s BxDF; using it is much more efficient than using C++’s new and delete operators here. That memory is then initialized with the value returned by the material’s GetBxDF() method before the complete BSDF is returned to the caller.

```text
   BSDF 544
   BxDF 538
   Material 674
   MaterialEvalContext 676
   MixMaterial 681
   SampledWavelengths 173
   ScratchBuffer 1078
   TaggedPointer 1073
   TaggedPointer::Dispatch() 1075
   TextureEvaluator 676
```

〈Allocate memory for ConcreteBxDF and return BSDF for material〉 ≡ 675

ConcreteBxDF *bxdf = scratchBuffer.Alloc<ConcreteBxDF>(); *bxdf = mtl->GetBxDF(texEval, ctx, lambda); return BSDF(ctx.ns, ctx.dpdus, bxdf);

Materials that incorporate subsurface scattering must define a GetBSSRDF() method that follows a similar form. They must also include a using declaration in their class definition that defines a concrete BSSRDF type. (The code for rendering BSSRDFs is included only in the online edition.) template <typename TextureEvaluator> ConcreteBSSRDF GetBSSRDF(TextureEvaluator texEval, MaterialEvalContext ctx, SampledWavelengths &lambda) const;

The Material class provides a corresponding GetBSSRDF() method that uses the provided ScratchBuffer to allocate storage for the material-specific BSSRDF. 〈Material Interface〉 ≡ 674

template <typename TextureEvaluator> BSSRDF GetBSSRDF(TextureEvaluator texEval, MaterialEvalContext ctx, SampledWavelengths &lambda, ScratchBuffer &buf) const;

The MaterialEvalContext that GetBxDF() and GetBSSRDF() take plays a similar role to other *EvalContext classes: it encapsulates only the values that are necessary for material evaluation. They are a superset of those that are used for texture evaluation, so it inherits from TextureEvalContext. Doing so has the added advantage that MaterialEvalContexts can be passed directly to the texture evaluation methods. 〈MaterialEvalContext Definition〉 ≡ struct MaterialEvalContext : public TextureEvalContext { 〈MaterialEvalContext Public Methods 676〉 Vector3f wo; Normal3f ns; Vector3f dpdus; };

As before, there is not only a constructor that initializes a MaterialEvalContext from a SurfaceInteraction but also a constructor that takes the values for the members individually (not included here). 〈MaterialEvalContext Public Methods〉 ≡ 676

MaterialEvalContext() = default; MaterialEvalContext(const SurfaceInteraction &si) : TextureEvalContext(si), wo(si.wo),

```rust
   ns(si.shading.n), dpdus(si.shading.dpdu) {}
```

A TextureEvaluator is a class that is able to evaluate some or all of pbrt’s texture types. One of its methods takes a set of textures and reports whether it is capable of evaluating them, while others actually evaluate textures. On the face of it, there is no obvious need for such a class: why not allow Materials to call the Texture Evaluate() methods directly? This additional layer of abstraction aids performance with the wavefront integrator; it makes it possible to separate materials into those that have lightweight textures and those with heavyweight textures and to process them separately. Doing so is beneficial to performance on the GPU; see Section 15.3.9 for further discussion.

```text
   BSDF 544
   Interaction::wo 137
   Material 674
   MaterialEvalContext 676
   MaterialEvalContext::dpdus 676
   MaterialEvalContext::ns 676
   MaterialEvalContext::wo 676
   Normal3f 94
   SampledWavelengths 173
   ScratchBuffer 1078
   ScratchBuffer::Alloc() 1078
   SurfaceInteraction 138
   SurfaceInteraction:: shading::dpdu 139
   SurfaceInteraction:: shading::n 139
   Texture 655
   TextureEvalContext 650
   TextureEvaluator 676
   UniversalTextureEvaluator 677
   Vector3f 86
```

For now we will only define the UniversalTextureEvaluator, which can evaluate all textures. In practice, the indirection it adds is optimized away by the compiler such that it introduces no runtime overhead. It is used with all of pbrt’s integrators other than the one defined in Chapter 15.

〈UniversalTextureEvaluator Definition〉 ≡ class UniversalTextureEvaluator { public: 〈UniversalTextureEvaluator Public Methods 677〉

};

TextureEvaluators must provide a CanEvaluate() method that takes lists of FloatTextures and SpectrumTextures. They can then examine the types of the provided textures to determine if they are able to evaluate them. For the universal texture evaluator, the answer is always the same. 〈UniversalTextureEvaluator Public Methods〉 ≡ 677

bool CanEvaluate(std::initializer_list<FloatTexture>, std::initializer_list<SpectrumTexture>) const { return true; }

TextureEvaluators must also provide operator() method implementations that evaluate a given texture. Thus, given a texture evaluator texEval, material code should use the expression texEval(tex, ctx) rather than tex.Evaluate(ctx). The implementation of this method is again trivial for the universal evaluator. (A corresponding method for spectrum textures is effectively the same and not included here.) 〈UniversalTextureEvaluator Method Definitions〉 ≡ Float UniversalTextureEvaluator::operator()(FloatTexture tex, TextureEvalContext ctx) { return tex.Evaluate(ctx); }

Returning to the Material interface, all materials must provide a CanEvaluateTextures() method that takes a texture evaluator. They should return the result of calling its Can Evaluate() method with all of their textures provided. Code that uses Materials is then responsible for ensuring that a Material’s GetBxDF() or GetBSSRDF() method is only called with a texture evaluator that is able to evaluate its textures. 〈Material Interface〉 +≡ 674

template <typename TextureEvaluator> bool CanEvaluateTextures(TextureEvaluator texEval) const;

Materials also may modify the shading normals of objects they are bound to, usually in order to introduce the appearance of greater geometric detail than is actually present. The Material interface has two ways that they may do so, normal mapping and bump mapping.

pbrt’s normal mapping code, which will be described in Section 10.5.3, takes an image that specifies the shading normals. A nullptr value should be returned by this interface method if no normal map is included with a material. 〈Material Interface〉 +≡ 674

const Image *GetNormalMap() const;

Alternatively, shading normals may be specified via bump mapping, which takes a displacement function that specifies surface detail with a FloatTexture. A nullptr value should be returned if no such displacement function has been specified.

```text
    Float 23
    FloatTexture 656
    FloatTexture::Evaluate() 656
    Image 1079
    Material 674
    SpectrumTexture 656
    TextureEvalContext 650
    TextureEvaluator 676
```

〈Material Interface〉 +≡ 674

FloatTexture GetDisplacement() const;

What should be returned by HasSubsurfaceScattering() method implementations should be obvious; this method is used to determine for which materials in a scene it is necessary to do the additional processing to model that effect. 〈Material Interface〉 +≡ 674

bool HasSubsurfaceScattering() const;

### 10.5.1 MATERIAL IMPLEMENTATIONS

With the preliminaries covered, we will now present a few material implementations. All the Materials in pbrt are fairly basic bridges between Textures and BxDFs, so we will focus here on their basic form and some of the unique details of one of them.

Diffuse Material

DiffuseMaterial is the simplest material implementation and is a good starting point for understanding the material requirements. 〈DiffuseMaterial Definition〉 ≡ class DiffuseMaterial { public: 〈DiffuseMaterial Type Definitions 678〉 〈DiffuseMaterial Public Methods 678〉 private: 〈DiffuseMaterial Private Members 678〉 };

These are the BxDF and BSSRDF type definitions for DiffuseMaterial. Because this material does not include subsurface scattering, BSSRDF can be set to be void.

〈DiffuseMaterial Type Definitions〉 ≡ 678

using BxDF = DiffuseBxDF; using BSSRDF = void;

The constructor initializes the following member variables with provided values, so it is not included here. 〈DiffuseMaterial Private Members〉 ≡ 678

Image *normalMap; FloatTexture displacement; SpectrumTexture reflectance;

The CanEvaluateTextures() method is easy to implement; the various textures used for BSDF evaluation are passed to the given TextureEvaluator. Note that the displacement texture is not included here; if present, it is handled separately by the bump mapping code. 〈DiffuseMaterial Public Methods〉 ≡ 678

template <typename TextureEvaluator> bool CanEvaluateTextures(TextureEvaluator texEval) const { return texEval.CanEvaluate({}, {reflectance}); }

There is also not very much to GetBxDF(); it evaluates the reflectance texture, clamping the result to the range of valid reflectances before passing it along to the DiffuseBxDF constructor and returning a DiffuseBxDF.

```text
   BxDF 538
   DiffuseBxDF 546
   DiffuseMaterial 678
```

```text
   FloatTexture 656
   Image 1079
   Material 674
   SpectrumTexture 656
   Texture 655
   TextureEvaluator 676
   TextureEvaluator:: CanEvaluate() 677
```

〈DiffuseMaterial Public Methods〉 +≡ 678

template <typename TextureEvaluator> DiffuseBxDF GetBxDF(TextureEvaluator texEval, MaterialEvalContext ctx, SampledWavelengths &lambda) const { SampledSpectrum r = Clamp(texEval(reflectance, ctx, lambda), 0, 1); return DiffuseBxDF(r); }

GetNormalMap() and GetDisplacement() return the corresponding member variables and the remaining methods are trivial; see the source code for details.

Dielectric Material DielectricMaterial represents a dielectric interface.

〈DielectricMaterial Definition〉 ≡ class DielectricMaterial { public: 〈DielectricMaterial Type Definitions 679〉 〈DielectricMaterial Public Methods 679〉 private: 〈DielectricMaterial Private Members 679〉 };

It returns a DielectricBxDF and does not include subsurface scattering.

〈DielectricMaterial Type Definitions〉 ≡ 679

using BxDF = DielectricBxDF; using BSSRDF = void;

DielectricMaterial has a few more parameters than DiffuseMaterial. The index of refraction is specified with a SpectrumTexture so that it may vary with wavelength. Note also that two roughness values are stored, which allows the specification of an anisotropic microfacet distribution. If the distribution is isotropic, this leads to a minor inefficiency in storage and, shortly, texture evaluation, since both are always evaluated.

〈DielectricMaterial Private Members〉 ≡ 679

Image *normalMap; FloatTexture displacement; FloatTexture uRoughness, vRoughness; bool remapRoughness; Spectrum eta;

GetBxDF() follows a similar form to DiffuseMaterial, evaluating various textures and using their results to initialize the returned DielectricBxDF. 〈DielectricMaterial Public Methods〉 ≡ 679

template <typename TextureEvaluator> DielectricBxDF GetBxDF(TextureEvaluator texEval, MaterialEvalContext ctx, SampledWavelengths &lambda) const { 〈Compute index of refraction for dielectric material 680〉 〈Create microfacet distribution for dielectric material 680〉 〈Return BSDF for dielectric material 680〉 }

```text
   BxDF 538
   Clamp() 1033
   DielectricBxDF 563
   DiffuseBxDF 546
   FloatTexture 656
   Image 1079
   MaterialEvalContext 676
   SampledSpectrum 171
   SampledWavelengths 173
   Spectrum 165
   SpectrumTexture 656
   TextureEvaluator 676
   TextureEvaluator::operator() 677
```

If the index of refraction is the same for all wavelengths, then all wavelengths will follow the same path if a ray is refracted. Otherwise, they will go in different directions—this is dispersion. In that case, pbrt only follows a single ray path according to the first wavelength in SampledWavelengths rather than tracing multiple rays to track each of them, and a call to SampledWavelengths::TerminateSecondary() is necessary. (See Section 4.5.4 for more information.) DielectricMaterial therefore calls TerminateSecondary() unless the index of refraction is known to be constant, as determined by checking if eta’s Spectrum type is a Constant Spectrum. This check does not detect all cases where the sampled spectrum values are all the same, but it catches most of them in practice, and unnecessarily terminating the secondary wavelengths affects

performance but not correctness. A bigger shortcoming of the implementation here is that there is no dispersion if light is reflected at a surface and not refracted. In that case, all wavelengths could still be followed. However, how light paths will be sampled at the surface is not known at this point in program execution. 〈Compute index of refraction for dielectric material〉 ≡ 679

Float sampledEta = eta(lambda[0]); if (!eta.template Is<ConstantSpectrum> ()) lambda.TerminateSecondary();

It can be convenient to specify a microfacet distribution’s roughness with a scalar parameter in [0, 1], where values close to zero correspond to near-perfect specular reflection, rather than by specifying α values directly. The RoughnessToAlpha() method performs a mapping that gives a reasonably intuitive control for surface appearance. 〈TrowbridgeReitzDistribution Public Methods〉 +≡ 575

static Float RoughnessToAlpha(Float roughness) { return std::sqrt(roughness); }

The GetBxDF() method then evaluates the roughness textures and remaps the returned values if required. 〈Create microfacet distribution for dielectric material〉 ≡ 679

Float urough = texEval(uRoughness, ctx), vrough = texEval(vRoughness, ctx); if (remapRoughness) { urough = TrowbridgeReitzDistribution::RoughnessToAlpha(urough); vrough = TrowbridgeReitzDistribution::RoughnessToAlpha(vrough); } TrowbridgeReitzDistribution distrib(urough, vrough);

Given the index of refraction and microfacet distribution, it is easy to pull the pieces together to return the final BxDF. 〈Return BSDF for dielectric material〉 ≡ return DielectricBxDF(sampledEta, distrib);

```text
   ConstantSpectrum 167
   DielectricBxDF 563
   DielectricMaterial 679
   DielectricMaterial::eta 679
   DielectricMaterial:: remapRoughness 679
   DielectricMaterial:: uRoughness 679
```

```text
   DielectricMaterial:: vRoughness 679
   Float 23
   SampledWavelengths 173
   SampledWavelengths:: TerminateSecondary() 174
   Spectrum 165
   TaggedPointer::Is() 1074
   TrowbridgeReitzDistribution 575
   TrowbridgeReitzDistribution::RoughnessToAlpha() 680
```

Mix Material The final material implementation that we will describe in the text is MixMaterial, which stores two other materials and uses a Float-valued texture to blend between them. 〈MixMaterial Definition〉 ≡ class MixMaterial { public: 〈MixMaterial Type Definitions〉 〈MixMaterial Public Methods 681〉 private: 〈MixMaterial Private Members 681〉 }; 〈MixMaterial Private Members〉 ≡ 681

FloatTexture amount; Material materials[2];

MixMaterial does not cleanly fit into pbrt’s Material abstraction. For example, it is unable to define a single BxDF type that it will return, since its two constituent materials may have different BxDFs, and may themselves be MixMaterials, for that matter. Thus, MixMaterial requires special handling by the code that uses materials. (For example, there is a special case for MixMaterials in the SurfaceInteraction::GetBSDF() method described in Section 10.5.2.) This is not ideal: as a general point of software design, it would be better to have abstractions that make it possible to provide this functionality without requiring special-case handling in calling code. However, we were unable to find a clean way to do this while still being able to statically reason about the type of BxDF a material will return; that aspect of the Material interface offers enough of a performance benefit that we did not want to change it. Therefore, when a MixMaterial is encountered, one of its constituent materials is randomly chosen, with probability given by the floating-point amount texture. Thus, a 50/50 mix of two materials is not represented by

the average of their respective BSDFs and so forth, but instead by each of them being evaluated half the time. This is effectively the material analog of the stochastic alpha test that was described in Section 7.1.1. The ChooseMaterial() method implements the logic.

〈MixMaterial Public Methods〉 ≡ 681

template <typename TextureEvaluator> Material ChooseMaterial(TextureEvaluator texEval, MaterialEvalContext ctx) const { Float amt = texEval(amount, ctx); if (amt <= 0) return materials[0]; if (amt >= 1) return materials[1]; Float u = HashFloat(ctx.p, ctx.wo, materials[0], materials[1]); return (amt < u) ? materials[0] : materials[1]; }

```text
   BSDF 544
   Float 23
   FloatTexture 656
   HashFloat() 1042
   Material 674
   MaterialEvalContext 676
   MaterialEvalContext::wo 676
   MixMaterial::amount 681
   MixMaterial::materials 681
   SurfaceInteraction::GetBSDF() 682
   TextureEvalContext::p 650
   TextureEvaluator 676
```

Stochastic selection of materials can introduce noise in images at low sampling rates; see Figure 10.20. However, a few tens of samples are generally plenty to resolve any visual error. Furthermore, this approach does bring benefits: sampling and evaluation of the resulting BSDF is more efficient than if it was a weighted sum of the BSDFs from the constituent materials. MixMaterial provides an accessor that makes it possible to traverse all the materials in the scene, including those nested inside a MixMaterial, so that it is possible to perform operations such as determining which types of materials are and are not present in a scene. 〈MixMaterial Public Methods〉 +≡ 681

Material GetMaterial(int i) const { return materials[i]; }

![Figure 10.20](images/img-1068-229.png)

*Figure 10.20: Effect of Sampling Rate with the MixMaterial. In this scene, the MixMaterial is used to blend between blue and red diffuse materials for the dragon, using an equal weighting for each. (a) With one sample per pixel, there is visible noise in the corresponding pixels since each pixel only includes one of the two constituent materials. (b) With a sufficient number of samples (here, 128), stochastic selection of materials causes no visual harm. In practice, the pixel sampling rates necessary to reduce other forms of error from simulating light transport are almost always enough to resolve stochastic material sampling.*

A fatal error is issued if the GetBxDF() method is called. A call to GetBSSRDF() is handled similarly, in code not included here.

〈MixMaterial Public Methods〉 +≡ 681

template <typename TextureEvaluator> void GetBxDF(TextureEvaluator texEval, MaterialEvalContext ctx, SampledWavelengths &lambda) const { LOG_FATAL(“MixMaterial::GetBxDF() shouldn’t be called”); }

### 10.5.2 FINDING THE BSDF AT A SURFACE

Because pbrt’s Integrators use the SurfaceInteraction class to collect the necessary information associated with each intersection point, we will add a GetBSDF() method to this class that handles all the details related to computing the BSDF at its point.

```text
   BSDF 544
   Camera 206
   Integrator 22
   LOG_FATAL() 1065
   MaterialEvalContext 676
   MixMaterial 681
   RayDifferential 96
   SampledWavelengths 173
   Sampler 469
   ScratchBuffer 1078
   SurfaceInteraction 138
   TextureEvaluator 676
```

〈SurfaceInteraction Method Definitions〉 +≡ BSDF SurfaceInteraction::GetBSDF( const RayDifferential &ray, SampledWavelengths &lambda, Camera camera, ScratchBuffer &scratchBuffer, Sampler sampler) { 〈Estimate (u, v) and position differentials at intersection point 683〉 〈Resolve MixMaterial if necessary 683〉 〈Return unset BSDF if surface has a null material 683〉 〈Evaluate normal or bump map, if present 683〉 〈Return BSDF for surface interaction 684〉 }

This method first calls the SurfaceInteraction’s ComputeDifferentials() method to compute information about the projected size of the surface area around the intersection on the image plane for use in texture antialiasing. 〈Estimate (u, v) and position differentials at intersection point〉 ≡ 682

ComputeDifferentials(ray, camera, sampler.SamplesPerPixel());

As described in Section 10.5.1, if there is a MixMaterial at the intersection point, it is necessary to resolve it to be a regular material. A while loop here ensures that nested MixMaterials are handled correctly.

〈Resolve MixMaterial if necessary〉 ≡ 682

while (material.Is<MixMaterial>()) { MixMaterial *mix = material.Cast<MixMaterial>(); material = mix->ChooseMaterial(UniversalTextureEvaluator(), *this); }

If the final material is nullptr, it represents a non-scattering interface between two types of participating media. In this case, a default uninitialized BSDF is returned.

〈Return unset BSDF if surface has a null material〉 ≡ 682

if (!material) return {};

Otherwise, normal or bump mapping is performed before the BSDF is created. 〈Evaluate normal or bump map, if present〉 ≡ 682

FloatTexture displacement = material.GetDisplacement(); const Image *normalMap = material.GetNormalMap(); if (displacement || normalMap) { 〈Get shading ∂p/∂u and ∂p/∂v using normal or bump map 683〉 Normal3f ns(Normalize(Cross(dpdu, dpdv))); SetShadingGeometry(ns, dpdu, dpdv, shading.dndu, shading.dndv, false); }

The appropriate utility function for normal or bump mapping is called, depending on which technique is to be used. 〈Get shading ∂p/∂u and ∂p/∂v using normal or bump map〉 ≡ 683

Vector3f dpdu, dpdv; if (normalMap) NormalMap(*normalMap, *this, &dpdu, &dpdv); else BumpMap(UniversalTextureEvaluator(), displacement, *this, &dpdu, &dpdv);

With differentials both for texture filtering and for shading geometry now settled, the Material::GetBSDF() method can be called. Note that the universal texture evaluator is used both here and previously in the method, as there is no need to distinguish between different texture complexities in this part of the system.

```text
   BSDF 544
```

```text
    BumpMap() 687
    Cross() 91
    FloatTexture 656
    Image 1079
    Material::GetBSDF() 675
    Material::GetDisplacement() 678
    Material::GetNormalMap() 677
    MixMaterial 681
    MixMaterial::ChooseMaterial() 681
    Normal3f 94
    Normalize() 88
    NormalMap() 685
    Sampler::SamplesPerPixel() 469
    SurfaceInteraction::ComputeDifferentials() 637
    SurfaceInteraction::material 398
    SurfaceInteraction:: SetShadingGeometry() 140
    SurfaceInteraction:: shading::dndu 139
    SurfaceInteraction:: shading::dndv 139
    TaggedPointer::Cast() 1074
    TaggedPointer::Is() 1074
    UniversalTextureEvaluator 677
    Vector3f 86
```

〈Return BSDF for surface interaction〉 ≡ 682

BSDF bsdf = material.GetBSDF(UniversalTextureEvaluator(), *this, lambda, scratchBuffer); if (bsdf && GetOptions().forceDiffuse) { 〈Override bsdf with diffuse equivalent 684〉 } return bsdf;

pbrt provides an option to override all the materials in a scene with equivalent diffuse BSDFs; doing so can be useful for some debugging problems. In this case, the hemispherical–directional reflectance is used to initialize a DiffuseBxDF. 〈Override bsdf with diffuse equivalent〉 ≡ 684

SampledSpectrum r = bsdf.rho(wo, {sampler.Get1D()}, {sampler.Get2D()}); bsdf = BSDF(shading.n, shading.dpdu, scratchBuffer.Alloc<DiffuseBxDF>(r));

The SurfaceInteraction::GetBSSRDF() method, not included here, follows a similar path before calling Material::GetBSSRDF().

### 10.5.3 NORMAL MAPPING

Normal mapping is a technique that maps tabularized surface normals stored in images to surfaces and uses them to specify shading normals in order to give the appearance of fine geometric detail. With normal maps, one must choose a coordinate system for the stored normals. While any coordinate system may be chosen, one of the most useful is the local shading coordinate system at each point on a surface where the z axis is aligned with the surface normal and tangent vectors are aligned with x and y. (This is the same as the reflection coordinate system described in Section 9.1.1.) When that coordinate system is used, the approach is called tangent-space normal mapping. With tangent-space normal mapping, a given normal map can be applied to a variety of shapes, while choosing a coordinate system like object space would closely couple a normal map’s encoding to a specific geometric object. Normal maps are traditionally encoded in RGB images, where red, green, and blue respectively store the x, y, and z components of the surface normal. When tangent-space normal mapping is used, normal map images are typically predominantly blue, reflecting the fact that the z component of the surface normal has the largest magnitude unless the normal has been substantially perturbed. (See Figure 10.21.) This RGB encoding brings us to an unfortunate casualty from the adoption of spectral rendering in this version of pbrt: while pbrt’s SpectrumTextures previously returned RGB colors, they now return point-sampled spectral values. If an RGB image map is used for a spectrum texture, it is not possible to exactly reconstruct the original RGB colors; there will unavoidably be error in the Monte Carlo estimator that must be evaluated to find RGB. Introducing noise in the orientations of surface normals is unacceptable, since it would lead to systemic bias in rendered images. Consider a bumpy shiny object: error in the surface normal would lead to scattered rays intersecting objects that they would never intersect given the correct normals, which could cause arbitrarily large error.

```text
   BasicPBRTOptions:: forceDiffuse 1031
   BSDF 544
   BSDF::rho() 546
   DiffuseBxDF 546
   GetOptions() 1032
   Material::GetBSDF() 675
```

```text
   Material::GetBSSRDF() 676
   SampledSpectrum 171
   Sampler::Get1D() 470
   Sampler::Get2D() 470
   ScratchBuffer::Alloc() 1078
   SpectrumTexture 656
   SurfaceInteraction::material 398
   SurfaceInteraction:: shading::dpdu 139
   SurfaceInteraction:: shading::n 139
   UniversalTextureEvaluator 677
```

We might avoid that problem by augmenting the SpectrumTexture interface to include a method that returned RGB color, introducing a separate RGBTexture interface and texture implementations, or by introducing a NormalTexture that returned normals directly. Any of these could cleanly support normal mapping, though all would require a significant amount of additional code.

![Figure 10.21](images/img-1073-230.png)

*Figure 10.21: (a) A normal map modeling wrinkles for a pillow model. (b) Pillow geometry without normal map. (c) When applied to the pillow, the normal map gives a convincing approximation to more detailed geometry than is actually present. (Scene courtesy of Angelo Ferretti.)*

Because the capability of directly looking up RGB values is only needed for normal mapping, the NormalMap() function therefore takes an Image to specify the normal map. It assumes that the first three channels of the image represent red, green, and blue. With this approach we have lost the benefits of being able to scale and mix textures as well as the ability to apply a

variety of mapping functions to compute texture coordinates. While that is unfortunate, those capabilities are less often used with normal maps than with other types of textures, and so we prefer not to make the Texture interfaces more complex purely for normal mapping. 〈Normal Mapping Function Definitions〉 ≡ void NormalMap(const Image &normalMap, const NormalBumpEvalContext &ctx, Vector3f *dpdu, Vector3f *dpdv) { 〈Get normalized normal vector from normal map 686〉 〈Transform tangent-space normal to rendering space 686〉 〈Find ∂p/∂u and ∂p/∂v that give shading normal 687〉 }

Both NormalMap() and BumpMap() take a NormalBumpEvalContext to specify the local geometric information for the point where the shading geometry is being computed. 〈NormalBumpEvalContext Definition〉 ≡ struct NormalBumpEvalContext { 〈NormalBumpEvalContext Public Methods 686〉 〈NormalBumpEvalContext Public Members 686〉 };

As usual, it has a constructor, not included here, that performs initialization given a Surface Interaction.

```text
   BumpMap() 687
   Image 1079
   NormalBumpEvalContext 685
   NormalMap() 685
   SurfaceInteraction 138
   Vector3f 86
```

〈NormalBumpEvalContext Public Members〉 ≡ 685

Point3f p; Point2f uv; Normal3f n; struct { Normal3f n; Vector3f dpdu, dpdv; Normal3f dndu, dndv; } shading; Float dudx = 0, dudy = 0, dvdx = 0, dvdy = 0; Vector3f dpdx, dpdy;

It also provides a conversion operator to TextureEvalContext, which only needs a subset of the values stored in NormalBumpEvalContext. 〈NormalBumpEvalContext Public Methods〉 ≡ 685

operator TextureEvalContext() const { return TextureEvalContext(p, dpdx, dpdy, n, uv, dudx, dudy, dvdx, dvdy); }

The first step in the normal mapping computation is to read the tangent-space normal vector from the image map. The image wrap mode is hard-coded here since Repeat is almost always the desired mode, though it would be easy to allow the wrap mode to be set via a parameter. Note also that the v coordinate is inverted, again following the image texture coordinate convention discussed in Section 10.4.2. Normal maps are traditionally encoded in fixed-point image formats with pixel values that range from 0 to 1. This encoding allows the use of compact 8-bit pixel representations as well as compressed image formats that are supported by GPUs. Values read from the image must therefore be remapped to the range [−1, 1] to reconstruct an associated normal vector. The normal vector must be renormalized, as both the quantization in the image pixel format and the bilinear interpolation may have caused it to be non-unit-length. 〈Get normalized normal vector from normal map〉 ≡ 685

WrapMode2D wrap(WrapMode::Repeat); Point2f uv(ctx.uv[0], 1 - ctx.uv[1]); Vector3f ns(2 * normalMap.BilerpChannel(uv, 0, wrap) - 1, 2 * normalMap.BilerpChannel(uv, 1, wrap) - 1, 2 * normalMap.BilerpChannel(uv, 2, wrap) - 1); ns = Normalize(ns);

In order to transform the normal to rendering space, a Frame can be used to specify a coordinate system where the original shading normal is aligned with the +z axis. Transforming the tangent-space normal into this coordinate system gives the rendering-space normal. 〈Transform tangent-space normal to rendering space〉 ≡ 685

Frame frame = Frame::FromZ(ctx.shading.n); ns = frame.FromLocal(ns);

```text
   Float 23
   Frame 133
   Frame::FromLocal() 134
```

```text
    Frame::FromZ() 134
    GramSchmidt() 90
    Image::BilerpChannel() 1082
    Normal3f 94
    NormalBumpEvalContext::dpdx 686
    NormalBumpEvalContext::dpdy 686
    NormalBumpEvalContext::dudx 686
    NormalBumpEvalContext::dudy 686
    NormalBumpEvalContext::dvdx 686
    NormalBumpEvalContext::dvdy 686
    NormalBumpEvalContext::n 686
    NormalBumpEvalContext::p 686
    NormalBumpEvalContext:: shading::n 686
    NormalBumpEvalContext::uv 686
    Normalize() 88
    Point2f 92
    Point3f 92
    TextureEvalContext 650
    Vector3f 86
    WrapMode 1082
    WrapMode2D 1082
```

This function returns partial derivatives of the surface that account for the shading normal rather than the shading normal itself. Suitable partial derivatives can be found in two steps. First, a call to GramSchmidt() with the original ∂p/∂u and the new shading normal ns gives the closest vector to ∂p/∂u that is perpendicular to ns. ∂p/∂v is then found by taking the cross product of ns and the new ∂p/∂v, giving an orthogonal coordinate system. Both of these vectors are respectively scaled to have the same length as the original ∂p/∂u and ∂p/∂v vectors. 〈Find ∂p/∂u and ∂p/∂v that give shading normal〉 ≡ 685

Float ulen = Length(ctx.shading.dpdu), vlen = Length(ctx.shading.dpdv); *dpdu = Normalize(GramSchmidt(ctx.shading.dpdu, ns)) * ulen; *dpdv = Normalize(Cross(ns, *dpdu)) * vlen;

### 10.5.4 BUMP MAPPING

Another way to define shading normals is via a FloatTexture that defines a displacement at each point on the surface: each point p has a displaced point p′ associated with it, defined by p′ = p + d(p)n(p), where d(p) is the offset returned by the displacement texture at p and n(p) is the surface

normal at p (Figure 10.22). We can use this texture to compute shading normals so that the surface appears as if it actually had been offset by the displacement function, without modifying its geometry. This process is called bump mapping. For relatively small displacement functions, the visual effect of bump mapping can be quite convincing. An example of bump mapping is shown in Figure 10.23, which shows part of the San Miguel scene rendered with and without bump mapping. There, the bump map gives the appearance of a substantial amount of detail in the walls and floors that is not actually present in the geometric model. Figure 10.24 shows one of the image maps used to define the bump function in Figure 10.23. The BumpMap() function is responsible for computing the effect of bump mapping at the point being shaded given a particular displacement texture. Its implementation is based on finding an approximation to the partial derivatives ∂p/∂u and ∂p/∂v of the displaced surface and using them in place of the surface’s actual partial derivatives to compute the shading normal. Assume that the original surface is defined by a parametric function p(u, v), and the bump offset function is a scalar function d(u, v). Then the displaced surface is given by p′ (u, v) = p(u, v) + d(u, v)n(u, v), where n(u, v) is the surface normal at (u, v). 〈Bump Mapping Function Definitions〉 ≡ template <typename TextureEvaluator> void BumpMap(TextureEvaluator texEval, FloatTexture displacement, const NormalBumpEvalContext &ctx, Vector3f *dpdu, Vector3f *dpdv) { 〈Compute offset positions and evaluate displacement texture 689〉 〈Compute bump-mapped differential geometry 690〉 }

![Figure 10.22](images/img-1078-231.png)

*Figure 10.22: A displacement function associated with a material defines a new surface based on the old one, offset by the displacement amount along the normal at each point. pbrt does not compute a geometric representation of this displaced surface in the BumpMap() function, but instead uses it to compute shading normals for bump mapping.*

BumpMap() 687 Cross() 91 Float 23 FloatTexture 656 GramSchmidt() 90 Length() 88 NormalBumpEvalContext 685 NormalBumpEvalContext:: shading::dpdu 686 NormalBumpEvalContext:: shading::dpdv 686 Normalize() 88 TextureEvaluator 676 Vector3f 86

![Figure 10.23](images/img-1079-232.png)

*Figure 10.23: Detail of the San Miguel scene, rendered (a) without bump mapping and (b) with bump mapping. Bump mapping substantially increases the apparent geometric complexity of the model, without the increased rendering time and memory use that would result from a geometric representation with the equivalent amount of small-scale detail. (Scene courtesy of Guillermo M. Leal Llaguno.)*

![Figure 10.24](images/img-1080-233.png)

*Figure 10.24: The image used as a bump map for the tiles in the San Miguel rendering in Figure 10.23.*

The partial derivatives of p′ can be found using the chain rule. For example, the partial derivative in u is

We have already computed the value of ∂p(u, v)/∂u; it is ∂p/∂u and is available in the TextureEvalContext structure, which also stores the surface normal n(u, v) and the partial derivative ∂n(u, v)/∂u = ∂n/∂u. The displacement function d(u, v) can be readily evaluated, which leaves ∂d(u, v)/∂u as the only remaining term. There are two possible approaches to finding the values of ∂d(u, v)/∂u and ∂d(u, v)/∂v. One option would be to augment the FloatTexture interface with a method to compute partial derivatives of the underlying texture function. For example, for image map textures mapped to the surface directly using its (u, v) parameterization, these partial derivatives can be computed by subtracting adjacent texels in the u and v directions. However, this approach is difficult to extend to complex procedural textures like some

of the ones defined earlier in this chapter. Therefore, pbrt directly computes these values with forward differencing, without modifying the FloatTexture interface.

Recall the definition of the partial derivative:

Forward differencing approximates the value using a finite value of Δu and evaluating d(u, v) at two positions. Thus, the final expression for ∂p′/∂u is the following (for simplicity, we have dropped the explicit dependence on

(u, v) for some of the terms): Interestingly enough, most bump-mapping implementations ignore the final term under the assumption that d(u, v) is expected to be relatively small. (Since bump mapping is mostly useful for approximating small perturbations, this is a reasonable assumption.) The fact that many renderers do not compute the values ∂n/∂u and ∂n/∂v may also have something to do with this simplification. An implication of ignoring the last term is that the magnitude of the displacement function then does not affect the bump-mapped partial derivatives; adding a constant value to it globally does not affect the final result, since only differences of the bump function affect it. pbrt computes all three terms since it has ∂n/∂u and ∂n/∂v readily available, although in practice this final term rarely makes a visually noticeable difference. 〈Compute offset positions and evaluate displacement texture〉 ≡ 687

TextureEvalContext shiftedCtx = ctx; 〈Shift shiftedCtx du in the u direction 690〉 Float uDisplace = texEval(displacement, shiftedCtx); 〈Shift shiftedCtx dv in the v direction〉 Float vDisplace = texEval(displacement, shiftedCtx); Float displace = texEval(displacement, ctx);

One remaining issue is how to choose the offsets Δu and Δv for the finite differencing computations. They should be small enough that fine changes in d(u, v) are captured but large enough so that available floating-point precision is sufficient to give a good result. Here, we will choose Δu and Δv values that lead to an offset that is about half the image-space pixel sample

spacing and use them to update the appropriate member variables in the TextureEvalContext to reflect a shift to the offset position.

```text
   Float 23
   TextureEvalContext 650
```

〈Shift shiftedCtx du in the u direction〉 ≡ 689

Float du = .5f * (std::abs(ctx.dudx) + std::abs(ctx.dudy)); if (du == 0) du = .0005f; shiftedCtx.p = ctx.p + du * ctx.shading.dpdu; shiftedCtx.uv = ctx.uv + Vector2f(du, 0.f);

The 〈Shift shiftedCtx dv in the v direction〉 fragment is nearly the same as the fragment that shifts du, so it is not included here. Given the new positions and the displacement texture’s values at them, the partial derivatives can be computed directly using Equation (10.12): 〈Compute bump-mapped differential geometry〉 ≡ 687

*dpdu = ctx.shading.dpdu + (uDisplace - displace) / du * Vector3f(ctx.shading.n) + displace * Vector3f(ctx.shading.dndu); *dpdv = ctx.shading.dpdv + (vDisplace - displace) / dv * Vector3f(ctx.shading.n) + displace * Vector3f(ctx.shading.dndv);

## FURTHER READING

Ray Footprints The cone-tracing method of Amanatides (1984) was one of the first techniques for automatically estimating filter footprints for ray tracing. The beam-tracing algorithm of Heckbert and Hanrahan (1984) was another early extension of ray tracing to incorporate an area associated with each image sample rather than just an infinitesimal ray. The pencil-tracing method of Shinya et al. (1987) is another approach to this problem. Other related work on the topic of associating areas or footprints with rays includes Mitchell and Hanrahan’s paper (1992) on rendering caustics and Turkowski’s technical report (1993). Collins (1994) estimated the ray footprint by keeping a tree of all rays traced from a given camera ray, examining corresponding rays at the same level and position. The ray differentials used in pbrt are based on Igehy’s

(1999) formulation, which was extended by Suykens and Willems (2001) to handle glossy reflection in addition to perfect specular reflection. Belcour et al. (2017) applied Fourier analysis to the light transport equation in order to accurately and efficiently track ray footprints after scattering. Twelve floating-point values are required to store ray differentials, and Belcour et al.’s approach has similar storage requirements. This poses no challenge in a CPU ray tracer that only operates on one or a few rays at a time, but can add up to a considerable amount of storage (and consequently, bandwidth consumption) on the GPU. To address this issue, Akenine-Möller et al. (2019) developed a number of more space-efficient alternatives and showed their effectiveness for antialiasing that was further improved in subsequent work (Akenine-Möller et al. 2021; Boksansky et al. 2021). The approach we have implemented in CameraBase::Approximate_dp_dxy() was described by Li (2018).

Worley’s chapter in Texturing and Modeling (Ebert et al. 2003) on computing differentials for filter regions presents an approach similar to ours. See Elek et al. (2014) for an extension of ray differentials to include wavelength, which can improve results with spectral rendering.

```text
   CameraBase::Approximate_dp_dxy() 638
   Float 23
   NormalBumpEvalContext::dudx 686
   NormalBumpEvalContext::dudy 686
   NormalBumpEvalContext::p 686
   NormalBumpEvalContext:: shading::dndu 686
   NormalBumpEvalContext:: shading::dndv 686
   NormalBumpEvalContext:: shading::dpdu 686
   NormalBumpEvalContext:: shading::dpdv 686
   NormalBumpEvalContext:: shading::n 686
   NormalBumpEvalContext::uv 686
   TextureEvalContext::p 650
   TextureEvalContext::uv 650
   Vector2f 86
   Vector3f 86
```

Image Texture Maps Two-dimensional texture mapping with images was first introduced to graphics by Blinn and Newell (1976). Ever since Crow (1977) identified

aliasing as the source of many errors in images in graphics, much work has been done to find efficient and effective ways of antialiasing image maps. Dungan, Stenger, and Sutty (1978) were the first to suggest creating a pyramid of prefiltered texture images; they used the nearest texture sample at the appropriate level when looking up texture values, using supersampling in screen space to antialias the result. Feibush, Levoy, and Cook (1980) investigated a spatially varying filter function, rather than a simple box filter. (Blinn and Newell were aware of Crow’s results and used a box filter for their textures.) Williams (1983) used a MIP map image pyramid for texture filtering with trilinear interpolation. Shortly thereafter, Crow (1984) introduced summed area tables, which make it possible to efficiently filter over axis-aligned rectangular regions of texture space. Summed area tables handle anisotropy better than Williams’s method, although only for primarily axis-aligned filter regions. Heckbert (1986) wrote a good survey of early texture mapping algorithms. Greene and Heckbert (1986) originally developed the elliptically weighted average technique, and Heckbert’s master’s thesis (1989b) put the method on a solid theoretical footing. Fournier and Fiume (1988) developed an even higher-quality texture filtering method that focuses on using a bounded amount of computation per lookup. Nonetheless, their method appears to be less efficient than EWA overall. Lansdale’s master’s thesis (1991) has an extensive description of EWA and Fournier and Fiume’s method, including implementation details. A number of researchers have investigated generalizing Williams’s original method using a series of trilinear MIP map samples in an effort to increase quality without having to pay the price for the general EWA algorithm. By taking multiple samples from the MIP map, anisotropy is handled well while preserving the computational efficiency. Examples include Barkans’s (1997) description of texture filtering in the Talisman architecture, McCormack et al.’s (1999) Feline method, and Cant and Shrubsole’s (2000) technique. Manson and Schaefer (2013, 2014) have shown how to accurately approximate a variety of filter functions with a fixed small number of bilinearly interpolated sample values. An algorithm to convert an arbitrary filter into a set of bilinear lookups over multiple passes subject to a specified performance target was given by Schuster et al. (2020). These

sorts of approaches are particularly useful on GPUs, where hardware-accelerated bilinear interpolation is available. For scenes with many image textures where reading them all into memory simultaneously has a prohibitive memory cost, an effective approach can be to allocate a fixed amount of memory for image maps (a texture cache), load textures into that memory on demand, and discard the image maps that have not been accessed recently when the memory fills up (Peachey 1990). To enable good performance with small texture caches, image maps should be stored in a tiled format that makes it possible to load in small square regions of the texture independently of each other. Tiling techniques like these are used in graphics hardware to improve the performance of their texture memory caches (Hakura and Gupta 1997; Igehy et al. 1998, 1999). High-performance texture caching with parallel execution can be challenging because the cache contents may be frequently updated; it is desirable to minimize mutual exclusion in the cache implementation so that threads do not stall while others are updating the cache. For an effective approach to this problem, see Pharr (2017), who applied the read-copy update technique (McKenney and Slingwine 1998) to accomplish this. Smith’s (2002) website and document on audio resampling gives a good overview of resampling signals in one dimension. Heckbert’s (1989a) zoom source code is the canonical reference for image resampling. His implementation carefully avoids feedback without using auxiliary storage. A variety of texture synthesis algorithms have been developed that take an example texture image and then synthesize larger texture images that appear similar to the original texture while not being exactly the same. Survey articles by Wei et al. (2009) and Barnes and Zhang (2017) summarize work in this area. Convolutional neural networks have been applied to this task (Gatys et al. 2015; Sendik and Cohen-Or 2017), giving impressive results, and Frühstück et al. (2019) have showed the effectiveness of generative adversarial networks for this problem.

Solid Texturing and Noise Functions Three-dimensional solid texturing was originally developed by Gardner (1984, 1985), Perlin (1985a), and Peachey (1985). Norton, Rockwood, and Skolmoski (1982) developed the clamping method that is widely used for

antialiasing textures based on solid texturing. The general idea of procedural texturing, where texture is generated via computation rather than via looking up values from images, was introduced by Cook (1984), Perlin (1985a), and Peachey (1985). Noise functions, which randomly vary while still having limited frequency content, have been a key ingredient for many procedural texturing techniques. Perlin (1985a) introduced the first such noise function, and later revised it to correct a number of subtle shortcomings (Perlin 2002). (See also Kensler et al. (2008) for further improvements.) Many more noise functions have been developed; see Lagae et al. (2010) for a survey of work up to that year. Tricard et al. (2019) recently introduced a noise function (“phasor noise”) that can be filtered anisotropically and allows control of the orientation, frequency, and contrast of the noise function. Their paper also includes citations to other recent work on this topic. In recent years, the Shadertoy website, shadertoy.com, has become a hub of creative application of procedural modeling and texturing, all of it running interactively in web browsers. Shadertoy was developed by Quilez and Jeremias (2021).

Shading Languages The first languages and systems that supported the idea of user-supplied procedural shaders were developed by Cook (1984) and Perlin (1985a). (The texture composition model in this chapter is similar to Cook’s shade trees.) The RenderMan shading language, described in a paper by Hanrahan and Lawson (1990), remains the classic shading language in graphics, though a more modern shading language is available in Open Shading Language (OSL) (Gritz et al. 2010), which is open source and increasingly used for production rendering. It follows pbrt’s model of the shader returning a representation of the material rather than a final color value. See also Karrenberg et al. (2010), who introduced the AnySL shading language, which was designed for high performance as well as portability across multiple rendering systems (including pbrt).

See Ebert et al. (2003) and Apodaca and Gritz (2000) for techniques for writing procedural shaders; both of those have excellent discussions of issues related to antialiasing in procedural shaders.

Normal Mapping, Bump Mapping, and Shading Normals Blinn (1978) invented the bump-mapping technique. Kajiya (1985) generalized the idea of bump mapping the normal to frame mapping, which also perturbs the surface’s primary tangent vector and is useful for controlling the appearance of anisotropic reflection models. Normal mapping was introduced by Cohen et al. (1998). Mikkelsen’s thesis (2008) carefully investigates a number of the assumptions underlying bump mapping and normal mapping, proposes generalizations, and addresses a number of subtleties with respect to its application to real-time rendering. One visual shortcoming of normal and bump mapping is that those techniques do not naturally account for self-shadowing, where bumps cast shadows on the surface and prevent light from reaching nearby points. These shadows can have a significant impact on the appearance of rough surfaces. Max (1988) developed the horizon mapping technique, which efficiently accounts for this effect through precomputed information about each bump map. More recently, Conty Estevez et al. and Chiang et al. have introduced techniques based on microfacet shadowing functions to improve the visual fidelity of bump-mapped surfaces at shadow terminators (Conty Estevez et al. 2019, Chiang et al. 2019). Another challenging issue is that antialiasing bump and normal maps that have higher-frequency detail than can be represented in the image is quite difficult. In particular, it is not enough to remove high-frequency detail from the underlying function, but in general the BSDF needs to be modified to account for this detail. Fournier (1992) applied normal distribution functions to this problem, where the surface normal was generalized to represent a distribution of normal directions. Becker and Max (1993) developed algorithms for blending between bump maps and BRDFs that represented higher-frequency details. Schilling (1997, 2001) investigated this issue particularly for application to graphics hardware. Effective approaches to filtering bump maps were developed by Han et al. (2007) and Olano and Baker (2010). Both Dupuy et al. (2013) and Hery et al. (2014) developed techniques that convert displacements into anisotropic distributions of Beckmann microfacets. Further improvements to these

approaches were introduced by Kaplanyan et al. (2016), Tokuyoshi and Kaplanyan (2019), and Wu et al. (2019). A number of researchers have looked at the issue of antialiasing surface reflection functions. Early work in this area was done by Amanatides, who developed an algorithm to detect specular aliasing for a specific BRDF model (Amanatides 1992). Van Horn and Turk (2008) developed an approach to automatically generate MIP maps of reflection functions that represent the characteristics of shaders over finite areas in order to antialias them. Bruneton and Neyret (2012) surveyed the state of the art in this area, and Jarabo et al. (2014b) also considered perceptual issues related to filtering inputs to these functions. See also Heitz et al. (2014) for further work on this topic.

Displacement Mapping An alternative to bump mapping is displacement mapping, where the bump function is used to actually modify the surface geometry, rather than just perturbing the normal (Cook 1984; Cook et al. 1987). Advantages of displacement mapping include geometric detail on object silhouettes and the possibility of accounting for self-shadowing. Patterson and collaborators described an innovative algorithm for displacement mapping with ray tracing where the geometry is unperturbed, but the ray’s direction is modified such that the intersections that are found are the same as would be found with the displaced geometry (Patterson, Hoggar, and Logie 1991; Logie and Patterson 1994). Heidrich and Seidel (1998) developed a technique for computing direct intersections with procedurally defined displacement functions. One approach for displacement mapping has been to use an implicit function to define the displaced surface and to then take steps along rays until a zero crossing with the implicit function is found—this point is an intersection. This approach was first introduced by Hart (1996); see Donnelly (2005) for information about using this approach for displacement mapping on the GPU. (This approach was more recently popularized by Quilez (2015) on the Shadertoy website.) Another option is to finely tessellate the scene geometry and displace its vertices to define high-resolution meshes. Pharr and Hanrahan (1996) described an approach to this problem based on geometry caching, and Wang et al. (2000) described an

adaptive tessellation algorithm that reduces memory requirements. Smits, Shirley, and Stark (2000) lazily tessellate individual triangles, saving a substantial amount of memory. Measuring fine-scale surface geometry of real surfaces to acquire bump or displacement maps can be challenging. Johnson et al. (2011) developed a novel handheld system that can measure detail down to a few microns, which more than suffices for these uses.

Material Models Burley’s (2012) course notes describe a material model developed at Disney for feature films. This write-up includes extensive discussion of features of real-world reflection functions that can be observed in Matusik et al.’s (2003b) measurements of one hundred BRDFs and analyzes the ways that existing BRDF models do and do not fit these features well. These insights are then used to develop an “artist-friendly” material model that can express a wide range of surface appearances. The model describes reflection with a single color and ten scalar parameters, all of which are in the range [0, 1] and have fairly predictable effects on the appearance of the resulting material. An earlier material model designed to have intuitive parameters for artistic control was developed by Strauss (1990). The bidirectional texture function (BTF) is a generalization of the BRDF that was introduced by Dana et al. (1999). (BTFs are also referred to as spatially varying BRDFs (SVBRDFs).) It is a six-dimensional reflectance function that adds two dimensions to account for spatial variation to the BSDF. pbrt’s material model can thus be seen as imposing a particular factorization of the BTF where variation due to the spatial dimension is incorporated into textures that in turn provide values for a parametric BSDF that defines the directional distribution. The BTF representation is especially useful for material acquisition, as it does not impose a particular representation or specific factorization of the six dimensions. The survey articles on BTF acquisition and representation by Müller et al. (2005) and Filip and Haindl (2009) have good coverage of earlier work in this area. Rainer et al. (2019) recently trained a neural network to represent a given BTF; network evaluation took the position and lighting directions as parameters and returned the corresponding BTF value. This work was

subsequently generalized with a technique based on training a single network that provides a parameterization to which given BTFs can easily be mapped (Rainer et al. 2020). Kuznetsov et al. (2021) also used a neural approach, developing a compact representation that allowed 7D queries of position, two directions, and a filter size. EXERCISES

➋ 10.1 Read the papers by Manson and Schaefer (2013, 2014) on approximating high-quality filters with MIP maps and a small number of bilinear samples. Add an option to use their method for texture filtering in place of the EWA implementation currently in pbrt. Compare image quality for a number of scenes that use textures. How does running time compare? You may also find it beneficial to use a profiler to compare the amount of time it takes to run texture filtering code for each of the two approaches. ➋ 10.2 An additional advantage of properly antialiased image map lookups is that they improve cache performance. Consider, for example, the situation of undersampling a high-resolution image map: nearby samples on the screen will access widely separated parts of the image map, such that there is low probability that texels fetched from main memory for one texture lookup will already be in the cache for texture lookups at adjacent pixel samples. Modify pbrt so that it always does image texture lookups from the finest level of the MIPMap, being careful to ensure that the same number of texels are still being accessed. How does performance change? What do cache-profiling tools report about the overall change in effectiveness of the CPU cache? ➋ 10.3 Read Worley’s paper that describes a noise function with substantially different visual characteristics than Perlin noise (Worley 1996). Implement this cellular noise function, and add Textures to pbrt that are based on it. ➌ 10.4 Read some of the papers on filtering bump maps referenced in the “Further Reading” section of this chapter, choose one of the techniques described there, and implement it in pbrt. Show the visual artifacts from bump map aliasing without the technique you implement, as well as examples of how well your implementation addresses them. ➌ 10.5 Modify pbrt to support a shading language to allow user-written programs to compute texture values. Unless you are also interested in writing your own compiler, OSL (Gritz et al. 2010) is a good choice.

```text
     Texture 655
```

_________________ 1 Igehy’s formulation is slightly different from the one here—he effectively tracked the differences between the main ray and the offset rays, while we store the offset rays explicitly. The mathematics all work out to be the same in the end; we chose this alternative because we believe that it makes the algorithm’s operation for camera rays easier to understand. 2 One simplification that is present in this ideal filtering process is the implicit assumption that the texture function makes a linear contribution to frequency content in the image, so that filtering out its high frequencies removes high frequencies from the image. This is true for many uses of textures—for example, if an image map is used to modulate the diffuse term of a DiffuseMaterial. However, if a texture is used to determine the roughness of a glossy specular object, for example, this linearity assumption is incorrect, since a linear change in the roughness value has a nonlinear effect on the reflected radiance from the microfacet BRDF. We will ignore this issue here, since it is not easily solved in general. The “Further Reading” section has more discussion of this topic. 3 The term texture map is often used to refer to this type of texture, although this usage blurs the distinction between the mapping that computes texture coordinates and the texture function itself.

4 The name “MIP map” comes from the Latin multum in parvo, which means “much in little,” a nod to the image pyramid. 5 Heckbert’s thesis has the original derivation (Heckbert 1989b, p. 80): A and C have an extra term of 1 added to them so the ellipse is a minimum of one texel separation wide. This ensures that the ellipse will not fall between the texels when magnifying at the most detailed level.
