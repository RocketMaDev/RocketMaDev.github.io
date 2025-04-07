---
title: 从 Vim 到 NeoVim：感知不强
date: 2025/04/06 19:45:00
updated: 2025/04/07 22:23:00
tags:
    - non-ctf
excerpt: Vim原来的功能固然很强大，但是总有人说，现在是时候把Vim换成新时代的NeoVim了。话虽如此，但是要想把NeoVim配得和我现在用的Vim一样丝滑，就需要花大量的时间调配置。但当我全部配完后，却发现好像和之前配的vim差别并不大...
thumbnail: /assets/trueblog/vim2neovim.png
---

Vim原来的功能固然很强大，但是总有人说，现在是时候把Vim换成新时代的NeoVim了。
话虽如此，但是要想把NeoVim配得和我现在用的Vim一样丝滑，就需要花大量的时间调配置。
所以我迟迟没有更换的意愿，直到看见了 **Neovide** ， *dbgbgtf* 问我Vim可以丝滑滚动吗？
我拿出Neovide：不但有丝滑滚动，还有光标拖尾呢！

得益于NeoVim前后端分离，实现一个美观的编辑器前端并不是难事，甚至可以将NeoVim
集成到浏览器中。NeoVim还支持LSP等特性，有现代编辑器的各种特性。于是我决定配一下。

新编辑器需要新配置。为了适配NeoVim对lua的原生支持，我决定把所有配置迁移到lua。
一上来NeoVim的插件管理系统就比Vim复杂，我用了`Lazy.nvim`来管理插件，
首先需要学一下[spec怎么写](https://lazy.folke.io/spec)。然后是准备插件，
我的基础目标是先启用我Vim中的所有插件，如下所示：

```vim
call plug#begin('~/.vim/plugged')
Plug 'jiangmiao/auto-pairs'     " nvim-autopair.lua
Plug 'itchyny/lightline.vim'    " lualine.nvim.lua
Plug 'tpope/vim-commentary'     " nvim builtin
Plug 'tpope/vim-unimpaired'     " nvim builtin
Plug 'tpope/vim-repeat'         " nvim builtin
Plug 'tpope/vim-surround'       " nvim-surround.lua
Plug 'preservim/nerdtree', {'on': 'NERDTreeToggle'} " nvim-tree.lua
Plug 'neoclide/coc.nvim', {'branch': 'release'} " nvim-cmp.lua
Plug 'godlygeek/tabular'        " mini.align.lua
Plug 'preservim/vim-markdown'   " render-markdown.nvim.lua
Plug 'iamcco/markdown-preview.nvim', { 'do': 'cd app && yarn install' }
Plug 'kana/vim-textobj-user'    " nvim-various-textobjs.lua
Plug 'kana/vim-textobj-entire'  " nvim-various-textobjs.lua
Plug 'kana/vim-textobj-indent'  " nvim-various-textobjs.lua
Plug 'pocke/vim-textobj-markdown' " nvim-various-textobjs.lua
Plug 'cposture/vim-textobj-argument' " nvim-treesitter-textobjects.lua
Plug 'lilydjwg/fcitx.vim'       " fcitx.nvim.lua
Plug 'easymotion/vim-easymotion' " hop.nvim.lua
call plug#end()
```

除了这些vim中写过的插件，我还尝试加入dap支持，但是无论是`nvim-dap-ui`
还是`nvim-dap-view`，都会出现一些奇奇怪怪的错误，最后只好作罢。
个人觉得效果比较好的插件是`render-markdown.nvim`，在NeoVim里显示效果还是不错的。
在把全部的插件配完以后，我意识到，实际上和我原来的vim配置差距并不大...

{% notel purple fa-unlock-keyhole 在NeoVim中使用root权限保存文件 %}
在NeoVim中是无法直接使用`:w !sudo tee % > /dev/null`的，因为会显示需要一个终端，
即使加上`-S`也无法通过标准输入来输入密码。这时有一些work around可以选，
比如我就选了`pkexec`。写了以下lua代码并添加到`init.lua`中：

```lua init.lua
-- Define :WW command to save the file using pkexec and force reload
vim.api.nvim_create_user_command('WW', function()
    vim.cmd('silent! w !pkexec env DISPLAY=$DISPLAY XAUTHORITY=$XAUTHORITY tee % >/dev/null') -- Save file using pkexec
    vim.cmd('edit!')  -- Force reload the file
end, {})
```

当执行`:WW`时会唤起KDE的要求权限窗口。

<img src="/assets/trueblog/pkexec.png" width="60%">
{% endnotel %}

Anyway，我已经把[我的配置放到GitHub](https://github.com/RocketMaDev/RocketMaDev/tree/main/.config/nvim)
上了，如果想借鉴的可以到这个地址看看。接下来是配置好的NeoVim演示：

[![demo](/assets/trueblog/neovimCover.png)](https://www.bilibili.com/video/BV1rmRmYmEXb)

## 参考

1. [Plugin Spec | lazy.nvim](https://lazy.folke.io/spec)
2. [My neovim setup](https://github.com/RocketMaDev/RocketMaDev/tree/main/.config/nvim)
3. [配置一下Neovim&Neovide_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1rmRmYmEXb)
