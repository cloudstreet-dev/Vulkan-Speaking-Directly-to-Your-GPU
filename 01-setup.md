# Chapter 1: Setting Up Your Vulkan Development Environment

## Or: Installing 47 Dependencies to Draw One Triangle

Welcome back! You've survived the introduction, which means you're either very determined or very stubborn. Both are excellent qualities for Vulkan development. In this chapter, we'll set up everything you need to start writing Vulkan code. Fair warning: this is like setting up a home chemistry lab, except the chemicals are SDK versions and the explosions are segmentation faults.

## The Shopping List

Here's what we need to install:
1. **A C++ compiler** (C++17 or later)
2. **CMake** (for building our projects)
3. **The Vulkan SDK** (the star of the show)
4. **GLFW** (for creating windows)
5. **GLM** (for mathematics that doesn't make you cry)
6. **A good debugger** (you'll be using this A LOT)
7. **Validation layers** (your new best friend)

## Platform-Specific Setup

### Windows: The Land of DirectX Refugees

First, let's check if you have a Vulkan-capable GPU:
1. Download GPU-Z or GPU Caps Viewer
2. Look for "Vulkan" support
3. If it says "No," it's time for a GPU upgrade (or tears)

**Installing the Tools:**

```bash
# If you have Chocolatey (and you should):
choco install cmake
choco install vulkan-sdk

# Or download manually:
# Visual Studio 2019/2022: https://visualstudio.microsoft.com/
# Vulkan SDK: https://vulkan.lunarg.com/sdk/home#windows
# CMake: https://cmake.org/download/
```

**Setting up Visual Studio:**
1. Install Visual Studio 2019 or 2022 (Community edition is fine)
2. During installation, select "Desktop development with C++"
3. Also grab "CMake tools for Windows" if you're feeling fancy

### macOS: The Metal Rebellion

Ah, macOS. Apple's walled garden where Vulkan runs through MoltenVK (a translation layer to Metal). It's like speaking French to someone who only understands Japanese, using Google Translate. What could go wrong?

```bash
# Install Homebrew if you haven't already
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install the tools
brew install cmake
brew install glfw
brew install glm

# Install Vulkan SDK
# Download from: https://vulkan.lunarg.com/sdk/home#mac
# After installation, add to your shell profile (.zshrc or .bash_profile):
export VULKAN_SDK=/usr/local/VulkanSDK/[version]/macOS
export PATH=$VULKAN_SDK/bin:$PATH
export DYLD_LIBRARY_PATH=$VULKAN_SDK/lib:$DYLD_LIBRARY_PATH
export VK_ICD_FILENAMES=$VULKAN_SDK/share/vulkan/icd.d/MoltenVK_icd.json
export VK_LAYER_PATH=$VULKAN_SDK/share/vulkan/explicit_layer.d
```

**Note for Mac users:** You're technically not running "real" Vulkan. You're running Vulkan-on-top-of-Metal. It's like vegan bacon – it works, but purists will judge you.

### Linux: The Promised Land

Linux users, you chose the path of righteousness. Your package manager has your back.

**Ubuntu/Debian:**
```bash
# Update your package list
sudo apt update

# Install the essentials
sudo apt install build-essential cmake
sudo apt install vulkan-tools
sudo apt install libvulkan-dev vulkan-validationlayers-dev
sudo apt install libglfw3-dev
sudo apt install libglm-dev

# Test if Vulkan is working
vulkaninfo
```

**Arch Linux (I use Arch BTW):**
```bash
sudo pacman -S vulkan-devel cmake glfw-wayland glm
```

**Fedora:**
```bash
sudo dnf install vulkan-tools vulkan-loader-devel \
    vulkan-validation-layers-devel glfw-devel cmake glm-devel
```

## Verifying Your Setup

Time to check if everything works. Run this command:

```bash
vulkaninfo
```

If you see a wall of text describing your GPU's capabilities, congratulations! You're ready. If you see an error, welcome to the first of many debugging sessions.

Common issues:
- **"Cannot find vulkaninfo"**: Your PATH isn't set up correctly
- **"No Vulkan devices found"**: Your drivers need updating
- **Segmentation fault**: Classic. Check your drivers
- **Works but shows Intel GPU instead of NVIDIA/AMD**: Laptop? You need to configure your discrete GPU

## Project Structure: Meet Project Pineapple

Let's create our project structure. We'll build on this throughout the book:

```
project-pineapple/
├── CMakeLists.txt
├── src/
│   ├── main.cpp
│   ├── VulkanApp.hpp
│   ├── VulkanApp.cpp
│   └── shaders/
│       ├── vertex.vert
│       └── fragment.frag
├── assets/
│   ├── models/
│   │   └── pineapple.obj
│   └── textures/
│       └── pineapple.png
└── build/
    └── (generated files)
```

## Your First CMakeLists.txt

Here's our CMake configuration. It's like a recipe, but instead of "bake at 350°F," it's "link against vulkan-1.lib":

```cmake
cmake_minimum_required(VERSION 3.16)
project(ProjectPineapple)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find packages
find_package(Vulkan REQUIRED)
find_package(glfw3 REQUIRED)
find_package(glm REQUIRED)

# Add executable
add_executable(ProjectPineapple
    src/main.cpp
    src/VulkanApp.cpp
)

# Include directories
target_include_directories(ProjectPineapple PRIVATE
    ${Vulkan_INCLUDE_DIRS}
    ${GLM_INCLUDE_DIRS}
)

# Link libraries
target_link_libraries(ProjectPineapple
    ${Vulkan_LIBRARIES}
    glfw
    ${CMAKE_DL_LIBS}
)

# Shader compilation
file(GLOB_RECURSE GLSL_SOURCE_FILES
    "${PROJECT_SOURCE_DIR}/src/shaders/*.frag"
    "${PROJECT_SOURCE_DIR}/src/shaders/*.vert"
)

foreach(GLSL ${GLSL_SOURCE_FILES})
    get_filename_component(FILE_NAME ${GLSL} NAME)
    set(SPIRV "${PROJECT_BINARY_DIR}/shaders/${FILE_NAME}.spv")
    add_custom_command(
        OUTPUT ${SPIRV}
        COMMAND ${CMAKE_COMMAND} -E make_directory "${PROJECT_BINARY_DIR}/shaders/"
        COMMAND ${Vulkan_GLSLC_EXECUTABLE} ${GLSL} -o ${SPIRV}
        DEPENDS ${GLSL}
    )
    list(APPEND SPIRV_BINARY_FILES ${SPIRV})
endforeach(GLSL)

add_custom_target(
    Shaders
    DEPENDS ${SPIRV_BINARY_FILES}
)

add_dependencies(ProjectPineapple Shaders)

# Platform-specific settings
if(APPLE)
    target_link_libraries(ProjectPineapple
        "-framework Cocoa"
        "-framework IOKit"
        "-framework CoreVideo"
    )
endif()
```

## The Simplest Vulkan Program That Does Nothing

Let's write our first Vulkan program. It will create a Vulkan instance and immediately destroy it. That's it. 100 lines of code to do essentially nothing. Welcome to Vulkan!

**src/main.cpp:**
```cpp
#include <vulkan/vulkan.h>
#include <GLFW/glfw3.h>
#include <iostream>
#include <stdexcept>
#include <vector>
#include <cstring>
#include <cstdlib>

class HelloVulkan {
public:
    void run() {
        initWindow();
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    GLFWwindow* window;
    VkInstance instance;

    void initWindow() {
        glfwInit();
        glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
        glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
        window = glfwCreateWindow(800, 600, "Project Pineapple", nullptr, nullptr);
    }

    void initVulkan() {
        createInstance();
    }

    void createInstance() {
        // Application info - telling Vulkan about our app
        VkApplicationInfo appInfo{};
        appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
        appInfo.pApplicationName = "Project Pineapple";
        appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
        appInfo.pEngineName = "No Engine (We're Hardcore)";
        appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
        appInfo.apiVersion = VK_API_VERSION_1_2;

        // Instance create info - what extensions do we need?
        VkInstanceCreateInfo createInfo{};
        createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
        createInfo.pApplicationInfo = &appInfo;

        // Get required extensions from GLFW
        uint32_t glfwExtensionCount = 0;
        const char** glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

        createInfo.enabledExtensionCount = glfwExtensionCount;
        createInfo.ppEnabledExtensionNames = glfwExtensions;

        // Enable validation layers in debug mode
        #ifndef NDEBUG
        const std::vector<const char*> validationLayers = {
            "VK_LAYER_KHRONOS_validation"
        };
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();
        #else
        createInfo.enabledLayerCount = 0;
        #endif

        // Finally create the instance!
        if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
            throw std::runtime_error("Failed to create Vulkan instance!");
        }

        std::cout << "Vulkan instance created successfully! 🎉" << std::endl;
    }

    void mainLoop() {
        while (!glfwWindowShouldClose(window)) {
            glfwPollEvents();
        }
    }

    void cleanup() {
        vkDestroyInstance(instance, nullptr);
        glfwDestroyWindow(window);
        glfwTerminate();
    }
};

int main() {
    HelloVulkan app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

## Building and Running

```bash
# Create build directory
mkdir build
cd build

# Configure with CMake
cmake ..

# Build
cmake --build .

# Run
./ProjectPineapple  # On Linux/Mac
ProjectPineapple.exe  # On Windows
```

If everything works, you'll see:
- A black window titled "Project Pineapple"
- Console output: "Vulkan instance created successfully! 🎉"
- No crashes (hopefully)

## Understanding Validation Layers

Validation layers are like having a very pedantic friend watching over your shoulder, pointing out every tiny mistake. They're annoying but invaluable.

Enable them by setting the `DEBUG` flag:
```bash
cmake -DCMAKE_BUILD_TYPE=Debug ..
```

With validation layers enabled, Vulkan will tell you when you:
- Forget to destroy objects (memory leaks)
- Use objects incorrectly
- Pass invalid parameters
- Do things in the wrong order
- Generally mess up

The messages look like this:
```
Validation layer: Validation Error: You forgot to set the sType field, you absolute muppet
```

(Okay, they're more professional than that, but you get the idea.)

## Exercise 1.1: Enumeration Madness

Let's explore what Vulkan extensions and layers are available on your system:

```cpp
void listAvailableExtensions() {
    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> extensions(extensionCount);
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());

    std::cout << "Available extensions (" << extensionCount << "):\n";
    for (const auto& extension : extensions) {
        std::cout << "\t" << extension.extensionName << " (v"
                  << extension.specVersion << ")\n";
    }
}
```

Call this function before creating your instance. You'll see things like:
- `VK_KHR_surface` - for rendering to windows
- `VK_EXT_debug_utils` - for debugging
- Various platform-specific extensions

## Exercise 1.2: Validation Layer Check

Add this function to check if validation layers are available:

```cpp
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    const std::vector<const char*> validationLayers = {
        "VK_LAYER_KHRONOS_validation"
    };

    for (const char* layerName : validationLayers) {
        bool layerFound = false;

        for (const auto& layerProperties : availableLayers) {
            if (strcmp(layerName, layerProperties.layerName) == 0) {
                layerFound = true;
                break;
            }
        }

        if (!layerFound) {
            return false;
        }
    }

    return true;
}
```

## Common Setup Issues and Solutions

**"I get a black window but it immediately closes"**
- Check your mainLoop() function
- Make sure glfwPollEvents() is being called

**"vkCreateInstance returns VK_ERROR_INCOMPATIBLE_DRIVER"**
- Your GPU drivers need updating
- You might be running in a VM without GPU passthrough

**"Cannot find validation layers"**
- Install the Vulkan SDK properly
- On Linux, install vulkan-validation-layers package

**"It compiles but crashes immediately"**
- Welcome to Vulkan! Check the debugger
- 90% chance you forgot to initialize something

## What We've Learned

Today we:
1. Set up a complete Vulkan development environment
2. Created our first Vulkan instance
3. Learned that even "Hello World" is complicated in Vulkan
4. Discovered validation layers (our new best friend)
5. Built the foundation for Project Pineapple

## What's Next

In Chapter 2, we'll dive deep into Vulkan's architecture. We'll learn about:
- Physical vs. logical devices
- Queues and queue families
- The command buffer system
- Synchronization primitives
- Why Vulkan has 47 different object types for seemingly simple tasks

But for now, celebrate! You've successfully set up Vulkan and created your first instance. That's more than many people achieve before rage-quitting.

## Bonus Exercise: The Vulkan Info Dump

Create a function that prints detailed information about your GPU:

```cpp
void printGPUInfo() {
    uint32_t deviceCount = 0;
    vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);

    std::vector<VkPhysicalDevice> devices(deviceCount);
    vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());

    for (const auto& device : devices) {
        VkPhysicalDeviceProperties props;
        vkGetPhysicalDeviceProperties(device, &props);

        std::cout << "GPU: " << props.deviceName << "\n";
        std::cout << "Type: ";
        switch(props.deviceType) {
            case VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU:
                std::cout << "Discrete GPU (The good stuff!)\n";
                break;
            case VK_PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU:
                std::cout << "Integrated GPU (It'll do...)\n";
                break;
            default:
                std::cout << "Unknown (Mysterious!)\n";
        }
        std::cout << "Vulkan Version: "
                  << VK_VERSION_MAJOR(props.apiVersion) << "."
                  << VK_VERSION_MINOR(props.apiVersion) << "."
                  << VK_VERSION_PATCH(props.apiVersion) << "\n";
    }
}
```

Remember: We're just getting started. The real fun begins when we actually try to draw something!

---

*Next Chapter Preview: We'll learn why Vulkan needs 15 different objects just to clear the screen to blue. Spoiler: It's for MAXIMUM PERFORMANCE.*