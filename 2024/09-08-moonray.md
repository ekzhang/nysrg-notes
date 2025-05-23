- MoonRay — [[September 8th, 2024]]
    - AOV system is interesting. Light path expressions primarily.
        - "Material AOV" provides different kinds of syntax for debugging materials.
    - Use of different JIT compilation through LLVM to implement their ISPC framework. SPMD computation on an Arras cluster.
        - They take a JSON describing the material and compile it down into shaders!
        - Kind of cool to have an out-of-the-box optimization.
        - ISPC / SPMD is close to a CUDA thread actually, running multiple of them within the same core. Only works for this compiler. It "looks like" gang scheduling, but in reality it's perhaps converting all of the instructions into AVX?
        - Exotic execution model, CMU slides: https://www.cs.cmu.edu/afs/cs/academic/class/15418-s18/www/lectures/03_progmodels.pdf
        - https://ispc.github.io/ispc_for_xe.html
        - This is also why the BVH data structure is important? Intel Embree vs Nvidia RTX
    - Vectorized path tracing, i.e., uses all SIMD lanes.
    - Relies on several queues for radiance (dedupe samples), rays, and shading. Each queue is processed by a handler when it reaches its maximum size, for efficiency. Shading queue converts AoS -> SoA and then evaluates every shader & texture with SIMD.
        - Custom image loader with OpenImageIO (OIIO)
        - Integrator spawns rays with Monte Carlo which are incoherent, get placed in a separate queue for sorting.
        - Occlusion rays to lights are also generated and handled asynchronously.
    - "Vectorized Production Path Tracing" (HPG '17) has more context on this.
        - Embree for ray intersections, two uni-directional path tracing implementations based on depth-search (scalar) and wavefront breadth-first (SIMD / ISPC).
        - __"do the potential performance benefits gained outweigh the extra work required to harness the vector hardware?"__
        - ![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2Fekzhang%2FffK7c2zelb.png?alt=media&token=2c1fa5de-838f-49db-8619-79435a63a3b4)
        - Integrator / secondary rays spawner implements multiple importance sampling, Russian roulette, path splitting.
        - "Another key tenet of DOD is __where there is one, there is usually more than one__, or more plainly, we should work in batches where possible."
        - Intel TBB is how tasks are spawned.
        - Sorting by [light-set index, UDIM tile, mip level, uv coordinates] in a 32-bit integer.
            - Up to 128 light sets.
        - Then every shader node is vectorized, since they are JIT compiled into an ISPC program and executed on the processor on batches. Texture sampling is done with OIIO loading / caching, and they do point sampling between two adjacent MIP levels for "instant" lookup.
        - Thread-local primarily to avoid contention. The shade queue is not thread-local, for memory reasons. Queues kind of imply BFS because they are limited in size.
    - Vectorized path tracing is about decomposing the system into smaller parts based on their data / memory access patterns. Like a little factory with different machines sending messages.
        - Ray queue is only concerned about global geometry, nothing else.
        - Shading queue cares about materials, light, sampling … but only at one single point.
        - Radiance queue is just environmental illumination.
        - Occlusion queue only works on scene lights and importance sampling.
