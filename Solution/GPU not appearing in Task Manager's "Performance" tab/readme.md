### 請以"系統管理員身分執行"開啟"命令提示字元"，然後複製以下第一步全部貼上後 Enter 再貼上第二步後 Enter，而後將工作管理員重新開啟。
### Please execute "Open Command Prompt" as "System Administrator", then copy the first step below, paste all of them, enter, paste the second step, enter, and then reopen the work administrator.

![image](https://user-images.githubusercontent.com/55220866/166925245-be8136a0-015a-4d2b-87f1-7b6f11a13e06.png)

```
─── 第一步 ───
cd c:\windows\system32
lodctr /R
cd c:\windows\sysWOW64
lodctr /R

─── 第二步 ───
WINMGMT.EXE /RESYNCPERF
```
