# Chapter 5: Shaders and the Graphics Pipeline

## Or: Programming Your GPU to Paint Pixels

Shaders are where graphics programming gets fun. They're tiny programs that run on your GPU, transforming vertices and coloring pixels. If the graphics pipeline is a factory, shaders are the specialized workers who actually do the transforming and painting. And unlike factory workers, they never complain about working conditions or ask for coffee breaks.

## What Are Shaders, Really?

Shaders are programs that run in parallel on your GPU's cores. When you draw a triangle with 3 vertices, the vertex shader runs 3 times in parallel. When that triangle covers 50,000 pixels, the fragment shader runs 50,000 times in parallel. This is why GPUs have thousands of cores – they're shader-running machines.

Think of it like this:
- **CPU code**: "For each vertex, transform it"
- **GPU shaders**: "Everyone transform your vertex NOW!"

The GPU doesn't iterate; it swarms.

## The Graphics Pipeline Stages

Let's explore each stage in detail:

```
[Vertex Buffer]
    ↓
[Vertex Shader] ← You write this!
    ↓
[Tessellation] (Optional)
    ↓
[Geometry Shader] (Optional) ← You can write this!
    ↓
[Rasterization]
    ↓
[Fragment Shader] ← You write this!
    ↓
[Color Blending]
    ↓
[Framebuffer]
```

## GLSL: The Shader Language

GLSL (OpenGL Shading Language) is C-like but with built-in vector and matrix types. It's designed for parallel execution with no recursion, no function pointers, and no dynamic memory allocation. It's C with training wheels, but those training wheels let you go really, really fast.

**Important:** Vulkan doesn't use GLSL directly. You must compile GLSL to SPIR-V bytecode:
```bash
# Compile vertex shader
glslc shader.vert -o vert.spv

# Compile fragment shader
glslc shader.frag -o frag.spv
```

### Basic GLSL Types

```glsl
// Scalars
float a = 1.0;
int b = 42;
uint c = 42u;
bool d = true;

// Vectors
vec2 pos2D = vec2(0.5, 0.5);
vec3 color = vec3(1.0, 0.0, 0.0);  // Red
vec4 rgba = vec4(color, 1.0);       // Add alpha

// Matrices
mat2 rotation2D;
mat3 normalMatrix;
mat4 mvpMatrix;  // Model-View-Projection

// Swizzling (GLSL's party trick)
vec3 rgb = rgba.rgb;  // Extract RGB
vec2 xy = rgba.xy;    // Extract XY
vec4 bgra = rgba.bgra; // Reorder components
```

## Advanced Vertex Shader

Let's write a vertex shader that does actual transformation:

```glsl
#version 450

// Uniform buffer for transformation matrices
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
    float time;
} ubo;

// Vertex attributes
layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;
layout(location = 3) in vec3 inNormal;

// Outputs to fragment shader
layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;
layout(location = 2) out vec3 fragNormal;
layout(location = 3) out vec3 fragWorldPos;

void main() {
    // Animated vertex displacement (wave effect)
    vec3 pos = inPosition;
    pos.y += sin(pos.x * 5.0 + ubo.time) * 0.1;

    // Transform to world space
    vec4 worldPos = ubo.model * vec4(pos, 1.0);
    fragWorldPos = worldPos.xyz;

    // Transform normal to world space
    fragNormal = mat3(transpose(inverse(ubo.model))) * inNormal;

    // Final position in clip space
    gl_Position = ubo.proj * ubo.view * worldPos;

    // Pass through color and texture coordinates
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

## Advanced Fragment Shader

Now let's add lighting to our fragment shader:

```glsl
#version 450

// Inputs from vertex shader
layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;
layout(location = 2) in vec3 fragNormal;
layout(location = 3) in vec3 fragWorldPos;

// Output
layout(location = 0) out vec4 outColor;

// Uniforms
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
    float time;
} ubo;

// Texture sampler
layout(binding = 1) uniform sampler2D texSampler;

void main() {
    // Light properties
    vec3 lightPos = vec3(2.0 * sin(ubo.time), 2.0, 2.0 * cos(ubo.time));
    vec3 lightColor = vec3(1.0);
    float lightPower = 10.0;

    // Surface properties
    vec3 normal = normalize(fragNormal);
    vec3 lightDir = normalize(lightPos - fragWorldPos);
    vec3 viewDir = normalize(-fragWorldPos);  // Camera at origin in view space

    // Diffuse lighting (Lambertian)
    float diff = max(dot(normal, lightDir), 0.0);
    vec3 diffuse = diff * lightColor;

    // Specular lighting (Blinn-Phong)
    vec3 halfwayDir = normalize(lightDir + viewDir);
    float spec = pow(max(dot(normal, halfwayDir), 0.0), 32.0);
    vec3 specular = spec * lightColor;

    // Ambient lighting
    vec3 ambient = vec3(0.1);

    // Combine with texture
    vec3 texColor = texture(texSampler, fragTexCoord).rgb;
    vec3 result = (ambient + diffuse + specular) * texColor * fragColor;

    // Gamma correction
    result = pow(result, vec3(1.0/2.2));

    outColor = vec4(result, 1.0);
}
```

## Descriptor Sets and Bindings

Descriptor sets are how shaders access resources like buffers and textures. Think of them as a table of contents for your shader's resources:

```cpp
void createDescriptorSetLayout() {
    // Uniform buffer binding
    VkDescriptorSetLayoutBinding uboLayoutBinding{};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
    uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT;
    uboLayoutBinding.pImmutableSamplers = nullptr;

    // Texture sampler binding
    VkDescriptorSetLayoutBinding samplerLayoutBinding{};
    samplerLayoutBinding.binding = 1;
    samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
    samplerLayoutBinding.descriptorCount = 1;
    samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;
    samplerLayoutBinding.pImmutableSamplers = nullptr;

    std::array<VkDescriptorSetLayoutBinding, 2> bindings = {
        uboLayoutBinding,
        samplerLayoutBinding
    };

    VkDescriptorSetLayoutCreateInfo layoutInfo{};
    layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
    layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
    layoutInfo.pBindings = bindings.data();

    if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create descriptor set layout!");
    }
}
```

## Push Constants: Fast Small Data

Push constants are a fast way to send small amounts of data to shaders:

```cpp
// In your pipeline layout creation
VkPushConstantRange pushConstantRange{};
pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT;
pushConstantRange.offset = 0;
pushConstantRange.size = sizeof(PushConstants);

VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.pushConstantRangeCount = 1;
pipelineLayoutInfo.pPushConstantRanges = &pushConstantRange;
```

Use them in your shader:
```glsl
layout(push_constant) uniform PushConstants {
    vec4 color;
    float time;
} pc;
```

And update them when recording commands:
```cpp
PushConstants constants{};
constants.color = glm::vec4(1.0f, 0.0f, 0.0f, 1.0f);
constants.time = currentTime;

vkCmdPushConstants(commandBuffer, pipelineLayout,
    VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
    0, sizeof(PushConstants), &constants);
```

## Specialization Constants: Compile-Time Configuration

Specialization constants let you configure shaders at pipeline creation time:

```glsl
layout(constant_id = 0) const int LIGHT_COUNT = 4;
layout(constant_id = 1) const float AMBIENT_STRENGTH = 0.1;

void main() {
    vec3 ambient = vec3(AMBIENT_STRENGTH);

    for (int i = 0; i < LIGHT_COUNT; i++) {
        // Process each light
    }
}
```

Configure them when creating the pipeline:
```cpp
// Store this as a class member, not a local variable!
struct SpecializationData {
    int lightCount = 4;
    float ambientStrength = 0.1f;
};

// In your pipeline creation function:
SpecializationData specializationData;

VkSpecializationMapEntry entries[2] = {
    {0, offsetof(SpecializationData, lightCount), sizeof(int)},
    {1, offsetof(SpecializationData, ambientStrength), sizeof(float)}
};

VkSpecializationInfo specializationInfo{};
specializationInfo.mapEntryCount = 2;
specializationInfo.pMapEntries = entries;
specializationInfo.dataSize = sizeof(SpecializationData);
specializationInfo.pData = &specializationData;

shaderStageInfo.pSpecializationInfo = &specializationInfo;
```

## Compute Shaders: General Purpose GPU Programming

Compute shaders don't draw anything; they just compute:

```glsl
#version 450

layout(local_size_x = 16, local_size_y = 16) in;

layout(binding = 0, rgba8) uniform image2D inputImage;
layout(binding = 1, rgba8) uniform image2D outputImage;

void main() {
    ivec2 pixelCoords = ivec2(gl_GlobalInvocationID.xy);

    // Box blur
    vec4 color = vec4(0.0);
    for (int x = -1; x <= 1; x++) {
        for (int y = -1; y <= 1; y++) {
            color += imageLoad(inputImage, pixelCoords + ivec2(x, y));
        }
    }
    color /= 9.0;

    imageStore(outputImage, pixelCoords, color);
}
```

## Exercise 5.1: Psychedelic Shader

Create a fragment shader that generates patterns without textures:

```glsl
#version 450

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

layout(push_constant) uniform PushConstants {
    float time;
} pc;

void main() {
    vec2 uv = fragTexCoord - 0.5;
    float distance = length(uv);

    // Rotating spiral
    float angle = atan(uv.y, uv.x);
    float spiral = sin(distance * 20.0 - pc.time * 2.0 + angle * 5.0);

    // Color waves
    vec3 color;
    color.r = sin(pc.time + distance * 10.0) * 0.5 + 0.5;
    color.g = sin(pc.time * 1.3 + distance * 10.0 + 2.0) * 0.5 + 0.5;
    color.b = sin(pc.time * 1.7 + distance * 10.0 + 4.0) * 0.5 + 0.5;

    color *= spiral * 0.5 + 0.5;

    outColor = vec4(color, 1.0);
}
```

## Exercise 5.2: Geometry Shader for Explosion Effect

Make triangles explode outward:

```glsl
#version 450

layout(triangles) in;
layout(triangle_strip, max_vertices = 3) out;

layout(location = 0) in vec3 inColor[];
layout(location = 0) out vec3 outColor;

layout(push_constant) uniform PushConstants {
    float explosionFactor;
} pc;

void main() {
    // Calculate triangle center
    vec4 center = (gl_in[0].gl_Position +
                   gl_in[1].gl_Position +
                   gl_in[2].gl_Position) / 3.0;

    // Emit vertices moved away from center
    for (int i = 0; i < 3; i++) {
        vec4 pos = gl_in[i].gl_Position;
        vec4 dir = normalize(pos - center);

        gl_Position = pos + dir * pc.explosionFactor;
        outColor = inColor[i];
        EmitVertex();
    }

    EndPrimitive();
}
```

## Exercise 5.3: Tessellation for Smooth Surfaces

Add tessellation to create smooth surfaces from coarse meshes:

```glsl
// Tessellation Control Shader
#version 450

layout(vertices = 3) out;

layout(location = 0) in vec3 inColor[];
layout(location = 0) out vec3 outColor[];

void main() {
    outColor[gl_InvocationID] = inColor[gl_InvocationID];

    if (gl_InvocationID == 0) {
        float tessLevel = 16.0;
        gl_TessLevelInner[0] = tessLevel;
        gl_TessLevelOuter[0] = tessLevel;
        gl_TessLevelOuter[1] = tessLevel;
        gl_TessLevelOuter[2] = tessLevel;
    }

    gl_out[gl_InvocationID].gl_Position = gl_in[gl_InvocationID].gl_Position;
}
```

## Pipeline Caching: Don't Compile Twice

Pipeline creation is expensive. Cache it:

```cpp
void createPipelineCache() {
    VkPipelineCacheCreateInfo pipelineCacheCreateInfo{};
    pipelineCacheCreateInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO;

    // Try to load existing cache
    std::vector<char> cacheData;
    std::ifstream cacheFile("pipeline.cache", std::ios::binary | std::ios::ate);

    if (cacheFile.is_open()) {
        size_t fileSize = cacheFile.tellg();
        cacheData.resize(fileSize);
        cacheFile.seekg(0);
        cacheFile.read(cacheData.data(), fileSize);
        cacheFile.close();

        pipelineCacheCreateInfo.initialDataSize = cacheData.size();
        pipelineCacheCreateInfo.pInitialData = cacheData.data();
    }

    vkCreatePipelineCache(device, &pipelineCacheCreateInfo, nullptr, &pipelineCache);
}

void savePipelineCache() {
    size_t cacheSize;
    vkGetPipelineCacheData(device, pipelineCache, &cacheSize, nullptr);

    std::vector<char> cacheData(cacheSize);
    vkGetPipelineCacheData(device, pipelineCache, &cacheSize, cacheData.data());

    std::ofstream cacheFile("pipeline.cache", std::ios::binary);
    cacheFile.write(cacheData.data(), cacheData.size());
}
```

## Shader Debugging Tips

Debugging shaders is hard because you can't use printf or breakpoints. Here are some tricks:

### 1. Color Debugging
```glsl
// Visualize normals
outColor = vec4(normal * 0.5 + 0.5, 1.0);

// Visualize UV coordinates
outColor = vec4(fragTexCoord, 0.0, 1.0);

// Visualize depth
float depth = gl_FragCoord.z;
outColor = vec4(vec3(depth), 1.0);
```

### 2. Validation Layers
Enable shader validation:
```cpp
const char* validationLayers[] = {
    "VK_LAYER_KHRONOS_validation"
};
```

### 3. RenderDoc
Use RenderDoc to capture and inspect frames. It shows:
- Shader inputs/outputs
- Texture contents
- Buffer contents
- Draw call parameters

## Performance Optimization

### 1. Minimize Texture Reads
```glsl
// Bad: Multiple dependent reads
vec3 color1 = texture(sampler, uv).rgb;
vec3 color2 = texture(sampler, uv + color1.xy).rgb;
vec3 color3 = texture(sampler, uv + color2.xy).rgb;

// Better: Independent reads
vec3 color1 = texture(sampler, uv).rgb;
vec3 color2 = texture(sampler, uv + vec2(0.01)).rgb;
vec3 color3 = texture(sampler, uv + vec2(0.02)).rgb;
```

### 2. Avoid Branching
```glsl
// Bad: Branching
if (distance > 0.5) {
    color = vec3(1.0);
} else {
    color = vec3(0.0);
}

// Better: Use step/smoothstep
color = vec3(step(0.5, distance));
```

### 3. Use Built-in Functions
```glsl
// Bad: Manual calculation
float len = sqrt(v.x * v.x + v.y * v.y + v.z * v.z);

// Better: Built-in function
float len = length(v);
```

## Shader Variants and Uber-Shaders

Instead of creating hundreds of similar shaders, use specialization constants:

```glsl
layout(constant_id = 0) const bool USE_TEXTURE = true;
layout(constant_id = 1) const bool USE_NORMAL_MAP = true;
layout(constant_id = 2) const bool USE_LIGHTING = true;

void main() {
    vec3 color = fragColor;

    if (USE_TEXTURE) {
        color *= texture(texSampler, fragTexCoord).rgb;
    }

    if (USE_LIGHTING) {
        // Apply lighting calculations
    }

    outColor = vec4(color, 1.0);
}
```

## What We've Learned

We've covered:
- GLSL shader programming
- Vertex transformation pipeline
- Fragment shading and lighting
- Descriptor sets and resource binding
- Push constants and specialization constants
- Compute shaders
- Pipeline caching
- Shader debugging techniques
- Performance optimization

You now understand how to program your GPU to transform vertices and paint pixels exactly how you want.

## The Power of Shaders

Shaders are where the magic happens in graphics programming. They're your artistic brush, your mathematical transformer, your pixel painter. Every visual effect you've ever seen in a game or application – from simple color tints to complex water simulations – is implemented in shaders.

## What's Next

Chapter 6 will dive into memory management and buffers. We'll learn:
- Different types of GPU memory
- Staging buffers for efficient uploads
- Buffer update strategies
- Memory allocation best practices
- Vertex and index buffer optimization

Your shaders are hungry for data. Let's learn how to feed them efficiently!

## Shader Philosophy

Remember: GPUs are not CPUs. They don't think sequentially; they think in parallel. When writing shaders:
- Think in terms of "all pixels" not "each pixel"
- Avoid dependent operations
- Embrace the parallel mindset
- Let the GPU do what it does best: run the same code on massive amounts of data simultaneously

---

*Next Chapter Preview: Memory management in Vulkan. We'll learn why GPUs have more types of memory than a computer science textbook, and how to use each one effectively.*