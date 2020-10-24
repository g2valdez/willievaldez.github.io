---
layout: post
title: (WIP) Light Attenuation
subtitle: The blog post where I pretend to know about math
tags: [C++, OpenGL, Shaders, Lighting]
---
If you want your game to look good, you gotta have good lighting. At a very basic level, light sources are broken up into 3 basic types: directional, spot light, and point light.

![](/assets/img/light_types.png "Light types: point, spot, and directional")

Both spot and point lights come from a light source, while directional lights have no source, only a direction. When a light has a source, it can also have attenuation. Essentially, the farther you are from the source of the light, the less brightly it will illuminate you. Once you get far enough away from a light source, your brightness will eventually become zero. We'll call this drop off point the radius of the light. For the sake of this blog post, I will be working with point lights.

I'm not gonna go too deeply into [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language), but eventually, code funnels down to the graphics card, which is responsible for illuminating each "fragment" in view. You can run custom code for each fragment to determine the final fragment color. This is where we will be writing the attenuation function.

## Inputs & Outputs
One of the inputs of the fragment shader is the XYZ position of the fragment that we're going to render. I decided to call it `fragVert` (fragment vertex). Another input is the color of the object you are displaying. `color`

Additionally, we will be using "[uniforms](https://www.khronos.org/opengl/wiki/Uniform_%28GLSL%29)" to add custom inputs to our fragment shader.
In [Blog Post #2](https://willievaldez.github.io/2020-10-03-AutomaticRegistration) I talked about setting uniforms generically. Here are two of those uniforms that I set:

```cpp
struct PointLight
{
  vec3 pos;
  float intensity;
  float radius;
};

// There is no such thing as variable length vectors in GLSL,
// so our max will be 128 light sources
uniform int numLights;
uniform PointLight lights[128];
```

All of these inputs will be mixed around together to output a vec4, which is an RGBa representation of the color of the fragment in question. We're gonna call this output variable `FragColor` (fragment color).

## Binary Attenuation
This is the simplest form of attenuation. If you are within the radius of the light, your brightness is 1.0, if you are outside the radius, brightness is 0.0;

![](/assets/img/binary_function.png "If distance < 5.0, brightness = 1.0, else brightness = 0.0")

This is pretty easy to implement. Iterate through all lights, if the fragment is within the radius of one of the lights, increment its attenuation factor (or brightness/intensity) by the intensity of the light source;

```cpp
float attenuation = 0.0f;
for(int i = 0; i < numLights; i++)
{
	PointLight light = lights[i];
	float distToLight = distance(fragVert, light.pos);

	// binary attenuation
	if (distToLight < light.radius)
	{
		attenuation += light.intensity;
	}
}
if (attenuation > 1.0f)
{
	attenuation = 1.0f;
}

FragColor = color * attenuation;

```

![](/assets/img/binary_example.gif "example of binary attenuation in a 2d game")
