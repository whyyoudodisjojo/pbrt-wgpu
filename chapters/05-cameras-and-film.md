# Chapter 5: Cameras and Film

In Chapter 1, we described the pinhole camera model that is commonly used in computer graphics. This model is easy to describe and simulate, but it neglects important effects that physical lenses have on light passing through them. For example, everything rendered with a pinhole camera is in sharp focus—a state of affairs not possible with real lens systems. Such images often look computer generated for their perfection. More generally, the distribution of radiance leaving a lens system is quite different from the distribution entering it; modeling this effect of lenses is important for accurately simulating the radiometry of image formation.

Camera lens systems introduce various aberrations that affect the images that they form; for example, vignetting causes a darkening toward the edges of images due to less light making it through to the edges of the film or sensor than to the center. Lenses can also cause pincushion or barrel distortion, which causes straight lines to be imaged as curves. Although lens designers work to minimize aberrations in their designs, they can still have a meaningful effect on images.

This chapter starts with a description of the Camera interface, after which we present a few implementations, starting with ideal pinhole models. We then generalize those models to account for the effect of finite aperture using a simple approximation of a single lens. (The online edition of this book includes a more sophisticated camera implementation that simulates light passing through a collection of glass lens elements to form an image, similar to real-world cameras.) After light has been captured by a camera, it is measured by a sensor. While traditional film uses a chemical process to measure light, most modern cameras use solid-state sensors that are divided into pixels, each of which counts the number of photons that arrive over a period of time for some range of wavelengths. Accurately modeling the radiometry of how sensors measure light is an important part of simulating the process of image formation.

To that end, all of pbrt’s camera models use an instance of the Film class, which defines the basic interface for the classes that represent images captured by cameras. We describe two film implementations in this chapter, both of which use the PixelSensor class to model the spectral response of a particular image sensor, be it film or digital. The film and sensor classes are described in the final section of this chapter.

```text
    Film 244
    PixelSensor 234
```

## 5.1 CAMERA INTERFACE

The Camera class uses the usual TaggedPointer-based approach to dynamically dispatch interface method calls to the correct implementation based on the actual type of the camera. (As usual, we will not include the implementations of those methods in the book here.) Camera is defined in the file base/camera.h.

〈Camera Definition〉 ≡ class Camera : public TaggedPointer<PerspectiveCamera, OrthographicCamera, SphericalCamera, RealisticCamera> { public: 〈Camera Interface 206〉 };

The implementation of the first three Cameras follows in this chapter. RealisticCamera is described only in the online edition.

The first method that cameras must implement is GenerateRay(), which computes the ray corresponding to a given image sample. It is important that the direction component of the returned ray be normalized—many other parts of the system will depend on this behavior. If for some reason there is no valid ray for the given CameraSample, then the pstd::optional return value should be unset. The SampledWavelengths for the ray are passed as a non-const reference so that cameras can model dispersion in their lenses, in which case only a single wavelength of light is tracked by the ray and the GenerateRay() method will call SampledWavelengths::TerminateSecondary().

〈Camera Interface〉 ≡ 206

```rust
fn generate_ray(&self, sample: CameraSample, lambda: &mut SampledWavelengths) -> Option<CameraRay>;
```

The CameraSample structure that is passed to GenerateRay() holds all the sample values needed to specify a camera ray. Its pFilm member gives the point on the film to which the generated ray should carry radiance. The point on the lens the ray passes through is in pLens (for cameras that include the notion of lenses), and time gives the time at which the ray should sample the scene. If the camera itself is in motion, the time value determines what camera position to use when generating the ray.

Finally, the filterWeight member variable is an additional scale factor that is applied when the ray’s radiance is added to the image stored by the film; it accounts for the reconstruction filter used to filter image samples at each pixel. This topic is discussed in Sections 5.4.3 and 8.8.

〈CameraSample Definition〉 ≡ struct CameraSample { Point2f pFilm; Point2f pLens; Float time = 0; Float filterWeight = 1;

```rust
}
```

The CameraRay structure that is returned by GenerateRay() includes both a ray and a spectral weight associated with it. Simple camera models leave the weight at the default value of one, while more sophisticated ones like RealisticCamera return a weight that is used in modeling the radiometry of image formation. (Section 5.4.1 contains more information about how exactly this weight is computed and used in the latter case.) Camera 206 CameraRay 207 CameraSample 206 Float 23 OrthographicCamera 217 PerspectiveCamera 220 Point2f 92 RealisticCamera 206 SampledWavelengths 173 SampledWavelengths::TerminateSecondary() 174 SphericalCamera 229 TaggedPointer 1073

〈CameraRay Definition〉 ≡ struct CameraRay { Ray ray; SampledSpectrum weight = SampledSpectrum(1); };

Cameras must also provide an implementation of GenerateRayDifferential(), which computes a main ray like GenerateRay() but also computes the corresponding rays for pixels shifted one pixel in the x and y directions on the film plane. This information about how camera rays change as a function of position on the film helps give other parts of the system a notion of how much of the film area a particular camera ray’s sample represents, which is useful for antialiasing texture lookups. 〈Camera Interface〉 +≡ 206

pstd::optional<CameraRayDifferential> GenerateRayDifferential( CameraSample sample, SampledWavelengths &lambda) const;

GenerateRayDifferential() returns an instance of the CameraRayDifferential structure, which is equivalent to CameraRay, except it stores a RayDifferential.

〈CameraRayDifferential Definition〉 ≡ struct CameraRayDifferential { RayDifferential ray; SampledSpectrum weight = SampledSpectrum(1); };

Camera implementations must provide access to their Film, which allows other parts of the system to determine things such as the resolution of the output image. 〈Camera Interface〉 +≡ 206

Film GetFilm() const;

Just like real-world cameras, pbrt’s camera models include the notion of a shutter that opens for a short period of time to expose the film to light. One result of this nonzero exposure time is motion blur: objects that are in motion relative to the camera during the exposure are blurred. Time is yet another thing that is amenable to point sampling and Monte Carlo integration: given an appropriate distribution of ray times between the shutter open time and the shutter close time, it is possible to compute images that exhibit motion blur. The SampleTime() interface method should therefore map a uniform random sample u in the range [0, 1) to a time when the camera’s shutter is open. Normally, it is just used to linearly interpolate between the shutter open and close times. 〈Camera Interface〉 +≡ 206

Float SampleTime(Float u) const;

The last interface method allows camera implementations to set fields in the ImageMetadata class to specify transformation matrices related to the camera. If the output image format has support for storing this sort of

auxiliary information, it will be included in the final image that is written to disk. 〈Camera Interface〉 +≡ 206

void InitMetadata(ImageMetadata *metadata) const;

```text
    CameraRay 207
    CameraRayDifferential 207
    CameraSample 206
    Film 244
    Float 23
    ImageMetadata 1086
    Ray 95
    RayDifferential 96
    SampledSpectrum 171
    SampledWavelengths 173
```

### 5.1.1 CAMERA COORDINATE SPACES

Before we start to describe the implementation of pbrt’s camera models, we will define some of the coordinate spaces that they use. In addition to world space, which was introduced in Section 3.1, we will now introduce four additional coordinate spaces, object space, camera space, camera-world space, and rendering space.1 In sum, we have: Object space: This is the coordinate system in which geometric primitives are defined. For example, spheres in pbrt are defined to be centered at the origin of their object space. World space: While each primitive may have its own object space, all objects in the scene are placed in relation to a single world space. A world-from-object transformation determines where each object is located in world space. World space is the standard frame that all other spaces are defined in terms of. Camera space: A camera is placed in the scene at some world space point with a particular viewing direction and orientation. This camera defines a new coordinate system with its origin at the camera’s location. The z axis of this coordinate system is mapped to the viewing direction, and the y axis is mapped to the up direction.

Camera-world space: Like camera space, the origin of this coordinate system is the camera’s position, but it maintains the orientation of world space (i.e., unlike camera space, the camera is not necessarily looking down the z axis). Rendering space: This is the coordinate system into which the scene is transformed for the purposes of rendering. In pbrt, it may be world space, camera space, or camera-world space. Renderers based on rasterization traditionally do most of their computations in camera space: triangle vertices are transformed all the way from object space to camera space before being projected onto the screen and rasterized. In that context, camera space is a handy space for reasoning about which objects are potentially visible to the camera. For example, if an object’s camera space bounding box is entirely behind the z = 0 plane (and the camera does not have a field of view wider than 180 degrees), the object will not be visible. Conversely, many ray tracers (including all versions of pbrt prior to this one) render in world space. Camera implementations may start out in camera space when generating rays, but they transform those rays to world space where all subsequent ray intersection and shading calculations are performed. A problem with that approach stems from the fact that floating-point numbers have more precision close to the origin than far away from it. If the camera is placed far from the origin, there may be insufficient precision to accurately represent the part of the scene that it is looking at. Figure 5.1 illustrates the precision problem with rendering in world space. In Figure 5.1(a), the scene is rendered with the camera and objects as they were provided in the original scene specification, which happened to be in the range of ±10 in each coordinate in world space. In Figure 5.1(b), both the camera and the scene have been translated 1,000,000 units in each dimension. In principle, both images should be the same, but much less precision is available for the second viewpoint, to the extent that the discretization of floating-point numbers is visible in the geometric model. Rendering in camera space naturally provides the most floating-point precision for the objects closest to the camera. If the scene in Figure 5.1 is rendered in camera space, translating both the camera and the scene

geometry by 1,000,000 units has no effect—the translations cancel. However, there is a problem with using camera space with ray tracing. Scenes are often modeled with major features aligned to the coordinate axes (e.g., consider an architectural model, where the floor and ceiling might be aligned with y planes). Axis-aligned bounding boxes of such features are degenerate in one dimension, which reduces their surface area. Acceleration structures like the BVH that will be introduced in Chapter 7 are particularly effective with such bounding boxes. In turn, if the camera is rotated with respect to the scene, axis-aligned bounding boxes are less effective at bounding such features and rendering performance is affected: for the scene in Figure 5.1, rendering time increases by 27%.

```text
                                                            art
       Figure 5.1: Effect of the Loss of Floating-Point Precision Far from the Origin. (a) As originally
       specified, this scene is within 10 units of the origin. Rendering the scene in world space produces the
       expected image. (b) If both the scene and the camera are translated 1,000,000 units from the origin and
       the scene is rendered in world space, there is significantly less floating-point precision to represent the
       scene, giving this poor result. (c) If the translated scene is rendered in camera-world space, much more
       precision is available and the geometric detail is preserved. However, the viewpoint has shifted slightly
       due to a loss of accuracy in the representation of the camera position. (Model courtesy of Yasutoshi Mori.)
```

Rendering using camera-world space gives the best of both worlds: the camera is at the origin and the scene is translated accordingly. However, the rotation is not applied to the scene geometry, thus preserving good bounding boxes for the acceleration structures. With camera-world space, there is no increase in rendering time and higher precision is maintained, as is shown in Figure 5.1(c). The CameraTransform class abstracts the choice of which particular coordinate system is used for rendering by handling the details of transforming among the various spaces. 〈CameraTransform Definition〉 ≡ class CameraTransform { public: 〈CameraTransform Public Methods〉 private:

〈CameraTransform Private Members 210〉 };

Camera implementations must make their CameraTransform available to other parts of the system, so we will add one more method to the Camera interface. 〈Camera Interface〉 +≡ 206

const CameraTransform &GetCameraTransform() const;

CameraTransform maintains two transformations: one from camera space to the rendering space, and one from the rendering space to world space. In pbrt, the latter transformation cannot be animated; any animation in the camera transformation is kept in the first transformation. This ensures that a moving camera does not cause static geometry in the scene to become animated, which in turn would harm performance.2 〈CameraTransform Private Members〉 ≡ 210

AnimatedTransform renderFromCamera; Transform worldFromRender;

The CameraTransform constructor takes the world-from-camera transformation as specified in the scene description and decomposes it into the two transformations described earlier. The default rendering space is camera-world, though this choice can be overridden using a command-line option.

```text
   AnimatedTransform 135
   BasicPBRTOptions::renderingSpace 1031
   Camera 206
   CameraTransform 210
   Options 1032
   RenderingCoordinateSystem 1032
   Transform 120
```

〈CameraTransform Method Definitions〉 ≡ CameraTransform::CameraTransform(const AnimatedTransform &worldFromCamera) { switch (Options->renderingSpace) {

```rust
RenderingCoordinateSystem::Camera => {
    // Compute worldFromRender for camera-space rendering
}
RenderingCoordinateSystem::CameraWorld => {
    // Compute worldFromRender for camera-world space rendering
}
```

} case RenderingCoordinateSystem::World: { 〈Compute worldFromRender for world-space rendering 211〉 } } 〈Compute renderFromCamera transformation 211〉 }

For camera-space rendering, the world-from-camera transformation should be used for worldFromRender and an identity transformation for the render-from-camera transformation, since those two coordinate systems are equivalent. However, because worldFromRender cannot be animated, the implementation takes the world-from-camera transformation at the midpoint of the frame and then folds the effect of any animation in the camera transformation into renderFromCamera. 〈Compute worldFromRender for camera-space rendering〉 ≡ 210

Float tMid = (worldFromCamera.startTime + worldFromCamera.endTime) / 2; worldFromRender = worldFromCamera.Interpolate(tMid); break;

For the default case of rendering in camera-world space, the world-from-render transformation is given by translating to the camera’s position at the midpoint of the frame. 〈Compute worldFromRender for camera-world space rendering〉 ≡ 210

Float tMid = (worldFromCamera.startTime + worldFromCamera.endTime) / 2; Point3f pCamera = worldFromCamera(Point3f(0, 0, 0), tMid); worldFromRender = Translate(Vector3f(pCamera)); break;

For world-space rendering, worldFromRender is the identity transformation. 〈Compute worldFromRender for world-space rendering〉 ≡ 210

worldFromRender = Transform(); break;

Once worldFromRender has been set, whatever transformation remains in worldFromCamera is extracted and stored in renderFromCamera.

〈Compute renderFromCamera transformation〉 ≡ 210

Transform renderFromWorld = Inverse(worldFromRender); Transform rfc[2] = { renderFromWorld * worldFromCamera.startTransform, renderFromWorld * worldFromCamera.endTransform }; renderFromCamera = AnimatedTransform(rfc[0], worldFromCamera.startTime, rfc[1], worldFromCamera.endTime);

The CameraTransform class provides a variety of overloaded methods named RenderFrom Camera(), CameraFromRender(), and RenderFromWorld() that transform points, vectors, normals, and rays among the coordinate systems it manages. Other methods return the corresponding transformations directly. Their straightforward implementations are not included here.

```text
    AnimatedTransform 135
    AnimatedTransform::Interpolate() 135
    CameraTransform 210
    CameraTransform::renderFromCamera 210
    CameraTransform::worldFromRender 210
    Float 23
    Point3f 92
    Transform 120
    Transform::Inverse() 121
    Transform::Transform() 120
    Translate() 123
    Vector3f 86
```

### 5.1.2 THE CameraBase CLASS

All of the camera implementations in this chapter share some common functionality that we have factored into a single class, CameraBase, from which all of them inherit.3 CameraBase, as well as all the camera implementations, is defined in the files cameras.h and cameras.cpp.

〈CameraBase Definition〉 ≡ class CameraBase { public: 〈CameraBase Public Methods 213〉 protected: 〈CameraBase Protected Members 212〉 〈CameraBase Protected Methods 212〉 };

The CameraBase constructor takes a variety of parameters that are applicable to all of pbrt’s cameras:

One of the most important is the transformation that places the camera in the scene, which is represented by a CameraTransform and is stored in the cameraTransform member variable. Next is a pair of floating-point values that give the times at which the camera’s shutter opens and closes. A Film instance stores the final image and models the film sensor. Last is a Medium instance that represents the scattering medium that the camera lies in, if any (Medium is described in Section 11.4). A small structure bundles them together and helps shorten the length of the parameter lists for Camera constructors.

〈CameraBaseParameters Definition〉 ≡ struct CameraBaseParameters { CameraTransform cameraTransform; Float shutterOpen = 0, shutterClose = 1; Film film; Medium medium;

};

We will only include the constructor’s prototype here because its implementation does no more than assign the parameters to the corresponding member variables. 〈CameraBase Protected Methods〉 ≡ 212

CameraBase(CameraBaseParameters p); 〈CameraBase Protected Members〉 ≡ 212

CameraTransform cameraTransform; Float shutterOpen, shutterClose; Film film; Medium medium;

```text
   CameraBase 212
   CameraBaseParameters 212
   CameraTransform 210
   Film 244
   Float 23
   Medium 714
```

CameraBase can implement a number of the methods required by the Camera interface directly, thus saving the trouble of needing to redundantly implement them in the camera implementations that inherit from it. For example, accessor methods make the Film and CameraTransform available. 〈CameraBase Public Methods〉 ≡ 212

Film GetFilm() const { return film; } const CameraTransform &GetCameraTransform() const { return cameraTransform; }

The SampleTime() method is implemented by linearly interpolating between the shutter open and close times using the sample u. 〈CameraBase Public Methods〉 +≡ 212

Float SampleTime(Float u) const { return Lerp(u, shutterOpen, shutterClose); }

CameraBase provides a GenerateRayDifferential() method that computes a ray differential via multiple calls to a camera’s GenerateRay() method. One subtlety is that camera implementations that use this method still must implement a Camera GenerateRayDifferential() method themselves, but then call this method from theirs. (Note that this method’s signature is different than that one.) Cameras pass their this pointer as a Camera parameter, which allows it to call the camera’s GenerateRay() method. This additional complexity stems from our not using virtual functions for the camera interface, which means that the CameraBase class does not on its own have the ability to call that method unless a Camera is provided to it. 〈CameraBase Method Definitions〉 ≡ pstd::optional<CameraRayDifferential> CameraBase::GenerateRayDifferential(Camera camera, CameraSample sample, SampledWavelengths &lambda) { 〈Generate regular camera ray cr for ray differential 213〉 〈Find camera ray after shifting one pixel in the x direction 214〉 〈Find camera ray after shifting one pixel in the y direction〉 〈Return approximate ray differential and weight 214〉 }

The primary ray is found via a first call to GenerateRay(). If there is no valid ray for the given sample, then there can be no ray differential either. 〈Generate regular camera ray cr for ray differential〉 ≡ 213

pstd::optional<CameraRay> cr = camera.GenerateRay(sample, lambda); if (!cr) return {}; RayDifferential rd(cr->ray);

Two attempts are made to find the x ray differential: one using forward differencing and one using backward differencing by a fraction of a pixel. It is important to try both of these due to vignetting at the edges of images formed by realistic camera models—sometimes the main ray is valid but shifting in one direction moves past the image formed by the lens system. In that case, trying the other direction may successfully generate a ray.

```text
   Camera 206
   Camera::GenerateRay() 206
   CameraBase 212
   CameraBase::cameraTransform 212
   CameraBase::film 212
   CameraBase::shutterClose 212
   CameraBase::shutterOpen 212
   CameraRay 207
   CameraRay::ray 207
   CameraRayDifferential 207
   CameraSample 206
   CameraTransform 210
   Film 244
   Float 23
   Lerp() 72
   RayDifferential 96
   SampledWavelengths 173
```

〈Find camera ray after shifting one pixel in the x direction〉 ≡ 213

pstd::optional<CameraRay> rx; for (Float eps : {.05f, -.05f}) { CameraSample sshift = sample; sshift.pFilm.x += eps; 〈Try to generate ray with sshift and compute x differential 214〉 }

If it was possible to generate the auxiliary x ray, then the corresponding pixel-wide differential is initialized via differencing. 〈Try to generate ray with sshift and compute x differential〉 ≡ 214

if (rx = camera.GenerateRay(sshift, lambda); rx) { rd.rxOrigin = rd.o + (rx->ray.o - rd.o) / eps; rd.rxDirection = rd.d + (rx->ray.d - rd.d) / eps; break; }

The implementation of the fragment 〈Find camera ray after shifting one pixel in the y direction〉 follows similarly and is not included here. If a valid ray was found for both x and y, we can go ahead and set the hasDifferentials member variable to true. Otherwise, the main ray can still be traced, just without differentials available. 〈Return approximate ray differential and weight〉 ≡ 213

```rust
rd.has_differentials = rx.is_some() && ry.is_some();
return CameraRayDifferential { ray: rd, weight: cr.weight };
```

Finally, for the convenience of its subclasses, CameraBase provides various transformation methods that use the CameraTransform. We will only include the Ray method here; the others are analogous.

〈CameraBase Protected Methods〉 +≡ 212

Ray RenderFromCamera(const Ray &r) const { return cameraTransform.RenderFromCamera(r); }

## 5.2 PROJECTIVE CAMERA MODELS

One of the fundamental issues in 3D computer graphics is the 3D viewing problem: how to project a 3D scene onto a 2D image for display. Most of the classic approaches can be expressed by a 4 × 4 projective transformation matrix. Therefore, we will introduce a projection matrix camera class, ProjectiveCamera, and then define two camera models based on it. The first implements an orthographic projection, and the other implements a perspective projection—two classic and widely used projections. 〈ProjectiveCamera Definition〉 ≡ class ProjectiveCamera : public CameraBase { public: 〈ProjectiveCamera Public Methods 216〉 protected: 〈ProjectiveCamera Protected Members 216〉 };

```text
   Camera::GenerateRay() 206
   CameraBase 212
   CameraBase::cameraTransform 212
   CameraRay 207
   CameraRay::ray 207
   CameraRay::weight 207
   CameraRayDifferential 207
   CameraSample 206
   CameraSample::pFilm 206
```

```text
   CameraTransform 210
   CameraTransform::RenderFromCamera() 211
   Float 23
   Ray 95
   Ray::d 95
   Ray::o 95
   RayDifferential::hasDifferentials 96
   RayDifferential::rxDirection 96
   RayDifferential::rxOrigin 96
```

```text
                                                            art
        Figure 5.2: Several camera-related coordinate spaces are commonly used to simplify the implementation
        of Cameras. The camera class holds transformations between them. Scene objects in rendering space are
        viewed by the camera, which sits at the origin of camera space and points along the +z axis. Objects
        between the near and far planes are projected onto the film plane at z = near in camera space. The film
        plane is at z = 0 in raster space, where x and y range from (0, 0) to the image resolution in pixels.
        Normalized device coordinate (NDC) space normalizes raster space so that x and y range from (0, 0) to (1,
        1).
```

The orthographic and perspective projections both require the specification of two planes perpendicular to the viewing direction: the near and far planes. When rasterization is used for rendering, objects that are not between those two planes are culled and not included in the final image. (Culling objects in front of the near plane is particularly important in order to avoid a singularity at the depth 0 and because otherwise the projection matrices map points behind the camera to appear to be in front of it.) In a ray tracer, the projection matrices are used purely to determine rays leaving the camera and these concerns do not apply; there is therefore less need to worry about setting those planes’ depths carefully in this context. Three more coordinate systems (summarized in Figure 5.2) are useful for defining and discussing projective cameras: Screen space: Screen space is defined on the film plane. The camera projects objects in camera space onto the film plane; the parts inside the screen window are visible in the image that is generated. Points at the near plane are mapped to a depth z value of 0 and points at the far plane are mapped to 1. Note that,

although this is called “screen” space, it is still a 3D coordinate system, since z values are meaningful. Normalized device coordinate (NDC) space: This is the coordinate system for the actual image being rendered. In x and y, this space ranges from (0, 0) to (1, 1), with (0, 0) being the upper-left corner of the image. Depth values are the same as in screen space, and a linear transformation converts from screen to NDC space. Raster space: This is almost the same as NDC space, except the x and y coordinates range from (0, 0) to the resolution of the image in x and y pixels. Projective cameras use 4 × 4 matrices to transform among all of these spaces.

```text
   ProjectiveCamera 214
```

In addition to the parameters required by the CameraBase class, the ProjectiveCamera takes the projective transformation matrix, the screen space extent of the image, and additional parameters related to the distance at which the camera is focused and the size of its lens aperture. If the lens aperture is not an infinitesimal pinhole, then parts of the image may be blurred, as happens for out-of-focus objects with real lens systems. Simulation of this effect will be discussed later in this section. 〈ProjectiveCamera Public Methods〉 ≡ 214

ProjectiveCamera(CameraBaseParameters baseParameters, const Transform &screenFromCamera, Bounds2f screenWindow, Float lensRadius, Float focalDistance) : CameraBase(baseParameters), screenFromCamera(screenFromCamera), lensRadius(lensRadius), focalDistance(focalDistance) { 〈Compute projective camera transformations 216〉 }

ProjectiveCamera implementations pass the projective transformation up to the base class constructor shown here. This transformation gives the screen-from-camera projection; from that, the constructor can easily

compute the other transformations that go all the way from raster space to camera space. 〈Compute projective camera transformations〉 ≡ 216

〈Compute projective camera screen transformations 216〉 cameraFromRaster = Inverse(screenFromCamera) * screenFromRaster; 〈ProjectiveCamera Protected Members〉 ≡ 214

Transform screenFromCamera, cameraFromRaster;

The only nontrivial transformation to compute in the constructor is the raster-from-screen projection. It is computed in two steps, via composition of the raster-from-NDC and NDC-from-screen transformations. An important detail here is that the y coordinate is inverted by the final transformation; this is necessary because increasing y values move up the image in screen coordinates but down in raster coordinates. 〈Compute projective camera screen transformations〉 ≡ 216

Transform NDCFromScreen = Scale(1 / (screenWindow.pMax.x - screenWindow.pMin.x), 1 / (screenWindow.pMax.y - screenWindow.pMin.y), 1) * Translate(Vector3f(-screenWindow.pMin.x, -screenWindow.pMax.y, 0)); Transform rasterFromNDC = Scale(film.FullResolution().x, -film.FullResolution().y, 1); rasterFromScreen = rasterFromNDC * NDCFromScreen; screenFromRaster = Inverse(rasterFromScreen); 〈ProjectiveCamera Protected Members〉 +≡ 214

Transform rasterFromScreen, screenFromRaster;

```text
   Bounds2::pMax 98
   Bounds2::pMin 98
   Bounds2f 97
   CameraBase 212
   CameraBase::film 212
   CameraBaseParameters 212
   Film::FullResolution() 246
   Float 23
   ProjectiveCamera 214
   ProjectiveCamera::cameraFromRaster 216
   ProjectiveCamera::focalDistance 226
```

```text
    ProjectiveCamera::lensRadius 226
    ProjectiveCamera::rasterFromScreen 216
    ProjectiveCamera::screenFromCamera 216
    ProjectiveCamera::screenFromRaster 216
    Scale() 123
    Transform 120
    Transform::Inverse() 121
    Translate() 123
    Vector3f 86
```

### 5.2.1 ORTHOGRAPHIC CAMERA

The orthographic camera is based on the orthographic projection transformation. The orthographic transformation takes a rectangular region of the scene and projects it onto the front face of the box that defines the region. It does not give the effect of foreshortening—objects becoming smaller on the image plane as they get farther away—but it does leave parallel lines parallel, and it preserves relative distance between objects. Figure 5.3 shows how this rectangular volume defines the visible region of the scene.

```text
                                                               art
         Figure 5.3: The orthographic view volume is an axis-aligned box in camera space, defined such that
         objects inside the region are projected onto the z = near face of the box.
```

```text
                                                               art
         Figure 5.4: Kroken Scene Rendered with Different Camera Models. Images are rendered from the
         same viewpoint with (a) orthographic and (b) perspective cameras. The lack of foreshortening makes the
         orthographic view feel like it has less depth, although it does preserve parallel lines, which can be a useful
         property. (Scene courtesy of Angelo Ferretti.)
```

〈OrthographicCamera Definition〉 ≡ class OrthographicCamera : public ProjectiveCamera { public:

〈OrthographicCamera Public Methods 218〉 private: 〈OrthographicCamera Private Members 218〉 };

*Figure 5.4 compares the result of using the orthographic projection for rendering to that of the perspective projection defined in the next section.*

```text
   ProjectiveCamera 214
```

The orthographic camera constructor generates the orthographic transformation matrix with the Orthographic() function, which will be defined shortly. 〈OrthographicCamera Public Methods〉 ≡ 217

OrthographicCamera(CameraBaseParameters baseParameters, Bounds2f screenWindow, Float lensRadius, Float focalDist) : ProjectiveCamera(baseParameters, Orthographic(0, 1), screenWindow, lensRadius, focalDist) { 〈Compute differential changes in origin for orthographic camera rays 218〉

〈Compute minimum differentials for orthographic camera 640〉 }

The orthographic viewing transformation leaves x and y coordinates unchanged but maps z values at the near plane to 0 and z values at the far plane to 1. To do this, the scene is first translated along the z axis so that the near plane is aligned with z = 0. Then, the scene is scaled in z so that the far plane maps to z = 1. The composition of these two transformations gives the overall transformation. For a ray tracer like pbrt, we would like the near plane to be at 0 so that rays start at the plane that goes through the camera’s position; the far plane’s position does not particularly matter. 〈Transform Function Definitions〉 +≡ Transform Orthographic(Float zNear, Float zFar) { return Scale(1, 1, 1 / (zFar - zNear)) * Translate(Vector3f(0, 0, -zNear));

}

Thanks to the simplicity of the orthographic projection, it is easy to directly compute the differential rays in the x and y directions in the GenerateRayDifferential() method. The directions of the differential rays will be the same as the main ray (as they are for all rays generated by an orthographic camera), and the difference in origins will be the same for all rays. Therefore, the constructor here precomputes how much the ray origins shift in camera space coordinates due to a single pixel shift in the x and y directions on the film plane. 〈Compute differential changes in origin for orthographic camera rays〉 ≡ 218

dxCamera = cameraFromRaster(Vector3f(1, 0, 0)); dyCamera = cameraFromRaster(Vector3f(0, 1, 0)); 〈OrthographicCamera Private Members〉 ≡ 217

Vector3f dxCamera, dyCamera;

We can now go through the code that takes a sample point in raster space and turns it into a camera ray. The process is summarized in Figure 5.5. First, the raster space sample position is transformed into a point in camera space, giving a point located on the near plane, which is the origin of the camera ray. Because the camera space viewing direction points down the z axis, the camera space ray direction is (0, 0, 1). If the lens aperture is not a pinhole, the ray’s origin and direction are modified so that defocus blur is simulated. Finally, the ray is transformed into rendering space before being returned. 〈OrthographicCamera Method Definitions〉 ≡ pstd::optional<CameraRay> OrthographicCamera::GenerateRay( CameraSample sample, SampledWavelengths &lambda) const { 〈Compute raster and camera sample positions 219〉 Ray ray(pCamera, Vector3f(0, 0, 1), SampleTime(sample.time), medium); 〈Modify ray for depth of field 226〉 return CameraRay{RenderFromCamera(ray)}; }

```text
   Bounds2f 97
   CameraBase::RenderFromCamera() 214
   CameraBase::SampleTime() 213
   CameraBaseParameters 212
   CameraRay 207
   CameraSample 206
   CameraSample::time 206
   Float 23
   Orthographic() 218
   OrthographicCamera 217
   OrthographicCamera::dxCamera 218
   OrthographicCamera::dyCamera 218
   ProjectiveCamera 214
   ProjectiveCamera::cameraFromRaster 216
   Ray 95
   SampledWavelengths 173
   Scale() 123
   Transform 120
   Translate() 123
   Vector3f 86
```

```text
                                                             art
        Figure 5.5: To create a ray with the orthographic camera, a raster space position on the film plane is
        transformed to camera space, giving the ray’s origin on the near plane. The ray’s direction in camera
        space is (0, 0, 1), down the z axis.
```

Once all the transformation matrices have been set up, it is easy to transform the raster space sample point to camera space. 〈Compute raster and camera sample positions〉 ≡ 218, 222

Point3f pFilm = Point3f(sample.pFilm.x, sample.pFilm.y, 0); Point3f pCamera = cameraFromRaster(pFilm);

The implementation of GenerateRayDifferential() performs the same computation to generate the main camera ray. The differential ray origins are found using the offsets computed in the OrthographicCamera constructor, and then the full ray differential is transformed to rendering space.

〈OrthographicCamera Method Definitions〉 +≡ pstd::optional<CameraRayDifferential> OrthographicCamera::GenerateRayDifferential(CameraSample sample, SampledWavelengths &lambda) const { 〈Compute main orthographic viewing ray〉 〈Compute ray differentials for OrthographicCamera 219〉 ray.hasDifferentials = true; return CameraRayDifferential{RenderFromCamera(ray)}; } 〈Compute ray differentials for OrthographicCamera〉 ≡ 219

if (lensRadius > 0) { 〈Compute OrthographicCamera ray differentials accounting for lens〉 } else { ray.rxOrigin = ray.o + dxCamera; ray.ryOrigin = ray.o + dyCamera; ray.rxDirection = ray.ryDirection = ray.d; }

### 5.2.2 PERSPECTIVE CAMERA

The perspective projection is similar to the orthographic projection in that it projects a volume of space onto a 2D film plane. However, it includes the effect of foreshortening: objects that are far away are projected to be smaller than objects of the same size that are closer. Unlike the orthographic projection, the perspective projection does not preserve distances or angles, and parallel lines no longer remain parallel. The perspective projection is a reasonably close match to how an eye or camera lens generates images of the 3D world.

```text
    CameraBase::RenderFromCamera() 214
    CameraRayDifferential 207
    CameraSample 206
    CameraSample::pFilm 206
    OrthographicCamera 217
    OrthographicCamera::dxCamera 218
    OrthographicCamera::dyCamera 218
    Point3f 92
    ProjectiveCamera::cameraFromRaster 216
    ProjectiveCamera::lensRadius 226
    Ray::d 95
    Ray::o 95
```

```text
   RayDifferential::hasDifferentials 96
   RayDifferential::rxDirection 96
   RayDifferential::rxOrigin 96
   RayDifferential::ryDirection 96
   RayDifferential::ryOrigin 96
   SampledWavelengths 173
```

〈PerspectiveCamera Definition〉 ≡ class PerspectiveCamera : public ProjectiveCamera { public: 〈PerspectiveCamera Public Methods 220〉 private: 〈PerspectiveCamera Private Members 221〉 }; 〈PerspectiveCamera Public Methods〉 ≡ 220

PerspectiveCamera(CameraBaseParameters baseParameters, Float fov, Bounds2f screenWindow, Float lensRadius, Float focalDist) : ProjectiveCamera(baseParameters, Perspective(fov, 1e-2f, 1000.f), screenWindow, lensRadius, focalDist) { 〈Compute differential changes in origin for perspective camera rays 221〉

〈Compute cosTotalWidth for perspective camera 222〉 〈Compute image plane area at z = 1 for PerspectiveCamera〉 〈Compute minimum differentials for PerspectiveCamera 640〉 }

The perspective projection describes perspective viewing of the scene. Points in the scene are projected onto a viewing plane perpendicular to the z axis. The Perspective() function computes this transformation; it takes a field-of-view angle in fov and the distances to a near z plane and a far z plane (Figure 5.6). 〈Transform Function Definitions〉 +≡ Transform Perspective(Float fov, Float n, Float f) { 〈Perform projective divide for perspective projection 221〉 〈Scale canonical perspective view to specified field of view 221〉 }

The transformation is most easily understood in two steps: 1. Points p in camera space are projected onto the viewing plane. A bit of algebra shows that the projected x′ and y′ coordinates on the viewing plane can be computed by dividing x and y by the point’s z coordinate value. The projected z depth is remapped so that z values at the near plane are 0 and z values at the far plane are 1. The computation we would like to do is

```text
                                                              art
                  Figure 5.6: The perspective transformation matrix projects points in camera space onto
                  the near plane. The x′ and y′ coordinates of the projected points are equal to the
                  unprojected x and y coordinates divided by the z coordinate. That operation is depicted
                  here, where the effect of the projection is indicated by an arrow. The projected z′
                  coordinate is then computed so that points on the near plane map to z′ = 0 and points on
                  the far plane map to z′ = 1.
```

```text
             Bounds2f 97
             CameraBaseParameters 212
             Float 23
             Perspective() 220
             PerspectiveCamera 220
             ProjectiveCamera 214
             Transform 120
```

```text
                                                              art
         All of this computation can be encoded in a 4 × 4 matrix that can
         then be applied to homogeneous coordinates:
                                                              art
          〈Perform projective divide for perspective projection〉 ≡                                           220
```

```rust
let persp = SquareMatrix::<4>::from_array([
    1.0, 0.0, 0.0, 0.0,
    0.0, 1.0, 0.0, 0.0,
    0.0, 0.0, f / (f - n), -f * n / (f - n),
    0.0, 0.0, 1.0, 0.0,
]);
```

```text
       2. The angular field of view (fov) specified by the user is accounted
          for by scaling the (x, y) values on the projection plane so that
          points inside the field of view project to coordinates between [−1,
          1] on the view plane. For square images, both x and y lie between
```

```text
              [−1, 1] in screen space. Otherwise, the direction in which the
              image is narrower maps to [−1, 1], and the wider direction maps
              to a proportionally larger range of screen space values. Recall that
              the tangent is equal to the ratio of the opposite side of a right
              triangle to the adjacent side. Here the adjacent side has length 1,
              so the opposite side has the length tan(fov/2). Scaling by the
              reciprocal of this length maps the field of view to the range [−1,
              1].
                〈Scale canonical perspective view to specified field of view〉 ≡   220
```

```rust
let inv_tan_ang = 1.0 / (Radians(fov) / 2.0).tan();
Scale(inv_tan_ang, inv_tan_ang, 1.0) * Transform(persp);
```

As with the OrthographicCamera, the PerspectiveCamera’s constructor computes information about how the rays it generates change with shifts in pixels. In this case, the ray origins are unchanged and the ray differentials are only different in their directions. Here, we compute the change in position on the near perspective plane in camera space with respect to shifts in pixel location. 〈Compute differential changes in origin for perspective camera rays〉 ≡ 220

dxCamera = cameraFromRaster(Point3f(1, 0, 0)) -cameraFromRaster(Point3f(0, 0, 0)); dyCamera = cameraFromRaster(Point3f(0, 1, 0)) -cameraFromRaster(Point3f(0, 0, 0)); 〈PerspectiveCamera Private Members〉 ≡ 220

Vector3f dxCamera, dyCamera;

```text
   Float 23
   OrthographicCamera 217
   PerspectiveCamera::dxCamera 221
   PerspectiveCamera::dyCamera 221
   Point3f 92
   ProjectiveCamera::cameraFromRaster 216
   Scale() 123
   SquareMatrix 1049
   Transform 120
   Vector3f 86
```

The cosine of the maximum angle of the perspective camera’s field of view will occasionally be useful. In particular, points outside the field of view can be quickly culled via a dot product with the viewing direction and comparison to this value. This cosine can be found by computing the angle between the camera’s viewing vector and a vector to one of the corners of the image (see Figure 5.7). This corner needs a small adjustment here to account for the width of the filter function centered at each pixel that is used to weight image samples according to their location (this topic is discussed in Section 8.8).

```text
                                                            art
       Figure 5.7: Computing the Cosine of the Perspective Camera’s Maximum View Angle. A cone that
       bounds the viewing directions of a PerspectiveCamera can be found by using the camera’s viewing
       direction as the center axis and by computing the cosine of the angle θ between that axis and a vector to
       one of the corners of the image. In camera space, that simplifies to be the z component of that vector,
       normalized.
```

〈Compute cosTotalWidth for perspective camera〉 ≡ 220

Point2f radius = Point2f(film.GetFilter().Radius()); Point3f pCorner(-radius.x, -radius.y, 0.f); Vector3f wCornerCamera = Normalize(Vector3f(cameraFromRaster(pCorner))); cosTotalWidth = wCornerCamera.z; 〈PerspectiveCamera Private Members〉 +≡ 220

Float cosTotalWidth;

With the perspective projection, camera space rays all originate from the origin, (0, 0, 0). A ray’s direction is given by the vector from the origin to the point on the near plane, pCamera, that corresponds to the provided CameraSample’s pFilm location. In other words, the ray’s vector direction is component-wise equal to this point’s position, so rather than doing a useless subtraction to compute the direction, we just initialize the direction directly from the point pCamera. 〈PerspectiveCamera Method Definitions〉 ≡ pstd::optional<CameraRay> PerspectiveCamera::GenerateRay(

CameraSample sample, SampledWavelengths &lambda) const { 〈Compute raster and camera sample positions 219〉 Ray ray(Point3f(0, 0, 0), Normalize(Vector3f(pCamera)), SampleTime(sample.time), medium); 〈Modify ray for depth of field 226〉 return CameraRay{RenderFromCamera(ray)}; }

The GenerateRayDifferential() method follows the implementation of GenerateRay(), except for this additional fragment that computes the differential rays. 〈Compute offset rays for PerspectiveCamera ray differentials〉 ≡ if (lensRadius > 0) { 〈Compute PerspectiveCamera ray differentials accounting for lens〉 } else { ray.rxOrigin = ray.ryOrigin = ray.o; ray.rxDirection = Normalize(Vector3f(pCamera) + dxCamera); ray.ryDirection = Normalize(Vector3f(pCamera) + dyCamera); }

```text
   CameraBase::film 212
   CameraBase::RenderFromCamera() 214
   CameraBase::SampleTime() 213
   CameraRay 207
   CameraSample 206
   CameraSample::time 206
   Film::GetFilter() 246
   Filter::Radius() 515
   Float 23
   Normalize() 88
   PerspectiveCamera 220
   PerspectiveCamera::cosTotalWidth 222
   PerspectiveCamera::dxCamera 221
```

```text
    PerspectiveCamera::dyCamera 221
    Point2f 92
    Point3f 92
    ProjectiveCamera::cameraFromRaster 216
    ProjectiveCamera::lensRadius 226
    Ray 95
    Ray::o 95
    RayDifferential::rxDirection 96
    RayDifferential::rxOrigin 96
    RayDifferential::ryDirection 96
    RayDifferential::ryOrigin 96
    SampledWavelengths 173
    Vector3f 86
```

### 5.2.3 THE THIN LENS MODEL AND DEPTH OF FIELD

An ideal pinhole camera that only allows rays passing through a single point to reach the film is not physically realizable; while it is possible to make cameras with extremely small apertures that approach this behavior, small apertures allow relatively little light to reach the film sensor. With a small aperture, long exposure times are required to capture enough photons to accurately capture the image, which in turn can lead to blur from objects in the scene moving while the camera shutter is open. Real cameras have lens systems that focus light through a finite-sized aperture onto the film plane. Camera designers (and photographers using cameras with adjustable apertures) face a trade-off: the larger the aperture, the more light reaches the film and the shorter the exposures that are needed. However, lenses can only focus on a single plane (the focal plane), and the farther objects in the scene are from this plane, the blurrier they are. The larger the aperture, the more pronounced this effect is. The RealisticCamera (included only in the online edition of the book) implements a fairly accurate simulation of lens systems in real-world cameras. For the simple camera models introduced so far, we can apply a classic approximation from optics, the thin lens approximation, to model the effect of finite apertures with traditional computer graphics projection models. The thin lens approximation models an optical system as a single lens with spherical profiles, where the thickness of the lens is small relative to the radius of curvature of the lens.

Under the thin lens approximation, incident rays that are parallel to the optical axis and pass through the lens focus at a point behind the lens called the focal point. The distance the focal point is behind the lens, f, is the lens’s focal length. If the film plane is placed at a distance equal to the focal length behind the lens, then objects infinitely far away will be in focus, as they image to a single point on the film. Figure 5.8 illustrates the basic setting. Here we have followed the typical lens coordinate system convention of placing the lens perpendicular to the z axis, with the lens at z = 0 and the scene along −z. (Note that this is a different coordinate system from the one we used for camera space, where the viewing direction is +z.) Distances on the scene side of the lens are denoted with unprimed variables z, and distances on the film side of the lens (positive z) are primed, z′.

```text
                                                                art
        Figure 5.8: A thin lens, located along the z axis at z = 0. Incident rays that are parallel to the optical axis
        and pass through a thin lens (dashed lines) all pass through a point p, the focal point. The distance
        between the lens and the focal point, f, is the lens’s focal length.
```

```text
   RealisticCamera 206
```

```text
                                                                art
        Figure 5.9: To focus a thin lens at a depth z in the scene, Equation (5.2) can be used to compute the
        distance z′ on the film side of the lens that points at z focus to. Focusing is performed by adjusting the
        distance between the lens and the film plane.
```

For points in the scene at a depth z from a thin lens with focal length f, the Gaussian lens equation relates the distances from the object to the lens and from the lens to the image of the point: art Note that for z = −∞, we have z′ = f, as expected.

We can use the Gaussian lens equation to solve for the distance between the lens and the film that sets the plane of focus at some z, the focal distance (Figure 5.9): art A point that does not lie on the plane of focus is imaged to a disk on the film plane, rather than to a single point. The boundary of this disk is called the circle of confusion. The size of the circle of confusion is affected by the diameter of the aperture that light rays pass through, the focal distance, and the distance between the object and the lens. Although the circle of confusion only has zero radius for a single depth, a range of nearby depths have small enough circles of confusion that they still appear to be in focus. (As long as its circle of confusion is smaller than the spacing between pixels, a point will effectively appear to be in focus.) The range of depths that appear in focus are termed the depth of field. Figure 5.10 shows this effect, in the Watercolor scene. As the size of the lens aperture increases, blurriness increases the farther a point is from the plane of focus. Note that the pencil cup in the center remains in focus throughout all the images, as the plane of focus has been placed at its depth. Figure 5.11 shows depth of field used to render the landscape scene. Note how the effect draws the viewer’s eye to the in-focus grass in the center of the image. The Gaussian lens equation also lets us compute the size of the circle of confusion; given a lens with focal length f that is focused at a distance zf, the film plane is at art. Given another point at depth z, the Gaussian lens equation gives the distance z′ that the lens focuses the point to. This point is either in front of or behind the film plane; Figure 5.12(a) shows the case where it is behind. The diameter of the circle of confusion is given by the intersection of the cone between z′ and the lens with the film plane. If we know the diameter of the lens dl, then we can use similar triangles to solve for the diameter of the circle of confusion dc (Figure 5.12(b)): art

```text
                                                            art
      Figure 5.10: (a) Scene rendered with no defocus blur, (b) extensive depth of field due to a relatively small
      lens aperture, which gives only a small amount of blurriness in the out-of-focus regions, and (c) a very
      large aperture, giving a larger circle of confusion in the out-of-focus areas, resulting in a greater amount
      of blur on the film plane. (Scene courtesy of Angelo Ferretti.)
```

```text
                                                            art
      Figure 5.11: Depth of field gives a greater sense of depth and scale to this part of the landscape scene.
      (Scene courtesy of Laubwerk.)
```

```text
                                                            art
      Figure 5.12: (a) If a thin lens with focal length f is focused at some depth zf, then the distance from the
      lens to the focus plane is    art, given by the Gaussian lens equation. A point in the scene at depth z ≠ zf
      will be imaged as a circle on the film plane; here z focuses at z′, which is behind the film plane. (b) To
      compute the diameter of the circle of confusion, we can apply similar triangles: the ratio of dl, the
      diameter of the lens, to z′ must be the same as the ratio of dc, the diameter of the circle of confusion, to
         art.
```

```text
                                                            art
      Figure 5.13: The diameter of the circle of confusion as a function of depth for a 50-mm focal length lens
      with 25-mm aperture, focused at 1 meter.
```

Solving for dc, we have

Applying the Gaussian lens equation to express the result in terms of scene depths, we can find that art Note that the diameter of the circle of confusion is proportional to the diameter of the lens. The lens diameter is often expressed as the lens’s f-number n, which expresses diameter as a fraction of focal length, dl = f/n.

*Figure 5.13 shows a graph of this function for a 50-mm focal length lens with a 25-mm aperture, focused at zf = 1 m. Note that the blur is asymmetric with depth around the focal plane and grows much more quickly for objects in front of the plane of focus than for objects behind it. Modeling a thin lens in a ray tracer is remarkably straightforward: all that is necessary is to choose a point on the lens and find the appropriate ray that starts on the lens at that point such that objects in the plane of focus are in focus on the film (Figure 5.14). Therefore, projective cameras take two extra parameters for depth of field: one sets the size of the lens aperture, and the other sets the focal distance. 〈ProjectiveCamera Protected Members〉 +≡ 214*

Float lensRadius, focalDistance;

It is generally necessary to trace many rays for each image pixel in order to adequately sample the lens for smooth defocus blur. Figure 5.15 shows the landscape scene from Figure 5.11 with only four samples per pixel (Figure 5.11 had 2048 samples per pixel). 〈Modify ray for depth of field〉 ≡ 218, 222

if (lensRadius > 0) { 〈Sample point on lens 228〉 〈Compute point on plane of focus 228〉 〈Update ray for effect of lens 228〉 }

```text
   Float 23
   ProjectiveCamera::lensRadius 226
```

```text
                                                             art
       Figure 5.14: (a) For a pinhole camera model, a single camera ray is associated with each point on the film
       plane (filled circle), given by the ray that passes through the single point of the pinhole lens (empty
       circle). (b) For a camera model with a finite aperture, we sample a point (filled circle) on the disk-shaped
       lens for each ray. We then compute the ray that passes through the center of the lens (corresponding to the
       pinhole model) and the point where it intersects the plane of focus (solid line). We know that all objects in
       the plane of focus must be in focus, regardless of the lens sample position. Therefore, the ray
       corresponding to the lens position sample (dashed line) is given by the ray starting on the lens sample
       point and passing through the computed intersection point on the plane of focus.
```

```text
                                                             art
       Figure 5.15: Landscape scene with depth of field and only four samples per pixel: the depth of field is
       undersampled and the image is grainy. (Scene courtesy of Laubwerk.)
```

The SampleUniformDiskConcentric() function, which is defined in Section A.5.1, takes a (u, v) sample position in [0, 1)2 and maps it to a 2D unit disk centered at the origin (0, 0). To turn this into a point on the lens, these coordinates are scaled by the lens radius. The CameraSample class provides the (u, v) lens-sampling parameters in the pLens member variable. 〈Sample point on lens〉 ≡ 226

Point2f pLens = lensRadius * SampleUniformDiskConcentric(sample.pLens);

The ray’s origin is this point on the lens. Now it is necessary to determine the proper direction for the new ray. We know that all rays from the given image sample through the lens must converge at the same point on the plane of focus. Furthermore, we know that rays pass through the center of the lens without a change in direction, so finding the appropriate point of convergence is a matter of intersecting the unperturbed ray from the pinhole model with the plane of focus and then setting the new ray’s direction to be the vector from the point on the lens to the intersection point.

For this simple model, the plane of focus is perpendicular to the z axis and the ray starts at the origin, so intersecting the ray through the lens center with the plane of focus is straightforward. The t value of the intersection is given by art 〈Compute point on plane of focus〉 ≡ 226

Float ft = focalDistance / ray.d.z; Point3f pFocus = ray(ft);

Now the ray can be initialized. The origin is set to the sampled point on the lens, and the direction is set so that the ray passes through the point on the plane of focus, pFocus.

〈Update ray for effect of lens〉 ≡ 226

ray.o = Point3f(pLens.x, pLens.y, 0); ray.d = Normalize(pFocus - ray.o);

To compute ray differentials with the thin lens, the approach used in the fragment 〈Update ray for effect of lens〉 is applied to rays offset one pixel in the x and y directions on the film plane. The fragments that implement this, 〈Compute OrthographicCamera ray differentials accounting for lens〉 and 〈Compute PerspectiveCamera ray differentials accounting for lens〉, are not included here. 5.3 SPHERICAL CAMERA

One advantage of ray tracing compared to scan line or rasterization-based rendering methods is that it is easy to employ unusual image projections. We have great freedom in how the image sample positions are mapped into ray directions, since the rendering algorithm does not depend on properties such as straight lines in the scene always projecting to straight lines in the image. In this section, we will describe a camera model that traces rays in all directions around a point in the scene, giving a view of everything that is visible from that point. The Spherical Camera supports two spherical parameterizations from Section 3.8 to map points in the image to associated directions. Figure 5.16 shows this camera in action with the San Miguel model.

```text
   CameraSample 206
   CameraSample::pLens 206
   Float 23
   Normalize() 88
   OrthographicCamera 217
   PerspectiveCamera 220
   Point2f 92
   Point3f 92
   ProjectiveCamera::focalDistance 226
   ProjectiveCamera::lensRadius 226
   Ray::d 95
   Ray::o 95
   SampleUniformDiskConcentric() 1014
```

```text
                                                           art
        Figure 5.16: The San Miguel scene rendered with the SphericalCamera, which traces rays in all
        directions from the camera position. (a) Rendered using an equirectangular mapping. (b) Rendered with
        an equal-area mapping. (Scene courtesy of Guillermo M. Leal Llaguno.)
```

〈SphericalCamera Definition〉 ≡ class SphericalCamera : public CameraBase { public: 〈SphericalCamera::Mapping Definition 230〉 〈SphericalCamera Public Methods 229〉 private: 〈SphericalCamera Private Members 230〉 };

SphericalCamera does not derive from ProjectiveCamera since the projections that it uses are nonlinear and cannot be captured by a single 4 × 4 matrix. 〈SphericalCamera Public Methods〉 ≡ 229

SphericalCamera(CameraBaseParameters baseParameters, Mapping mapping) : CameraBase(baseParameters), mapping(mapping) { 〈Compute minimum differentials for SphericalCamera〉

```rust
}
```

The first mapping that SphericalCamera supports is the equirectangular mapping that was defined in Section 3.8.3. In the implementation here, θ values range from 0 at the top of the image to π at the bottom of the image, and ϕ values range from 0 to 2π, moving from left to right across the image. The equirectangular mapping is easy to evaluate and has the advantage that lines of constant latitude and longitude on the sphere remain straight. However, it preserves neither area nor angles between curves on the sphere (i.e., it is not conformal). These issues are especially evident at the top and bottom of the image in Figure 5.16(a). Therefore, the SphericalCamera also supports the equal-area mapping from Section 3.8.3. With this mapping, any finite solid angle of directions on the sphere maps to the same area in the image, regardless of where it is on the sphere. (This mapping is also used by the ImageInfiniteLight, which is described in Section 12.5.2, and so images rendered using this camera can be used as light sources.) The equal-area mapping’s use with the SphericalCamera is shown in Figure 5.16(b).

An enumeration reflects which mapping should be used.

```text
   CameraBase 212
   CameraBaseParameters 212
   ImageInfiniteLight 767
   ProjectiveCamera 214
   SphericalCamera 229
```

〈SphericalCamera::Mapping Definition〉 ≡ 229

enum Mapping { EquiRectangular, EqualArea }; 〈SphericalCamera Private Members〉 ≡ 229

Mapping mapping;

The main task of the GenerateRay() method is to apply the requested mapping. The rest of it follows the earlier GenerateRay() methods.

〈SphericalCamera Method Definitions〉 ≡ pstd::optional<CameraRay> SphericalCamera::GenerateRay(

CameraSample sample, SampledWavelengths &lambda) const { 〈Compute spherical camera ray direction 230〉 Ray ray(Point3f(0, 0, 0), dir, SampleTime(sample.time), medium); return CameraRay{RenderFromCamera(ray)}; }

For the use of both mappings, (u, v) coordinates in NDC space are found by dividing the raster space sample location by the image’s overall resolution. Then, after the mapping is applied, the y and z coordinates are swapped to account for the fact that both mappings are defined with z as the “up” direction, while y is “up” in camera space. 〈Compute spherical camera ray direction〉 ≡ 230

Point2f uv(sample.pfilm.x / film.FullResolution().x, sample.pfilm.y / film.FullResolution().y); Vector3f dir; if (mapping == EquiRectangular) { 〈Compute ray direction using equirectangular mapping 230〉 } else { 〈Compute ray direction using equal-area mapping 230〉 } pstd::swap(dir.y, dir.z);

For the equirectangular mapping, the (u, v) coordinates are scaled to cover the (θ, ϕ) range and the spherical coordinate formula is used to compute the ray direction. 〈Compute ray direction using equirectangular mapping〉 ≡ 230

Float theta = Pi * uv[1], phi = 2 * Pi * uv[0]; dir = SphericalDirection(std::sin(theta), std::cos(theta), phi);

The (u, v) values for the CameraSample may be slightly outside of the range [0, 1]2, due to the pixel sample filter function. A call to WrapEqualAreaSquare() takes care of handling the boundary conditions before EqualAreaSquareToSphere() performs the actual mapping. 〈Compute ray direction using equal-area mapping〉 ≡ 230

uv = WrapEqualAreaSquare(uv); dir = EqualAreaSquareToSphere(uv);

```text
   CameraBase::film 212
   CameraBase::RenderFromCamera() 214
   CameraBase::SampleTime() 213
   CameraRay 207
   CameraSample 206
   CameraSample::pFilm 206
   CameraSample::time 206
   EqualAreaSquareToSphere() 112
   Film::FullResolution() 246
   Float 23
   Pi 1033
   PixelSensor 234
   Point2f 92
   Point3f 92
   Ray 95
   SampledWavelengths 173
   SphericalCamera::mapping 230
   SphericalCamera::Mapping::EquiRectangular 230
   SphericalDirection() 106
   Vector3f 86
   WrapEqualAreaSquare() 113
```

## 5.4 FILM AND IMAGING

After the camera’s projection or lens system forms an image of the scene on the film, it is necessary to model how the film measures light to create the final image generated by the renderer. This section starts with an overview of the radiometry of how light is measured on the film and then continues with the topic of how spectral energy is converted to tristimulus colors (typically, RGB). This leads to the PixelSensor class, which models that process as well as further processing that is generally performed by cameras. After next considering how image samples on the film are accumulated into the pixels of the final image, we introduce the Film interface and then two implementations of it that put this model into practice.

```text
                                                              art
        Figure 5.17: The Image of the Scene on the Lens, as Seen from Two Points on the Film Plane. Both
        are from a rendering of the San Miguel scene. (a) As seen from a point where the scene is in sharp focus;
        the incident radiance is effectively constant over its area. (b) As seen from a pixel in an out-of-focus area,
        a small image of part of the scene is visible, with potentially rapidly varying radiance.
```

### 5.4.1 THE CAMERA MEASUREMENT EQUATION

Given a simulation of the process of real image formation, it is also worthwhile to more carefully define the radiometry of the measurement made by a film or a camera sensor. Rays from the rear of the lens to the film carry radiance from the scene. As considered from a point on the film plane, there is thus a set of directions from which radiance is incident. The distribution of radiance leaving the lens is affected by the amount of defocus blur seen by the point on the film—Figure 5.17 shows two images of the radiance from the lens as seen from two points on the film. Given the incident radiance function, we can define the irradiance at a point on the film plane. If we start with the definition of irradiance in terms of radiance, Equation (4.7), we can then convert from an integral over solid angle to an integral over area (in this case, an area Ae of the plane tangent to the rear lens element) using Equation (4.9). This gives us the irradiance for a point p on the film plane: art Figure 5.18 shows the geometry of the situation. Because the film plane is perpendicular to the lens’s plane, θ = θ′. We can further take advantage of the fact that the distance between p and p′ is equal to the axial distance from the film plane to the lens (which we will denote here by z) divided by cos θ. Putting this all together, we have art For cameras where the extent of the film is relatively large with respect to the distance z, the cos4 θ term can meaningfully reduce the incident irradiance—this factor also contributes to vignetting. Most modern digital cameras correct for this effect with preset correction factors that increase pixel values toward the edges of the sensor.

```text
    Film 244
```

Integrating irradiance at a point on the film over the time that the shutter is open gives radiant exposure, which is the radiometric unit for energy per area, J/m2.

```text
                                                               art
         Figure 5.18: Geometric setting for the irradiance measurement equation, (5.3). Radiance can be measured
         as it passes through points p′ on the plane tangent to the rear lens element to a point on the film plane p. z
         is the axial distance from the film plane to the rear element tangent plane, and θ is the angle between the
         vector from p′ to p and the optical axis.
```

art (Radiant exposure is also known as fluence.) Measuring radiant exposure at a point captures the effect that the amount of energy received on the film plane is partially related to the length of time the camera shutter is open. Photographic film (or CCD or CMOS sensors in digital cameras) measures radiant energy over a small area.4 Taking Equation (5.4) and also integrating over sensor pixel area, Ap, we have art

the Joules arriving at a pixel; this is called the camera measurement equation. Although these factors apply to all of the camera models introduced in this chapter, they are only included in the implementation of the RealisticCamera. The reason is purely pragmatic: most renderers do not model this effect, so omitting it from the simpler camera models makes it easier to compare images rendered by pbrt with those rendered by other systems.

### 5.4.2 MODELING SENSOR RESPONSE

Traditional film is based on a chemical process where silver halide crystals produce silver bromide when they are exposed to light. Silver halide is mostly sensitive to blue light, but color images can be captured using

multiple layers of crystals with color filters between them and dyes that make silver halide more responsive to other wavelengths. Modern digital cameras use CCD or CMOS sensors where each pixel effectively counts the number of photons it is exposed to by transforming photons into electrical charge. A variety of approaches to capturing color images have been developed, but the most common of them is to have a color filter over each pixel so that each measures red, green, or blue by counting only the photons that make it through the filter. Each pixel is often supplemented with a microlens that increases the amount of light that reaches the sensor.

```text
   RealisticCamera 206
```

For both film and digital sensors, color measurements at pixels can be modeled using spectral response curves that characterize the color filter or film’s chemical response to light as a function of wavelength. These functions are defined such that, for example, given an incident spectral distribution s(λ), a pixel’s red component is given by art Digital sensor pixels are typically arranged in a mosaic, with twice as many green pixels as red and blue, due to the human visual system’s greater sensitivity to green. One implication of pixel mosaics is that a demosaicing algorithm must be used to convert these sensor pixels to image pixels where the red, green, and blue color components are colocated. The naive approach of taking quads of mosaiced pixels and using their color values as is does not work well, since the constituent sensor pixels are at slightly different locations. There are many challenges in designing digital sensors, most of them stemming from the small size of pixels, which is a result of demand for high-resolution images. The smaller a pixel is, the fewer photons it is exposed to given a lens and exposure time, and in turn, the harder it is to accurately measure the light. Pixel arrays suffer from a variety of types of noise, of which shot noise is generally the most significant. It is due to the discrete nature of photons: there is random fluctuation in the number of photons that are counted, which matters more the fewer of them that are captured. Shot noise can be modeled using a Poisson distribution.

Each pixel must receive a sufficient amount of light either to cause the necessary chemical reactions or to count enough photons to capture an accurate image. In Equation (5.5), we saw that the energy captured at a pixel depends on the incident radiance, the pixel area, the exit pupil area, and the exposure time. With pixel area fixed for a given camera design, both increasing the lens aperture area and increasing the exposure time may introduce undesired side-effects in return for the additional light provided. A larger aperture reduces depth of field, which may lead to undesired defocus blur. Longer exposures can also cause blur due to moving objects in the scene or due to camera motion while the shutter is open. Sensors and film therefore provide an additional control in the form of an ISO setting. For physical film, ISO encodes its responsiveness to light (higher ISO values require less light to record an image). In digital cameras, ISO controls the gain—a scaling factor that is applied to pixel values as they are read from the sensor. With physical cameras, increasing gain exacerbates noise, as noise in the initial pixel measurements is amplified. Because pbrt does not model the noise present in readings from physical sensors, the ISO value can be set arbitrarily to achieve a desired exposure. In pbrt’s sensor model, we model neither mosaicing nor noise, nor other effects like blooming, where a pixel that is exposed to enough light will “spill over” and start increasing the measured value at adjacent pixels. We also do not simulate the process of image readout from the sensor: many cameras use a rolling shutter where scanlines are read in succession. For scenes with rapidly moving objects, this can give surprising results. Exercises at the end of the chapter suggest modifying pbrt in various ways to explore these effects. The PixelSensor class implements pbrt’s semi-idealized model of pixel color measurement. It is defined in the files film.h and film.cpp.5 〈PixelSensor Definition〉 ≡ class PixelSensor { public: 〈PixelSensor Public Methods 234〉 〈PixelSensor Public Members 237〉

private: 〈PixelSensor Private Methods 236〉 〈PixelSensor Private Members 234〉 };

PixelSensor models three components of sensor pixels’ operation: 1. Exposure controls: These are the user-settable parameters that control how bright or dark the image is. 2. RGB response: PixelSensor uses spectral response curves that are based on measurements of physical camera sensors to model the conversion of spectral radiance to tristimulus colors. 3. White balance: Cameras generally process the images they capture, including adjusting initial RGB values according to the color of illumination to model chromatic adaptation in the human visual system. Thus, captured images appear visually similar to what a human observer would remember having seen when taking a picture. pbrt includes a realistic camera model as well as idealized models based on projection matrices. Because pinhole cameras have apertures with infinitesimal area, we make some pragmatic trade-offs in the implementation of the PixelSensor so that images rendered with pinhole models are not completely black. We leave it the Camera’s responsibility to model the effect of the aperture size. The idealized models do not account for it at all, while the RealisticCamera does so in the 〈Compute weighting for RealisticCamera ray〉 fragment. The PixelSensor then only accounts for the shutter time and the ISO setting. These two factors are collected into a single quantity called the imaging ratio. The PixelSensor constructor takes the sensor’s RGB matching functions — art, ḡ, and art—and the imaging ratio as parameters. It also takes the color space requested by the user for the final output RGB values as well as the spectrum of an illuminant that specifies what color to consider to be white in the scene; together, these will make it possible to convert spectral energy to RGB as measured by the sensor and then to RGB in the output color space.

*Figure 5.19 shows the effect of modeling camera response, comparing rendering a scene using the XYZ matching functions to compute initial pixel colors with rendering with the matching functions for an actual camera sensor. 〈PixelSensor Public Methods〉 ≡ 234*

PixelSensor(Spectrum r, Spectrum g, Spectrum b, const RGBColorSpace *outputColorSpace, Spectrum sensorIllum, Float imagingRatio, Allocator alloc) : r_bar(r, alloc), g_bar(g, alloc), b_bar(b, alloc), imagingRatio(imagingRatio) { 〈Compute XYZ from camera RGB matrix 236〉 } 〈PixelSensor Private Members〉 ≡ 234

DenselySampledSpectrum r_bar, g_bar, b_bar; Float imagingRatio;

```text
   Allocator 40
   Camera 206
   DenselySampledSpectrum 167
   Float 23
   PixelSensor 234
   RealisticCamera 206
   RGBColorSpace 183
   Spectrum 165
```

```text
                                                           art
        Figure 5.19: The Effect of Accurately Modeling Camera Sensor Response. (a) Scene rendered using
        the XYZ matching functions for the PixelSensor. (b) Scene rendered using measured sensor response
        curves for a Canon EOS 5D camera. Note that the color tones are slightly cooler—they have less orange
        and more blue to them. (Scene courtesy of Beeple.)
```

The RGB color space in which a sensor pixel records light is generally not the same as the RGB color space that the user has specified for the final image. The former is generally specific to a camera and is determined by the physical properties of its pixel color filters, and the latter is generally a device-independent color space like sRGB or one of the other color spaces

described in Section 4.6.3. Therefore, the PixelSensor constructor computes a 3 × 3 matrix that converts from its RGB space to XYZ. From there, it is easy to convert to a particular output color space. This matrix is found by solving an optimization problem. It starts with over twenty spectral distributions, representing the reflectance of patches with a variety of colors from a standardized color chart. The constructor computes the RGB colors of those patches under the camera’s illuminant in the camera’s color space as well as their XYZ colors under the illuminant of the output color space. If these colors are respectively denoted by column vectors, then we can consider the problem of finding a 3 × 3 matrix M:

```text
   PixelSensor 234
```

As long as there are more than three reflectances, this is an over-constrained problem that can be solved using linear least squares. 〈Compute XYZ from camera RGB matrix〉 ≡ 234

〈Compute rgbCamera values for training swatches 236〉 〈Compute xyzOutput values for training swatches〉 〈Initialize XYZFromSensorRGB using linear least squares 237〉

Given the sensor’s illuminant, the work of computing the RGB coefficients for each reflectance is handled by the ProjectReflectance() method. 〈Compute rgbCamera values for training swatches〉 ≡ 236

Float rgbCamera[nSwatchReflectances][3]; for (int i = 0; i < nSwatchReflectances; ++i) { RGB rgb = ProjectReflectance<RGB>(swatchReflectances[i], sensorIllum, &r_bar, &g_bar, &b_bar); for (int c = 0; c < 3; ++c) rgbCamera[i][c] = rgb[c]; }

For good results, the spectra used for this optimization problem should present a good variety of representative real-world spectra. The ones used

in pbrt are based on measurements of a standard color chart.6

〈PixelSensor Private Members〉 +≡ 234

static constexpr int nSwatchReflectances = 24; static Spectrum swatchReflectances[nSwatchReflectances];

The ProjectReflectance() utility method takes spectral distributions for a reflectance and an illuminant as well as three spectral matching functions art for a tristimulus color space. It returns a triplet of color coefficients ci given by art where r is the spectral reflectance function, L is the illuminant’s spectral distribution, and art is a spectral matching function. Under the assumption that the second matching function art generally corresponds to luminance or at least something green, the color that causes the greatest response by the human visual system, the returned color triplet is normalized by art. In this way, the linear least squares fit at least roughly weights each RGB/XYZ pair according to visual importance. The ProjectReflectance() utility function takes the color space triplet type as a template parameter and is therefore able to return both RGB and XYZ values as appropriate. Its implementation follows the same general form as Spectrum::InnerProduct(), computing a Riemann sum over 1 nm spaced wavelengths, so it is not included here. 〈PixelSensor Private Methods〉 ≡ 234

template <typename Triplet> static Triplet ProjectReflectance(Spectrum r, Spectrum illum, Spectrum b1, Spectrum b2, Spectrum b3);

```text
   Float 23
   PixelSensor::nSwatchReflectances 236
   PixelSensor::ProjectReflectance() 236
   PixelSensor::swatchReflectances 236
   RGB 182
   Spectrum 165
   Spectrum::InnerProduct() 178
   XYZ 178
```

The fragment that computes XYZ coefficients in the output color space, 〈Compute xyzOutput values for training swatches〉, is generally similar to the one for RGB, with the differences that it uses the output illuminant and the XYZ spectral matching functions and initializes the xyzOutput array. It is therefore also not included here. Given the two matrices of color coefficients, a call to the LinearLeastSquares() function solves the optimization problem of Equation (5.7). 〈Initialize XYZFromSensorRGB using linear least squares〉 ≡ 236

```rust
let m = LinearLeastSquares(rgb_camera, xyz_output, n_swatches);
if m.is_none() {
    error_exit("Sensor XYZ from RGB matrix could not be solved.");
}
self.xyz_from_sensor_rgb = m.unwrap();
```

Because the RGB and XYZ colors are computed using the color spaces’ respective illuminants, the matrix M also performs white balancing. 〈PixelSensor Public Members〉 ≡ 234

SquareMatrix<3> XYZFromSensorRGB;

A second PixelSensor constructor uses XYZ matching functions for the pixel sensor’s spectral response curves. If a specific camera sensor is not specified in the scene description file, this is the default. Note that with this usage, the member variables r_bar, g_bar, and b_bar are misnamed in that they are actually X, Y, and Z. 〈PixelSensor Public Methods〉 +≡ 234

PixelSensor(const RGBColorSpace *outputColorSpace, Spectrum sensorIllum, Float imagingRatio, Allocator alloc) : r_bar(&Spectra::X(), alloc), g_bar(&Spectra::Y(), alloc), b_bar(&Spectra::Z(), alloc), imagingRatio(imagingRatio) { 〈Compute white balancing matrix for XYZ PixelSensor 237〉 }

By default, no white balancing is performed when PixelSensor converts to XYZ coefficients; that task is left for post-processing. However, if the user does specify a color temperature, white balancing is handled by the XYZFromSensorRGB matrix. (It is otherwise the identity matrix.) The WhiteBalance() function that computes this matrix will be described shortly; it takes the chromaticities of the white points of two color spaces and returns a matrix that maps the first to the second. 〈Compute white balancing matrix for XYZ PixelSensor〉 ≡ 237

if (sensorIllum) { Point2f sourceWhite = SpectrumToXYZ(sensorIllum).xy(); Point2f targetWhite = outputColorSpace->w; XYZFromSensorRGB = WhiteBalance(sourceWhite, targetWhite); }

```text
   Allocator 40
   ErrorExit() 1064
   Float 23
   LinearLeastSquares() 1051
   NSpectrumSamples 171
   PixelSensor 234
   PixelSensor::b_bar 234
   PixelSensor::g_bar 234
   PixelSensor::imagingRatio 234
   PixelSensor::r_bar 234
   PixelSensor::XYZFromSensorRGB 237
   Point2f 92
   RGBColorSpace 183
   RGBColorSpace::w 183
   SampledSpectrum 171
   SampledWavelengths 173
   Spectrum 165
   SpectrumToXYZ() 178
   SquareMatrix 1049
   WhiteBalance() 238
   XYZ::xy() 180
```

The main functionality provided by the PixelSensor is the ToSensorRGB() method, which converts a point-sampled spectral distribution L(λi) in a SampledSpectrum to RGB coefficients in the sensor’s

color space. It does so via Monte Carlo evaluation of the sensor response integral, Equation (5.6), giving estimators of the form art where n is equal to NSpectrumSamples. The associated PDF values are available from the SampledWavelengths and the sum over wavelengths and division by n is handled using SampledSpectrum::Average(). These coefficients are scaled by the imaging ratio, which completes the conversion. 〈PixelSensor Public Methods〉 +≡ 234

RGB ToSensorRGB(SampledSpectrum L, const SampledWavelengths &lambda) const { L = SafeDiv(L, lambda.PDF());

```rust
return imaging_ratio * RGB::new(
    (r_bar.sample(lambda) * l).average(),
    (g_bar.sample(lambda) * l).average(),
    (b_bar.sample(lambda) * l).average(),
);
}
```

Chromatic Adaptation and White Balance One of the remarkable properties of the human visual system is that the color of objects is generally seen as the same, even under different lighting conditions; this effect is called chromatic adaptation. Cameras perform a similar function so that photographs capture the colors that the person taking the picture remembers seeing; in that context, this process is called white balancing. pbrt provides a WhiteBalance() function that implements a white balancing algorithm called the von Kries transform. It takes two chromaticities: one is the chromaticity of the illumination and the other the chromaticity of the color white. (Recall the discussion in Section 4.6.3 of why white is not usually a constant spectrum but is instead defined as the color that humans perceive as white.) It returns a 3 × 3 matrix that applies the corresponding white balancing operation to XYZ colors. 〈White Balance Definitions〉 ≡ SquareMatrix<3> WhiteBalance(Point2f srcWhite, Point2f targetWhite) { 〈Find LMS coefficients for source and target white 238〉 〈Return white balancing matrix for source and target white 239〉 }

White balance with the von Kries transform is performed in the LMS color space, which is a color space where the responsivity of the three matching functions is specified to match the three types of cone in the human eye. By performing white balancing in the LMS space, we can model the effect of modulating the contribution of each type of cone in the eye, which is believed to be how chromatic adaptation is implemented in humans. After computing normalized XYZ colors corresponding to the provided chromaticities, the LMSFromXYZ matrix can be used to transform to LMS from XYZ.

〈Find LMS coefficients for source and target white〉 ≡ 238

XYZ srcXYZ = XYZ::FromxyY(srcWhite), dstXYZ = XYZ::FromxyY(targetWhite); auto srcLMS = LMSFromXYZ * srcXYZ, dstLMS = LMSFromXYZ * dstXYZ;

3 × 3 matrices that convert between LMS and XYZ are available as constants. 〈Color Space Constants〉 ≡ extern const SquareMatrix<3> LMSFromXYZ, XYZFromLMS;

Given a color in LMS space, white balancing is performed by dividing out the color of the scene’s illuminant and then multiplying by the color of the desired illuminant, which can be represented by a diagonal matrix. The complete white balance matrix that operates on XYZ colors follows directly.

```text
   LMSFromXYZ 238
   PixelSensor::b_bar 234
   PixelSensor::g_bar 234
   PixelSensor::imagingRatio 234
   PixelSensor::r_bar 234
   Point2f 92
   RGB 182
   SampledSpectrum 171
   SampledSpectrum::Average() 172
   SampledSpectrum::SafeDiv() 172
   SampledWavelengths 173
   SampledWavelengths::PDF() 174
   Spectrum::Sample() 175
   SquareMatrix 1049
   WhiteBalance() 238
   XYZ 178
   XYZ::FromxyY() 180
```

```text
                                                           art
        Figure 5.20: The Effect of White Balance. (a) Image of a scene with a yellow illuminant that has a
        similar spectral distribution to an incandescent light bulb. (b) White balanced image, using a color
        temperature of 3000 K. Due to chromatic adaptation, this image is much closer than (a) to what a human
        observer would perceive viewing this scene. (Scene courtesy of Wig42 from Blend Swap, via Benedikt
        Bitterli.)
```

〈Return white balancing matrix for source and target white〉 ≡ 238

SquareMatrix<3> LMScorrect = SquareMatrix<3>::Diag( dstLMS[0] / srcLMS[0], dstLMS[1] / srcLMS[1], dstLMS[2] / srcLMS[2]); return XYZFromLMS * LMScorrect * LMSFromXYZ;

*Figure 5.20 shows an image rendered with a yellowish illuminant and the image after white balancing with the illuminant’s chromaticity.*

Sampling Sensor Response Because the sensor response functions used by a PixelSensor describe the sensor’s wavelength-dependent response to radiance, it is worth at least approximately accounting for their variation when sampling the wavelengths of light that a ray is to carry. At minimum, a wavelength where all of them are zero should never be chosen, as that wavelength will make no contribution to the final image. More generally, applying importance sampling according to the sensor response functions is desirable as it offers the possibility of reducing error in the estimates of Equation (5.8).

```text
   LMSFromXYZ 238
   PixelSensor 234
   SquareMatrix 1049
   SquareMatrix::Diag() 1049
   XYZFromLMS 238
```

```text
                                                          art
       Figure 5.21: (a) Plot of normalized PDFs corresponding to the CIE Y matching function and the sum of
       the X, Y, and Z matching functions. (b) Plot of the parametric distribution pv(λ) from Equation (5.9).
```

However, choosing a distribution to use for sampling is challenging since the goal is minimizing error perceived by humans rather than strictly minimizing numeric error. Figure 5.21(a) shows the plots of both the CIE Y matching function and the sum of X, Y, and Z matching functions, both of which could be used. In practice, sampling according to Y alone gives excessive chromatic noise, but sampling by the sum of all three matching functions devotes too many samples to wavelengths between 400 nm and 500 nm, which are relatively unimportant visually. A parametric probability distribution function that balances these concerns and works well for sampling the visible wavelengths is art

with art

A = 0.0072 nm−1, and B = 538 nm. Figure 5.21(b) shows a plot of pv(λ).

Our implementation samples over the wavelength range from 360 nm to 830 nm. The normalization constant that converts f into a PDF is precomputed. 〈Sampling Inline Functions〉 +≡ Float VisibleWavelengthsPDF(Float lambda) { if (lambda < 360 || lambda > 830) return 0; return 0.0039398042f / Sqr(std::cosh(0.0072f * (lambda - 538))); }

The PDF can be sampled using the inversion method; the result is implemented in Sample VisibleWavelengths().

```text
   Float 23
   Sqr() 1034
```

〈Sampling Inline Functions〉 +≡ Float SampleVisibleWavelengths(Float u) { return 538 - 138.888889f * std::atanh(0.85691062f -1.82750197f * u); }

We can now implement another sampling method in the SampledWavelengths class, Sample Visible(), which uses this technique. 〈SampledWavelengths Public Methods〉 +≡ 173

static SampledWavelengths SampleVisible(Float u) { SampledWavelengths swl; for (int i = 0; i < NSpectrumSamples; ++i) {

〈Compute up for ith wavelength sample 241〉 swl.lambda[i] = SampleVisibleWavelengths(up); swl.pdf[i] = VisibleWavelengthsPDF(swl.lambda[i]); } return swl; }

Like SampledWavelengths::SampleUniform(), SampleVisible() uses a single random sample to generate all wavelength samples. It uses a slightly different approach, taking uniform steps across the [0, 1) sample space before sampling each wavelength. 〈Compute up for ith wavelength sample〉 ≡ 241

Float up = u + Float(i) / NSpectrumSamples; if (up > 1) up -= 1;

Using this distribution for sampling in place of a uniform distribution is worthwhile. Figure 5.22 shows two images of a scene, one rendered using uniform wavelength samples and the other rendered using SampleVisible(). Color noise is greatly reduced, with only a 1% increase in runtime.

```text
                                                         art
        Figure 5.22: (a) Scene rendered with 4 samples per pixel, each with 4 wavelength samples, sampled
        uniformly over the visible range. (b) Rendered at the same sampling rates but instead sampling
        wavelengths using SampledWavelengths::SampleVisible(). This image has substantially less color
        noise, at a negligible cost in additional computation. (Model courtesy of Yasutoshi Mori.)
```

```text
   Float 23
   NSpectrumSamples 171
   SampledWavelengths 173
   SampledWavelengths::lambda 173
   SampledWavelengths::pdf 173
   SampledWavelengths::SampleUniform() 173
   SampledWavelengths::SampleVisible() 241
   SampleVisibleWavelengths() 241
   VisibleWavelengthsPDF() 240
```

### 5.4.3 FILTERING IMAGE SAMPLES

The main responsibility of Film implementations is to aggregate multiple spectral samples at each pixel in order to compute a final value for it. In a physical camera, each pixel integrates light over a small area. Its response may have some spatial variation over that area that depends on the physical design of the sensor. In Chapter 8 we will consider this operation from the perspective of signal processing and will see that the details of where the image function is sampled and how those samples are weighted can significantly affect the final image quality. Pending those details, for now we will assume that some filter function f is used to define the spatial variation in sensor response around each image pixel. These filter functions quickly go to zero, encoding the fact that pixels only respond to light close to them on the film. They also encode any further spatial variation in the pixel’s response. With this approach, if we have an image function r(x, y) that gives the red color at an arbitrary position on the film (e.g., as measured using a sensor response function art with Equation (5.6)), then the filtered red value rf at a position (x, y) is given by art where the filter function f is assumed to integrate to 1. As usual, we will estimate this integral using point samples of the image function. The estimator is art Two approaches have been used in graphics to sample the integrand. The first, which was used in all three previous versions of pbrt, is to sample the image uniformly. Each image sample may then contribute to multiple pixels’ final values, depending on the extent of the filter function being used. This approach gives the estimator art where A is the film area. Figure 5.23 illustrates the approach; it shows a pixel at location (x, y) that has a pixel filter with extent radius.x in the x direction and radius.y in the y direction. All the samples at positions (xi, yi) inside the box given by the filter extent may contribute to the pixel’s value, depending on the filter function’s value for f (x − xi, y − yi).

While Equation (5.12) gives an unbiased estimate of the pixel value, variation in the filter function leads to variance in the estimates. Consider the case of a constant image function r: in that case, we would expect the resulting image pixels to all be exactly equal to r. However, the sum of filter values f (x − xi, y − yi) will not generally be equal to 1: it only equals 1 in expectation. Thus, the image will include noise, even in this simple setting. If the alternative estimator art is used instead, that variance is eliminated at the cost of a small amount of bias. (This is the weighted importance sampling Monte Carlo estimator.) In practice, this trade-off is worthwhile.

```text
   Film 244
```

```text
                                                             art
        Figure 5.23: 2D Image Filtering. To compute a filtered pixel value for the pixel marked with a filled
        circle located at (x, y), all the image samples inside the box around (x, y) with extent radius.x and
        radius.y need to be considered. Each of the image samples (xi, yi), denoted by open circles, is weighted
        by a 2D filter function, f (x − xi, y − yi). The weighted average of all samples is the final pixel value.
```

Equation (5.10) can also be estimated independently at each pixel. This is the approach used in this version of pbrt. In this case, it is worthwhile to sample points on the film using a distribution based on the filter function. This approach is known as filter importance sampling. With it, the spatial variation of the filter is accounted for purely via the distribution of sample locations for a pixel rather than scaling each sample’s contribution according to the filter’s value. If p ∝ f, then those two factors cancel in Equation (5.11) and we are left with an average of the r(xi, yi) sample values scaled by the constant of proportionality. However, here we must handle the rare (for rendering) case of estimating an integral that may be negative: as we will see in Chapter 8, filter functions that are partially negative can give better results than those that are nonnegative. In that case, we have p ∝ |f|, which gives art

where sign(x) is 1 if x > 0, 0 if it is 0, and −1 otherwise. However, this estimator has the same problem as Equation (5.12): even with a constant function r, the estimates will have variance depending on how many of the sign function evaluations give 1 and how many give −1. Therefore, this version of pbrt continues to use the weighted importance sampling estimator, computing pixel values as art with w(x, y) = f (x, y)/p(x, y). The first of these two approaches has the advantage that each image sample can contribute to multiple pixels’ final filtered values. This can be beneficial for rendering efficiency, as all the computation involved in computing the radiance for an image sample can be used to improve the accuracy of multiple pixels. However, using samples generated for other pixels is not always helpful: some of the sample generation algorithms implemented in Chapter 8 carefully position samples in ways that ensure good coverage of the sampling domain in a pixel. If samples from other pixels are mixed in with those, the full set of samples for a pixel may no longer have that same structure, which in turn can increase error. By not sharing samples across pixels, filter importance sampling does not have this problem. Filter importance sampling has further advantages. It makes parallel rendering easier: if the renderer is parallelized in a way that has different threads working on different pixels, there is never a chance that multiple threads will need to concurrently modify the same pixel’s value. A final advantage is that if there are any samples that are much brighter than the others due to a variance spike from a poorly sampled integrand, then those samples only contribute to a single pixel, rather than being smeared over multiple pixels. It is easier to fix up the resulting single-pixel artifacts than a neighborhood of them that have been affected by such a sample.

### 5.4.4 THE FILM INTERFACE

With the foundations of sensor response and pixel sample filtering established, we can introduce the Film interface. It is defined in the file base/film.h.

〈Film Definition〉 ≡ class Film : public TaggedPointer<RGBFilm, GBufferFilm, SpectralFilm> { public: 〈Film Interface 244〉 };

SpectralFilm, which is not described here, records spectral images over a specified wavelength range that is discretized into non-overlapping ranges. See the documentation of pbrt’s file format for more information about the SpectralFilm’s use.

Samples can be provided to the film in two ways. The first is from the Sampler selecting points on the film at which the Integrator estimates the radiance. These samples are provided to the Film via the AddSample() method, which takes the following parameters: The sample’s pixel coordinates, pFilm. The spectral radiance of the sample, L. The sample’s wavelengths, lambda. An optional VisibleSurface that describes the geometry at the first visible point along the sample’s camera ray. A weight for the sample to use in computing Equation (5.13) that is returned by Filter::Sample().

Film implementations can assume that multiple threads will not call AddSample() concurrently with the same pFilm location (though they should assume that threads will call it concurrently with different ones). Therefore, it is not necessary to worry about mutual exclusion in this method’s implementation unless some data that is not unique to a pixel is modified. 〈Film Interface〉 ≡ 244

void AddSample(Point2i pFilm, SampledSpectrum L, const SampledWavelengths &lambda, const VisibleSurface *visibleSurface, Float weight);

```text
   Film 244
```

```text
   Filter::Sample() 516
   Float 23
   GBufferFilm 253
   Integrator 22
   Point2i 92
   RGBFilm 248
   SampledSpectrum 171
   SampledWavelengths 173
   SpectralFilm 244
   TaggedPointer 1073
   VisibleSurface 245
```

The Film interface also includes a method that returns a bounding box of all the samples that may be generated. Note that this is different than the bounding box of the image pixels in the common case that the pixel filter extents are wider than a pixel. 〈Film Interface〉 +≡ 244

Bounds2f SampleBounds() const;

VisibleSurface holds an assortment of information about a point on a surface. 〈VisibleSurface Definition〉 ≡ class VisibleSurface { public: 〈VisibleSurface Public Methods 245〉 〈VisibleSurface Public Members 245〉 };

In addition to the point, normal, shading normal, and time, VisibleSurface stores the partial derivatives of depth at each pixel, ∂z/∂x and ∂z/∂y, where x and y are in raster space and z in camera space. These values are useful in image denoising algorithms, since they make it possible to test whether the surfaces in adjacent pixels are coplanar. The surface’s albedo is its spectral distribution of reflected light under uniform illumination; this quantity can be useful for separating texture from illumination before denoising. 〈VisibleSurface Public Members〉 ≡ 245

```rust
p: Point3f,
n: Normal3f,
ns: Normal3f,
uv: Point2f,
time: Float,
dpdx: Vector3f,
dpdy: Vector3f,
albedo: SampledSpectrum,
```

We will not include the VisibleSurface constructor here, as its main function is to copy appropriate values from the SurfaceInteraction into its member variables. 〈VisibleSurface Public Methods〉 ≡ 245

VisibleSurface(const SurfaceInteraction &si, SampledSpectrum albedo, const SampledWavelengths &lambda);

The set member variable indicates whether a VisibleSurface has been initialized. 〈VisibleSurface Public Members〉 +≡ 245

bool set = false; 〈VisibleSurface Public Methods〉 +≡ 245

operator bool() const { return set; }

Film implementations can indicate whether they use the VisibleSurface * passed to their AddSample() method via UsesVisibleSurface(). Providing this information allows integrators to skip the expense of initializing a VisibleSurface if it will not be used. 〈Film Interface〉 +≡ 244

bool UsesVisibleSurface() const;

```text
   Bounds2f 97
   Float 23
   Normal3f 94
   Point2f 92
   Point3f 92
   SampledSpectrum 171
   SampledWavelengths 173
   SurfaceInteraction 138
   Vector3f 86
   VisibleSurface 245
   VisibleSurface::set 245
```

Light transport algorithms that sample paths starting from the light sources (such as bidirectional path tracing) require the ability to “splat” contributions to arbitrary pixels. Rather than computing the final pixel value as a weighted average of contributing splats, splats are simply summed. Generally, the more splats that are around a given pixel, the brighter the pixel will be. AddSplat() splats the provided value at the given location in the image. In contrast to AddSample(), this method may be called concurrently by multiple threads that end up updating the same pixel. Therefore, Film implementations must either implement some form of mutual exclusion or use atomic operations in their implementations of this method. 〈Film Interface〉 +≡ 244

void AddSplat(Point2f p, SampledSpectrum v, const SampledWavelengths &lambda);

Film implementations must also provide a SampleWavelengths() method that samples from the range of wavelengths that the film’s sensor responds to (e.g., using SampledWavelengths::SampleVisible()).

〈Film Interface〉 +≡ 244

SampledWavelengths SampleWavelengths(Float u) const;

In addition, they must provide a handful of methods that give the extent of the image and the diagonal length of its sensor, measured in meters. 〈Film Interface〉 +≡ 244

Point2i FullResolution() const; Bounds2i PixelBounds() const; Float Diagonal() const;

A call to the Film::WriteImage() method directs the film to do the processing necessary to generate the final image and store it in a file. In addition to the camera transform, this method takes a scale factor that is applied to the samples provided to the AddSplat() method.

〈Film Interface〉 +≡ 244

void WriteImage(ImageMetadata metadata, Float splatScale = 1);

The ToOutputRGB() method allows callers to find the output RGB value that results for given spectral radiance samples from applying the PixelSensor’s model, performing white balancing, and then converting to the output color space. (This method is used by the SPPMIntegrator included in the online edition, which has requirements that cause it to maintain the final image itself rather than using a Film implementation.) 〈Film Interface〉 +≡ 244

RGB ToOutputRGB(SampledSpectrum L, const SampledWavelengths &lambda) const;

A caller can also request the entire image to be returned, as well as the RGB value for a single pixel. The latter method is used for displaying in-progress images during rendering. 〈Film Interface〉 +≡ 244

Image GetImage(ImageMetadata *metadata, Float splatScale = 1); RGB GetPixelRGB(Point2i p, Float splatScale = 1) const;

Finally, Film implementations must provide access to a few additional values for use in other parts of the system. 〈Film Interface〉 +≡ 244

Filter GetFilter() const; const PixelSensor *GetPixelSensor() const; std::string GetFilename() const;

```text
   Bounds2i 97
   Film 244
   Film::WriteImage() 246
   Filter 515
   Float 23
   Image 1079
   ImageMetadata 1086
   PixelSensor 234
   Point2f 92
   Point2i 92
   RGB 182
   SampledSpectrum 171
   SampledWavelengths 173
   SampledWavelengths::SampleVisible() 241
```

### 5.4.5 COMMON FILM FUNCTIONALITY

As we did with CameraBase for Camera implementations, we have written a FilmBase class that Film implementations can inherit from. It collects commonly used member variables and is able to provide a few of the methods required by the Film interface. 〈FilmBase Definition〉 ≡ class FilmBase { public: 〈FilmBase Public Methods 247〉 protected: 〈FilmBase Protected Members 247〉 };

The FilmBase constructor takes a number of values: the overall resolution of the image in pixels; a bounding box that may specify a subset of the full image; a filter function; a PixelSensor; the length of the diagonal of the film’s physical area; and the filename for the output image. These are all bundled up into a small structure in order to shorten the parameter lists of forthcoming constructors. 〈FilmBaseParameters Definition〉 ≡ struct FilmBaseParameters { Point2i fullResolution; Bounds2i pixelBounds; Filter filter; Float diagonal; const PixelSensor *sensor; std::string filename; };

The FilmBase constructor then just copies the various values from the parameter structure, converting the film diagonal length from millimeters (as specified in scene description files) to meters, the unit used for measuring distance in pbrt.

〈FilmBase Public Methods〉 ≡ 247

FilmBase(FilmBaseParameters p) : fullResolution(p.fullResolution), pixelBounds(p.pixelBounds), filter(p.filter), diagonal(p.diagonal * .001f), sensor(p.sensor), filename(p.filename) { } 〈FilmBase Protected Members〉 ≡ 247

Point2i fullResolution; Bounds2i pixelBounds; Filter filter; Float diagonal; const PixelSensor *sensor; std::string filename;

```text
   Bounds2i 97
   Camera 206
   CameraBase 212
   Film 244
   FilmBase 247
   FilmBaseParameters 247
   Filter 515
   Float 23
   PixelSensor 234
   Point2i 92
```

Having these values makes it possible to immediately implement a number of the methods required by the Film interface. 〈FilmBase Public Methods〉 +≡ 247

Point2i FullResolution() const { return fullResolution; } Bounds2i PixelBounds() const { return pixelBounds; } Float Diagonal() const { return diagonal; } Filter GetFilter() const { return filter; } const PixelSensor *GetPixelSensor() const { return sensor; } std::string GetFilename() const { return filename; }

An implementation of SampleWavelengths() samples according to the distribution in Equation (5.9). 〈FilmBase Public Methods〉 +≡ 247

SampledWavelengths SampleWavelengths(Float u) const {

```rust
return SampledWavelengths::sample_visible(u);
}
```

The Film::SampleBounds() method can also be easily implemented, given the Filter. Computing the sample bounds involves both expanding by the filter radius and accounting for half-pixel offsets that come from the conventions used in pbrt for pixel coordinates; these are explained in more detail in Section 8.1.4. 〈FilmBase Method Definitions〉 ≡ Bounds2f FilmBase::SampleBounds() const { Vector2f radius = filter.Radius(); return Bounds2f(pixelBounds.pMin - radius + Vector2f(0.5f, 0.5f), pixelBounds.pMax + radius - Vector2f(0.5f, 0.5f)); }

### 5.4.6 RGBFilm

RGBFilm records an image represented by RGB color.

〈RGBFilm Definition〉 ≡ class RGBFilm : public FilmBase { public: 〈RGBFilm Public Methods 249〉 private: 〈RGBFilm::Pixel Definition 249〉 〈RGBFilm Private Members 249〉 };

In addition to the parameters that are passed along to FilmBase, RGBFilm takes a color space to use for the output image, a parameter that allows specifying the maximum value of an RGB color component, and a parameter that controls the floating-point precision in the output image. 〈RGBFilm Method Definitions〉 ≡

RGBFilm::RGBFilm(FilmBaseParameters p, const RGBColorSpace *colorSpace, Float maxComponentValue, bool writeFP16, Allocator alloc) : FilmBase(p), pixels(p.pixelBounds, alloc), colorSpace(colorSpace), maxComponentValue(maxComponentValue), writeFP16(writeFP16) { filterIntegral = filter.Integral(); 〈Compute outputRGBFromSensorRGB matrix 249〉 }

```text
   Allocator 40
   Bounds2f 97
   Bounds2i 97
   Bounds3::pMax 98
   Bounds3::pMin 98
   Film::SampleBounds() 245
   FilmBase 247
   FilmBase::diagonal 247
   FilmBase::filename 247
   FilmBase::filter 247
   FilmBase::fullResolution 247
   FilmBase::pixelBounds 247
   FilmBase::sensor 247
   FilmBaseParameters 247
   Filter 515
   Filter::Integral() 516
   Filter::Radius() 515
   Float 23
   PixelSensor 234
   Point2i 92
   RGBColorSpace 183
   RGBFilm 248
   RGBFilm::filterIntegral 249
   SampledWavelengths 173
   SampledWavelengths::SampleVisible() 241
   Vector2f 86
```

The integral of the filter function will be useful to normalize the filter values used for samples provided via AddSplat(), so it is cached in a

member variable. 〈RGBFilm Private Members〉 ≡ 248

const RGBColorSpace *colorSpace; Float maxComponentValue; bool writeFP16; Float filterIntegral;

The color space for the final image is given by a user-specified RGBColorSpace that is unlikely to be the same as the sensor’s RGB color space. The constructor therefore computes a 3 × 3 matrix that transforms sensor RGB values to the output color space. 〈Compute outputRGBFromSensorRGB matrix〉 ≡ 248

outputRGBFromSensorRGB = colorSpace->RGBFromXYZ * sensor->XYZFromSensorRGB; 〈RGBFilm Private Members〉 +≡ 248

SquareMatrix<3> outputRGBFromSensorRGB;

Given the pixel resolution of the (possibly cropped) image, the constructor allocates a 2D array of Pixel structures, with one for each pixel. The running weighted sums of pixel contributions are represented using RGB colors in the rgbSum member variable. weightSum holds the sum of filter weight values for the sample contributions to the pixel. These respectively correspond to the numerator and denominator in Equation (5.13). Finally, rgbSplat holds an (unweighted) sum of sample splats.

Double-precision floating point is used for all of these quantities. Single-precision floats are almost always sufficient, but when used for reference images rendered with high sample counts they may have insufficient precision to accurately store their associated sums. Although it is rare for this error to be visually evident, it can cause problems with reference images that are used to evaluate the error of Monte Carlo sampling algorithms. Figure 5.24 shows an example of this problem. We rendered a reference image of a test scene using 4 million samples in each pixel, using both 32-bit and 64-bit floating-point values for the RGBFilm pixel values. We then plotted mean squared error (MSE) as a function of sample count. For an unbiased Monte Carlo estimator, MSE is O(1/n) in the number of samples

taken n; on a log–log plot, it should be a straight line with slope −1. However, we can see that for n > 1000 with a 32-bit float reference image, the reduction in MSE seems to flatten out—more samples do not seem to reduce error. With 64-bit floats, the curve maintains its expected path. 〈RGBFilm::Pixel Definition〉 ≡ 248

struct Pixel { double rgbSum[3] = {0., 0., 0.}; double weightSum = 0.; AtomicDouble rgbSplat[3]; }; 〈RGBFilm Private Members〉 +≡ 248

Array2D<Pixel> pixels;

The RGBFilm does not use the VisibleSurface * passed to AddSample().

〈RGBFilm Public Methods〉 ≡ 248

bool UsesVisibleSurface() const { return false; }

```text
   Array2D 1069
   AtomicDouble 1100
   FilmBase::sensor 247
   Float 23
   PixelSensor::XYZFromSensorRGB 237
   RGBColorSpace 183
   RGBColorSpace::RGBFromXYZ 184
   RGBFilm 248
   RGBFilm::outputRGBFromSensorRGB 249
   SquareMatrix 1049
```

```text
                                                              art
        Figure 5.24: Mean Squared Error as a Function of Sample Count. When rendering a scene using an
        unbiased Monte Carlo estimator, we expect MSE to be related to the number of samples n by O(1/n). With
        a log–log plot, this rate corresponds to a straight line with slope −1. For the test scene considered here, we
        can see that using 32-bit floats for the reference image causes reported error to inaccurately stop
        decreasing after 1,000 or so samples.
```

AddSample() converts spectral radiance to sensor RGB before updating the Pixel corresponding to the point pFilm.

〈RGBFilm Public Methods〉 +≡ 248

void AddSample(Point2i pFilm, SampledSpectrum L, const SampledWavelengths &lambda, const VisibleSurface *, Float weight) { 〈Convert sample radiance to PixelSensor RGB 250〉 〈Optionally clamp sensor RGB value 251〉 〈Update pixel values with filtered sample contribution 251〉 }

The radiance value is first converted to RGB by the sensor. 〈Convert sample radiance to PixelSensor RGB〉 ≡ 250, 252

RGB rgb = sensor->ToSensorRGB(L, lambda);

Images rendered with Monte Carlo integration can exhibit bright spikes of noise in pixels where the sampling distributions that were used do not match the integrand well such that when f (x)/p(x) is computed in the Monte Carlo estimator, f (x) is very large and p(x) is very small. (Such pixels are colloquially called “fireflies.”) Many additional samples may be required to get an accurate estimate for that pixel. A widely used technique to reduce the effect of fireflies is to clamp all sample contributions to some maximum amount. Doing so introduces error: energy is lost, and the image is no longer an unbiased estimate of the true image. However, when the aesthetics of rendered images are more important than their mathematics, this can be a useful remedy. Figure 5.25 shows an example of its use. The RGBFilm’s maxComponentValue parameter can be set to a threshold that is used for clamping. It is infinite by default, and no clamping is performed.

```text
   FilmBase::sensor 247
   Float 23
   PixelSensor::ToSensorRGB() 238
   Point2i 92
   RGB 182
   SampledSpectrum 171
   SampledWavelengths 173
   VisibleSurface 245
```

```text
                                                           art
        Figure 5.25: Image with High Variance in Some Pixels. This scene suffers from variance spikes in
        pixels due to difficult-to-sample light paths that occasionally intersect the sun. (a) Image rendered
        normally. (b) Image rendered with clamping, where pixel sample RGB values are clamped to have values
        no larger than 10. The image looks much better with clamping, though at a cost of some loss of energy.
        (Model courtesy of Yasutoshi Mori.)
```

〈Optionally clamp sensor RGB value〉 ≡ 250, 252

Float m = std::max({rgb.r, rgb.g, rgb.b}); if (m > maxComponentValue) rgb *= maxComponentValue / m;

Given the possibly clamped RGB value, the pixel it lies in can be updated by adding its contributions to the running sums of the numerator and denominator of Equation (5.13). 〈Update pixel values with filtered sample contribution〉 ≡ 250

Pixel &pixel = pixels[pFilm]; for (int c = 0; c < 3; ++c) pixel.rgbSum[c] += weight * rgb[c]; pixel.weightSum += weight;

```text
   Float 23
   RGBFilm::maxComponentValue 249
   RGBFilm::Pixel::rgbSum 249
   RGBFilm::Pixel::weightSum 249
   RGBFilm::pixels 249
```

The AddSplat() method first reuses the first two fragments from AddSample() to compute the RGB value of the provided radiance L.

〈RGBFilm Method Definitions〉 +≡ void RGBFilm::AddSplat(Point2f p, SampledSpectrum L, const SampledWavelengths &lambda) { 〈Convert sample radiance to PixelSensor RGB 250〉 〈Optionally clamp sensor RGB value 251〉 〈Compute bounds of affected pixels for splat, splatBounds 252〉

for (Point2i pi : splatBounds) { 〈Evaluate filter at pi and add splat contribution 252〉 } }

Because splatted contributions are not a result of pixel samples but are points in the scene that are projected onto the film plane, it is necessary to consider their contribution to multiple pixels, since each pixel’s reconstruction filter generally extends out to include contributions from nearby pixels. First, a bounding box of potentially affected pixels is found using the filter’s radius. See Section 8.1.4, which explains the conventions for indexing into pixels in pbrt and, in particular, the addition of (0.5, 0.5) to the pixel coordinate here. 〈Compute bounds of affected pixels for splat, splatBounds〉 ≡ 252

Point2f pDiscrete = p + Vector2f(0.5, 0.5); Vector2f radius = filter.Radius(); Bounds2i splatBounds(Point2i(Floor(pDiscrete - radius)), Point2i(Floor(pDiscrete + radius)) + Vector2i(1, 1)); splatBounds = Intersect(splatBounds, pixelBounds);

If the filter weight is nonzero, the splat’s weighted contribution is added. Unlike with AddSample(), no sum of filter weights is maintained; normalization is handled later using the filter’s integral, as per Equation (5.10). 〈Evaluate filter at pi and add splat contribution〉 ≡ 252

Float wt = filter.Evaluate(Point2f(p - pi - Vector2f(0.5, 0.5))); if (wt != 0) { Pixel &pixel = pixels[pi]; for (int i = 0; i < 3; ++i) pixel.rgbSplat[i].Add(wt * rgb[i]); }

GetPixelRGB() returns the final RGB value for a given pixel in the RGBFilm’s output color space.

〈RGBFilm Public Methods〉 +≡ 248

RGB GetPixelRGB(Point2i p, Float splatScale = 1) const { const Pixel &pixel = pixels[p]; RGB rgb(pixel.rgbSum[0], pixel.rgbSum[1], pixel.rgbSum[2]); 〈Normalize rgb with weight sum 253〉 〈Add splat value at pixel 253〉 〈Convert rgb to output RGB color space 253〉 return rgb; }

```text
   AtomicDouble::Add() 1100
   Bounds2::Intersect() 99
   Bounds2i 97
   FilmBase::filter 247
   Filter::Evaluate() 516
   Filter::Radius() 515
   Float 23
   Point2f 92
   Point2i 92
   RGB 182
   RGBFilm::Pixel::rgbSplat 249
   RGBFilm::Pixel::rgbSum 249
   RGBFilm::pixels 249
   SampledSpectrum 171
   SampledWavelengths 173
   Vector2f 86
   Vector2i 86
```

First, the final pixel contribution from the values provided by AddSample() is computed via Equation (5.13). 〈Normalize rgb with weight sum〉 ≡ 252

Float weightSum = pixel.weightSum; if (weightSum != 0) rgb /= weightSum;

Then Equation (5.10) can be applied to incorporate any splatted values. 〈Add splat value at pixel〉 ≡ 252

for (int c = 0; c < 3; ++c) rgb[c] += splatScale * pixel.rgbSplat[c] / filterIntegral;

Finally, the color conversion matrix brings the RGB value into the output color space. 〈Convert rgb to output RGB color space〉 ≡ 252

rgb = outputRGBFromSensorRGB * rgb;

ToOutputRGB()’s implementation first uses the sensor to compute a sensor RGB and then converts to the output color space. 〈RGBFilm Public Methods〉 +≡ 248

RGB ToOutputRGB(SampledSpectrum L, const SampledWavelengths &lambda) const { RGB sensorRGB = sensor->ToSensorRGB(L, lambda); return outputRGBFromSensorRGB * sensorRGB; }

We will not include the straightforward RGBFilm WriteImage() or GetImage() method implementations in the book. The former calls GetImage() before calling Image::Write(), and the latter fills in an image using GetPixelRGB() to get each pixel’s value.

### 5.4.7 GBufferFilm

The GBufferFilm stores not only RGB at each pixel, but also additional information about the geometry at the first visible intersection point. This additional information is useful for a variety of applications, ranging from image denoising algorithms to providing training data for machine learning applications. 〈GBufferFilm Definition〉 ≡ class GBufferFilm : public FilmBase { public: 〈GBufferFilm Public Methods〉 private: 〈GBufferFilm::Pixel Definition 254〉 〈GBufferFilm Private Members〉 };

```text
    FilmBase 247
```

```text
   FilmBase::sensor 247
   Float 23
   Image::Write() 1086
   PixelSensor::ToSensorRGB() 238
   RGB 182
   RGBFilm 248
   RGBFilm::filterIntegral 249
   RGBFilm::outputRGBFromSensorRGB 249
   RGBFilm::Pixel::rgbSplat 249
   RGBFilm::Pixel::weightSum 249
   SampledSpectrum 171
   SampledWavelengths 173
   VarianceEstimator 1048
```

We will not include any of the GBufferFilm implementation other than its Pixel structure, which augments the one used in RGBFilm with additional fields that store geometric information. It also stores estimates of the variance of the red, green, and blue color values at each pixel using the VarianceEstimator class, which is defined in Section B.2.11. The rest of the implementation is a straightforward generalization of RGBFilm that also updates these additional values. 〈GBufferFilm::Pixel Definition〉 ≡ 253

struct Pixel { double rgbSum[3] = {0., 0., 0.}; double weightSum = 0., gBufferWeightSum = 0.; AtomicDouble rgbSplat[3]; Point3f pSum; Float dzdxSum = 0, dzdySum = 0; Normal3f nSum, nsSum; Point2f uvSum; double rgbAlbedoSum[3] = {0., 0., 0.}; VarianceEstimator<Float> rgbVariance[3]; };

FURTHER READING

In his seminal Sketchpad system, Sutherland (1963) was the first to use projection matrices for computer graphics. Akenine-Möller et al. (2018) have provided a particularly well-written derivation of the orthographic and perspective projection matrices. Other good references for projections are Rogers and Adams’s Mathematical Elements for Computer Graphics

(1990) and Eberly’s book (2001) on game engine design. See Adams and Levoy (2007) for a broad analysis of the types of radiance measurements that can be taken with cameras that have non-pinhole apertures. An unusual projection method was used by Greene and Heckbert (1986) for generating images for OMNIMAX® theaters. Potmesil and Chakravarty (1981, 1982, 1983) did early work on depth of field and motion blur in computer graphics. Cook and collaborators developed a more accurate model for these effects based on the thin lens model; this is the approach used for the depth of field calculations in Section 5.2.3 (Cook et al. 1984; Cook 1986). An alternative approach to motion blur was described by Gribel and Akenine-Möller (2017), who analytically computed the time ranges of ray–triangle intersections to eliminate stochastic sampling in time. Kolb, Mitchell, and Hanrahan (1995) showed how to simulate complex camera lens systems with ray tracing in order to model the imaging effects of real cameras; the RealisticCamera is based on their approach. Steinert et al. (2011) improved a number of details of this simulation, incorporating wavelength-dependent effects and accounting for both diffraction and glare. Joo et al. (2016) extended this approach to handle aspheric lenses and modeled diffraction at the aperture stop, which causes some brightening at the edges of the circle of confusion in practice. See the books by Hecht (2002) and Smith (2007) for excellent introductions to optics and lens systems. Hullin et al. (2012) used polynomials to model the effect of lenses on rays passing through them; they were able to construct polynomials that approximate entire lens systems from polynomial approximations of individual lenses. This approach saves the computational expense of tracing rays through lenses, though for complex scenes, this cost is generally negligible in relation to the rest of the rendering computations. Hanika and Dachsbacher (2014) improved the accuracy of this approach and showed how to combine it with bidirectional path tracing. Schrade et al. (2016) showed good results with approximation of wide-angle lenses using sparse higher-degree polynomials.

```text
   AtomicDouble 1100
   Float 23
   Normal3f 94
   Point2f 92
   Point3f 92
   RGBFilm 248
   VarianceEstimator 1048
```

Film and Imaging The film sensor model presented in Section 5.4.2 and the PixelSensor class implementation are from the PhysLight system described by Langlands and Fascione (2020). See also Chen et al. (2009), who described the implementation of a fairly complete simulation of a digital camera, including the analog-to-digital conversion and noise in the measured pixel values inherent in this process. Filter importance sampling, as described in Section 8.8, was described in a paper by Ernst et al. (2006). This technique is also proposed in Shirley’s Ph.D. thesis (1990). The idea of storing additional information about the properties of the visible surface in a pixel was introduced by Perlin (1985a) and Saito and Takahashi (1990), who also coined the term G-Buffer. Shade et al. (1998) introduced the generalization of storing information about all the surfaces along each camera ray and applied this representation to view interpolation, using the originally hidden surfaces to handle disocclusion. Celarek et al. (2019) developed techniques for evaluating sampling schemes based on computing both the expectation and variance of MSE and described approaches for evaluating error in rendered images across both pixels and frequencies. The sampling technique that approximates the XYZ matching curves is due to Radziszewski et al. (2009). The SpectralFilm uses a representation for spectral images in the OpenEXR format that was introduced by Fichet et al. (2021). As discussed in Section 5.4.2, the human visual system generally factors out the illumination color to perceive surfaces’ colors independently of it. A

number of methods have been developed to process photographs to perform white balancing to eliminate the tinge of light source colors; see Gijsenij et al. (2011) for a survey. White balancing photographs can be challenging, since the only information available to white balancing algorithms is the final pixel values. In a renderer, the problem is easier, as information about the light sources is directly available; Wilkie and Weidlich (2009) developed an efficient method to perform accurate white balancing in a renderer.

Denoising A wide range of approaches have been developed for removing Monte Carlo noise from rendered images. Here we will discuss those that are based on the statistical characteristics of the sample values themselves. In the “Further Reading” section of Chapter 8, we will discuss ones that derive filters that account for the underlying light transport equations used to form the image. Zwicker et al.’s report (2015) has thorough coverage of both approaches to denoising through 2015. We will therefore focus here on some of the foundational work as well as more recent developments. Lee and Redner (1990) suggested using an alpha-trimmed mean filter for this task; it discards some number of samples at the low and high range of the sample values. The median filter, where all but a single sample are discarded, is a special case of it. Jensen and Christensen (1995) observed that it can be effective to separate out the contributions to pixel values based on the type of illumination they represent; low-frequency indirect illumination can be filtered differently from high-frequency direct illumination, thus reducing noise in the final image. They developed an effective filtering technique based on this observation.

```text
   PixelSensor 234
   SpectralFilm 244
```

McCool (1999) used the depth, surface normal, and color at each pixel to determine how to blend pixel values with their neighbors in order to better preserve edges in the filtered image. Keller and collaborators introduced the discontinuity buffer (Keller 1998; Wald et al. 2002). In addition to filtering slowly varying quantities like indirect illumination separately from more quickly varying quantities like surface reflectance, the discontinuity buffer

also uses geometric quantities like the surface normal to determine filter extents. Dammertz et al. (2010) introduced a denoising algorithm based on edge-aware image filtering, applied hierarchically so that very wide kernels can be used with good performance. This approach was improved by Schied et al. (2017), who used estimates of variance at each pixel to set filter widths and incorporated temporal reuse, using filtered results from the previous frame in a real-time ray tracer. Bitterli et al. (2016) analyzed a variety of previous denoising techniques in a unified framework and derived a new approach based on a first-order regression of pixel values. Boughida and Boubekeur (2017) described a Bayesian approach based on statistics of all the samples in a pixel, and Vicini et al. (2019) considered the problem of denoising “deep” images, where each pixel may contain multiple color values, each at a different depth. Some filtering techniques focus solely on the outlier pixels that result when the sampling probability in the Monte Carlo estimator is a poor match to the integrand and is far too small for a sample. (As mentioned previously, the resulting pixels are sometimes called “fireflies,” in a nod to their bright transience.) Rushmeier and Ward (1994) developed an early technique to address this issue based on detecting outlier pixels and spreading their energy to nearby pixels in order to maintain an unbiased estimate of the true image. DeCoro et al. (2010) suggested storing all pixel sample values and then rejecting outliers before filtering them to compute final pixel values. Zirr et al. (2018) proposed an improved approach that uses the distribution of sample values at each pixel to detect and reweight outlier samples. Notably, their approach does not need to store all the individual samples, but can be implemented by partitioning samples into one of a small number of image buffers based on their magnitude. More recently, Buisine et al. (2021) proposed using a median of means filter, which is effective at removing outliers but has slower convergence than the mean. They therefore dynamically select between the mean and median of means depending on the characteristics of the sample values. As with many other areas of image processing and understanding, techniques based on machine learning have recently been applied to denoising rendered images. This work started with Kalantari et al. (2015),

who used relatively small neural networks to determine parameters for conventional denoising filters. Approaches based on deep learning and convolutional neural networks soon followed with Bako et al. (2017), Chaitanya et al. (2017), and Vogels et al. (2018) developing autoencoders based on the u-net architecture (Ronneberger et al. 2015). Xu et al. (2019) applied adversarial networks to improve the training of such denoisers. Gharbi et al. (2019) showed that filtering the individual samples with a neural network can give much better results than sampling the pixels with the samples already averaged. Munkberg and Hasselgren (2020) described an architecture that reduces the memory and computation required for this approach.

EXERCISES

➋ 5.1 Some types of cameras expose the film by sliding a rectangular slit across the film. This leads to interesting effects when objects are moving in a different direction from the exposure slit (Glassner 1999; Stephenson 2007). Furthermore, most digital cameras read out pixel values from scanlines in succession over a period of a few milliseconds; this leads to rolling shutter artifacts, which have similar visual characteristics. Modify the way that time samples are generated in one or more of the camera implementations in this chapter to model such effects. Render images with moving objects that clearly show the effect of accounting for this issue. ➋ 5.2 Write an application that loads images rendered by the SphericalCamera and uses texture mapping to apply them to a sphere centered at the eyepoint such that they can be viewed interactively. The user should be able to freely change the viewing direction. If the correct texture-mapping function is used for generating texture coordinates on the sphere, the image generated by the application will appear as if the viewer was at the camera’s location in the scene when it was rendered, thus giving the user the ability to interactively look around the scene. ➋ 5.3 Focal stack rendering: A focal stack is a series of images of a fixed scene where the camera is focused at a different distance for each image. Hasinoff and Kutulakos (2011) and Jacobs et al. (2012) introduced a number of applications of focal stacks, including freeform depth of field, where the user can specify arbitrary depths that are in focus, achieving effects not possible with traditional optics. Render focal stacks with pbrt and write an interactive tool to control focus effects with them. ➌ 5.4 Light field camera: Ng et al. (2005) discussed the physical design and applications of a camera that captures small images of the exit pupil across the film, rather than averaging the radiance over the entire exit pupil at each pixel, as conventional cameras do. Such a camera captures a representation of the light field—the spatially and directionally varying distribution of radiance arriving at the camera sensor. By capturing the light field, a number of interesting operations are possible, including refocusing photographs after they have been taken. Read Ng et al.’s paper and implement a Camera in pbrt that captures the light field of a scene. Write a tool to allow users to interactively refocus these light fields. ➌ 5.5 The Cameras in this chapter place the film at the center of and perpendicular to the optical axis. While this is the most common configuration of actual cameras, interesting effects

```text
        can be achieved by adjusting the film’s placement with respect to the lens system.
```

```text
        For example, the plane of focus in the current implementation is always perpendicular to
        the optical axis; if the film plane (or the lens system) is tilted so that the film is not
        perpendicular to the optical axis, then the plane of focus is no longer perpendicular to the
        optical axis. (This can be useful for landscape photography, for example, where aligning
        the plane of focus with the ground plane allows greater depth of field even with larger
        apertures.) Alternatively, the film plane can be shifted so that it is not centered on the
        optical axis; this shift can be used to keep the plane of focus aligned with a very tall
        object, for example.
```

Modify the PerspectiveCamera to allow one or both of these adjustments and render images showing the result. (You may find Kensler’s (2021) chapter useful.) ➋ 5.6 The clamping approach used to suppress outlier sample values in the RGBFilm and GBufferFilm is a heavy-handed solution that can cause a significant amount of energy loss in the image. (Consider, for example, pixels where the sun is directly visible—the radiance along rays in those pixels may be extremely high, though it is not a cause of spiky pixels and should not be clamped.) Implement a more principled solution to this problem such as the technique of Zirr et al. (2018). Render images with your implementation and pbrt’s current approach and compare the results.

```text
            SphericalCamera 229
```

➋ 5.7 Investigate the sources of noise in camera sensors and mathematical models to simulate them. Then, modify the PixelSensor class to model the effect of noise. In addition to shot noise, which depends on the number of photons reaching each pixel, you may also want to model factors like read noise and dark noise, which are independent of the number of photons. Render images that exhibit noise and show the effect of different types of it as exposure time varies. ➋ 5.8 Because they are based on floating-point addition, which is not associative, the AddSplat() methods implemented in this chapter do not live up to pbrt’s goal of producing deterministic output: if different threads add splats to the same pixel in a different order over multiple runs of pbrt, the final image may differ. An alternative implementation might allocate a separate buffer for each thread’s splats and then sum the buffers at the end of rendering, which would be deterministic but would incur a memory cost proportional to the number of threads. Either implement that approach or come up with another one to address this issue and implement it in pbrt. Measure the memory and performance overhead of your approach as well as how often the current implementation is non-deterministic. Is the current implementation defensible? ➌ 5.9 Image-based rendering is the general name for a set of techniques that use one or more images of a scene to synthesize new images from viewpoints different from the original ones. One such approach is light field rendering, where a set of images from a densely spaced set of positions is used—as described by Levoy and Hanrahan (1996) and Gortler et al. (1996). Read these two papers on light fields, and modify pbrt to directly generate light fields of scenes, without requiring that the renderer be run multiple times, once for each camera position. It will probably be necessary to write a specialized Camera, Sampler, and Film to do this. Also, write an interactive light field viewer that loads light fields generated by your implementation and that generates new views of the scene.

```text
                   PixelSensor 234
```

_________________ 1 “Camera-world space” and “rendering space” are non-standard names, though we are unaware of generally accepted names for those coordinate spaces. 2 A moving camera generally does not affect ray tracing performance, as rendering with one just causes different camera rays to be traced. Moving geometry requires larger bounding boxes to bound the motion of objects, which in turn reduces the effectiveness of acceleration structures. Thus, it is undesirable to make objects move that do not need to be in motion. 3 One inconvenience with pbrt’s custom dynamic dispatch approach is that the interface class cannot provide such functionality via default method implementations. It is not too much work to do so with an explicitly shared base class as is done here, however. 4 A typical size for pixels in digital cameras in 2022-era mobile phones is 1.8 microns per side. 5 Its original implementation is due to Anders Langlands and Luca Fascione and is based on the sensor model in Weta Digital’s PhysLight system, which is used in Weta’s Manuka renderer. 6 These reflectance measurements are courtesy of Danny Pascale and are used with permission.
