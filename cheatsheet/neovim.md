## Neovim Cheat Sheet

- 我的nvim配置：https://github.com/Martinits/dotfiles/tree/master/.config/nvim

- 对应的快捷键cheatsheet：

| mode         | keybinding                   | action                                                  |
|--------------|------------------------------|---------------------------------------------------------|
| help         | \<C-]\>                      | jump to link                                            |
| normal       | ct                           | join two lines                                          |
|              | gJ                           | join two withoutspace                                   |
|              | U                            | line undo (act as a change, not undo)                   |
|              | * #                          | search current word                                     |
|              | i I a A o O                  | enter insert mode                                       |
|              | w b e ge                     | word jump                                               |
|              | W B E gE                     | word jump separated by blanks                           |
|              | { }                          | paragraph jump                                          |
|              | << >>                        | remove/add indent                                       |
|              | <{obj} >{obj}                | indent for object                                       |
|              | \<C-s\> \<C-e\>              | goto start end of line                                  |
|              | gj gk                        | move up/down in wrapped line                            |
|              | f F t T ; ,                  | find reverse-find to reverse-to repeat reverse-repeat   |
|              | [[ ][ [] ]]                  | jump to function start/end                              |
|              | [x ]y                        | jump to pair xy                                         |
|              | \<C-UD\>                     | scroll halfscreen                                       |
|              | '' \<C-o\> \<C-i\>(\<TAB\>)  | jump backward or forward                                |
|              | 'x mx                        | mark and jump to mark (capital for global marks)        |
|              | " q @                        | register (capital for append)                           |
|              | ~ g~ gu gU g~~ guu gUU       | case change                                             |
|              | gv                           | repeat visual select                                    |
|              | gi                           | return to insert place                                  |
|              | zz zt zb                     | move current line to middle/top/bottom of screen        |
|              | zc zo zuz                    | fold unfold update-fold                                 |
|              | zr zm zR zM                  | reduce/more fold, reduce/more all fold                  |
|              | zn zN zi                     | open/close all fold temporarily, and toggle             |
| visual       | ~ u U                        | case change                                             |
|              | \< \>                        | shift block                                             |
| command      | \<C-HL\> \<C-FB\> \<C-SE\>   | move forward/backward char/word start/end               |
|              | \<C-JK\>                     | history forward/backward                                |
|              | \<C-WU\>                     | delete word/till line start                             |
|              | \<C-D\>                      | show completion list                                    |
|              | /pattern/{num,b,e} ?         | search p                                                |
|              | {num},{num}                  | line ranger: can use: % ^ . $ + - ? / '                 |
|              | {num}:                       | equal to :.,.+{num-1}                                   |
|              | s/pattern/replace/gc         | search and replace                                      |
|              | g/pattern/command/gc         | search and run command                                  |
|              | \< \> \{num}                 | use them in pattern                                     |
|              | !                            | filter                                                  |
| insert       | \<C-A\>                      | insert again                                            |
|              | \<C-R\> x                    | paste register x                                        |
|              | \<C-O\>                      | temp normal command                                     |
|              | \<C-D\> \<C-T\>              | remove/add indent                                       |
| text obj     | \|                           | table entry                                             |
|              | %                            | pairs                                                   |
|              | f                            | function                                                |
|              | c                            | classobj                                                |
|              | g                            | git hunk                                                |
|              | s                            | statement                                               |
|              | l                            | loop                                                    |
|              | p                            | parameter                                               |
| treesitter   | \<LEADER\>sp sP              | swap next / prev parameter                              |
|              | [f [c                        | goto prev function / class start                        |
|              | [F [C                        | goto prev funciton / class end                          |
|              | ]f ]c                        | goto next function / class start                        |
|              | ]F ]C                        | goto next function / class end                          |
| marks        | [\` ]\`                      | jump to prev / next mark                                |
|              | [' ]'                        | jump to prev / next mark's linestart                    |
|              | [- ]-                        | jump to prev / next marker of the same type             |
|              | [= ]=                        | jump to prev / next marker of all type                  |
|              | m/ m?                        | show all mark / markers                                 |
|              | m, m.                        | place next available mark / place with remove first     |
|              | m- m\<SPACE\> m\<BS\>        | delete marks in current line / current buffer / markers |
| git          | \<LEADER\>gp gn              | git goto prev / next hunk                               |
|              | \<LEADER\>gs gr gu           | git hunk stage / reset / undo_stage                     |
|              | \<LEADER\>gS gR              | git buffer stage / reset                                |
|              | \<LEADER\>gv gb gB           | git hunk preview / toggle_blame / blame                 |
|              | \<LEADER\>gd gD              | git buffer diff to index / diff to HEAD                 |
|              | \<LEADER\>gt                 | git toggle show_deleted                                 |
| lsp          | \<LEADER\>rn                 | rename variable                                         |
|              | \<LEADER\>dd                 | show diagnostic loclist                                 |
|              | \<LEADER\>- =                | diagnostic prev / next                                  |
|              | \<LEADER\>[ ]                | diagnostic prev / next only error                       |
|              | gd gD gt gp gr               | goto def dec typedef imple reference                    |
|              | gH                           | hover info                                              |
|              | \<LEADER\>mt                 | format                                                  |
|              | \<LEADER\>ca                 | code action                                             |
|              | \<LEADER\>o                  | outline                                                 |
|              | \<LEADER\>dp dk              | peek definition / with lspsaga                          |
|              | \<LEADER\>lf                 | lsp symbol finder                                       |
|              | \<LEADER\>dl dc db           | show line / cursor / buffer diagnostic                  |
|              | \<LEADER\>ci co              | call hierachy: incoming / outcoming                     |
|              | \<LEADER\>i                  | toggle inlay hint                                       |
|              | \<M-x\>                      | lsp_signature: toggle                                   |
|              | \<M-j\>                      | lsp_signature: select next signature                    |
| dap          | tb tB                        | toggle breakpoint / set conditional breakpoint          |
|              | \<F4\>                       | terminate                                               |
|              | \<F5\>                       | continue                                                |
|              | \<F6\>                       | step over                                               |
|              | \<F7\>                       | step into                                               |
|              | \<F8\>                       | step out                                                |
|              | \<F9\>                       | run last                                                |
|              | \<M-v\>                      | dapui: eval                                             |
| cmp          | \<TAB\> \<S-TAB\> \<CR\>     | choose item and confirm                                 |
|              | \<C-J\> \<C-K\>              | cycle through items                                     |
|              | \<C-F\> \<F-B\>              | scroll doc                                              |
|              | \<C-SPACE\>                  | complete completion                                     |
|              | \<C-E\>                      | abort                                                   |
| bufferline   | \<LEADER\>bg bc bp           | buffer pick / pickclose / togglepin                     |
| todo         | \<LEADER\>tn tp              | goto next / prev todo                                   |
|              | \<LEADER\>ts                 | search todo with telescope                              |
| surround     | csxy                         | change surround x to y (left half with space)           |
|              | dsx                          | delete surround x                                       |
|              | ysiwx                        | add surround x                                          |
|              | Sx                           | add surround x in visual mode                           |
| substitute   | s ss S                       | substitute                                              |
|              | \<LEADER\>ss sw              | substitute range                                        |
| comment      | [count]gcc gbc gc gb         | comment.nvim: toggle comment in normal / visual         |
| terminal     | \<C-\\\>                     | toggle terminal                                         |
| move.nvim    | \<A-hjkl\>                   | move line(s) or block                                   |
| autopairs    | \<M-e\>                      | autopairs: fast wrap                                    |
| nvim-tree    | tt tf                        | nvim-tree: toggle / focus                               |
| neoclip      | \<LEADER\>y                  | yank history with telescope                             |
|              | \<C-P\> \<C-B\> \<C-Q\>      | paste / paste behind / replay                           |
| telescope    | \<LEADER\>ff fg fb fh        | telescope: find file/content/buffer/helptag             |
|              | \<LEADER\>fw fd              | telescope: cursor string / diagnostic                   |
|              | \<LEADER\>fs fS              | telescope: buffer / workspace symbols                   |
|              | gR                           | telescope: lsp references                               |
|              | \<C-j\> \<C-k\>              | telescope: move up / down                               |
|              | \<C-u\> \<C-d\>              | telescope: preview up / down                            |
|              | \<C-c\>                      | quit                                                    |
| spectre      | \<LEADER\>pp pw pf           | spectre: normal / current word / current file           |
| whichkey     | z=                           | whichkey: show spell suggestions                        |
|              | ! :WhichKey                  | show all keymaps                                        |
| treesj       | gJ gS gT                     | join / split / toggle                                   |
| hop          | \<LEADER\>hw hh              | hop word / anywhere                                     |
| flash        | \<LEADER\>jj                 | flash: jump                                             |
|              | \<LEADER\>jt                 | flash: treesitter                                       |
|              | \<C-S\>                      | toggle flash anytime                                    |
|              | r                            | operator pending: remote mode                           |
|              | R                            | treesitter search                                       |
| matchup      | % [% ]% z%                   | matchpairs and text objects                             |
| table mode   | \<LEADER\>tm                 | table mode toggle                                       |
|              | \<LEADER\>tt                 | table mode format selected lines                        |
|              | \<LEADER\>tdd tdc            | table mode delete row or line                           |
|              | \<LEADER\>tic tiC            | table mode insert column after or before                |
|              | [\| ]\| {\| }\|              | table mode move left right up down                      |
| undotree     | \<F2\>                       | undo tree toggle                                        |
| tabular      | \<LEADER\>tb                 | tabularize                                              |
| visual multi | \<C-n\> \<C-Up\> \<C-Down\>  | visual multi select                                     |
|              | n N [ ]                      | visual multi get/select next/prev                       |
|              | q Q                          | visual multi skip and get next / remove current         |
| rnvimr       | \<M-o\>                      | rnvimr toggle                                           |
|              | \<C-t\> \<C-x\> \<C-v\>      | (in rnvimr) tabedit / splitedit /vsplitedit             |
|              | gw yw                        | (in rnvimr) goto nvim cwd / emt rnvimr cwd              |
| easy align   | ga                           | easy align                                              |
| commands     | :SaveSession :RestoreSession | autosession                                             |
|              | :PeekOpen :PeekClose         | peek: markdown preview                                  |
|              | :Glow :Glow!                 | glow: markdown preview                                  |
