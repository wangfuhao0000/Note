## 起步

### 基本用法

一个最基本的CMakeLists.txt文件最少需要包含以下三行：

```cmake
cmake_minimum_required (VERSION 2.6)

# 设置项目的名称
project (Tutorial)

# 添加可执行文件
add_executable(Tutorial tutorial.cpp)
```

注意：cmake的语法支持大小、小写和大小写混合，上边的代码中我们使用的cmake语法是小写的。注意，只有系统指令是不区分大小写的，但是变量和字符串是区分大小写的。

创建一个tutorial.cxx文件，用来计算一个数字的平方根，内容如下:

```c++
// A simple program that computes the square root of a number
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
int main (int argc, char *argv[])
{
  if (argc < 2)
    {
    fprintf(stdout,"Usage: %s number\n",argv[0]);
    return 1;
    }
  double inputValue = atof(argv[1]);
  double outputValue = sqrt(inputValue);
  fprintf(stdout,"The square root of %g is %g\n",
          inputValue, outputValue);
  return 0;
}
```

这样就完成一个最简单的cmake程序。

### 构建程序

用cmake来编译这段代码，进入命令行执行**内部构建**命令（后边会讲外部构建）：

```shell
cmake .
```

这是输出一系列的log信息

```shell
-- The C compiler identification is AppleClang 9.0.0.9000039
-- The CXX compiler identification is AppleClang 9.0.0.9000039
-- Check for working C compiler: /Library/Developer/CommandLineTools/usr/bin/cc
-- Check for working C compiler: /Library/Developer/CommandLineTools/usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /Library/Developer/CommandLineTools/usr/bin/c++
-- Check for working CXX compiler: /Library/Developer/CommandLineTools/usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/saka/Desktop/Tutorial/Step1
```

同时生成了三个文件`CMakeCache.txt`、`Makefile`、`cmake_install.cmake`和一个文件夹`CmakeFiles`,然后执行

```shell
make 
```

即可生成可执行程序`Tutorial`。在ubuntu或者centos上可能会提示找不到`math.h`文件,这时候我们需要在cmakeLists.txt文件中最后添加

```cmake
target_link_libraries(Tutorial apue.a)
```

然后重新编译即可。需要删除刚才生成的额外的文件。

### 添加版本号

下面讲解如何为程序添加版本号和带有使用版本号的头文件。

`set(KEY VALUE)`接受两个参数，用来声明变量。在camke语法中使用`KEY`并不能直接取到`VALUE`，**必须使用`${KEY}`这种写法来取到`VALUE`**。

```makefile
cmake_minimum_required (VERSION 2.6)
project (Tutorial)
# The version number.
set (Tutorial_VERSION_MAJOR 1)
set (Tutorial_VERSION_MINOR 0)
 
# configure a header file to pass some of the CMake settings
# to the source code
configure_file (
  "${PROJECT_SOURCE_DIR}/TutorialConfig.h.in"
  "${PROJECT_BINARY_DIR}/TutorialConfig.h"
  )
 
# add the binary tree to the search path for include files
# so that we will find TutorialConfig.h
include_directories("${PROJECT_BINARY_DIR}")
 
# add the executable
add_executable(Tutorial tutorial.cxx)
```

配置文件将会被写入到可执行文件目录下，所以我们的项目必须包含这个文件夹来使用这些配置头文件。我们需要在工程目录下新建一个`TutorialConfig.h.in`,内容如下：

```
// the configured options and settings for Tutorial
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

上面的代码中的`@Tutorial_VERSION_MAJOR@`和`@Tutorial_VERSION_MINOR@`将会被替换为`CmakeLists.txt`中的1和0。 然后修改`Tutorial.cxx`文件如下，用来在不输入额外参数的情况下输出版本信息：

```
// A simple program that computes the square root of a number
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include "TutorialConfig.h"
 
int main (int argc, char *argv[])
{
  if (argc < 2)
    {
    fprintf(stdout,"%s Version %d.%d\n",
            argv[0],
            Tutorial_VERSION_MAJOR,
            Tutorial_VERSION_MINOR);
    fprintf(stdout,"Usage: %s number\n",argv[0]);
    return 1;
    }
  double inputValue = atof(argv[1]);
  double outputValue = sqrt(inputValue);
  fprintf(stdout,"The square root of %g is %g\n",
          inputValue, outputValue);
  return 0;
}
复制代码
```

然后执行

```
cmake .
make
./Tutorial
复制代码
```

即可看到输出内容:

![img](https://user-gold-cdn.xitu.io/2018/1/29/161425f0d0feccdb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 添加库

### 构建自己的库

这个库将包含我们自己计算一个数字的平方根的计算方法。生成的程序可以使用这个库，而不是由编译器提供的标准平方根函数(`math.h`)。

在本教程中，我们将把库放到一个名为`mathfunction`的子目录中,在工程目录下新建`mathfunction`文件夹。这个文件夹中新建CMakeLists.txt文件，包含以下一行代码:

```cmake
add_library(MathFunctions mysqrt.cxx)
```

然后在这个文件夹中创建源文件`mysqrt.cxx`，它只有一个名为`mysqrt`的函数，与编译器的sqrt函数提供了类似的功能。

为了利用新库，我们在工程根目录下的`CMakeLists.txt`中添加`add_subdirectory()`来构建我们自己的库。我们还添加了另一个include目录,以便`MathFunctions / MathFunctions.h`可以为函数原型找到头文件,该文件代码如下：

```
double mysqrt(double x);
```

然后创建`mysqrt.cxx`文件，内容如下

```cpp
#include "MathFunctions.h"
#include <stdio.h>

// a hack square root calculation using simple operations
double mysqrt(double x)
{
  if (x <= 0) {
    return 0;
  }

  double result;
  double delta;
  result = x;

  // do ten iterations
  int i;
  for (i = 0; i < 10; ++i) {
    if (result <= 0) {
      result = 0.1;
    }
    delta = x - (result * result);
    result = result + 0.5 * delta / result;
    fprintf(stdout, "Computing sqrt of %g to be %g\n", x, result);
  }
  return result;
}
```

最后一个更改是将新库添加到可执行文件。根目录下CMakeLists.txt的最后添加以下代码

```
include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
add_subdirectory (MathFunctions) 
 
# add the executable
add_executable (Tutorial tutorial.cxx)
target_link_libraries (Tutorial MathFunctions)
```

现在文件目录如下

```
.
├── CMakeLists.txt
├── MathFunctions
│   ├── CMakeLists.txt
│   ├── MathFunctions.h
│   └── mysqrt.cxx
├── TutorialConfig.h.in
└── tutorial.cxx
复制代码
```

### 构建可选选项

`MathFunctions`是我们自己构建的库，有时候我们需要控制这个库是否应该使用，那么可以为使用这个库添加一个开关，在构建大型项目时非常有用。

在项目根目录下的`CMakeLists.txt`文件中添加如下代码：

```
# should we use our own math functions?
option (USE_MYMATH 
        "Use tutorial provided math implementation" ON)
```

假如你使用的是CMake GUI，`USE_MYMATH`默认值是用户可以根据需要更改。该设置将存储在缓存中，以便用户在每次运行CMake时生成默认配置。然后我们就可以选择性的构建和使用mathfunction库。修改根目录下CMakeLists.txt:

```
# add the MathFunctions library?
#
if (USE_MYMATH)
  include_directories ("${PROJECT_SOURCE_DIR}/MathFunctions")
  add_subdirectory (MathFunctions)
  set (EXTRA_LIBS ${EXTRA_LIBS} MathFunctions)
endif (USE_MYMATH)
 
# add the executable
add_executable (Tutorial tutorial.cxx)
target_link_libraries (Tutorial  ${EXTRA_LIBS})
```

这将使用`USE_MYMATH`的设置来确定是否应该编译和使用mathfunction库。注意，使用一个变量(在本例中是EXTRA_LIBS)来设置可选的库，然后将它们链接到可执行文件中。这是一种常见的方法，用于保持较大的项目具有许多可选组件。 首先在Configure.h.in文件中添加以下内容：

```
#cmakedefine USE_MYMATH
```

然后我们就可以使用`USE_MYMATH`这个变量了，最后修改`Tutorial.cxx`源代码如下：

```
// A simple program that computes the square root of a number
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include "TutorialConfig.h"
#ifdef USE_MYMATH
#include "MathFunctions.h"
#endif
 
int main (int argc, char *argv[])
{
  if (argc < 2)
    {
    fprintf(stdout,"%s Version %d.%d\n", argv[0],
            Tutorial_VERSION_MAJOR,
            Tutorial_VERSION_MINOR);
    fprintf(stdout,"Usage: %s number\n",argv[0]);
    return 1;
    }
 
  double inputValue = atof(argv[1]);
 
#ifdef USE_MYMATH
  double outputValue = mysqrt(inputValue);
#else
  double outputValue = sqrt(inputValue);
#endif
 
  fprintf(stdout,"The square root of %g is %g\n",
          inputValue, outputValue);
  return 0;
}
```

我们编译执行看以下结果

1. 使用自定义的库(USE_MYMATH=ON)

```
$ ~/Desktop/Tutorial/Step2/ ./Tutorial 4
Computing sqrt of 4 to be 2.5
Computing sqrt of 4 to be 2.05
Computing sqrt of 4 to be 2.00061
Computing sqrt of 4 to be 2
Computing sqrt of 4 to be 2
Computing sqrt of 4 to be 2
Computing sqrt of 4 to be 2
Computing sqrt of 4 to be 2
Computing sqrt of 4 to be 2
Computing sqrt of 4 to be 2
The square root of 4 is 2
```

1. 不使用自定义的库(USE_MYMATH=OFF)

```
$ ~/Desktop/Tutorial/Step2/ ./Tutorial 4
The square root of 4 is 2
```

可以看到，这个开关达到了我们需要的效果。