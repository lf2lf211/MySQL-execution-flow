# MySQL-execution-flow



[TOC]

# 作者
[lf2lf211](https://github.com/lf2lf211)

# MySQL SQL Query 過程(待整理)

## 參考 
# MySQL SQL insert過程(待整理)

## 參考 
# MySQL SQL update 過程(整理版)

## 名詞解釋

### **redolog**

重做日誌，屬於物理日誌，上面儲存的是數據庫中最終內容，有固定的大小，可以循環讀寫，一般設置 innodb_flush_log_at_trx_commit为1，表示commit事務時將redolog上面的數據刷入到磁碟，具有兩個狀態分別是 prepare和commit，在MYSQL重起恢復時會根據commit狀態恢復數據。

### **binlog**

歸檔日誌，屬於邏輯日誌，上面儲存的是最初的修改邏輯，可以簡單的理解為sql語句，可以追加寫，一般設置 sync_binlog為1，表示commit事務時將binlog上面的數據刷入到磁碟進行歸檔。數據恢復和同步都是通過binlog來實現。

### **double write buffer**
執行過程
![](https://i.imgur.com/Ib3ozsY.png)

恢復過程
![](https://i.imgur.com/bduFhND.png)



### **元數據(元資料)**

元數據描述數據的結構和意義。切記：元數據是抽象概念，具有上下文，在開發環境中有多種用途。
元數據是抽象概念，當人們描述現實世界的現象時，就會產生抽象信息，這些抽象信息便可以看作是元數據。例如，在描述風、雨和陽光這些自然現象時，就需要使用"天氣"這類抽象概念。還可以通過定義溫度、降水量和濕度等概念對天氣作進一步的抽象概括。

[詳細可參考](https://jinnan80-gmail-com.iteye.com/blog/333974) 

  
### **排它锁**
排它鎖與共享鎖相對應，就是指對於多個不同的事務，對同一個資源只能有一把鎖。與共享鎖類型，在需要執行的語句後面加上for update就可以了

### **共享鎖**
共享鎖指的就是對於多個不同的事務，對同一個資源共享同一個鎖。相當於對於同一把門，它擁有多個鑰匙一樣。就像這樣，你家有一個大門，大門的鑰匙有好幾把，你有一把，你女朋友有一把，你們都可能通過這把鑰匙進入你們家，這個就是所謂的共享鎖。
剛剛說了，對於悲觀鎖，一般數據庫已經實現了，共享鎖也屬於悲觀鎖的一種，那麼共享鎖在mysql中是通過什麼命令來調用呢。通過查詢資料，了解到通過在執行語句後面加上lock in share mode就代表對某些資源加上共享鎖了。

| 連線 |
| -------- | 
| begin;     |
| SELECT * from city where id = "1"  lock in share mode;    |
[其他鎖](https://blog.csdn.net/puhaiyang/article/details/72284702) 
[補充](https://blog.csdn.net/bohu83/article/details/82493737) 

### **undo log**
undo log有兩個作用：提供回滾和多個行版本控制(MVCC)。
在數據修改的時候，不僅記錄了redo，還記錄了相對應的undo，如果因為某些原因導致事務失敗或回滾了，可以藉助該undo進行回滾。
### **undo tablespace**
當進行update record期間，還未下commit 時，table中舊的資料會被放在undo tablespace，如果執行rollback，即可將舊資料還原，但如果做完commit後，資料就會被寫到redo中。
### **MVCC** 
Multi-Version Concurrency Control多版本並發控制，MVCC 是一種並發控制的方法，一般在數據庫管理系統中，實現對數據庫的並發訪問；在編程語言中實現事務內存。
### 髒頁
髒頁是指數據庫實例內存的數據比磁盤的數據要新。

```sql
 update T set value = value+1 where ID =2
```
圖中，深綠色背景為在執行器中執行(server層)，淺色則是在 InnoDB引擎中執行。

![](https://i.imgur.com/6oXnG6V.png)






## 執行順序


1. 客戶端通過TCP連接發送連結請求到mysql連接器，連接器會對該請求進行權限驗證及連接資源分配（max_connections，8小時超時）。

2. 建立連接後客戶端發送一條語句 mysql收到該語句後 通過命令分發氣判斷其是否是一條更新語句 如果是則直接發送給分析器做語法分析。

3. 分析器(Parser)階段 mysql檢查語法以及運算符號是否正確 正確則進入下個階段。

4. 預處理器(Preprocessor)階段，檢查權限以及其他額外語意是否合法，如資料表數據列是否存在以及正確。

5. 語句檢查完畢後，會將與據傳遞給優化器進行優化，預測出所有方式所消耗成本，挑出最小成本方法，且生成執行計畫。

6. 執行器會根據生成的執行計畫open table，此時會先查看該表上是否有元數據排他鎖(如果有元數據共享鎖則無影響)，如果有元數據排他鎖，則事務被阻塞，進入等待狀態(由lock_wait_timeout决定，默認是為一年)，直到被釋放，如果無元數據鎖或是有元數據共享鎖，則該事務在表上加上元數據共享鎖。

7. 進入引擎層(Storage Engine)（默認[innodb](/n-txVolES_m86fcp6BKv8Q)），會去innodb_buffer_pool裡面的data dictionary得到表的相關訊息，再根據訊息去innodb_buffer_pool裡面的lock info查看是否有相關的鎖訊息，如果有則等待(因為要加排他鎖)，如果沒有則增加排他鎖，更新lock info。

8. 取讀取相關數據頁到innodb_buffer_pool中（如果數據頁本身就在緩存中，則不用從硬盤讀取），將頁中的原始數據保存到undo log buffer中（undo log buffer會進行刷盤操作寫入到undo tablespace中）。


9. 在innodb_buffer_pool中將相關頁面更新，該頁變成髒頁（髒頁會進行刷盤操作寫入所屬表空間中）頁面修改完成後，會把修改後的物理頁面保存到redolog buffer中（redolog buffer會進行刷盤操作寫入到redo tablespace中）


10. 如果開啟binlog，則更新數據的邏輯語句也會記錄在binlog_cache中（binlog會進行刷盤操作寫入到binlog file 中）。

11. 如果該表上有二級索引並且本次操作會影響到二級索引，則會把相關的二級索引修改寫入到innodb_buffer_pool中的change buffer裡（change buffer 會以相關參數定義的規則進行刷盤操作寫入所屬表空間中）


12. 到這裡 前期的準備工作到此已經做完了，之後便是事務的commit或者rollback操作。一般情況下執行的是commit操作。

13. 執行commit操作後，由於要保證redolog與binlog的一致性，redolog採用2階段提交方式。將undo log buffer及redo log buffer刷盤，並將該事務的redolog標記為prepare狀態。

14. 然後將binlog_cache數據刷盤，如果開啟了主從架構，此時會將binlog_cache中的信息通過io線程發送給從機，如果開啟了半同步複製則需要等待從機落盤（relay log）並反饋。如果是異步複製則無需等待(默認是異步複製)

15. binlog落盤完成後，再將redolog中該事務信息標記為commit，釋放相關鎖資源。一個更新的操作已經完成，返回給客戶端成功更新提示。

16. 標記undolog中該事務修改頁的原始快照信息為delete，當無其他事務引用該原始數據時(MVCC)，再將其刪除 如果此時觸發了髒頁刷盤操作，會先將髒頁寫入到double write buffer中然後再寫到其所在表空間的相應位置。
17. 防止寫入過程中出現斷頁，因為mysql頁面默認為16K，linux操作系統最大為4K，如果寫了8K時系統掛了，這個數據頁將不完整，標記為損壞



## 為什麼要分開寫redolog呢？兩階段提交？

是為了保證以上所有的過程中，如果出現MySQL實例奔潰的情況，都不會導致事務的丟失或異常。

+ 具體介紹：

  + 如果是在redolog prepare之前crash，就是還沒寫任何日誌，那麼事務就不存在，在恢復後從redolog和binlog中都不會有任何的問題。

  + 如果寫完redolog 的prepare出現了crash，那麼恢復時，通過redolog和binlog的對比，會發現只要有一個prepare中的日誌，那麼會將事務進行回滾，保證redolog和binlog的統一。

  + 如果寫完binlog後出現crash，那麼恢復時，會進行根據binlog日誌對redolog進行補償，對redolog之前prepare的記錄修改為commit狀態，事務得到保證。

  + 最後commit後crash，redolog和binlog都是正常的。




## 參考 
 > [MySQL中一条SQL语句的执行过程](https://blog.csdn.net/finalkof1983/article/details/84450896)
 > [name= 一梦如是YFL] [time= Nov 24, 2018 18:35 PM]
 > [【Mysql】 update语句更新原理](https://blog.csdn.net/w372426096/article/details/88057365)
 > [name=  Franco蜡笔小强][time= Mar 01, 2019 14:44 PM]
 > [MySQL -update语句流程总结](https://blog.csdn.net/weixin_34161064/article/details/87986896)
 > [name=  njit_peiyuan ][time= Dec 29, 2018 03:29 AM]
