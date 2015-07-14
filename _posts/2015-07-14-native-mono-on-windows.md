---
layout: post
title: Using Mono to avoid the .net framework dependency
published: false
---

## Using Mono to avoid the .net framework dependency

I spent the last few days trying to get an app running on mono without the dependency on the .NET framework. I found [this blog](http://blog.shilbert.com/2012/06/using-mono-to-avoid-depending-on-net-framework/) by Scott Hilbert.  It essentially boils down to:

1. Install Mono
2. Install Cygwin
  1. Install the packages gcc-mingw, mingw-zlib1, mingw-zlib-devel, and pkg-config
3. Use [this script](https://gist.github.com/shilrobot/3005371) in some compacity.

Well, it's never that simple is it?

Trying it out on a HelloWorld.exe I get this.

```bash
OS is: Windows
WARNING:
  Check that the machine.config file you are bundling
  doesn't contain sensitive information specific to this machine.
Sources: 1 Auto-dependencies: True
   embedding: C:\projects\HelloWorld\HelloWorld\bin\Debug\HelloWorld.exe
   embedding: C:\progra~2\Mono\lib\mono\4.5\mscorlib.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\FSharp.Core.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\System.Core.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\System.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\Mono.Security.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\System.Configuration.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\System.Xml.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\System.Security.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\Mono.Posix.dll
   embedding: C:\progra~2\Mono\lib\mono\4.5\System.Numerics.dll
Machine config from: C:\Program Files (x86)\Mono\etc\mono\4.5\machine.config
Compiling:
i686-pc-mingw32-as -o temp.o temp.s
i686-pc-mingw32-gcc -U _WIN32 -g -o Output.exe -Wall temp.c `pkg-config --cflags --libs mono-2`  temp.o
i686-pc-mingw32-gcc: error: `pkg-config: No such file or directory
i686-pc-mingw32-gcc: error: unrecognized command line option ‘--cflags’
i686-pc-mingw32-gcc: error: unrecognized command line option ‘--libs’
i686-pc-mingw32-gcc: error: mono-2`: No such file or directory
ERROR: [Fail]
```
