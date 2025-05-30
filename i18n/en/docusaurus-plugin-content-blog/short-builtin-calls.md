---
title: "Short builtin calls"
author: "[Toon Verwaest](https://twitter.com/tverwaes), The Big Short"
avatars: 
  - toon-verwaest
date: 2021-05-06
tags: 
  - JavaScript
description: "In V8 v9.1 we’ve temporarily unembedded the builtins on desktop to avoid performance issues resulting from far indirect calls."
tweet: "1394267917013897216"
---

In V8 v9.1 we’ve temporarily disabled [embedded builtins](https://v8.dev/blog/embedded-builtins) on desktop. While embedding builtins significantly improves memory usage, we’ve realized that function calls between embedded builtins and JIT compiled code can come at a considerable performance penalty. This cost depends on the microarchitecture of the CPU. In this post we’ll explain why this is happening, what the performance looks like, and what we’re planning to do to resolve this long-term.

<!--truncate-->
## Code allocation

Machine code generated by V8’s just-in-time (JIT) compilers is allocated dynamically on memory pages owned by the VM. V8 allocates memory pages within a contiguous address space region, which itself either lies somewhere randomly in memory (for [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) reasons), or somewhere inside of the 4-GiB virtual memory cage we allocate for [pointer compression](https://v8.dev/blog/pointer-compression).

V8 JIT code very commonly calls into builtins. Builtins are essentially snippets of machine code that are shipped as part of the VM. There are builtins that implement full JavaScript standard library functions, such as [`Function.prototype.bind`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_objects/Function/bind), but many builtins are helper snippets of machine code that fill in the gap between the higher-level semantics of JS and the low-level capabilities of the CPU. For example, if a JavaScript function wants to call another JavaScript function, it is common for the function implementation to call a `CallFunction` builtin that figures out how the target JavaScript function should be called; i.e., whether it’s a proxy or a regular function, how many arguments it expects, etc. Since these snippets are known when we build the VM, they are "embedded" in the Chrome binary, which means that they end up within the Chrome binary code region.

## Direct vs. indirect calls

On 64-bit architectures, the Chrome binary, which includes these builtins, lies arbitrarily far away from JIT code. With the [x86-64](https://en.wikipedia.org/wiki/X86-64) instruction set, this means we can’t use direct calls: they take a 32-bit signed immediate that’s used as an offset to the address of the call, and the target may be more than 2 GiB away. Instead, we need to rely on indirect calls through a register or memory operand. Such calls rely more heavily on prediction since it’s not immediately apparent from the call instruction itself what the target of the call is. On [ARM64](https://en.wikipedia.org/wiki/AArch64) we can’t use direct calls at all since the range is limited to 128 MiB. This means that in both cases we rely on the accuracy of the CPU's indirect branch predictor.

## Indirect branch prediction limitations

When targeting x86-64 it would be nice to rely on direct calls. It should reduce strain on the indirect branch predictor as the target is known after the instruction is decoded, but it also doesn't require the target to be loaded into a register from a constant or memory. But it's not just the obvious differences visible in the machine code.

Due to [Spectre v2](https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html) various device/OS combinations have turned off indirect branch prediction. This means that on such configurations we’ll get very costly stalls on function calls from JIT code that rely on the `CallFunction` builtin.

More importantly, even though 64-bit instruction set architectures (the “high-level language of the CPU”) support indirect calls to far addresses, the microarchitecture is free to implement optimisations with arbitrary limitations. It appears common for indirect branch predictors to presume that call distances do not exceed a certain distance (e.g., 4GiB), requiring less memory per prediction. E.g., the [Intel Optimization Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-optimization-manual.pdf) explicitly states:

> For 64-bit applications, branch prediction performance can be negatively impacted when the target of a branch is more than 4 GB away from the branch.

While on ARM64 the architectural call range for direct calls is limited to 128 MiB, it turns out that [Apple’s M1](https://en.wikipedia.org/wiki/Apple_M1) chip has the same microarchitectural 4 GiB range limitation for indirect call prediction. Indirect calls to a call target further away than 4 GiB always seem to be mispredicted. Due to the particularly large [re-order buffer](https://en.wikipedia.org/wiki/Re-order_buffer) of the M1, the component of the CPU that enables future predicted instructions to be executed speculatively out-of-order, frequent misprediction results in an exceptionally large performance penalty.

## Temporary solution: copy the builtins

To avoid the cost of frequent mispredictions, and to avoid unnecessarily relying on branch prediction where possible on x86-64, we’ve decided to temporarily copy the builtins into V8's pointer compression cage on desktop machines with enough memory. This puts the copied builtin code close to dynamically generated code. The performance results heavily depend on the device configuration, but here are some results from our performance bots:

![Browsing benchmarks recorded from live pages](/_img/short-builtin-calls/v8-browsing.svg)

![Benchmark score improvement](/_img/short-builtin-calls/benchmarks.svg)

Unembedding builtins does increase memory usage on affected devices by 1.2 to 1.4 MiB per V8 instance. As a better long-term solution we’re looking into allocating JIT code closer to the Chrome binary. That way we can re-embed the builtins to regain the memory benefits, while additionally improving the performance of calls from V8-generated code to C++ code.
