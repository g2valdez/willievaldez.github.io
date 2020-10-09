---
layout: post
title: (WIP) An Untyped Container
subtitle: If you sometimes wish C++ was a little more like JavaScript
tags: [cpp, oop]
---
##The Background
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

##The Goal
I want a class that can be used as a *single* parameter in my `Asset::Render` function.
This class contains an array of objects that are instances of multiple different types.
This class should be easy to add to, and easy to process. I don't want a huge `if/else if` block that checks the type of the object, and then runs the right `glUniform*` function call.

##The Design
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
Okay, now let's make the templated version:
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

##The Usage
Now we just have to make definitions for the template specializations of `SetUniform`. In a separate cpp file, just write out the implementation for various supported types:
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

