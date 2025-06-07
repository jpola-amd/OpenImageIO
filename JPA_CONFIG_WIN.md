# Dev env configuration

The goal is to build the OpenShadingLanguage dependencies as easily as possible. There are two groups of items we need to handle:

## 1. AcademySoftwareFoundation components:
- openexr
- openvdb + nanovdb (with AMDGPU support)
- opencolorio
- openimageio

## 2. Rest of the dependencies required by the ASF components:
For this we will use vcpkg.

## 3. LLVM
The safest option is to use the same version that is used by the ROCm that we will use for OSL.

---

# VCPKG

I want to download most of the dependencies, so I picked vcpkg as a provider.

```
VCPKG_root = D:\vcpkg
```

Required dependencies:
```shell
vcpkg.exe install jpeg pybind11 libpng freetype tbb giflib opencv ffmpeg libheif tiff freeType libjpeg-turbo OpenEXR oplibpng zlib libjxl minizip-ng fmt boost blosc ffmpeg bzip brotli giflib glew glut libheif liblzma libpng libraw libwebp opencv opengl openjpeg ptex pugixml robin-hood-hashing robin-map tbb yaml-cpp zstd
```
I hope that's all.

All the ASF components and LLVM are installed by `CMAKE_INSTALL_PREFIX` in `D:\Sandbox\OpenShadingLanguage\dist`. In total, we will have only two directories where we store the dependencies:
1. ASF + LLVM: `D:\Sandbox\OpenShadingLanguage\dist`
2. VCPKG: `D:\vcpkg\installed\x64-windows`

---

## Custom projects

### 1. llvm-project
The commit that is matching my version of ROCm. llvm-project is required for building OSL, ROCm installation does not provide the llvm project.

```
clang version 20.0.0git (git@github.amd.com:Compute-Mirrors/llvm-project 9dfb54abb25a32d9adf38f58b9a78d922f17b167)
Target: x86_64-pc-windows-msvc
Thread model: posix
InstalledDir: C:\opt\rocm-downloads\rocm-hipsdk-6.4-48\bin
```

**Important**: Use out of source build 
I used cmake-gui. The main cmake is `llvm-project/llvm`, Enable X86, AMDGPU, NVPTX targets. 
Enable clang. Visual Studio 2022 builds the project without any issue.

### 2. IMath
https://github.com/jpola-amd/Imath branch amd/amdgcn

### 3. OpenEXR
```
git@github.com:AcademySoftwareFoundation/openexr.git
cmake ..\ -DImath_DIR=D:\Sandbox\OpenShadingLanguage\dist\lib\cmake\Imath -DCMAKE_INSTALL_PREFIX=D:\Sandbox\OpenShadingLanguage\dist
cmake --build ./ --config Release --target install
```

### 4. OpenVDB
```
git@github.com:AcademySoftwareFoundation/openvdb.git
```
Totally manual, I used cmake-gui and set all dependencies by hand :/

### 5. OpenColorIO

**Change:** Problem with function `glErrorString`  
`D:\Sandbox\OpenShadingLanguage\3rdParty-sources\OpenColorIO\src\libutils\oglapphelpers\glsl.cpp`

We could change it to:
```cpp
bool GetGLError(std::string & error)
{
    const GLenum glErr = glGetError();
    if(glErr!=GL_NO_ERROR)
    {
#ifdef __APPLE__
        // Unfortunately no gluErrorString equivalent on Mac.
        error = "OpenGL Error";
#else
        error = (const char*)glErrorStringREGAL(glErr); // <-----
#endif
        return true;
    }
    return false;
}
```

#### Missing minizip-ng requirements for OpenColorIO, ocioarchive, test_cpu

Open the VS project and add some dependencies that minizip-ng requires:
- `D:\vcpkg\installed\x64-windows\lib\lzma.lib`
- `D:\vcpkg\installed\x64-windows\lib\bz2.lib`
- `D:\vcpkg\installed\x64-windows\lib\zstd.lib`
- `bcrypt.lib`

#### Weird step but required (Important):

Generate the OpenImageIO initial config to generate `build/include/OpenImageIO/version` file. This must be provided to OpenColorIO configuration to get the namespace prefix.

---

# Another install configuration

It is required to provide the full absolute install paths to all the components (`bin/`, `include/`, `shared/`, etc.). Otherwise, the installation will not work!

## cmake config

```shell
cmake ..\ -DCMAKE_MODULE_PATH=D:/vcpkg/installed/x64-windows -DGLEW_ROOT=D:/vcpkg/installed/x64-windows -DGLUT_ROOT=D:/vcpkg/installed/x64-windows -DOpenEXR_ROOT=D:\Sandbox\OpenShadingLanguage\dist\ -DOCIO_BUILD_DOCS=OFF -DCMAKE_INSTALL_PREFIX=D:\Sandbox\OpenShadingLanguage\dist
```

---

## OpenImageIO: https://github.com/jpola-amd/OpenImageIO branch: amd/hip

### 3.1 OpenImageIO configuration

* `mkdir build`
* 
    ```shell
    cmake ..\ -DZLIB_ROOT=D:\vcpkg\installed\x64-windows -DImath_ROOT=D:\Sandbox\OpenShadingLanguage\dist -DOpenEXR_ROOT=D:\vcpkg\installed\x64-windows -DJPEG_ROOT=D:\vcpkg\installed\x64-windows -DTIFF_ROOT=D:\vcpkg\installed\x64-windows -Dlibjpeg-turbo_ROOT=D:\vcpkg\installed\x64-windows -Dpybind11_ROOT=D:\vcpkg\installed\x64-windows -DPNG_ROOT=D:\vcpkg\installed\x64-windows -DFreetype_ROOT=D:\vcpkg\installed\x64-windows -DOpenColorIO_ROOT=D:\vcpkg\installed\x64-windows -DTBB_ROOT=D:\vcpkg\installed\x64-windows -DGIF_ROOT=D:\vcpkg\installed\x64-windows -DOpenCV_ROOT=D:\vcpkg\installed\x64-windows -DPtex_ROOT=D:\vcpkg\installed\x64-windows -DFFmpeg_ROOT=D:\vcpkg\installed\x64-windows -DLibheif_ROOT=D:\vcpkg\installed\x64-windows -DQt5_ROOT=C:\Qt\5.15.2\msvc2019_64\lib\cmake -DOpenVDB_ROOT=D:\vcpkg\installed\x64-windows\ -DCMAKE_PREFIX_PATH=D:\vcpkg\installed\x64-windows\
    ```
    Or use my `CMakePresets.json` and adjust for your needs.