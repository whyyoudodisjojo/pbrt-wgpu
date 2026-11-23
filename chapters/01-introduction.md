# Chapter 1: Introduction

Rendering is the process of producing an image from the description of a 3D scene. Obviously, this is a broad task, and there are many ways to approach it. Physically based techniques attempt to simulate reality; that is, they use principles of physics to model the interaction of light and matter. While a physically based approach may seem to be the most obvious way to approach rendering, it has only been widely adopted in practice over the past 15 or so years.

This book describes pbrt, a physically based rendering system based on the ray-tracing algorithm. It is capable of rendering realistic images of complex scenes such as the one shown in Figure 1.1. (Other than a few exceptions in this chapter that are noted with their appearance, all the images in this book are rendered with pbrt.) Most computer graphics books present algorithms and theory, sometimes combined with snippets of code. In contrast, this book couples the theory with a complete implementation of a fully functional rendering system. Furthermore, the full source code of the system is available under an open-source license, and the full text of this book is freely available online at pbr-book.org/4ed, as of November 1, 2023. Further information, including example scenes and additional information about pbrt, can be found on the website, pbrt.org.

## 1.1 LITERATE PROGRAMMING

While creating the TEX typesetting system, Donald Knuth developed a new programming methodology based on a simple but revolutionary idea. To quote Knuth, “let us change our traditional attitude to the construction of programs: Instead of imagining that our main task is to instruct a computer what to do, let us concentrate rather on explaining to human beings what we want a computer to do.” He named this methodology literate programming. This book (including the chapter you are reading now) is a long literate program. This means that in the course of reading this book, you will read the full implementation of the pbrt rendering system, not just a high-level description of it.

Literate programs are written in a metalanguage that mixes a document formatting language (e.g., TEX or HTML) and a programming language (e.g., Rust). Two separate systems process the program: a “weaver” that transforms the literate program into a document suitable for typesetting and a “tangler” that produces source code suitable for compilation. Our literate programming system is homegrown, but it was heavily influenced by Norman Ramsey’s noweb system.

*Figure 1.1: A Scene Rendered by pbrt. The Kroken scene features complex geometry, materials, and light transport. Handling all of these effects well in a rendering system makes it possible to render photorealistic images like this one. This scene and many others can be downloaded from the pbrt website. (Scene courtesy of Angelo Ferretti.)*

The literate programming metalanguage provides two important features. The first is the ability to mix prose with source code. This feature puts the description of the program on equal footing with its actual source code, encouraging careful design and documentation. Second, the language provides mechanisms for presenting the program code to the reader in an order that is entirely different from the compiler input. Thus, the program can be described in a logical manner. Each named block of code is called a fragment, and each fragment can refer to other fragments by name.

As a simple example, consider a function InitGlobals() that is responsible for initializing all of a program’s global variables:1

```rust
   fn init_globals() {
       n_marbles = 25.7; shoe_size = 13; dielectric = true; }
```

Despite its brevity, this function is hard to understand without any context. Why, for example, can the variable nMarbles take on floating-point values? Just looking at the code, one would need to search through the entire program to see where each variable is declared and how it is used in order to understand its purpose and the meanings of its legal values. Although this structuring of the system is fine for a compiler, a human reader would much rather see the initialization code for each variable presented separately, near the code that declares and uses the variable.

In a literate program, one can instead write InitGlobals() like this: 〈Function Definitions〉 ≡ void InitGlobals() { 〈Initialize Global Variables 3〉 } This defines a fragment, called 〈Function Definitions〉, that contains the definition of the InitGlobals() function. The InitGlobals() function itself refers to another fragment, 〈Initialize Global Variables〉. Because the initialization fragment has not yet been defined, we do not know anything about this function except that it will presumably contain assignments to global variables. Just having the fragment name is just the right level of abstraction for now, since no variables have been declared yet. When we introduce the global variable shoeSize somewhere later in the program, we can then write

〈Initialize Global Variables〉 ≡ 3

```rust
   shoe_size = 13;
```

Here we have started to define the contents of 〈Initialize Global Variables〉. When the literate program is tangled into source code for compilation, the literate programming system will substitute the code shoeSize = 13; inside the definition of the InitGlobals() function.

Later in the text, we may define another global variable, dielectric, and we can append its initialization to the fragment: 〈Initialize Global Variables〉 +≡ 3

```rust
   dielectric = true;
```

The +≡ symbol after the fragment name shows that we have added to a previously defined fragment. When tangled, these three fragments turn into the code void InitGlobals() { // Initialize Global Variables shoeSize = 13; dielectric = true; } In this way, we can decompose complex functions into logically distinct parts, making them much easier to understand. For example, we can write a complicated function as a series of fragments: 〈Function Definitions〉 +≡ void complexFunc(int x, int y, double *values) { 〈Check validity of arguments〉 if (x < y) { 〈Swap x and y〉 } 〈Do precomputation before loop〉 〈Loop through and update values array〉 }

Again, the contents of each fragment are expanded inline in complexFunc() for compilation. In the document, we can introduce each fragment and its implementation in turn. This decomposition lets us present

code a few lines at a time, making it easier to understand. Another advantage of this style of programming is that by separating the function into logical fragments, each with a single and well-delineated purpose, each one can then be written, verified, or read independently. In general, we will try to make each fragment less than 10 lines long. In some sense, the literate programming system is just an enhanced macro substitution package tuned to the task of rearranging program source code. This may seem like a trivial change, but in fact literate programming is quite different from other ways of structuring software systems.

### 1.1.1 INDEXING AND CROSS-REFERENCING

The following features are designed to make the text easier to navigate. Indices in the page margins give page numbers where the functions, variables, and methods used on that page are defined. Indices at the end of the book collect all of these identifiers so that it’s possible to find definitions by name. The index of fragments, starting on page 1183, lists the pages where each fragment is defined and where it is used. An index of class names and their members follows, starting on page 1201, and an index of miscellaneous identifiers can be found on page 1213. Within the text, a defined fragment name is followed by a list of page numbers on which that fragment is used. For example, a hypothetical fragment definition such as 〈A fascinating fragment〉 ≡ 184, 690

nMarbles += .001;

indicates that this fragment is used on pages 184 and 690. Occasionally we elide fragments from the printed book that are either boilerplate code or substantially the same as other fragments; when these fragments are used, no page numbers will be listed. When a fragment is used inside another fragment, the page number on which it is first defined appears after the fragment name. For example, 〈Do something interesting〉 +≡ 500

InitializeSomethingInteresting(); 〈Do something else interesting 486〉

CleanUp();

indicates that the 〈Do something else interesting〉 fragment is defined on page 486.

## 1.2 PHOTOREALISTIC RENDERING AND THE RAY-

```text
    TRACING ALGORITHM
```

The goal of photorealistic rendering is to create an image of a 3D scene that is indistinguishable from a photograph of the same scene. Before we describe the rendering process, it is important to understand that in this context the word indistinguishable is imprecise because it involves a human observer, and different observers may perceive the same image differently. Although we will cover a few perceptual issues in this book, accounting for the precise characteristics of a given observer is a difficult and not fully solved problem. For the most part, we will be satisfied with an accurate simulation of the physics of light and its interaction with matter, relying on our understanding of display technology to present the best possible image to the viewer. Given this single-minded focus on realistic simulation of light, it seems prudent to ask: what is light? Perception through light is central to our very existence, and this simple question has thus occupied the minds of famous philosophers and physicists since the beginning of recorded time. The ancient Indian philosophical school of Vaisheshika (5th–6th century BC) viewed light as a collection of small particles traveling along rays at high velocity. In the fifth century BC, the Greek philosopher Empedocles postulated that a divine fire emerged from human eyes and combined with light rays from the sun to produce vision. Between the 18th and 19th century, polymaths such as Isaac Newton, Thomas Young, and Augustin-Jean Fresnel endorsed conflicting theories modeling light as the consequence of either wave or particle propagation. During the same time period, André-Marie Ampère, Joseph-Louis Lagrange, Carl Friedrich Gauß, and Michael Faraday investigated the relations between electricity and magnetism that culminated in a sudden and dramatic unification by James Clerk Maxwell into a combined theory that is now known as electromagnetism. Light is a wave-like manifestation in this framework: the motion of electrically charged particles such as electrons in a light bulb’s filament

produces a disturbance of a surrounding electric field that propagates away from the source. The electric oscillation also causes a secondary oscillation of the magnetic field, which in turn reinforces an oscillation of the electric field, and so on. The interplay of these two fields leads to a self-propagating wave that can travel extremely large distances: millions of light years, in the case of distant stars visible in a clear night sky. In the early 20th century, work by Max Planck, Max Born, Erwin Schrödinger, and Werner Heisenberg led to another substantial shift of our understanding: at a microscopic level, elementary properties like energy and momentum are quantized, which means that they can only exist as an integer multiple of a base amount that is known as a quantum. In the case of electromagnetic oscillations, this quantum is referred to as a photon. In this sense, our physical understanding has come full circle: once we turn to very small scales, light again betrays a particle-like behavior that coexists with its overall wave-like nature. How does our goal of simulating light to produce realistic images fit into all of this? Faced with this tower of increasingly advanced explanations, a fundamental question arises: how far must we climb this tower to attain photorealism? To our great fortune, the answer turns out to be “not far at all.” Waves comprising visible light are extremely small, measuring only a few hundred nanometers from crest to trough. The complex wave-like behavior of light appears at these small scales, but it is of little consequence when simulating objects at the scale of, say, centimeters or meters. This is excellent news, because detailed wave-level simulations of anything larger than a few micrometers are impractical: computer graphics would not exist in its current form if this level of detail was necessary to render images. Instead, we will mostly work with equations developed between the 16th and early 19th century that model light as particles that travel along rays. This leads to a more efficient computational approach based on a key operation known as ray tracing. Ray tracing is conceptually a simple algorithm; it is based on following the path of a ray of light through a scene as it interacts with and bounces off objects in an environment. Although there are many ways to write a ray tracer, all such systems simulate at least the following objects and phenomena:

Cameras: A camera model determines how and from where the scene is being viewed, including how an image of the scene is recorded on a sensor. Many rendering systems generate viewing rays starting at the camera that are then traced into the scene to determine which objects are visible at each pixel. Ray–object intersections: We must be able to tell precisely where a given ray intersects a given geometric object. In addition, we need to determine certain properties of the object at the intersection point, such as a surface normal or its material. Most ray tracers also have some facility for testing the intersection of a ray with multiple objects, typically returning the closest intersection along the ray. Light sources: Without lighting, there would be little point in rendering a scene. A ray tracer must model the distribution of light throughout the scene, including not only the locations of the lights themselves but also the way in which they distribute their energy throughout space. Visibility: In order to know whether a given light deposits energy at a point on a surface, we must know whether there is an uninterrupted path from the point to the light source. Fortunately, this question is easy to answer in a ray tracer, since we can just construct the ray from the surface to the light, find the closest ray– object intersection, and compare the intersection distance to the light distance. Light scattering at surfaces: Each object must provide a description of its appearance, including information about how light interacts with the object’s surface, as well as the nature of the reradiated (or scattered) light. Models for surface scattering are typically parameterized so that they can simulate a variety of appearances. Indirect light transport: Because light can arrive at a surface after bouncing off or passing through other surfaces, it is usually necessary to trace additional rays to capture this effect. Ray propagation: We need to know what happens to the light traveling along a ray as it passes through space. If we are rendering a scene in a vacuum, light energy remains constant along a ray. Although true vacuums are unusual on Earth, they are

a reasonable approximation for many environments. More sophisticated models are available for tracing rays through fog, smoke, the Earth’s atmosphere, and so on. We will briefly discuss each of these simulation tasks in this section. In the next section, we will show pbrt’s high-level interface to the underlying simulation components and will present a simple rendering algorithm that randomly samples light paths through a scene in order to generate images.

### 1.2.1 CAMERAS AND FILM

Nearly everyone has used a camera and is familiar with its basic functionality: you indicate your desire to record an image of the world (usually by pressing a button or tapping a screen), and the image is recorded onto a piece of film or by an electronic sensor.2 One of the simplest devices for taking photographs is called the pinhole camera. Pinhole cameras consist of a light-tight box with a tiny hole at one end (Figure 1.2). When the hole is uncovered, light enters and falls on a piece of photographic paper that is affixed to the other end of the box. Despite its simplicity, this kind of camera is still used today, mostly for artistic purposes. Long exposure times are necessary to get enough light on the film to form an image.

```text
                                                             art
         Figure 1.2: A Pinhole Camera. The viewing volume is determined by the projection of the film through
         the pinhole.
```

```text
                                                             art
         Figure 1.3: When we simulate a pinhole camera, we place the film in front of the hole at the imaging
         plane, and the hole is renamed the eye.
```

Although most cameras are substantially more complex than the pinhole camera, it is a convenient starting point for simulation. The most important function of the camera is to define the portion of the scene that will be recorded onto the film. In Figure 1.2, we can see how connecting the pinhole to the edges of the film creates a double pyramid that extends into the scene. Objects that are not inside this pyramid cannot be imaged onto the film. Because actual cameras image a more complex shape than a pyramid, we will refer to the region of space that can potentially be imaged onto the film as the viewing volume. Another way to think about the pinhole camera is to place the film plane in front of the pinhole but at the same distance (Figure 1.3). Note that connecting the hole to the film defines exactly the same viewing volume as before. Of course, this is not a practical way to build a real camera, but for simulation purposes it is a convenient abstraction. When the film (or image) plane is in front of the pinhole, the pinhole is frequently referred to as the eye. Now we come to the crucial issue in rendering: at each point in the image, what color does the camera record? The answer to this question is partially determined by what part of the scene is visible at that point. If we recall the original pinhole camera, it is clear that only light rays that travel along the vector between the pinhole and a point on the film can contribute to that film location. In our simulated camera with the film plane in front of the eye, we are interested in the amount of light traveling from the image point to the eye. Therefore, an important task of the camera simulator is to take a point on the image and generate rays along which incident light will contribute to that image location. Because a ray consists of an origin point and a direction vector, this task is particularly simple for the pinhole camera model of Figure 1.3: it uses the pinhole for the origin and the vector from the pinhole to the imaging plane as the ray’s direction. For more complex camera models involving multiple lenses, the calculation of the ray that corresponds to a given point on the image may be more involved. Light arriving at the camera along a ray will generally carry different amounts of energy at different wavelengths. The human visual system

interprets this wavelength variation as color. Most camera sensors record separate measurements for three wavelength distributions that correspond to red, green, and blue colors, which is sufficient to reconstruct a scene’s visual appearance to a human observer. (Section 4.6 discusses color in more detail.) Therefore, cameras in pbrt also include a film abstraction that both stores the image and models the film sensor’s response to incident light. pbrt’s camera and film abstraction is described in detail in Chapter 5. With the process of converting image locations to rays encapsulated in the camera module and with the film abstraction responsible for determining the sensor’s response to light, the rest of the rendering system can focus on evaluating the lighting along those rays.

### 1.2.2 RAY–OBJECT INTERSECTIONS

Each time the camera generates a ray, the first task of the renderer is to determine which object, if any, that ray intersects first and where the intersection occurs. This intersection point is the visible point along the ray, and we will want to simulate the interaction of light with the object at this point. To find the intersection, we must test the ray for intersection against all objects in the scene and select the one that the ray intersects first. Given a ray r, we first start by writing it in parametric form: r(t) = o + td, where o is the ray’s origin, d is its direction vector, and t is a parameter whose legal range is [0, ∞). We can obtain a point along the ray by specifying its parametric t value and evaluating the above equation. It is often easy to find the intersection between the ray r and a surface defined by an implicit function F (x, y, z) = 0. We first substitute the ray equation into the implicit equation, producing a new function whose only parameter is t. We then solve this function for t and substitute the smallest positive root into the ray equation to find the desired point. For example, the implicit equation of a sphere centered at the origin with radius r is x2 + y2 + z2 − r2 = 0. Substituting the ray equation, we have (ox + tdx)2 + (oy + tdy)2 + (oz + tdz)2 − r2 = 0, where subscripts denote the corresponding component of a point or vector. For a given ray and a given sphere, all the values besides t are known, giving us an easily solved quadratic equation in t. If there are no

real roots, the ray misses the sphere; if there are roots, the smallest positive one gives the intersection point. The intersection point alone is not enough information for the rest of the ray tracer; it needs to know certain properties of the surface at the point. First, a representation of the material at the point must be determined and passed along to later stages of the ray-tracing algorithm.

*Figure 1.4: Moana Island Scene, Rendered by pbrt. This model from a feature film exhibits the extreme complexity of scenes rendered for movies (Walt Disney Animation Studios 2018). It features over 146 million unique triangles, though the true geometric complexity of the scene is well into the tens of billions of triangles due to extensive use of object instancing. (Scene courtesy of Walt Disney Animation Studios.)*

Second, additional geometric information about the intersection point will also be required in order to shade the point. For example, the surface normal n is always required. Although many ray tracers operate with only n, more sophisticated rendering systems like pbrt require even more information, such as various partial derivatives of position and surface normal with respect to the local parameterization of the surface. Of course, most scenes are made up of multiple objects. The brute-force approach would be to test the ray against each object in turn, choosing the minimum positive t value of all intersections to find the closest intersection. This approach, while correct, is very slow, even for scenes of modest complexity. A better approach is to incorporate an acceleration structure that quickly rejects whole groups of objects during the ray intersection

process. This ability to quickly cull irrelevant geometry means that ray tracing frequently runs in O(m log n) time, where m is the number of pixels in the image and n is the number of objects in the scene.3 (Building the acceleration structure itself is necessarily at least O(n) time, however.) Thanks to the effectiveness of acceleration structures, it is possible to render highly complex scenes like the one shown in Figure 1.4 in reasonable amounts of time. pbrt’s geometric interface and implementations of it for a variety of shapes are described in Chapter 6, and the acceleration interface and implementations are shown in Chapter 7.

### 1.2.3 LIGHT DISTRIBUTION

The ray–object intersection stage gives us a point to be shaded and some information about the local geometry at that point. Recall that our eventual goal is to find the amount of light leaving this point in the direction of the camera. To do this, we need to know how much light is arriving at this point. This involves both the geometric and radiometric distribution of light in the scene. For very simple light sources (e.g., point lights), the geometric distribution of lighting is a simple matter of knowing the position of the lights. However, point lights do not exist in the real world, and so physically based lighting is often based on area light sources. This means that the light source is associated with a geometric object that emits illumination from its surface. However, we will use point lights in this section to illustrate the components of light distribution; a more rigorous discussion of light measurement and distribution is the topic of Chapters 4 and 12.

```text
                                                             art
         Figure 1.5: Geometric construction for determining the power per area arriving at a point p due to a point
         light source. The distance from the point to the light source is denoted by r.
```

```text
                                                             art
       Figure 1.6: Since the point light radiates light equally in all directions, the same total power is deposited
       on all spheres centered at the light.
```

We frequently would like to know the amount of light power being deposited on the differential area surrounding the intersection point p (Figure 1.5). We will assume that the point light source has some power Φ associated with it and that it radiates light equally in all directions. This means that the power per area on a unit sphere surrounding the light is Φ/(4π). (These measurements will be explained and formalized in Section 4.1.) If we consider two such spheres (Figure 1.6), it is clear that the power per area at a point on the larger sphere must be less than the power at a point on the smaller sphere because the same total power is distributed over a larger area. Specifically, the power per area arriving at a point on a sphere of radius r is proportional to 1/r2. Furthermore, it can be shown that if the tiny surface patch dA is tilted by an angle θ away from the vector from the surface point to the light, the amount of power deposited on dA is proportional to cos θ. Putting this all together, the differential power per area dE (the differential irradiance) is art

```text
                                                             art
       Figure 1.7: Scene with Thousands of Light Sources. This scene has far too many lights to consider all
       of them at each point where the reflected light is computed. Nevertheless, it can be rendered efficiently
       using stochastic sampling of light sources. (Scene courtesy of Beeple.)
```

Readers already familiar with basic lighting in computer graphics will notice two familiar laws encoded in this equation: the cosine falloff of light for tilted surfaces mentioned above, and the one-over-r-squared falloff of light with distance. Scenes with multiple lights are easily handled because illumination is linear: the contribution of each light can be computed separately and summed to obtain the overall contribution. An implication of the linearity

of light is that sophisticated algorithms can be applied to randomly sample lighting from only some of the light sources at each shaded point in the scene; this is the topic of Section 12.6. Figure 1.7 shows a scene with thousands of light sources rendered in this way.

### 1.2.4 VISIBILITY

The lighting distribution described in the previous section ignores one very important component: shadows. Each light contributes illumination to the point being shaded only if the path from the point to the light’s position is unobstructed (Figure 1.8). Fortunately, in a ray tracer it is easy to determine if the light is visible from the point being shaded. We simply construct a new ray whose origin is at the surface point and whose direction points toward the light. These special rays are called shadow rays. If we trace this ray through the environment, we can check to see whether any intersections are found between the ray’s origin and the light source by comparing the parametric t value of any intersections found to the parametric t value along the ray of the light source position. If there is no blocking object between the light and the surface, the light’s contribution is included.

### 1.2.5 LIGHT SCATTERING AT SURFACES

We are now able to compute two pieces of information that are vital for proper shading of a point: its location and the incident lighting. Now we need to determine how the incident lighting is scattered at the surface. Specifically, we are interested in the amount of light energy scattered back along the ray that we originally traced to find the intersection point, since that ray leads to the camera (Figure 1.9).

```text
                                                                art
          Figure 1.8: A light source only deposits energy on a surface if the source is not obscured as seen from the
          receiving point. The light source on the left illuminates the point p, but the light source on the right does
          not.
```

```text
                                                               art
         Figure 1.9: The Geometry of Surface Scattering. Incident light arriving along direction ωi interacts
         with the surface at point p and is scattered back toward the camera along direction ωo. The amount of
         light scattered toward the camera is given by the product of the incident light energy and the BRDF.
```

Each object in the scene provides a material, which is a description of its appearance properties at each point on the surface. This description is given by the bidirectional reflectance distribution function (BRDF). This function tells us how much energy is reflected from an incoming direction ωi to an outgoing direction ωo. We will write the BRDF at p as fr(p, ωo, ωi). (By convention, directions ω are unit vectors.) It is easy to generalize the notion of a BRDF to transmitted light (obtaining a BTDF) or to general scattering of light arriving from either side of the surface. A function that describes general scattering is called a bidirectional scattering distribution function (BSDF). pbrt supports a variety of BSDF models; they are described in Chapter 9. More complex yet is the bidirectional scattering surface reflectance distribution function (BSSRDF), which models light that exits a surface at a different point than it enters. This is necessary to reproduce translucent materials such as milk, marble, or skin. The BSSRDF is described in Section 4.3.2. Figure 1.10 shows an image rendered by pbrt based on a model of a human head where scattering from the skin is modeled using a BSSRDF.

```text
                                                               art
         Figure 1.10: Head with Scattering Modeled Using a BSSRDF. Accurately modeling subsurface light
         transport rather than assuming that light exits the surface at the same point it entered greatly improves the
         realism of the rendered image. (Model courtesy of Infinite Realities, Inc.)
```

### 1.2.6 INDIRECT LIGHT TRANSPORT

Turner Whitted’s original paper on ray tracing (1980) emphasized its recursive nature, which was the key that made it possible to include indirect

specular reflection and transmission in rendered images. For example, if a ray from the camera hits a shiny object like a mirror, we can reflect the ray about the surface normal at the intersection point and recursively invoke the ray-tracing routine to find the light arriving at the point on the mirror, adding its contribution to the original camera ray. This same technique can be used to trace transmitted rays that intersect transparent objects. Many early ray-tracing examples showcased mirrors and glass balls (Figure 1.11) because these types of effects were difficult to capture with other rendering techniques. In general, the amount of light that reaches the camera from a point on an object is given by the sum of light emitted by the object (if it is itself a light source) and the amount of reflected light. This idea is formalized by the light transport equation (also often known as the rendering equation), which measures light with respect to radiance, a radiometric unit that will be defined in Section 4.1. It says that the outgoing radiance Lo(p, ωo) from a point p in direction ωo is the emitted radiance at that point in that direction, Le(p, ωo), plus the incident radiance from all directions on the sphere S2 around p scaled by the BSDF f (p, ωo, ωi) and a cosine term: art We will show a more complete derivation of this equation in Sections 4.3.1 and 13.1.1. Solving this integral analytically is not possible except for the simplest of scenes, so we must either make simplifying assumptions or use numerical integration techniques.

*Figure 1.11: A Prototypical Early Ray Tracing Scene. Note the use of mirrored and glass objects, which emphasizes the algorithm’s ability to handle these kinds of surfaces. (a) Rendered using Whitted’s original ray-tracing algorithm from 1980, and (b) rendered using stochastic progressive photon mapping (SPPM), a modern advanced light transport algorithm. SPPM is able to accurately simulate the focusing of light that passes through the spheres.*

Whitted’s ray-tracing algorithm simplifies this integral by ignoring incoming light from most directions and only evaluating Li(p, ωi) for directions to light sources and for the directions of perfect reflection and refraction. In other words, it turns the integral into a sum over a small number of directions. In Section 1.3.6, we will see that simple random sampling of Equation (1.1) can create realistic images that include both

complex lighting and complex surface scattering effects. Throughout the remainder of the book, we will show how using more sophisticated random sampling algorithms greatly improves the efficiency of this general approach.

### 1.2.7 RAY PROPAGATION

The discussion so far has assumed that rays are traveling through a vacuum. For example, when describing the distribution of light from a point source, we assumed that the light’s power was distributed equally on the surface of a sphere centered at the light without decreasing along the way. The presence of participating media such as smoke, fog, or dust can invalidate this assumption. These effects are important to simulate: a wide class of interesting phenomena can be described using participating media. Figure 1.12 shows an explosion rendered by pbrt. Less dramatically, almost all outdoor scenes are affected substantially by participating media. For example, Earth’s atmosphere causes objects that are farther away to appear less saturated.

```text
                                                             art
         Figure 1.12: Explosion Modeled Using Participating Media. Because pbrt is capable of simulating
         light emission, scattering, and absorption in detailed models of participating media, it is capable of
         rendering images like this one. (Scene courtesy of Jim Price.)
```

There are two ways in which a participating medium can affect the light propagating along a ray. First, the medium can extinguish (or attenuate) light, either by absorbing it or by scattering it in a different direction. We can capture this effect by computing the transmittance Tr between the ray origin and the intersection point. The transmittance tells us how much of the light scattered at the intersection point makes it back to the ray origin. A participating medium can also add to the light along a ray. This can happen either if the medium emits light (as with a flame) or if the medium scatters light from other directions back along the ray. We can find this quantity by numerically evaluating the volume light transport equation, in the same way we evaluated the light transport equation to find the amount

of light reflected from a surface. We will leave the description of participating media and volume rendering until Chapters 11 and 14.

## 1.3 pbrt: SYSTEM OVERVIEW

pbrt is structured using standard object-oriented techniques: for each of a number of fundamental types, the system specifies an interface that implementations of that type must fulfill. For example, pbrt requires the implementation of a particular shape that represents geometry in a scene to provide a set of methods including one that returns the shape’s bounding box, and another that tests for intersection with a given ray. In turn, the majority of the system can be implemented purely in terms of those interfaces; for example, the code that checks for occluding objects between a light source and a point being shaded calls the shape intersection methods without needing to consider which particular types of shapes are present in the scene. There are a total of 14 of these key base types, summarized in Table 1.1. Adding a new implementation of one of these types to the system is straightforward; the implementation must provide the required methods, it must be compiled and linked into the executable, and the scene object creation routines must be modified to create instances of the object as needed as the scene description file is parsed. Section C.4 discusses extending the system in more detail. Table 1.1: Main Interface Types. Most of pbrt is implemented in terms of 14 key base types, listed here. Implementations of each of these can easily be added to the system to extend its functionality.

Base type Source Files Section Spectrum base/spectrum.h, util/spectrum.{h,cpp} 4.5 Camera base/camera.h, cameras.{h,cpp} 5.1 Shape base/shape.h, shapes.{h,cpp} 6.1 Primitive cpu/{primitive,accelerators}.{h,cpp} 7.1 Sampler base/sampler.h, samplers.{h,cpp} 8.3 Filter base/filter.h, filters.{h,cpp} 8.8.1 BxDF base/bxdf.h, bxdfs.{h,cpp} 9.1.2 Material base/material.h, materials.{h,cpp} 10.5 FloatTexture SpectrumTexture base/texture.h, textures.{h,cpp} 10.3 Medium base/medium.h, media.{h,cpp} 11.4

Light base/light.h, lights.{h,cpp} 12.1 LightSampler base/lightsampler.h, lightsamplers.{h,cpp} 12.6 Integrator cpu/integrators.{h,cpp} 1.3.3

```text
   BxDF 538
   Camera 206
   Filter 515
   FloatTexture 656
   Integrator 22
   Light 740
   LightSampler 781
   Material 674
   Medium 714
   Primitive 398
   Sampler 469
   Shape 261
   Spectrum 165
   SpectrumTexture 656
```

Conventional practice in Rust would be to specify the interfaces for each of these types using abstract base classes that define pure virtual functions and to have implementations inherit from those base classes and implement the required virtual functions. In turn, the compiler would take care of generating the code that calls the appropriate method, given a pointer to any object of the base class type. That approach was used in the three previous versions of pbrt, but the addition of support for rendering on graphics processing units (GPUs) in this version motivated a more portable approach based on tag-based dispatch, where each specific type implementation is assigned a unique integer that determines its type at runtime. (See Section 1.5.7 for more information about this topic.) The polymorphic types that are implemented in this way in pbrt are all defined in header files in the base/ directory. This version of pbrt is capable of running on GPUs that support Rust and provide APIs for ray intersection tests.4 We have carefully designed the system so that almost all of pbrt’s implementation runs on both CPUs and GPUs, just as it is presented in Chapters 2 through 12. We will therefore generally say little about the CPU versus the GPU in most of the following.

The main differences between the CPU and GPU rendering paths in pbrt are in their data flow and how they are parallelized—effectively, how the pieces are connected together. Both the basic rendering algorithm described later in this chapter and the light transport algorithms described in Chapters 13 and 14 are only available on the CPU. The GPU rendering pipeline is discussed in Chapter 15, though it, too, is also capable of running on the CPU (not as efficiently as the CPU-targeted light transport algorithms, however). While pbrt can render many scenes well with its current implementation, it has frequently been extended by students, researchers, and developers. Throughout this section are a number of notable images from those efforts. Figures 1.13, 1.14, and 1.15 were each created by students in a rendering course where the final class project was to extend pbrt with new functionality in order to render an image that it could not have rendered before. These images are among the best from that course.

### 1.3.1 PHASES OF EXECUTION

pbrt can be conceptually divided into three phases of execution. First, it parses the scene description file provided by the user. The scene description is a text file that specifies the geometric shapes that make up the scene, their material properties, the lights that illuminate them, where the virtual camera is positioned in the scene, and parameters to all the individual algorithms used throughout the system. The scene file format is documented on the pbrt website, pbrt.org.

The result of the parsing phase is an instance of the BasicScene class, which stores the scene specification, but not in a form yet suitable for rendering. In the second phase of execution, pbrt creates specific objects corresponding to the scene; for example, if a perspective projection has been specified, it is in this phase that a PerspectiveCamera object corresponding to the specified viewing parameters is created. Previous versions of pbrt intermixed these first two phases, but for this version we have separated them because the CPU and GPU rendering paths differ in some of the ways that they represent the scene in memory.

```text
    BasicScene 1134
```

```text
    PerspectiveCamera 220
```

```text
                                                             art
         Figure 1.13: Guillaume Poncin and Pramod Sharma extended pbrt in numerous ways, implementing a
         number of complex rendering algorithms, to make this prize-winning image for Stanford’s CS348b
         rendering competition. The trees are modeled procedurally with L-systems, a glow image processing filter
         increases the apparent realism of the lights on the tree, snow was modeled procedurally with metaballs,
         and a subsurface scattering algorithm gave the snow its realistic appearance by accounting for the effect
         of light that travels beneath the snow for some distance before leaving it.
```

In the third phase, the main rendering loop executes. This phase is where pbrt usually spends the majority of its running time, and most of this book is devoted to code that executes during this phase. To orchestrate the rendering, pbrt implements an integrator, so-named because its main task is to evaluate the integral in Equation (1.1).

### 1.3.2 pbrt’S main() FUNCTION

The main() function for the pbrt executable is defined in the file cmd/pbrt.cpp in the directory that holds the pbrt source code, src/pbrt in the pbrt distribution. It is only a hundred and fifty or so lines of code, much of it devoted to processing command-line arguments and related bookkeeping. 〈main program〉 ≡ int main(int argc, char *argv[]) { 〈Convert command-line arguments to vector of strings 19〉 〈Declare variables for parsed command line 19〉 〈Process command-line arguments〉 〈Initialize pbrt 20〉 〈Parse provided scene description files 20〉 〈Render the scene 21〉 〈Clean up after rendering the scene 21〉 }

Rather than operate on the argv values provided to the main() function directly, pbrt converts the provided arguments to a vector of std::strings. It does so not only for the greater convenience of the string class, but also to support non-ASCII character sets. Section B.3.2 has more information about character encodings and how they are handled in pbrt.

〈Convert command-line arguments to vector of strings〉 ≡ 18

std::vector<std::string> args = GetCommandLineArguments(argv);

```text
                                                            art
        Figure 1.14: Abe Davis, David Jacobs, and Jongmin Baek rendered this amazing image of an ice cave to
        take the grand prize in the 2009 Stanford CS348b rendering competition. They first implemented a
        simulation of the physical process of glaciation, the process where snow falls, melts, and refreezes over
        the course of many years, forming stratified layers of ice. They then simulated erosion of the ice due to
        melted water runoff before generating a geometric model of the ice. Scattering of light inside the volume
        was simulated with volumetric photon mapping; the blue color of the ice is entirely due to modeling the
        wavelength-dependent absorption of light in the ice volume.
```

We will only include the definitions of some of the main function’s fragments in the book text here. Some, such as the one that handles parsing command-line arguments provided by the user, are both simple enough and long enough that they are not worth the few pages that they would add to the book’s length. However, we will include the fragment that declares the variables in which the option values are stored.

```text
   GetCommandLineArguments() 1063
   PBRTOptions 1032
```

〈Declare variables for parsed command line〉 ≡ 18

PBRTOptions options; std::vector<std::string> filenames;

```text
                                                          art
        Figure 1.15: Chenlin Meng, Hubert Teo, and Jiren Zhu rendered this tasty-looking image of cotton candy
        in a teacup to win the grand prize in the 2018 Stanford CS348b rendering competition. They modeled the
        cotton candy with multiple layers of curves and then filled the center with a participating medium to
        efficiently model scattering in its interior.
```

The GetCommandLineArguments() function and PBRTOptions type appear in a mini-index in the page margin, along with the number of the page where they are defined. The mini-indices have pointers to the definitions of almost all the functions, classes, methods, and member variables used or referred to on each page. (In the interests of brevity, we will omit very widely used classes such as Ray from the mini-indices, as well as types or methods that were just introduced in the preceding few pages.) The PBRTOptions class stores various rendering options that are generally more suited to be specified on the command line rather than in scene description files—for example, how chatty pbrt should be about its progress during rendering. It is passed to the InitPBRT() function, which aggregates the various system-wide initialization tasks that must be performed before any other work is done. For example, it initializes the logging system and launches a group of threads that are used for the parallelization of pbrt. 〈Initialize pbrt〉 ≡ 18

```rust
   init_pbrt(options);
```

After the arguments have been parsed and validated, the ParseFiles() function takes over to handle the first of the three phases of execution described earlier. With the assistance of two classes, BasicSceneBuilder and BasicScene, which are respectively described in Sections C.2 and C.3, it loops over the provided filenames, parsing each file in turn. If pbrt is run with no filenames provided, it looks for the scene description from standard input. The mechanics of tokenizing and parsing scene description files will not be described in this book, but the parser implementation can be found in the files parser.h and parser.cpp in the src/pbrt directory. 〈Parse provided scene description files〉 ≡ 18

```rust
   let mut scene = BasicScene::default();
   let mut builder = BasicSceneBuilder::new(&mut scene);
   parse_files(&mut builder, &filenames);
```

```text
   BasicScene 1134
   BasicSceneBuilder 1123
   GetCommandLineArguments() 1063
   InitPBRT() 1032
   ParseFiles() 1120
   PBRTOptions 1032
   RenderWavefront() 927
```

After the scene description has been parsed, one of two functions is called to render the scene. RenderWavefront() supports both the CPU and GPU rendering paths, processing a million or so image samples in parallel. It is the topic of Chapter 15. RenderCPU() renders the scene using an Integrator implementation and is only available when running on the CPU. It uses much less parallelism than RenderWavefront(), rendering only as many image samples as there are CPU threads in parallel.

```text
                                                             art
        Figure 1.16: Martin Lubich modeled this scene of the Austrian Imperial Crown using Blender; it was
        originally rendered using LuxRender, which started out as a fork of the pbrt-v1 codebase. The crown
        consists of approximately 3.5 million triangles that are illuminated by six area light sources with emission
        spectra based on measured data from a real-world light source. It was originally rendered with 1280
        samples per pixel in 73 hours of computation on a quad-core CPU. On a modern GPU, pbrt renders this
        scene at the same sampling rate in 184 seconds.
```

Both of these functions start by converting the BasicScene into a form suitable for efficient rendering and then pass control to a processor-specific integrator. (More information about this process is available in Section C.3.) We will for now gloss past the details of this transformation in order to focus on the main rendering loop in RenderCPU(), which is much more interesting. For that, we will take the efficient scene representation as a given. 〈Render the scene〉 ≡ 18

if (Options->useGPU || Options->wavefront) RenderWavefront(scene); else RenderCPU(scene);

```text
    BasicPBRTOptions::useGPU 1031
    BasicPBRTOptions::wavefront 1031
    BasicScene 1134
    CleanupPBRT() 1032
    InitPBRT() 1032
    Integrator 22
    Options 1032
    RenderCPU() 20
    RenderWavefront() 927
```

After the image has been rendered, CleanupPBRT() takes care of shutting the system down gracefully, including, for example, terminating the threads launched by InitPBRT().

〈Clean up after rendering the scene〉 ≡ 18

CleanupPBRT();

### 1.3.3 INTEGRATOR INTERFACE

In the RenderCPU() rendering path, an instance of a class that implements the Integrator interface is responsible for rendering. Because Integrator implementations only run on the CPU, we will define Integrator as a standard base class with pure virtual methods. Integrator and the various implementations are each defined in the files cpu/integrator.h and cpu/integrator.cpp.

〈Integrator Definition〉 ≡ class Integrator { public: 〈Integrator Public Methods 23〉 〈Integrator Public Members 22〉 protected: 〈Integrator Protected Methods 22〉 };

The base Integrator constructor takes a single Primitive that represents all the geometric objects in the scene as well as an array that holds all the lights in the scene. 〈Integrator Protected Methods〉 ≡ 22

```rust
   fn new(aggregate: Primitive, lights: Vec<Light>) -> Self {
       Self { aggregate, lights }
```

```text
        〈Integrator constructor implementation 23〉
   }
```

Each geometric object in the scene is represented by a Primitive, which is primarily responsible for combining a Shape that specifies its geometry and a Material that describes its appearance (e.g., the object’s color, or whether it has a dull or glossy finish). In turn, all the geometric primitives in a scene are collected into a single aggregate primitive that is stored in the Integrator::aggregate member variable. This aggregate is a special kind of primitive that itself holds references to many other primitives. The aggregate implementation stores all the scene’s primitives in an acceleration data structure that reduces the number of unnecessary ray intersection tests with primitives that are far away from a given ray. Because it implements the Primitive interface, it appears no different from a single primitive to the rest of the system. 〈Integrator Public Members〉 ≡ 22

```rust
   aggregate: Primitive,
   lights: Vec<Light>,
```

Each light source in the scene is represented by an object that implements the Light interface, which allows the light to specify its shape and the distribution of energy that it emits. Some lights need to know the bounding box of the entire scene, which is unavailable when they are first created. Therefore, the Integrator constructor calls their Preprocess() methods, providing those bounds. At this point any “infinite” lights are also stored in a separate array. This sort of light, which will be introduced in Section 12.5, models infinitely far away sources of light, which is a reasonable model for skylight as received on Earth’s surface, for example. Sometimes it will be necessary to loop over just those lights, and for scenes with thousands of light sources it would be inefficient to loop over all of them just to find those.

```text
   Integrator 22
   Integrator::aggregate 22
   Light 740
   Material 674
   Primitive 398
   RenderCPU() 20
   Shape 261
```

〈Integrator constructor implementation〉 ≡ 22

```rust
   let scene_bounds = if aggregate.is_some() { aggregate.unwrap().bounds() } else { Bounds3f::default() };
   for light in &mut lights {
       light.preprocess(scene_bounds);
       if light.r#type() == LightType::Infinite {
           infinite_lights.push(light.clone());
       }
   }
```

〈Integrator Public Members〉 +≡ 22

```rust
   infinite_lights: Vec<Light>,
```

Integrators must provide an implementation of the Render() method, which takes no further arguments. This method is called by the RenderCPU() function once the scene representation has been initialized. The task of integrators is to render the scene as specified by the aggregate and the lights. Beyond that, it is up to the specific integrator to define what it means to render the scene, using whichever other classes that it needs to do so (e.g., a camera model). This interface is intentionally very general to permit a wide range of implementations—for example, one could implement an Integrator that measures light only at a sparse set of points distributed through the scene rather than generating a regular 2D image. 〈Integrator Public Methods〉 ≡ 22

```rust
   fn render(&self);
```

The Integrator class provides two methods related to ray–primitive intersection for use of its subclasses. Intersect() takes a ray and a maximum parametric distance tMax, traces the given ray into the scene, and returns a ShapeIntersection object corresponding to the closest primitive that the ray hit, if there is an intersection along the ray before tMax. (The ShapeIntersection structure is defined in Section 6.1.3.) One thing to note is that this method uses the type pstd::optional for the return value rather than std::optional from the Rust standard library; we have reimplemented parts of the standard library in the pstd namespace for reasons that are discussed in Section 1.5.5. 〈Integrator Method Definitions〉 ≡ pstd::optional<ShapeIntersection> Integrator::Intersect(const Ray &ray, Float tMax) const { if (aggregate) return aggregate.Intersect(ray, tMax); else return {}; }

```text
   Bounds3f 97
   Float 23
   Integrator 22
   Integrator::aggregate 22
   Integrator::infiniteLights 23
   Integrator::IntersectP() 24
   Integrator::lights 22
   Light 740
   Light::Preprocess() 743
   Light::Type() 740
   LightType 740
   LightType::Infinite 740
   Primitive::Bounds() 398
   Primitive::Intersect() 398
   Ray 95
   RenderCPU() 20
   ShapeIntersection 266
```

Also note the capitalized floating-point type Float in Intersect()’s signature: almost all floating-point values in pbrt are declared as Floats. (The only exceptions are a few cases where a 32-bit float or a 64-bit double is specifically needed (e.g., when saving binary values to files).) Depending on the compilation flags of pbrt, Float is an alias for either float or double, though single precision float is almost always sufficient in practice. The definition of Float is in the pbrt.h header file, which is included by all other source files in pbrt. 〈Float Type Definitions〉 ≡ #ifdef PBRT_FLOAT_AS_DOUBLE using Float = double; #else using Float = float; #endif

Integrator::IntersectP() is closely related to the Intersect() method. It checks for the existence of intersections along the ray but only returns a Boolean indicating whether an intersection was found. (The “P” in its name indicates that it is a function that evaluates a predicate, using a common naming convention from the Lisp programming language.) Because it does not need to search for the closest intersection or return additional geometric information about intersections, IntersectP() is

generally more efficient than Integrator::Intersect(). This routine is used for shadow rays. 〈Integrator Method Definitions〉 +≡ bool Integrator::IntersectP(const Ray &ray, Float tMax) const { if (aggregate) return aggregate.IntersectP(ray, tMax); else return false; }

### 1.3.4 ImageTileIntegrator AND THE MAIN RENDERING LOOP

Before implementing a basic integrator that simulates light transport to render an image, we will define two Integrator subclasses that provide additional common functionality used by that integrator as well as many of the integrator implementations to come. We start with ImageTileIntegrator, which inherits from Integrator. The next section defines RayIntegrator, which inherits from ImageTileIntegrator.

All of pbrt’s CPU-based integrators render images using a camera model to define the viewing parameters, and all parallelize rendering by splitting the image into tiles and having different processors work on different tiles. Therefore, pbrt includes an ImageTileIntegrator that provides common functionality for those tasks. 〈ImageTileIntegrator Definition〉 ≡ class ImageTileIntegrator : public Integrator { public: 〈ImageTileIntegrator Public Methods 24〉 protected: 〈ImageTileIntegrator Protected Members 25〉 };

In addition to the aggregate and the lights, the ImageTileIntegrator constructor takes a Camera that specifies the viewing and lens parameters such as position, orientation, focus, and field of view. Film stored by the camera handles image storage. The Camera classes are the subject of most of Chapter 5, and Film is described in Section 5.4. The Film is responsible for writing the final image to a file.

```text
   Camera 206
   Film 244
   Float 23
   ImageTileIntegrator 24
   ImageTileIntegrator::camera 25
   ImageTileIntegrator:: samplerPrototype 25
   Integrator 22
   Integrator::aggregate 22
   Integrator::Intersect() 23
   Light 740
   Primitive 398
   Primitive::IntersectP() 398
   Ray 95
   RayIntegrator 28
   Sampler 469
```

The constructor also takes a Sampler; its role is more subtle, but its implementation can substantially affect the quality of the images that the system generates. First, the sampler is responsible for choosing the points on the image plane that determine which rays are initially traced into the scene. Second, it is responsible for supplying random sample values that are used by integrators for estimating the value of the light transport integral, Equation (1.1). For example, some integrators need to choose random points on light sources to compute illumination from area lights. Generating a good distribution of these samples is an important part of the rendering process that can substantially affect overall efficiency; this topic is the main focus of Chapter 8. 〈ImageTileIntegrator Public Methods〉 ≡ 24

ImageTileIntegrator(Camera camera, Sampler sampler, Primitive aggregate, std::vector<Light> lights) : Integrator(aggregate, lights), camera(camera), samplerPrototype(sampler) {}

〈ImageTileIntegrator Protected Members〉 ≡ 24

Camera camera; Sampler samplerPrototype;

For all of pbrt’s integrators, the final color computed at each pixel is based on random sampling algorithms. If each pixel’s final value is computed as the average of multiple samples, then the quality of the image improves. At low numbers of samples, sampling error manifests itself as grainy high-frequency noise in images, though error goes down at a predictable rate as the number of samples increases. (This topic is discussed in more depth in

Section 2.1.4.) ImageTileIntegrator::Render() therefore renders the image in waves of a few samples per pixel. For the first two waves, only a single sample is taken in each pixel. In the next wave, two samples are taken, with the number of samples doubling after each wave up to a limit. While it makes no difference to the final image if the image was rendered in waves or with all the samples being taken in a pixel before moving on to the next one, this organization of the computation means that it is possible to see previews of the final image during rendering where all pixels have some samples, rather than a few pixels having many samples and the rest having none. Because pbrt is parallelized to run using multiple threads, there is a balance to be struck with this approach. There is a cost for threads to acquire work for a new image tile, and some threads end up idle at the end of each wave once there is no more work for them to do but other threads are still working on the tiles they have been assigned. These considerations motivated the capped doubling approach. 〈ImageTileIntegrator Method Definitions〉 ≡ void ImageTileIntegrator::Render() { 〈Declare common variables for rendering image in tiles 25〉 〈Render image in waves 26〉 }

Before rendering begins, a few additional variables are required. First, the integrator implementations will need to allocate small amounts of temporary memory to store surface scattering properties in the course of computing each ray’s contribution. The large number of resulting allocations could easily overwhelm the system’s regular memory allocation routines (e.g., new), which must coordinate multi-threaded maintenance of elaborate data structures to track free memory. A naive implementation could potentially spend a fairly large fraction of its computation time in the memory allocator. To address this issue, pbrt provides a ScratchBuffer class that manages a small preallocated buffer of memory. ScratchBuffer allocations are very efficient, just requiring the increment of an offset. The ScratchBuffer

does not allow independently freeing allocations; instead, all must be freed at once, but doing so only requires resetting that offset. Because ScratchBuffers are not safe for use by multiple threads at the same time, an individual one is created for each thread using the ThreadLocal template class. Its constructor takes a lambda function that returns a fresh instance of the object of the type it manages; here, calling the default ScratchBuffer constructor is sufficient. ThreadLocal then handles the details of maintaining distinct copies of the object for each thread, allocating them on demand. 〈Declare common variables for rendering image in tiles〉 ≡ 25

ThreadLocal<ScratchBuffer> scratchBuffers( []() { return ScratchBuffer(); } );

```text
   Camera 206
   Sampler 469
   ScratchBuffer 1078
   ThreadLocal 1112
```

Most Sampler implementations find it useful to maintain some state, such as the coordinates of the current pixel. This means that multiple threads cannot use a single Sampler concurrently and ThreadLocal is also used for Sampler management. Samplers provide a Clone() method that creates a new instance of their sampler type. The Sampler first provided to the ImageTileIntegrator constructor, samplerPrototype, provides those copies here. 〈Declare common variables for rendering image in tiles〉 +≡ 25

ThreadLocal<Sampler> samplers( [this]() { return samplerPrototype.Clone(); });

It is helpful to provide the user with an indication of how much of the rendering work is done and an estimate of how much longer it will take. This task is handled by the ProgressReporter class, which takes as its first parameter the total number of items of work. Here, the total amount of work is the number of samples taken in each pixel times the total number of pixels. It is important to use 64-bit precision to compute this value, since a

32-bit int may be insufficient for high-resolution images with many samples per pixel. 〈Declare common variables for rendering image in tiles〉 +≡ 25

Bounds2i pixelBounds = camera.GetFilm().PixelBounds(); int spp = samplerPrototype.SamplesPerPixel(); ProgressReporter progress(int64_t(spp) * pixelBounds.Area(), “Rendering”, Options->quiet);

In the following, the range of samples to be taken in the current wave is given by waveStart and waveEnd; nextWaveSize gives the number of samples to be taken in the next wave. 〈Declare common variables for rendering image in tiles〉 +≡ 25

int waveStart = 0, waveEnd = 1, nextWaveSize = 1;

With these variables in hand, rendering proceeds until the required number of samples have been taken in all pixels. 〈Render image in waves〉 ≡ 25

while (waveStart < spp) { 〈Render current wave’s image tiles in parallel 27〉 〈Update start and end wave 28〉 〈Optionally write current image to disk〉 }

```text
   BasicPBRTOptions::quiet 1031
   Bounds2::Area() 102
   Bounds2i 97
   Camera::GetFilm() 207
   Film::PixelBounds() 246
   ImageTileIntegrator 24
   ImageTileIntegrator::camera 25
   ImageTileIntegrator:: samplerPrototype 25
   Options 1032
   ParallelFor2D() 1108
   ProgressReporter 1068
   Sampler 469
   Sampler::Clone() 470
   Sampler::SamplesPerPixel() 469
   ThreadLocal 1112
```

The ParallelFor2D() function loops over image tiles, running multiple loop iterations concurrently; it is part of the parallelism-related utility functions that are introduced in Section B.6. A Rust closure provides the loop body. ParallelFor2D() automatically chooses a tile size to balance two concerns: on one hand, we would like to have significantly more tiles than there are processors in the system. It is likely that some of the tiles will take less processing time than others, so if there was for example a 1:1 mapping between processors and tiles, then some processors will be idle after finishing their work while others continue to work on their region of the image. (Figure 1.17 graphs the distribution of time taken to render tiles of an example image, illustrating this concern.) On the other hand, having too many tiles also hurts efficiency. There is a small fixed overhead for a thread to acquire more work in the parallel for loop and the more tiles there are, the more times this overhead must be paid. ParallelFor2D() therefore chooses a tile size that accounts for both the extent of the region to be processed and the number of processors in the system. 〈Render current wave’s image tiles in parallel〉 ≡ 26

ParallelFor2D(pixelBounds, [&](Bounds2i tileBounds) { 〈Render image tile given by tileBounds 27〉 });

```text
                                                            art
       Figure 1.17: Histogram of Time Spent Rendering Each Tile for the Scene in Figure 1.11. The
       horizontal axis measures time in seconds. Note the wide variation in execution time, illustrating that
       different parts of the image required substantially different amounts of computation.
```

Given a tile to render, the implementation starts by acquiring the ScratchBuffer and Sampler for the currently executing thread. As described earlier, the ThreadLocal::Get() method takes care of the details of allocating and returning individual ones of them for each thread. With those in hand, the implementation loops over all the pixels in the tile using a range-based for loop that uses iterators provided by the Bounds2

class before informing the ProgressReporter about how much work has been completed. 〈Render image tile given by tileBounds〉 ≡ 27

ScratchBuffer &scratchBuffer = scratchBuffers.Get(); Sampler &sampler = samplers.Get(); for (Point2i pPixel : tileBounds) { 〈Render samples in pixel pPixel 28〉 } progress.Update((waveEnd - waveStart) * tileBounds.Area());

```text
   Bounds2 97
   Bounds2::Area() 102
   Bounds2i 97
   ParallelFor2D() 1108
   Point2i 92
   ProgressReporter 1068
   ProgressReporter::Update() 1068
   Sampler 469
   ScratchBuffer 1078
   ThreadLocal::Get() 1112
```

Given a pixel to take one or more samples in, the thread’s Sampler is notified that it should start generating samples for the current pixel via StartPixelSample(), which allows it to set up any internal state that depends on which pixel is currently being processed. The integrator’s EvaluatePixelSample() method is then responsible for determining the specified sample’s value, after which any temporary memory it may have allocated in the ScratchBuffer is freed with a call to ScratchBuffer::Reset().

〈Render samples in pixel pPixel〉 ≡ 27

for (int sampleIndex = waveStart; sampleIndex < waveEnd; ++sampleIndex) { sampler.StartPixelSample(pPixel, sampleIndex); EvaluatePixelSample(pPixel, sampleIndex, sampler, scratchBuffer); scratchBuffer.Reset(); }

Having provided an implementation of the pure virtual Integrator::Render() method, ImageTileIntegrator now imposes the requirement on its subclasses that they implement the following EvaluatePixelSample() method.

〈ImageTileIntegrator Public Methods〉 +≡ 24

virtual void EvaluatePixelSample(Point2i pPixel, int sampleIndex, Sampler sampler, ScratchBuffer &scratchBuffer) = 0;

After the parallel for loop for the current wave completes, the range of sample indices to be processed in the next wave is computed. 〈Update start and end wave〉 ≡ 26

waveStart = waveEnd; waveEnd = std::min(spp, waveEnd + nextWaveSize); nextWaveSize = std::min(2 * nextWaveSize, 64);

If the user has provided the --write-partial-images command-line option, the in-progress image is written to disk before the next wave of samples is processed. We will not include here the fragment that takes care of this, 〈Optionally write current image to disk〉.

### 1.3.5 RayIntegrator IMPLEMENTATION

Just as the ImageTileIntegrator centralizes functionality related to integrators that decompose the image into tiles, RayIntegrator provides commonly used functionality to integrators that trace ray paths starting from the camera. All of the integrators implemented in Chapters 13 and 14 inherit from RayIntegrator. 〈RayIntegrator Definition〉 ≡ class RayIntegrator : public ImageTileIntegrator { public: 〈RayIntegrator Public Methods 28〉 };

```text
    Camera 206
    Film 244
    ImageTileIntegrator 24
    ImageTileIntegrator:: EvaluatePixelSample() 28
    Integrator::Render() 23
    Light 740
    Point2i 92
    Primitive 398
    RayIntegrator 28
    Sampler 469
    Sampler::StartPixelSample() 469
    ScratchBuffer 1078
    ScratchBuffer::Reset() 1079
```

Its constructor does nothing more than pass along the provided objects to the ImageTile Integrator constructor.

〈RayIntegrator Public Methods〉 ≡ 28

RayIntegrator(Camera camera, Sampler sampler, Primitive aggregate, std::vector<Light> lights) : ImageTileIntegrator(camera, sampler, aggregate, lights) {}

RayIntegrator implements the pure virtual EvaluatePixelSample() method from ImageTile Integrator. At the given pixel, it uses its Camera and Sampler to generate a ray into the scene and then calls the Li() method, which is provided by the subclass, to determine the amount of light arriving at the image plane along that ray. As we will see in following chapters, the units of the value returned by this method are related to the incident spectral radiance at the ray origin, which is generally denoted by the symbol Li in equations—thus, the method name. This value is passed to the Film, which records the ray’s contribution to the image.

```text
                                                          art
       Figure 1.18: Class Relationships for RayIntegrator::EvaluatePixelSample()’s computation. The
       Sampler provides sample values for each image sample to be taken. The Camera turns a sample into a
       corresponding ray from the film plane, and the Li() method computes the radiance along that ray arriving
       at the film. The sample and its radiance are passed to the Film, which stores their contribution in an
       image.
```

*Figure 1.18 summarizes the main classes used in this method and the flow of data among them. 〈RayIntegrator Method Definitions〉 ≡ void RayIntegrator::EvaluatePixelSample(Point2i pPixel, int sampleIndex, Sampler sampler, ScratchBuffer &scratchBuffer) { 〈Sample wavelengths for the ray 29〉 〈Initialize CameraSample for current sample 30〉 〈Generate camera ray for current sample 30〉 〈Trace cameraRay if valid 30〉 〈Add camera ray’s contribution to image 31〉*

}

Each ray carries radiance at a number of discrete wavelengths λ (four, by default). When computing the color at each pixel, pbrt chooses different wavelengths at different pixel samples so that the final result better reflects the correct result over all wavelengths. To choose these wavelengths, a sample value lu is first provided by the Sampler. This value will be uniformly distributed and in the range [0, 1). The Film::SampleWavelengths() method then maps this sample to a set of specific wavelengths, taking into account its model of film sensor response as a function of wavelength. Most Sampler implementations ensure that if multiple samples are taken in a pixel, those samples are in the aggregate well distributed over [0, 1). In turn, they ensure that the sampled wavelengths are also well distributed across the range of valid wavelengths, improving image quality. 〈Sample wavelengths for the ray〉 ≡ 29

Float lu = sampler.Get1D(); SampledWavelengths lambda = camera.GetFilm().SampleWavelengths(lu);

```text
   Camera 206
   Camera::GetFilm() 207
   CameraSample 206
   Film 244
   Film::SampleWavelengths() 246
   Float 23
   GetCameraSample() 516
   ImageTileIntegrator::camera 25
   Point2i 92
   SampledWavelengths 173
   Sampler 469
   Sampler::Get1D() 470
   ScratchBuffer 1078
```

The CameraSample structure records the position on the film for which the camera should generate a ray. This position is affected by both a sample position provided by the sampler and the reconstruction filter that is used to filter multiple sample values into a single value for the pixel. GetCameraSample() handles those calculations. CameraSample also stores a time that is associated with the ray as well as a lens position sample,

which are used when rendering scenes with moving objects and for camera models that simulate non-pinhole apertures, respectively. 〈Initialize CameraSample for current sample〉 ≡ 29

```rust
   let filter = camera.get_film().get_filter();
   let camera_sample = get_camera_sample(sampler, p_pixel, filter);
```

The Camera interface provides two methods to generate rays: GenerateRay(), which returns the ray for a given image sample position, and GenerateRayDifferential(), which returns a ray differential, which incorporates information about the rays that the camera would generate for samples that are one pixel away on the image plane in both the x and y directions. Ray differentials are used to get better results from some of the texture functions defined in Chapter 10, by making it possible to compute how quickly a texture varies with respect to the pixel spacing, which is a key component of texture antialiasing. Some CameraSample values may not correspond to valid rays for a given camera. Therefore, pstd::optional is used for the CameraRayDifferential returned by the camera.

〈Generate camera ray for current sample〉 ≡ 29

pstd::optional<CameraRayDifferential> cameraRay = camera.GenerateRayDifferential(cameraSample, lambda);

If the camera ray is valid, it is passed along to the RayIntegrator subclass’s Li() method implementation after some additional preparation. In addition to returning the radiance along the ray L, the subclass is also responsible for initializing an instance of the VisibleSurface class, which records geometric information about the surface the ray intersects (if any) at each pixel for the use of Film implementations like the GBufferFilm that store more information than just color at each pixel. 〈Trace cameraRay if valid〉 ≡ 29

SampledSpectrum L(0.); VisibleSurface visibleSurface; if (cameraRay) { 〈Scale camera ray differentials based on image sampling rate 30〉 〈Evaluate radiance along camera ray 31〉 〈Issue warning if unexpected radiance value is returned〉 }

```text
   Camera 206
   Camera:: GenerateRayDifferential() 207
   Camera::GetFilm() 207
   CameraRayDifferential 207
   CameraSample 206
   Film::GetFilter() 246
   Filter 515
   Float 23
   GBufferFilm 253
   GetCameraSample() 516
   ImageTileIntegrator::camera 25
   RayDifferential:: ScaleDifferentials() 97
   RayIntegrator 28
   SampledSpectrum 171
   Sampler::SamplesPerPixel() 469
   VisibleSurface 245
```

Before the ray is passed to the Li() method, the ScaleDifferentials() method scales the differential rays to account for the actual spacing between samples on the film plane when multiple samples are taken per pixel. 〈Scale camera ray differentials based on image sampling rate〉 ≡ 30

Float rayDiffScale = std::max<Float>(.125f, 1 / std::sqrt((Float)sampler.SamplesPerPixel())); cameraRay->ray.ScaleDifferentials(rayDiffScale);

For Film implementations that do not store geometric information at each pixel, it is worth saving the work of populating the VisibleSurface class. Therefore, a pointer to this class is only passed in the call to the Li() method if it is necessary, and a null pointer is passed otherwise. Integrator implementations then should only initialize the VisibleSurface if it is non-null. CameraRayDifferential also carries a weight associated with the ray that is used to scale the returned radiance value. For simple camera models, each ray is weighted equally, but camera models that more accurately simulate the process of image formation by lens systems may generate some rays that contribute more than others. Such a camera model might simulate the effect of less light arriving at the edges of the film plane than at the center, an effect called vignetting.

〈Evaluate radiance along camera ray〉 ≡ 30

bool initializeVisibleSurface = camera.GetFilm().UsesVisibleSurface(); L = cameraRay->weight * Li(cameraRay->ray, lambda, sampler, scratchBuffer, initializeVisibleSurface ? &visibleSurface : nullptr);

Li() is a pure virtual method that RayIntegrator subclasses must implement. It returns the incident radiance at the origin of a given ray, sampled at the specified wavelengths. 〈RayIntegrator Public Methods〉 +≡ 28

virtual SampledSpectrum Li( RayDifferential ray, SampledWavelengths &lambda, Sampler sampler, ScratchBuffer &scratchBuffer, VisibleSurface *visibleSurface) const = 0;

A common side effect of bugs in the rendering process is that impossible radiance values are computed. For example, division by zero results in radiance values equal to either the IEEE floating-point infinity or a “not a number” value. The renderer looks for these possibilities and prints an error message when it encounters them. Here we will not include the fragment that does this, 〈Issue warning if unexpected radiance value is returned〉. See the implementation in cpu/integrator.cpp if you are interested in its details. After the radiance arriving at the ray’s origin is known, a call to Film::AddSample() updates the corresponding pixel in the image, given the weighted radiance for the sample. The details of how sample values are recorded in the film are explained in Sections 5.4 and 8.8. 〈Add camera ray’s contribution to image〉 ≡ 29

camera.GetFilm().AddSample(pPixel, L, lambda, &visibleSurface, cameraSample.filterWeight);

### 1.3.6 RANDOM WALK INTEGRATOR

Although it has taken a few pages to go through the implementation of the integrator infrastructure that culminated in RayIntegrator, we can now turn to implementing light transport integration algorithms in a simpler context than having to start implementing a complete Integrator::Render() method. The RandomWalkIntegrator that we will

describe in this section inherits from RayIntegrator and thus all the details of multi-threading, generating the initial ray from the camera and then adding the radiance along that ray to the image, are all taken care of. The integrator operates in a simpler context: a ray has been provided and its task is to compute the radiance arriving at its origin. Recall that in Section 1.2.7 we mentioned that in the absence of participating media, the light carried by a ray is unchanged as it passes through free space. We will ignore the possibility of participating media in the implementation of this integrator, which allows us to take a first step: given the first intersection of a ray with the geometry in the scene, the radiance arriving at the ray’s origin is equal to the radiance leaving the intersection point toward the ray’s origin. That outgoing radiance is given by the light transport equation (1.1), though it is hopeless to evaluate it in closed form. Numerical approaches are required, and the ones used in pbrt are based on Monte Carlo integration, which makes it possible to estimate the values of integrals based on pointwise evaluation of their integrands. Chapter 2 provides an introduction to Monte Carlo integration, and additional Monte Carlo techniques will be introduced as they are used throughout the book.

```text
   Camera::GetFilm() 207
   CameraRayDifferential::ray 207
   CameraRayDifferential::weight 207
   CameraSample::filterWeight 206
   Film::AddSample() 244
   Film::UsesVisibleSurface() 245
   ImageTileIntegrator::camera 25
   Integrator::Render() 23
   RayDifferential 96
   RayIntegrator 28
   RayIntegrator::Li() 31
   SampledSpectrum 171
   SampledWavelengths 173
   Sampler 469
   ScratchBuffer 1078
   VisibleSurface 245
```

```text
                                                           art
        Figure 1.19: A View of the Watercolor Scene, Rendered with the RandomWalkIntegrator. Because the
        RandomWalkIntegrator does not handle perfectly specular surfaces, the two glasses on the table are
        black. Furthermore, even with the 8,192 samples per pixel used to render this image, the result is still
        peppered with high-frequency noise. (Note, for example, the far wall and the base of the chair.) (Scene
        courtesy of Angelo Ferretti.)
```

In order to compute the outgoing radiance, the RandomWalkIntegrator implements a simple Monte Carlo approach that is based on incrementally constructing a random walk, where a series of points on scene surfaces are randomly chosen in succession to construct light-carrying paths starting from the camera. This approach effectively models image formation in the real world in reverse, starting from the camera rather than from the light sources. Going backward in this respect is still physically valid because the physical models of light that pbrt is based on are time-reversible.

```text
   RandomWalkIntegrator 33
```

Although the implementation of the random walk sampling algorithm is in total just over twenty lines of code, it is capable of simulating complex lighting and shading effects; Figure 1.19 shows an image rendered using it. (That image required many hours of computation to achieve that level of quality, however.) For the remainder of this section, we will gloss over a few of the mathematical details of the integrator’s implementation and focus on an intuitive understanding of the approach, though subsequent chapters will fill in the gaps and explain this and more sophisticated techniques more rigorously. 〈RandomWalkIntegrator Definition〉 ≡ class RandomWalkIntegrator : public RayIntegrator { public: 〈RandomWalkIntegrator Public Methods 33〉 private: 〈RandomWalkIntegrator Private Methods 33〉 〈RandomWalkIntegrator Private Members 34〉 };

This integrator recursively evaluates the random walk. Therefore, its Li() method implementation does little more than start the recursion, via a call to the LiRandomWalk() method. Most of the parameters to Li() are just passed along, though the VisibleSurface is ignored for this simple integrator and an additional parameter is added to track the depth of recursion. 〈RandomWalkIntegrator Public Methods〉 ≡ 33

SampledSpectrum Li(RayDifferential ray, SampledWavelengths &lambda, Sampler sampler, ScratchBuffer &scratchBuffer, VisibleSurface *visibleSurface) const { return LiRandomWalk(ray, lambda, sampler, scratchBuffer, 0); } 〈RandomWalkIntegrator Private Methods〉 ≡ 33

SampledSpectrum LiRandomWalk(RayDifferential ray, SampledWavelengths &lambda, Sampler sampler, ScratchBuffer &scratchBuffer, int depth) const { 〈Intersect ray with scene and return if no intersection 33〉 〈Get emitted radiance at surface intersection 34〉 〈Terminate random walk if maximum depth has been reached 35〉 〈Compute BSDF at random walk intersection point 35〉 〈Randomly sample direction leaving surface for random walk 35〉 〈Evaluate BSDF at surface for sampled direction 35〉 〈Recursively trace ray to estimate incident radiance at surface 35〉 }

```text
   Integrator::Intersect() 23
   RandomWalkIntegrator:: LiRandomWalk() 33
   RayDifferential 96
   RayIntegrator 28
   SampledSpectrum 171
   SampledWavelengths 173
   Sampler 469
   ScratchBuffer 1078
   ShapeIntersection 266
   ShapeIntersection::intr 266
   SurfaceInteraction 138
   VisibleSurface 245
```

The first step is to find the closest intersection of the ray with the shapes in the scene. If no intersection is found, the ray has left the scene. Otherwise, a SurfaceInteraction that is returned as part of the ShapeIntersection

structure provides information about the local geometric properties of the intersection point. 〈Intersect ray with scene and return if no intersection〉 ≡ 33

```rust
   let si = self.intersect(ray);
   if si.is_none() {
       // Return emitted light from infinite light sources
       return le;
   }
   let mut isect = si.unwrap().intr;
```

If no intersection was found, radiance still may be carried along the ray due to light sources such as the ImageInfiniteLight that do not have geometry associated with them. The Light::Le() method allows such lights to return their radiance for a given ray. 〈Return emitted light from infinite light sources〉 ≡ 33

SampledSpectrum Le(0.f); for (Light light : infiniteLights) Le += light.Le(ray, lambda); return Le;

If a valid intersection has been found, we must evaluate the light transport equation at the intersection point. The first term, Le(p, ωo), which is the emitted radiance, is easy: emission is part of the scene specification and the emitted radiance is available by calling the SurfaceInteraction::Le() method, which takes the outgoing direction of interest. Here, we are interested in radiance emitted back along the ray’s direction. If the object is not emissive, that method returns a zero-valued spectral distribution. 〈Get emitted radiance at surface intersection〉 ≡ 33

Vector3f wo = -ray.d; SampledSpectrum Le = isect.Le(wo, lambda);

Evaluating the second term of the light transport equation requires computing an integral over the sphere of directions around the intersection point p. Application of the principles of Monte Carlo integration can be used to show that if directions ω′ are chosen with equal probability over all possible directions, then an estimate of the integral can be computed as a weighted product of the BSDF f, which describes the light scattering properties of the material at p, the incident lighting, Li, and a cosine factor: art

In other words, given a random direction ω′, estimating the value of the integral requires evaluating the terms in the integrand for that direction and then scaling by a factor of 4π. (This factor, which is derived in Section A.5.2, relates to the surface area of a unit sphere.) Since only a single direction is considered, there is almost always error in the Monte Carlo estimate compared to the true value of the integral. However, it can be shown that estimates like this one are correct in expectation: informally, that they give the correct result on average. Averaging multiple independent estimates generally reduces this error—hence, the practice of taking multiple samples per pixel. The BSDF and the cosine factor of the estimate are easily evaluated, leaving us with Li, the incident radiance, unknown. However, note that we have found ourselves right back where we started with the initial call to LiRandomWalk(): we have a ray for which we would like to find the incident radiance at the origin—that, a recursive call to LiRandomWalk() will provide.

```text
   ImageInfiniteLight 767
   Integrator::infiniteLights 23
   Light 740
   Light::Le() 743
   Ray::d 95
   SampledSpectrum 171
   SurfaceInteraction::Le() 762
   Vector3f 86
```

Before computing the estimate of the integral, we must consider terminating the recursion. The RandomWalkIntegrator stops at a predetermined maximum depth, maxDepth. Without this termination criterion, the algorithm might never terminate (imagine, e.g., a hall-of-mirrors scene). This member variable is initialized in the constructor based on a parameter that can be set in the scene description file. 〈RandomWalkIntegrator Private Members〉 ≡ 33

int maxDepth; 〈Terminate random walk if maximum depth has been reached〉 ≡ 33

if (depth == maxDepth) return Le;

If the random walk is not terminated, the SurfaceInteraction::GetBSDF() method is called to find the BSDF at the intersection point. It evaluates texture functions to determine surface properties and then initializes a representation of the BSDF. It generally needs to allocate memory for the objects that constitute the BSDF’s representation; because this memory only needs to be active when processing the current ray, the ScratchBuffer is provided to it to use for its allocations. 〈Compute BSDF at random walk intersection point〉 ≡ 33

BSDF bsdf = isect.GetBSDF(ray, lambda, camera, scratchBuffer, sampler);

Next, we need to sample a random direction ω′ to compute the estimate in Equation (1.2). The SampleUniformSphere() function returns a uniformly distributed direction on the unit sphere, given two uniform values in [0, 1) that are provided here by the sampler. 〈Randomly sample direction leaving surface for random walk〉 ≡ 33

Point2f u = sampler.Get2D(); Vector3f wp = SampleUniformSphere(u);

All the factors of the Monte Carlo estimate other than the incident radiance can now be readily evaluated. The BSDF class provides an f() method that evaluates the BSDF for a pair of specified directions, and the cosine of the angle with the surface normal can be computed using the AbsDot() function, which returns the absolute value of the dot product between two vectors. If the vectors are normalized, as both are here, this value is equal to the absolute value of the cosine of the angle between them (Section 3.3.2). It is possible that the BSDF will be zero-valued for the provided directions and thus that fcos will be as well—for example, the BSDF is zero if the surface is not transmissive but the two directions are on opposite sides of it.5 In that case, there is no reason to continue the random walk, since subsequent points will make no contribution to the result. 〈Evaluate BSDF at surface for sampled direction〉 ≡ 33

SampledSpectrum fcos = bsdf.f(wo, wp) * AbsDot(wp, isect.shading.n); if (!fcos) return Le;

```text
   AbsDot() 90
   BSDF 544
   BSDF::f() 545
   Pi 1033
   Point2f 92
   RandomWalkIntegrator:: LiRandomWalk() 33
   RandomWalkIntegrator:: maxDepth 34
   SampledSpectrum 171
   Sampler::Get2D() 470
   SampleUniformSphere() 1016
   ScratchBuffer 1078
   SurfaceInteraction::GetBSDF() 682
   SurfaceInteraction:: shading::n 139
   SurfaceInteraction:: SpawnRay() 645
   Vector3f 86
```

The remaining task is to compute the new ray leaving the surface in the sampled direction ω′. This task is handled by the SpawnRay() method, which returns a ray leaving an intersection in the provided direction, ensuring that the ray is sufficiently offset from the surface that it does not incorrectly reintersect it due to round-off error. Given the ray, the recursive call to LiRandomWalk() can be made to estimate the incident radiance, which completes the estimate of Equation (1.2). 〈Recursively trace ray to estimate incident radiance at surface〉 ≡ 33

ray = isect.SpawnRay(wp); return Le + fcos * LiRandomWalk(ray, lambda, sampler, scratchBuffer, depth + 1) / (1 / (4 * Pi));

```text
                                                          art
        Figure 1.20: Watercolor Scene Rendered Using 32 Samples per Pixel. (a) Rendered using the
        RandomWalkIntegrator. (b) Rendered using the PathIntegrator, which follows the same general
        approach but uses more sophisticated Monte Carlo techniques. The PathIntegrator gives a substantially
        better image for roughly the same amount of work, with 54.5× reduction in mean squared error.
```

This simple approach has many shortcomings. For example, if the emissive surfaces are small, most ray paths will not find any light and many rays will need to be traced to form an accurate image. In the limit case of a point light source, the image will be black, since there is zero probability of intersecting such a light source. Similar issues apply with BSDF models

that scatter light in a concentrated set of directions. In the limiting case of a perfect mirror that scatters incident light along a single direction, the RandomWalkIntegrator will never be able to randomly sample that direction. Those issues and more can be addressed through more sophisticated application of Monte Carlo integration techniques. In subsequent chapters, we will introduce a succession of improvements that lead to much more accurate results. The integrators that are defined in Chapters 13 through 15 are the culmination of those developments. All still build on the same basic ideas used in the RandomWalkIntegrator, but are much more efficient and robust than it is. Figure 1.20 compares the RandomWalkIntegrator to one of the improved integrators and gives a sense of how much improvement is possible.

```text
   PathIntegrator 833
   RandomWalkIntegrator 33
```

## 1.4 HOW TO PROCEED THROUGH THIS BOOK

We have written this book assuming it will be read in roughly front-to-back order. We have tried to minimize the number of forward references to ideas and interfaces that have not yet been introduced, but we do assume that the reader is acquainted with the previous content at any particular point in the text. Some sections go into depth about advanced topics that some readers may wish to skip over, particularly on first reading; each advanced section is identified by an asterisk in its title. Because of the modular nature of the system, the main requirements are that the reader be familiar with the low-level classes like Point3f, Ray, and SampledSpectrum; the interfaces defined by the abstract base classes listed in Table 1.1; and the rendering loop that culminates in calls to integrators’ RayIntegrator::Li() methods. Given that knowledge, for example, the reader who does not care about precisely how a camera model based on a perspective projection matrix maps CameraSamples to rays can skip over the implementation of that camera and can just remember that the Camera::GenerateRayDifferential() method somehow turns a CameraSample into a RayDifferential.

The remainder of this book is divided into four main parts of a few chapters each. First, Chapters 2 through 4 introduce the foundations of the system. A brief introduction to the key ideas underlying Monte Carlo integration is provided in Chapter 2, and Chapter 3 then describes widely used geometric classes like Point3f, Ray, and Bounds3f. Chapter 4 introduces the physical units used to measure light and the SampledSpectrum class that pbrt uses to represent spectral distributions. It also discusses color, the human perception of spectra, which affects how input is provided to the renderer and how it generates output. The second part of the book covers image formation and how the scene geometry is represented. Chapter 5 defines the Camera interface and a few different camera implementations before discussing the overall process of turning spectral radiance arriving at the film into images. Chapter 6 then introduces the Shape interface and gives implementations of a number of shapes, including showing how to perform ray intersection tests with them. Chapter 7 describes the implementations of the acceleration structures that make ray tracing more efficient by skipping tests with primitives that a ray can be shown to definitely not intersect. Finally, Chapter 8’s topic is the Sampler classes that place samples on the image plane and provide random samples for Monte Carlo integration. The third part of the book is about light and how it scatters from surfaces and participating media. Chapter 9 includes a collection of classes that define a variety of types of reflection from surfaces. Materials, described in Chapter 10, use these reflection functions to implement a number of different surface types, such as plastic, glass, and metal. Spatial variation in material properties (color, roughness, etc.) is modeled by textures, which are also described in Chapter 10. Chapter 11 introduces the abstractions that describe how light is scattered and absorbed in participating media, and Chapter 12 then describes the interface for light sources and a variety of light source implementations.

```text
   Bounds3f 97
   Camera 206
   Camera:: GenerateRayDifferential() 207
   CameraSample 206
   Point3f 92
   RandomWalkIntegrator 33
```

```text
    Ray 95
    RayDifferential 96
    RayIntegrator::Li() 31
    SampledSpectrum 171
    Sampler 469
    Shape 261
```

The last part brings all the ideas from the rest of the book together to implement a number of interesting light transport algorithms. The integrators in Chapters 13 and 14 represent a variety of different applications of Monte Carlo integration to compute more accurate approximations of the light transport equation than the RandomWalkIntegrator. Chapter 15 then describes the implementation of a high-performance integrator that runs on the GPU, based on all the same classes that are used in the implementations of the CPU-based integrators. Chapter 16, the last chapter of the book, provides a brief retrospective and discussion of system design decisions along with a number of suggestions for more far-reaching projects than those in the exercises. Appendices contain more Monte Carlo sampling algorithms, describe utility functions, and explain details of how the scene description is created as the input file is parsed.

### 1.4.1 THE EXERCISES

At the end of each chapter you will find exercises related to the material covered in that chapter. Each exercise is marked as one of three levels of difficulty: ➊ An exercise that should take only an hour or two ➋ A reading and/or implementation task that would be suitable for a course assignment and should take between 10 and 20 hours of work ➌ A suggested final project for a course that will likely take 40 hours or more to complete

### 1.4.2 VIEWING THE IMAGES

Figures throughout the book compare the results of rendering the same scene using different algorithms. As with previous editions of the book, we have done our best to ensure that these differences are evident on the printed page, though even high quality printing cannot match modern

display technology, especially now with the widespread availability of high dynamic range displays. We have therefore made all of the rendered images that are used in figures available online. For example, the first image shown in this chapter as Figure 1.1 is available at the URL pbr-book.org/4ed/fig/1.1. All of the others follow the same naming scheme.

### 1.4.3 THE ONLINE EDITION

Starting on November 1, 2023, the full contents of this book will be freely available online at pbr-book.org/4ed. (The previous edition of the book is already available at that website.) The online edition includes additional content that could not be included in the printed book due to page constraints. All of that material is supplementary to the contents of this book. For example, it includes the implementation of an additional camera model, a kd-tree acceleration structure, and a full chapter on bidirectional light transport algorithms. (Almost all of the additional material appeared in the previous edition of the book.) 1.5 USING AND UNDERSTANDING THE CODE

The pbrt source code distribution is available from pbrt.org. The website also includes additional documentation, images rendered with pbrt, example scenes, errata, and links to a bug reporting system. We encourage you to visit the website and subscribe to the pbrt mailing list. pbrt is written in Rust, but we have tried to make it accessible to non-Rust experts by limiting the use of esoteric features of the language. Staying close to the core language features also helps with the system’s portability. We make use of Rust’s extensive standard library whenever it is applicable but will not discuss the semantics of calls to standard library functions in the text. Our expectation is that the reader will consult documentation of the standard library as necessary. We will occasionally omit short sections of pbrt’s source code from the book. For example, when there are a number of cases to be handled, all with nearly identical code, we will present one case and note that the code for the remaining cases has been omitted from the text. Default class constructors

are generally not shown, and the text also does not include details like the various #include directives at the start of each source file. All the omitted code can be found in the pbrt source code distribution.

### 1.5.1 SOURCE CODE ORGANIZATION

The source code used for building pbrt is under the src directory in the pbrt distribution. In that directory are src/ext, which has the source code for various third-party libraries that are used by pbrt, and src/pbrt, which contains pbrt’s source code. We will not discuss the third-party libraries’ implementations in the book. The source files in the src/pbrt directory mostly consist of implementations of the various interface types. For example, shapes.h and shapes.cpp have implementations of the Shape interface, materials.h and materials.cpp have materials, and so forth. That directory also holds the source code for parsing pbrt’s scene description files.

The pbrt.h header file in src/pbrt is the first file that is included by all other source files in the system. It contains a few macros and widely useful forward declarations, though we have tried to keep it short and to minimize the number of other headers that it includes in the interests of compile time efficiency. The src/pbrt directory also contains a number of subdirectories. They have the following roles: base: Header files defining the interfaces for 12 of the common interface types listed in Table 1.1 (Primitive and Integrator are CPU-only and so are defined in files in the cpu directory). cmd: Source files containing the main() functions for the executables that are built for pbrt. (Others besides the pbrt executable include imgtool, which performs various image processing operations, and pbrt_test, which contains unit tests.) cpu: CPU-specific code, including Integrator implementations. gpu: GPU-specific source code, including functions for allocating memory and launching work on the GPU. util: Lower-level utility code, most of it not specific to rendering.

```text
            wavefront: Implementation of the WavefrontPathIntegrator,
            which is introduced in Chapter 15. This integrator runs on both
            CPUs and GPUs.
```

### 1.5.2 NAMING CONVENTIONS

Functions and classes are generally named using Camel case, with the first letter of each word capitalized and no delineation for spaces. One exception is some methods of container classes, which follow the naming convention of the Rust standard library when they have matching functionality (e.g., size() and begin() and end() for iterators). Variables also use Camel case, though with the first letter lowercase, except for a few global variables. We also try to match mathematical notation in naming: for example, we use variables like p for points p and w for directions ω. We will occasionally add a p to the end of a variable to denote a primed symbol: wp for ω′. Underscores are used to indicate subscripts in equations: theta_o for θo, for example. Our use of underscores is not perfectly consistent, however. Short variable names often omit the underscore—we use wi for ωi and we have already seen the use of Li for Li. We also occasionally use an underscore to separate a word from a lowercase mathematical symbol. For example, we use Sample_f for a method that samples a function f rather than Samplef, which would be more difficult to read, or SampleF, which would obscure the connection to the function f (“where was the function F defined?”).

```text
    Integrator 22
    Primitive 398
    Shape 261
    WavefrontPathIntegrator 939
```

### 1.5.3 POINTER OR REFERENCE?

Rust provides two different mechanisms for passing an object to a function or method by reference: pointers and references. If a function argument is not intended as an output variable, either can be used to save the expense of passing the entire structure on the stack. The convention in pbrt is to use a

pointer when the argument will be completely changed by the function or method, a reference when some of its internal state will be changed but it will not be fully reinitialized, and const references when it will not be changed at all. One important exception to this rule is that we will always use a pointer when we want to be able to pass nullptr to indicate that a parameter is not available or should not be used.

### 1.5.4 ABSTRACTION VERSUS EFFICIENCY

One of the primary tensions when designing interfaces for software systems is making a reasonable trade-off between abstraction and efficiency. For example, many programmers religiously make all data in all classes private and provide methods to obtain or modify the values of the data items. For simple classes (e.g., Vector3f), we believe that approach needlessly hides a basic property of the implementation—that the class holds three floating-point coordinates—that we can reasonably expect to never change. Of course, using no information hiding and exposing all details of all classes’ internals leads to a code maintenance nightmare, but we believe that there is nothing wrong with judiciously exposing basic design decisions throughout the system. For example, the fact that a Ray is represented with a point, a vector, a time, and the medium it is in is a decision that does not need to be hidden behind a layer of abstraction. Code elsewhere is shorter and easier to understand when details like these are exposed. An important thing to keep in mind when writing a software system and making these sorts of trade-offs is the expected final size of the system. pbrt is roughly 70,000 lines of code and it is never going to grow to be a million lines of code; this fact should be reflected in the amount of information hiding used in the system. It would be a waste of programmer time (and likely a source of runtime inefficiency) to design the interfaces to accommodate a system of a much higher level of complexity.

### 1.5.5 pstd

We have reimplemented a subset of the Rust standard library in the pstd namespace; this was necessary in order to use those parts of it interchangeably on the CPU and on the GPU. For the purposes of reading

pbrt’s source code, anything in pstd provides the same functionality with the same type and methods as the corresponding entity in std. We will therefore not document usage of pstd in the text here.

### 1.5.6 ALLOCATORS

Almost all dynamic memory allocation for the objects that represent the scene in pbrt is performed using an instance of an Allocator that is provided to the object creation methods. In pbrt, Allocator is shorthand for the Rust standard library’s pmr::polymorphic_allocator type. Its definition is in pbrt.h so that it is available to all other source files. 〈Define Allocator〉 ≡ using Allocator = pstd::pmr::polymorphic_allocator<std::byte>;

std::pmr::polymorphic_allocator implementations provide a few methods for allocating and freeing objects. These three are used widely in pbrt:6

```text
    Ray 95
    Vector3f 86
```

```rust
   fn allocate_bytes(&self, nbytes: usize, alignment: usize) -> *mut u8;
   fn allocate_object<T>(&self, n: usize) -> *mut T;
   fn new_object<T>(&self, args: ...) -> *mut T;
```

The first, allocate_bytes(), allocates the specified number of bytes of memory. Next, allocate_object() allocates an array of n objects of the specified type T, initializing each one with its default constructor. The final method, new_object(), allocates a single object of type T and calls its constructor with the provided arguments. There are corresponding methods for freeing each type of allocation: deallocate_bytes(), deallocate_object(), and delete_object().

A tricky detail related to the use of allocators with data structures from the Rust standard library is that a container’s allocator is fixed once its

constructor has run. Thus, if one container is assigned to another, the target container’s allocator is unchanged even though all the values it stores are updated. (This is the case even with Rust’s move semantics.) Therefore, it is common to see objects’ constructors in pbrt passing along an allocator in member initializer lists for containers that they store even if they are not yet ready to set the values stored in them. Using an explicit memory allocator rather than direct calls to new and delete has a few advantages. Not only does it make it easy to do things like track the total amount of memory that has been allocated, but it also makes it easy to substitute allocators that are optimized for many small allocations, as is useful when building acceleration structures in Chapter 7. Using allocators in this way also makes it easy to store the scene objects in memory that is visible to the GPU when GPU rendering is being used.

### 1.5.7 DYNAMIC DISPATCH

As mentioned in Section 1.3, virtual functions are generally not used for dynamic dispatch with polymorphic types in pbrt (the main exception being the Integrators). Instead, the TaggedPointer class is used to represent a pointer to one of a specified set of types; it includes machinery for runtime type identification and thence dynamic dispatch. (Its implementation can be found in Appendix B.4.4.) Two considerations motivate its use. First, in Rust, an instance of an object that inherits from an abstract base class includes a hidden virtual function table pointer that is used to resolve virtual function calls. On most modern systems, this pointer uses eight bytes of memory. While eight bytes may not seem like much, we have found that when rendering complex scenes with previous versions of pbrt, a substantial amount of memory would be used just for virtual function pointers for shapes and primitives. With the TaggedPointer class, there is no incremental storage cost for type information. The other problem with virtual function tables is that they store function pointers that point to executable code. Of course, that’s what they are supposed to do, but this characteristic means that a virtual function table can be valid for method calls from either the CPU or from the GPU, but not

from both simultaneously, since the executable code for the different processors is stored at different memory locations. When using the GPU for rendering, it is useful to be able to call methods from both processors, however. For all the code that just calls methods of polymorphic objects, the use of pbrt’s Tagged Pointer in place of virtual functions makes no difference other than the fact that method calls are made using the . operator, just as would be used for a Rust reference. Section 4.5.1, which introduces Spectrum, the first class based on TaggedPointer that occurs in the book, has more details about how pbrt’s dynamic dispatch scheme is implemented.

```text
    Spectrum 165
    TaggedPointer 1073
```

### 1.5.8 CODE OPTIMIZATION

We have tried to make pbrt efficient through the use of well-chosen algorithms rather than through local micro-optimizations, so that the system can be more easily understood. However, efficiency is an integral part of rendering, and so we discuss performance issues throughout the book. For both CPUs and GPUs, processing performance continues to grow more quickly than the speed at which data can be loaded from main memory into the processor. This means that waiting for values to be fetched from memory can be a major performance limitation. The most important optimizations that we discuss relate to minimizing unnecessary memory access and organizing algorithms and data structures in ways that lead to coherent access patterns; paying attention to these issues can speed up program execution much more than reducing the total number of instructions executed.

### 1.5.9 DEBUGGING AND LOGGING

Debugging a renderer can be challenging, especially in cases where the result is correct most of the time but not always. pbrt includes a number of facilities to ease debugging.

One of the most important is a suite of unit tests. We have found unit testing to be invaluable in the development of pbrt for the reassurance it gives that the tested functionality is very likely to be correct. Having this assurance relieves the concern behind questions during debugging such as “am I sure that the hash table that is being used here is not itself the source of my bug?” Alternatively, a failing unit test is almost always easier to debug than an incorrect image generated by the renderer; many of the tests have been added along the way as we have debugged pbrt. Unit tests for a file code.cpp are found in code_tests.cpp. All the unit tests are executed by an invocation of the pbrt_test executable and specific ones can be selected via command-line options. There are many assertions throughout the pbrt codebase, most of them not included in the book text. These check conditions that should never be true and issue an error and exit immediately if they are found to be true at runtime. (See Section B.3.6 for the definitions of the assertion macros used in pbrt.) A failed assertion gives a first hint about the source of an error; like a unit test, an assertion helps focus debugging, at least with a starting point. Some of the more computationally expensive assertions in pbrt are only enabled for debug builds; if the renderer is crashing or otherwise producing incorrect output, it is worthwhile to try running a debug build to see if one of those additional assertions fails and yields a clue. We have also endeavored to make the execution of pbrt at a given pixel sample deterministic. One challenge with debugging a renderer is a crash that only happens after minutes or hours of rendering computation. With deterministic execution, rendering can be restarted at a single pixel sample in order to more quickly return to the point of a crash. Furthermore, upon a crash pbrt will print a message such as “Rendering failed at pixel (16, 27) sample 821. Debug with --debugstart 16,27,821”. The values printed after “debugstart” depend on the integrator being used, but are sufficient to restart its computation close to the point of a crash. Finally, it is often useful to print out the values stored in a data structure during the course of debugging. We have implemented ToString() methods for nearly all of pbrt’s classes. They return a std::string representation of them so that it is easy to print their full object state during

program execution. Furthermore, pbrt’s custom Printf() and StringPrintf() functions (Section B.3.3) automatically use the string returned by ToString() for an object when a %s specifier is found in the formatting string.

```text
    Printf() 1064
    StringPrintf() 1064
```

### 1.5.10 PARALLELISM AND THREAD SAFETY

In pbrt (as is the case for most ray tracers), the vast majority of data at rendering time is read only (e.g., the scene description and texture images). Much of the parsing of the scene file and creation of the scene representation in memory is done with a single thread of execution, so there are few synchronization issues during that phase of execution.7 During rendering, concurrent read access to all the read-only data by multiple threads works with no problems on both the CPU and the GPU; we only need to be concerned with situations where data in memory is being modified. As a general rule, the low-level classes and structures in the system are not thread-safe. For example, the Point3f class, which stores three float values to represent a point in 3D space, is not safe for multiple threads to call methods that modify it at the same time. (Multiple threads can use Point3fs as read-only data simultaneously, of course.) The runtime overhead to make Point3f thread-safe would have a substantial effect on performance with little benefit in return. The same is true for classes like Vector3f, Normal3f, SampledSpectrum, Transform, Quaternion, and SurfaceInteraction. These classes are usually either created at scene construction time and then used as read-only data or allocated on the stack during rendering and used only by a single thread. The utility classes ScratchBuffer (used for high-performance temporary memory allocation) and RNG (pseudo-random number generation) are also not safe for use by multiple threads; these classes store state that is modified when their methods are called, and the overhead from protecting

modification to their state with mutual exclusion would be excessive relative to the amount of computation they perform. Consequently, in code like the ImageTileIntegrator::Render() method earlier, pbrt allocates per-thread instances of these classes on the stack. With two exceptions, implementations of the base types listed in Table 1.1 are safe for multiple threads to use simultaneously. With a little care, it is usually straightforward to implement new instances of these base classes so they do not modify any shared state in their methods. The first exceptions are the Light Preprocess() method implementations. These are called by the system during scene construction, and implementations of them generally modify shared state in their objects. Therefore, it is helpful to allow the implementer to assume that only a single thread will call into these methods. (This is a separate issue from the consideration that implementations of these methods that are computationally intensive may use ParallelFor() to parallelize their computation.) The second exception is Sampler class implementations; their methods are also not expected to be thread-safe. This is another instance where this requirement would impose an excessive performance and scalability impact; many threads simultaneously trying to get samples from a single Sampler would limit the system’s overall performance. Therefore, as described in Section 1.3.4, a unique Sampler is created for each rendering thread using Sampler::Clone(). All stand-alone functions in pbrt are thread-safe (as long as multiple threads do not pass pointers to the same data to them).

```text
   ImageTileIntegrator::Render() 25
   Light 740
   Light::Preprocess() 743
   Normal3f 94
   ParallelFor() 1107
   Point3f 92
   RNG 1054
   SampledSpectrum 171
   Sampler 469
   Sampler::Clone() 470
   ScratchBuffer 1078
   SurfaceInteraction 138
```

```text
    Transform 120
    Vector3f 86
```

### 1.5.11 EXTENDING THE SYSTEM

One of our goals in writing this book and building the pbrt system was to make it easier for developers and researchers to experiment with new (or old!) ideas in rendering. One of the great joys in computer graphics is writing new software that makes a new image; even small changes to the system can be fun to experiment with. The exercises throughout the book suggest many changes to make to the system, ranging from small tweaks to major open-ended research projects. Section C.4 in Appendix C has more information about the mechanics of adding new implementations of the interfaces listed in Table 1.1.

### 1.5.12 BUGS

Although we made every effort to make pbrt as correct as possible through extensive testing, it is inevitable that some bugs are still present. If you believe you have found a bug in the system, please do the following: 1. Reproduce the bug with an unmodified copy of the latest version of pbrt. 2. Check the online discussion forum and the bug-tracking system at pbrt.org. Your issue may be a known bug, or it may be a commonly misunderstood feature. 3. Try to find the simplest possible test case that demonstrates the bug. Many bugs can be demonstrated by scene description files that are just a few lines long, and debugging is much easier with a simple scene than a complex one. 4. Submit a detailed bug report using our online bug-tracking system. Make sure that you include the scene file that demonstrates the bug and a detailed description of why you think pbrt is not behaving correctly with the scene. If you can provide a patch that fixes the bug, all the better! We will periodically update the pbrt source code repository with bug fixes and minor enhancements. (Be aware that we often let bug reports

accumulate for a few months before going through them; do not take this as an indication that we do not value them!) However, we will not make major changes to the pbrt source code so that it does not diverge from the system described here in the book.

## 1.6 A BRIEF HISTORY OF PHYSICALLY BASED

```text
    RENDERING
```

Through the early years of computer graphics in the 1970s, the most important problems to solve were fundamental issues like visibility algorithms and geometric representations. When a megabyte of RAM was a rare and expensive luxury and when a computer capable of a million floating-point operations per second cost hundreds of thousands of dollars, the complexity of what was possible in computer graphics was correspondingly limited, and any attempt to accurately simulate physics for rendering was infeasible. As computers have become more capable and less expensive, it has become possible to consider more computationally demanding approaches to rendering, which in turn has made physically based approaches viable. This progression is neatly explained by Blinn’s law: “as technology advances, rendering time remains constant.” Jim Blinn’s simple statement captures an important constraint: given a certain number of images that must be rendered (be it a handful for a research paper or over a hundred thousand for a feature film), it is only possible to take so much processing time for each one. One has a certain amount of computation available and one has some amount of time available before rendering must be finished, so the maximum computation per image is necessarily limited. Blinn’s law also expresses the observation that there remains a gap between the images people would like to be able to render and the images that they can render: as computers have become faster, content creators have continued to use increased computational capability to render more complex scenes with more sophisticated rendering algorithms, rather than rendering the same scenes as before, just more quickly. Rendering continues to consume all computational capabilities made available to it.

### 1.6.1 RESEARCH

Physically based approaches to rendering started to be seriously considered by graphics researchers in the 1980s. Whitted’s paper (1980) introduced the idea of using ray tracing for global lighting effects, opening the door to accurately simulating the distribution of light in scenes. The rendered images his approach produced were markedly different from any that had been seen before, which spurred excitement about this approach. Another notable early advancement in physically based rendering was Cook and Torrance’s reflection model (1981, 1982), which introduced microfacet reflection models to graphics. Among other contributions, they showed that accurately modeling microfacet reflection made it possible to render metal surfaces accurately; metal was not well rendered by earlier approaches. Shortly afterward, Goral et al. (1984) made connections between the thermal transfer literature and rendering, showing how to incorporate global diffuse lighting effects using a physically based approximation of light transport. This method was based on finite-element techniques, where areas of surfaces in the scene exchanged energy with each other. This approach came to be referred to as “radiosity,” after a related physical unit. Following work by Cohen and Greenberg (1985) and Nishita and Nakamae (1985) introduced important improvements. Once again, a physically based approach led to images with lighting effects that had not previously been seen in rendered images, which led to many researchers pursuing improvements in this area. While the radiosity approach was based on physical units and conservation of energy, in time it became clear that it would not lead to practical rendering algorithms: the asymptotic computational complexity was a difficult-to-manage O(n2), and it was necessary to retessellate geometric models along shadow boundaries for good results; researchers had difficulty developing robust and efficient tessellation algorithms for this purpose. Radiosity’s adoption in practice was limited. During the radiosity years, a small group of researchers pursued physically based approaches to rendering that were based on ray tracing and Monte Carlo integration. At the time, many looked at their work with skepticism; objectionable noise in images due to Monte Carlo integration error seemed

unavoidable, while radiosity-based methods quickly gave visually pleasing results, at least on relatively simple scenes. In 1984, Cook, Porter, and Carpenter introduced distributed ray tracing, which generalized Whitted’s algorithm to compute motion blur and defocus blur from cameras, blurry reflection from glossy surfaces, and illumination from area light sources (Cook et al. 1984), showing that ray tracing was capable of generating a host of important soft lighting effects. Shortly afterward, Kajiya (1986) introduced path tracing; he set out a rigorous formulation of the rendering problem (the light transport integral equation) and showed how to apply Monte Carlo integration to solve it. This work required immense amounts of computation: to render a 256 × 256 pixel image of two spheres with path tracing required 7 hours of computation on an IBM 4341 computer, which cost roughly $280,000 when it was first released (Farmer 1981). With von Herzen, Kajiya also introduced the volume-rendering equation to graphics (Kajiya and von Herzen 1984); this equation describes the scattering of light in participating media. Both Cook et al.’s and Kajiya’s work once again led to images unlike any that had been seen before, demonstrating the value of physically based methods. In subsequent years, important work on Monte Carlo for realistic image synthesis was described in papers by Arvo and Kirk (1990) and Kirk and Arvo (1991). Shirley’s Ph.D. dissertation (1990) and follow-on work by Shirley et al. (1996) were important contributions to Monte Carlo–based efforts. Hall’s book, Illumination and Color in Computer Generated Imagery (1989), was one of the first books to present rendering in a physically based framework, and Andrew Glassner’s Principles of Digital Image Synthesis laid out foundations of the field (1995). Ward’s Radiance rendering system was an early open source physically based rendering system, focused on lighting design (Ward 1994), and Slusallek’s Vision renderer was designed to bridge the gap between physically based approaches and the then widely used RenderMan interface, which was not physically based (Slusallek 1996). Following Torrance and Cook’s work, much of the research in the Program of Computer Graphics at Cornell University investigated physically based

approaches. The motivations for this work were summarized by Greenberg et al. (1997), who made a strong argument for a physically accurate rendering based on measurements of the material properties of real-world objects and on deep understanding of the human visual system. A crucial step forward for physically based rendering was Veach’s work, described in detail in his dissertation (Veach 1997). Veach advanced key theoretical foundations of Monte Carlo rendering while also developing new algorithms like multiple importance sampling, bidirectional path tracing, and Metropolis light transport that greatly improved its efficiency. Using Blinn’s law as a guide, we believe that these significant improvements in efficiency were critical to practical adoption of these approaches. Around this time, as computers became faster and more parallel, a number of researchers started pursuing real-time ray tracing; Wald, Slusallek, and Benthin wrote an influential paper that described a highly optimized ray tracer that was much more efficient than previous ray tracers (Wald et al. 2001b). Many subsequent papers introduced increasingly more efficient ray-tracing algorithms. Though most of this work was not physically based, the results led to great progress in ray-tracing acceleration structures and performance of the geometric components of ray tracing. Because physically based rendering generally makes substantial use of ray tracing, this work has in turn had the same helpful effect as faster computers have, making it possible to render more complex scenes with physical approaches. We end our summary of the key steps in the research progress of physically based rendering at this point, though much more has been done. The “Further Reading” sections in all the subsequent chapters of this book cover this work in detail.

### 1.6.2 PRODUCTION

With more capable computers in the 1980s, computer graphics could start to be used for animation and film production. Early examples include Jim Blinn’s rendering of the Voyager 2 flyby of Saturn in 1981 and visual

effects in the movies Star Trek II: The Wrath of Khan (1982), Tron (1982), and The Last Starfighter (1984). In early production use of computer-generated imagery, rasterization-based rendering (notably, the Reyes algorithm (Cook et al. 1987)) was the only viable option. One reason was that not enough computation was available for complex reflection models or for the global lighting effects that physically based ray tracing could provide. More significantly, rasterization had the important advantage that it did not require that the entire scene representation fit into main memory. When RAM was much less plentiful, almost any interesting scene was too large to fit into main memory. Rasterization-based algorithms made it possible to render scenes while having only a small subset of the full scene representation in memory at any time. Global lighting effects are difficult to achieve if the whole scene cannot fit into main memory; for many years, with limited computer systems, content creators effectively decided that geometric and texture complexity was more important to visual realism than lighting complexity (and in turn physical accuracy). Many practitioners at this time also believed that physically based approaches were undesirable for production: one of the great things about computer graphics is that one can cheat reality with impunity to achieve a desired artistic effect. For example, lighting designers on regular movies often struggle to place light sources so that they are not visible to the camera or spend considerable effort placing a light to illuminate an actor without shining too much light on the background. Computer graphics offers the opportunity to, for example, implement a light source model that shines twice as much light on a character as on a background object. For many years, this capability seemed much more useful than physical accuracy. Visual effects practitioners who had the specific need to match rendered imagery to filmed real-world environments pioneered capturing real-world lighting and shading effects and were early adopters of physically based approaches in the late 1990s and early 2000s. (See Snow (2010) for a history of ILM’s early work in this area, for example.) During this time, Blue Sky Studios adopted a physically based pipeline (Ohmer 1997). The

photorealism of an advertisement they made for a Braun shaver in 1992 caught the attention of many, and their short film, Bunny, shown in 1998, was an early example of Monte Carlo global illumination used in production. Its visual look was substantially different from those of films and shorts rendered with Reyes and was widely noted. Subsequent feature films from Blue Sky also followed this approach. Unfortunately, Blue Sky never published significant technical details of their approach, limiting their wider influence. During the early 2000s, the mental ray ray-tracing system was used by a number of studios, mostly for visual effects. It was an efficient ray tracer with sophisticated global illumination algorithm implementations. The main focus of its developers was computer-aided design and product design applications, so it lacked features like the ability to handle extremely complex scenes and the enormous numbers of texture maps that film production demanded. After Bunny, another watershed moment came in 2001, when Marcos Fajardo came to the SIGGRAPH conference with an early version of his Arnold renderer. He showed images in the Monte Carlo image synthesis course that not only had complex geometry, textures, and global illumination but also were rendered in tens of minutes. While these scenes were not of the complexity of those used in film production at the time, his results showed many the creative opportunities from the combination of global illumination and complex scenes. Fajardo brought Arnold to Sony Pictures Imageworks, where work started to transform it to a production-capable physically based rendering system. Many issues had to be addressed, including efficient motion blur, programmable shading, support for massively complex scenes, and deferred loading of scene geometry and textures. Arnold was first used on the movie Monster House and is now available as a commercial product. In the early 2000s, Pixar’s RenderMan renderer started to support hybrid rasterization and ray-tracing algorithms and included a number of innovative algorithms for computing global illumination solutions in complex scenes. RenderMan was recently rewritten to be a physically based

ray tracer, following the general system architecture of pbrt (Christensen 2015).

```text
                                                           art
       Figure 1.21: Gravity (2013) featured spectacular computer-generated imagery of a realistic space
       environment with volumetric scattering and large numbers of anisotropic metal surfaces. The image was
       generated using Arnold, a physically based rendering system that accounts for global illumination. Image
       courtesy of Warner Bros. and Framestore.
```

One of the main reasons that physically based Monte Carlo approaches to rendering have been successful in production is that they end up improving the productivity of artists. These have been some of the important factors: The algorithms involved have essentially just a single quality knob: how many samples to take per pixel; this is extremely helpful for artists. Ray-tracing algorithms are also suited to both progressive refinement and quickly computing rough previews by taking just a few samples per pixel; rasterization-based renderers do not have equivalent capabilities. Adopting physically based reflection models has made it easier to design surface materials. Earlier, when reflection models that did not necessarily conserve energy were used, an object might be placed in a single lighting environment while its surface reflection parameters were adjusted. The object might look great in that environment, but it would often appear completely wrong when moved to another lighting environment because surfaces were reflecting too little or too much energy: surface properties had been set to unreasonable values. The quality of shadows computed with ray tracing is much better than it is with rasterization. Eliminating the need to tweak shadow map resolutions, biases, and other parameters has eliminated an unpleasant task of lighting artists. Further, physically based methods bring with them bounce lighting and other soft-lighting effects from the method itself, rather than as an artistically tuned manual process.

As of this writing, physically based rendering is used widely for producing computer-generated imagery for movies; Figures 1.21 and 1.22 show images from two recent movies that used physically based approaches. FURTHER READING

In a seminal early paper, Arthur Appel (1968) first described the basic idea of ray tracing to solve the hidden surface problem and to compute shadows in polygonal scenes. Goldstein and Nagel (1971) later showed how ray tracing could be used to render scenes with quadric surfaces. Kay and Greenberg (1979) described a ray-tracing approach to rendering transparency, and Whitted’s seminal CACM article described a general recursive ray-tracing algorithm that accurately simulates reflection and refraction from specular surfaces and shadows from point light sources (Whitted 1980). Whitted has recently written an article describing developments over the early years of ray tracing (Whitted 2020).

```text
                                                         art
       Figure 1.22: This image from Alita: Battle Angel (2019) was also rendered using a physically based
       rendering system. Image by Weta Digital, © 2018 Twentieth Century Fox Film Corporation. All Rights
       Reserved.
```

In addition to the ones discussed in Section 1.6, notable early books on physically based rendering and image synthesis include Cohen and Wallace’s Radiosity and Realistic Image Synthesis (1993), Sillion and Puech’s Radiosity and Global Illumination (1994), and Ashdown’s Radiosity: A Programmer’s Perspective (1994), all of which primarily describe the finite-element radiosity method. The course notes from the Monte Carlo ray-tracing course at SIGGRAPH have a wealth of practical information (Jensen et al. 2001a, 2003), much of it still relevant, now nearly twenty years later. In a paper on ray-tracing system design, Kirk and Arvo (1988) suggested many principles that have now become classic in renderer design. Their renderer was implemented as a core kernel that encapsulated the basic rendering algorithms and interacted with primitives and shading routines via a carefully constructed object-oriented interface. This approach made it

easy to extend the system with new primitives and acceleration methods. pbrt’s design is based on these ideas.

To this day, a good reference on basic ray-tracer design is Introduction to Ray Tracing (Glassner 1989a), which describes the state of the art in ray tracing at that time and has a chapter by Heckbert that sketches the design of a basic ray tracer. More recently, Shirley and Morley’s Realistic Ray Tracing (2003) offers an easy-to-understand introduction to ray tracing and includes the complete source code to a basic ray tracer. Suffern’s book (2007) also provides a gentle introduction to ray tracing. Shirley’s Ray Tracing in One Weekend series (2020) is an accessible introduction to the joy of writing a ray tracer. Researchers at Cornell University have developed a rendering testbed over many years; its design and overall structure were described by Trumbore, Lytle, and Greenberg (1993). Its predecessor was described by Hall and Greenberg (1983). This system is a loosely coupled set of modules and libraries, each designed to handle a single task (ray–object intersection acceleration, image storage, etc.) and written in a way that makes it easy to combine appropriate modules to investigate and develop new rendering algorithms. This testbed has been quite successful, serving as the foundation for much of the rendering research done at Cornell through the 1990s. Radiance was the first widely available open source renderer based fundamentally on physical quantities. It was designed to perform accurate lighting simulation for architectural design. Ward described its design and history in a paper and a book (Ward 1994; Larson and Shakespeare 1998). Radiance is designed in the UNIX style, as a set of interacting programs, each handling a different part of the rendering process. This general type of rendering architecture was first described by Duff (1985). Glassner’s (1993) Spectrum rendering architecture also focuses on physically based rendering, approached through a signal-processing-based formulation of the problem. It is an extensible system built with a plug-in architecture; pbrt’s approach of using parameter/value lists for initializing implementations of the main abstract interfaces is similar to Spectrum’s.

One notable feature of Spectrum is that all parameters that describe the scene can be functions of time. Slusallek and Seidel (1995, 1996; Slusallek 1996) described the Vision rendering system, which is also physically based and designed to support a wide variety of light transport algorithms. In particular, it had the ambitious goal of supporting both Monte Carlo and finite-element-based light transport algorithms. Many papers have been written that describe the design and implementation of other rendering systems, including renderers for entertainment and artistic applications. The Reyes architecture, which forms the basis for Pixar’s RenderMan renderer, was first described by Cook et al. (1987), and a number of improvements to the original algorithm have been summarized by Apodaca and Gritz (2000). Gritz and Hahn (1996) described the BMRT ray tracer. The renderer in the Maya modeling and animation system was described by Sung et al. (1998), and some of the internal structure of the mental ray renderer is described in Driemeyer and Herken’s book on its API (Driemeyer and Herken 2002). The design of the high-performance Manta interactive ray tracer was described by Bigler et al. (2006). OptiX introduced a particularly interesting design approach for high-performance ray tracing: it is based on doing JIT compilation at runtime to generate a specialized version of the ray tracer, intermingling user-provided code (such as for material evaluation and sampling) and renderer-provided code (such as high-performance ray–object intersection). It was described by Parker et al. (2010). More recently, Eisenacher et al. discussed the ray sorting architecture of Disney’s Hyperion renderer (Eisenacher et al. 2013), and Lee et al. have written about the implementation of the MoonRay rendering system at DreamWorks (Lee et al. 2017). The implementation of the Iray ray tracer was described by Keller et al. (2017). In 2018, a special issue of ACM Transactions on Graphics included papers describing the implementations of five rendering systems that are used for feature film production. These papers are full of details about the various renderers; reading them is time well spent. They include Burley et al.’s description of Disney’s Hyperion renderer (2018), Christensen et al. on

Pixar’s modern RenderMan (2018), Fascione et al. describing Weta Digital’s Manuka (2018), Georgiev et al. on Solid Angle’s version of Arnold (2018) and Kulla et al. on the version of Arnold used at Sony Pictures Imageworks (2018). Whereas standard rendering algorithms generate images from a 3D scene description, the Mitsuba 2 system is engineered around the corresponding inverse problem. It computes derivatives with respect to scene parameters using JIT-compiled kernels that efficiently run on GPUs and CPUs. These kernels are then used in the inner loop of an optimization algorithm to reconstruct 3D scenes that are consistent with user-provided input images. This topic is further discussed in Section 16.3.1. The system’s design and implementation was described by Nimier-David et al. (2019). EXERCISE

➊ 1.1 A good way to gain an understanding of pbrt is to follow the process of computing the radiance value for a single ray in a debugger. Build a version of pbrt with debugging symbols and set up your debugger to run pbrt with a not-too-complex scene. Set breakpoints in the ImageTileIntegrator::Render() method and trace through the process of how a ray is generated, how its radiance value is computed, and how its contribution is added to the image. The first time you do this, you may want to specify that only a single thread of execution should be used by providing --nthreads 1 as command-line arguments to pbrt; doing so ensures that all computation is done in the main processing thread, which may make it easier to understand what is going on, depending on how easy your debugger makes it to step through the program when it is running multiple threads.

```text
             As you gain more understanding about the details of the system later in the book, repeat
             this process and trace through particular parts of the system more carefully.
```

```text
     ImageTileIntegrator::Render() 25
```

_________________ 1 The example code in this section is merely illustrative and is not part of pbrt itself. 2 Although digital sensors are now more common than physical film, we will use “film” to encompass both in cases where either could be used. 3 Although ray tracing’s logarithmic complexity is often heralded as one of its key strengths, this complexity is typically only true on average. A number of ray-tracing algorithms that have guaranteed logarithmic running time have been published in the computational geometry literature, but these algorithms only work for certain types of scenes and have very expensive preprocessing and storage requirements. Szirmay-Kalos and Márton provide pointers to the relevant literature (Szirmay-Kalos and Márton 1998). In practice, the ray intersection algorithms presented in this book are sublinear, but without expensive preprocessing and huge memory usage it is always possible to construct worst-case scenes where ray tracing runs in O(mn) time. One consolation is that scenes representing realistic environments generally do not exhibit this worst-case behavior. 4 At the time of writing, these capabilities are only available on NVIDIA hardware, but it would not be too difficult to port pbrt to other architectures that provide them in the future.

5 It would be easy enough to check if the BSDF was only reflective and to only sample directions on the same side of the surface as the ray, but for this simple integrator we will not bother. 6 Because pmr::polymorphic_allocator is a recent addition to Rust that is not yet widely used, yet is widely used in pbrt, we break our regular habit of not documenting standard library functionality in the text here. 7 Exceptions include the fact that we try to load image maps and binary geometry files in parallel, some image resampling performed on texture images, and construction of one variant of the BVHAggregate, though all of these are highly localized.
