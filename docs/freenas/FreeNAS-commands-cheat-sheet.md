---
layout: default
title: FreeNAS commands cheat sheet
parent: FreeNAS
nav_order: 2
---

# FreeNAS commands cheat sheet
{: .no_toc }
This are useful commands I have found managing FreeNAS.

## Table of contents
{: .no_toc .text-delta }
1. TOC
{:toc}
---

## Find

| Command  | Explanation |
| ------------- | ------------- |
| `find /mnt/directory1/ -name "Absurd Superpowers*" -type f -exec mv {} /mnt/directory1/moved \;`  | Find files located in `/mnt/directory1` named `"Absurd Superpowers*"` and move those files to `/mnt/directory1/moved` |
| `find /mnt/directory1/ | grep Pool | xargs -I@ mv @ mnt/directory1/moved/`  | Find files located in `/mnt/directory1` named `"Pool"` and move those files to `/mnt/directory1/moved`  |
| `find . -type f -size -3M -delete` | Find and delete files located in current directory that is under 3MB in size |

## Replacement
Replace volume name, v, with a 0.
```bash
#!/bin/sh
for f in *v*; do mv -v "$f" "$(echo "$f" | tr 'v' '0')"; done
```
