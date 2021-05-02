任何进程都涉及到数据处理以及存储/显示，一个进程的执行通常包含很多个离散的时间块，比如涉及到CPU的计算`cpu_burst`，涉及到IO的`io_burst`。本lab假定只有一个CPU核，意味着在同一时间下，只有一个进程可以运行，而且每个进程都是单线程的。但是可能会有很多个进程在等待运行，所以在同一个时间线下，会包含三种类型的进程：(1). 使用CPU进行计算的，(2)进行IO的。（3）等待使用CPU的。

每个进程包含四个参数：

* Arrival Time(AT)
* Total CPU Time(TC), 该进程所需要的CPU时间
* CPU burst (CB), upper limit of compute demand
* IO Burst(IO), upper limit of IO time

一个进程会有如下四种状态：

![image-20210502101531474](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210502101531474.png)

一个进程在到达的时候，首先是进入`CREATED`状态然后进入`READY`，当它被`scheduler`安排的时候，它转变到`RUNNING`，然后一个`cpu_burst`从`[1,...CB]`中选出来作为当前CPU的处理时间，如果当下剩下的执行时间小于`cpu_burst`，则将`cpu_burst`置为该执行时间。然后它需要处理IO，进入`BLOCKED`状态，与`cpu_burst`一样，`io_burst`也是从`[1,...IO]`中随机选择的。如果有preemtion发生的话，则

需要注意的是可能会有一些进程的AT相同，则根据它们在input中出现顺序来处理。还有一种情况是，不同的事件的结束时间相同，比如进程1的IO操作在X时间操作完成，同时进程2的CPU操作也在X时间完成，那么根据它们的产生时间来顺序处理。

我们需要维护两个队列，一个是scheduler的队列，里面放的是进程，当进程从`xxx->READY`的时候，它会被放入队列中。

```c++
class Process
{
    public:
        int pid; // process id
        State state; // current state
        int AT; //Arrival time
        int TC; // Total CPU time
        int CB; // CPU Burst
        int IO; // IO Burst
        int static_priority; // static priority
        int dynamic_priority; // dynamic priority
        int cpu_burst_rem; // cpu burst remaining
        int state_ts; // time when moved to current state
        int CW; // CPU waiting
        int FT; // Finish time
        int IT; // IO time
        int RT; // Response time
        int TT; // Turnaround time
        int CT; // CPU time or total time when cpu is used
};
```

进程中存储了上述的一些变量。

我们还需要维护一个存储事件的队列，事件中存储了它发生的时间，它所指的进程，它的旧状态和转移。

```c++
class Event
{
    public:
        int timestamp;
        Process *process;
        Trans_to transition;
        State old_state;
};
```

初始时，进程队列和事件队列都为空。input中按照时间顺序存储了不同的的进程和其对应的参数，首先读入所有input中的进程，将其一个一个插入到事件队列中。

然后开始模拟scheduler的过程，首先从事件队列中抛出一个事件，它是`CRETED->READY`，将其所指的进程加入到进程队列中（其实就是readyQ），如果事件队列中的下一个事件的时间和当前时间相同，则继续抛出事件。否则则，因为当前没有进程在运行，我们需要从进程队列中抛出新的进程来运行（创造新的事件`READY->RUNNING`将其加入到事件队列中），如果进程队列此时为空，说明没有进程处在READY中，我们需要从事件队列中抛出事件（可能有`BLOCKED->READY`）之类的。具体详见下面代码：

```c++
void simulation(DES *des)
{
    Process *proc;
    int current_time;
    int time_in_prev_state;
    bool call_scheduler;
    int cpu_burst;
    Process *proc_running = NULL;
    Event *e;

    while((e = des->getEvent())!=NULL)
    {
        proc = e->process;
        current_time = e->timestamp;
        time_in_prev_state = current_time - proc->state_ts;

        proc->state_ts = current_time;

        switch(e->transition)
        {
            case TRANS_TO_READY:{
                                    string old;
                                    if(e->old_state==CREATED)
                                        old = "CREATED";
                                    else if(e->old_state==BLOCKED)
                                        old = "BLOCK";
                                    else if(e->old_state==RUNNING)
                                        old = "RUNNG";
                                
                                    if(verbose)
                                        cout << current_time << " " << proc->pid << " " << time_in_prev_state << ": " << old << " -> " << "READY" << endl;

                                    if(e->old_state==BLOCKED)
                                    {
                                        proc->dynamic_priority = proc->static_priority - 1;
                                        proc->IT += time_in_prev_state;
                                        io_cnt--;
                                        if(io_cnt ==0)
                                        {
                                            total_io_utilization += current_time - io_start;
                                        }
                                        // if(verbose)
                                        //     cout << current_time << " " << proc->pid << " " << time_in_prev_state << ": " << old << " -> " << "READY" << endl;
                                    }
                                    else if(e->old_state==RUNNING)
                                    {
                                        proc->CT += time_in_prev_state;
                                        total_cpu_utilization += time_in_prev_state;
                                    }

                                    if(isPrePrio && proc_running!=NULL)  
                                    {
                                        if(proc->state==BLOCKED || proc->state==CREATED)
                                        {
                                            int cts;
                                            for(list<Event>::iterator i = des->event_queue.begin(); i != des->event_queue.end();i++)
                                            {
                                                Event e = *i;
                                                if(e.process==proc_running)
                                                {
                                                    cts = e.timestamp;
                                                }
                                            }

                                            if(proc->dynamic_priority > proc_running->dynamic_priority && current_time < cts)
                                            {
                                                des->deleteFutureEvents(proc_running);
                                                Event e(current_time, proc_running, TRANS_TO_PREEMPT);
                                                e.old_state = RUNNING;
                                                des->putEvent(e);
                                            }
                                        }
                                    }

                                    scheduler->addProcess(proc);
                                    call_scheduler = true;
                                    break;
                                }
            case TRANS_TO_RUN:  { 
                                    proc->state = RUNNING;
                                    proc_running = proc;
                                    if(proc->cpu_burst_rem>0)
                                    {
                                        cpu_burst = proc->cpu_burst_rem;
                                    }
                                    else
                                    {
                                        cpu_burst = myrandom(proc->CB);
                                       
                                        if(cpu_burst > proc->TC - proc->CT)
                                        {
                                            cpu_burst = proc->TC - proc->CT;
                                        }
                                        
                                    }

                                    proc->CW += time_in_prev_state;
                                    total_cw += time_in_prev_state;
                                    proc->cpu_burst_rem = cpu_burst;

                                    if(verbose)
                                    {
                                        printf("%d %d %d: %s -> %s cb=%2d rem=%d prio=%d\n",current_time, proc->pid, time_in_prev_state, "READY", "RUNNG" ,cpu_burst, proc->TC-proc->CT, proc->dynamic_priority);
                                    }

                                    if(cpu_burst <= quantum)
                                    {
                                        Event e(current_time+cpu_burst, proc, TRANS_TO_BLOCK);
                                        e.old_state = RUNNING;
                                        des->putEvent(e);
                                    }
                                    else
                                    {
                                        Event e(current_time+quantum, proc, TRANS_TO_PREEMPT);
                                        e.old_state = RUNNING;
                                        des->putEvent(e);
                                    }

                                    break;
                                }  
            case TRANS_TO_BLOCK:    {
                                        proc_running = NULL;
                                        proc->CT += proc->cpu_burst_rem;
                                        total_cpu_utilization += time_in_prev_state;
                                        proc->cpu_burst_rem -= time_in_prev_state;
                                        

                                        if(proc->TC-proc->CT == 0)
                                        {
                                            proc->state = DONE;
                                            proc->FT = current_time;
                                            proc->TT = current_time - proc->AT;

                                            last_event_finish = current_time;
                                            total_tt += proc->TT;   

                                            if (verbose)
                                            {
                                                 printf("%d %d %d: Done \n", current_time, proc->pid, time_in_prev_state);
                                            }

                                        }
                                        else
                                        {
                                            proc->state = BLOCKED;

                                            if(io_cnt==0)
                                                io_start = current_time;

                                            io_cnt++;

                                            int ib = myrandom(proc->IO);
                                            Event e(current_time+ib, proc, TRANS_TO_READY);
                                            e.old_state = BLOCKED;
                                            
                                            des->putEvent(e);
                                            if(verbose)
                                            {
                                                printf("%d %d %d: %s -> %s ib=%2d rem=%d \n",current_time, proc->pid, time_in_prev_state,  "RUNNG", "BLOCK", ib, proc->TC-proc->CT);
                                            }
                                        }
//                                        scheduler->printRunQueue();
                                        call_scheduler = true;
                                        break;
                                        
                                    }
            case TRANS_TO_PREEMPT:  {
                                        proc->cpu_burst_rem -= time_in_prev_state;
                                        proc->CT += time_in_prev_state;

                                        if(verbose)
                                        {
                                            printf("%d %d %d: %s -> %s cb=%2d rem=%d prio=%d\n",current_time, proc->pid, time_in_prev_state, "RUNNG", "READY", proc->cpu_burst_rem, proc->TC-proc->CT, proc->dynamic_priority);
                                        }

                                        proc->state = PREMPT;
                                        proc_running = NULL;
                                        total_cpu_utilization += time_in_prev_state;
                                        scheduler->addProcess(proc);
                                        call_scheduler = true;
                                        break;
                                    }
        }
        des->rm_event();
        // cout << endl;
        // cout << " Event queue: " << endl;
        // des->printQueue();

        if(call_scheduler)
        {
            // cout << "\n here \n";
            if(des->getNextEventTime() == current_time)
            {
                continue;
            }

            call_scheduler = false;

            if(proc_running==NULL)
            {   
                proc_running = scheduler->getNextProcess();
                
                if(proc_running==NULL)
                    continue;

                Event e(current_time, proc_running, TRANS_TO_RUN);
                e.old_state = READY;
                des->putEvent(e);
            }
        }
        // cout << "Event Queue: "  << endl;;
        // des->printQueue();
    }
}
```

