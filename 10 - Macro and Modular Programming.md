你好！我是你的 **emu8086 組合語言家教**。

我們終於來到期末考前的最後一個大單元：**【巨集 (Macro) 與模組化程式設計】**。

這章非常輕鬆愉快，因為你已經撐過了最難的堆疊參數傳遞。這一章的本質不是「學新指令」，而是**「學會如何偷懶」**和**「整理程式碼」**。

---

### 1. 重點觀念：巨集 (Macro) vs. 函式 (Procedure)

在 C 語言中，你有兩種方式可以重複使用程式碼：
1.  **函式 (Function)**：`void func() { ... }`
2.  **前處理指令 (Define)**：`#define MAX(a,b) ((a)>(b)?(a):(b))`

在 8086 組合語言中：
*   **函式 (Procedure, PROC)**：
    *   就是上一章學的 `CALL` / `RET`。
    *   **特性**：程式碼只有一份，執行時跳過去再跳回來。
    *   **優點**：**節省記憶體空間** (Space)。
    *   **缺點**：執行較慢 (因為 `CALL/RET` 和堆疊操作有額外開銷)。
*   **巨集 (Macro)**：
    *   就是 **「文字替換」**。
    *   **特性**：組譯器在編譯前，會把你寫巨集的地方，直接**複製貼上**原本的程式碼。
    *   **優點**：**執行速度快** (沒有 `CALL` 的動作，直接往下跑)。
    *   **缺點**：**浪費記憶體空間** (呼叫 10 次，程式碼就會膨脹 10 倍)。

**期末考重點提示：**
考試一定會考 **Macro 與 Procedure 的比較**。
關鍵字：**巨集換空間 (Space) 求時間 (Time)，程序換時間求空間。**

---

### 2. 相關暫存器如何被用到

巨集本身不特定使用哪些暫存器，**完全看你巨集內容寫什麼**。但也因此產生了一個嚴重問題：**副作用 (Side Effect)**。

*   **AX, BX, CX, DX...**：
    *   如果你寫一個巨集 `PRINT_CHAR` 裡面用到了 `AH`。
    *   主程式在呼叫前剛好在用 `AH` 算數學。
    *   **結果**：巨集展開後，主程式的 `AH` 被蓋掉了，程式邏輯壞死。
*   **解決方法**：
    *   在巨集開頭 **PUSH** 所有會用到的暫存器。
    *   在巨集結尾 **POP** 所有會用到的暫存器 (順序相反)。

---

### 3. 詳細解釋每一個相關指令

#### A. MACRO / ENDM (定義巨集)
*   **語法**：
    ```assembly
    巨集名稱 MACRO [參數1, 參數2...]
        ; 程式碼片段
    ENDM
    ```
*   **用途**：定義一段要被重複貼上的程式碼。

#### B. LOCAL (區域標籤)
*   **語法**：`LOCAL label_name`
*   **用途**：**非常重要！**
    *   如果在巨集裡面寫了迴圈 (`JMP L1`)。
    *   當你呼叫兩次巨集，程式碼被貼上兩次，就會出現**兩個 `L1`**。
    *   組譯器會報錯：**Duplicate Label (重複標籤)**。
    *   `LOCAL` 告訴組譯器：「每次展開時，自動幫這個標籤改名 (如 `??0001`, `??0002`)」。

#### C. INCLUDE (模組化)
*   **語法**：`INCLUDE 'filename.inc'`
*   **用途**：將寫好的巨集存在另一個檔案，主程式直接引用。C 語言的 `#include`。

---

### 4. & 5. 範例程式碼與 C 語言對照 (emu8086)

我們來寫一個常用的功能：**印出一個字元**，並比較 C 語言的 `#define`。

**C 語言：**
```c
#define PRINT_CHAR(c) { \
    char temp = c;      \
    putc(temp, stdout); \
}

// 使用
PRINT_CHAR('A');
PRINT_CHAR('B');
```

**Assembly (emu8086)：**
```assembly
; 定義巨集 (通常寫在程式最上面，或是放在獨立檔案)
PUTC MACRO char
    PUSH AX         ; [保護現場] 暫存 AX
    PUSH DX         ; [保護現場] 暫存 DX
    
    MOV DL, char    ; 參數替換：把傳進來的 char 放進 DL
    MOV AH, 02H     ; DOS 功能 02H: 印出字元
    INT 21H
    
    POP DX          ; [恢復現場] POP 順序要相反
    POP AX
ENDM

; 定義一個有迴圈的巨集 (展示 LOCAL)
ABS_MACRO MACRO Reg
    LOCAL SKIP_NEG  ; 宣告 SKIP_NEG 是區域標籤，避免重複定義錯誤
    
    CMP Reg, 0
    JGE SKIP_NEG    ; 如果是正數，跳過
    NEG Reg         ; 如果是負數，取補數
SKIP_NEG:
ENDM

; --- 主程式 ---
DATA SEGMENT
DATA ENDS

CODE SEGMENT
    ASSUME CS:CODE, DS:DATA
START:
    MOV AX, DATA
    MOV DS, AX

    ; 1. 使用 PUTC 巨集
    PUTC 'H'        ; 組譯器會在這裡貼上 6 行指令
    PUTC 'i'        ; 這裡又貼上 6 行指令
    
    ; 2. 使用 ABS 巨集
    MOV BX, -5
    ABS_MACRO BX    ; 展開後，BX 變 5
    
    MOV CX, 10
    ABS_MACRO CX    ; 展開後，不會報錯，因為 LOCAL 會自動換標籤名

    MOV AX, 4C00H
    INT 21H
CODE ENDS
END START
```

---

### 6. 程式執行前後變化與推導 (Trace)

我們要理解**「展開 (Expansion)」**的概念。

假設 `PUTC` 巨集內部有 6 條指令 (PUSH, PUSH, MOV, MOV, INT, POP, POP)。每條指令平均佔 2 bytes。

| 動作 | 程式碼外觀 (Source Code) | 記憶體中的指令 (Memory / Machine Code) | IP 變化 (估算) |
| :--- | :--- | :--- | :--- |
| **寫法** | `PUTC 'A'` | `PUSH AX` <br> `PUSH DX` <br> `MOV DL, 'A'` <br> `MOV AH, 02` <br> `INT 21` <br> `POP DX` <br> `POP AX` | IP 前進約 12 bytes |
| **比較** | 若用 `CALL PUTC` | `CALL PUTC` | IP 前進 3 bytes (但跳到很遠的地方執行再回來) |

**執行後暫存器變化：**
由於我們在巨集裡用了 `PUSH/POP` 保護：
*   執行前 `AX=XXXX`。
*   巨集內 `AH` 變 `02`。
*   執行後 `AX` 恢復 `XXXX` (這就是 Side Effect Free)。

---

### 7. 初學者在 emu8086 最常犯的錯誤

1.  **重複標籤 (Duplicate Label)**：
    *   **錯誤**：在巨集裡寫 `L1: LOOP L1`，然後在主程式呼叫兩次巨集。
    *   **後果**：組譯器報錯 "Label already defined"。
    *   **修正**：在 `MACRO` 下一行加上 `LOCAL L1`。
2.  **暫存器污染 (Register Clobbering)**：
    *   **錯誤**：巨集裡改了 `CX` (例如用來做 Loop)，結果主程式原本也在用 `CX` 跑外層迴圈。
    *   **後果**：主程式的迴圈亂掉，變成無窮迴圈或立即結束。
    *   **修正**：善用 stack (`PUSH CX` ... `POP CX`)。
3.  **參數傳遞誤解**：
    *   巨集的參數是**字串替換**。
    *   如果你傳入 `[BX]`，巨集展開時就是 `MOV AX, [BX]`。
    *   如果你傳入 `AX`，展開就是 `MOV AX, AX` (無意義)。
    *   這跟 C 語言的「傳值 call by value」完全不同。

---

### 8. 考前特訓：10 題「自己動手寫」小練習

這些題目模仿期中考題風格，但針對巨集與記憶體結構。

**題目：**

1.  **[概念比較]** 請簡述 `MACRO` 與 `PROC` (子程序) 在「執行速度」與「程式碼大小 (Memory Size)」上的主要差異。
2.  **[指令功能]** `LOCAL` 指令在巨集定義中主要解決什麼問題？如果不加會發生什麼錯誤？
3.  **[記憶體計算]** 假設一個巨集 `M1` 內部包含 5 個指令，每個指令長度為 2 bytes。若在主程式中呼叫了 `M1` 4 次，請問這部分程式碼在記憶體中總共佔用多少 bytes？
4.  **[記憶體計算 - 對照組]** 承上題，若改為 `PROC` 寫法 (函式本體 5 個指令)，主程式呼叫 4 次 (每次 `CALL` 佔 3 bytes)。請問總共佔用多少 bytes？(含函式本體與 4 次呼叫)。
5.  **[參數傳遞]** 定義巨集 `MOV_MACRO MACRO A, B` 內容為 `MOV A, B`。若呼叫 `MOV_MACRO [BX], AX`，展開後的指令為何？這屬於哪種定址法？
6.  **[副作用除錯]**
    ```assembly
    MY_MACRO MACRO
        MOV CX, 10
        LOOP $
    ENDM
    ; 主程式
    MOV CX, 5
    MY_MACRO
    LOOP START
    ```
    請問這段程式碼執行時，主程式的迴圈會發生什麼問題？
7.  **[語法修正]** 以下巨集定義有一個致命語法錯誤，請修正：
    ```assembly
    TEST MACRO
        JMP L1
    L1: NOP
    ENDM
    ```
    (提示：若呼叫兩次會怎樣？)
8.  **[展開推導]**
    ```assembly
    ADD_MACRO MACRO P1
        ADD AX, P1
    ENDM
    ; 呼叫
    ADD_MACRO 10H
    ```
    展開後的機器碼邏輯等同於哪個指令？執行後 `AX` (原值 0) 變多少？
9.  **[Include]** 指令 `INCLUDE 'MYLIB.INC'` 其實是告訴組譯器做什麼動作？
10. **[堆疊平衡]** 如果在巨集開頭寫了 `PUSH AX`，但在巨集結尾忘了寫 `POP AX`，連續呼叫該巨集 100 次後，`SP` (Stack Pointer) 會產生什麼變化？

---

### 9. 練習題詳解與推導

**1. [概念比較]**
*   **Macro**：速度快 (無 Call/Ret 開銷)，程式碼大 (重複展開)。
*   **Proc**：速度慢 (有跳轉與堆疊開銷)，程式碼小 (只有一份實體)。

**2. [指令功能]**
*   解決 **重複標籤定義 (Duplicate Label Definition)** 的錯誤。
*   組譯器會自動為標籤產生唯一名稱 (如 ??0001)。

**3. [記憶體計算 - Macro]**
*   巨集是「複製貼上」。
*   (5 指令 * 2 bytes) * 4 次呼叫 = **40 bytes**。

**4. [記憶體計算 - Proc]**
*   函式本體：5 * 2 = 10 bytes。
*   呼叫指令：4 次 * 3 bytes (CALL) = 12 bytes。
*   總共：10 + 12 = **22 bytes**。
*   *(結論：在這個例子中，用 Proc 比較省空間)*

**5. [參數傳遞]**
*   展開後：`MOV [BX], AX`。
*   定址法：**暫存器間接定址法 (Register Indirect)**。

**6. [副作用除錯]**
*   **暫存器污染**。
*   主程式原本 `CX=5` 準備跑迴圈。
*   巨集內部把 `CX` 改成 `10` 並且跑到 `0`。
*   巨集結束後 `CX=0`。
*   主程式的 `LOOP` 執行 `0-1` 變成 `FFFF`，導致外層變成**無窮迴圈**。

**7. [語法修正]**
*   缺少 `LOCAL` 定義。
*   修正：
    ```assembly
    TEST MACRO
        LOCAL L1  ; 加上這行
        JMP L1
    L1: NOP
    ENDM
    ```

**8. [展開推導]**
*   展開為：`ADD AX, 10H`。
*   AX = 0 + 10H = **0010H**。
*   這就是「立即定址法」。

**9. [Include]**
*   將 'MYLIB.INC' 檔案的內容，**原封不動地複製貼上** 到目前指令的位置。

**10. [堆疊平衡]**
*   每次呼叫 `PUSH AX`，SP 會減 2。
*   呼叫 100 次，SP 會減 200 (200 bytes)。
*   這會導致 **Memory Leak (Stack Overflow)**，最後可能覆蓋到其他資料段或程式碼。

希望這份教學能幫你搞懂巨集的概念！記住：**巨集就是 Copy-Paste**，這句話可以解決 90% 的巨集問題。這是期末考前的最後一塊拼圖，祝你考試順利！如果還有哪裡不清楚，歡迎隨時問我。