---
title: "Debugging wasm with VSCode"
description: "Debugging wasm with VSCode"
excerpt: "Debugging wasm inside VSCode based on WAMR's source debugging feature"
date: 2022-10-26T16:28:09+08:00
lastmod: 2022-10-26T16:28:09+08:00
draft: false
weight: 50
images: []
categories: ['debug', 'tool']
tags: ['debug']
contributors: ['Xu Jun']
pinned: false
homepage: false
---

## Debugging with VSCode

VSCode's flexible extension system make it very easy to integrate any debuggers to its UI, we have developed a VSCode extension to provide some simple project management functionality, and of cause, including the debugging feature.

1. Build the extension from source

    Please follow this [document](https://github.com/bytecodealliance/wasm-micro-runtime/tree/main/test-tools/wamr-ide) to build the extension package.

    > We are planning to automatically publish this extension by GitHub action, once finished, this step is not necessary.

2. Install the `.vsix` file from VSCode

    ![](install_vsix.png)

Now, enjoy the VSCode's debugging session! This video demonstrates the basic usage of the extension.

<video width="100%" height="auto" controls>
    <source src="debug_demo.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>
