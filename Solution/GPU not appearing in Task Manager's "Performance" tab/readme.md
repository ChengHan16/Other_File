### 請以系統"管理執行"開啟"命令提示字元"，然後複製以下第一步全部貼上後 Enter 再輸入第二步，而後將工作管理員重新開啟。
### Please open the "Command Prompt" with the system "Administrative Execution", then copy and paste the following first step, Enter and then enter the second step, and then restart the work administrator.
```
─── 第一步 ───
cd c:\windows\system32
lodctr /R
cd c:\windows\sysWOW64
lodctr /R

─── 第二步 ───
WINMGMT.EXE /RESYNCPERF
```
