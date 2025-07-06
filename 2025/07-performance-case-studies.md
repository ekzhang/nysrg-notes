- Case studies on software performance (Jul 2025)
    - uv
        - Looks like we were able to generate a reasonable flame graph, targeting some good build options. Optimizations turned on, but also debug symbols.
        - ```shell
          #!/bin/bash
          
          # cd into directory of the script
          cd "$(dirname "$0")/"
          
          UV=$(realpath ../target/profiling/uv)
          
          sudo rm -rf .venv
          sudo rm -rf cargo-flamegraph.trace
          
          $UV venv
          $UV python install 3.13.3
          
          echo "uv managed python list:"
          ls -Gh ~/.local/share/uv/python
          
          sudo flamegraph --root -- \
          $UV --managed-python --color always pip install --no-cache torch
          ```
        - https://gist.github.com/ekzhang/01c72db3c76387918c4d95db7d6df0d3
        - Debug flamegraph has a lot more rectangles / nodes, probably because many function calls get inlined out when compiling in release mode. In particular, you can see the whole call stack for uv_client / uv_distribution and zlib while in debug mode, but most of the calls get turned into zero-cost abstractions when optimizations are enabled.
        - Creating files actually seems to be a nontrivial number of samples in release mode. I wonder why, that's kind of interesting. Probably an artifact of the event loop removing call stacks — actually file creation originates from cached requests & populating site-packages?
        - Reading some code now, I'm starting to get a sense for the shape of `uv pip install`.
            - There's a lot of code here, maybe some 6k lines per uv-client, uv-distribution crates, and then another 20k in uv/src/commands (the CLI) and so on.
            - A lot of the code seems to be unit tests though (probably at least 50%), for data structures with a whole bunch of manual serialization impls (why not use Serde? I guess custom logic for Arch, Os, Python version, etc.) and request constructors. It's a very organized way of implementing a complex protocol.
            - uv-python: data structures for environment preference, interpreter version, filenames, "missing version" errors, platform, etc.
            - uv-client / CachedClient: This seems to be in the hot path of `uv pip install`, and probably other requests. It wraps a registry client, and there's some implementation of the HTTP caching headers framework to store downloaded blobs on disk.
                - BurntSushi wrote a bunch of this code, you can tell from the extremely detailed comments haha. Really thought through every design choice in data layout.
            - Interesting that they're using the async-compression crate, guess that's just fast enough. It makes sense. Networking is probably going to be your bottleneck for most developer or CI setups; this isn't a reverse-proxy or network intensive service.
            - uv-distribution / DistributionDatabase: Also on the hot path. Converts a Dist (source or binary distribution) object, which references registry/git/path/url/…, into a wheel or wheel metadata. Does this via a `client` (see above) and a `builder`.
                - Charlie Marsh is also big on comments. They stand out a lot. Documenting historical and present behavior, finding hashes, following links, validating metadata and so on. I guess this is what it takes to bring order to the chaos of Python packaging.
                - This package does seem very complex though. There's a lot of branching factor in every step of the code, as it decides which path to take based on cache availability, archiving, and recency.
                - (I guess this is good since it handles all the possible cases like editable vs git vs remote vs registry. Curious how it evolved though. Is there a spec?)
            - uv-resolver: This is a beast, some 20k lines of code split up between package resolution, lockfile format (7.5k), [PubGrub algorithm](https://nex3.medium.com/pubgrub-2fb6470504f) (2.5k), the core resolver itself (4.5k), and the top-level modules which look to be some kind of configurable, high-level graph algorithm. (wtf is UniversalMarker?)
                - PubGrub is cool. A concrete algorithm based on ideas from constraint solving systems, ASP / SMT techniques, symbolic stuff.
                - Now I know why uv doesn't do that stupid poetry / pip thing where it goes through every O(n^2) versions of aiobotocore + aioboto3 checking for compatibility with a particular version of botocore, lol.
        - Ran an [old fork of tokei](https://github.com/nviennot/tokei) on the repository, looks like there is 126k lines of Rust and 165k lines of Rust tests, including cfg(test) blocks. Lots of code!
            - 143k lines of tests are in the `crates/uv/tests` folder, lol. I guess that's where a lot of the CLI-level tests live, which makes sense. It's fast enough as-is, and those tests are separate from the main build, so they don't slow down development.
            - The core uv-client, uv-distribution, uv-installer code is only about 4k, 5k, 2k lines though, including all of the parsing and struct boilerplate.
            - I guess that's pretty normal for implementing a packaging standard, with caches!
            - (But something like ty is likely much more complex — so many PL / compiler data structures tailored for static analysis.)
        - I guess, strategically, if I was focused on uv performance (assuming uv was somehow slow, which it's not atm but maybe was early on), I'd look at a couple axes:
            - Making sure stuff that can be cached gets cached. The bulk of the code in uv already focuses on this, afaict, lots of branching and casework to "do it right."
            - Reducing I/O overhead. Using the best zero-copy syscalls, making a really good HTTP client with retries, building a solid on-disk data structure.
            - Improving the resolution algorithm to behave predictably, install less duplicates, run faster for large projects with many dependencies. Also, prefetching or optimizing registry invocations during resolution to avoid blocking serial requests.
        - Non-focus would be on parsing, fs, or compression performance, since it seems like that's not the bottleneck anyway. Rust is plenty fast enough with -opt-level=3.
        - More profiling methods, just for fun: dtrace. Going to check out file I/O.
            - Hmm, dtruss isn't working unless I disable System Integrity Protection.
              ```shell
              STAMP=$(date +%Y%m%d-%H%M%S)
              TRACE="uv-fileio-${STAMP}.log"
              sudo dtruss -f -n uv \
                -t open -t open_nocancel -t openat -t openat_nocancel \
                -t read -t read_nocancel -t write -t write_nocancel \
                -- "$UV" --managed-python --color always pip install --no-cache torch \
                2> "$TRACE"
              ```
            - Not going to do that right now as it would take too long. Won't learn dtrace / dtruss then unless you're doing macOS-specific work and need to profile it there, with SIP disabled. Most cloud stuff is Linux anyway, just bpftrace.
                - Gleb was able to get dtrace working, but he disabled SIP. Visualized the flamegraph from it after figuring out commands.
                - It was more detailed than the sampling profiler, but showed the same basic idea.
        - Is anyone profiling with Instruments on macOS?
            - Seems like a bunch of people are using this. Also [speedscope](https://github.com/jlfwong/speedscope#usage) as a trace visualizer. Although honestly it doesn't seem to be that much more featureful than flamegraph SVGs, which have the advantage of being able to be "dropped" anywhere.
            - Honestly I kind of wish there was a desktop app for these. Browsers are fine I guess, but nothing beats the convenience of double-clicking on a file.
            - Another interesting link: https://github.com/s7nfo/Cirron
