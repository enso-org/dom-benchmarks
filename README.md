# Visualization rendering benchmarks

This document describes how we intend to choose the display method for JavaScript visualizations inside the IDE. The same steps can be used in the future to check if the display method should still be used or if another method is better by then. Some additional notes can be found in `Additional notes.md`.

## Problem

We are looking for the best method to display visualizations inside the IDE. Here is a list of some things that we are interested in:

1. high number of FPS in different scenarios
2. avoiding long render pauses, even when the average number of FPS is high
3. high graphics quality, at least when the visualizations are not moving on screen

## Procedure

Here is the precise procedure that we use:

1. We create benchmarks that are supposed to test all possible configurations and to be representative of  the performance of visualizations inside the IDE. We run those benchmarks on some combinations of macOS, Windows and Linux with Chrome, Firefox, Safari, Opera and Edge. (Note that results can also depend on the hardware.)
2. We judge each configuration based on the average number of FPS in the benchmark results.
3. We watch the best configurations in action to check if they have any apparent flaws (like blurriness or stuttering).
4. We choose the the best configuration that has no flaws and use it to display visualizations while they stand still. When they are in motion then the IDE should automatically switch to the configuration that  is best if we permit blurry graphics.

## Benchmarks

The file `Benchmarks.html` will run all our benchmarks when opened in a web browser. This should take two to three hours. The browser window should be displayed in full screen mode during the whole time. If the screen turns off while the benchmarks are running then the results will likely not be accurate. The configuration and result of each benchmark is immediately printed to the browser's console as a JSON array. When all benchmarks are over, an array of all benchmark configurations together with their results is printed to the console. To analyze this data, we convert it to a spreadsheet through tools that can be found on the web.

Each benchmark runs for five seconds and displays 1000 dummy visualizations on screen consisting of an SVG with 100 circles in it. Those nodes are children of one common div element that we call root. The root is moved around and zoomed through different methods that depend on the benchmark. The following list contains all parameters that we examine. There is one benchmark for each combination of parameters.

1. `translateMethod`

   This parameter can be `"translate"` or `"topleft"`. In the first case, we move the root through `transform: translateX(...)`; in the second case, we move it through `left: ...`.

2. `zoomMethod`

   This can be set to `"scale"` or `"translateZ"`. The first will zoom the root through `transform: scale(...)`; the second will set the `perspective` property on the root's parent and zoom by moving the root closer to the camera through `transform: translateZ(...)`.

3. `rootWillChange`

   Can be `true` or `false` and determines if we set `will-change: transform` on the root.

4. `childrenWillChange`

   Determines if we set `will-change: transform` on each visualization.

5. `rootForce3D`

   Rotates the root slightly to the side, potentially forcing the browser to use a different rendering method.

6. `childrenForce3D`

   Rotates each visualization slightly to the side, potentially forcing the browser to use a different rendering method.

7. `rootPerspective`

   Sets `perspective: 1000px` on the root's parent.

8. `childrenPerspective`

   Sets `perspective: 1000px` on the root (the parent of all visualizations).

9. `preserve3D`

   Sets `transform-style: preserve-3d` on the root, combining all visualizations into a single 3D context.

10. `shadowChildren`

   Wraps each visualization in a shadow DOM.

## Result

Our benchmark results can be found in `Benchmark results.xlsx` which can be opened with most spreadsheet applications. The results were recorded in Chrome 90.0.4430.93 and Firefox 88.0 on Windows 10 version 20H2 on a Lenovo IdeaPad 5i with a 15.6 inch 141 PPI display, using `Benchmarks.html` from commit e261eb4f44075b1bef8bb26f74a28236cc5fadc3.

The spreadsheet contains two sheets, "results" and "switching rootWillChange". The first one contains one row for each configuration that we tested. The second represents two configurations per row, one for `rootWillChange: true` and one for `rootWillChange: false`. This will be helpful if we decide to switch the `will-change` parameter during runtime while keeping the other parameters fixed.

The spreadsheet also contains notes about the flaws that I observed in the best performing benchmarks. Some of them were blurred and some caused glitches like those recorded in `Glitches.gif`. The best configuration with blurring, and the best configuration without any flaws at all are highlighted in the spreadsheet and listed in the following table.

|                     | Blurry    | Sharp     |
| ------------------- | --------- | --------- |
| Firefox FPS         | 2.6       | 1.3       |
| Chrome FPS          | 59.5      | 6.0       |
| Average FPS         | 31.0      | 3.6       |
| translateMethod     | translate | translate |
| zoomMethod          | scale     | scale     |
| rootWillChange      | true      | false     |
| childrenWillChange  | false     | false     |
| rootForce3D         | false     | false     |
| childrenForce3D     | false     | false     |
| rootPerspective     | false     | false     |
| childrenPerspective | false     | false     |
| preserve3D          | true      | false     |
| shadowChildren      | false     | true      |

We could use the configuration with sharp graphics all the time or switch to the configuration with blurred graphics but better performance whenever we expect a motion to start during the next frames. In the second case, it might be problematic that the configurations differ in `preserve3D` and `shadowChildren`. Switching `transform-style: preserve-3d` on or off could cause unnecessary repaints. Wrapping visualizations in shadow DOMs or unwrapping them during runtime would cause even more expensive calculations and would affect the styling of visualizations. For this reason, we have the sheet "switching rootWillChange". It allows us to choose the best pair of configurations that have all parameters in common except for `rootWillChange`, such that `rootWillChange: false` produces sharp graphics and both configuration are glitch-free:

|                                 | Compromise |
| ------------------------------- | ---------- |
| Firefox FPS (no rootWillChange) | 2.0        |
| Chrome FPS (no rootWillChange)  | 4.7        |
| Firefox FPS (rootWillChange)    | 2.8        |
| Chrome FPS (rootWillChange)     | 58.6       |
| Average FPS                     | 17.0       |
| translateMethod                 | topleft    |
| zoomMethod                      | scale      |
| childrenWillChange              | false      |
| rootForce3D                     | false      |
| childrenForce3D                 | false      |
| rootPerspective                 | true       |
| childrenPerspective             | true       |
| preserve3D                      | true       |
| shadowChildren                  | false      |