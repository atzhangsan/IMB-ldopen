link_map 详细解析
link-map是动态连接器
内部使用的一个结构，通过它保持对已装载的库和库中符号的跟踪。实际上link-map是一个
链表，表中的每一项都有一个指向装载库的指针。就象动态连接器所做的，当需要去
查找符号的时候，我们能向前或向后遍历这个链表 ，通过访问链表上的每一个库去
发现我们要找的符号。
link-map由每个目标文件的GOT(全局偏移表)的第二个入
口(GOT[1])指向。对我们来说，从GOT[1]读取link-map结点地址，然后沿着link-map结点进
行搜索，直到发现我们想去查找的符号，这并没有什么困难。
```
在link.h中：
struct link_map
{
ElfW(Addr) l_addr; /* 共享对象被装载的基地址
char *l_name; /* 在里面发现目标的绝对文件名 */
ElfW(Dyn) *l_ld; /* 共享对象的Dynamic section(动态区域) */
struct link_map *l_next, *l_prev; /* 被装载对象的链 */
};
这个结构非常清晰，无须加以说明。但是不管怎样，在这里我们还是对其中的每一项
加以简短的说明。
l_addr: 共享对象被装载的基地址，这个值也能在/proc/<pid>/maps找到
l_name: 指向在string table中的库名的指针
l_ld: 指向共享对象的dynamic section(动态区域)的指针
l_next: 指向下一个link_map结点的指针
l_prev: 指向前一个link_map结点的指针
这个用link_map结构来解析符号的想法是简单的，我们遍历link_map链表，比较每个l_name
项，
直到我们的符号所在的库被发现。然后我们移到l_ld结构并搜索dynamic section(动态区域
)
直到DT_SYMTAB和DT_STRTAB被发现，最终我们能从DT_SYMTAB找到我们的符号。
这可能很慢，但对我们举例子来说应该很好。使用HASH table(哈希表，即散列表)

进行符号查找将是快速的也是首选的，但是那是留给读者做练习的;D。
```
aiaiaiaiaiaiia--
```
/* A dummy link map for the executable, used by dlopen to access the global
   scope.  We don't export any symbols ourselves, so this can be minimal.  */
static struct link_map _dl_main_map =
  {
    .l_name = (char *) "",// /* 在里面发现目标的绝对文件名 */
    .l_real = &_dl_main_map,
    .l_ns = LM_ID_BASE,
    .l_libname = &(struct libname_list) { .name = "", .dont_free = 1 },
    .l_searchlist =
      {
	.r_list = &(struct link_map *) { &_dl_main_map },
	.r_nlist = 1,
      },
    .l_symbolic_searchlist = { .r_list = &(struct link_map *) { NULL } },
    .l_type = lt_executable,
    .l_scope_mem = { &_dl_main_map.l_searchlist },
    .l_scope_max = (sizeof (_dl_main_map.l_scope_mem)
		    / sizeof (_dl_main_map.l_scope_mem[0])),
    .l_scope = _dl_main_map.l_scope_mem,
    .l_local_scope = { &_dl_main_map.l_searchlist },
    .l_used = 1,
    .l_flags_1 = DF_1_NODEFLIB,
    .l_tls_offset = NO_TLS_OFFSET,
    .l_serial = 1,
  };
  
