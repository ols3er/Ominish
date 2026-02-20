# OB2D Linux 專屬 minishell -> ominish 功能實作狀態

本文件整理目前 ominish 已實作的各項功能。

---

## 一、已實作功能

### 1. 註解

- **`#`**：從 `#` 到行尾視為註解（引號內不處理）
- **`//`**：從 `//` 到行尾視為註解
- if/then/else/fi 內 condition、then_body、else_body 執行前會先 strip 註解
- 為避免 `fi` 被誤刪，if 區塊**先解析再 strip**，非 if 區塊則先 strip 再處理

### 2. 流程控制：if ... then ... else ... fi

- 支援單行：`if [[ 1 ]]; then echo ok; else echo b; fi`
- 支援多行：換行輸入，以 `> ` 提示繼續
- condition 可為任一命令（含 `[[ ]]`、`(( ))`），依 `$?` 決定分支
- `$?` 反映最後執行的指令的 exit status

### 3. 流程控制：while ... do ... done

- 支援單行：`while (( x < 3 )); do echo $x; x=$((x+1)); done`
- 支援多行：換行輸入，以 `> ` 提示繼續
- condition 可為任一命令，`$? == 0` 時執行 body 並重複
- 可與分號混合：`x=0; while (( x < 3 )); do echo $x; x=$((x+1)); done`

### 4. 流程控制：for ... in ... do ... done

- 語法：`for var in word1 word2 ...; do body; done`
- 支援單行：`for i in 1 2 3; do echo $i; done`
- 支援多行：換行輸入，以 `> ` 提示繼續
- list 支援變數展開與單引號，展開後依空白切分為多個 word
- break / continue 於 for 內同樣有效

### 5. break 與 continue

- **break**：結束最內層 while/for 迴圈
- **continue**：略過 body 剩餘命令，進入下一輪迭代
- 僅在 while/for 內有效；於 if 或一般命令出現時印出「只能在迴圈內使用」

### 6. 邏輯鏈：&& 與 ||

- **`A && B && C`**：前一個成功才執行下一個
- **`A || B || C`**：前一個失敗才執行下一個
- `&&` 與 `||` 可混合：`cmd1 && cmd2 || cmd3`
- 運算結果用 `$?` 傳回

### 6.5 管道 pipeline |

- **`cmd1 | cmd2 | cmd3`**：前一指令的 stdout 作為下一指令的 stdin
- 單一 `|` 分割，`||` 不分割（視為邏輯運算子）
- 引號內的 `|` 不分割
- `$?` 反映 pipeline 中最後一個指令的 exit status
- 與 `&&`、`||` 可混用：`echo a | cat && true`
- **輸出處理**：最後一階輸出經 pipe 導向父行程再寫入 stdout，使 TrackingWriter 可追蹤最後位元組；若輸出未以換行結束（如 `echo hello | head -c 3`），下一個 prompt 前會自動補換行，避免與多行 prompt（┌─...）混在一起

### 7. 分號分隔命令

- `;` 分隔多個命令，依序執行
- 單引號內的分號不視為分隔符

### 8. 條件判斷 [[ ]]

#### 8.1 二元比較

- **運算符**：`=`, `==`, `!=`, `<`, `>`, `<=`, `>=`
- `==` 與 `=` 對字串作用相同；`==` 右側無引號時可做 glob 樣式比對
- 兩邊皆為整數時走數值比較；否則為字串比較
- 引號內字串會正確去除外層引號再比較
- **尚未實作**：`-eq`, `-ne`, `-lt`, `-gt` 等 test 風格運算子

#### 8.2 字串判斷

- `[[ -z 字串A ]]`：空字串為真
- `[[ -n 字串A ]]`：非空字串為真

#### 8.3 檔案條件

- `[[ -e 檔案 ]]`：存在
- `[[ -f 檔案 ]]`：一般檔
- `[[ -d 檔案 ]]`：目錄

#### 8.4 正則比對

- `[[ $a =~ regex ]]`：Perl 風格 regex
- 支援：`.` `*` `+` `?` `[abc]` `[^abc]` `[a-z]` `^` `$` `\`

### 9. 算術運算

- **`$(( expr ))`**：展開為計算結果
- **`(( expr ))`**：作為條件，0→`$?`=0，非 0→`$?`=1
- 支援變數如 `x=5; echo $(( x++ )); echo $x`
- **浮點運算**：支援小數，如 `echo $((3.5+7.88))` → `11.38`；整數結果不帶小數點，如 `echo $((3+7))` → `10`
- **科學記號**：支援 `1.2e3`、`2.5E-4` 等形式，如 `echo $((1.2e3+PI))` → `1203.1415926536`
- **內建常量**：`PI` / `pi` 恆為 `3.141592653589793`，如 `echo $((2*PI))` → `6.283185307179586`
- **數學函式**：`sin()`, `cos()`, `tan()`, `sec()`, `csc()`, `cot()`, `asin()`, `acos()`, `atan()`, `sqrt()`, `pow(x,y)`；`sec`/`csc`/`cot`/`asin`/`acos` 定義域錯誤時輸出 `nan`
- 原生支援：浮點運算
- 實作方式：沒有呼叫 bc、awk 等外部程式。
- 型別與運算：使用內建 f64 與 std.math（pow、mod 等），運算在行程內完成。
- 硬體支援：f64 對應 IEEE 754 雙精度浮點數，由 CPU 直接執行浮點運算。
- 精度到小數點後 15 位。
- 實作科學記號支援：例如支援 echo $(( 1.2e3 + PI )) 這種語法。
- 實作 sin,cos,tan,pow,sqrt
- echo $((sqrt(-1))) 或 echo $((1/0)) 回應為 nan

### 10. 變數與環境

- **引號與參數分割**：`tokenizeWithQuotes` 在未加引號的 token 遇到空白即結束，不把後續引號段併入同一 token，故 `echo "$str"`、`printf "%s" "$str"` 等會正確得到兩個參數並展開；執行前以獨立 arena 複製 token，避免與 lexer 共用 allocator 導致引號內參數被覆蓋。
- 賦值：`VAR=value`
- **SHELL**：進入 ominish 時自動設為執行檔的絕對路徑（經 `std.fs.selfExePathAlloc` 取得）
- **EUID**：進入時自動設為 `geteuid()` 數值，供 `${EUID}` 與 `[[ ${EUID} == 0 ]]`（是否 root）使用
- **陣列**：`arr=(val1 val2 val3)`；`$arr` 或 `${arr[0]}` 取首元素；`${arr[n]}` 取第 n 個；`${arr[@]}` 展開全部；`${#arr}` 取長度
- **陣列＋命令替換**：`files=($(ls /tmp))`、`items=($(echo a b c))` 等可正確解析；tokenizer 與 parseArrayAssignment 皆將 `$(...)` 視為一整組，不在命令內空白處分割
- **陣列追加**：`arr+=(val4 val5)` 將新元素追加到既有陣列
- **關聯陣列**：`declare -A map` 宣告；`map[key]=value` 賦值；`${map[key]}` 取值；`${map[@]}` 展開所有值；`${#map}` 取元素個數
- 展開：`$VAR`、`$HOME`、`$?`
- **`$(...)` 命令替換**：執行子 shell，以其 stdout 作為替換結果；換行會變成空白；支援巢狀與多次替換，例如 `$(whoami)`、`$(echo a) $(echo b)`、`$(echo $(whoami))`
- `$?` 傳回前一指令的 exit status

### 11. 路徑展開與 Globbing

- **`~`** / **`～`**（全形）：`~` → HOME，`~/path` → HOME + "/path"
- **`$HOME`**：環境變數 HOME 展開
- **Globbing**：`*`、`?`、`[abc]`、`[^abc]`、`[a-z]` 萬用字元展開；`ls /tmp/s*`、`echo *.zig`、`ls /tmp/systemd-private*` 會展開為符合的檔名；無匹配時保留原字串；僅展開當前目錄層級（不遞迴）

### 12. 內建指令（builtins/）

- **cd**：cd、cd ~、cd $HOME、cd ~/dir、錯誤處理
- **clear**：清除終端畫面（送出 `\x1b[2J\x1b[H` ANSI 序列）
- **echo**：標準 echo 行為
- **pwd**：印出目前工作目錄，優先使用 PWD（邏輯路徑、保留 symlink），與提示 `\w` 一致
- **export**：`export VAR=value` 或 `export VAR`，將變數加入環境供子行程繼承；單獨 `export` 無參數時列出所有全域變數及其內容
- **env**：`env` 無參數印出所有已導出的環境變數；`env VAR=VAL command` 以臨時環境執行 command，不影響當前 Shell
- **printenv**：`printenv VAR1 VAR2` 只印出指定變數的內容（值），不顯示變數名稱；無參數印出所有
- **unset**：`unset VAR1 VAR2` 從 env_map 完全移除變數；唯讀變數（$?、$HOME）不可 unset
- **alias**：`alias` 列出所有別名；`alias name` 顯示指定別名；`alias name='value'` 設定別名；執行命令時第一個詞會先行別名展開（先嘗試將整行以展開結果重新解析執行；若直接 exec 失敗則改由 `/bin/sh -c` 執行，且傳給 sh 的指令字串會先做別名展開）
- **unalias**：`unalias name` 移除別名；`unalias -a` 移除全部
- 別名以 `__alias__name` 為 key 存於 env_map；`alias ll='ls -l'` 後執行 `ll /tmp` 會以 `ls -l /tmp` 之行為執行並正常列出目錄
- **別名查表前 WTF-8 檢查**：`getAlias` 在呼叫 `env_map.get(key)` 前會以 `std.unicode.wtf8ValidateSlice(name)` 檢查第一個詞是否為合法 WTF-8；若否（例如在 prompt 貼上 emoji 後用 Backspace 刪到不完整 UTF-8 位元組），直接回傳 null、不查表，避免 `env_map.get` 因 key 非法而 panic，該輸入改當一般指令處理。
- **declare**：`declare -A name` 宣告關聯陣列
- **stock**：`stock 買入價 賣出價` 計算股票收益（手續費 0.1425%、交易稅 0.3%、每張 1000 股）；`stock(100,110)` 單獨指令或 `$(( stock(100,110) ))` 僅輸出收益數值
- **help**：`help` 列出所有內建指令與簡短用法；`help 指令名` 只顯示該指令；無此內建時印出「無此內建指令」
- **printf**：支援 %s %d %i %u %x %X %o %f %e %E %g %G %c %% %b %q %Q 等；**width**：%d/%i/%u/%x/%X/%o/%f 支援左側補空白（如 `printf "%10d" 123456` → `    123456`）；**%f** 支援 precision（如 `%4.2f`）；**%e/%E** 科學記號（%E 大寫）；格式字串無 `\n` 時不自動換行（與 bash 一致）；`%%` 搭配多餘參數時不重複輸出、不無限迴圈；%q 可做 shell 可重用輸出
- **eval**：`eval arg1 arg2 ...` 將參數以空白連接後重新解析執行；支援二次展開、&&/||、if/while/for；遞迴深度限制 16；狀態變更影響當前 Shell
- **source** / **`.`**：`source 路徑` 或 `. 路徑` 於當前 shell 讀取並執行腳本或 init 設定檔（如 `source ~/.ominishrc`、`. ~/.ominishrc`）；支援 ~ 與變數展開
  - 單雙引號皆支援：`eval "x=1; echo $x"`、`eval 'x=1; echo $x'` → 輸出 `1`
  - `a=b; b=10; eval echo '$'$a` → 輸出 `10`
  - exit：exit [n] 結束 shell，n 省略則用 $?

### 13. I/O 重定向

- **`>`**：輸出覆蓋寫入檔案（`O_WRONLY | O_CREAT | O_TRUNC`）
- **`>>`**：輸出追加寫入檔案（`O_WRONLY | O_CREAT | O_APPEND`）
- **`<`**：標準輸入從檔案讀取（`O_RDONLY`）
- 重定向符號可出現在命令任意位置（如 `> out ls` 或 `ls > out`）
- 檔名支援變數展開（如 `ls > $HOME/list.txt`）
- **外部命令**：使用 `dup2()` 在 fork 後、execve 前設定子行程的 stdin/stdout（`execWithRedirect`）；先嘗試 `execveZ`（系統 environ），失敗再試 `execvpeZ`，若仍失敗則改以 `/bin/sh -c '指令'` 執行，且傳給 sh 的指令字串會依 env_map 做別名展開（例如 `ll /tmp` → `sh -c "ls -l /tmp"`），確保 alias 後的外部命令可正常輸出
- **內建指令**：使用 `dup()` 保存原 fd、`dup2()` 套用重定向、執行內建、`restore()` 恢復（`applyForBuiltin` + `BuiltinRedirectGuard`）
- 新建檔案權限 0644；重定向失敗時 `$?` 為非 0
- **注意**：`>`、`>>`、`<` 與檔名之間需有空白（例如 `ls > out`）；`ls>out` 尚未支援
- **內建指令支援重定向**：`echo hello > out`、`echo x >> log`、`printenv VAR > env.txt`、`export VAR=val > env.txt` 等皆可

### 14. 行編輯與歷史

- **Raw 模式**：關閉 ICANON、ECHO，逐字讀取
- **左右方向鍵**：ESC [ C / ESC [ D，游標移動
- **Backspace / DEL**：刪除游標前一字元
- **上下鍵**：歷史導覽（getPrevious / getNext）
- 非 TTY 時自動改用 readUntilDelimiterOrEof

### 15. Backspace 處理

- 僅在 move_left > 0 時送出 `\x1b[{n}D`
- termios 僅關閉 ICANON 與 ECHO
- just_backspace 保留以略過 BS+SP 序列中夾帶的 space

### 16. Tab 自動補全

- **觸發**：在 readLineWithCursor 循環中捕獲 `\t`（Tab）
- **上下文識別**：自游標向左搜尋空白，提取半成品單詞；行首補全指令，非行首補全路徑，`$` 開頭補全變數，`${arr[` 補全陣列索引
- **指令補全**：搜尋 PATH 執行檔與內建指令（cd、echo、env、eval、export、help、printenv、printf、unset、alias、unalias 等），唯一匹配時加空白
- **路徑補全**：搜尋目前目錄，支援 `~` 展開（如 `cd ~/Doc[TAB]`）；目錄補全後加 `/`，檔案加空白
- **變數補全**：`$H[TAB]` → `$HOME` 等
- **陣列索引補全**：`${arr[[TAB]` 列出 `${arr[0]}`, `${arr[1]}`, ... 及 `${arr[@]}`
- **多重匹配**：先補全公共前綴；無進展時以 ls 風格單行列出候選（空白分隔），最多 100 項
- **無匹配**：蜂鳴

### 17. REPL 輸出與 prompt 顯示

- **格式字串無 \\n 不自動換行**：與 bash 一致，若命令輸出未以換行結束（如 `printf "%e" 123`），下一個 prompt 緊接在輸出後、不自動補換行

### 18. PS1 提示與 ANSI 顯示

- **PS1**：從環境變數 `PS1` 讀取，未設定則預設 `ominish> `
- **跳脫序列**：`\u` 使用者名、`\h` 主機名（短）、`\H` 主機名（完整）、`\w` 當前目錄（含 ~）、`\W` 目錄 basename、`\$` 提示符（root 為 #）、`\n` 換行、`\?` 上一個 exit status、`\\` 反斜線、`\e`/`\033` ESC、`\nnn` 八進位字元
- **`\[...\]`**：非列印區塊，內容可含 ANSI 色碼（如 `\[\033[0;31m\]` 紅色），會遞迴展開內部跳脫
- **ANSI 色碼**：支援 `\e[0;31m`、`\033[01;33m` 等標準序列
- **PS1 內 `$(...)` 命令替換**：若提供 run_command 與 env_arena，getPrompt 會對 expandPrompt 結果再呼叫 expandTokenUntilStable，支援如 `$(whoami)> `、`$([[ $? != 0 ]] && echo "[✗]")` 等
- **PS1 賦值引號**：設定時值首尾引號必須一致（`'...'` 或 `"..."`），若寫成 `'..."` 會解析錯誤、提示不生效。
- **邏輯路徑顯示**：提示中的 `\w`、`\W` 優先使用環境變數 PWD（symlink 目錄顯示為實際路徑，如 `~/MYBUILD/Ominish` 而非解析後的 `/BK-.../ZIG`）；`cd` 成功後會更新 PWD

### 19. 啟動設定檔與互動／非互動模式

- **~/.ominish_profile**：進入 ominish 時先讀取並執行（類似 .bash_profile / .profile）
- **~/.ominishrc**：繼而讀取並執行（類似 .bashrc）
- 兩檔皆為選用；換行視同分號，支援變數展開、內建指令、if/while/for 等
- **引號需成對**：賦值時若值以單引號 `'` 開頭，結尾也必須是單引號；若以雙引號 `"` 開頭，結尾也必須是雙引號。首尾引號不一致（例如 `PS1='...'` 誤寫成 `PS1='..."`）會導致解析錯誤、變數未正確設定。
- **僅互動模式顯示橫幅**：「Loaded .ominish_profile」「Loaded .ominishrc」與「OB2D Linux minishell」僅在 **stdin 為 TTY（互動模式）** 時顯示，且每 process 只顯示一次；管線或重定向 stdin（如 `echo 'echo test' | ./ominish`）時不顯示上述橫幅。
- **非互動模式不顯示 prompt**：stdin 非 TTY 時不輸出 prompt（多行橫幅與 `└╼$>` 等），僅輸出命令結果；且空 prompt 時不進行「從輸入剝除 prompt」的處理，避免無限迴圈並正確在 EOF 結束。

### 20. 模組化架構

- 指令與職責拆至不同模組（見 PROJECT_STRUCTURE.md）
- 內建指令置於 `builtins/` 子目錄

### 21. 單元測試

- **執行方式**：`zig build test`（與 `zig build run` 相同，需在本機安裝 Zig）
- **機制**：使用 Zig 內建 test runner，以 `src/` 為根編譯測試執行檔，會自動收集並執行所有依賴模組內的 `test "描述" { ... }` 區塊
- **範例**：`` 內有 `isValidVarName`、`findAssignmentEq` 的測試；`` 內有 `splitCommands` 的測試（含引號內不分割 `;`）
- **撰寫**：在任意 `.zig` 中加上 `test "名稱" { try std.testing.expect(...); }` 即可，該模組被 main 依賴鏈引用時，其測試會被一併執行

---

## 二、模組對照

| 模組 | 主要功能 |
|------|----------|
| - | stripComments, splitByLogic, splitByPipeline, parseIfBlock, parseWhileBlock, parseForBlock, splitCommands, tokenizeWithQuotes（陣列賦值延伸時跳過 `$(...)`） |
| - | expandGlob、expandArgsGlob；萬用字元 * ? [abc] 比對與展開 |
| - | [[ ]] 求值：二元比較、-z/-n、-e/-f/-d、=~ regex、glob |
| - | Perl-style regex 比對 |
| - | REPL 主迴圈、TrackingWriter（追蹤輸出最後位元組）、runCommandBody、runPipeline、processStatements、if/while/for 塊處理、eval、break/continue；執行前對參數做 glob 展開 |
| - | 執行單一命令（(( ))、[[ ]]、內建、衍生行程）；與 redirect 整合，重定向時調用 execWithRedirect 或 applyForBuiltin |
| - | $(( )) 與 (( )) 算術；內建 stock(buy,sell) 函式 |
| - | 變數展開、賦值、$?、parseConditionalCommand、isAssocArrayAssignment、isArrayAppend、applyAssocArrayAssignment、appendArrayAssignment；parseArrayAssignment/parseArrayAppend 將 `$(...)` 視為單一值 |
| - | stripRedirects、execWithRedirect（fork+dup2+execve；失敗時 fallback 為 sh -c + 別名展開）、applyForBuiltin、BuiltinRedirectGuard |
| - | extractWord、getCommandCompletions、getPathCompletions、getVariableCompletions、getArrayIndexCompletions、commonPrefix |
| - | readLineWithCursor、方向鍵、Backspace、Tab 補全 |
| - | 命令歷史與上下鍵導覽 |
| - | isBuiltin、cd、echo、export、env、printenv、unset、declare、alias、unalias、stock 分發 |
| - | expandPrompt、getPrompt；PS1 跳脫序列 \u \h \w \$ \n \e \[ \] 等 |

---

## 三、測試範例（快速驗證）

```bash
# 註解
echo a # 註解
echo b  // 另一種註解

# if then else
if [[ 1 = 1 ]]; then echo ok; fi
if false; then echo a; else echo b; #test fi

# while loop 與 break/continue
x=0; while (( x < 3 )); do echo $x; x=$((x+1)); done
while [[ 1 = 1 ]]; do echo loop; break; done

# for loop
for i in 1 2 3; do echo $i; done
for f in a b c; do echo "file: $f"; done

# && 與 ||
true && echo a && echo b
false || echo ok

# globbing
ls /tmp/s*
echo *.zig
ls /tmp/systemd-private*

# pipeline
echo hello | cat
echo a | cat | cat
echo hello | head -c 3   # 輸出 hel 未以換行結束，下個 prompt 前會自動補換行
export | grep SHELL     # 輸出以換行結束，不補多餘換行

# 條件與算術
[[ -z "" ]]; echo $?
x=5; echo $(( x++ )); echo $x
echo $((3.5+7.88))
echo $((2*PI))
echo $((sin(PI/2)))

# cd 與路徑
cd ~; echo $HOME; ls ~/

# export 與 env
export MYVAR=hello; echo $MYVAR
export
env
env MYVAR=world echo $MYVAR
echo $MYVAR
printenv MYVAR HOME
printenv
unset MYVAR; echo "MYVAR=$MYVAR"
unset '?'   # 唯讀，會印出錯誤
unset 'HOME'   # 唯讀，會印出錯誤

# 別名（設定後執行會展開為對應指令；若直接 exec 失敗則經 sh -c 執行並做別名展開）
alias ll='ls -l'
ll /tmp                    # 等同 ls -l /tmp，會列出 /tmp 目錄
alias lll='ls -la'
lll /tmp                   # 等同 ls -la /tmp
alias
alias ll
unalias ll
alias

# 陣列
arr=(a b c); echo ${arr[0]} ${arr[1]} ${#arr}
echo ${arr[@]}
arr+=(d e); echo ${arr[@]}
files=($(ls /tmp)); echo ${#files}; echo ${files[@]}

# 關聯陣列
declare -A map; map[a]=1; map[b]=2; echo ${map[a]} ${map[b]}
echo ${map[@]}

eval echo hello
eval "cd .."
echo $((PI))
echo $((2*pi))
echo $((PI*2))
echo $((1.2e3+PI))    # 1200 + 3.14... ≈ 1203.14
echo $((2.5E-4*1e4))  # 0.00025 * 10000 = 2.5

# 正則
[[ abc =~ a.* ]]; echo $?

# I/O 重定向（外部命令與內建指令皆支援）
echo hello > /tmp/test_out
echo append >> /tmp/test_out
cat < /tmp/test_out
ls > /tmp/ls_out; wc -l < /tmp/ls_out
export X=1; printenv X > /tmp/env_out

# PS1 提示與 ANSI
export PS1='\u@\h:\w\$ '
# PS1='\[\033[0;31m\]\u\[\033[0m\]\$ '   # 紅色使用者名
# PS1='\u:\W\$ '   # 使用者:basename$

stock(100, 110)
stock 100 110
prices=(110 115 120)
for p in ${prices[@]}; do
  echo "當賣出價為 $p 時："
  stock 100 $p
done
當賣出價為 110 時：
買入手續費: 142.5000
賣出手續費: 156.7500
賣出交易稅: 330.0000
成本合計: 629.2500
股票收益: 9370.7500
當賣出價為 115 時：
買入手續費: 142.5000
賣出手續費: 163.8750
賣出交易稅: 345.0000
成本合計: 651.3750
股票收益: 14348.6250
當賣出價為 120 時：
買入手續費: 142.5000
賣出手續費: 171.0000
賣出交易稅: 360.0000
成本合計: 673.5000
股票收益: 19326.5000

# Tab 補全（須在 TTY 下）
# ec[TAB] → echo 
# echo [TAB] → 列出目前目錄（ls 風格單行）
# cd ~/Doc[TAB] → cd ~/Documents/
# $H[TAB] → $HOME
# arr=(a b c); echo ${arr[[TAB] → 列出 ${arr[0]} ${arr[1]} ${arr[2]} ${arr[@]}
```
