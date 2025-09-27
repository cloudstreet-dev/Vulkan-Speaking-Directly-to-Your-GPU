# Chapter 3: Creating Your First Vulkan Application

## Or: 800 Lines of Code to Make a Window Blue

Remember in Chapter 1 when we created a Vulkan instance and felt accomplished? That was cute. Now we're going to create a real Vulkan application that actually renders something. By "something," I mean we'll clear the screen to a solid color. That's it. 800 lines for a blue screen. Your operating system does this for free when it crashes!

## The Master Plan

Here's what we need to create, in order:
1. Instance (✓ already did this!)
2. Surface (window connection)
3. Physical device selection
4. Logical device and queues
5. Swap chain
6. Image views
7. Render pass
8. Framebuffers
9. Command pool and buffers
10. Synchronization objects
11. Main loop

It's like making a sandwich, if sandwiches required 11 different types of permits.

## Project Structure Evolution

Let's organize our code properly. We'll create a `VulkanApp` class that manages everything:

**src/VulkanApp.hpp:**
```cpp
#pragma once

#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
#include <vector>
#include <optional>
#include <iostream>
#include <stdexcept>
#include <cstdlib>
#include <cstring>
#include <set>
#include <algorithm>

struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};

struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};

class VulkanApp {
public:
    void run();

private:
    // Window
    GLFWwindow* window;
    const uint32_t WIDTH = 800;
    const uint32_t HEIGHT = 600;

    // Vulkan core objects
    VkInstance instance;
    VkDebugUtilsMessengerEXT debugMessenger;
    VkSurfaceKHR surface;
    VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
    VkDevice device;
    VkQueue graphicsQueue;
    VkQueue presentQueue;

    // Swap chain
    VkSwapchainKHR swapChain;
    std::vector<VkImage> swapChainImages;
    VkFormat swapChainImageFormat;
    VkExtent2D swapChainExtent;
    std::vector<VkImageView> swapChainImageViews;

    // Rendering
    VkRenderPass renderPass;
    std::vector<VkFramebuffer> swapChainFramebuffers;
    VkCommandPool commandPool;
    std::vector<VkCommandBuffer> commandBuffers;

    // Synchronization
    std::vector<VkSemaphore> imageAvailableSemaphores;
    std::vector<VkSemaphore> renderFinishedSemaphores;
    std::vector<VkFence> inFlightFences;
    const int MAX_FRAMES_IN_FLIGHT = 2;
    size_t currentFrame = 0;

    // Validation layers
    const std::vector<const char*> validationLayers = {
        "VK_LAYER_KHRONOS_validation"
    };

    const std::vector<const char*> deviceExtensions = {
        VK_KHR_SWAPCHAIN_EXTENSION_NAME
    };

    #ifdef NDEBUG
    const bool enableValidationLayers = false;
    #else
    const bool enableValidationLayers = true;
    #endif

    // Initialization steps
    void initWindow();
    void initVulkan();
    void mainLoop();
    void cleanup();

    // Vulkan setup
    void createInstance();
    void setupDebugMessenger();
    void createSurface();
    void pickPhysicalDevice();
    void createLogicalDevice();
    void createSwapChain();
    void createImageViews();
    void createRenderPass();
    void createFramebuffers();
    void createCommandPool();
    void createCommandBuffers();
    void createSyncObjects();

    // Rendering
    void drawFrame();
    void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex);

    // Helper functions
    bool checkValidationLayerSupport();
    std::vector<const char*> getRequiredExtensions();
    QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device);
    bool isDeviceSuitable(VkPhysicalDevice device);
    SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device);
    VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats);
    VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes);
    VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities);
};
```

## Step 1: Creating the Surface

The surface is the connection between Vulkan and your window system. It's platform-specific, but GLFW handles the dirty work:

```cpp
void VulkanApp::createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create window surface!");
    }
    std::cout << "✓ Surface created successfully\n";
}
```

That was easy! Too easy. Don't worry, it gets harder.

## Step 2: Picking a Physical Device

Not all GPUs are created equal. Let's be picky:

```cpp
void VulkanApp::pickPhysicalDevice() {
    uint32_t deviceCount = 0;
    vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);

    if (deviceCount == 0) {
        throw std::runtime_error("Failed to find GPUs with Vulkan support!");
    }

    std::vector<VkPhysicalDevice> devices(deviceCount);
    vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());

    // Find the best suitable device
    for (const auto& device : devices) {
        if (isDeviceSuitable(device)) {
            physicalDevice = device;
            break;
        }
    }

    if (physicalDevice == VK_NULL_HANDLE) {
        throw std::runtime_error("Failed to find a suitable GPU!");
    }

    // Print what we picked
    VkPhysicalDeviceProperties props;
    vkGetPhysicalDeviceProperties(physicalDevice, &props);
    std::cout << "✓ Selected GPU: " << props.deviceName << "\n";
}

bool VulkanApp::isDeviceSuitable(VkPhysicalDevice device) {
    // Check queue families
    QueueFamilyIndices indices = findQueueFamilies(device);

    // Check device extensions
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());

    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    bool extensionsSupported = requiredExtensions.empty();

    // Check swap chain support
    bool swapChainAdequate = false;
    if (extensionsSupported) {
        SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
        swapChainAdequate = !swapChainSupport.formats.empty() &&
                           !swapChainSupport.presentModes.empty();
    }

    return indices.isComplete() && extensionsSupported && swapChainAdequate;
}
```

## Step 3: Creating the Logical Device

Now we create our interface to the GPU:

```cpp
void VulkanApp::createLogicalDevice() {
    QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

    std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
    std::set<uint32_t> uniqueQueueFamilies = {
        indices.graphicsFamily.value(),
        indices.presentFamily.value()
    };

    float queuePriority = 1.0f;
    for (uint32_t queueFamily : uniqueQueueFamilies) {
        VkDeviceQueueCreateInfo queueCreateInfo{};
        queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
        queueCreateInfo.queueFamilyIndex = queueFamily;
        queueCreateInfo.queueCount = 1;
        queueCreateInfo.pQueuePriorities = &queuePriority;
        queueCreateInfos.push_back(queueCreateInfo);
    }

    // Specify device features we want (none for now)
    VkPhysicalDeviceFeatures deviceFeatures{};

    // Create the logical device
    VkDeviceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
    createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
    createInfo.pQueueCreateInfos = queueCreateInfos.data();
    createInfo.pEnabledFeatures = &deviceFeatures;
    createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
    createInfo.ppEnabledExtensionNames = deviceExtensions.data();

    if (enableValidationLayers) {
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();
    } else {
        createInfo.enabledLayerCount = 0;
    }

    if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create logical device!");
    }

    // Get queue handles
    vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
    vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);

    std::cout << "✓ Logical device created\n";
}
```

## Step 4: Creating the Swap Chain

The swap chain is where the magic happens. It's a queue of images waiting to be shown on screen:

```cpp
void VulkanApp::createSwapChain() {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);

    // How many images in the swap chain?
    // Minimum + 1 for triple buffering
    uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;

    if (swapChainSupport.capabilities.maxImageCount > 0 &&
        imageCount > swapChainSupport.capabilities.maxImageCount) {
        imageCount = swapChainSupport.capabilities.maxImageCount;
    }

    VkSwapchainCreateInfoKHR createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
    createInfo.surface = surface;
    createInfo.minImageCount = imageCount;
    createInfo.imageFormat = surfaceFormat.format;
    createInfo.imageColorSpace = surfaceFormat.colorSpace;
    createInfo.imageExtent = extent;
    createInfo.imageArrayLayers = 1;  // Always 1 unless stereoscopic 3D
    createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;

    // Handle queue families
    QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
    if (!indices.isComplete()) {
        throw std::runtime_error("Required queue families not found!");
    }
    uint32_t queueFamilyIndices[] = {
        indices.graphicsFamily.value(),
        indices.presentFamily.value()
    };

    if (indices.graphicsFamily != indices.presentFamily) {
        createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
        createInfo.queueFamilyIndexCount = 2;
        createInfo.pQueueFamilyIndices = queueFamilyIndices;
    } else {
        createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    }

    createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
    createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
    createInfo.presentMode = presentMode;
    createInfo.clipped = VK_TRUE;  // Don't care about obscured pixels
    createInfo.oldSwapchain = VK_NULL_HANDLE;

    if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create swap chain!");
    }

    // Get swap chain images
    vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
    swapChainImages.resize(imageCount);
    vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());

    swapChainImageFormat = surfaceFormat.format;
    swapChainExtent = extent;

    std::cout << "✓ Swap chain created with " << imageCount << " images\n";
}
```

## Step 5: Creating the Render Pass

The render pass describes the framebuffer attachments and how they'll be used:

```cpp
void VulkanApp::createRenderPass() {
    // Color attachment description
    VkAttachmentDescription colorAttachment{};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;  // Clear to color
    colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;  // Keep the result
    colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;

    // Subpass
    VkAttachmentReference colorAttachmentRef{};
    colorAttachmentRef.attachment = 0;
    colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpass{};
    subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.colorAttachmentCount = 1;
    subpass.pColorAttachments = &colorAttachmentRef;

    // Subpass dependency
    VkSubpassDependency dependency{};
    dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
    dependency.dstSubpass = 0;
    dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependency.srcAccessMask = 0;
    dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;

    // Create render pass
    VkRenderPassCreateInfo renderPassInfo{};
    renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
    renderPassInfo.attachmentCount = 1;
    renderPassInfo.pAttachments = &colorAttachment;
    renderPassInfo.subpassCount = 1;
    renderPassInfo.pSubpasses = &subpass;
    renderPassInfo.dependencyCount = 1;
    renderPassInfo.pDependencies = &dependency;

    if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
        throw std::runtime_error("Failed to create render pass!");
    }

    std::cout << "✓ Render pass created\n";
}
```

## Handling Window Resizes

Swap chains can become out of date when the window is resized:

```cpp
void VulkanApp::recreateSwapChain() {
    int width = 0, height = 0;
    glfwGetFramebufferSize(window, &width, &height);
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createRenderPass();
    createFramebuffers();
    createCommandBuffers();
}

void VulkanApp::cleanupSwapChain() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    vkFreeCommandBuffers(device, commandPool, static_cast<uint32_t>(commandBuffers.size()),
                        commandBuffers.data());

    vkDestroyRenderPass(device, renderPass, nullptr);

    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}
```

## The Main Draw Function

Here's where it all comes together:

```cpp
void VulkanApp::drawFrame() {
    // Wait for previous frame
    vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

    // Acquire image from swap chain
    uint32_t imageIndex;
    VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX,
        imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    if (result == VK_ERROR_OUT_OF_DATE_KHR) {
        recreateSwapChain();
        return;
    } else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
        throw std::runtime_error("Failed to acquire swap chain image!");
    }

    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    // Record command buffer
    vkResetCommandBuffer(commandBuffers[currentFrame], 0);
    recordCommandBuffer(commandBuffers[currentFrame], imageIndex);

    // Submit command buffer
    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};
    VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
    submitInfo.waitSemaphoreCount = 1;
    submitInfo.pWaitSemaphores = waitSemaphores;
    submitInfo.pWaitDstStageMask = waitStages;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffers[currentFrame];

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphores[currentFrame]};
    submitInfo.signalSemaphoreCount = 1;
    submitInfo.pSignalSemaphores = signalSemaphores;

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
        throw std::runtime_error("Failed to submit draw command buffer!");
    }

    // Present the image
    VkPresentInfoKHR presentInfo{};
    presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;
    presentInfo.waitSemaphoreCount = 1;
    presentInfo.pWaitSemaphores = signalSemaphores;

    VkSwapchainKHR swapChains[] = {swapChain};
    presentInfo.swapchainCount = 1;
    presentInfo.pSwapchains = swapChains;
    presentInfo.pImageIndices = &imageIndex;

    result = vkQueuePresentKHR(presentQueue, &presentInfo);

    if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
        recreateSwapChain();
    } else if (result != VK_SUCCESS) {
        throw std::runtime_error("Failed to present swap chain image!");
    }

    currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```

## Recording Command Buffers

This is where we tell the GPU what to do:

```cpp
void VulkanApp::recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex) {
    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;

    if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
        throw std::runtime_error("Failed to begin recording command buffer!");
    }

    // Start render pass
    VkRenderPassBeginInfo renderPassInfo{};
    renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
    renderPassInfo.renderPass = renderPass;
    renderPassInfo.framebuffer = swapChainFramebuffers[imageIndex];
    renderPassInfo.renderArea.offset = {0, 0};
    renderPassInfo.renderArea.extent = swapChainExtent;

    // Clear color (beautiful cornflower blue)
    VkClearValue clearColor = {{{0.0f, 0.2f, 0.4f, 1.0f}}};
    renderPassInfo.clearValueCount = 1;
    renderPassInfo.pClearValues = &clearColor;

    vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);

    // Here's where we'd draw things... if we had any

    vkCmdEndRenderPass(commandBuffer);

    if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
        throw std::runtime_error("Failed to record command buffer!");
    }
}
```

## Exercise 3.1: Color Cycler

Make the clear color change over time:

```cpp
void VulkanApp::recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex) {
    // ... setup ...

    // Animated color!
    static float hue = 0.0f;
    hue += 0.001f;
    if (hue > 1.0f) hue -= 1.0f;

    // HSV to RGB conversion (simplified)
    float r = std::abs(hue * 6.0f - 3.0f) - 1.0f;
    float g = 2.0f - std::abs(hue * 6.0f - 2.0f);
    float b = 2.0f - std::abs(hue * 6.0f - 4.0f);

    VkClearValue clearColor = {{{
        std::max(0.0f, std::min(1.0f, r)),
        std::max(0.0f, std::min(1.0f, g)),
        std::max(0.0f, std::min(1.0f, b)),
        1.0f
    }}};

    // ... rest of function ...
}
```

## Exercise 3.2: FPS Counter

Add a simple FPS counter:

```cpp
void VulkanApp::mainLoop() {
    auto lastTime = std::chrono::high_resolution_clock::now();
    int frameCount = 0;

    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();

        frameCount++;
        auto currentTime = std::chrono::high_resolution_clock::now();
        float delta = std::chrono::duration<float, std::chrono::seconds::period>
                      (currentTime - lastTime).count();

        if (delta >= 1.0f) {
            std::cout << "FPS: " << frameCount << "\n";
            frameCount = 0;
            lastTime = currentTime;
        }
    }

    vkDeviceWaitIdle(device);
}
```

## Exercise 3.3: Device Properties Explorer

Create a comprehensive report about your GPU:

```cpp
void VulkanApp::printDetailedDeviceInfo() {
    VkPhysicalDeviceProperties props;
    vkGetPhysicalDeviceProperties(physicalDevice, &props);

    std::cout << "\n=== GPU PROPERTIES ===\n";
    std::cout << "Device: " << props.deviceName << "\n";
    std::cout << "Type: ";

    switch(props.deviceType) {
        case VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU:
            std::cout << "Discrete GPU\n";
            break;
        case VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU:
            std::cout << "Integrated GPU\n";
            break;
        case VK_PHYSICAL_DEVICE_TYPE_VIRTUAL_GPU:
            std::cout << "Virtual GPU\n";
            break;
        case VK_PHYSICAL_DEVICE_TYPE_CPU:
            std::cout << "CPU (This is weird...)\n";
            break;
        default:
            std::cout << "Unknown\n";
    }

    std::cout << "Vendor ID: " << props.vendorID << "\n";
    std::cout << "Device ID: " << props.deviceID << "\n";

    std::cout << "\n=== LIMITS ===\n";
    std::cout << "Max Image Dimension 2D: " << props.limits.maxImageDimension2D << "\n";
    std::cout << "Max Uniform Buffer Range: " << props.limits.maxUniformBufferRange << "\n";
    std::cout << "Max Push Constants Size: " << props.limits.maxPushConstantsSize << "\n";
    std::cout << "Max Memory Allocation Count: " << props.limits.maxMemoryAllocationCount << "\n";

    std::cout << "\n=== FEATURES ===\n";
    VkPhysicalDeviceFeatures features;
    vkGetPhysicalDeviceFeatures(physicalDevice, &features);

    std::cout << "Geometry Shader: " << (features.geometryShader ? "YES" : "NO") << "\n";
    std::cout << "Tessellation Shader: " << (features.tessellationShader ? "YES" : "NO") << "\n";
    std::cout << "Multi Viewport: " << (features.multiViewport ? "YES" : "NO") << "\n";
    std::cout << "Anisotropic Filtering: " << (features.samplerAnisotropy ? "YES" : "NO") << "\n";
}
```

## The Complete Application

When you put it all together, you get an application that:
1. Creates a window
2. Initializes Vulkan
3. Clears the screen to a color
4. Runs at your monitor's refresh rate
5. Properly cleans everything up

It's 800+ lines of code to do what OpenGL does in 50, but now you control EVERYTHING.

## Common Issues and Solutions

**"Swap chain out of date"**
- Window was resized
- Solution: Recreate swap chain

**"Lost device"**
- GPU crash or driver timeout
- Solution: Recreate everything (yes, really)

**"Validation error: Command buffer is in invalid state"**
- Forgot to begin/end recording
- Solution: Check your begin/end pairs

**"Black screen but no errors"**
- Clear color might be black
- Present mode might be wrong
- Check if drawFrame is being called

## What We've Accomplished

You've now written a complete Vulkan application that:
- Properly initializes all Vulkan objects
- Manages the swap chain
- Records and submits command buffers
- Synchronizes CPU and GPU operations
- Clears the screen to a color

This is the foundation for everything else. Every Vulkan application needs these components.

## Performance Considerations

Even for clearing the screen, we can optimize:

1. **Pre-record command buffers** when possible
2. **Use multiple frames in flight** (we're using 2)
3. **Choose the right present mode**:
   - IMMEDIATE: No vsync, may tear
   - FIFO: Vsync (we're using this)
   - MAILBOX: Triple buffering
   - FIFO_RELAXED: Adaptive vsync

## What's Next

In Chapter 4, we'll finally draw a triangle. It'll require:
- Vertex buffers
- Vertex shaders
- Fragment shaders
- Graphics pipeline creation
- Even MORE boilerplate

But that triangle will be YOUR triangle, rendered with YOUR pipeline, and nobody can take that away from you.

## A Reality Check

If you're thinking "this is insane for just clearing the screen," you're absolutely right. But remember:
- This same code scales to AAA games
- You now control every aspect of rendering
- Modern engines hide this complexity for you
- You understand what's actually happening

Plus, you can now casually mention at parties that you write Vulkan code, and watch people's eyes glaze over.

---

*Next Chapter Preview: The triangle chapter. 1000 lines of code. One triangle. Three vertices. Infinite satisfaction.*