`vi`로 파일을 수정하려 했더니 다음과 swap 파일을 발견했다는 오류가 뜹니다.

```shell
E325: ATTENTION
Found a swap file by the name "inventory/mycluster/.inventory.ini.swp"
          owned by: ubuntu   dated: Sun May  9 09:45:13 2021
         file name: ~ubuntu/kubespray/inventory/mycluster/inventory.ini
          modified: YES
         user name: ubuntu   host name: instance-1
        process ID: 18727
While opening file "inventory/mycluster/inventory.ini"
             dated: Sun May  9 11:39:45 2021
      NEWER than swap file!
(1) Another program may be editing the same file.  If this is the case,
    be careful not to end up with two different instances of the same
    file when making changes.  Quit, or continue with caution.
(2) An edit session for this file crashed.
    If this is the case, use ":recover" or "vim -r inventory/mycluster/inventory.ini"
    to recover the changes (see ":help recovery").
    If you did this already, delete the swap file "inventory/mycluster/.inventory.ini.swp"
    to avoid this message.
Swap file "inventory/mycluster/.inventory.ini.swp" already exists!
[O]pen Read-Only, (E)dit anyway, (R)ecover, (D)elete it, (Q)uit, (A)bort: 
```



#### Swap 파일이란?

> vi 편집기는 작업을 하는 동안 비정상적인 종료 등을 대비하기 위해 파일을 임시 저장합니다. 현재 작업중인 파일에 `.swp`을 붙이고 숨김 파일로 만들어 임시 저장하는데, 이것이 **스왑 파일**입니다. 즉, **vi 편집기의 작업 내용을 임시 저장하는 파일**인 것이죠.

정상적인 방법으로 vi 편집기로 하는 작업을 완료하면 스왑 파일은 자동으로 삭제되는데, 네트워크가 끊기거나 강제 종료하는 등 작업이 비정상적으로 종료되면 스왑 파일은 삭제되지 않고 남아있게 됩니다.



제가 수정하려는 파일인 `inventory.ini`이 `inventory.swp`보다 최신인 것이 문제라고 하네요. 오류 메세지에서 권장하는 방법을 따라가 봅니다.



1. 다른 프로그램이 해당 파일을 수정 중일 수 있다고 하네요. 다른 프로세스가 작업 중인지 다음과 같은 명령어로 확인합니다.

   ```shell
   $ ps -ef | grep inventory/mycluster/inventory.ini
   ubuntu 27780 20788  0 12:46 pts/0    00:00:00 grep --color=auto inventory/mycluster/inventory.ini
   ```

   `grep` 말고는 사용 중인 프로세스가 없네요.

   

2. 그렇다면 edit session이 충돌난 것이니 `(R)ecover` 옵션으로 복구하라고 합니다.

   ```shell
   Using swap file "inventory/mycluster/.inventory.ini.swp"
   Original file "~/kubespray/inventory/mycluster/inventory.ini"
   E308: Warning: Original file may have been changed
   Recovery completed. You should check if everything is OK.
   (You might want to write out this file under another name
   and run diff with the original file to check for changes)
   You may want to delete the .swp file now.
   Press ENTER or type command to continue
   ```

   복구 후 `inventory.ini` 파일을 수정하려 하면 위와 같은 오류 메세지가 발생합니다. `swp` 파일을 먼저 삭제하고 수정하면 정상적으로 변경됩니다!