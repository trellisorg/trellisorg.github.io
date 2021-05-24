---
layout: post
title: Comparing Angular v11 (Webpack 4) to Angular v12 (Webpack 5) build performance and size 
categories: angular compiler webpack 
tags: build performance bundle
---

Recently Angular v12 was released, and with it Webpack 5 under the hood of the Angular CLI/Compiler. With it came a lot
of improvements, some of which are live in v12 and some are planned to be released in future version, ie. the ground
work has been laid.

This is a comparison of v11 (Webpack 4) to v12 (Webpack 5) using real world example. Each app here is within the Trellis
codebase. If some benchmarks are missing or seem off please feel free to tweet at
<a href="https://twitter.com/JayCooperBell">@jaycooperbell</a> and I will see what I can do about investigating and
fixing them!

For obvious reasons I cannot share the actual codebase these benchmarks and metrics were measured from, but I can share
the configurations and setups.

### Configurations

These benchmarks are a combination of Angular version, AOT enabled/disabled, `ng serve`, `ng build`, serve reload with
small and large changes, `build` with and without webpack cache.

Each of the builds was run using the following configuration (unless otherwise specified). Note: `aot: false`
assumes `buildOptimizer: false`.

1. `vendorChunk: true`
2. `optimization: true`
3. `buildOptimizer: true`
4. `aot: true`
5. `node --max_old_space_size=4096` - set for all commands so we do not run into javascript max heap errors

The `serve` configurations do not run with `optimization: true` as the Webpack dev server usually encounters a memory
issue.

In the following tables the following can be assumed:

1. A `small` change is either adding a duplicate `BrowserAnimationsModule` to the `AppModule` of the app or removing the
   duplicate. This may not be incredibly scientific, but it won't actually change the bundle size or compilation process
   in any meaningful way.

2. A `large` change for all but the `small` app is adding a property to an interface that is throughout the application.

3. The `node_modules/.cache` is cleared in between all `build` and `serve` runs as to ensure webpack was doing a clean
   build each time.

We will be looking at 4 different Angular apps today, we will refer to them as `large`, `medium1`, `medium2` and `small`
. They have the following setup:

<table>
    <tr>
        <th>
         Stat
        </th>
        <th>
            large
        </th>
        <th>
            medium1
        </th>
        <th>
            medium2
        </th>
        <th>
            small
        </th>
    </tr>
    <tr>
        <td>
            Number of Libraries
        </td>
        <td>
            400
        </td>
        <td>
            323
        </td>
        <td>
            250
        </td>
        <td>
            70
        </td>
    </tr>
    <tr>
        <td>
            Lazy Loaded Chunks
        </td>
        <td>
            98
        </td>
        <td>
            52
        </td>
        <td>
            27
        </td>
        <td>
            11
        </td>
    </tr>
</table>

### Size

I did not think we needed to go in deep on bundle sizes with/without `aot` and optimization etc because in v12
production mode is enabled by default, so the following table is just a normal prod build with all optimizations enabled.
All measurements are in megabytes (MB) and taken from the "parsed size" by using `webpack-bundle-analyzer`.

<table>
    <tr>
        <th>
         Version
        </th>
        <th>
            large
        </th>
        <th>
            medium1
        </th>
        <th>
            medium2
        </th>
        <th>
            small
        </th>
    </tr>
    <tr>
        <td>
            v11
        </td>
        <td>
            6.08
        </td>
        <td>
            4.96
        </td>
        <td>
            3.47
        </td>
        <td>
            1.92
        </td>
    </tr>
    <tr>
        <td>
            v12
        </td>
        <td>
            5.97
        </td>
        <td>
            4.72
        </td>
        <td>
            3.43
        </td>
        <td>
            1.92
        </td>
    </tr>
</table>

### Build Speed

- All speeds are measured in seconds
- All benchmarks were run on a Macbook Pro 2.3GHz i9 with 16GB of RAM.

#### ng build

<table>
    <tr>
        <th>
         Command
        </th>
        <th>
            large
        </th>
        <th>
            medium1
        </th>
        <th>
            medium2
        </th>
        <th>
            small
        </th>
    </tr>
    <tr>
        <td>v11 `build`</td>
        <td>359.182</td>
        <td>233.536</td>
        <td>138.212</td>
        <td>109.851</td>
    </tr>
    <tr>
        <td>v11 `build` (no aot)</td>
        <td>216.970</td>
        <td>121.133</td>
        <td>87.063</td>
        <td>56.726</td>
    </tr>
    <tr>
        <td>v12 `build`</td>
        <td>350.488</td>
        <td>236.748</td>
        <td>160.011</td>
        <td>137.307</td>
    </tr>
    <tr>
        <td>v12 `build` w/ webpack cache</td>
        <td>314.366</td>
        <td>251.175</td>
        <td>160.430</td>
        <td>95,080</td>
    </tr>
    <tr>
        <td>v12 `build` (no aot)</td>
        <td>264.419</td>
        <td>164.174</td>
        <td>112.823</td>
        <td>134.187</td>
    </tr>
    <tr>
        <td>v12 `build` (no aot) - w/ webpack cache</td>
        <td>219.058</td>
        <td>160.323</td>
        <td>113.743</td>
        <td>82.720</td>
    </tr>
</table>

#### ng serve

- The `small` and `large` in the first column determine how "big" of a change was made. See above for explanation.

<table>
    <tr>
        <th>
         Command
        </th>
        <th>
            large
        </th>
        <th>
            medium1
        </th>
        <th>
            medium2
        </th>
        <th>
            small
        </th>
    </tr>
    <tr>
        <td>v11 serve</td>
        <td>135.573</td>
        <td>100.668</td>
        <td>60.852</td>
        <td>34.609</td>
    </tr>
    <tr>
        <td>v11 serve (no aot)</td>
        <td>94.762</td>
        <td>70.415</td>
        <td>50.235</td>
        <td>64.678</td>
    </tr>
    <tr>
        <td>v12 serve</td>
        <td>215.846</td>
        <td>172.057</td>
        <td>117.094</td>
        <td>90.813</td>
    </tr>
    <tr>
        <td>v12 serve (no aot)</td>
        <td>144.112</td>
        <td>123.583</td>
        <td>85.858</td>
        <td>82.614</td>
    </tr>
    <tr>
        <td>v11 reload - small</td>
        <td>9.969</td>
        <td>12.993</td>
        <td>8.073</td>
        <td>2.312</td>
    </tr>
    <tr>
        <td>v11 reload (no aot) - small</td>
        <td>9310</td>
        <td>8195</td>
        <td>5194</td>
        <td>10782</td>
    </tr>
    <tr>
        <td>v11 reload - large</td>
        <td>28.928</td>
        <td>23.758</td>
        <td>13.593</td>
        <td>9.806</td>
    </tr>
    <tr>
        <td>v11 reload (no aot) - large</td>
        <td>25.792</td>
        <td>19.165</td>
        <td>11.815</td>
        <td>11.508</td>
    </tr>
    <tr>
        <td>v12 reload - small</td>
        <td>49.881</td>
        <td>44.062</td>
        <td>37.193</td>
        <td>21.538</td>
    </tr>
    <tr>
        <td>v12 reload (no aot) - small</td>
        <td>36.102</td>
        <td>33.669</td>
        <td>26.461</td>
        <td>19.530</td>
    </tr>
    <tr>
        <td>v12 reload - large</td>
        <td>45.140</td>
        <td>63.218</td>
        <td>17.789</td>
        <td>14.888</td>
    </tr>
    <tr>
        <td>v12 reload (no aot) - large</td>
        <td>35.907</td>
        <td>28.544</td>
        <td>16.931</td>
        <td>10.246</td>
    </tr>
</table>
