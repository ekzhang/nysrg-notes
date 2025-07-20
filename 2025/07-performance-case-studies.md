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
    - Nano-vLLM
        - Simpler implementation of vLLM, was able to get better performance (tok/s) on specific configurations of running on certain hardware and so on.
        - LLM inference engines use paged attention, dynamic batching, prefill + generation steps.
        - This project is pretty small, ~1k lines of code. Simple PyTorch.
        - Got it running on Nvidia, although flash-attn has some janky installation requirements that make it incompatible with uv pip install. Also opened an mlx-llm .gputrace within Xcode Instruments.
        - Imports take 4.3 seconds warm (mostly flash-attn / torch), creating LLM for 0.6b parameters takes 3.05 seconds. The generate() then takes 9.8 seconds.
        - ```plain text
          calling llm.generate() with prompts: ['<|im_start|>user\nintroduce yourself<|im_end|>\n<|im_start|>assistant\n', '<|im_start|>user\nlist all prime numbers within 100<|im_end|>\n<|im_start|>assistant\n'] 7.751103448000322
          Generating: 100%|████████████████████████████████████████████████████| 2/2 [00:13<00:00,  6.61s/it, Prefill=23tok/s, Decode=25tok/s]
          returned from llm.generate() 29.576559246000215
          -------------------------------------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                                                             Name    Self CPU %      Self CPU   CPU total %     CPU total  CPU time avg     Self CUDA   Self CUDA %    CUDA total  CUDA time avg    # of Calls  
          -------------------------------------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                                                   llm.generate()        24.72%        3.269s       100.00%       13.227s       13.227s       0.000us         0.00%     906.361ms     906.361ms             1  
                            _compile.compile_inner (dynamo_timed)         8.64%        1.143s        15.91%        2.104s     210.385ms       0.000us         0.00%      21.056us       2.106us            10  
                                                     aten::linear         0.75%      98.789ms        10.94%        1.446s      50.000us       0.000us         0.00%     756.007ms      26.134us         28928  
                                                     aten::matmul         0.62%      81.869ms         8.54%        1.129s      39.032us       0.000us         0.00%     756.007ms      26.134us         28928  
                                                         aten::mm         5.45%     720.837ms         7.92%        1.047s      36.202us     753.055ms        76.38%     756.007ms      26.134us         28928  
                                         TorchDynamo Cache Lookup         7.82%        1.034s         7.82%        1.034s      23.897us       0.000us         0.00%       0.000us       0.000us         43264  
                                       Torch-Compiled Region: 0/3         4.95%     655.215ms         6.52%     862.084ms     113.194us       0.000us         0.00%      17.075ms       2.242us          7616  
                                       Torch-Compiled Region: 2/1         4.27%     564.879ms         6.08%     803.742ms     105.533us       0.000us         0.00%      15.067ms       1.978us          7616  
                                                   cuLaunchKernel         5.39%     712.399ms         5.40%     714.339ms       9.369us       3.016ms         0.31%       3.018ms       0.040us         76244  
                                       Torch-Compiled Region: 0/5         3.42%     452.837ms         4.76%     629.890ms      93.734us       0.000us         0.00%      12.569ms       1.870us          6720  
          -------------------------------------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
          Self CPU time total: 13.227s
          Self CUDA time total: 985.904ms
          ```
        - It took about 10 seconds to generate, but notably the CUDA execution time was only 900 ms, so there's a huge discrepancy. I think you need more prompts / batch size in order to make full use of the GPU. Probably it's low utilization at the moment.
        - Also, recording the profile with `with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA], record_shapes=True) as prof:` was fine, it only had a minor impact (<20% for sure) on the performance, but it took like a minute to actually collect the profile at the end and print out the `prof.key_averages()` line. Wonder what work it's deferring to the end of profile collection.
        - Switching gears to look at the codebase, there's BlockManager which hashes token IDs and produces blocks of allocated memory (Paged attention), and ModelRunner, which moves torch.tensor() calls and dynamically tracks them in cache.
        - prepare_prefill(), prepare_decode(), etc., with pin_memory=True for faster copies.
        - Overall a pretty simple LLM engine algorithm! What this profile reveals is that almost everything is matmul, and we could improve utilization / FLOPs in this example.
        - ```python
          import time
          start_time = time.monotonic()
          print("starting imports")
          
          import os
          from nanovllm import LLM, SamplingParams
          from transformers import AutoTokenizer
          
          # ERIC
          from torch.profiler import profile, ProfilerActivity, record_function
          
          
          
          def main():
              print("starting example.py", time.monotonic() - start_time)
              path = os.path.expanduser("~/huggingface/Qwen3-0.6B/")
              tokenizer = AutoTokenizer.from_pretrained(path)
              print("created autotokenizer instance", tokenizer, time.monotonic() - start_time)
              llm = LLM(path, enforce_eager=True, tensor_parallel_size=1)
              print("created LLM instance", llm, time.monotonic() - start_time)
          
              sampling_params = SamplingParams(temperature=0.6, max_tokens=256)
              prompts = [
                  "introduce yourself",
                  "list all prime numbers within 100",
              ]
              prompts = [
                  tokenizer.apply_chat_template(
                      [{"role": "user", "content": prompt}],
                      tokenize=False,
                      add_generation_prompt=True,
                      enable_thinking=True
                  )
                  for prompt in prompts
              ]
              print("calling llm.generate() with prompts:", prompts, time.monotonic() - start_time)
              with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA], record_shapes=True) as prof:
                  with record_function("llm.generate()"):
                      outputs = llm.generate(prompts, sampling_params)
              print("returned from llm.generate()", time.monotonic() - start_time)
              print(prof.key_averages().table(sort_by="cpu_time_total", row_limit=10))
          
              for prompt, output in zip(prompts, outputs):
                  print("\n")
                  print(f"Prompt: {prompt!r}")
                  print(f"Completion: {output['text']!r}")
          
          
          if __name__ == "__main__":
              main()
          ```
        - Probably better to use the graphical viewer later on. Just remember that there's a lot of profiling data, so you have to wait a while for the file to save / be created.
    - resvg
        - This is a minimal, reproducible and self-contained SVG renderer in Rust. It's about 2500 lines of code, and it's developed alongside the "usvg" parser, which is about 14000 lines of code. Compiles pretty quickly!
        - Unlike mainstream SVG implementations like in Chrome and so on, this one doesn't use system text rendering, it relies on rustybuzz / ttf-parser. So it's reproducible.
        - There's a codegen'd file, parser/svgtree/names.rs, seems to be a Lexer.
        - Alright, seems to be working. Here's how long it takes to render a [Paris map](https://upload.wikimedia.org/wikipedia/commons/7/74/Paris_department_land_cover_location_map.svg) in release vs debug mode, looks like a 20x difference in total actually.
        - ```plain text
          ubuntu@ip-10-1-4-234:~/resvg$ time ./eric-benchmark.sh 
          Warning (in usvg::text:129): No match for '"DejaVu Sans Condensed"' font-family.
          Warning (in usvg::text:129): No match for '"DejaVu Sans Condensed"' font-family.
          Warning (in usvg::text:129): No match for '"DejaVu Sans Condensed"' font-family.
          
          real    0m0.408s
          user    0m0.375s
          sys     0m0.033s
          ubuntu@ip-10-1-4-234:~/resvg$ time ./eric-benchmark.sh 
          Warning (in usvg::text:129): No match for '"DejaVu Sans Condensed"' font-family.
          Warning (in usvg::text:129): No match for '"DejaVu Sans Condensed"' font-family.
          Warning (in usvg::text:129): No match for '"DejaVu Sans Condensed"' font-family.
          
          real    0m8.005s
          user    0m7.962s
          sys     0m0.042s
          ```
        - Going to edit the SVG to remove "DejaVu Sans Condensed" since we don't have that font on my Ubuntu system. Seems like it's working with just "DejaVu Sans".
        - Alright, going to start profiling! I set up a `bench` build with debuginfo and no stripping.
        - Having `--width 10000`, I can get it to take longer, like 1.8 seconds. Going to standardize on `--width 2000` for now, which is taking about 460 ms right now.
            - Interesting that the time taken scales sublinearly with image width, even as the image dimensions are increasing quadratically. I guess most of the compute is probably spent parsing and simplifying the SVG in that case, not rasterizing.
            - Which makes sense — this means SVGs can be zoomed performantly, since you construct the in-memory data structure and save it for future rendering.
        - Here's a flamegraph for the workload. Seems like about 60% of the time is spent on rasterization in tiny-skia, and the remaining 40% is parsing.
        - https://gist.github.com/ekzhang/3a5e671e6eafb3fbda9c63551d046976
        - Note that running flamegraph takes a while… seems like over 100 MB/s of data generated by the samples. So that's kind of interesting. I wonder how that breaks down.
        - If you increase the resolution to width=10000, then it looks like the parsing stage disappears from the flamegraph (only 7% of samples now? — seems proportional) and now it's all tiny_skia crate spans, as well as some PNG conversion time.
            - Of the parsing time, about 70% of it is in `usvg::parser::converter`. This recursively goes through all the SVG elements. You can actually see the structure of the SVG through the flamegraph's stack trace distribution, showing how many nested levels of groups there are.
            - Converter is actually only a single file, not too large, less than 1000 lines of code. The main function seems to be convert_element(). It takes SvgNode from usvg's svgtree data structure, then for instance, convert_group() calculates bounding boxes and produces a Group struct, from usvg's tree module.
            - Interesting that resvg is taking usvg svgtree and producing usvg tree. I would've thought only the input data structure belongs to usvg.
            - Most parsing time is taken by parsing paths (and numbers inside the paths) to be converted, and likewise path bbox's, from a geometry data structure in tiny-skia's PathStroker. This is distinct from tiny-skia's drawing library with [Pixmap](https://docs.rs/tiny-skia/latest/tiny_skia/struct.Pixmap.html).
            - Also fun that this exercises only "fill paths" and "hairline strokes" — "anti_hairline" means an anti-aliased line with 1px width no matter what your zoom level is.
        - Now that I understand the library (pretty straightforward, just parses and renders the SVG file format), let's try some other profiling strategies for fun. Curious if I can count page faults, since I see a significant number of them in the flame graph.
        - Going to use bpftrace for this. Looks like I can use `handle_mm_fault`, cool!
        - Nice, I see like 23k-27k page faults with width=2000. And it increases maybe to 240k with width=10000. This makes sense, although not super sure why the memory usage keeps going up as we construct a pixel buffer.
            - `perf stat` is probably better for this for getting a summary report and navigating it with paths / debug info. bpftrace might be better at network operations and so on, even if it can technically do all the same things with enough effort.
            - Can see some other metrics like branches, makes sense.
            - This library isn't particularly optimized, and neither is its dependencies. But it's naturally fast enough, being in Rust and so on, and rustybuzz. Lots of room for performance optimization with buffers if you really needed it (see Forma, for instance).
