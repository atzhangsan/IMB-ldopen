glibc2.17适用于centos7.04（先设置断定b dl_open_worker，然后display/8i $pc ,然后输入ni，和si 对照源码，和c原件中的行号对比）
=========================
```
dl_open_worker (void *a)
{
  struct dl_open_args *args = a;
  const char *file = args->file;
  int mode = args->mode;
  struct link_map *call_map = NULL;

  /* Check whether _dl_open() has been called from a valid DSO.  */
  if (__check_caller (args->caller_dl_open,
		      allow_libc|allow_libdl|allow_ldso) != 0)
    _dl_signal_error (0, "dlopen", NULL, N_("invalid caller"));

  /* Determine the caller's map if necessary.  This is needed in case
     we have a DST, when we don't know the namespace ID we have to put
     the new object in, or when the file name has no path in which
     case we need to look along the RUNPATH/RPATH of the caller.  */
  const char *dst = strchr (file, '$');
  if (dst != NULL || args->nsid == __LM_ID_CALLER
      || strchr (file, '/') == NULL)
     {
      const void *caller_dlopen = args->caller_dlopen;

#ifdef SHARED
      /* We have to find out from which object the caller is calling.
	 By default we assume this is the main application.  */
      call_map = GL(dl_ns)[LM_ID_BASE]._ns_loaded;*/=_dl_ns[0]._ns_loaded的值是._ns_loaded = &_dl_main_map
      */(gdb) p call_map  $2 = (struct link_map *) 0x7ffff7ffe150  可以看出就是这个 0x7ffff7ffe150，这个test的link_map 结构的内存地址。

#endif

      struct link_map *l;
      for (Lmid_t ns = 0; ns < GL(dl_nns); ++ns)
	for (l = GL(dl_ns)[ns]._ns_loaded; l != NULL; l = l->l_next)
	  if (caller_dlopen >= (const void *) l->l_map_start
	      && caller_dlopen < (const void *) l->l_map_end
	      && (l->l_contiguous
		  || _dl_addr_inside_object (l, (ElfW(Addr)) caller_dlopen)))
	    {
	      assert (ns == l->l_ns);
	      call_map = l;
	      goto found_caller;
	    }

    found_caller:
      if (args->nsid == __LM_ID_CALLER)
	{
#ifndef SHARED
	  /* In statically linked apps there might be no loaded object.  */
	  if (call_map == NULL)
	    args->nsid = LM_ID_BASE;
	  else
#endif
	    args->nsid = call_map->l_ns;
	}
    }
   ```
 看_dl_ns[0]._ns_loaded定义
   ```
    /* Namespace information.  */
struct link_namespaces _dl_ns[DL_NNS] =
  {
    [LM_ID_BASE] =
      {
	,
	._ns_nloaded = 1,
	._ns_main_searchlist = &_dl_main_map.l_searchlist,
      }
  };
  ```
  进入函数时候参数 l_map_object (loader=loader@entry=0x7ffff7ffe150, name=name@entry=0x400c20 "./libadd_c.so", type=type@entry=2, 
    trace_mode=trace_mode@entry=0, mode=mode@entry=-1879048191, nsid=0) at dl-load.c:2094
 ```
 ```
 共享库简介
- Introduction of Shared Library
不同的名字 Names
一个共享库常常有多个名字，比如

lrwxrwxrwx  1 hong hong    43 Apr 19 23:31 libadd.so -> /home/hong/workspace/playground/libadd.so.1*
lrwxrwxrwx  1 hong hong    47 Apr 19 23:31 libadd.so.1 -> /home/hong/workspace/playground/libadd.so.1.0.1*
-rwxrwxr-x  1 hong hong  8559 Apr 16 21:34 libadd.so.1.0.1*
像libadd.so.1这样的叫做soname，通常是由“lib”，紧接着库的名字，紧接着“.so”，然后跟着一个版本号。版本号通常是递增的。

像libadd.so.1.0.1这样的叫做real name，通常是soname之后再加一个“小版本号（minor number）”和一个“发布版本号（release number）”。也可以不加“发布版本号”。

像libadd.so这样的叫做linker name。这通常是给链接器（linker）用的。比如，运行下面的命令，linker就会去找叫libadd.so或者 libadd.a的库。

g++ main.cpp -ladd
通常real name对应的是真正的库文件，而soname和linker name对应的只是个符号链接（symbolic link）而以。soname对应的 符号链接可以通过ldconfig来创建。只要把共享库放在某个ldconfig知道的目录下，比如/usr/local/lib，然后再运行

ldconfig
即可。

linker name对应的符号链接不会被ldconfig创建，需要自己手工创建一下。

使用共享库
动态链接的可执行程序在运行时会先加载一个dynamic loader，一般叫ld-linux.so*，然后这个loader再去加载这个可执行文件用到的 其它共享库。dynamic loader的具体名字可以在可执行文件里找到，

>readelf `which cat` -l
...
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  ...
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  ...
从上面的输出里可以看到，在我的环境里它叫/lib64/ld-linux-x86-64.so.2。
ld-linux-x86-64.so.2是如何找到一个可执行文件需要的其它共享库的呢？在它的man page（man ld-linux或者man ld-so）里大概 是这么说的，

先看rpath指定的目录里有没有要找的共享库 
再看LD_LIBRARY_PATH指定的目录 
再看/etc/ld.so.cache文件里缓存的共享库信息里有没有 
如果还没找到，最后去默认的系统路径，/lib和/usr/lib，里去搜索，

这里，暂时先不去关心rpath和LD_LIBRARY_PATH。缓存文件/etc/ld.so.cache是被ldconfig程序管理的。系统配置文件/etc/ld.so.conf 里记录着共享库的搜索目录，比如/usr/local/lib等。一个可执行文件里只记录着共享库的soname，并没有完整的路径，比如

>readelf a.out -d

Dynamic section at offset 0xe18 contains 25 entries:
  Tag        Type                         Name/Value
0x0000000000000001 (NEEDED)             Shared library: [libadd.so.1]
0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
如果每次都要去/usr/local/lib、/lib、/usr/lib等目录里搜索会影响性能，所以把共享库的soname和完整的路径缓存在文件 /etc/ld.so.cache里以提高查找性能。通过ldconfig来维护缓存的有效性。当新加、删除共享库或者共享库搜索目录变化时，记得运行 ldconfig来更新缓存信息。一些ldconfig的简单用法，

# 打印缓存信息
> ldconfig -p | head -3
916 libs found in cache `/etc/ld.so.cache'
	libzephyr.so.4 (libc6) => /usr/lib/libzephyr.so.4
	libzbar.so.0 (libc6) => /usr/lib/libzbar.so.0
# 不更新缓存，更新指定目录下的共享库的符号链接
> ldconfig -n /opt/dummy/lib
# 建立符号链接，更新缓存
> ldconfig
通常/etc/ld.so.conf指定的目录中会包含/usr/local/lib目录，一般来说用户安装的共享库会放在这个目录里。/lib通常用来 放置启动时需要的库，而/usr/lib通常用来放置随Linux发行版自带的库。除了/usr/local/lib，也可以在/etc/ld.so.conf指定 新的目录来放置共享库。

创建共享库
gcc -fPIC -g -c -Wall a.c
gcc -fPIC -g -c -Wall b.c
gcc -shared -Wl,-soname,libmystuff.so.1 \
    -o libmystuff.so.1.0.1 a.o b.o -lc
上述命令的一些解释：

-fPIC选项用来生成位置无关代码（position-independent code）。有两个类似的选项，-fPIC和-fpic。两者的区别是-fpic在 生成位置无关代码时会检查global offset table (GOT)有没有超过相关平台的限制，如果超过则报错；而-fPIC会避免这些限制。据说， -fpic生成的目标文件相比-fPIC要小一些。反正用-fPIC总是没错的。两个选项的具体描述见man gcc。

-Wl,-soname,libmystuff.so.1，-Wl指明后面跟着的选项-soname是传给linker的。注意是用逗号来分隔，而不是空格；如果有空格， 记得要转义。-soname选项后跟的共享库的soname。如果使用了这个选项，那么在创建共享库时，给定的soname会写到库文件中（ELF的 DT_SONAME字段）。如果一个可执行文件链接到设置了DT_SONAME字段共享库，会（详见man ld），

When an executable is linked with a shared object which has a DT_SONAME field, then when the executable is run the dynamic linker（注：就是ld-linux-x86-64.so.2） will attempt to load the shared object specified by the DT_SONAME field rather than the using the file name given to the linker（注：应该是指通过-lmystuff传给 linker的文件名，也就是libmystuff.so）.

-o libmystuff.so.1.0.1指定了共享库的real name，包含详细的版本信息。

-lc，如果这个共享库依赖别的库，那么用-l选项指定。-lxxx应该放在*.o的后面，否则gcc会报undefined reference to的错误消息。 这是因为如果链接器遇到-lxxx时，如果发现那时没有任何.o文件用到这个库的符号（symbol），就不会把这个库链接进来。即使，-l 选项的后面有.o文件用到了那个库，链接器也不会再去尝试链接这个库。详见man ld。

另外，为了方便调试，最好不要剥离库的符号，也不要在编译时指定-fomit-frame-pointer选项。

安装和使用共享库
下面的例子都是安装到/usr/local/lib目录下，实际上任何/etc/ld.so.conf指定的目录以及默认目录/lib和/usr/lib都是可以的。

如果要安装一个新的共享库，那么把库文件拷贝到/usr/local/lib，然后运行ldconfig更新缓存并建立符号链接。

>sudo cp libadd.so.1.0.1 /usr/local/lib/
>sudo ldconfig
# 确认一下
>ldconfig -p | grep libadd
	libadd.so.1 (libc6,x86-64) => /usr/local/lib/libadd.so.1
>ll /usr/local/lib/
lrwxrwxrwx  1 root root    15 Apr 20 21:59 libadd.so.1 -> libadd.so.1.0.1*
如果要更新共享库，且共享库只是小版本升级。那么只需要更新一下符号链接就可以，不需要更新缓存。因为缓存只是记录着soname和它的全路径信息， 并没有任何小版本信息。

>sudo cp libadd.so.1.0.2 /usr/local/lib/
# -n只更新符号链接；可以指定目录
>sudo ldconfig -n /usr/local/lib/
>ll /usr/local/lib/
lrwxrwxrwx  1 root root    15 Apr 20 21:57 libadd.so.1 -> libadd.so.1.0.2*
 -rwxr-xr-x  1 root root  8559 Apr 20 01:19 libadd.so.1.0.1*
 -rwxr-xr-x  1 root root  8559 Apr 20 21:56 libadd.so.1.0.2*
如果库是大版本升级，那么符号链接和缓存都需要更新。

>g++ -shared -Wl,-soname,libadd.so.3 -o libadd.so.3.0.1 add.o 
>sudo cp libadd.so.3.0.1 /usr/local/lib/
>sudo ldconfig
>ldconfig -p | grep libadd
	libadd.so.3 (libc6,x86-64) => /usr/local/lib/libadd.so.3
	libadd.so.1 (libc6,x86-64) => /usr/local/lib/libadd.so.1
ldconfig是如何建立符号链接的呢？前面讲到，通过-soname选项可以把库的soname存到库文件里，ldconfig会读取这个字符串，以它 作为符号链接的名字。所以创建共享库时，记得指定-soname选项并赋予正确的名字。

最后，因为ldconfig只会创建从real name到soname的如何链接，为了让链接器可以找到共享库，还需要建立linker name对应的 符号链接。

>sudo ln -s /usr/local/lib/libadd.so.1 /usr/local/lib/libadd.so
使用共享库有两种使用场景：编译/链接时和运行可执行文件时。

编译/链接时（linker怎么找到共享库）
如果共享库已经放在了/usr/local/lib等“标准目录”下，那么直接通过-l使用即可，

>g++ main.cpp -ladd
如果由于某些原因共享库还没安装到“标准目录”下，比如库还处于开发中，那么可以通过-L指定库的搜索目录，

>g++ -L/home/hong/workspace/playground main.cpp -ladd
man ld中详细地描述了链接器是怎么搜索共享库的。

运行可执行文件时（dynamic loader怎么找到共享库）
同样地，如果共享库已经放在了/usr/local/lib等“标准目录”下，那么无需额外的工作。

如果由于某些原因共享库还没安装到“标准目录”下，比如库还处于开发中，那么需要告诉dynamic loader（ld-linux-x86-64.so.2） 去哪里找这个库。否则会有下面的错误

>./a.out
./a.out: error while loading shared libraries: libadd.so.1: cannot open shared object file: No such file or directory
使用LD_LIBRARY_PATH，可以在.bashrc等文件里全局地设置LD_LIBRARY_PATH变量或者在某个shell里单独地设置，

>LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH ./a.out
LD_LIBRARY_PATH最好只作为一个临时的手段，这篇文章里讲了为什么使用LD_LIBRARY_PATH是不好的。

使用rpath，rpath会把库的搜索目录编译到可执行文件里。

>g++ main.cpp -ladd -L/home/hong/workspace/playground -Wl,-rpath,/home/hong/workspace/playground
>readelf a.out -d
Dynamic section at offset 0xe08 contains 26 entries:
  Tag        Type                         Name/Value
0x0000000000000001 (NEEDED)             Shared library: [libadd.so.1]
0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
0x000000000000000f (RPATH)              Library rpath: [/home/hong/workspace/playground]
0x000000000000000c (INIT)               0x400568
版本控制 Versioning
系统常常会存在着不同版本的共享库，不同的可执行文件可能使用不同版本的共享库，那么是如何做到不发生冲突的呢？

从上面的readelf的输出可以看到，可执行文件里存在它依赖的共享库的soname，也就是共享库的“大版本信息”。soname 只是一个符号链接，它指向某个具体版本的库文件，比如libadd.so.1.0.1。soname指向的库文件可能会更新。为了使得 库文件的更新不影响可执行文件的运行，这就要求库的更新不会产生任何兼容性问题。如果存在兼容性问题，那么在生产库 的时候需要增加soname里的版本号。

这篇文章里讲了会引起兼容性问题的典型情形，比如改变接口行为，删除接口等。

其它
ldd
使用ldd可以查看一个可执行文件需要的共享库，比如

>ldd /bin/cat
	linux-vdso.so.1 =>  (0x00007fff47b63000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2644e8e000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f2645269000)
这篇文章和man ldd都提到了ldd可能会有安全问题。不要对不信任的程序执行ldd命令。可以用下面的命令代替ldd，

>objdump -p a.out | grep NEEDED
  NEEDED               libadd.so.1
  NEEDED               libc.so.6
TODOs
使用/etc/ld.so.preload覆盖共享库的部分接口
LD_PRELOAD
LD_DEBUG

```
