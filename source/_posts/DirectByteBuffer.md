---
title: DirectByteBuffer
author: ygdxd
type: 原创
date: 2019-04-28 15:09:41
tags: nio
---

### 堆外内存DirectByteBuffer
DirectByteBuffer是jdk提供的访问对外内存的一种实现，堆外内存的优势在于，1.使用socket网络传输时，它能够节省堆内存到堆外内存的复制消耗。2.对于磁盘io,可以使用内存映射，提高效率。3.不需要考虑gc问题。它并不受jvm内存管理,所有当对外内存不足时，系统会显式地调用一次System.gc()，如果还是不能申请到足够内存，系统就会报出


{% codeblock %}
java.lang.OutOfMemoryError: Direct buffer memory

{% endcodeblock %}

<!-- more -->

如果我们在jvm参数上加上 -XX:+DisableExplicitGC 那么就会使显式gc无效。同时我们也可以通过增大-XX:MaxDirectMemorySize来增加堆外内存

下面看一下DirectByteBuffer的创建
{% codeblock DirectByteBuffer.java lang:java %}

DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        //是否对齐
        boolean pa = VM.isDirectMemoryPageAligned();
        //默认每页的大小
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        
        //在申请或者释放内存时都应该调用  里面通过CAS来count
        //在这个方法中 如果检测到内存不够就会显式调用system.gc()
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            //调用jni申请内存
            base = UNSAFE.allocateMemory(size);
        } catch (OutOfMemoryError x) {
        	  //cas 减少数量
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        UNSAFE.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        //添加cleaner 用于回收内存
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;


{% endcodeblock %}


在看下Cleaner,它继承了PhantomReference。 PhantomReference的作用在于跟踪垃圾回收过程

{% blockquote %}

Phantom reference objects, which are enqueued after the collector determines that their referents may otherwise be reclaimed.
{% endblockquote %}

Reference中有个ReferenceHandler 它会一直检测getAndClearReferencePendingList()获得的list,


{% codeblock Reference.java lang:java %}

private static void processPendingReferences() {
        // Only the singleton reference processing thread calls
        // waitForReferencePendingList() and getAndClearReferencePendingList().
        // These are separate operations to avoid a race with other threads
        // that are calling waitForReferenceProcessing().
        waitForReferencePendingList();
        Reference<Object> pendingList;
        synchronized (processPendingLock) {
            pendingList = getAndClearReferencePendingList();
            processPendingActive = true;
        }
        while (pendingList != null) {
            Reference<Object> ref = pendingList;
            pendingList = ref.discovered;
            ref.discovered = null;
			  //这里如果ref是Cleaner 就会调用它的clean()方法
            if (ref instanceof Cleaner) {
                ((Cleaner)ref).clean();
                // Notify any waiters that progress has been made.
                // This improves latency for nio.Bits waiters, which
                // are the only important ones.
                synchronized (processPendingLock) {
                    processPendingLock.notifyAll();
                }
            } else {
                ReferenceQueue<? super Object> q = ref.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(ref);
            }
        }
        // Notify any waiters of completion of current round.
        synchronized (processPendingLock) {
            processPendingActive = false;
            processPendingLock.notifyAll();
        }
    }


{% endcodeblock %}

而cleaner会调用Deallocator里面的

{% codeblock DirectByteBuffer.java lang:java %}
public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            //释放内存
            UNSAFE.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

{% endcodeblock %}



### netty堆外内存

AbstractByteBufAllocator 

protected abstract ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity);

在它的实现里有池化和非池化，这里主要介绍池化策略。

###### PooledByteBufAllocator

PooledByteBufAllocator采用的是jemalloc来进行内存分配。jemalloc将内存划分为一个个Arena，而在PooledByteBufAllocator里面，程序维护了heapArenas和directArenas，分别代表堆内和堆外Arena。每个Arena又有多个chunk组成。可以看到PoolArena中有PoolChunkList - 存储chunk，SizeClass枚举类 - 对分配内存的大小作区分，tinySubpagePools - 用来保存为tiny规格分配的内存页的链表， smallSubpagePools -用来保存为small规格分配的内存页的链表。

{% codeblock PooledByteBufAllocator.java lang:java %}

@Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) {
            //分配内存
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
{% endcodeblock %}




{% codeblock PoolArena.java lang:java %}

private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        //计算大小 
        final int normCapacity = normalizeCapacity(reqCapacity);
        //根据大小选择不同的PoolSubpage 和 allocate
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                //具体allocate
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                //计算出索引
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }
            //准备放到对应的page数组中
            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
                    incTinySmallAllocation(tiny);
                    return;
                }
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
            }

            incTinySmallAllocation(tiny);
            return;
        }
        if (normCapacity <= chunkSize) {
            //在缓存外分配
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
                ++allocationsNormal;
            }
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
    }


{% endcodeblock %}

我们可以看到分配内存的信息会被保存在Aerna的chunk和page对应链表中。同时它会根据请求分配的大小选择不同的chunk和page.

### 参考

[堆外内存之 DirectByteBuffer 详解](http://www.importnew.com/26334.html)

[涤生](https://www.jianshu.com/p/8e942cd7b572)

