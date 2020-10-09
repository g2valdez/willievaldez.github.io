---
layout: post
title: A Typeless Container
subtitle: For when you sometimes wish C++ was a little more like JavaScript
tags: [cpp, oop]
---
Do silly constructs like "classes" sometimes get in the way of your programming? Are loosely typed languages "too loose" for you? Well boy, do I have a blog post for you! I was fiddling around with my OpenGL game engine and decided to make a container that can store and run custom functions on any object type.

## The Background
In OpenGL, you feed variables to the shader by making calls like this:
```cpp
// mat4
glm::mat4 toWorld = glm::translate(glm::mat4(1.0f), position);
GLuint matrixid = glGetUniformLocation(shaderProgram, "model");
glUniformMatrix4fv(matrixid, 1, GL_FALSE, &toWorld[0][0]);

// vec3
glm::vec3 red(1.0f, 0.0f, 0.0f);
GLuint colorId = glGetUniformLocation(shaderProgram, "color");
glUniform3fv(colorId, 1, &red[0]);

// int
int width = 1920;
GLuint texWidthId = glGetUniformLocation(shaderProgram, "width");
glUniform1i(texWidthId, width);

// float
float materialShine = 0.7f;
GLuint matShineId = glGetUniformLocation(shaderProgram, "shine");
glUniform1f(matShineId, materialShine);

// bool (I know, weird)
int usesTexture = 1; // true
GLuint usesTexId = glGetUniformLocation(shaderProgram, "usesTexture");
glUniform1i(usesTexId, usesTexture);

```

In order to keep my code base tidy, I wanted to isolate this process of setting "[uniforms](https://www.khronos.org/opengl/wiki/Uniform_%28GLSL%29)" into a single function. I quickly realized that, unless I wanted to overload the crap out of my `Asset::Render` function signature, I would need something generic to pass into the function, which it could understand and be able to set shader uniforms, uhmm... uniformly.

## The Goal
I want a class that can be used as a *single* parameter in my `Asset::Render` function.
This class contains an array of objects that are instances of multiple different types.
This class should be easy to add to, and easy to process. I don't want a huge `if/else if` block that checks the type of the object, and then runs the right `glUniform*` function call.

## The Design
Let's start by making a simple non-templated class. Our container of objects will consist of pointers to this type
```cpp
class UniformWrapper
{
public:
	UniformWrapper(const std::string& uniformName) : m_uniformName(uniformName) {};
	virtual void SetUniform(GLuint shaderProgram) const = 0;
protected:
	std::string m_uniformName;
};
```
Okay, now let's make the templated version, which stores an arbitrary data type and defines the per-type `SetUniform` function:
```cpp
template<typename T>
class TemplatedUniformWrapper : public UniformWrapper
{
public:
	TemplatedUniformWrapper(const std::string& uniformName, const T& data)
		: UniformWrapper(uniformName)
		, m_data(data) {};

	void SetUniform(GLuint shaderProgram) const override;
private:
	const T& m_data;
};
```
Finally, let's make a class that can store many uniform wrappers. (I used a vector as a container, you can use anything you want!)
```cpp
class UniformContainer
{
public:
	template<typename T>
	void AddObject(const std::string& uniformName, const T& obj)
	{
		m_vector.push_back(new TemplatedUniformWrapper<T>(uniformName, obj));
	};

	void SetUniforms(GLuint shaderProgram) const;

private:
	std::vector<UniformWrapper*> m_vector;
};
```

## The Usage
Now we just have to make template specializations for `SetUniform`. In a separate cpp file, just write out the implementation for various supported types:
```cpp
 // mat4
void TemplatedUniformWrapper<glm::mat4>::SetUniform(unsigned int shaderProgram) const
{
	GLuint uniformIntId = glGetUniformLocation(shaderProgram, m_uniformName.c_str());
	glUniformMatrix4fv(uniformIntId, 1, GL_FALSE, &m_data[0][0]);
}

// vec3
void TemplatedUniformWrapper<glm::vec3>::SetUniform(unsigned int shaderProgram) const
{
	GLuint uniformVec3Id = glGetUniformLocation(shaderProgram, m_uniformName.c_str());
	glUniform3fv(uniformVec3Id, 1, &m_data[0]);
}

// int
void TemplatedUniformWrapper<int>::SetUniform(unsigned int shaderProgram) const
{
	GLuint uniformIntId = glGetUniformLocation(shaderProgram, m_uniformName.c_str());
	glUniform1i(uniformIntId, m_data);
}

// float
void TemplatedUniformWrapper<float>::SetUniform(unsigned int shaderProgram) const
{
	GLuint uniformFltId = glGetUniformLocation(shaderProgram, m_uniformName.c_str());
	glUniform1f(uniformFltId, m_data);
}

// bool (I know, weird)
void TemplatedUniformWrapper<bool>::SetUniform(unsigned int shaderProgram) const
{
	int isTrue = m_data ? 1 : 0;
	GLuint uniformBoolId = glGetUniformLocation(shaderProgram, m_uniformName.c_str());
	glUniform1i(uniformBoolId, isTrue);
}
```

Done! That wasn't so painful, right? Now, here's how to use this new class:
```cpp
UniformContainer uniforms;
uniforms.AddObject("objCenter", glm::vec3(1.0f, 3.0f, 5.0f));
uniforms.AddObject("animationFrame", 2);
m_asset->Render(uniforms);
```
Inside of `Asset::Render`, add a call to `uniforms.SetUniforms()`, and you're ready to roll.

## Reinventing the Wheel?
If you want a typeless container, why not just use an `std::vector<void*>` or [boost::any](https://www.boost.org/doc/libs/1_48_0/doc/html/boost/any.html)?
The problem with `void*` is that it's *too* loosely typed. After casting something to a `void*` it's virtually impossible to get the original object type. At that point, I wouldn't be able to run specialized functions based on the object's type.
I personally have an aversion of any kind of `if (type1) doStuff(); else if (type2) doOtherStuff();` code, and if I wanted to iterate through an `std::vector<boost::any>`, some form of that code would be implemented.

More importantly, I enjoy making these things. Even if there is a better, more optimal out-of-the-box implementation, I get those sweet sweet dopamine hits when I successfully implement this kind of stuff.