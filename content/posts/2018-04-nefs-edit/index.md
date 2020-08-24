---
title: "NeFS Edit"
date: 2018-04-07T10:21:23-05:00
draft: false
---

## Overview

The NeFS archive file format has been partially reversed engineered. The NeFS format is used in various games developed by Codemasters (such as [DiRT 4](https://www.dirt4game.com/)) using their proprietary [EGO game engine](https://en.wikipedia.org/wiki/EGO_(game_engine)).

The NeFS Edit tool was created to extract files from and modify these archive files. The tool has limitations and has issues with certain archives. Read the readme for more information. Binaries and source code provided as-is with no support.

- [NeFS Edit downloads](https://github.com/victorbush/ego.nefsedit/releases)
- [NeFS Edit source](https://github.com/victorbush/ego.nefsedit)
- [NeFS file format documentation](https://github.com/victorbush/ego.nefsedit/wiki)

## Custom DiRT 4 Liveries
If you just want custom liveries, it is now possible to do this without editing NeFS files.

Copy the PSSG files to the following paths within the game’s data directory, creating the subdirectories as required:

```
cars/[car]/livery_[livery]/textures_high/[car]_tex_high_[livery].pssg
cars/[car]/livery_[livery]/textures_low/[car]_tex_low_[livery].pssg
```

Source: https://twitter.com/Kick_Up/status/968865726340714498