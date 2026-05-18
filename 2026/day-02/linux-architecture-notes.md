1. Core Components of Linux

 #Kernel
  -The kernel is the core part of Linux.
  -It manages:
    CPU
    Memory
    Devices
    Filesystems
    Processes
    a communication channel between hardware and software.

 #shell
  -Area where user applications run.
  -Examples:
   Bash shell
   Commands like ls, cp, mkdir

 #Init / systemd
  -First process started by the kernel (PID 1).
  -In modern Linux, systemd is the init system.
  -Responsible for:
    Starting services
    Managing background processes
    
2. Process Creation and Management
 #How Processes Are Created
  -A process starts when you give command.
  -Linux uses:
    fork() → creates a new process
    exec() → loads the program into memory
 #Process States
    Running → currently exicuting
    Sleeping → waiting for input
    Stopped → doing nothing 
    Zombie → finished execution but parent has not collected status
    Orphan → parent process ended before child process
 #Process Management Commands
    ps → shows running processes
    top → real-time process monitoring
    kill → terminate a process
    jobs → shows background jobs
    nice → change process priority

3. What systemd Does and Why It Matters
 
 #systemd Functions
  -Starts services during boot
  -Restarts failed services automatically

 #Why systemd Matters
  -Faster boot process
  -loads other processes
4. 5 Daily Linux Commands
	
 ls	
 cd	
 pwd
 mkdir
 touch	

