# Understand Cache

## Cache organization: block frames and blocks

Caches fetch data in chunks called blocks (or "lines"). The blocks form the basic unit of cache organization. 

When the CPU requests a byte from a particular RAM block, it needs to be able to determine three things very quickly:

1. Whether or not the needed block is actually in the cache. (Whether there is cache hit or cache miss);
2. The location of the block within the cache (cache hit).
3. The location of the desired byte within the block (cache hit).

The caches accommodate(容纳) all three needs by associating a special piece of memory, called a `tag`, with each block frame in the cache. The tag fields allow the CPU to determine the answer to all three of these questions.

There are several different way to organize the structure of cache. Let see them.

### Fully associative

The most simple scheme for mapping RAM blocks to cache block frames (we also call cache line) is called "fully associative" mapping. Under this scheme, any RAM block can be stored in any available block frame.

![Fully_Associated](.\picture\Fully_Associated.png)

The problem with this scheme is that if you want to retrieve a specific block from the cache, you have to check the tag of every block frames in the entire cache because the desired block could be in any of the frames. So the larger the cache the worse the delay gets.

### Direct mapped

In a direct-mapped cache, each block frame can cache only a certain subset of the blocks in main memory. It likes a Hash table. 

![Direct_Mapping](.\picture\Direct_Mapping.png)

In the above diagram we can see. Each of the red blocks (block 0, 8, 16) can only be cached in the red block frame (frame 0). Likewise, blocks 1, 9, 17 can only be cached in the frame 1, and so on.

So the number of tags that must be checked on each fetch is greatly reduced. For example, if the CPU needs a byte from blocks 0, 8 or 16, it knows that it only need to check the block frame 0 to determine if the desired block is in the cache. This is much faster and more efficient than checking every frame in the cache.

And also, the drawback is very clear. If we read RAM in this sequence 0, 8, 16, 0, 8, 16 and so on, every time will cause a cache miss.

Another example reflex the drawback is: if blocks 0-3 and 8-11 combine to form an 8-block "working set". In the same time, the cache can only store four blocks at most at a time. Due to block 0 and 8, block 1 and 9, and so on share the same block frame. So this can cause a lot of cache miss, meanwhile, only half of the cache can be used in this case.

### N-way associative cache

To get the benefits of direct-mapping but mitigate the drawback. The N-way associative cache is invented (is proposed). 

![N-way_associative](.\picture\N-way_associative.png)

In this diagram, we see the four-way associative cache. It combines the `fully associative` and `direct mapped` cache.

In simple description, the ram blocks will be first mapped to a block frames set, in the set, it's perform like a `fully associative` cache.

For example, let's look the diagram. All the red blocks can be stored in a arbitrary frames in the red set of frames (set 0), and any of the yellow blocks can be stored in any frames in the yellow set of frames (set 1). We can think like cut a fully associative cache into two, so the odds of block frame will be cached by set 0, and the even of block frame will be cached by set 1.

The cache pictured above is said `four-way associative` because the cache is divided into "sets" of four frames each. A larger cache could accommodate more sets, reducing the odds of a collision even more. If a cache has three sets, only 1/3 of the cache needs to be searched for a given block. In a cache with four sets, only 1/4 of the cache is searched.