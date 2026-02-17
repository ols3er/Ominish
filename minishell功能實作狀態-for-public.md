# minishell 功能實作狀態

本文件整理目前 ominish minishell 已實作的各項功能。

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

### 4. break 與 continue

- **break**：結束最內層 while 迴圈
- **continue**：略過 body 剩餘命令，進入下一輪迭代
- 僅在 while 內有效；於 if 或一般命令出現時印出「只能在迴圈內使用」

### 5. 邏輯鏈：&& 與 ||

- **`A && B && C`**：前一個成功才執行下一個
- **`A || B || C`**：前一個失敗才執行下一個
- `&&` 與 `||` 可混合：`cmd1 && cmd2 || cmd3`
- 運算結果用 `$?` 傳回

### 6. 分號分隔命令

- `;` 分隔多個命令，依序執行
- 單引號內的分號不視為分隔符

### 7. 條件判斷 [[ ]]

#### 7.1 二元比較

- **運算符**：`=`, `==`, `!=`, `<`, `>`, `<=`, `>=`
- `==` 與 `=` 對字串作用相同；`==` 右側無引號時可做 glob 樣式比對
- 兩邊皆為整數時走數值比較；否則為字串比較
- 引號內字串會正確去除外層引號再比較
- **尚未實作**：`-eq`, `-ne`, `-lt`, `-gt` 等 test 風格運算子

#### 7.2 字串判斷

- `[[ -z 字串A ]]`：空字串為真
- `[[ -n 字串A ]]`：非空字串為真

#### 7.3 檔案條件

- `[[ -e 檔案 ]]`：存在
- `[[ -f 檔案 ]]`：一般檔
- `[[ -d 檔案 ]]`：目錄

#### 7.4 正則比對

- `[[ $a =~ regex ]]`：Perl 風格 regex
- 支援：`.` `*` `+` `?` `[abc]` `[^abc]` `[a-z]` `^` `$` `\`

### 8. 算術運算

- **`$(( expr ))`**：展開為計算結果
- **`(( expr ))`**：作為條件，0→`$?`=0，非 0→`$?`=1
- 支援變數如 `x=5; echo $(( x++ )); echo $x`

### 9. 變數與環境

- 賦值：`VAR=value`
- 展開：`$VAR`、`$HOME`、`$?`
- `$?` 傳回前一指令的 exit status

### 10. 路徑展開

- **`~`** / **`～`**（全形）：`~` → HOME，`~/path` → HOME + "/path"
- **`$HOME`**：環境變數 HOME 展開

### 11. 內建指令

- **cd**：cd、cd ~、cd $HOME、cd ~/dir、錯誤處理
- **echo**：標準 echo 行為

### 12. 行編輯與歷史

- **Raw 模式**：關閉 ICANON、ECHO，逐字讀取
- **左右方向鍵**：ESC [ C / ESC [ D，游標移動
- **Backspace / DEL**：刪除游標前一字元
- **上下鍵**：歷史導覽（getPrevious / getNext）
- 非 TTY 時自動改用 readUntilDelimiterOrEof

### 13. Backspace 處理

- 僅在 move_left > 0 時送出 `\x1b[{n}D`
- termios 僅關閉 ICANON 與 ECHO
- just_backspace 保留以略過 BS+SP 序列中夾帶的 space

### 14. 模組化架構

- 指令與職責拆至不同模組
- 內建指令獨立於子目錄

---

## 二、測試範例（快速驗證）

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

# && 與 ||
true && echo a && echo b
false || echo ok

# 條件與算術
[[ -z "" ]]; echo $?
x=5; echo $(( x++ )); echo $x

# cd 與路徑
cd ~; echo $HOME; ls ~/

# 正則
[[ abc =~ a.* ]]; echo $?
```
