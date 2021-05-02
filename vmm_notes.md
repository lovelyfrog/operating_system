vmm: virtual memory manager

先回答几个问题

为什么需要cache?

为什么要引入virtual memory?

## 为什么需要cache

直接访问RAM比较慢，需要一个中间存放内存的地方，可以更快速的访问。而且程序通常只访问小部分的地址空间，而一个地址被访问，意味着它相邻的地址也很快会被访问(**spatial locality**)，而且它本身也可能很快再次被访问(**temporal locality**)。

![image-20210430213456273](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430213456273.png)

我们从访问某个地址空间时，在RAM中如果它存在，那么它连续的一个**block**（包含多个word（1个word由4个byte组成也就是32bits））会传递到cache中，这样就会高效的利用前面所说的两个locality。但是如何知道我要访问的地址在cache中呢？而且cache的大小是远小于RAM的，意味着很多个RAM上的block会映射到一个cache上的block，如何映射？以及当cache满时，采取何种策略踢出一个block以放入新的block？这是三个重要的问题，后面会一一满满回答。

###四个问题

地址的最后几位是offset field，用来标识一个block中的特定的bytes

![image-20210430215512583](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430215512583.png)

![image-20210430215524583](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430215524583.png)

上述是一个8 byte的block，它的地址空间是32位的，而每个地址对应的都是一个byte的数据。

假定一个cache共有C个bytes，一个block有K个bytes，那么该cache一共有C/K个block。

在一个block的地址中，除了offset之外的bits全部都是相同的，这些bits组成了**tag field**，用来标识该block是从哪来的。

![image-20210430220103029](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430220103029.png)

实际上在一个**cache slot**中，它包含了一整个data block(K * 8 个bits)，**tag bits**作为标识，还有一个**valid bit**(1 bit)来判定它是否是garbage。所以一个cache slot中共有`8K + tag + 1`个bits。

那么tag的作用就可以回答第一个问题。

对于第二个问题，我们可以看一个例子：**Fully Associative Caches**。

它其实是一种映射规则（后面我们也会看到不同的规则，如**Direct Mapped**），即每个memory中的block都可以映射到cache中的任意block。查看一个memory address的数据是否存在在cache中，需要查看所有的cache slots，看是否有一个slot与memory address的tag一致，而且valid bit为1，如果有，则该memory address存在于cache中，如果不存在则需要从RAM中拿数据到cache中。下图可以形象的感受这一过程。

![image-20210430222425249](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430222425249.png)

对于第三个问题，涉及到了**block replacement policy**。有很多中策略，最简单的比如随机选择，但该策略的效率可能不是很高，因为它的随机性可能会导致接下来会访问的block被替换掉。我们希望的是，替换掉我们未来最远才会用到的那个block，但是这是无法预知的，因此我们只能退而求其次，替换掉最近没怎么用过的block: **Least Recently Used (LRU)**

下图是一个cache的例子，它是一个64 Byte的地址空间

![image-20210430223901953](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430223901953.png)

cache中的数据不仅可以读，也可以写，如果我们对cache中的数据写了，如何保证它与RAM中数据的连续性？这是第四个问题。

首先我们最容易想到的策略是，一旦我们改了cache中的数据，我们就直接将其同步到RAM中(**write-through policy**)，但这样是很慢的。

更好的方法是，只对cache写，而当该block被移除时，将其更新到RAM中(**write-back policy**)，所以我们需要一个额外的`dirty bit`来标识其是否被写了。

![image-20210430225557966](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430225557966.png)

除了FA cache之外，还有其他类型的cache。

### Directed-Mapped Cache

还记得之前我们提到的FA cache，对于任意的memory block可以映射到任意的cache slot中。而**Directed-Mapped Cache**与之不同，每个memory block只能映射到有且仅有一个slot上。

它的好处在于，check的时候只需要看一个slot就可以了，而不像FA一样需要遍历所有的slot来check一个memory block是否存在。而且它不需要replacement policy。但是该方法不能最大化的利用cache的空间，可能会有很多空的slot没被使用。

这种映射规则就需要我们在除了offset field和tag field之外，多出一个**index field**，用于标识该memory address属于cache中的哪个index。

![image-20210430232120109](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430232120109.png)

有I个index bits就代表着cache中有$2^I$个slots。

同样的还是一个64 Byte的地址空间：

![image-20210430232408474](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430232408474.png)

其中offset-2bits, index-2bits, tag-2bits。

下面更直观的映射图：

![image-20210430232505708](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210430232505708.png)

当一个memory request: 001011发生的时候，首先看index field: 10，查看index为10的slot的valid bit是否为1,如果为1的话，查看它的tag 是否与001011的tag: 00匹配。

除了FA, DM之外还有一种**N-way set-associative**的cache，其实就是将cache分成一些set，每个set包含N个slots。而每个set对应的是index field。给定一个memory request，先看index field，与其对应的set，然后看在该set下的N个slots中是否有满足与memory request中的tag一致的slot，如果有且其valid bit为1的话，则request hit。

### 多级cache

AMAT: Average Memory Access Time，即平均存储访问时间

`AMAT = Hit time + Miss rate x Miss penalty`

多级cache的目标就是减小miss penalty。

![image-20210501094730024](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501094730024.png)

其中不同的cache侧重点不同。`L1 cache`关注low hit time，它的miss penalty可以通过更大的`L2 cache`来减小，所以即使它的miss rate很高，miss penalty也不至于很大。而`L2 cache,L3 cache `侧重于low miss rate，尽可能避免访问RAM。

两款芯片cache对比：

![image-20210501100349075](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501100349075.png)

## 为什么要引入virtual memory

在os context switch的时候，一个process可能会overwrite另一个process的数据，而我们希望不同的进程之间是不会相互干扰的。还有就是虽然RAM上可能有空间，但是空闲的空间是分散的，不能把一整个process完全连续的放进去，比如下图：

![image-20210501102756023](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501102756023.png)

为了解决上述的两个问题virtual memory被提出了。它给每个进程一个幻觉，就是只有它一个进程在使用整个存储空间。

* VM: 一个巨大的且每个进程相信只有它自己可以访问的私有空间
* Physical Memory(PM): 有限的RAM，所有进程共享其空间

那么一个最简单想法就是给予不同的进程相应的base 和 bound，虽然他们的virtual address的起点都从`0x0..0`开始，但是它加上base之后对应的就是RAM上真实的地址，而bound限制了它的大小。

![image-20210501104300245](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501104300245.png)

这种做法很简答直接，但是问题也很明显，如果我们想要更大的大于bound的空间，就比较难处理了。还有就是如果RAM碎片化，我们想要的空间无法连续的获得，这样就会浪费很多碎片化的空间。

解决方案就是`Paged Memory`，通过将PM和VM分成很多个大小相同的单元（`page`），使得在PM和VM中page size是相同的。

![image-20210501105543113](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501105543113.png)

可以发现，PM上进程的page之间不必连续，那么怎么从VM中连续的page映射到PM上的page呢？我们需要存储和维护一个`page table`，其中保存了每个VM上的page对应的PM上的page的映射关系。

![image-20210501105819833](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501105819833.png)

现在我们总结一下使用VM带来的好处：

* protection: 不同的进程之间有独立的私有的地址空间，这样保护了进程之间不受恶意干扰
* Demand paging: 由于VM是巨大的（可以看成无穷大的），使得计算机能够运行需要任意大小空间的程序（将所需的page不断的从disk传到RAM中）

但是代价就是VM到PM的地址转换（硬件上的操作）。

![image-20210501111506665](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501111506665.png)

那么接下来的问题就是具体这一转换是怎么操作的。

我们可以将virtual address和 physical address都分成两个部分，前面是Page number,后面是offset：

![image-20210501112613266](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501112613266.png)

![image-20210501112647728](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501112647728.png)

给定一个virtual address，我们取出他的page number，然后在page table中查找对应的physical page number，将offset与之合并就获得了physical address:

![image-20210501112901619](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501112901619.png) 

在page table的每个entry中，除了有VPN， PPN之外，还有一些其他的bit:

* ` Dirty` : 标识page最近是否被修改
* `valid`: 1标识该virtual page在physical memory中，0标识需要从disk中提取page，也就是Page Fault
* `Access Rights`：read, write, executable  

下图是完整的流程：

![image-20210501113827397](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501113827397.png)

思考一下这样是如何获得进程之间的保护的。在最原始的系统中，load/store所操作的是真实的physical address，这也就意味着任何的程序都可以操作任何地址，即使是它本不该操作的（比如os内核）。但是page table的使用，使得我们需要将vpn转换成ppn，我们只可以操作我们自己的page，其他的映射都是invalid的。

那么page table储存在哪里呢？存储在cache上是不太可能的，因为cache的大小是有限的，而page table有很多，且每个PT的大小与地址空间成正比。因此我们可以把PT放在RAM上。我们需要额外存储一个PT base使得我们可以在RAM中找到PT：

![image-20210501115046192](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501115046192.png)

有的时候可能我们只需要很少的page，但是却从RAM中加载了整个PT，这是非常耗时的，我们希望我们可以仅加载我们想要的那部分PT entries。这就涉及到了层级的PT

![image-20210501115612976](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501115612976.png)

如上图，我们只加载我们想要的那部分的PT。但是这会带来新的问题，我们每次都要从RAM上加载PT，而这里面有三层的PT，也就是需要访问三次RAM，这是非常耗时的。

我们可以借鉴cache的思想，创建映射关系的cache，不过这在VM中不叫cache而叫Translation Lookaside Buffer(TLB)。所以我们就可以先在TLB中找映射关系，如果找不到再从RAM中加载PT。

![image-20210501121428046](/Users/lovelyfrog/Library/Application Support/typora-user-images/image-20210501121428046.png)

而如果RAM中也没有的话(Page Fault)，则需要从disk中找，然后将其加载到RAM中，更新相应的PT entry，然后加载到TLB中。这一步的操作就会更加的费时间。

## nyu os lab3干了啥

在该lab中，需要模拟os的virtual memory manager，使用page table translation 将多个process的virtual address 映射到physical frame上。假定每个进程有64个virtual page，而physical page frame的数量由program option指定，最多128个frame。

每个process包含不同的**virtual memory areas (VMA)**:比如heap, stack, code,data之类的，而不同的VMA也具有不同的特征，是否是write_protected, file_mapped之类的。

 在本实验中，vma有4个参数特征：`start_vpage, end_vpage, write_protected, file_mapped`。

每个process都会给出一个instruction的序列，instruction的类型有：

* `c procid`: context switch to process #procid
* `r vpage`:  load/read on vpage of the currently running process
* `w vpage`: store/write on vpage of the currently running process
* `e procid`: exit current process

每个PTE都由`PRESENT/VALID, REFERENCED, MODIFIED, WRITE_PROTECT, PAGEDOUT`和physical frame的`frame_number`(PPN)组成。

每个进程都需要一个有64个PTE的page table，而且我们还需要维护一个全局的`frame_table`来记录从frame 到 proc.vpage的 reverse mapping。

对于每个instruction，我们首先要检查该PTE是否是present，如果是则直接操作就可以了，如果不是则会发生page fault，即frame_table上也就是physical memory中没有这样的page，需要从disk中提取这个page到frame_table中。因此我们需要在frame_table中选择一个victim_frame。因为在frame中我们保存了它的逆映射->proc:vpage，所以我们需要先unmap这个proc:vpage，如果该frame中的数据被修改过了，我们需要将它page out至swap device(OUT)，或者它是file_mapped，我们需要将其写回file中(FOUT)。那么这个新的frame可能从disk中来，也可能从swap device（IN）或者从file(FIN)中来。如果是从swap device来，说明它之前被page out 到swap space，所以我们只需检查它的`PAGEDOUT` bit就可以了。如果这个vpage它既没有paged out也不是file mapped的话，它对应的frame就应该仍然是zero filled content(ZERO)。代码如下：

```c++
PTE* curr_PTE = curr_proc->page_table[curr_inst->page_or_pid];

if (curr_PTE->PRESENT == 0) {
  // printf("%d %d\n", curr_proc->pid, curr_inst->page_or_pid);
  VMA *vma = get_vma_of_page(curr_proc, curr_inst->page_or_pid);

  if (vma != NULL) {
    Frame *frame = get_next_frame();

    if (frame->vpage != NULL) {
      if (O_display) printf(" UNMAP %d:%d\n", frame->pid, frame->vpage->vpage_number);

      frame->vpage->PRESENT = 0;
      processes.at(frame->pid)->unmaps++;
      total_cost += unmap_cost;

      if (frame->vpage->MODIFIED == 1) {
        if (frame->vpage->is_file_mapped == 1) {
          if(O_display) printf(" FOUT\n");
          processes.at(frame->pid)->fouts++;
          total_cost += fout_cost;
        } else {
          if (O_display) printf(" OUT\n");
          processes.at(frame->pid)->outs++;
          frame->vpage->PAGEDOUT = 1;
          total_cost += out_cost;
        }
      }

    }

    curr_PTE->PRESENT = 1;
    curr_PTE->REFERENCED = 0;
    curr_PTE->MODIFIED = 0;
    curr_PTE->frame_number = frame->frame_number;
    curr_PTE->is_file_mapped = vma->file_mapped;
    curr_PTE->WRITE_PROTECT = vma->write_protecetd;

    frame->age = 0;
    frame->time_last_used = inst_index;
    // page map to frame
    set_Frame(frame, curr_proc->pid, curr_PTE);

    if (curr_PTE->is_file_mapped == 1) {
      if (O_display) printf(" FIN\n");
      curr_proc->fins++;
      total_cost += fin_cost;
    } else if (curr_PTE->PAGEDOUT == 1) {
      if (O_display) printf(" IN\n");
      curr_proc->ins++;
      total_cost += in_cost;
    } else {
      if (O_display) printf(" ZERO\n");
      curr_proc->zeros++;
      total_cost += zero_cost;
    }

    if (O_display) printf(" MAP %d\n", frame->frame_number);
    curr_proc->maps++;
    total_cost += map_cost;

  } else {
    if (O_display) printf(" SEGV\n");
    curr_proc->segv++;
    total_cost += segv_cost;
    if (curr_inst->command == 'r') total_cost += read_cost;
    if (curr_inst->command == 'w') total_cost += write_cost;
    continue;
  }
}
if (curr_inst->command == 'r') {
  curr_PTE->REFERENCED = 1;
  total_cost += read_cost;
}

if (curr_inst->command == 'w') {
  curr_PTE->REFERENCED = 1;
  total_cost += write_cost;
  if (curr_PTE->WRITE_PROTECT == 0) {
    curr_PTE->MODIFIED = 1;
  } else {
    if (O_display) printf(" SEGPROT\n");
    curr_proc->segprot++;
    total_cost += segprot_cost;
  }
}
}
```

