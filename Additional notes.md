# Notes

This document contains some notes that might be relevant for the efficient rendering of our JavaScript visualizations inside of web browsers. All concrete FPS numbers refer to the benchmarks in `index.html`, commit 5793eaf1ab80131474eb2353eb58c0962bfc87fa, with `elementType: "html"`.  The commit also contains the specific benchmark results.

## Browsers

- I ran the benchmarks in Chrome 90.0.4430.93, Firefox 88.0 and Opera 76.0.4017.94 on Windows 10 version 20H2 on a Lenovo IdeaPad 5i with a 15.6 inch 141 PPI display.

- Opera behaves  very similar to Chrome since they use the same rendering engine.

- Chrome reached 60 Fps when it seems to cache the scene as a texture instead of painting it on every frame. But this causes blurry graphics. We can achieve similar performance with better texture resolution if we scale elements using `transform: scale(...)` to increase their resolution and then set `will-change: transform` after that to trigger optimizations. This would still be a compromise seems to stay fixed after we set `will-change`, no matter how far we zoom in. When the image stands still, even without zoom, the results of this approach are visibly inferior to the default rendering method.
- In benchmarks without blurring, Chrome reached around 25 Fps .
- Firefox reached only 20 Fps when using `transform` instead of setting `left`/`top` for translations, which is necessary to avoid occasional long DOM updates that would occur roughly every 2 seconds and would cause multiple dropped frames in a row.
- In benchmarks without blurring, Firefox reached around 17 Fps.
- Chrome is much faster in the SVG benchmarks.

- If we observe that visualizations cause performance problems in practice then we should likely advise users not to use Firefox.

## Observations

- Setting `transform-style: preserve-3d` on the root div
    - Can improve performance.
    - The advantage from this is easy to break, for example if we wrap each child in an additional div.
    - It does not seem to affect the performance of the SVG benchmarks.
    - In Firefox, this can do both, preventing and causing blurry graphics
- Setting `will-change: transform`
    - Improves performance when we avoid the problems mentioned below.
      
    - The property exists to tune performance.
      
    - It can easily be set and unset when the motion starts or ends. In contrast: If we switch between fundamentally different set-ups, for example between `transform: scale(...)` and 3D transformation for zooming, this would make the implementation more complex and would likely be more difficult for the browser which does not understand what we are trying to achieve and would be forced to repaint elements immediately.
    - As stated in the specification, this should be set early enough to allow preparation. There is no clear definition of "early enough". One or two frames would likely suffice, but this depends on the exact scenario.
    - Usually leads to blurry graphics, especially when we zoom in after setting it.
      
    - In Chrome, scaled elements will keep at least the resolution they had when the property was set. In Firefox, they seem to scale to arbitrary resolutions.
    - Setting this on too many elements can lead to bad performance.
    - If the total on-screen area is too large (achievable through overlapping elements) then browsers will not use hardware acceleration.
    - Large areas off-screen do not cause problems, because the browser will split them into independent tiles and only render the visible ones.
    - Impact in Chrome can be huge. In Firefox, it is only around 3 Fps.
    - Can improve the look of motions when it prevents elements from aligning with pixel boundaries.
- Rounding coordinates to integers
    - Improves performance.
        - Likely because it requires less antialiasing, maybe because it avoids some re-rendering after sub-pixel shifts.
    - Things jump from pixel to pixel.
    - Without hardware acceleration, it can even improve the look because the scene jumps as a whole, which avoids the wobble effect that we see when individual parts align to pixel boundaries and jump independently.
        - The same might be achievable if we make sure that the offsets between elements are whole pixel units. For example, we could snap visualizations to pixel boundaries when they get dropped. But zooming will likely break such attempts.
- Setting `perspective` can trigger hardware acceleration in Firefox, even when only 2D transforms like `scale` are used.
- Setting `top` and `left`
    - Improves performance in Firefox.
    - Leads to roughly 8 more Fps.
    - Contradicts popular advice.
    - Leads to occasional long pauses for DOM updates.
- Forcing hardware accelerated rendering through 3D transformations
    - Can improve performance.
    - Often leads to blurry graphics, worse compared to setting `will-change`.
    - Does not work reliably when the browser recognizes that it can be simulated in 2D. This can depend on factors like the size of an element or the viewport.
    - Leads to visual glitches when the total area is too large and some things cannot get rendered.
    - I see no advantage to setting `will-change`.
- If visualizations contain elements with non-default positions or similar layout-related properties then Chrome will traverse them on transform changes of the root, even when hardware acceleration is used and they obviously do not need to be re-rendered. The Chrome profiler displays this as "Update Layer Tree". I tried many solutions, including the `contain` property. But nothing seems to help. Firefox seems similar.
- Similar to "Update Layer Tree", there might be expensive style recalculations in some settings. They can be prevented by shadow DOMs. But too many shadow DOMs seem to cause bad performance in Chrome.

## Suggestions

- Use `scale` for zooming to get best quality and be sure that we do not accidentally trigger hardware acceleration.
- Set `transform-style: preserve-3d`. We would use this even if it did not have any performance benefits. We should keep in mind that the performance benefits are not totally reliable and it might cause blurry graphics in Firefox.
- Maybe compare performance and appearance with and without `will-change: transform` in realistic situations.
    - We could try to enable it only when we expect motions.
        - Ideally, we would do this before the motion starts.
        - This would increase the code's complexity.
    - Setting it on the whole scene or on individual visualizations can both make sense.
        - The first will not help when individual visualizations are moved.
        - In the second case, we should only set it when that visualization moves independently from the rest. This way, we can avoid performance problems.
    - To control the resolution, we can scale an element before we set `will-change`. It might be necessary to trigger a style recalculation or even a repaint in between.
    - We can easily do this at an arbitrary point in the future.
- If we set `perspective` then we should check if it has any negative impact.