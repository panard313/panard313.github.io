---
layout: article
title: shell skills
key: 20181217
tags:
  - shell
lang: zh-Hans
---

# 批处理带空格的文件名
    SAVEIFS=$IFS;IFS=$(echo -en "\n\b");for i in ```ls ~/Music/|tail -n 100`;do adb push $i /sdcard/Music/;echo done $i;done
