## 介绍

本实验的目的是使你熟悉**进程控制**和**信号处理**的概念。你需要实现一个简单的支持作业控制的Unix shell 程序。

## 实验内容

本次实验就是自己实现一个shell，不过不是从头自己写，是完成几个函数的实现。

- eval函数包含了shell的主要操作，读取命令行，fork子进程，执行
- builtin_cmd函数包含了处理内置命令行函数的操作，包括quit, fg, bg, jobs
- do_bgfg函数用来处理fg和bg操作，主要是对进程变换状态以及发送SIGCONT信号
- waitfg函数用来等待前台程序结束，因为回收子进程交给了sigchld_handler来做，所以waitfg只要用sleep写一个忙等待来等到前台进程结束。
- 三个信号的操作函数也是要重点实现的内容。

## 实验思路

这次实验比较难，有很多需要注意的细节，我开头没有理解透整个shell的操作，外加要求没有完全看清，导致前期做的非常困难。

- 子进程的回收在sigchld_handler中来做，waitfg只要忙等待前台进程结束就行。
- 后台进程运行之后放着不用管就行，父进程不用等到后台子进程运行结束，直接可以继续后续操作（开头写错了，导致语句输出顺序出现错误，以及一些后台进程被运行完后才开始执行后续操作）
- 熟练使用waitpid中的参数，WNOHANG|WUNTRACED组合在一起用是立即返回的意思。
用WIFEXITED(status)，WIFSIGNALED(status)，WIFSTOPPED(status)等来补获终止或者被停止的子进程的退出状态（主要用于sigchld_handler中）
- 对于每个fork的子进程，执行setgpid(0, 0)，这样就会以子进程号单独开一个进程组，也可以方便的使用kill(-pid, SIGNAL)来把信号发到pid所在的整个进程组。
- 对于调用sigchld_handler回收子进程时，必须用掩码阻塞SIGCHLD信号，防止子进程还没执行，就回收了。
- 对于每个子进程，加入joblist之后，在结束时在sigchld_handler中有三种操作，一种是正常结束，deletejob，一种是被信号终止了，也要deletejob，还有一种是被信号停止了，不用delete，只要修改job的状态。
- 最重要的是，不要死磕一个trace，其实有的trace，需要你实现很多内容，可能实现了前一个，后面几个都完成了，所以需要仔细分析所有的需求。
- 看一下给的其他程序，myspin，主要是一个延迟函数，所以在前台运行，就要等它延迟结束，如果在后台运行，就可以在exevce之后不管。mystop是用来发送停止信号，myint是用来发送终止信号。

## 函数实现

### eval

```C
/*
 * eval - Evaluate the command line that the user has just typed in
 *
 * If the user has requested a built-in command (quit, jobs, bg or fg)
 * then execute it immediately. Otherwise, fork a child process and
 * run the job in the context of the child. If the job is running in
 * the foreground, wait for it to terminate and then return.  Note:
 * each child process must have a unique process group ID so that our
 * background children don't receive SIGINT (SIGTSTP) from the kernel
 * when we type ctrl-c (ctrl-z) at the keyboard.
*/
void eval(char *cmdline)
{
    char *argv[MAXARGS];   // 参数数组
    char buf[MAXLINE];
    int bg;                 // 是否后台运行
    pid_t pid;
    sigset_t mask_one, prev_one;

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);
    if(argv[0] == NULL) return;

    if(!builtin_cmd(argv)){
        sigprocmask(SIG_BLOCK, &mask_one, &prev_one);//block SIGCHLD
        if((pid = fork()) == 0){   // 在子进程中fork返回0
            sigprocmask(SIG_SETMASK, &prev_one, NULL);//unblock SIGCHLD
            if(setpgid(0, 0) < 0){
                printf("setpgid error");
                exit(0);
            }
            if(execve(argv[0], argv, environ) < 0){
                printf("%s: Command not found.\n", argv[0]);
                exit(0);
            }
        }
        else{
            if(bg)
                addjob(jobs, pid, BG, cmdline);
            else
                addjob(jobs, pid, FG, cmdline);

            sigprocmask(SIG_SETMASK, &prev_one, NULL);//unblock SIGCHLD

            if(bg){
                printf("[%d] (%d) %s",pid2jid(pid), pid, cmdline);
            }
            else{
                waitfg(pid);
            }
        }
    }
    return;
}
```

eval函数，没啥多说的，需要注意的上面都讲过了，主要就是对每个操作是bg还是fg要分清楚操作，加掩码阻塞SIGCHLD信号的代码书本上有说明，然后就是父进程，对于前后台进程，加入joblist要注意状态BG和FG。然后后台进程输出一句话，前台进程需要等待子进程运行结束。

### builtin_cmd

```C
/*
 * builtin_cmd - If the user has typed a built-in command then execute
 *    it immediately.
 */
int builtin_cmd(char **argv)
{
    if(!strcmp(argv[0], "quit"))
        exit(0);
    if(!strcmp(argv[0], "jobs")){
        listjobs(jobs);
        return 1;
    }
    if(!strcmp(argv[0], "bg")){
        do_bgfg(argv);
        return 1;
    }
    if(!strcmp(argv[0], "fg")){
        do_bgfg(argv);
        return 1;
    }
    return 0;     /* not a builtin command */
}
```

判断是否是内置命令行函数，如果是，就执行相应操作，如果不是，就返回到eval中，创建子进程执行操作。（listjobs我开头没有弄好子进程回收，导致前台程序还留在 joblist中，然后输出时会输出前台进程。要考虑到前台进程运行结束之后，才能执行jobs命令，所以绝对不会出现状态是前台的进程）

### do_bgfg

```C
/*
 * do_bgfg - Execute the builtin bg and fg commands
 */
void do_bgfg(char **argv)
{
    int id;
    struct job_t *job;
    if(argv[1] == NULL){
        printf("%s command requires PID or %%jobid argument\n",argv[0]);
        return;
    }
    /*先得到job ID*/
    if(argv[1][0]=='%'){
        if(argv[1][1] >='0'&& argv[1][1] <='9'){
            id=atoi(argv[1]+1);
            job = getjobjid(jobs, id);
            if(job==NULL){
                printf("%s: No such job\n",argv[1]);
                return;
            }
        }
        else {
            printf("%s: argument must be a PID or %%jobid\n",argv[0]);
            return;
        }
    }
    else{
        if(argv[1][0] >='0'&& argv[1][0] <='9'){
            id = atoi(argv[1]);
            job = getjobpid(jobs, id);
            if(job == NULL){
                printf("(%s): No such process\n",argv[1]);
                return;
            }
        }
        else {
            printf("%s: argument must be a PID or %%jobid\n",argv[0]);
            return;
        }
    }

    kill(-(job->pid), SIGCONT);
    if(!strcmp(argv[0], "bg")){
        job->state = BG;
        printf("[%d] (%d) %s",job->jid, job->pid, job->cmdline);
    }
    else {
        job->state = FG;
        waitfg(job->pid);
    }

    return;
}
```

处理bg和fg命令的函数，主要是解析参数，判断是否会出现命令错误。找到需要操作的进程，然后用kill函数对它发出SIGCONT信号，因为前面创建进程的时候都是用setgpid，每个进程在单独 一个进程组，所以这边kill用的非常舒服。

进程继续执行之后，只需要给后台进程改一下state，前台进程的话，也要改state，还要waitfg

### waitfg

```C
/*
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    while(pid==fgpid(jobs)){
        sleep(0);
    }
    return;
}
```

等到前台进程执行结束的函数。调用自带的fgpid函数就能知道当前前台进程号，因为不需要负责子进程回收，只要判断当前进程号是否是前台进程，如果前台进程号变化，就说明结束了，就跳出循环，否则忙等待。

### sigchld_handler



```C
/*
 * sigchld_handler - The kernel sends a SIGCHLD to the shell whenever
 *     a child job terminates (becomes a zombie), or stops because it
 *     received a SIGSTOP or SIGTSTP signal. The handler reaps all
 *     available zombie children, but doesn't wait for any other
 *     currently running children to terminate.
 */
void sigchld_handler(int sig)
{
    pid_t pid;
    int status;
    while((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0){
        if(WIFEXITED(status)){
            deletejob(jobs, pid);
        }
        if(WIFSTOPPED(status)){
            struct job_t *job = getjobpid(jobs, pid);
            int jid = pid2jid(pid);
            printf("Job [%d] (%d) stopped by signal %d\n",jid, pid, WSTOPSIG(status));
            job->state = ST;
        }
        if(WIFSIGNALED(status)){
            int jid = pid2jid(pid);
            printf("Job [%d] (%d) terminated by signal %d\n",jid, pid, WTERMSIG(status));
            deletejob(jobs, pid);
        }
    }
    return;
}
```

该函数是SIGCHLD信号的响应函数。

该函数中运用waitpid（）函数并且用WNOHANG|WUNTRACED参数，该参数的作用是判断当前进程中是否存在已经停止或者终止的进程，如果存在则返回pid，不存在则**立即返回**

通过另外一个&status参数，我们可以判断返回的进程是由于什么原因停止或暂停的。

对于一个子进程结束，主要有3种原因，正常运行结束，收到SIGINT终止，收到SIGTSTP停止。

- WIFEXITED(status):
如果进程是正常返回即为true，什么是正常返回呢？就是通过调用exit（）或者return返回的
- WIFSIGNALED(status)：
如果进程因为捕获一个信号而终止的，则返回true
- WIFSTOPPED(status)：
如果返回的进程当前是被停止，则为true

### sigint_handler

```C
/*
 * sigint_handler - The kernel sends a SIGINT to the shell whenver the
 *    user types ctrl-c at the keyboard.  Catch it and send it along
 *    to the foreground job.
 */
void sigint_handler(int sig)
{
    pid_t pid = fgpid(jobs);
    if(pid!=0){
        kill(-pid,sig);
    }
    return;
}
```

给当前前台进程发送SIGINT信号，用kill发送比较好

### sigtstp_handler

```C
void sigtstp_handler(int sig) 
{
    pid_t pid = fgpid(jobs);
    if(pid!=0){
        kill(-pid,sig);
    }
    return;
}
```

和上面类似。

## 总结

shell lab总体来说难度较大，因为开头没有理解shell的流程，对很多命令的运行情况不熟悉，所以对出来的结果很懵逼。不过后来慢慢搞懂了运行顺序和情况，信号的作用，进程的运行情况和周期，以及运行过程，就知道应该注意的点在哪里了，然后一步步的实现就可以了，这个实验让我对shell和进程的理解加深了很多。




