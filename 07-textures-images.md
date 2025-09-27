# Chapter 7: Textures and Images

## Or: Teaching Your GPU to Look at Pictures

We've been rendering solid colors like it's 1985. Time to join the modern era and slap some textures on our geometry. Textures are just 2D arrays of colors, but Vulkan treats them like they're nuclear launch codes – with extreme care and about 500 lines of setup code.

## Images vs Buffers: The Spatial Difference

Buffers are linear chunks of memory. Images are spatially-organized memory optimized for 2D/3D access patterns. When you access neighboring pixels in a texture, they're likely in the same cache line. It's the difference between books on a shelf (buffer) and a well-organized filing cabinet (image).

```
Buffer: [R][G][B][A][R][G][B][A][R][G][B][A]...
         ↑ Linear access

Image:  [R][G][B][A] [R][G][B][A]
        [R][G][B][A] [R][G][B][A]
         ↑ Spatial locality
```

## Image Layouts: The Many Moods of a Texture

Images in Vulkan have layouts – different organizational states for different purposes:

- **VK_IMAGE_LAYOUT_UNDEFINED**: "I don't care what's in here"
- **VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL**: "Ready to receive data"
- **VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL**: "Optimized for shader reading"
- **VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL**: "Ready for rendering to"
- **VK_IMAGE_LAYOUT_PRESENT_SRC_KHR**: "Ready to show on screen"

You must explicitly transition between layouts. It's like a costume change – your image needs to be in the right outfit for the right occasion.

## Creating an Image

Let's create a proper texture loading system:

```cpp
class TextureManager {
private:
    VkDevice device;
    VkPhysicalDevice physicalDevice;
    VkCommandPool commandPool;
    VkQueue graphicsQueue;

public:
    struct Texture {
        VkImage image;
        VkDeviceMemory memory;
        VkImageView imageView;
        VkSampler sampler;
        uint32_t width, height;
        uint32_t mipLevels;
    };

    Texture createTexture(const std::string& filepath) {
        Texture texture;

        // Load image data
        int texWidth, texHeight, texChannels;
        stbi_uc* pixels = stbi_load(filepath.c_str(), &texWidth, &texHeight,
                                    &texChannels, STBI_rgb_alpha);

        if (!pixels) {
            throw std::runtime_error("Failed to load texture: " + filepath);
        }

        // Use RAII wrapper to ensure pixels are freed
        struct PixelDeleter {
            void operator()(stbi_uc* p) { stbi_image_free(p); }
        };
        std::unique_ptr<stbi_uc, PixelDeleter> pixelGuard(pixels);

        texture.width = static_cast<uint32_t>(texWidth);
        texture.height = static_cast<uint32_t>(texHeight);
        texture.mipLevels = static_cast<uint32_t>(
            std::floor(std::log2(std::max(texWidth, texHeight)))) + 1;

        VkDeviceSize imageSize = texWidth * texHeight * 4;

        // Create staging buffer
        BufferManager::Buffer stagingBuffer = createBuffer(
            imageSize,
            VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
            VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT
        );

        // Copy pixels to staging buffer
        void* data;
        vkMapMemory(device, stagingBuffer.memory, 0, imageSize, 0, &data);
        memcpy(data, pixels, static_cast<size_t>(imageSize));
        vkUnmapMemory(device, stagingBuffer.memory);

        // Create image
        createImage(texture.width, texture.height, texture.mipLevels,
                   VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL,
                   VK_IMAGE_USAGE_TRANSFER_SRC_BIT |
                   VK_IMAGE_USAGE_TRANSFER_DST_BIT |
                   VK_IMAGE_USAGE_SAMPLED_BIT,
                   VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
                   texture.image, texture.memory);

        // Transition and copy
        transitionImageLayout(texture.image, VK_FORMAT_R8G8B8A8_SRGB,
                            VK_IMAGE_LAYOUT_UNDEFINED,
                            VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
                            texture.mipLevels);

        copyBufferToImage(stagingBuffer.buffer, texture.image,
                         texture.width, texture.height);

        // Generate mipmaps (this also transitions to SHADER_READ_ONLY)
        generateMipmaps(texture.image, VK_FORMAT_R8G8B8A8_SRGB,
                       texture.width, texture.height, texture.mipLevels);

        // Cleanup staging buffer
        vkDestroyBuffer(device, stagingBuffer.buffer, nullptr);
        vkFreeMemory(device, stagingBuffer.memory, nullptr);

        // Create image view and sampler
        texture.imageView = createImageView(texture.image,
                                           VK_FORMAT_R8G8B8A8_SRGB,
                                           VK_IMAGE_ASPECT_COLOR_BIT,
                                           texture.mipLevels);
        texture.sampler = createSampler(texture.mipLevels);

        return texture;
    }

private:
    void createImage(uint32_t width, uint32_t height, uint32_t mipLevels,
                    VkFormat format, VkImageTiling tiling,
                    VkImageUsageFlags usage, VkMemoryPropertyFlags properties,
                    VkImage& image, VkDeviceMemory& imageMemory) {

        VkImageCreateInfo imageInfo{};
        imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
        imageInfo.imageType = VK_IMAGE_TYPE_2D;
        imageInfo.extent.width = width;
        imageInfo.extent.height = height;
        imageInfo.extent.depth = 1;
        imageInfo.mipLevels = mipLevels;
        imageInfo.arrayLayers = 1;
        imageInfo.format = format;
        imageInfo.tiling = tiling;
        imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
        imageInfo.usage = usage;
        imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
        imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;

        if (vkCreateImage(device, &imageInfo, nullptr, &image) != VK_SUCCESS) {
            throw std::runtime_error("Failed to create image!");
        }

        // Allocate memory
        VkMemoryRequirements memRequirements;
        vkGetImageMemoryRequirements(device, image, &memRequirements);

        VkMemoryAllocateInfo allocInfo{};
        allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
        allocInfo.allocationSize = memRequirements.size;
        allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

        if (vkAllocateMemory(device, &allocInfo, nullptr, &imageMemory) != VK_SUCCESS) {
            throw std::runtime_error("Failed to allocate image memory!");
        }

        vkBindImageMemory(device, image, imageMemory, 0);
    }
};
```

## Image Layout Transitions

The ritual of changing an image's layout:

```cpp
void transitionImageLayout(VkImage image, VkFormat format,
                          VkImageLayout oldLayout, VkImageLayout newLayout,
                          uint32_t mipLevels) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkImageMemoryBarrier barrier{};
    barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
    barrier.oldLayout = oldLayout;
    barrier.newLayout = newLayout;
    barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.image = image;
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    barrier.subresourceRange.baseMipLevel = 0;
    barrier.subresourceRange.levelCount = mipLevels;
    barrier.subresourceRange.baseArrayLayer = 0;
    barrier.subresourceRange.layerCount = 1;

    VkPipelineStageFlags sourceStage;
    VkPipelineStageFlags destinationStage;

    if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED &&
        newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
        barrier.srcAccessMask = 0;
        barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

        sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
        destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    }
    else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL &&
             newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
        barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
        barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

        sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
        destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    }
    else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL &&
             newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
        barrier.srcAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
        barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

        sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
        destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    }
    else if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED &&
             newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
        barrier.srcAccessMask = 0;
        barrier.dstAccessMask = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT |
                               VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

        sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
        destinationStage = VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
    }
    else {
        throw std::invalid_argument("Unsupported layout transition!");
    }

    vkCmdPipelineBarrier(
        commandBuffer,
        sourceStage, destinationStage,
        0,
        0, nullptr,
        0, nullptr,
        1, &barrier
    );

    endSingleTimeCommands(commandBuffer);
}
```

## Mipmapping: The Art of Blurry Textures

Mipmaps are progressively smaller versions of your texture for when objects are far away:

```cpp
void generateMipmaps(VkImage image, VkFormat imageFormat,
                    int32_t texWidth, int32_t texHeight, uint32_t mipLevels) {

    // Check if image format supports linear blitting
    VkFormatProperties formatProperties;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, imageFormat, &formatProperties);

    if (!(formatProperties.optimalTilingFeatures &
          VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT)) {
        throw std::runtime_error("Texture format doesn't support linear blitting!");
    }

    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkImageMemoryBarrier barrier{};
    barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
    barrier.image = image;
    barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    barrier.subresourceRange.baseArrayLayer = 0;
    barrier.subresourceRange.layerCount = 1;
    barrier.subresourceRange.levelCount = 1;

    int32_t mipWidth = texWidth;
    int32_t mipHeight = texHeight;

    for (uint32_t i = 1; i < mipLevels; i++) {
        barrier.subresourceRange.baseMipLevel = i - 1;
        barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
        barrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
        barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
        barrier.dstAccessMask = VK_ACCESS_TRANSFER_READ_BIT;

        vkCmdPipelineBarrier(commandBuffer,
            VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0,
            0, nullptr,
            0, nullptr,
            1, &barrier);

        // Blit from previous mip level
        VkImageBlit blit{};
        blit.srcOffsets[0] = {0, 0, 0};
        blit.srcOffsets[1] = {mipWidth, mipHeight, 1};
        blit.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        blit.srcSubresource.mipLevel = i - 1;
        blit.srcSubresource.baseArrayLayer = 0;
        blit.srcSubresource.layerCount = 1;
        blit.dstOffsets[0] = {0, 0, 0};
        blit.dstOffsets[1] = {mipWidth > 1 ? mipWidth / 2 : 1,
                              mipHeight > 1 ? mipHeight / 2 : 1, 1};
        blit.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        blit.dstSubresource.mipLevel = i;
        blit.dstSubresource.baseArrayLayer = 0;
        blit.dstSubresource.layerCount = 1;

        vkCmdBlitImage(commandBuffer,
            image, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
            image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
            1, &blit,
            VK_FILTER_LINEAR);

        // Transition to shader read
        barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
        barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
        barrier.srcAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
        barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

        vkCmdPipelineBarrier(commandBuffer,
            VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
            0, nullptr,
            0, nullptr,
            1, &barrier);

        if (mipWidth > 1) mipWidth /= 2;
        if (mipHeight > 1) mipHeight /= 2;
    }

    // Transition last mip level
    barrier.subresourceRange.baseMipLevel = mipLevels - 1;
    barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
    barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    vkCmdPipelineBarrier(commandBuffer,
        VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
        0, nullptr,
        0, nullptr,
        1, &barrier);

    endSingleTimeCommands(commandBuffer);
}
```

## Samplers: How to Read Textures

Samplers define how shaders read from textures:

```cpp
VkSampler createSampler(float maxLod) {
    VkSamplerCreateInfo samplerInfo{};
    samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
    samplerInfo.magFilter = VK_FILTER_LINEAR;  // Magnification filter
    samplerInfo.minFilter = VK_FILTER_LINEAR;  // Minification filter

    // Address mode for UVW
    samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
    samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
    samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;

    // Anisotropic filtering
    VkPhysicalDeviceProperties properties{};
    vkGetPhysicalDeviceProperties(physicalDevice, &properties);

    samplerInfo.anisotropyEnable = VK_TRUE;
    samplerInfo.maxAnisotropy = properties.limits.maxSamplerAnisotropy;

    // Border color for clamp to border
    samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;

    // Use unnormalized coordinates? (pixels vs 0-1)
    samplerInfo.unnormalizedCoordinates = VK_FALSE;

    // Comparison function for shadow maps
    samplerInfo.compareEnable = VK_FALSE;
    samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;

    // Mipmapping
    samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    samplerInfo.mipLodBias = 0.0f;
    samplerInfo.minLod = 0.0f;
    samplerInfo.maxLod = maxLod;

    VkSampler sampler;
    if (vkCreateSampler(device, &samplerInfo, nullptr, &sampler) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create texture sampler!");
    }

    return sampler;
}
```

## Texture Atlases: Many Textures, One Image

Pack multiple textures into one to reduce state changes:

```cpp
class TextureAtlas {
private:
    struct AtlasRegion {
        float u0, v0, u1, v1;  // UV coordinates
        uint32_t width, height;
    };

    Texture atlasTexture;
    std::unordered_map<std::string, AtlasRegion> regions;
    uint32_t atlasWidth, atlasHeight;

public:
    void buildAtlas(const std::vector<std::string>& texturePaths) {
        // Load all textures
        struct LoadedImage {
            std::string name;
            stbi_uc* data;
            int width, height;
        };

        std::vector<LoadedImage> images;
        for (const auto& path : texturePaths) {
            int w, h, channels;
            stbi_uc* pixels = stbi_load(path.c_str(), &w, &h, &channels, STBI_rgb_alpha);
            if (pixels) {
                images.push_back({path, pixels, w, h});
            }
        }

        // Simple packing algorithm (not optimal, but works)
        atlasWidth = 2048;
        atlasHeight = 2048;
        std::vector<uint8_t> atlasData(atlasWidth * atlasHeight * 4, 0);

        uint32_t currentX = 0;
        uint32_t currentY = 0;
        uint32_t rowHeight = 0;

        for (const auto& img : images) {
            if (currentX + img.width > atlasWidth) {
                currentX = 0;
                currentY += rowHeight;
                rowHeight = 0;
            }

            // Copy image to atlas
            for (int y = 0; y < img.height; y++) {
                memcpy(&atlasData[((currentY + y) * atlasWidth + currentX) * 4],
                       &img.data[y * img.width * 4],
                       img.width * 4);
            }

            // Store region
            regions[img.name] = {
                static_cast<float>(currentX) / atlasWidth,
                static_cast<float>(currentY) / atlasHeight,
                static_cast<float>(currentX + img.width) / atlasWidth,
                static_cast<float>(currentY + img.height) / atlasHeight,
                static_cast<uint32_t>(img.width),
                static_cast<uint32_t>(img.height)
            };

            currentX += img.width;
            rowHeight = std::max(rowHeight, static_cast<uint32_t>(img.height));

            stbi_image_free(img.data);
        }

        // Create atlas texture from combined data
        // ... upload atlasData to GPU texture ...
    }

    AtlasRegion getRegion(const std::string& name) {
        return regions[name];
    }
};
```

## Cubemaps: 360° of Texture

For skyboxes and environment mapping:

```cpp
Texture createCubemap(const std::array<std::string, 6>& faces) {
    Texture cubemap;

    // Load all 6 faces
    std::vector<stbi_uc*> faceData;
    int width, height;

    for (const auto& face : faces) {
        int w, h, channels;
        stbi_uc* pixels = stbi_load(face.c_str(), &w, &h, &channels, STBI_rgb_alpha);

        if (!pixels) {
            throw std::runtime_error("Failed to load cubemap face: " + face);
        }

        if (faceData.empty()) {
            width = w;
            height = h;
        } else if (w != width || h != height) {
            throw std::runtime_error("Cubemap faces must be the same size!");
        }

        faceData.push_back(pixels);
    }

    // Create image with 6 layers
    VkImageCreateInfo imageInfo{};
    imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    imageInfo.imageType = VK_IMAGE_TYPE_2D;
    imageInfo.extent.width = width;
    imageInfo.extent.height = height;
    imageInfo.extent.depth = 1;
    imageInfo.mipLevels = 1;
    imageInfo.arrayLayers = 6;
    imageInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
    imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
    imageInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
    imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
    imageInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT;

    // ... create and upload image ...

    // Create image view for cubemap
    VkImageViewCreateInfo viewInfo{};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = cubemap.image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_CUBE;
    viewInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 6;

    // ... cleanup ...

    return cubemap;
}
```

## Exercise 7.1: Texture Animation

Create an animated texture by cycling through frames:

```glsl
// Fragment shader
#version 450

layout(binding = 1) uniform sampler2D texSampler;

layout(push_constant) uniform PushConstants {
    float time;
    float frameCount;
} pc;

layout(location = 0) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;

void main() {
    // Calculate current frame
    int currentFrame = int(pc.time * 10.0) % int(pc.frameCount);

    // Adjust UV for sprite sheet
    vec2 frameSize = vec2(1.0 / pc.frameCount, 1.0);
    vec2 uv = fragTexCoord * frameSize;
    uv.x += frameSize.x * float(currentFrame);

    outColor = texture(texSampler, uv);
}
```

## Exercise 7.2: Normal Mapping

Implement normal mapping for detailed surfaces:

```glsl
// Fragment shader with normal mapping
#version 450

layout(binding = 1) uniform sampler2D diffuseTexture;
layout(binding = 2) uniform sampler2D normalTexture;

layout(location = 0) in vec2 fragTexCoord;
layout(location = 1) in vec3 fragTangent;
layout(location = 2) in vec3 fragBitangent;
layout(location = 3) in vec3 fragNormal;
layout(location = 4) in vec3 fragWorldPos;

layout(location = 0) out vec4 outColor;

void main() {
    // Sample normal map
    vec3 normal = texture(normalTexture, fragTexCoord).rgb;
    normal = normalize(normal * 2.0 - 1.0);  // Transform from [0,1] to [-1,1]

    // Create TBN matrix
    vec3 T = normalize(fragTangent);
    vec3 B = normalize(fragBitangent);
    vec3 N = normalize(fragNormal);
    mat3 TBN = mat3(T, B, N);

    // Transform normal to world space
    vec3 worldNormal = normalize(TBN * normal);

    // Use worldNormal for lighting calculations
    vec3 lightDir = normalize(vec3(1.0, 1.0, 1.0));
    float diff = max(dot(worldNormal, lightDir), 0.0);

    vec3 diffuse = texture(diffuseTexture, fragTexCoord).rgb;
    outColor = vec4(diffuse * diff, 1.0);
}
```

## Exercise 7.3: Procedural Textures

Generate textures on the GPU:

```glsl
// Compute shader for procedural texture generation
#version 450

layout(local_size_x = 16, local_size_y = 16) in;
layout(binding = 0, rgba8) uniform writeonly image2D outputImage;

// Perlin noise implementation
float noise(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);

    float a = random(i);
    float b = random(i + vec2(1.0, 0.0));
    float c = random(i + vec2(0.0, 1.0));
    float d = random(i + vec2(1.0, 1.0));

    vec2 u = f * f * (3.0 - 2.0 * f);
    return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) +
           (d - b) * u.x * u.y;
}

void main() {
    ivec2 texCoord = ivec2(gl_GlobalInvocationID.xy);
    vec2 uv = vec2(texCoord) / 512.0;

    // Generate wood grain pattern
    float n = 0.0;
    n += noise(uv * 4.0) * 0.5;
    n += noise(uv * 8.0) * 0.25;
    n += noise(uv * 16.0) * 0.125;
    n += noise(uv * 32.0) * 0.0625;

    // Create wood rings
    float rings = sin(uv.x * 30.0 + n * 20.0) * 0.5 + 0.5;

    // Wood colors
    vec3 darkWood = vec3(0.4, 0.2, 0.1);
    vec3 lightWood = vec3(0.7, 0.5, 0.3);
    vec3 color = mix(darkWood, lightWood, rings);

    imageStore(outputImage, texCoord, vec4(color, 1.0));
}
```

## Performance Tips

### 1. Use Texture Compression
```cpp
// Use compressed formats when possible
VK_FORMAT_BC1_RGB_SRGB_BLOCK   // DXT1
VK_FORMAT_BC3_SRGB_BLOCK        // DXT5
VK_FORMAT_ASTC_4x4_SRGB_BLOCK   // ASTC (mobile)
```

### 2. Optimize Texture Access
```glsl
// Bad: Dependent texture reads
vec3 color1 = texture(sampler1, uv).rgb;
vec2 newUV = uv + color1.xy;
vec3 color2 = texture(sampler2, newUV).rgb;

// Good: Independent reads
vec3 color1 = texture(sampler1, uv).rgb;
vec3 color2 = texture(sampler2, uv + vec2(0.1)).rgb;
```

### 3. Use Array Textures
```cpp
// Instead of binding multiple textures
VkImageCreateInfo imageInfo{};
imageInfo.arrayLayers = 16;  // 16 textures in one

// Access in shader
texture(sampler2DArray, vec3(uv, float(textureIndex)));
```

## What We've Learned

We've covered:
- Image creation and memory allocation
- Image layout transitions
- Texture loading from files
- Mipmapping for quality and performance
- Samplers and filtering
- Texture atlases
- Cubemaps
- Normal mapping
- Procedural texture generation
- Performance optimization

You now know how to work with textures in Vulkan – from loading simple images to advanced techniques like normal mapping.

## The Texture Philosophy

Textures are the soul of modern graphics. They turn flat geometry into rich, detailed surfaces. A simple quad becomes a window, a wall, a painting. Six triangles become a skybox containing an entire world.

But remember: textures are memory-hungry beasts. A single 4K texture is 64MB uncompressed. Use them wisely.

## What's Next

Chapter 8 brings it all together. We'll build our glorious spinning pineapple with:
- 3D transformations
- Model loading
- Lighting
- Texturing
- Animation

Everything we've learned culminates in one tropical fruit!

## A Textured Thought

Every pixel you see in modern games is textured. The walls, the characters, the UI – it's all textures mapped onto geometry. You now understand how those pixels get from a file on disk to your screen, transformed and filtered through the GPU's pipeline.

---

*Next Chapter Preview: The grand finale! We'll combine everything to create our masterpiece: a fully textured, lit, spinning pineapple. It's going to be glorious.*