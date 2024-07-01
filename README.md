1. Architecture Diagram:
2. Introduction of each module:
I. Top.v:
功能:
主要功能是用來將下方的 module 連接起來。下面的每一個 module 都
是一個 component。Top module 將各個不同 module 的 input port 與
output port 連接起來，組成完整的 datapath。
Code 解釋:
在 code 開頭宣告了在 datapath 中會使用到的 wire，這些 wire 可以與
各個 module 的 input 與 output signal 連接在一起
舉例:在 controller 中，.opcode_in(dc_out_opcode)即表示將
dc_out_opcode 這條 wire 連接到 controller 的 opcode_in 的 port
II. ALU.v:
功能:
用來進行各種 instruction 所需的各種算術即邏輯運算。根據 instruction
的 opcode、func3、func7 可能會進行不同的運算。ALU 具有兩個
operand，經過運算後可產生一個 output。
Code 解釋:
由於 instruction 的 type 係由 opcode 決定，所以在 code 一開始我們先
將 define 各個 instruction type 對應到的 opcode。此外，由於指令所要
執行的行為由 func3 決定，所以我們同時也 define 了各個指令對應的
func3，方便接下來的分類。
接著，我們先以 opcode 分類，分成 R_type、B_type、J_type 等等，在
依 func3 及 func7 判斷指令的種類，再對各個指令撰寫其運算過程。
這裡比較特殊的是 I_type 的 opcode 並非全都相同，而是依運算行為分
成 load 及 arithmetic 兩類。
III. Decoder.v:
功能:
Instruction 中包含了 opcode、func3、func7、rs1、rs2 及 rd 等訊息，而
decoder 的功能就是將 instruction 的上述資訊擷取下來並輸出，利於接
下來 ALU 進行判斷要進行何種運算。
code 解釋:
opcode 的擷取是擷取 instruction 的第 0 到 7bit，func3 是在第 12 到
14bit，rs1 是第 15 到 19bit，rs2 是在第 20 到 24bit，最後，func7 是在
第 25 到 31bit。
Corner case:
這邊需要注意的是 opcode 我們只需要取第 2 到 6bit，因為在 RV32I 中
opcode 的前 2bit 都是 1，不隨指令而改變。另外 func7 也只要取
1bit，因為除了第 6 個 bit，func7 的其他 bit 皆不隨指令變化而更動，
所以我們只要取會改變的 instruction 的 bit30 即可。
IV. Imme_Ext.v:
功能:
用來處理指令中所包括的 immediate，並將其拓寬至 32bit，以利接下
來 ALU 的運算。
Code 解釋:
Immediate 的 extension 分為 2 種，分別為 signed 及 unsigned。而各不
得類型的指令存放 immediate 的區段不同，因此需要依指令類型擷取
immediate 的資訊，再進行 signed extension 或 unsigned extension。
Corne case:
R_type 的指令並沒有包含 immediate，所以我們將 R_type 的
immediate 設為 0。
V. JB_Unit.v:
功能:
用來處理 J_type 及 B_type 指令的跳轉地址，得到新的 PC(非 PC+4)。
Code 解釋:
Operand1 儲存目前的地址，operand2 則是儲存 immediate 或偏移量，
所以 operand1+operand2 則為目標地址
VI. LD_Filter.v:
功能:
用來過濾我們取出的資料。由於我們取出之資料為 32bits(8bytes)，所
以舉例來說，若我們要執行 lb 指令，我們只需擷取前 8 個 bits，左邊
再 sign extension。
code 解釋:
我們將 load 指令判斷為 lb、lh、lw、lbu、lhu 等，並將其擷取對應所
需之資料長，並再看指令為 signed 或 unsigend 對應是否做 signed 或
unsigned extension。
VII. RegFile.v:
功能:
Regfile 的主要功能為決定 rs1 及 rs2 的 data 為何，將值放置入 rs1、
rs2，讓 ALU 可以依照取出的值進行算術或邏輯運算。而若有需要
(wb_en==1)時，則將 ALU 運算完的值寫入 rd。
Code 解釋:
我們將 rs1_index 對應到的 register 之值放入 rs1，rs2 亦做同樣的操
作。Rd 則放入運算完的結果。
Corner case:
要注意的是若 rd_index==0 時是不可以將值寫入 rd 的，因為第 0 號
regieter 的值永遠是 0，不可改變。
VIII.Adder.v:
功能:
進行加法運算，可以將兩個輸入的值相加，輸出結果則為二者相加所
得的數值。在 datapath 中用來進行 PC+4 的運算。
Code 解釋:
有 3 個輸入訊號 x、y、cin，將 3 者做加法運算後得到 sum，若有進位
的話則 cout=1。
IX. Reg_PC.v:
功能:
主要是處理 PC 的 component。可以接收 rst signal，若輸入 rst 為 1，
則將 next_PC 重設為 0，若 rst 並不等於 1，則 next_PC 則為以計算好
的下個 clock 的 PC(可能為 present_PC+4c 或經由 branch 或 jump 所要
跳轉到的新地址)。
Code 解釋:
若 rst==1，則 next_PC=0(表 program counter 重新計算)。
若 rst==0，則 next_PC=計算好的數值。
X. Mux.v:
功能:
為一多工器，可以有 2 個 input port，通過 1 個 bit 的 sel siganl 選擇
output o 為哪一個 input
Code 解釋:
若 sel==1，則 o=in1，反之則為 in0
XI. SRAM.v:
功能:
用來做與 CPI 連接的 memory。我們模擬的 SRAM 總共有 65535 個 byte
address。SRAM 運作的機制為每個 address(4bytes)的每 1 個 byte 會接
收到 1 個 w_en signal，若 w_en==1 則該 byte 被 write data 寫入。
Code 解釋:
若 w_en[0]==1，則 mem[address]=write_data[7:0](其餘 3byte address 邏
輯、相同操作模式皆相同)。補充:此部分為同步電路。
若做 read operation，則 read_data=mem[address+3]、
mem[address+2]、mem[address+1]、mem[address]的串接數列。
XII. Controller.v
功能:
用來輸出各個 datapath 的 component 的控制訊號，以控制整個
datapath 的運作。
next_pc_sel 用來決定 next_pc 是甚麼，im_w_en 控制是否可以寫資料
進入 memory，wb_en 控制 Reg_file 是否要將 ALU 運算後的值寫入
rd，jb_op1_sel 用來控制進入 ALU 的 operand1 為何，jb_op2_sel 用來
控制進入 ALU 的 operand2 為何，opcode、func3、func7 則控制 ALU 讓
其知道目前是甚麼指令並做出對應的運算，wb_sel 則控制要 write back
入 rd 的數值為何，dm_w_en 則控制要 store 入 memory 的 data 為何。
Code 解釋:
我們在前面先定義的各個 type 對應到的 opcode，以及判斷為甚麼
type 的條件，方便 coding。
如果是 U_type、J_type、I_type、R_type 的話，便輸出 wb_en=1(寫東
西進 rd)。
如果是 R_type、B_type 且若 jb_op1_sel==1 則多工器輸出 rs1_data，若
否則輸出 current_pc。
如果是 U_type、J_type、JALR 的話，alu_op1_sel=1 且 mux 選擇輸出為
current_pc，若否則 alu_op1_sel=0 且 mux 選擇輸出為 rs1_data。
如果是 R_type、B_type 的話，alu_op2_sel=0 且 mux 選擇輸出為
rs2_data，若否則 alu_op2_sel=1 且 mux 選擇輸出為 imm_ext_data。
如果是 I_load 的話，則 wb_sel=1 且選輸出為 load_data_f，若否
wb_sel=0，且選擇輸出為 alu_out。
如果是 s_type 則視其 func3 給與 dm_w_en 不同的樹出數值。

