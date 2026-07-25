
# C++编译器的安装位置
## Windows OS
**Visual Studio** — X:\Microsoft Visual Studio 9.0\VC\crt\src; X:\Program files (x86)\Microsoft Visual Studio 14.0\VC\include\; X:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\VC\Tools\MSVC\14.16.27023\crt\src  
Note,  
1) You even don't have to install Visual Studio to explore Microsoft implementation. Online compilers can help: [rextester.com/NRP17506](https://rextester.com/NRP17506 "rextester.com/NRP17506")  
2) In Visual Studio if you interesting in concrete(specific) STL-element implementation (for example, any function), right click on its mention in your code and chose "Go to Definition" in context menu. (Or place cursor on this mention and push "F12")

## Linux/Unix
**gcc** — /usr/include/c++/; /usr/lib/gcc/CTARGET/CTARGET/VERSION/include/g++-v4/  
Note, `locate iostream` is a good solution to find the installation location.

# 参考
1. https://www.cnblogs.com/klchang/p/13207884.html
