# Installation guide for MOSCAL2.0

`MOSCAL2.0` is a very useful software to study open quantun systems. The software is developed by one of our collaborators, Dr. YiJing Yan's group.
The project is now open on [the USTC gitlab page](https://git.lug.ustc.edu.cn/czh123/moscal2.0). 

!!! info "TLDR"
    The following is a simple description of the `MOSCAL2.0` pacakge, if you only want to compile the library, you can skip this and just go to the minimal build section.

The `MOSCAL2.0` program is written in `c++` and a user interface written in `python`. The design idea is to use simple python scripts to 
generate inputs for the `c++` library. Therefore, without knowing the `c++` internals of `MOSCAL2.0`, the user can just write python scripts and 
simulate open quantum systems.

The `MOSCAL2.0` has the following technical advantages comparing to other implementations, for example [`HierarchicalEOM.jl`](https://github.com/NCKU-QFort/HierarchicalEOM.jl):

- `MOSCAL2.0` has a clean theory: the library is built based on the idea of **Dissipatons**, a quasi-particle describing system-bath interactions. In this idea, the auxiliary density operators (ADOs) are regarded as different degrees of excitations of the non-interacting system, i.e., the dissipaton density operators (DDOs).  Such theoretical designs help our collaborator derive useful algebras to efficiently manipulate DDOs, more importantly derive new applications such as quadratic-system bath coupling easily.
- `MOSCAL2.0` is Efficient: the library is written `c++`, with powerful parallelization and filtering schemes:
    - Many operations on the DDOs are written in parallel algorithms, and these algorithms are carefully designed to be thread-safe.
    - Instead of make a hard truncation over the hierarchical tiers and use a fixed propagation matrix, `MOSCAL2.0` applies a dynamic filtering scheme. Particularly, the user will apply a soft hierarchical cutoff. As a result, all the possible DDOs within the maximum hierarchical depth can be considered if they are numerically significant enough. The filtering tolerance here is input by the user. With filtering, it is easy to choose a very deep hierarchical depth, with way less computation cost by taking advantage of the sparse nature of the HEOM propagator.

To harness efficiency of the `MOSCAL2.0` package, let us then go through the process of build the `c++` binaries. 

**NOTE:** If you are experienced using the `Docker`, you should follow the official instruction on [`MOSCAL2.0`](https://git.lug.ustc.edu.cn/czh123/moscal2.0) site. If you intend to build the `c++` library from scratch, follow the instructions given below.

## Build the `c++` library on you `MacOS` machine

### Minimal build
First, you clone the package on your local machine.
```bash
git clone https://git.lug.ustc.edu.cn/czh123/moscal2.0 
```

Then, you need to install the necessary dependencies of `MOSCAL2.0`.  
- The parallelization of `MOSCAL2.0` depends on `OpenMP`.
- The easy building of the `MOSCAL2.0` depends on `cmake`.
Luckily, our group member has a `MACOS` workstation. Installing these dependent packages can be simplified by
[`homebrew`](https://brew.sh).

```bash
# If you don't already have homebrew installed, 
# you can go to https://brew.sh to learn about it and install homebrew on your computer
# after brew is on you machine, just execute 
brew install libomp
echo "export OpenMP_ROOT=$(brew --prefix)/opt/libomp" >> ~/.zshrc # This ensures cmake finds OpenMP
brew install cmake
```

After all the necessary tools (`OpenMP` and `cmake`) have been installed, we can start the building process.
The first thing you should do is traversing to the `/path/to/moscal2.0/code` directory.

```bash
cd /path/to/moscal2.0/code
```

!!! info "Note"
    The current version of `moscal2.0` code has a redundant line, which spoils the minimal build. 
    We are sorry that you have to comment out one of the line in file `/path/to/moscal2.0/code/include/headfile.hpp`
    ```bash
    # open the file /path/to/moscal2.0/code/include/headfile.hpp
    # go to the line that says "include <openssl/md5.h>"
    # then comment this line out by `//`
    # hint: this line was the 14-th line in headfile.hpp (for Commit bc62005f);
    ```
    If you have done the comment, you shall have
    ```cpp
    // include <openssl/md5.h>
    ```
    If you cannot find this line, or see this line already commented. That mean this redundancy is patched.

Then, you will need to create a new file in the `code` directory named `CMakeLists.txt`. Then you copy the following `CMakeLists.txt` content to your local file. 
```cmake
cmake_minimum_required(VERSION 3.12)
project(MOSCAL2.0 LANGUAGES CXX)
set(CMAKE_BUILD_TYPE RLEASE)
set(CMAKE_CXX_STANDARD 17)   # use modern c++ standard

# The compiler flags for optimization and compile time options
add_compile_options(-O3)              # Generic maximum code optimization
add_compile_options(-ftree-vectorize) # SIMD vectorize the loops
add_compile_options(-funroll-loops)   # explicit optimize the loops
add_compile_options(-fPIE)            # make code run everywhere
add_compile_options(-Wall -Wextra -Wpedantic) # Enable warning

include(GNUInstallDirs) # Enable cmake to find the system installations

# Find openssl
# find_package(OpenSSL REQUIRED)

# Find libomp for parallelization
if (APPLE)
    # execute shell command OpenMP_ROOT=$(brew --prefix)/opt/libomp
    find_program(HOMEBREW_EXECUTABLE brew)
    if (HOMEBREW_EXECUTABLE)
        execute_process(
            COMMAND ${HOMEBREW_EXECUTABLE} --prefix
            OUTPUT_VARIABLE BREW_PREFIX
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        set(OpenMP_ROOT "${BREW_PREFIX}/opt/libomp" CACHE PATH "Path to libomp")
    endif()
endif()
find_package(OpenMP)

if(OpenMP_FOUND)
    message(STATUS "OpenMP found")
else()
    if (APPLE)
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64") # Intel Mac
            # Set default paths for libomp
            set(ENV{LDFLAGS} "-L/usr/local/opt/libomp/lib")
            set(ENV{CPPFLAGS} "-I/usr/local/opt/libomp/include")
        elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64") # Apple Silicon
            # Set default paths for arm64 Macs
            set(ENV{LDFLAGS} "-L/opt/homebrew/opt/libomp/lib")
            set(ENV{CPPFLAGS} "-I/opt/homebrew/opt/libomp/include")
        endif()
    else()
        # set the default path for any unix machine
        set(ENV{LDFLAGS} "-L/usr/local/opt/libomp/lib")
        set(ENV{CPPFLAGS} "-I/usr/local/opt/libomp/include")
    endif()

    message(WARNING "OpenMP not found, using default paths instead")
    # Set the LDFLAGS and CPPFLAGS manually
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} $ENV{LDFLAGS} $ENV{CPPFLAGS}")
endif()

# do not try to find pacakge for Eigen3, just use shipped one
# do not try to find pacakge for nlohmann, just use the shipped one

# find folly to harness the parallelization algorithms
find_package(Folly)
if(Folly_FOUND)
    message(STATUS "Found Folly: ")
    message(STATUS "Include directory: ${FOLLY_INCLUDE_DIR}")
    message(STATUS "Library: ${FOLLY_LIBRARIES}")

    # Look for gflags
    find_package(gflags)

    if(gflags_FOUND)
        message(STATUS "Found gflags:")
        message(STATUS "Include directory: ${GFLAGS_INCLUDE_DIRS}")
        message(STATUS "Libraries: ${GFLAGS_LIBRARIES}")
    else()
        message(WARNING "gflags not found.")
    endif()
else()
    message(WARNING "Folly not found. The built library will be un-parallelizable.")
endif()

# now make all the subdirectories visible to cmake
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/include/low_level # for low-level implementations of DDO EOMs
    ${CMAKE_CURRENT_SOURCE_DIR}/include/aux       # for operations on DDOs or Aux Density operators
)

### now define a function to build specifc applications
# Define a function to build the main_threads_omp_equ_corr.cpp file
function(build_population BINARY_NAME)
    add_executable(${BINARY_NAME} main_threads_omp_equ_corr.cpp)

    target_compile_definitions(${BINARY_NAME} PRIVATE ${ARGN})

    # target_link_libraries(${BINARY_NAME} PRIVATE OpenSSL::SSL)
    if (OpenMP_FOUND)
        # use OpenMP::OpenMP_CXX
        target_link_libraries(${BINARY_NAME} PRIVATE OpenMP::OpenMP_CXX)
    endif()

    if (Folly_FOUND)
    target_link_libraries(${BINARY_NAME} PRIVATE
        ${GFLAGS_LIBRARIES}
        Folly::folly
    )
    else()
        target_compile_definitions(${BINARY_NAME} PRIVATE STD)
        message(WARNING "Folly not found. Use the STD macro. This build will not be parallelized since you don't have Folly.")
    endif()

endfunction()

build_population(bose_linear_2.out NSYS=2 BOSE_LINEAR NORMAL)       # 2x2 system coupled with a linear boson environment
build_population(bose_quad_2.out NSYS=2 BOSE_QUAD NORMAL)           # 2x2 system coupled with a quadratic boson environment
build_population(bose_linear.out BOSE_LINEAR NORMAL)                # a general system (meaning the system matrix is any nxn matrix) coupled with a linear boson environment
build_population(bose_quad.out BOSE_QUAD NORMAL)                    # a general system (meaning the system matrix is any nxn matrix) coupled with a quadratic boson environment

```
This configuration file will precisely tell the `cmake` program about the detailed recipes for building `MOSCAL2.0`. 

Finally, we can build the `MOSCAL2.0` easily 
```bash
# please create CMakeLists.txt in /path/to/moscal2.0/code 
mkdir build
cd build
cmake ..
make
```
If everything runs correctly, you shall see the `cmake` program handles the building process that many greens lines, (and some pinks for error). 
If you didn't see any red alerts and error messages, you can go out and check that you get new files `bose_linear.out`, `bose_linear_2.out`, `bose_quad.out` and `bose_quad_2.out` in your `build` directory. 
These, are indeed the executables you need to run the open quantum simulation examples. On how to use these binaries, see the python examples in `/path/to/moscal2.0/test`. All the above, concludes the building process for the 

### Extra efficiency from Facebook's folly library.
!!! bug
    Please stop reading for now! The `folly` part now has some bug. We cannot use the full version of `MOSCAL2.0` yet!

Now if your feel really geeky and greedy, that you feel an urge to speed up your calculations even more, you keep reading from here.
In fact, not all the components in `MOSCAL2.0` are parallelized if you follow the minimalistic build in the last section. Particularly, the filtering algorithm will be serial.
For geeks, this is because the parallel filtering algorithm of `MOSCAL2.0` depends on a package called `folly`. And if 

TLDR; you will have to install the `folly` library to unleash the full power of `MOSCAL2.0`. Luckily, if you have `homebrew`, this will be quite easy.
```bash
brew install folly
# after properly install the folly library, go on rebuild the `MOSCAL2.0` program.
# cmake is smart enough to see if folly is installed.
cd /path/to/moscal2.0/code/build
cmake ..
make
```

If `folly` is correctly installed, you shall have a fully parallelized distribution of `MOSCAL2.0`.

### Build macros: control what module you are building
Following the minimal build example, you obtain four binary files:

- `bose_linear_2.out`: This binary is for the open quantum dynamics of a 2*2 quantum system interacting with a bosonic bath.
- `bose_quadratic_2.out`: This binary is for the open quantum dynamics of a 2*2 quantum system interacting with a bosonic bath.
- `bose_linear.out`: This binary is for the open quantum dynamics of an arbitrary N*N quantum system interacting with a bosonic bath. 
- `bose_quadratic.out`: This binary is for the open quantum dynamics of an arbitrary N*N quantum system interacting with a bosonic bath. 

What if you want to study some other system bath interactions? For example, a fermionic system? 

To build other `MOSCAL2.0` modules, a basic understanding of **macros** becomes essential. In essence, macros serve as a means of communication with the compiler. In the context of `MOSCAL2.0`, **macros** are crucial for specifying the desired program configurations. For instance, if you aim to study a linear system-bath interaction involving a bosonic bath, you'd communicate this intent to the compiler using the `BOSE_LINEAR` macro. Additionally, when dealing with smaller systems, like a 2x2 setup, employing the `NSYS=2` macro informs the compiler to generate an optimized program leveraging stack allocation. 

!!! warning
    Stack allocation is beneficial for small matrices, typically those with dimensions up to 4x4. It optimizes memory management and improves performance. **However**, for larger systems exceeding dimension of 4, it's advisable to avoid specifying the `NSYS` macro to prevent potential memory issues associated with stack limitations.

In practical application, the `CMakeLists.txt` file provides a convenient `cmake` function called `build_population` for obtaining the desired binaries. You can utilize this function by specifying the necessary macros. The format is as follows:
```cmake
build_population(your_binary_name MACRO1 MACRO2 ...)
```
For instance, suppose you aim to compile a 4x4 system coupled with a linear fermionic system-bath coupling program. Simply include this line in your `CMakeLists.txt`
```cmake
build_population(fermi_linear_4.out NSYS=4 FERMI_LINEAR NORMAL)
```

For your references, here is a table of macros used by `MOSCAL2.0`

| Macro             | Description                                                               |
| :---------------: | :-------------------------------------------------------------------------|
| `BOSE_LINEAR`     | Specifies a linear system-bath interaction with bosons.                    |
| `BOSE_QUAD`       | Specifies a quadratic system-bath interaction with bosons.                 |
| `FERMI_LINEAR`    | Specifies a linear system-bath interaction with fermions.                  |
| `FERMI_QUAD`      | Specifies a quadratic system-bath interaction with fermions.               |
| `SPARSE`          | Enforces the use of sparse matrices, ideal for large, less dense systems.  |
| `NORMAL`          | Controls an unspecified filtering algorithm (default setting).             |
| `NSYS=4`          | Informs the compiler to optimize for a system of size 4, <br> potentially improving computations for this specific size.                    |



The table above showcases the most commonly used macros within `MOSCAL2.0`. While there are additional macros that unlock further modules in the `MOSCAL2.0` package, they are beyond the scope of this tutorial. For more comprehensive information, please refer to the [official documentation](https://git.lug.ustc.edu.cn/czh123/moscal2.0).


## Build the `c++` library on the Westlake University HPC. 
TODO: This section is under construction.


## Install the official `python` module
!!! info "Note"
    We regret to inform you that the official `python` module for `MOSCAL2.0` does not provide a `setup.py` for installation.
    
    While the installation of the python module is not mandatory, it significantly streamlines usage. Although there's the option to manually copy the module's source code to a desired location, opting for installation greatly simplifies the process and ensures ease of use.
    We recommend considering installation for a more seamless experience in utilizing the python module functionalities within `MOSCAL2.0`.

First, please locate the directory of the python module `deom`. You will find it in `/path/to/moscal2.0/test/deom`.

Second, please choose your preferred location for this python module. Feel free to choose any location on your computer. 
Then copy the `deom` directory here.
```bash
mkdir deom # choose any location
cp -r /path/to/moscal2.0/test/deom/ deom
```

Third, create a `setup.py` file in this directory
```python
# setup.py

 import os
 from setuptools import setup, find_packages

 def read(fname):
     return open(os.path.join(os.path.dirname(__file__), fname)).read()

 # Read the requirements from requirements.txt file
 with open('requirements.txt') as f:
     requirements = f.read().splitlines()

 setup(
     name="deom",
     description="Python interface to the c++ open quantum system program: DEOM",
     author="czh",
     long_description="Python interface to the c++ open quantum system program: DEOM",
     install_requires=requirements,
 )
```
and a `requirements.txt` file in this directory
```txt
cvxopt==1.3.2
numpy==1.22.1
pandas==1.5.1
scipy==1.7.3
sympy==1.11.1
```

!!! warning "Seggestions"
    If you don't already use `conda` to manage python environments, consider give it a try:
    ```bash
    conda create -n deom python

    # when every you want to use this environment 
    conda acitvate deom
    ```

Finally, feel free to use this one liner to install the package
```bash
pip install -e . 
```