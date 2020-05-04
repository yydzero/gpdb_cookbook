
# troubleshooting cookbook

## 显示详细的错误信息

有时候错误信息不够详尽，譬如看不到出错的文件和行号。 这个可以通过 psql中设置
详尽级别可以获得：

    yydzero=# \set VERBOSITY verbose
    yydzero=# create table t1 (id int);
    ERROR:  42P07: relation "t1" already exists
    LOCATION:  heap_create_with_catalog, heap.c:1122

使用 psql 内建的命令 \errverbose 可以获得上次命令的错误信息

从 PostgreSQL 9.6 开始支持这一功能。

## 控制 psql 页面设置

    integrator=# \pset pager off
    Pager usage is off.
    integrator=# \pset format unaligned

## gdb 故障进程，获得callstack

    If you wanted to show a stack trace, you could attach gdb to PID from SELECT
    pg_backend_pid(), "b"reak on errdetail, run the query, and then "bt" when it
    fails.

一个例子：

    gdb -p 27930
    GNU gdb (GDB) Red Hat Enterprise Linux 8.0.1-30.amzn2.0.3
    ...
    Attaching to process 27930
    Reading symbols from /usr/local/pgsql/bin/postgres...done.
    ...
    (gdb) b errdetail
    Breakpoint 1 at 0x82b210: file elog.c, line 872.
    (gdb) cont
    Continuing.
    Breakpoint 1, errdetail (fmt=fmt(at)entry=0x9d9958 "Failed on request of size %zu in memory context \"%s\".") at elog.c:872
    872     {
    (gdb) bt
    #0  errdetail (fmt=fmt(at)entry=0x9d9958 "Failed on request of size %zu in memory context \"%s\".") at elog.c:872
    #1  0x000000000084e320 in MemoryContextAlloc (context=0x1111600, size=size(at)entry=32800) at mcxt.c:794
    #2  0x000000000060ce7a in dense_alloc (size=384, size(at)entry=381, hashtable=<optimized out>, hashtable=<optimized out>)
         at nodeHash.c:2696
    #3  0x000000000060d788 in ExecHashTableInsert (hashtable=hashtable(at)entry=0x10ead08, slot=<optimized out>, hashvalue=194758122)
         at nodeHash.c:1614
    #4  0x0000000000610c6f in ExecHashJoinNewBatch (hjstate=0x10806b0) at nodeHashjoin.c:1051
    #5  ExecHashJoinImpl (parallel=false, pstate=0x10806b0) at nodeHashjoin.c:539
    #6  ExecHashJoin (pstate=0x10806b0) at nodeHashjoin.c:565

## vm.overcommit\_memory and vm.overcommit\_ratio

有些内存相关的错误和这两个设置有关。 Greenplum建议设置为2.

cat /proc/meminfo | grep Commit


    Tomas Vondra just added a good idea that explains why I get the out of
    memory with still having so much cache available:
    
    # sysctl vm.overcommit_memory
    vm.overcommit_memory = 2
    # sysctl vm.overcommit_ratio
    vm.overcommit_ratio = 50
    
    as he predicted.
    
    # cat /proc/meminfo |grep Commit
    CommitLimit:     3955192 kB
    Committed_AS:    2937352 kB

## 如何在特定的memory context 上设置断点

    If you can use gdb at all, it's not that hard to break on allocations
    into a specific context; I've done it many times.  The strategy is
    basically
    
    1. Let query run long enough for memory usage to start increasing,
    then attach to backend with gdb.
    
    2. Set breakpoint at, probably, AllocSetAlloc.  (In some cases,
    reallocs could be the problem, but I doubt it here.)  Then "c".
    
    3. When it stops, "p *context" and see if this is the context
    you're looking for.  In this case, since we want to know about
    allocations into ExecutorState and we know there's only one
    active one, you just have to look at the context name.  In general
    you might have to look at the backtrace.  Anyway, if it isn't the
    one you want, just "c" until you get to an allocation into the
    one you do want.
    
    4. Once you have found out the address of the context you care
    about, make the breakpoint conditional on the context argument
    being that one.  It might look like this:
    
    Breakpoint 1, AllocSetAlloc (context=0x1483be0, size=480) at aset.c:715
    715     {
    (gdb) p *context
    $1 = {type = T_AllocSetContext, isReset = false, allowInCritSection = false,
      methods = 0xa33f40, parent = 0x0, firstchild = 0x1537f30, prevchild = 0x0,
      nextchild = 0x0, name = 0xa3483f "TopMemoryContext", ident = 0x0,
      reset_cbs = 0x0}
    (gdb) cond 1  context == 0x1483be0
    
    5. Now repeatedly "c", and check the stack trace each time, for a
    dozen or two times to get a feeling for where the allocations are
    being requested.
    
    In some cases you might be able to find the context address in a
    more efficient way than what I suggest in #3 --- for instance,
    you could instead set a breakpoint where the context is created
    and snag its address immediately, or you could dig around in
    backend data structures to find it.  But these ways generally
    require more familiarity with the code than just watching the
    requests go by.

## dump memory context with debugger

    call MemoryContextStats(TopMemoryContext)

## setup conditional break point

调试内存OOM的一个技巧：内存OOM发生错误时的栈可能不是造成内存大量占用的栈，
而仅仅是经常分配，譬如per tuple内存，的内存。这个时候设置断点大概率不能
找到实际问题的栈。

可以通过一个技巧来解决这个问题：获得调用 AllocSetAlloc 的栈中保存的调用者的
指令地址rip，然后设置条件断点。

    https://www.postgresql.org/message-id/flat/bc138e9f-c89e-9147-5395-61d51a757b3b@gusw.net

    (gdb) info frame
    Stack level 0, frame at 0x7ffcbf92fdd0:
      rip = 0x849030 in AllocSetAlloc (aset.c:718); saved rip = 0x84e7dd
      called by frame at 0x7ffcbf92fdf0
      source language c.
      Arglist at 0x7ffcbf92fdc0, args: context=0x29a6450, size=371
      Locals at 0x7ffcbf92fdc0, Previous frame's sp is 0x7ffcbf92fdd0
      Saved registers:
       rip at 0x7ffcbf92fdc8

    (gdb) b AllocSetAlloc if  (int)strcmp(context->name, "ExecutorState") == 0 && *(int *)$rsp != 0x84e7dd

    And there we go:

    // 这样获得如下的栈：

    Breakpoint 6, AllocSetAlloc (context=0x29a6450, size=8) at aset.c:718
    718     {
    (gdb) bt 8
    #0  AllocSetAlloc (context=0x29a6450, size=8) at aset.c:718
    #1  0x000000000084e8ad in palloc0 (size=size(at)entry=8) at mcxt.c:969
    #2  0x0000000000702b63 in makeBufFileCommon (nfiles=nfiles(at)entry=1) at buffile.c:119
    #3  0x0000000000702e4c in makeBufFile (firstfile=68225) at buffile.c:138
    #4  BufFileCreateTemp (interXact=interXact(at)entry=false) at buffile.c:201
    #5  0x000000000061060b in ExecHashJoinSaveTuple (tuple=0x2ba1018, hashvalue=<optimized out>, fileptr=0x6305b00) at nodeHashjoin.c:1220
    #6  0x000000000060d766 in ExecHashTableInsert (hashtable=hashtable(at)entry=0x2b50ad8, slot=<optimized out>, hashvalue=<optimized out>)
         at nodeHash.c:1663
    #7  0x0000000000610c8f in ExecHashJoinNewBatch (hjstate=0x29a6be0) at nodeHashjoin.c:1051
    (More stack frames follow...)

    (gdb) info frame
    Stack level 0, frame at 0x7ffcbf92fd90:
      rip = 0x849030 in AllocSetAlloc (aset.c:718); saved rip = 0x84e8ad
      called by frame at 0x7ffcbf92fdb0
      source language c.
      Arglist at 0x7ffcbf92fd80, args: context=0x29a6450, size=8
      Locals at 0x7ffcbf92fd80, Previous frame's sp is 0x7ffcbf92fd90
      Saved registers:
       rip at 0x7ffcbf92fd88
    (gdb) b AllocSetAlloc if  (int)strcmp(context->name, "ExecutorState") == 0 && *(int *)$rsp != 0x84e7dd && 0x84e8ad != *(int *)$rsp
    Note: breakpoint 6 also set at pc 0x849030.
    Breakpoint 7 at 0x849030: file aset.c, line 718.
    (gdb) delete 6
    
    Now if I continue I don't seem to be stopping any more.


## adjust code behavior inside GDB

下面例子通过一个变量，可以在gdb里面控制何时打开print

    > int _alloc_info = 0;
    > #ifdef HAVE_ALLOCINFO
    > #define AllocFreeInfo(_cxt, _chunk) \
    >         if(_alloc_info) \
    >             fprintf(stderr, "AllocFree: %s: %p, %zu\n", \
    >                 (_cxt)->header.name, (_chunk), (_chunk)->size)
    > #define AllocAllocInfo(_cxt, _chunk) \
    >         if(_alloc_info) \
    >             fprintf(stderr, "AllocAlloc: %s: %p, %zu\n", \
    >                 (_cxt)->header.name, (_chunk), (_chunk)->size)
    > #else
    > #define AllocFreeInfo(_cxt, _chunk)
    > #define AllocAllocInfo(_cxt, _chunk)
    > #endif
    >
    > so with this I do
    >
    > (gdb) b AllocSetAlloc
    > (gdb) cont
    > (gdb) set _alloc_info=1
    > (gdb) disable
    > (gdb) cont

## valgrind

    valgrind is a useful idea, given that Gunther is building his own
    postgres (so he could compile it with -DUSE_VALGRIND + --enable-cassert,
    which are needed to get valgrind to understand palloc allocations).
    I don't recall details right now, but it is possible to trigger
    a valgrind report intra-session similar to what you get by default
    at process exit.  You could wait till the memory has bloated a
    good deal and then ask for one of those reports that classify
    allocations by call chain (I think you want the memcheck tool for
    this, not the default valgrind tool).

## 编译激活特定调试信息

    ./configure CFLAGS=-DHJDEBUG=1

## lldb fork child process

如果单独启动master，而不启动segment，master 会处于不断重启 dtx recovery child
process 的过程中。那么如何知道为什么 dtx recovery 进程不能启动成功那？

    lldb -p <postgres main process id>
    (lldb) b fork
    (lldb) c

另一个终端：
    
    $ lldb -w -n postgres  // wait for postgres process to attach

等待 lldb attach 到新启动的 postgres 子进程时，查看 call stack trace：

    (lldb) bt
    * thread #1, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
      * frame #0: 0x00007fff5b68a5aa libsystem_kernel.dylib`__select + 10
        frame #1: 0x000000010ad74da8 postgres`pg_usleep(microsec=50000) at pgsleep.c:56
        frame #2: 0x000000010ad16584 postgres`FtsNotifyProber at cdbfts.c:129
        frame #3: 0x000000010ad02a52 postgres`cdbgang_createGang_async(segments=0x00007fbd64821bc0, segmentType=SEGMENTTYPE_EXPLICT_WRITER) at cdbgang_async.c:302
        frame #4: 0x000000010acffcb4 postgres`cdbgang_createGang(segments=0x00007fbd64821bc0, segmentType=SEGMENTTYPE_EXPLICT_WRITER) at cdbgang.c:90
        frame #5: 0x000000010acffe36 postgres`AllocateGang(ds=0x00007fbd6484d4d8, type=GANGTYPE_PRIMARY_WRITER, segments=0x00007fbd64821bc0) at cdbgang.c:127
        frame #6: 0x000000010acfe62f postgres`cdbdisp_dispatchCommandInternal(pQueryParms=0x00007fbd64821c20, flags=0, segments=0x00007fbd64821bc0, cdb_pgresults=0x00007ffee579ba40) at cdbdisp_query.c:416
        frame #7: 0x000000010acfe5d2 postgres`CdbDispatchCommandToSegments(strCommand="select gid from pg_prepared_xacts", flags=0, segments=0x00007fbd64821bc0, cdb_pgresults=0x00007ffee579ba40) at cdbdisp_query.c:349
        frame #8: 0x000000010acfe4e9 postgres`CdbDispatchCommand(strCommand="select gid from pg_prepared_xacts", flags=0, cdb_pgresults=0x00007ffee579ba40) at cdbdisp_query.c:321
        frame #9: 0x000000010ad72105 postgres`gatherRMInDoubtTransactions at cdbdtxrecovery.c:274
        frame #10: 0x000000010ad71b32 postgres`recoverInDoubtTransactions at cdbdtxrecovery.c:198
        frame #11: 0x000000010ad71946 postgres`recoverTM at cdbdtxrecovery.c:146
        frame #12: 0x000000010ad718de postgres`DtxRecoveryMain(main_arg=0) at cdbdtxrecovery.c:644
        frame #13: 0x000000010a98c2bf postgres`StartBackgroundWorker at bgworker.c:753
        frame #14: 0x000000010a9a4eab postgres`do_start_bgworker(rw=0x00007fbd63e00510) at postmaster.c:6173
        frame #15: 0x000000010a99ee39 postgres`maybe_start_bgworker at postmaster.c:6413
        frame #16: 0x000000010a99f3cb postgres`ServerLoop at postmaster.c:2018
        frame #17: 0x000000010a99c7b3 postgres`PostmasterMain(argc=6, argv=0x00007fbd63f05fd0) at postmaster.c:1515
        frame #18: 0x000000010a85f1ff postgres`main(argc=6, argv=0x00007fbd63f05fd0) at main.c:245
        frame #19: 0x00007fff5b549ed9 libdyld.dylib`start + 1
    (lldb)
