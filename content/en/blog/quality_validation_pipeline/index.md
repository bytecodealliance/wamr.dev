---
title: "WAMR Quality validation pipeline"
description: ""
excerpt: ""
date: 2023-01-17T14:17:16+08:00
lastmod: 2023-01-17T14:17:16+08:00
draft: false
weight: 50
images: []
categories: ['quality']
tags: []
contributors: ['Xu Jun']
pinned: false
homepage: false
---

WAMR has a long pipeline of quality validation, they are basically separated to two parts.
- Public validation pipeline
	- Quick validation for most commonly used features
	- Ensure every PR will not break the basic functionality
- Internal validation pipeline
	- 7 * 24 hour available servers for full feature validation
	- Some special designed methods to cover corner cases

## Public validation pipeline

- Public tests requires:
	- fast and efficient
	- license friendly
	- cover most important features

![](public_test_pipeline.excalidraw.png)

## Internal validation pipeline

- These tests are maintained internally because they are:
	- time consuming, or never end
	- using incompatible licenses
	- compiled binaries which may not be suitable to commit to repo
	- requiring complex environment setup
	- using commercial software which require paid licenses

![](internal_test_pipeline.excalidraw.png)

