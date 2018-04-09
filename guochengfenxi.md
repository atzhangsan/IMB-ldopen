glibc2.17适用于centos7.04
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
  
