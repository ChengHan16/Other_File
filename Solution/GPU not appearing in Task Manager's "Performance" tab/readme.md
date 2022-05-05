### 請以"系統管理員身分執行"開啟"命令提示字元"，然後複製下方指令，全部貼上後 Enter 而後將工作管理員重新開啟就有了。
### Please execute as "system administrator" to open "command prompt", then copy and paste all the following commands, then Enter and then restart the work administrator.
![image](https://user-images.githubusercontent.com/55220866/166925674-afdb1bcb-9cba-48a6-a295-7010896181b1.png)
```
cd c:\windows\system32
lodctr /R
cd c:\windows\sysWOW64
lodctr /R
```
