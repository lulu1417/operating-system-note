# 作業系統
## Ch3：Process
* Process定義：在記憶體中被執行的程式
* Process包括：
    1. Code segment：程式碼
    --------2~4 變數放在以下三個地方--------
    3. Data section：全域變數
    4. Stack：區域變數，function call完就會消失
    5. Heap：動態分配的變數，例如malloc()出的變數
    --------6、7管理process的資料--------
    6. Program counter：為CPU中的暫存器，主要功能為紀錄目前process執行到哪和register content(因為程式在執行時不會一直存在於register的同個位置，而是頻繁地切換，因此要紀錄程式的register content)
    7. Resource：紀錄類似token的東西，表示開了哪些file、用了多少socket的port、hardware的control等
* Thread：使用CPU最小的單位
    * 單一執行緒：一個process對到一個thread
    * 多執行緒：多個thread處理同個process，同個process的thread會共享code section、data section和OS resourece，但各自有獨立的program counter、stack和register

* Process的狀態
    * new：process被load進disk，建立process的五個segment
    * ready：在memory中，準備被執行，用queue來管理，scheduler會決定被CPU執行的先後順序
    * running：被CPU選中執行中的process
    * waiting：process在做I/O或event wait的時候，會放掉CPU的執行權，需要CPU時才會再進入ready state
    * terminated：process結束，釋放記憶體空間
* 一次只會有一個process在running狀態，但會有很多個process在ready或waiting狀態
* 如何管理：OS幫每個process建立一個process control block，裡面包含：
    * pointer：由於process在ready queue中是以linked list的形式存放，pointer會指到該process的下一個process
    * process state：紀錄他目前在哪個狀態
    * program counter：process執行到哪
    * CPU register
    * CPU scheduling information
    * Memory-manage information：紀錄base和limit register
    * I/O status information：紀錄在哪個I/O裝置做I/O
    * Accounting information：紀錄開了幾個file
* Context switch：在不同的process之間切換
    * 例如：一個process被CPU執行到一半，call了system call，此時CPU就必須切換去執行OS，而該process就會進入Idle狀態，中間切換的時候要先記錄原process的狀態到他的PCB，切換到新的process時也要從他的PCB裡load進資訊
    * ![](https://i.imgur.com/SmT54S1.png)
    * context switch的時間是overhead(如上圖，會有兩個process都處在idel狀態的時候)
    * 應減少context switch時間：
        * OS的設計能call一個function就同時把register內容load到memory和寫入register內容一併執行
        * hardware support：hardware中本身就設有幾個register，不用把PCB的content先移到memory

* 問題：waiting那邊的event指的是什麼？
### Scheduling
* long-term：控制multi-programming的程度，new->ready
    * 目標：讓CPU bound和I/O bound的時間是重疊、平均的
    * memory夠大的情況下不需要long-term scheduler，所有要執行的program都load進memory
* short-term：ready->running
    * 需要頻繁的執行
    * 執行時間越短越好
* mid-term：ready->waiting
    * 沒有long-term scheduler的情況下，先全load進memory，再透過mid-term來挑選誰要留下來
### Process Creation
* process會生出小process，每個都有自己的pid
* share resource三種情況：
    * parent和child共用一樣的resource
    * parent和child共用部份相同的resource
    * parent和child自己有自己的resource，沒有共享
* 執行的兩種情況：
    *  parent和child同時執行
    *  parent等child執行完才執行
*  memory content的兩種情況：
    *  child複製一份parent的(看起來一模一樣只有memory address不一樣)
    *  child從parent那邊load進memory content
* copy-on-write：當呼叫者請求相同資源時，他們會拿到指到同個資源位置的指標，只有當呼叫者要修改資源時，系統才會真的分配一塊特製的副本，而上述過程對其他呼叫者來說都是透明的。有點類似git的概念，不儲存整份程式碼備份，而是只儲存不同的地方。這麼做可以大量節省複製重複內容的空間。
* 三種system call：
    * fork()：複製一份
    * execlp()：重新reset，換一隻新的程式執行。會覆蓋原來的程式碼(copy-on-write)
    * wait()：等process的child完成再執行
* 問題：Ch3的F，16:40程式碼看不太懂
    * ![](https://i.imgur.com/oKLELTc.jpg)
    * parent process執行完 ```A = fork();```後但還沒得到回傳值時複製一個child，child不會再call一次fork()，但會得到fork()的回傳值```A = 0```。換句話說就是一個process call fork()但此時該process已經分裂成兩個了，所以會得到兩個不一樣的回傳值。
* 所以講這麼久，到底為什麼要有parent生出child這個機制？為什麼不直接跑兩個不同的process就好?
    1. 是為了要實現多工，一次可以跑很多個process，尤其在GUI特別常見這樣的設計，因為要讓使用者能同時背景跑很多軟體、或即時開啟、關閉視窗等等。
    2. process的init很吃空間和時間，直接複製一個比較快
    3. 要注意的是parent和child的memory content只有在一開始複製時會一樣的，之後就是各自執行，區域變數都會不一樣。跟共享資源時保持資料一致性等等是無關的，反而是因為要實現多工，所以parent、不同child之間的變數內容不一樣才是正常的。
* ![](https://i.imgur.com/SXnHYVK.jpg)
    * 如果child的程式改成執行execlp()，則child執行完就結束，不會再fork()另一支程式。
* 當自己的child佔用太多資源時，或child不再被需要時，parent可以把自己的child abort()掉。
* 當一個parent被kill時，他的children也會一起被kill

### Interprocess Comunication
* IPC的目的：
    * 資源的交換：ex. socket
    * 多個process同時做一個工作，可能可以加快執行速度(not always)
    * 同時執行多個工作(conveniece)
    * 模組化：ex. micro kernal
* Communication Method
![](https://i.imgur.com/SP5ixMh.jpg)
    * Shared Memory：多個process共用同一塊memory
        * 優點：速度快
        * 缺點：需要更注重user synchronizaion
        * 適用於參與者少、溝通單純時，例如本機程式之間的溝通
        * OS預設情況下Shared Memory是不被允許的
    * Message Passing：每個process有自己的一小塊memory空間，透過system call的message passing傳遞內容。
        * 優點：可避開synchronization問題，某些情況下(ex.small data)效率較好
        * 適用於參與者多、溝通複雜時，例如跨電腦的程式溝通
* Message Passing Methods
    * Socket：
        * 根據ip和port來辨識誰和誰傳遞資料
        * 沒有資料結構，資料以一堆bytes傳遞
    * Remote Procedure Calls：
        * 可以去call另外一個process、甚至是另一台電腦的function
* 
     

## Ch2：Structure
* 可以把OS視為對系統唯一的api，他提供的system call就是OS services
![](https://i.imgur.com/WitKs8x.jpg)
    * user interface: GUI vs. CLI
    * Communication：不只是一台電腦內程式間的溝通，也可以指跨電腦的溝通
        * 傳遞放在memory中的資料，分為Msg passing和shared memory
        * Msg passing：memory copy，每個process都有自己的一小塊儲存空間放相同的memory content，copy都要透過kernel
        * shared memory：多個process共用一塊memory。multi-thread就是在create一個新的thread時部分memory預設就會新增一塊新的空間出來。
        * 前者缺點為速度慢，後者則是會遇到dead lock、synchronization的問題
* api和system call是不一樣的
* 為講求效能，system call是用組合語言寫的，而user program要去call system call的時候就需要透過api，api通常是由更高階的語言寫成的，例如手機系統的api就是java寫的
* 階層由上層到底層分別為：user process、api、system call，最後才會輪到OS來執行system call的內容
* 一支api可能會呼叫很多個system call，也有可能完全不會call到system call(目的是包起來比較方便使用者寫program)，例如基本的數學運算如abs()，pow()等
* 為方便debug，system call越短，彼此之間獨立運作、越不互相干擾越好
* user process在call api的時候並不會觸發interrupt，interrupt是在call system call的時候被觸發
* 為什麼要用Api?
    * 簡單、效能：讓OS提供的這個介面以更有效率的方式來實作使用者program(OS當然比使用者更熟悉system call的用法)，若沒有api，開發者必須自己把多個system call兜在一起
    * 可移植性：
* 三種system call傳遞參數到OS的方式
  1. 直接寫到register，CPU讀到就能做事了
  2. 將參數存在table in memory(就是傳pointer)
  3. 每個proccess都有一塊memory，就是stack，把要傳遞的參數push進stack，再由OS pop出來

### System Structure

* 對於使用者而言：系統設計越簡單、越穩定、越快(和系統互動的反應速度)越好
* 對系統而言：要能讓設計、debug、實做和維護更容易，也講求效能和穩定性
### Simple OS Structure
* 只有一、兩層階層的系統程式
* 例如MS-DOS：除了user program外，只分系統程式跟driver，接下來就是bios了
* 缺點：不安全、難維護
### Layered OS Structure
* 除了OS、driver和user program由上到下分別為Operator、User program、I/O management、driver、memory management、process allocation multi-programming和hardware
* 上層可call下層，下層不能call上層，例如memory可以call process，但memory不能call I/O
* 優點：安全、易維護
* 缺點：效率差、階層間難定義
### Micro kernel
* kernel的程式碼越少越好
* 有模組化的概念，kernel只負責在這些模組間溝通
* 優點：易維護、擴充
* 缺點：為避免synchrnization，參數傳遞要一直copy，效率較差
### Modular OS
* 在kernel內部做模組化，因此能避免像micro kernel在溝通時message passing效率差的問題
* loadable：可以load不同模組來調整OS的功能
### Virtual Machine
* 把真的OS和hardware視同為hardware的假OS
* 一台電腦可以有很多個虛擬機同時運作，而彼此之間獨立不互相干擾，都把OS視為屬於自己的而不是和別的虛擬機共用的
* 虛擬機的kernel也還是在user space，在執行kernel的instruction時，會先被擋下來，這時virtual machine implementation就會代替假的kernel來執行kernel instruction。
* hardware support：除了user mode、kernel mode之外，還有多一個vm mode
* critical instruction：在user space執行的結果和在kernel space執行的結果不一樣
* VM的用途：
    * 可保護真的OS
    * 更高的相容性，例如：MacOS灌linux
    * 提供作業系統的開發一個更安全的研究和測試環境，例如：研究OS如何被攻擊
    * 雲端計算
* VM的類型：
    * full virtualization：虛擬機完全不知道自己是虛擬機，以為自己是跑在真的hardware之上
    * para virtualization：比全虛擬多一個global zone的OS程式，該程式知道其他虛擬OS的存在，而其他guest OS(虛擬機)是特殊的、被修改過的，不是隨便一個guest OS就能裝上去的
    * para的效能會比full好，因為有global zone的OS程式作為adminitrator來做最後的整合，缺點為因為多一層從中協調者，因此會有overhead
## Ch1：Introduction
### 什麼是作業系統？
* 電腦系統的四個組成要素：user, application, hardware, OS
    * user：可能是人、機器或電腦，例如自動駕駛的車子之間，可視為多個使用者在同個系統下溝通互動
    * application：用來定義對系統資源操作與計算的方式
    * OS：負責控制和協調溝通，分配系統資源並控制硬體，例如：處理time sharing的問題等
    * hardware：基本的運算資源，例如：CPU、memory、I/O裝置(鍵盤滑鼠螢幕)等
* OS要解決的核心問題：讓使用者共用、讓使用者和系統互動
* OS和其他軟體的差別：OS是永久性的，只要電腦一啟動就會開始運作，可視為一個大型且永遠存在的api，其他api要透過這個主要的api來跟hardware溝通
### 一般的電腦系統
* 電腦系統分為user mode和kernel mode：
    * user mode：編譯user program，並link到system library，再執行編譯完的程式碼
    * kernel mode：包括OS和hardware，其中還有device driver來控制hardware，也算是OS的一部份但功能比較單純
    * 兩者間透過OS interface(system call)來溝通
* ***system call*** ：system的function call，指的是OS的api(system library)
* 例如印出字串```printf()```，最後會去呼叫 system library的function讓OS在螢幕上顯示字串
 ![](https://i.imgur.com/WEUCdF6.jpg)
* kernel一般指的就是OS
### 作業系統的目的
* 方便性：讓電腦更容易被操作
* 效能：OS能有效分配系統資源，尤其在有多位使用者的情況下
* 然而以上兩者是互相矛盾的，例如為了讓一般民眾也能輕易操作電腦，新增了GUI，卻可能使電腦效能更差
### 作業系統的重要性
* OS是user application和hardware之間唯一的interface，因此不容許有bug
* 會和hardware driver互相影響，因此OS最好設計成能和hardware driver完美配合，例如wintel
### 作業系統的組織
* 馮紐曼架構：Memory+CPU+I/O device組成電腦架構
* OS必須在多個CPU和裝置同時運作並且共用同一個memory的情況下能不影響到彼此
* CPU會下指令叫device controller將要放到memory的資料先寫進
* buffer(或反過來)，再把buffer的資料輸入或輸出到I/O device上
![](https://i.imgur.com/GnhGn8A.jpg)
* #### 但controller透過bus將資料寫入memory(或從memory中移出)的速度很慢，為了等待寫入完成，會造成CPU閒置，如何解決？


    * 原本的作法：在搬移過程中必須透過while中的peek()反覆檢查buffer狀態，看還有沒有空位，但在這個過程中CPU會被霸佔
    * 解決方法：interrupt(又稱為trap或signal)
        * CPU是event driven的：可以隨時打斷CPU，叫他去做別的事---CPU是被動被呼叫的
        * device controller在搬移資料的時候，CPU能同時去處理別的事，不用等資料搬完。當搬移結束時，再發出interrupt通知CPU(告知資料搬移的狀態、搬到memory的什麼位置等)
        * 由於CPU會中途被打斷，因此也要記下被打斷前原本的程式執行到哪
        * interrupt被觸發時機：
            * 使用者點下滑鼠、按下鍵盤等 (hardware routine)
            * 程式執行出現錯誤時，透過interrrupt印出錯誤訊息(software routine)
            * system call時，一定會觸發interrupt
            * user program會先被編譯完，link到system library的對應function，變成system call再觸發interrupt
        ![](https://i.imgur.com/Sijj2AO.jpg)

        * 目前所有電腦系統的CPU都是interrupt driven的
            * 分成software和hardware的interrupt。hardware叫signal，software叫trap
        
        * 當interrupt時，CPU會先去看interrupt vector，找到對應的funtcion pointer
        * 如果是signal，CPU要怎麼知道是誰發出的interrupt?
        當I/O裝置連接電腦的port時，port上會有一個寫死的欄位，像幫裝置編號一樣紀錄，以便在interrupt產生的時候讓CPU辨識是哪個裝置
        * interrupt的overhead：需要花更多儲存空間紀錄如CPU原本的程式執行到哪、誰發出interrupt等資訊
    #### interrupt流程：
    ![](https://i.imgur.com/XRmJNBi.jpg)

### 儲存裝置
* 儲存裝置階層
        1. register：CPU裡直接做一個可以暫時存放資料的地方
        2. cache：介於memory和CPU中間
        3. ***memory***
        4. disk
        5. 光碟
        6. 磁帶

    * 速度：越上層越快
    * 價格：越下曾越便宜
    * volatility(揮發性)：memory以上關掉就會遺失
    * memory以下的儲存空間都屬於secondary storage，可視為memory的擴充
    * 前一張提到的hard real-time system不會有secondary storage，否則會太慢
#### memory
* main memory是唯一的能讓CPU直接執行的儲存空間
* DRAM：便宜、較不耗電但速度慢;SRAM則相反，通常拿來做cache
#### disk
* 大多資料儲存的地方
* 有讀寫手臂和磁盤，讀取時間不固定，會根據讀寫的位置不同有不同速度(例如離磁頭的距離)
* 讀取時間包含seek time(找到資料位置)和旋轉時間(讀寫頭移到適當位置的時間)
#### cache
* ***這邊的cache和瀏覽器的cache是不同的概念***
* 將經常用到的資料暫時性的從速度慢的空間***複製*** 到速度較快的空間
* 儲存空間分階層，將越常用到的資料cache在越上層
* 由於CPU會從最靠近memory的儲存空間開始找需要的資料，且cache必須複製資料，因此cache錯資料造成miss反而會使速度更慢
* 處理巨量資料的系統，由於memory放不下，通常連cache都沒有
* 缺點：不一致性
    * cache中存的是資料的copy，若多支程式對資料做修改，會讓cache和disk中的資料內容不一致，尤其在share memory content的系統下
    * 例如在多核心系統，一隻程式跑到一半可能會換到另一個core上，看到的cache就不會是同一個
    * 對於像google的大公司，為了擴大系統規模，會放棄維持資料內容一致性
#### protection
* 在同時有很多使用者的情況下能互不影響，某隻程式壞掉，其他能照常運作
    * 這邊的多位使用者指的是例如兩個不同身份的人同時連上一台GCE
* hardware的Dual-mode：要讓CPU能區分user或OS發出的指令
    * user mode：OS以外在執行的時候
    * kernel mode：OS正在執行的時候
    * 將mode bit加到hardware
    * 問題：mode bit是怎麼被送到hardware去的?
    * CPU透過mode bit(0,1)來區分user(1)或kernel(0)，只要有interrupt就會從1切換成0
    * privileged intructions：在kernel mode才能執行
    * 所有I/O instructions都是privileged intructions
* memory protection：
    * CPU紀錄兩個register: base register & limit register
        * base register：最小的合法memory位址
        * limit register：可使用範圍的大小
            * malloc()：修改limit register
    * segmentation fault：指到不該指的memory空間，就是出於memory protection的一種
* CPU protection：
    * 避免一隻程式霸佔CPU
    * 例如切斷無窮迴圈的機制
    * 作法：timer---在固定的時間週期內丟出interrupt，呼叫OS的scheduler來決定接下來執行哪支程式
    * load-timer也是privileged intruction，使用者不能隨便修改
### Hardware的Dual-mode補充
定義：系統運作的狀態主要分為兩種模式
Monitor mode(又稱supervisor mode, system mode)
在此mode下，主要是OS的system processes在執行(OS的system process執行的狀態，在此mode下，OS掌控系統的控制權)
(eg. ISR, systemcall, 對應的service routine)
在此mode下才有權執行特權指令(priveleged instruction)
User mode
user program可以執行的狀態
在此mode之下，不能執行特權指令！否則會產生致命錯誤中斷(trap)，OS會強迫process中止
Dual mode的區分主要是靠Hardware支援，主要是由mode bit區分，通常
0：表monitor mode
1：表user mode
實施Dual mode的目的：
對Hardware重要的resources實施protection，把可能引起危害的一些機器指令，設為priveleged instruction，如此可防止user program直接使用這些指令，避免user program執行這些指令對系統或其它user造成危害
(∵user mode下無法執行特權指令)
    

## Ch0：Historycal Prospective

* Mainframe:巨大的電腦
* Batch system：CPU一次做一件事，把memory切成兩部份，一部份給系統，剩下的給使用者。已經消失在世上
    * 缺點：
        * one job one time，速度很慢
        * 使用者和系統之間沒有interaction(time sharing，指的是在程式快速的切換)
        * CPU有很長一段時間是閒置的，因為大部分時間要等I/O
* Multi-programming(多工)的概念及實作
    * 概念：為了增加CPU的使用率，Spool(simultaneous peripheral operation on-line):
    為了要能讓CPU在很長的I/O時間可以去處理別的process，必須讓CPU在I/O時間可以完全不需要下達指令，只要在I/O做完時通知CPU即可
    * 讓I/O上的chip也可以幫忙CPU做計算
    * interrput
    * CPU要執行的job(編譯完的程式碼)必須先從disk被load進memory，多工就是要讓多個job同時存在於memory
* Multi-tasking(time-sharing)
    * 解決multi-programming無法解決的問題：讓使用者能跟系統做互動、一台電腦可以讓多位使用者同時使用
    
    * 為了實現time-sharing而衍生出其他問題，因此有了：
        * virtual memory：為了將更多程式塞進memory，把disk的部分空間當成memory
        * 檔案系統和disk management：因為很多時候CPU處理的是操作file，因此需要有檔案系統對這些files做更有效的管理
        * syncronization和dealock：由於time-sharing讓程式跟程式之間能做更多的互動來完成一件事，為了確保在這些程式同時修改memory中的內容時能維持正常而設計的機制
    * 和multi-programming的差別在於：被執行的job會快速切換，不會一次做完才換下一個
        * 例如當使用者點下滑鼠，系統就會立刻執行被指定要做的事，一有input就會立刻react
        * 被切換的時機：每個程式都會有一個時間限制，時間到了就換
        * multi-programing的設計跟多位使用者同時使用電腦沒有關係
* 當初設計PC時未考量到有網路的情況，因此沒有防止OS被攻擊的機制

* Multi-processor(Parallel system)：多個CPU同時運作，透過system bus(硬體)將CPUs和memory連結在一起，一個主機板上可能有多個CPU插槽
    * 優點：增加每秒的計算量、economic：memory可以共用，省下成本、reliability：其中一個CPU壞了其他還是能照常運作
    * 又分為symmetric multi-processor system跟asymmetric multi-processor system：
        * SMP：每個CPU所扮演的角色完全相同,大部分的裝置都是這種
            * 缺點：需要一些locking的機制來解決synchronization
        * AMP：會有一個master CPU管理其他CPU，core數量可以更多，通常用於超級電腦，或一些大型特殊的電腦系統
            * 缺點：master CPU不能做計算，只能用來管理其他CPU有點浪費
    * Multi-core processor：CPU上有多個core，各自有各自的cache(可能有不同content)來幫助加速
        * 為了解決cache存有相同memory content的core出現不同步的情形，必須加入如當core改寫cache內容時也要及時向memory反應
    * Many-core processor：
        * 例：GPU(用顯卡做圖形計算)，上面有幾百個core，速度很快
        * Single instruction multi-data：同時做一件事但可放到不同的data上
        * 目前的趨勢：盡可能增加core數量，且儘量緊密放在一張chip上
* 可以在multi-processor的裝置上再放multi-core processor，就會非常緊密，有超級多core
    * 要注意必須在很多個core運作時不會撞在一起
* 要如何在多個CPU同時run的時候share memory？
    * Memory access architecture：分為UMA和NUMA
        * UMA (Uniform Memory Access)：所有CPU都用同樣的方式接到memory上，重點是memory access time要一致，速度要相同，好處：不管在哪個CPU上執行效能感覺都一樣，一般都用UMA
        * NUMA (Non-uniform Memory Access)：用階層的方式管理，將memory切成不同區塊，連自己的memory(local)速度會比較快，連別的CPU(remote)的要先跟那塊memory controller溝通比較慢，for比較大的系統，OS需要妥善管理memory的分配
* Distributed system
    * kwon as loosly coupled，例如透過網路將多台電腦連在一起
    * 目的：資源共享、分擔loading、*reliability（避免任一台電腦壞掉時資料爛掉）
    * 架構：peer to peer、client-server
        * client-server：例FTP
            * 缺點：server容易變成bottleneck，當server壞了整個系統就死了
            * Solution：用多台電腦來做server
        * peer to peer：大家都是相同的，沒有server，要有protocal來規範溝通
            * 例如：BitTorrent、網際網路、串流系統，會有多個節點分享相同資源供鄰近使用
            * 優點：reliability比client-server更高，因為不怕serve掛掉
        * clustered system：把許多台電腦連結在一起
            * 特點：網路的連結是local area的，選鄰近的、比較好的網路，因此相對會比較快
            * 例如：infiniBand
            * 又分成symmetric和asymmetric clustering，前者有master,後者則是平等的
* System purpose(使用目的)：
    * real-time system：要讓所有job在死線之前做完，scheduling非常重要
        * 例如汽車、武器、醫療設備
        * 又分hard跟soft：顧名思義，soft盡量在死線前完成，例如影片播放，先跑出輪廓，最後才是更高的畫質;hard則是一定要完成，上面的例子都是hard real-time system
        * hard 特徵：幾乎沒有secondery storage，通常只有memory，或read-only memory(ROM)，因為memory之外的storage讀取時間慢、難以掌握
    * multi-media system：
        * 例如影音串流平台、onlineTV
        * 特徵：使用壓縮技術
    * handheld system：
        * 例如：Android、ios、PDA、
        * 顧名思義是攜帶方便的裝置，特徵是短小輕薄，功能幾乎都是模組化的，電池效能和小螢幕的顯示很重要，slow processor(因為memory有限)

    * 原本的作法：在搬移過程中必須透過while中的peek()反覆檢查buffer狀態，看還有沒有空位，但在這個過程中CPU會被霸佔
    * 解決方法：interrupt(又稱為trap或signal)
        * CPU是event driven的：可以隨時打斷CPU，叫他去做別的事---CPU是被動被呼叫的
        * device controller在搬移資料的時候，CPU能同時去處理別的事，不用等資料搬完。當搬移結束時，再發出interrupt通知CPU(告知資料搬移的狀態、搬到memory的什麼位置等)
        * 由於CPU會中途被打斷，因此也要記下被打斷前原本的程式執行到哪
        * interrupt被觸發時機：
            * 使用者點下滑鼠、按下鍵盤等 (hardware routine)
            * 程式執行出現錯誤時，透過interrrupt印出錯誤訊息(software routine)
            * system call時，一定會觸發interrupt
            * user program會先被編譯完，link到system library的對應function，變成system call再觸發interrupt(?)
        * 目前所有電腦系統的CPU都是interrupt driven的
            * 分成software和hardware的interrupt。hardware叫signal，software叫trap
        
        * 當interrupt時，CPU會先去看interrupt vector，找到對應的funtcion pointer
        * 如果是signal，CPU要怎麼知道是誰發出的interrupt?
        當I/O裝置連接電腦的port時，port上會有一個寫死的欄位，像幫裝置編號一樣紀錄，以便在interrupt產生的時候讓CPU辨識是哪個裝置
        * interrupt的overhead：需要花更多儲存空間紀錄如CPU原本的程式執行到哪、誰發出interrupt等資訊
    #### interrupt流程：
    ![](https://i.imgur.com/XRmJNBi.jpg)
