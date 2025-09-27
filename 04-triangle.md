# Chapter 4: Drawing a Triangle - The Hello World of Graphics

## Or: 1000 Lines of Code for Three Points

We've arrived. The moment you've been waiting for. After three chapters of setup, configuration, and clearing screens to different colors, we're finally going to draw something. A triangle. Three vertices. One primitive. The simplest possible shape that isn't a point or a line.

In OpenGL, this would be maybe 100 lines. In Vulkan? Hold onto your hats, we're going deep.

## Why a Triangle?

Every graphics programmer's journey begins with a triangle. It's tradition, like "Hello World" for programming or burning your first meal for cooking. A triangle is:
- The simplest filled primitive
- The building block of all 3D graphics
- Proof your entire pipeline works
- Your GPU's favorite shape

Fun fact: Your beautiful 4K game characters? Millions of triangles. That realistic water? Triangles. The UI? Triangles pretending to be rectangles. It's triangles all the way down.

## What We Need to Add

To our existing application, we need:
1. **Vertex data** (the triangle's corners)
2. **Vertex buffer** (GPU memory for vertices)
3. **Vertex shader** (transforms vertices)
4. **Fragment shader** (colors pixels)
5. **Graphics pipeline** (the mega-configuration object)
6. **Pipeline layout** (configuration for the pipeline)
7. **Shader modules** (compiled shader code)

## Step 1: Define the Vertex Data

First, let's define what a vertex looks like and create our triangle:

```cpp
#include <glm/glm.hpp>
#include <array>
#include <cstddef>  // for offsetof

struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    // Tell Vulkan how to read this struct
    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};
        bindingDescription.binding = 0;
        bindingDescription.stride = sizeof(Vertex);
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
        return bindingDescription;
    }

    static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

        // Position attribute
        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        // Color attribute
        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Vertex, color);

        return attributeDescriptions;
    }
};

// Our beautiful triangle (counter-clockwise winding)
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},  // Bottom, red
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}},  // Top left, blue
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}}    // Top right, green
};
```

This creates the famous RGB triangle – each vertex has a different color that will blend across the surface.

## Step 2: Write the Shaders

Vulkan shaders are written in GLSL and compiled to SPIR-V bytecode. Let's write the simplest possible shaders:

**shaders/vertex.vert:**
```glsl
#version 450

// Input from vertex buffer
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

// Output to fragment shader
layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

**shaders/fragment.frag:**
```glsl
#version 450

// Input from vertex shader
layout(location = 0) in vec3 fragColor;

// Output color
layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

Compile them to SPIR-V:
```bash
glslc shaders/vertex.vert -o shaders/vert.spv
glslc shaders/fragment.frag -o shaders/frag.spv
```

## Step 3: Load Shader Modules

Add this helper function to load compiled shaders:

```cpp
std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("Failed to open file: " + filename);
    }

    size_t fileSize = (size_t) file.tellg();
    std::vector<char> buffer(fileSize);

    file.seekg(0);
    file.read(buffer.data(), fileSize);
    file.close();

    return buffer;
}

VkShaderModule createShaderModule(VkDevice device, const std::vector<char>& code) {
    VkShaderModuleCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
    createInfo.codeSize = code.size();
    createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());

    VkShaderModule shaderModule;
    if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create shader module!");
    }

    return shaderModule;
}
```

## Step 4: Create the Graphics Pipeline

This is it. The big one. The graphics pipeline configuration. Brace yourself:

```cpp
void createGraphicsPipeline() {
    // Load shaders
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");

    VkShaderModule vertShaderModule = createShaderModule(device, vertShaderCode);
    VkShaderModule fragShaderModule = createShaderModule(device, fragShaderCode);

    // Shader stage creation
    VkPipelineShaderStageCreateInfo vertShaderStageInfo{};
    vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
    vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
    vertShaderStageInfo.module = vertShaderModule;
    vertShaderStageInfo.pName = "main";

    VkPipelineShaderStageCreateInfo fragShaderStageInfo{};
    fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
    fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
    fragShaderStageInfo.module = fragShaderModule;
    fragShaderStageInfo.pName = "main";

    VkPipelineShaderStageCreateInfo shaderStages[] = {
        vertShaderStageInfo,
        fragShaderStageInfo
    };

    // Vertex input configuration
    auto bindingDescription = Vertex::getBindingDescription();
    auto attributeDescriptions = Vertex::getAttributeDescriptions();

    VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
    vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
    vertexInputInfo.vertexBindingDescriptionCount = 1;
    vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
    vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
    vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();

    // Input assembly - how to interpret vertex data
    VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
    inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
    inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
    inputAssembly.primitiveRestartEnable = VK_FALSE;

    // Viewport and scissor
    VkViewport viewport{};
    viewport.x = 0.0f;
    viewport.y = 0.0f;
    viewport.width = (float) swapChainExtent.width;
    viewport.height = (float) swapChainExtent.height;
    viewport.minDepth = 0.0f;
    viewport.maxDepth = 1.0f;

    VkRect2D scissor{};
    scissor.offset = {0, 0};
    scissor.extent = swapChainExtent;

    VkPipelineViewportStateCreateInfo viewportState{};
    viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
    viewportState.viewportCount = 1;
    viewportState.pViewports = &viewport;
    viewportState.scissorCount = 1;
    viewportState.pScissors = &scissor;

    // Rasterizer configuration
    VkPipelineRasterizationStateCreateInfo rasterizer{};
    rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
    rasterizer.depthClampEnable = VK_FALSE;
    rasterizer.rasterizerDiscardEnable = VK_FALSE;
    rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
    rasterizer.lineWidth = 1.0f;
    rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
    rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
    rasterizer.depthBiasEnable = VK_FALSE;

    // Multisampling (disabled for now)
    VkPipelineMultisampleStateCreateInfo multisampling{};
    multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
    multisampling.sampleShadingEnable = VK_FALSE;
    multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;

    // Color blending
    VkPipelineColorBlendAttachmentState colorBlendAttachment{};
    colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT |
                                          VK_COLOR_COMPONENT_G_BIT |
                                          VK_COLOR_COMPONENT_B_BIT |
                                          VK_COLOR_COMPONENT_A_BIT;
    colorBlendAttachment.blendEnable = VK_FALSE;

    VkPipelineColorBlendStateCreateInfo colorBlending{};
    colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
    colorBlending.logicOpEnable = VK_FALSE;
    colorBlending.attachmentCount = 1;
    colorBlending.pAttachments = &colorBlendAttachment;

    // Pipeline layout (empty for now)
    VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
    pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
    pipelineLayoutInfo.setLayoutCount = 0;
    pipelineLayoutInfo.pushConstantRangeCount = 0;

    if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create pipeline layout!");
    }

    // Finally, create the pipeline!
    VkGraphicsPipelineCreateInfo pipelineInfo{};
    pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
    pipelineInfo.stageCount = 2;
    pipelineInfo.pStages = shaderStages;
    pipelineInfo.pVertexInputState = &vertexInputInfo;
    pipelineInfo.pInputAssemblyState = &inputAssembly;
    pipelineInfo.pViewportState = &viewportState;
    pipelineInfo.pRasterizationState = &rasterizer;
    pipelineInfo.pMultisampleState = &multisampling;
    pipelineInfo.pDepthStencilState = nullptr;
    pipelineInfo.pColorBlendState = &colorBlending;
    pipelineInfo.pDynamicState = nullptr;
    pipelineInfo.layout = pipelineLayout;
    pipelineInfo.renderPass = renderPass;
    pipelineInfo.subpass = 0;

    if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo,
                                 nullptr, &graphicsPipeline) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create graphics pipeline!");
    }

    // Clean up shader modules
    vkDestroyShaderModule(device, fragShaderModule, nullptr);
    vkDestroyShaderModule(device, vertShaderModule, nullptr);

    std::cout << "✓ Graphics pipeline created successfully!\n";
}
```

That's 100+ lines just to configure how triangles should be rendered. Welcome to Vulkan!

## Step 5: Create the Vertex Buffer

For now, we'll use host-visible memory for simplicity. In production, use staging buffers (see Chapter 6):

```cpp
void createVertexBuffer() {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = sizeof(vertices[0]) * vertices.size();
    bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create vertex buffer!");
    }

    // Get memory requirements
    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);

    // Allocate memory
    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &vertexBufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("Failed to allocate vertex buffer memory!");
    }

    // Bind buffer to memory
    vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);

    // Copy vertex data to buffer
    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
    memcpy(data, vertices.data(), (size_t) bufferInfo.size);
    vkUnmapMemory(device, vertexBufferMemory);

    std::cout << "✓ Vertex buffer created with " << vertices.size() << " vertices\n";
}

uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties) {
    VkPhysicalDeviceMemoryProperties memProperties;
    vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);

    for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
        if ((typeFilter & (1 << i)) &&
            (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
            return i;
        }
    }

    throw std::runtime_error("Failed to find suitable memory type!");
}
```

## Step 6: Update Command Buffer Recording

Now we bind everything and draw:

```cpp
void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex) {
    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;

    if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
        throw std::runtime_error("Failed to begin recording command buffer!");
    }

    // Begin render pass
    VkRenderPassBeginInfo renderPassInfo{};
    renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
    renderPassInfo.renderPass = renderPass;
    renderPassInfo.framebuffer = swapChainFramebuffers[imageIndex];
    renderPassInfo.renderArea.offset = {0, 0};
    renderPassInfo.renderArea.extent = swapChainExtent;

    VkClearValue clearColor = {{{0.0f, 0.0f, 0.0f, 1.0f}}};
    renderPassInfo.clearValueCount = 1;
    renderPassInfo.pClearValues = &clearColor;

    vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);

    // Bind graphics pipeline
    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);

    // Bind vertex buffer
    VkBuffer vertexBuffers[] = {vertexBuffer};
    VkDeviceSize offsets[] = {0};
    vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);

    // Draw the triangle!
    vkCmdDraw(commandBuffer, static_cast<uint32_t>(vertices.size()), 1, 0, 0);

    vkCmdEndRenderPass(commandBuffer);

    if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
        throw std::runtime_error("Failed to record command buffer!");
    }
}
```

## The Moment of Truth

Run your application. If everything worked, you'll see:
- A beautiful RGB triangle
- Each corner a different primary color
- Smooth color interpolation across the surface
- Your first real Vulkan rendering!

If you see nothing, welcome to Vulkan debugging. Check:
- Shader compilation succeeded
- Vertex winding order (clockwise vs counter-clockwise)
- Pipeline creation succeeded
- Vertex buffer is bound correctly

## Exercise 4.1: Animated Triangle

Make the triangle rotate:

```glsl
// Updated vertex shader
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

layout(push_constant) uniform PushConstants {
    float time;
} pc;

void main() {
    float angle = pc.time;
    mat2 rotation = mat2(
        cos(angle), -sin(angle),
        sin(angle), cos(angle)
    );

    vec2 rotatedPos = rotation * inPosition;
    gl_Position = vec4(rotatedPos, 0.0, 1.0);
    fragColor = inColor;
}
```

Don't forget to update your pipeline layout for push constants!

## Exercise 4.2: Multiple Triangles

Create a more complex shape with multiple triangles:

```cpp
const std::vector<Vertex> vertices = {
    // First triangle (red)
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{-0.25f, 0.0f}, {1.0f, 0.0f, 0.0f}},

    // Second triangle (green)
    {{0.0f, -0.5f}, {0.0f, 1.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}},
    {{0.25f, 0.0f}, {0.0f, 1.0f, 0.0f}},

    // Third triangle (blue)
    {{-0.25f, 0.0f}, {0.0f, 0.0f, 1.0f}},
    {{0.25f, 0.0f}, {0.0f, 0.0f, 1.0f}},
    {{0.0f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

## Exercise 4.3: Wireframe Mode

Change the pipeline to render in wireframe:

```cpp
rasterizer.polygonMode = VK_POLYGON_MODE_LINE;
```

Note: This might require enabling a GPU feature!

## Understanding the Pipeline

Let's trace the journey of our triangle:

1. **Vertex Buffer** → Vertices loaded into GPU memory
2. **Vertex Shader** → Each vertex transformed
3. **Primitive Assembly** → Vertices grouped into triangles
4. **Rasterization** → Triangle converted to pixels
5. **Fragment Shader** → Each pixel colored
6. **Color Blending** → Final pixel written to framebuffer

Every single stage is configurable. That's the power (and complexity) of Vulkan.

## Common Triangle Issues

**"Triangle is black"**
- Check your vertex colors
- Verify fragment shader output

**"Triangle is inside-out"**
- Change winding order or front face setting

**"No triangle at all"**
- Check vertex positions (are they in NDC space?)
- Verify pipeline is bound
- Check draw call parameters

**"Triangle flickers"**
- Synchronization issue
- Not waiting for previous frame

## Performance Notes

Even for our simple triangle:
- The vertex buffer should be in device-local memory (we used host-visible for simplicity)
- Command buffers can be pre-recorded if the triangle doesn't change
- Pipeline state objects should be created at initialization, not per frame

## What We've Learned

You've successfully:
- Created a graphics pipeline with shaders
- Uploaded vertex data to the GPU
- Configured the entire rendering pipeline
- Drew your first Vulkan triangle!

This is a massive achievement. You've configured every stage of the GPU pipeline explicitly. No magic, no hidden state changes, just pure, explicit graphics programming.

## The Philosophy of Triangles

Why did this take 1000 lines? Because Vulkan assumes nothing:
- You specified the vertex format explicitly
- You configured every pipeline stage
- You managed memory manually
- You controlled synchronization

This verbosity is the price of control. In exchange, you get:
- Predictable performance
- No driver guessing
- Complete optimization control
- The ability to say "I rendered a triangle in Vulkan"

## What's Next

Chapter 5 will dive deep into shaders and the graphics pipeline. We'll explore:
- Advanced shader techniques
- Uniform buffers
- Push constants
- Descriptor sets
- Pipeline caching

Your triangle is just the beginning. Soon it'll have textures, lighting, and transform into our glorious pineapple!

## A Moment of Celebration

Take a moment to appreciate what you've accomplished. You've written more code to draw a triangle than most people write for entire applications. But that triangle is YOURS. Every pixel was placed there by YOUR pipeline, following YOUR explicit instructions.

Screenshot that triangle. Frame it. Show it to your friends (the ones who'll understand). You've earned it.

---

*Next Chapter Preview: Shaders - where the real magic happens. We'll make that triangle do things that would make OpenGL jealous.*