---
title: Using Neovim as an IDE for Python
date: 2021-07-24
tags: ['neovim','vim','python','ide','lsp']
categories: ['tutorial']
author: David Spencer
---

In this post I'll demonstrate my setup of neovim for python development.

### Setup

- [Install neovim](https://github.com/neovim/neovim)
- [Install Python](https://www.python.org/downloads/)

---

### neovim configuration

All of the config files for neovim will be in our config directory.
If any of the folders or files do not exist we will create them now.

```bash
mkdir -p ~/.config/nvim/lua
touch ~/.config/nvim/init.vim
touch ~/.config/nvim/lua/{lua_config,lsp_config}.lua
```

Using neovim, we will edit the init.vim

```vim
syntax on                       "syntax highlighting, see :help syntax
filetype plugin indent on       "file type detection, see :help filetype
set number                      "display line number
set relativenumber              "display relative line numbers
set path+=**                    "improves searching, see :help path
set noswapfile                  "disable use of swap files
set wildmenu                    "completion menu
set backspace=indent,eol,start  "ensure proper backspace functionality
set undodir=~/.cache/nvim/undo  "undo ability will persist after exiting file
set undofile                    "see :help undodir and :help undofile
set incsearch                   "see results while search is being typed, see :help incsearch
set smartindent                 "auto indent on new lines, see :help smartindent
set ic                          "ignore case when searching
set colorcolumn=80              "display color when line reaches pep8 standards
set expandtab                   "expanding tab to spaces
set tabstop=4                   "setting tab to 4 columns
set shiftwidth=4                "setting tab to 4 columns
set softtabstop=4               "setting tab to 4 columns
set showmatch                   "display matching bracket or parenthesis
set hlsearch incsearch          "highlight all pervious search pattern with incsearch

highlight ColorColumn ctermbg=9 "display ugly bright red bar at color column number

" Keybind Ctrl+l to clear search
nnoremap <C-l> :nohl<CR><C-l>:echo "Search Cleared"<CR>

" When python filetype is detected, F5 can be used to execute script 
autocmd FileType python nnoremap <buffer> <F5> :w<cr>:exec '!clear'<cr>:exec '!python3' shellescape(expand('%:p'), 1)<cr>

```

I set colorcolumn to 80 instead of 79 due to preferring the red wall, this way I can type up to the wall instead of having one character on the wall.

_ref: [pep8](https://neovim.io/doc/lsp/)_

---

### Installing vim-plug and LSP Plugins

vim-plug is my plugin manager of choice.
If you use a different plugin manager you can do so.

[vim-plug neovim install](https://github.com/junegunn/vim-plug#neovim)

There are two plugins we will need

- [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig#install)
- [completion-nvim](https://github.com/nvim-lua/completion-nvim#install)

To install these using vim-plug we will add the following section to our init.vim

```vim
"vim-plug configuration, plugins will be installed in ~/.config/nvim/plugged
call plug#begin('$XDG_CONFIG_HOME/nvim/plugged') 

Plug 'neovim/nvim-lspconfig'
Plug 'nvim-lua/completion-nvim'

call plug#end()

```

Install plugins

```vim
nvim
:PlugInstall
```

---

### Configuring neovim Language Server Protocol (lsp)

We have a pretty good setup at this point. But to make it better we can add a Language Server using the Language Server Client built into neovim

You can read all about it here. [neovim lsp](https://neovim.io/doc/lsp/)

Essentially each programming language has it's own language server that needs installed.
The nvim-lspconfig plugin we installed earlier ensures language servers are launched appropriately when needed. However, for this to work we have to install language servers ourselves.

There are other projects that aim to automate the installation of language servers; however, I have not tested them. _[automatic language server installation](https://github.com/neovim/nvim-lspconfig/wiki/Installing-language-servers-automatically)_

I will continue with the manual installation of a language server.

The nvim-lspconfig repo has an exhaustive list of language server options. I recommend using this when looking for a language server to use.

For python, I use [pylsp](https://github.com/neovim/nvim-lspconfig/blob/master/CONFIG.md#pylsp).

```bash
pip3 install -U setuptools pip
pip3 install 'python-lsp-server[all]'
```

With that installed, we can now configure neovim so that it will load pylsp when we are working on a python project.

Earlier we created some files

- ~/.config/nvim/lua/lsp_config.lua
- ~/.config/nvim/lua/lua_config.lua

Lets edit lua_config.lua first. Add the following to the lua_config.lua file.

```lua
require('lsp_config')
```

Add the following to the lsp_config.lua file

```lua
local lsp = require('lspconfig')
local completion = require('completion')

local custom_attach = function()
    completion.on_attach()
    -- Python specifically isn't setting omnifunc correctly, ftplugin conflict
    vim.api.nvim_buf_set_option(0, 'omnifunc', 'v:lua.vim.lsp.omnifunc')
end

lsp.pylsp.setup{on_attach=custom_attach}
```

The lsp_config.lua file contains a couple functions to make loading multiple language servers a bit cleaner.

If you want to add any additional language servers you can add a line to the bottom using the following template.

```lua
lsp.<language_server>.setup{on_attach=custom_attach}
```

The final step will be to add a configuration to our init.vim

Open ~/.config/nvim/init.vim and add the following lines.

```vim
" neovim LSP Configuration
lua require('lua_config')

" Enable Tab / Shift Tab to cycle completion options
inoremap <expr> <Tab>   pumvisible() ? "\<C-n>" : "\<Tab>"
inoremap <expr> <S-Tab> pumvisible() ? "\<C-p>" : "\<S-Tab>"
let g:completion_enable_fuzzy_match = 1
set completeopt=menuone,noinsert,noselect
```

---

### Conclusion

You should now be able to open any python file without any errors.

Additionally, you should have auto-complete and linting enabled.

You should be able to navigate between auto-complete options using Tab / Shift Tab. Or natively Ctrl+n or Ctrl+p.

![neovim as ide for python with lsp](/nvim_python_ide.gif#center)
