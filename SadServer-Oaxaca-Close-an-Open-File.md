## Oaxaca: Close an Open File

The file `/home/admin/somefile` is open for writing by some process. Close this file without killing the process.

`bash` has the file open with a **fd** of `77`:

```shell
admin@i-0a8158ef4cb3362f5:~$ ls
agent  openfile.sh  somefile

admin@i-0a8158ef4cb3362f5:~$ lsof somefile 
COMMAND PID  USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
bash    811 admin   77w   REG  259,1        0 272875 somefile
```

You can think of file descriptor as "Hey Kernel! open a file and give me a number". 

The command that triggered it, is `exec` inside `.bashrc`:

```shell
admin@i-0cbfc1b45f3b71168:~$ sudo grep -irln "somefile" /* 2> /dev/null
/home/admin/.bashrc
/home/admin/agent/check.sh
/home/admin/openfile.sh
^C
admin@i-0cbfc1b45f3b71168:~$ 
admin@i-0cbfc1b45f3b71168:~$ cat .bashrc | grep somefile
exec 77> /home/admin/somefile
```

Closing the file descriptor with `exec [fd]<&-`: 

```shell
admin@i-0a8158ef4cb3362f5:/proc/811$ ls -l fd; echo
total 0
lrwx------ 1 admin admin 64 Aug 10 12:30 0 -> /dev/pts/0
lrwx------ 1 admin admin 64 Aug 10 12:30 1 -> /dev/pts/0
lrwx------ 1 admin admin 64 Aug 10 12:30 2 -> /dev/pts/0
lrwx------ 1 admin admin 64 Aug 10 12:30 255 -> /dev/pts/0
l-wx------ 1 admin admin 64 Aug 10 12:30 77 -> /home/admin/somefile
admin@i-0a8158ef4cb3362f5:/proc/811$
admin@i-0a8158ef4cb3362f5:/proc/811$ exec 77<&-
```

`lsof` should return nothing:

```
admin@i-0a8158ef4cb3362f5:/proc/811$ cd ; lsof somefile

admin@i-0a8158ef4cb3362f5:~$ 
```

### Ref

- https://man7.org/linux/man-pages/man1/exec.1p.html