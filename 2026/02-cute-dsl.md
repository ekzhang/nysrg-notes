- CuTe DSL (Feb 2026)
    - Note that CuTe layouts are column major by default!
        - If you index a layout with a single index, it will be unraveled columnwise.
    - Start with the thread blocks and threads model, remember warp divergence and shared memory allocation and so on.
    - Elementwise kernel is slow without vector loads. Also, each thread needs to do a few operations (~16) for maximum performance. Might just be an artifact of the loading the scalar though (coalesced).
        - There doesn't seem to be vector elementwise multiply instructions in PTX. Adding higher vector multiply doesn't seem to really help.
        - But the vectorized `ld.global.v4.f32` achieves higher throughput than just `ld.global.f32`, maybe optimized better at runtime.
    - `cute.zipped_divide` / `cute.tiled_divide` are equivalent utilities (up to layout modes), and they let you assign each block a tile, and each thread within that block a value layout within the tile.
        - Requires the layouts to be divisible by tile size, be careful.
    - `cute.composition()` takes a cute.Tensor and a cute.Layout together and applies them, similar to `cute.make_composed_layout()` for layouts.
    - `cute.coalesce()` like composition, just changes the Layout, this time by removing redundant dimensions or size-1 dims that can be combined together; I wouldn't trust this though since it might be written for a column-major format.
    - It's a bit hard to write CuTe DSL code, the layouts can be kind of arcane and hard to use. One trick is to write a minimal `@cute.jit` function with a print statement and call it. Then you can develop within a tiny sandbox of sorts.
    - `cute.compile(...).__ptx__` is available when `CUTE_DSL_KEEP_PTX=1`.
    - RMSNorm kernel (8192, 131072)
        - [Simple reduction](https://veitner.bearblog.dev/simple-reduction-in-cutedsl/) blog post gets to 3.1 TB/s on B200, but the Quack kernel is 5.3 TB/s - it's around 70% faster. Both are faster than PyTorch (1.2 TB/s)
        - In addition to the lack of cluster memory synchronization (inherently limits to 2/3 of memory bandwidth), I think we're missing the vectorized load/store. Like, even without the block reduction part we only get 3.9 TB/s (note: 3 accesses instead of 2). :(
        - Hmm maybe let's add vectorization next. Adding vectorization was a pain (dealing with TensorSSA vs cute.Tensor type mismatch), but now it's **4.4 TB/s**, nice!
            - Also wrote a two-step warp reduction sum, which was also 4.4 TB/s. I think this is the highest bandwidth that I will get with this basic approach, where we're doing 3 global memory accesses for each element (two loads, one store).
            - Yes, confirmed this by removing that part of the code. It's still 4.4 TB/s.
        - The "multi-mode" indexing of tensors is really jank, but cute.flatten() helps.
        - Remember that CuTe DSL is **column-major** unlike NumPy, so (1, 4) * (4,) = (4, 4).
            - ```plain text
              # CuTe DSL is actually somewhat column-major:
              
              # NumPy
              zeros((1, 4)) * zeros((4,)) = zeros((1, 4))
              
              # Cute DSL
              TensorSSA[1, 4] * TensorSSA[4] = TensorSSA[4, 4]
              
              
              # Indexing a zipped_divide tensor while keep the "mode" part of the rank around unless you call cute.flatten() on it
              
              gX = cute.zipped_divide(mX, (1, 4))
              
              gX.shape                              # ((1, 4), (m, n/4))
              gX[None, (0, 0)].shape                # ((1, 4),)
              cute.flatten(gX[None, (0, 0)]).shape  # (1, 4)
              ```
    - A lot of stuff is only documented in passing in the [Cute DSL API](https://docs.nvidia.com/cutlass/latest/media/docs/pythonDSL/cute_dsl_api/cute.html), this doesn't index well in Google search so you should keep this page pulled up.
    - GEMM and async copy foundations
        - For GEMM (high arithmetic intensity, compute-bound), the only job you have is to feed the tensor core as fast as possible. >90% of all the FLOPs are in the tensor cores.
            - Tensor cores apply a weird swizzled layout.
            - wgmma_async, then commit a group and wait on the group to finish later.
        - You can create multiple warpgroups and have them specialize. One loads data or manages queues, the other one pipelines a bunch of async operations into the tensor cores. Each SM has like 4 tensor cores on Hopper, and thread blocks are scheduled on SMs.
        - So your programming fundamentally shifts.
        - Previously it was doing compute, but now your PTX operations are for coordinating memory access, doing TMA loads, swizzling, pipelining MMA operations.
        - 2 CTA GEMM (Blackwell)
            - These are instructions for TMEM copy (load more data) and matrix multiplications, where the cluster (CGA) collaborates with a pair of CTAs, scheduled on two clustered SMs.
            - I saw some stuff online about "one CTA is not enough to keep tensor cores busy," which is technically true, but I think it's missing the point. It seems to just allow you to do __larger tile sizes__ in your GEMM, I think, which increases arithmetic intensity. Otherwise, these wouldn't really fit in shared memory of the SM.
            - This is also why TMA multicast is important, you make the loaded tile available to TMA of multiple SM's that use it without increasing GMEM traffic.
    - GEMM in CuTe DSL
        - Atom = represents complex matmul or copy instruction.
        - Functions in the cute.nvgpu namespace (also sm100_utils) are for copies and MMAs. In CUDA, you would have to write raw PTX, but this is simpler.
        - If you __do__ need inline PTX, you can hack into [LLVM/MLIR internals](https://gist.github.com/Observer007/5796c58355c3d373bc6f0b690de978c3) for that.
        - TMA lets you asynchronously do bulk copies from GMEM to SMEM or TMEM.
            - Background: https://research.colfax-intl.com/tutorial-matrix-transpose-in-cutlass/
        - The warp-specialized Hopper WGMMA is [here](https://cudaforfun.substack.com/p/outperforming-cublas-on-h100-a-worklog). It uses a simple architecture with one producer and two consumer workgroups. (see also: ping-pong for epilogues)
            - Question: Why do you have two consumer workgroups?
                - In this case, it's just adding more registers. Each thread has up to 128 registers; this is a hardware limit. So create more workgroups to collect output registers.
                - But I don't understand why you can't just store after WGMMA k-loop?
                - Hmm I think it should be possible. Maybe just a tradeoff, this would require more operations / less bulk transfers.
            - Question: How does it relate to ping-pong?
                - I think ping-pong lets you __overlap epilogues__ more cleanly than having to asynchronously do the epilogue of the last instruction while dispatching the async wgmma of the current instruction.
                - Otherwise you'd need a bunch of registers to store the last instruction, which is kind of a pain to deal with. Coordinating two consumers is easier!
        - Got a working version in https://github.com/ekzhang/kernels - starting with juts 1CTA GEMM right now, it's about 1.2 TFLOPS bf16 (PyTorch/cuBLAS is around 1.6, so ~75%).
        - This is pretty good already, just wondering about why 2CTA is useful. Sounds like it's just larger tile sizes, which then support raising arithmetic intensity (also, TMA multicast).
    - cuTile (Bryce: Triton but better!)
        - Philosophy: With each new GPU architecture, Nvidia gives you a broader toolbox to use when optimizing your kernels. But it's very complicated! cuTile let's you worry about writing your program with good experience, and the compiler figures things out.
        - Goal is to remove the heterogeneous computing model across GPU generations (Hopper/Ampere/Blackwell) etc., and you need that sometime.
        - You can use performance hints in cuTile / Tile IR (like how many CTAs per CGA, pipelining depth) and also on the load/store level (latency hint). People recommend tuning over all of those numbers, but maybe you'll get to 85%-90% of speed-of-light.
            - If you really need absolute speed, use CuTe. Or if there's gaps in algorithms that aren't as great in terms of performance.
            - Recommends using cuTile for most everything else, including scientific computing.
        - The compiler picks where tiles are stored (SMEM/GMEM/TMEM). Needs power-of-two sizes for now, like Triton.
        - Programming model is having immutable tile arrays, along with loads from global memory. The compiler knows when to materialize arrays or not. Everything is a "copy," not a view, and the "control thread" kind of executes in-order while distributing operations to other resources within the block.
        - We're already pushing the limit of numerical formats (fp32->16->8->4). Next, disaggregation and heterogeneous architectures, etc.
    - Questions for Bryce
        - Why did Nvidia not choose to invest more in Triton? What does cuTile offer?
            - A: Example, it's challenging to lower Triton's pointer model to TMA. Can't predict the future if not working within Nvidia, know the direction of future hardware.
            - A: They're also contributing to a specific nvtriton backend. (Portability)
            - A: cuTile is only releasing for Blackwell right now, not Hopper. Ampere/Blackwell are actually a lot more similar than they are for Hopper (e.g., int8), Blackwell started by gluing two Ampere dies together + tensor cores.
            - Any other innovations? — not just focused on DL.
                - Soon, dropping the power-of-2 restriction on shapes.
                - Future work on stenciling, SIMT and tile interop.
        - There are some things that might not translate well to tile formats. How does cuTile manage things like 2CTA GEMM on Blackwell, or warp specialization and producer/consumer warpgroups that are common in matmuls?
            - A: The compiler does the warp specialization for you, and the clusters. Thinks that the inherent limitation in Triton is the block loads, rather than the specialization.
        - How does GMEM coalescing actually work? What is the chip doing at runtime?
            - A: Part of the coalescing is about moving through L1/L2 caches. Believes that the SM may have an internal buffer, it goes there __before__ getting to the L1. So it actually gets coalesced before things get to L1. (Latency penalty, doesn't matter for GPUs.)
        - Why does 2CTA GEMM fundamentally need to exist? Is it about increasing arithmetic intensity due to the higher tile size? Or tc utilization?
        - How does Nvidia think about designing the higher-level CuTe DSL helpers / utilities, such as pipelines and copies, in a way that's useful but still flexible?
        - Were features like SMs always intended to be public? Or did more underlying architecture start getting exposed over time as people pushed performance? Is there an example of a feature that used to be implementation detail but is now crucial?
            - A: Originally the chips had no public documentation! The driver has a lot of patches over certain "bugs" in the chips. (Ship very fast, no open ecosystem.)
        - [DeepSeek-V3](https://arxiv.org/pdf/2412.19437) criticized that fp8 tensor cores silently truncate outputs to 14 bit mantissa. Why, and how should we think about accumulation precision tradeoffs?
            - "In the current Tensor Core implementation of the NVIDIA Hopper architecture, FP8 GEMM suffers from limited accumulation precision. After aligning 32 mantissa products by right-shifting based on the maximum exponent, the Tensor Core only uses the highest 14 bits of each mantissa product for addition, and truncates bits exceeding this range."
            - "a compromise due to the Hopper architecture’s hardware deficiency in FP8 GEMM accumulation precision"
        - Hopper’s wgmma shifted the mental model from warp to warpgroup. Blackwell’s tcgen05 and 2-CTA GEMM shift it again toward SM clusters. Do you think we’re moving toward an abstraction where the fundamental unit of compute is no longer a warp or CTA, but a multi-SM cooperative unit? And how should software abstractions evolve if that’s true?
