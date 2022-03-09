3.	TECHNOLOGY BACKGROUND
3.1	Problem Definition - What technical problem did you solve?
在跟阿里合作过程中，我们接触到 了一个叫DADI（https://github.com/containerd/overlaybd）的项目，
我们在项目中完成的部分是把原本在用户态的读移植到了内核态。事实上，我们在之后的合作中发现，在一些高速存储设备里，内核态只读也可以提升性能的，这点经由阿里团队已经证实，但写的部分则未必，这点由我们intel的spdk团队证实了这点。

基于此，我们了实现读写分离的虚拟块设备。把内核态的块设备实现成只读，然后再在用户态来使用spdk来实现写的加速，我们就可以得到一个高速的存储系统。DADI的原始方案是被设计用于容器镜像加速，我们团队配合DADI实现内核的只读块设备驱动。如果加上用户态spdk，则可以实现完整的虚拟块设备，较好的性能，以及一些其他的优化方向。


3.2	Previous Solutions (if any):

A.	Describe any previous solutions used to solve the problem.
DADI的原始方案是被设计用于容器镜像加速，社区也有一些只读块设备或是文件系统，但这样的内核态只读，用户态写的通用存储方案，暂时还没有。

我们给DADI实现的只读块设备驱动，跟内核的其他只读驱动相比会有很大不同：
1．	基于压缩，最底下的存储层是使用lz4压缩过的block，这一层Dadi团队称之为zfile层
2．	基于docker image + git原理，只记录diff，不记录原始数据，这一层Dadi团队称之为overlaybd层
经过这两层的存储格式，就得到了比内核其他只读文件系统都要更好的性能。

B.	Describe disadvantages of those previous solutions.
传统的读写，考虑到的只是一条路走到底的方式，比如内核态的块设备驱动，device mapper，文件系统，只读文件系统，用户态的spdk，或是原始的dadi。这种做法下，并不能在性能和可靠性之间找到最佳路径。比如内核态读写，是无法充分利用硬件的，因为IO栈很长，无法充分利用NUMA，所以会有spdk。

但是用了spdk，如果读发生在用户态，未必一定是最高效的。DADI团队证实了这点；原始的DADI是通过tcmu，跑iSCSI协议,本来是用户态的读写。如果把发生在用户态的读改到内核态，内核态块设备驱动只读，反而提升了性能。

4.	OVERVIEW OF THE INVENTION
4.1	Short summary – In 1-3 sentences, describe the core of your solution.
这个专利的核心，就是实现读写分离，在内核态读，直接进page cache，然后再在用户态使用spdk加速写。我们给Dadi实现的内核态驱动，已经实现了只读，只需要向overlaybd里，加入spdk来加速写入过程。

现在的存储设备，IO性能都非常高，为了满足高性能不出错的前提，现在NVMe，或是我们的Optane，在读写时都具备原子性，在这一前提下，写入的最佳上下文是用户态。像类似于spdk的用户态块设备驱动，

同时，因为page cache复用等考虑，读操作，特别是类似于DaDi这种已经加过压缩的数据块，则比较适合在内核态直接解压到page cache，以获得较到的读性能。因为我们为DADI团队写的驱动刚好实现了只读的解压，寻址，同时又以块设备的形式虚拟出一个块设备驱动，这种前提下，我们两者结合，就实现了一个读写分离，高性能的虚拟块设备驱动。

4.2	Advantages – In 1-3 sentences, describe the value of the invention to Intel or to our customers. 
通过实现读写分离，再针对性地进行加速，就可以更好地发挥Optane这种设备的潜力。更进一步，如果这种方式被广而接受的话，处于spdk生态里的ARM，PowerPC就会受限于我们的设计方案，必须跟着我们的方案做实现。
对Intel来说：
1．	这个创新可以帮我们推Optane SSD，或是Optane AEP
2．	这个创新也可以帮我们加入一些新的CPU Offloading硬件


5.	DETAILS OF THE INVENTION

5.1	什么是DADI。
DADI是阿里贡献给社区的一种containerd镜像加速方案，他们原始的实现原理是，利用container image每层都是只读的特性，然后大概率前提下，他们发现container里的workload都是变动较少。于是他们借用了git的思想，不再记录块，只记录变动。每个container在关闭或是umount时，产生一次commit动作，形成新的层。在生产环境中，阿里的这套系统在存储上的表现非常优异。这一层，他们称之为overlaybd层。

 
在这一层之外，他们进一步加了压缩，一种很特别的压缩方式，一般的变长压缩，都会采用很长的block，然后再压缩成定长的块。而dadi选择了正好会命中page cache的压缩方式，采用4k的block size，压缩成不定长的块，然后再把块设备的起始地址，offset都记录到一个jump_table里。这样虚拟出来的块设备，如果产生读写的话，直接就可以通过跳表查到IO本应该有的位置。
实践证明，在两层page cache，加过压缩之后，性能得到了较大的提升。
 


5.2	基于DADI的读写分离虚拟块设备
我们为阿里将DADI从Container环境移植到了内核态，但因为在内核块设备驱动里没有合适的commit时机，所以我们中止在只读状态。给内核指定几个zfile压缩过的blob文件，我们就可以将之虚拟化成一个只读的块设备，然后再去mount这个设备之上的文件系统。之后，我们进一步将之虚拟化成device mapper，将整个驱动优化成dm-overlaybd和dm-zfile两层。

此时，这个块设备是如下这种状态，

 
整个磁盘处于只读状态，mount时，加载跳表到内存里，将一段足够长的内存映射给用户态的overlaybd

读数据时，此时发生page fault时，
1．	合适的起始偏移，长度，去查跳表里这段数据的位置，读取这个位置存在磁盘上的位置到内存里，返回给io_soft_queue。
2．	在解压后，这段数据会是类似于container环境的blob，只是带有大量的“空洞”，存的是diff信息。因为是blob，所以一般只会是只读层，而且这些读出来的数据，不会超出之前写入的范围。
3．	返回这些page cache，给上层的文件系统，用户态进程对于底层读写并无感知。

写是重动作，会解发context switch和刷IO，所以将会一直buffer在内存里，在DaDi目前的实现里，只有唯一这一层可写，叫Layer0。

 
发生写入动作时，只写在mmaped的内存里，哪怕是O_DIRECT，但不触发真正的写。此时的overlaybd就会一直处于busy wait状态。一旦数据量大到一定的程度（由overlaybd的配置来决定），就唤醒overlaybd，再通过SPDK去写真正的位置。产生commit动作时，并不直接写，而是做跟读相反的动作，
1．	找到page cache，加入到evicting queue
2．	Overlaybd自己会做一次merge index，更新zfile
3．	我们就会拿到一段zfile 数据段，再通过overlaybd刷回磁盘，此时就会有新的一层压缩过的只读blob。
4．	磁盘又进入到只读的状态。

Summary

我们基于阿里的dadi和spdk，实现了一种特殊的虚拟块设备。这种块设备将使用非常少的IO操作，有较高的性能，并能通过SPDK，引入一些NVMe和Optane相关的优化。这个虚拟块设备具备非常密的存储密度（压缩+空洞），SSD友好（Blob是按批次调用spdk刷入磁盘），最大的特点是实现了读写分离，磁盘将始终努力尝试进入只读状态。


