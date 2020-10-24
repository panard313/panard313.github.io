---
layout: article
title: vim plugin BookMark
key: 20180603
tags:
  - tool
  - vim
lang: zh-Hans
---

# vim bookmark插件

## Features

- Toggle bookmarks per line ⚑
- Add annotations per line ☰
- Navigate all bookmarks with quickfix window
- Bookmarks will be restored on next startup
- Bookmarks per working directory (optional)
- Fully customisable (signs, sign column, highlights, mappings)
- Integrates with Unite's quickfix source if installed
- Integrates with ctrlp.vim if installed
- Works independently from vim marks

## Installation

Before installation, please check your Vim supports signs by running :echo has('signs'). 1 means you're all set; 0 means you need to install a Vim with signs support. If you're compiling Vim yourself you need the 'big' or 'huge' feature set. MacVim supports signs.

Use your favorite plugin manager:

Vundle
Add Plugin 'MattesGroeger/vim-bookmarks' to .vimrc
Run :PluginInstall

## Usage

After installation you can directly start using it. You can do this by either using the default shortcuts or the commands:

Action	|Shortcut|	Command
-|:-:|-:
Add/remove bookmark at current line	|mm|	:BookmarkToggle
Add/edit/remove annotation at current line	|mi|	:BookmarkAnnotate <TEXT>
Jump to next bookmark in buffer	|mn|	:BookmarkNext
Jump to previous bookmark in buffer	|mp|	:BookmarkPrev
Show all bookmarks (toggle)	|ma|	:BookmarkShowAll
Clear bookmarks in current buffer only	|mc|	:BookmarkClear
Clear bookmarks in all buffers	|mx|	:BookmarkClearAll
Move up bookmark at current line	|[count]mkk|	:BookmarkMoveUp [<COUNT>]
Move down bookmark at current line	|[count]mjj|	:BookmarkMoveDown [<COUNT>]
Move bookmark at current line to another line	|[count]mg|	:BookmarkMoveToLine <LINE>
Save all bookmarks to a file	|	|:BookmarkSave <FILE_PATH>
Load bookmarks from a file	|	|:BookmarkLoad <FILE_PATH>

You can change the shortcuts as you like, just read on...
