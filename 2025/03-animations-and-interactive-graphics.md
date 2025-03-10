- Animation and interactive graphics (Mar 2025)
    - Starting off strong by compiling a list of a bunch of websites, pretty neat.
    - Animated Vega-Lite (VIS '23)
        - Goal is to unify interaction and animation. Remember that these two are related: both of them slice up the data based on some parameter, either time or some other click.
        - Time as an __encoding__ versus as an __event stream__.
        - You can drag a slider, and that would be equivalent to the clock moving forward in an animation. Kind of like the playback timeline in a video. But interaction can be for things other than time in Vega, like focusing on a particular series.
        - Zoom/pan as a "view transformation."
        - Vega-Lite has two encodings "x" and "y" for spatial position, as well as other encodings like "hue" and "size" — this paper introduces one for "time" as a field of data. Infer a default for tweening between points based on class of data, so you get animations!
        - Interesting that the default is 500ms per domain value. Information density?
        - "rescale" by time: interesting property, is this parameter in the rest of Vega-Lite as well? Feels like it could be applied to other slices / interactions with data.
        - Specific examples motivate the design of the grammar. I think this makes sense as a guiding principle. Over time, when designing creative tools / languages, you can look at what people have created in the past, and then discover how to express that again.
        - Sidenote: Lol the paper refers to itself as "low viscosity" instead "frictionless"!
    - Lottie animated graphics format https://lottie.github.io/lottie-spec/latest/single-page/
        - Recall that the purpose is to be exported from Adobe After Effects as a vector graphic, then embedded into software via the runtime library.
        - Compact JSON format. Gradients are a flat array of numbers, color & opacity stops.
        - Some properties can be animated. An animated property consists of an array of __keyframes__. Each keyframe is a timestamp, as well as two cubic bezier handle points for the interval before and after that keyframe. They form a cubic curve from from [0,0] to [1,1].
        - Just skimming. Besides keyframes and animations (spec unclear to me right now), the rest of the format is just: SVG shapes, colors, stroke/fill/mask/line caps, and transforms. Simple!
        - Animation = name, layers[], framerate, framenumbers, width/height, assets (reference, data URIs), markers (named time ranges), and slots (constants).
            - Layers can have children, so they follow the movement, kind of like an SVG group. It forms a transform hierarchy. Also they can have an ip..op range of frames in which they're visible (optimization for sequencing?).
            - Types of layers: solid bg, image (by asset ID), precomposition (component with other layers that can be transformed or referenced again), null (group), or shapes.
        - Overall a pretty simple format. Basically like SVG, but with precomposition, enter/exit times, and every property (scalars, position, color, gradients) is allowed to be a list of keyframes that are played back in animation.
    - Post-processing shaders https://blog.maximeheckel.com/posts/post-processing-as-a-creative-medium/
        - Obviously dithering is possible, but I like this blog post's trick of rounding (u,v) in a shader as their efficient means for pixelating an image. Why downscale it yourself, when you can have the graphics pipeline do it properly for you with implcit mipmaps?
        - Lots of ideas with the pixelation, in the rough category of "shapes" or "colors" per pixel.
        - Seems fun to design around this for web graphics! They're __very easy__ postprocessing shaders to write, just a few lines of code, and very interactive.
        - Sidenote: The on-demand interactive code editors embedded into the website are pretty damn clean. They also rerender immediately. It's understated but very good frontend work to build this well.
        - Speaking of advanced image effects, you can do a lot with CSS! These image filters for simulating the light on cards are kind of awesome. https://poke-holo.simey.me/
    - More 3D work to collect from people
        - https://darkroom.engineering/
        - https://www.joshuas.world/
        - https://qrshow.nyc/
