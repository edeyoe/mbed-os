# CMake Mbed OS

Requirements:
- CMake 3.19.0 and higher
- `mbed-tools` (python 3.6 and higher)

Two steps approach:

- Mbed OS core CMake
- Building an application with mbed-tools

Definitions and configurations would be defined in CMake files, an Mbed app configuration file (`/path/to/app/mbed_app.json`) and Mbed library configuration files (`/path/to/mbed-os/<LIBRARY_NAME>/mbed_lib.json`). `mbed-tools` would parse the Mbed library and app configuration files and generate an `mbed_config.cmake` file.

The following rules must be respected to include special prefix directories (TARGET_, COMPONENT_ and FEATURE_).

Target labels, components, and features defined in `/path/to/mbed-os/targets/targets.json` are used to help the build system determine which directories contain sources/include files to use in the build process. 

This is a problem as labels, components and features directories will most likely be renamed and will not necessarily use the special prefixes TARGET_, COMPONENT_ and FEATURE_ respectively. Whenever such directories are encountered, check the directory name without the special prefix in the CMake list `MBED_TARGET_LABELS` generated by `mbed-tools` then include the subdirectory to the build process if found.

For example when the user builds for the `NUCLEO_F411RE` Mbed target, `mbed-tools` includes the `STM` label to the  `MBED_TARGET_LABELS` list. `TARGET_STM` can only be added to the list of build directories as shown below:

```
# CMake input source file: mbed-os/targets/CMakeList.txt

if("STM" IN_LIST MBED_TARGET_LABELS)
    add_subdirectory(TARGET_STM)
endif()
```
The same could be applied to other labels like features or components.

## Mbed OS Core (Mbed OS repository)

There are numerous CMake files in the Mbed OS repository tree:

* A `CMakeLists.txt` entry point in the Mbed OS root, describing the top level build specification for the Mbed OS source tree.
* `CMakeLists.txt` entry points in each Mbed OS module subdirectory, describing the build specification for a module or component

A number of CMake scripts are contained in the `mbed-os/tools/cmake` directory:
* `core.cmake` - selects the core script from the `cmake/cores` directory, based on the value of the `MBED_CPU_CORE` variable
* `profile.cmake` - selects the profile script from the `cmake/profiles` directory, based on the value of the `MBED_PROFILE` variable
* `toolchain.cmake` - selects the toolchain script from the `cmake/toolchains` directory, based on the value of the `MBED_TOOLCHAIN` variable

The next sections will describe static CMake files within Mbed OS Core repository.

### 1. Mbed OS `CMakeLists.txt` Entry Point

The `CMakeLists.txt` entry point in the root of the Mbed OS repository contains the top level build specification for Mbed OS. This file also includes the auto generated `mbed_config.cmake` script, which is created by `mbed-tools`.

This is not intended to be included by an application. Applications must use `add_subdirectory(${MBED_PATH})`.

### 2. Toolchain CMake Scripts

All the toolchain settings are defined in the scripts found in `cmake/toolchains/`.

### 3. Profile CMake Scripts

The build profiles such as release or debug are defined in the scripts found in `cmake/profiles/`.

### 4. MCU Core CMake Scripts

The MCU core definitions are defined in the scripts found in `cmake/cores/`.

### 5. Utilities CMake Scripts

Custom functions/macros used within Mbed OS.

### 7. Component `CMakeLists.txt` Entry Point

This file statically defines the build specification of an Mbed OS component. It contains conditional statements that depend on the configuration parameters generated by `mbed-tools`.
The rule of thumb is to not expose header files that are internal. We would like to avoid having everything in the include paths as we do now.

## Building an Application

`mbed-tools` is the next generation of command line tooling for Mbed OS. `mbed-tools` replaces `mbed-cli` and the Python modules in the `mbed-os/tools` directory.

`mbed-tools` consolidates all of the required modules to build Mbed OS, along with the command line interface, into a single Python package which can be installed using standard Python packaging tools.

Each application contains a top-level CMakeLists.txt file. The `mbedtools new` command can create this top-level CMakeLists.txt, or a user can create it manually. Each application also has a number of json configuration files. `mbedtools configure` creates an Mbed configuration CMake file (`.mbedbuild/mbed_config.cmake`). The process for building an application looks like:

1. Parse the arguments provided to build command
1. Parse the application configuration
1. Get the target configuration
1. Get the Mbed OS configuration (select what modules we need and get their config, paths, etc)
1. Create .mbedbuild/mbed_config.cmake
1. Build an application

### Configuration

The main purpose of `mbed-tools` is to parse the Mbed configuration system's JSON files (`mbed_lib.json`, `mbed_app.json` and `targets.json`). The tool outputs a single CMake configuration script, which is included by `app.cmake` and `mbed-os/CMakeLists.txt`.

To generate the CMake config script (named `mbed_config.cmake`) the user can run the `configure` command:

`mbedtools configure -t <toolchain> -m <target>`

This will output `mbed_config.cmake` in a directory named `.mbedbuild` at the root of the program tree.

`mbed_config.cmake` contains several variable definitions used to select the toolchain, core and profile CMake scripts to be used in the build system generation:
* `MBED_TOOLCHAIN`
* `MBED_TARGET`
* `MBED_CPU_CORE`
* `MBED_C_LIB`
* `MBED_TARGET_SUPPORTED_C_LIBS`
* `MBED_PRINTF_LIB`

The tools also generate an `MBED_TARGET_LABELS` variable, containing the labels, components and feature definitions from `targets.json`, used to select the required Mbed OS components to be built.

The macro definitions parsed from the Mbed OS configuration system are also included in `mbed_config.cmake`, so it is no longer necessary for the tools to generate any `mbed_config.h` file to define macros.

### mbedignore

With a build system that can understand dependencies between applications, features, components, and targets, `mbedignore` is no longer needed. CMake creates a list of exactly what's needed for an application build, rather than do what the legacy tools did and include everything by default, leaving the application to specify what is not needed. It's far more natural to express what you need rather than what you don't need, not to mention more future proof should some new feature appear in Mbed OS which your application would need to ignore.
