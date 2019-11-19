---
layout: default
title: FreeNAS commands cheat sheet
parent: FreeNAS
nav_order: 9
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

## Move file into folder
Create a folder for each file based on the first part of the filename of a .pdf before the hyphen, and move the file in to the respective named folder.

```bash
root@LazyLibrarian:/mnt/LazyLibrarian # nano script.sh
#!/usr/local/bin/bash
for f in *–*.pdf; do d=${f%% –*}; mkdir -p "$d" &&  mv "$f" "$d"; done

root@LazyLibrarian:/mnt/LazyLibrarian # chmod +x script.sh
```
Execute by placing the `script.sh` file inside unsorted magazine folder, then do `./script.sh` and let it work it's magic. 

Then move these folders into your Magazine folder, and hit Library Scan.

## Is it a real hyphen?
Check if the - (above) is really a real hyphen.
```bash
root@LazyLibrarian:/mnt/LazyLibrarian # printf '–' | od -tx1 -An
          e2  80  93               
```
[http://www.ltg.ed.ac.uk/~richard/utf-8.cgi?input=–&mode=char](http://www.ltg.ed.ac.uk/~richard/utf-8.cgi?input=–&mode=char)


## Authors
Mr. Johnson
