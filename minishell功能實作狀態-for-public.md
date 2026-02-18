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

- 賦值：`VAR=value`
- **陣列**：`arr=(val1 val2 val3)`；`$arr` 或 `${arr[0]}` 取首元素；`${arr[n]}` 取第 n 個；`${arr[@]}` 展開全部；`${#arr}` 取長度
- 展開：`$VAR`、`$HOME`、`$?`
- `$?` 傳回前一指令的 exit status

### 11. 路徑展開

- **`~`** / **`～`**（全形）：`~` → HOME，`~/path` → HOME + "/path"
- **`$HOME`**：環境變數 HOME 展開

### 12. 內建指令（builtins/）

- **cd**：cd、cd ~、cd $HOME、cd ~/dir、錯誤處理
- **echo**：標準 echo 行為
- **export**：`export VAR=value` 或 `export VAR`，將變數加入環境供子行程繼承；單獨 `export` 無參數時列出所有全域變數及其內容
- **env**：`env` 無參數印出所有已導出的環境變數；`env VAR=VAL command` 以臨時環境執行 command，不影響當前 Shell
- **printenv**：`printenv VAR1 VAR2` 只印出指定變數的內容（值），不顯示變數名稱；無參數印出所有
- **unset**：`unset VAR1 VAR2` 從 env_map 完全移除變數；唯讀變數（$?、$HOME）不可 unset
- **stock**：`stock 買入價 賣出價` 計算股票收益（手續費 0.1425%、交易稅 0.3%、每張 1000 股）；`stock(100,110)` 單獨指令或 `$(( stock(100,110) ))` 僅輸出收益數值
- **eval**：`eval arg1 arg2 ...` 將參數以空白連接後重新解析執行；支援二次展開、&&/||、if/while/for；遞迴深度限制 16；狀態變更影響當前 Shell
  - 單雙引號皆支援：`eval "x=1; echo $x"`、`eval 'x=1; echo $x'` → 輸出 `1`
  - `a=b; b=10; eval echo '$'$a` → 輸出 `10`

### 13. I/O 重定向

- **`>`**：輸出覆蓋寫入檔案（`O_WRONLY | O_CREAT | O_TRUNC`）
- **`>>`**：輸出追加寫入檔案（`O_WRONLY | O_CREAT | O_APPEND`）
- **`<`**：標準輸入從檔案讀取（`O_RDONLY`）
- 重定向符號可出現在命令任意位置（如 `> out ls` 或 `ls > out`）
- 檔名支援變數展開（如 `ls > $HOME/list.txt`）
- **外部命令**：使用 `dup2()` 在 fork 後、execve 前設定子行程的 stdin/stdout（`execWithRedirect`）
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
- **指令補全**：搜尋 PATH 執行檔與內建指令（cd、echo、env、eval、export、printenv、unset），唯一匹配時加空白
- **路徑補全**：搜尋目前目錄，支援 `~` 展開（如 `cd ~/Doc[TAB]`）；目錄補全後加 `/`，檔案加空白
- **變數補全**：`$H[TAB]` → `$HOME` 等
- **陣列索引補全**：`${arr[[TAB]` 列出 `${arr[0]}`, `${arr[1]}`, ... 及 `${arr[@]}`
- **多重匹配**：先補全公共前綴；無進展時以 ls 風格單行列出候選（空白分隔），最多 100 項
- **無匹配**：蜂鳴

### 17. 模組化架構

- 指令與職責拆至不同模組（見 PROJECT_STRUCTURE.md）
- 內建指令置於 `builtins/` 子目錄

---

## 二、模組對照

| 模組 | 主要功能 |
|------|----------|
| - | stripComments, splitByLogic, parseIfBlock, parseWhileBlock, parseForBlock, splitCommands, tokenizeWithQuotes |
| - | [[ ]] 求值：二元比較、-z/-n、-e/-f/-d、=~ regex、glob |
| - | Perl-style regex 比對 |
| - | REPL 主迴圈、runCommandBody、processStatements、if/while/for 塊處理、eval、break/continue |
| - | 執行單一命令（(( ))、[[ ]]、內建、衍生行程）；與 redirect 整合，重定向時調用 execWithRedirect 或 applyForBuiltin |
| - | $(( )) 與 (( )) 算術；內建 stock(buy,sell) 函式 |
| - | 變數展開、賦值、$?、parseConditionalCommand |
| - | stripRedirects、execWithRedirect（fork+dup2+execve）、applyForBuiltin、BuiltinRedirectGuard |
| - | extractWord、getCommandCompletions、getPathCompletions、getVariableCompletions、getArrayIndexCompletions、commonPrefix |
| - | readLineWithCursor、方向鍵、Backspace、Tab 補全 |
| - | 命令歷史與上下鍵導覽 |
| - | isBuiltin、cd、echo、export、env、printenv、unset、stock 分發 |

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
# 陣列
arr=(a b c); echo ${arr[0]} ${arr[1]} ${#arr}
echo ${arr[@]}

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
stock(100, 110)

# Tab 補全（須在 TTY 下）
# ec[TAB] → echo 
# echo [TAB] → 列出目前目錄（ls 風格單行）
# cd ~/Doc[TAB] → cd ~/Documents/
# $H[TAB] → $HOME
# arr=(a b c); echo ${arr[[TAB] → 列出 ${arr[0]} ${arr[1]} ${arr[2]} ${arr[@]}
```
