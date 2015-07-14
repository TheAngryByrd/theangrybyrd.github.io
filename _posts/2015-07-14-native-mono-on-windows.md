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
3. Use mkbundle
4. Copy mono-2.0.dll

Use [this script](https://gist.github.com/shilrobot/3005371) in some compacity.

Well, it's never that simple is it?

AA doesn't exist?  Well investigating a little bit, mingw provides an assembler, I just added tis export

```bash
export AS="i686-pc-mingw32-as"
```

Lets run it again!

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

FAILURE. pkg-config can't be found? Seems like I can find it.

```base
$ pkg-config --cflags --libs mono-2
-mms-bitfields -I/cygdrive/c/progra~2/Mono/lib/pkgconfig/../../include/mono-2.0 -L/cygdrive/c/progra~2/Mono/lib/pkgconfig/../../lib -lmono-2.0 -mms-bitfields -lws2_32 -lpsapi -lole32 -lwinmm -loleaut32 -ladvapi32 -lversion
```

I must be missing something here.  Hopefully someone can point it out.  However, if I decide to run the compile using mingw in a seperate step I actually get some output!

```bssh
i686-pc-mingw32-gcc -U _WIN32 -g -o Output.exe -Wall temp.c `pkg-config --cflags --libs mono-2`  temp.o
```

Lets run the exe!

... Blank?  BLANK?

Well [stackoverflow](https://stackoverflow.com/questions/7507944/mkbundle-produces-non-functional-exe?rq=1) has something to say about it. What is this -mwindows flag used for?  According to the [MinGW FAQ](http://www.mingw.org/wiki/FAQ) it's used to removed DOS from showing up.  Ahhh.  So you're telling me I have to remove some flags from a file to get console programs to work?  Fun day.  Hopefully someone can figure out a way around this.

But after removing that -mwindows flag, success!

So behold, the master script to do all the things!
bootstrapper.bat gets mono and cygwin and packages.
mkbundler.sh will be run by cygwin to do the mkbundle, compile, and copying of mono-2.0.dll
