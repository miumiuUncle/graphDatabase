# Gemini图计算框架 

​		

## 一、部分代码块阅读

### 1、加载无向图

​		加载图时需要对顶点id进行映射，通过哈希将原始id打散在不同的机器，并行地分配全局从0开始连续递增的id，生成id映射关系后，每台机器都会存有id映射表的一部分，随后再将边数据分别按起点和终点哈希，发送到对应的机器进行编码，最终得到的数据即为可用于计算的数据。当计算结束后，需要将数据映射回原id。

#### 	1.1加载图数据

​		输入图数据路径和顶点数，统计出边加载图数据，读取过程中将无向图看作有向图，即一条边可以看作两个节点的出边。

```c++
//获取总节点数vertices
//边数量=文件大小/一条边的内存字节数
edges = total_bytes / edge_unit_size;
//初步估计每个实例需要加载的边（将总的边分配到p个实例上）
read_edges = edges / partitions;
//一个实例需要读取的字节数为
bytes_to_read = edge_unit_size * read_edges;
read_offset    //记录当前读到的边字节数所处位置。
out_degree   //记录每个节点的出度，数组长度为节点个数。
```

​		只要不超过一个实例能存储的字节数，就一直读取，统计每个顶点的出度

**注：**一条无向边被当做两条出边记录两次。



#### 1.2 划分图数据

##### 1.2.1局部感知分块

​		partition_offset记录partition中存储节点的下标，partition_offset[i+1]-partition_offset[i]=第i个分区可以处理的所有节点。

​		总的计算量=边×2+点×alpha

​		每个分区的计算量=每个节点的计算量的总和（按顺序遍历每个节点的计算量，累计计算每个节点的计算量，当达到平衡值得时候就将节点放入下一个分区。如果当前节点的计算量加上后大于平衡值，则将这个节点放入下一个分区）。

**注：**

~~~c++
for (VertexId v_i=partition_offset[i];v_i<partition_offset[i+1];v_i++) {
        //总边数-已放入partition中的边数
        remained_amount -= out_degree[v_i] + alpha;
      }
~~~

​		总的计算量的值如下：

~~~c++
remained_amount = edges * 2 + EdgeId(vertices) * alpha;
~~~

​		每添加一个节点，处理剩下的节点的时候，总剩余的计算量也要减去已经分配到分区中的节点的计算量，因为每条边都当成两个顶点的出边，所以在减去计算量的时候不需要乘以2.

##### 1.2.2 局部感知的子分块

​		与局部感知分块相同，只是是针对每个分区再进行更细粒度的划分，采用local_partition_offset来记录每个CPU内部的分块顶点id。注意将out_degree类型转换为numa感知数组（能知道所绑定的cpu），统计的到out_degree，由于是无向图，所以in_degree=out_degree。

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210825171651011.png" alt="image-20210825171651011" style="zoom:67%;" />

#### 1.3 构建图结构

```
outgoing_adj_index :论文中的idx，idx[v_i+1]-idx[v_i]表示v_i的出边数目
outgoing_adj_list ：nbr，出边集合，每个元素代表出边的dst和边属性
outgoing_adj_bitmap ：ext，有出边值为1，没有出边值为0.
compressed_outgoing_adj_index=[(0,0),(2,1),(4,3),(-,4)]
     (vtx,off)   记录有出边的节点id
     off[v_i+1]-off[v_i]=vtx[i]拥有的入(出)边数
```



<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210816201132649.png" alt="image-20210816201132649" style="zoom:80%;" />



**接收和发送消息**

1、接收消息

​		接收的消息为src和dst，以及边信息。根据接收到的消息设置outgoing_adj_bitmap和outgoing_adj_index，如果dst所在分区中没有src节点，则将src对应的outgoing_adj_bitmap设置为1，表示有出度。outgoing_adj_index[dst_part] [src] 将dst所在分区的src的出边设置为0（初始化为0），给src的出度赋值1，如果再次遇到src，此时src已经在这个分区中，再给出度+1，最后src对应的位置就是出度的个数。

2、发送消息

​		只要边数量没有达到当前实例要处理的所有边，就对边进行发送，将边拷贝到发送缓存中，进行发送，发送的目的地是dst所在的分区，完成一次发送后，再将dst和src交换，再进行一次发送。

##### 1.2.4 构建压缩邻居索引

​		循环每个CPU，对每个有出边的节点，outgoing_edges[s_i]+=outgoing_adj_index[s_i]  [v_i]。outgoing_edges记录每个CPU的出边数量。compressed_outgoing_adj_vertices记录了有出边的节点的数量。遍历每个节点，构造idx（outgoing_adj_index）。定义新的接收线程，目的为了构建outgoing_adj_list，记录的是出边的目标节点nbr。

​		对于接收到的接受缓存，获取接收到的src和dst。

```c++
dst_part = get_local_partition_id(dst);
pos=&outgoing_adj_index[dst_part] [src] ，src位置加1，
EdgeId pos = __sync_fetch_and_add(&outgoing_adj_index[dst_part] [src], 1)
//pos记录了dst所在的分区的src的地址。
outgoing_adj_list[dst_part] [pos].neighbour =dst
outgoing_adj_list[dst_part] [pos].edge_data = recv_buffer[e_i].edge_data;
```



### 2、点处理

​		Gemini中采用了两种通信方式：push和pull模式。

**push模式：**

单机：在上一轮得到值，不计算，这一轮计算并发送出去（发送给邻居）

多机：

针对顶点0来说的话：

（1）在上一轮中0的邻居（入边的邻居）已经将值发送到0这个点，但是没有计算；

（2）根据邻居发送过来的值进行计算；

（3）将master0计算结果的值发送到mirror0（mirror0的值是实时的）；

（4）将Node2中0顶点的值发送到邻居上1,2；

（5）将Node1中0顶点的值发送到邻居上5,6；

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210817114058275.png" alt="image-20210817114058275" style="zoom: 67%;" />

**pull模式：**

单机模式：这一轮去拉值，计算，不传出去。

多机模式：

mirror向master拉取消息，并将消息发送给邻居（master向mirror发送信息）

（1）对于顶点7，处于node2中，先拉取本地的顶点5,6的信息。

（2）找7的mirror，拉取顶点2的信息。

（3）对于所有信息做一个汇总。

（4）汇总的信息并计算，存储在master7上，mirror上的信息不是实时的。

（5）在下一轮的时候再进行发送。

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210817115654444.png" alt="image-20210817115654444" style="zoom: 67%;" />



​		在连通分量算法中，点处理函数的目的是对输入的节点设置为为活跃节点，传入的参数为cc的节点计算函数和节点。

#### 2.1 将线程设置为WORKING

​		对每个sockets中每个线程，获得线程要要处理的节点的起始id和终止id，并且将线程的工作状态设置为WORKING；每个线程处理的起始节点=当前CPU的id+一个CPU处理的所有节点/一个CPU中所有的线程

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210818110040782.png" alt="image-20210818110040782" style="zoom: 67%;" />

源码注释如下：

```c++
//获得当前CPU需要处理的点的数量
VertexId partition_size = local_partition_offset[s_i+1] - local_partition_offset[s_i];
//获得当前线程需要处理的点的起始id
//basic_chunk=64，除以它有乘以它确保
thread_state[t_i]->curr = local_partition_offset[s_i] + partition_size / threads_per_socket  / basic_chunk * basic_chunk * s_j;
//获得当前线程需要处理的点的终止id
thread_state[t_i]->end = local_partition_offset[s_i] + partition_size / threads_per_socket / basic_chunk * basic_chunk * (s_j+1);
//最后一个线程要处理的节点数量可能会多一些
if (s_j == threads_per_socket - 1) {
    thread_state[t_i]->end = local_partition_offset[s_i+1];
 }
//将线程的工作状态设置为WORKING
thread_state[t_i]->status = WORKING;
```



#### 2.2 处理节点

并行处理每一个线程，每个线程一次抓取64个节点处理。

1、正常执行

​		每个线程有一个local_reducer活跃点个数，遍历活跃节点，将节点vtx赋值给label[vtx]，当前线程的起始节点到终止节点，记录该线程处理了多少节点（local_reducer）。

​		目的：label[]更新，将label每个节点的值标记为节点id。

```c++
unsigned long word = active->data[WORD_OFFSET(v_i)];	
//遍历word的每一位，word等于0证明已将64条边均处理完
while (word != 0) {
	//如果该位为1，则表示有出边或者入边（转换为二进制求交集为1）
    if (word & 1) {
         local_reducer += process(v_i);
    }
    v_i++;
	//将word右移一位，处理下一个点
    word = word >> 1;
}
```



2、working_stealing过程

​		当线程完成自己的任务，则将线程状态设置为stealing状态，去找其他线程id（从本线程的id+1开始，顺时针抓取处于WORKING的线程的节点来处理），当线程为working状态的时候，就继续处理该线程的点（每次抓取64个点来处理）。

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210818112522954.png" alt="image-20210818112522954" style="zoom:80%;" />

​		对点处理的working_stealing，取到节点后，是一个正常的执行过程。

**点处理结束**

这一步点处理得到的结果是什么?

（1）更新数组label，根据每个节点id，对label[vtx]设置为节点id，（label[vtx]=vtx）（在label里面记录了活跃点，每个节点都为一个子图）。

（2）对每个线程求local_reducer，在对每个partition求reducer，再对全局求global_reducer。

### 3、边处理

​		连通分量在得到活跃点后，下一步就是边处理。遍历所有边。

​		传入的是push 模式下master发送的函数，mirror接收的函数，pull模式下mirror发送，master接收函数，active_in，进入到边处理函数后，步骤如下：

#### 3.1 重置消息缓存

​		重新设置消息缓存大小，确定缓存消息量达到多大后，将消息进行发送；

```c++
    for (int t_i=0;t_i<threads;t_i++) {
	  //重新设置local_send_bufferd的大小，local_send_buffer_limit  本地发送消息的缓存数目
	  //MessageBuffer ** local_send_buffer;是个二维数组
      local_send_buffer[t_i]->resize( sizeof(MsgUnit<M>) * local_send_buffer_limit );
      local_send_buffer[t_i]->count = 0;
    }
```



#### 3.2 统计活跃边数目

​		点处理函数得到活跃点个数，根据out_degree数组，统计活跃点的出度个数，即活跃边数目（在连通分量算法中，活跃点的所有出边都是活跃边）。

​		目的：给3.3选择什么样的方式进行（push还是pull）消息传递。

```c++
	//2.计算总的活跃边数（点的出边）
	//typedef uint64_t EdgeId;
	//根据活跃点去处理活跃边。
    EdgeId active_edges = process_vertices<EdgeId>(
      [&](VertexId vtx){
		//只用out_degree，需要注意有向图和无向图out_degree的构建
        return (EdgeId)out_degree[vtx];
      },
      active
    );
```



#### 3.3 确定计算模式

​		根据活跃边数目选择push模式还是pull模式：在活跃边数目小于总边数的1/20时，采用sparse模式，否则采用pull模式。对不同模式下，对每个实例中的每个CPU，设置不同大小的send_buffer和recv_buffer，就考虑一个CPU内部的通信量。

**注：**

接收消息量是其他分区的所有master数量*CPU数量，也就是说一个分区发送的时候master和mirror都要发送，实例的每个CPU都要发送一次，所以接收量就是别的分区的发送量。

**push模式：**

1、recv_buffer

接收消息的数量=其他分区的master数*实例的CPU数

2、send_buffer

发送消息的数量=该实例的master数*实例的CPU数

**pull模式：**

1、recv_buffer

接收消息数量=该实例的master数*实例的CPU数

2、send_buffer

发送消息数量=其他分区的master数*实例的CPU数

#### 3.4  push/pull

##### 3.4.1 push

​		如果采用push模式，接收消息的时候以队列（队列大小为partition个数）的方式接收。线程每次取一个块，一个块有64个顶点，当有出边的时候，采用sparse_signal（master向mirror发送消息），

1、遍历当前实例所需要处理的点：

```c++
for (VertexId begin_v_i=partition_offset[partition_id];begin_v_i<partition_offset[partition_id+1];begin_v_i+=basic_chunk) {
        VertexId v_i = begin_v_i;
		//获取64个活跃的顶点
        unsigned long word = active->data[WORD_OFFSET(v_i)];
        while (word != 0) {
		  //假如有出度的话
          if (word & 1) {
			//sparse_signal(push模式下的master，将给mirror的消息放入缓存)
			  /*
			  [&](VertexId src){
					graph->emit(src, label[src]);
			  },
			  */
            sparse_signal(v_i);
          }
          v_i++;
          word = word >> 1;
        }
 }
```

​		

​		发送消息用函数emit()，输入参数为src，label[src]（连通分量中，这个值记录了活跃点，也是将活跃点发送出去，根据出边信息将活跃点连接到一起），遍历块中的每个节点。

emit函数的作用：

​		将要发送的消息放到缓存里，然后达到发送值得时候，采用flush_local_buffer函数进行发送。针对最后一次没有达到阈值的时候，单独在进行一次flush_local_buffer

2、定义发送线程

逆时针的处理每个分区，对当前分区中的每个sockets，都执行MPI_Send，将消息发送到send_buffer

**补充一些发送内容**

```c++
MPI_Send(send_buffer[partition_id][s_i]->data, sizeof(MsgUnit<M>) * send_buffer[partition_id][s_i]->count, MPI_CHAR, i, PassMessage, MPI_COMM_WORLD);
//send_buffer[partition_id][s_i]->data 存储要发送的内容

```



3、定义接收线程

​		顺时针处理分区接收。完成i实例的消息接受后，将i记录到recv_queue中，主线程监听到接收队列的更新，就会进行sparse_slot处理

```c++
 for (int step=1;step<partitions;step++) {
		  //从partition_id+1顺时针到partition_id-1
		  //接收其他实例发过来的消息
          int i = (partition_id + step) % partitions;
          for (int s_i=0;s_i<sockets;s_i++) {
            MPI_Status recv_status;
            MPI_Probe(i, PassMessage, MPI_COMM_WORLD, &recv_status);
            MPI_Get_count(&recv_status, MPI_CHAR, &recv_buffer[i][s_i]->count);
            MPI_Recv(recv_buffer[i][s_i]->data, recv_buffer[i][s_i]->count, MPI_CHAR, i, PassMessage, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            //recv_buffer[i][s_i]->count 接收消息的量
			recv_buffer[i][s_i]->count /= sizeof(MsgUnit<M>);
          }
		  //完成i实例的消息接受后，会将i记录到recv_queue中，主线程中监听到recv_queue更新，即会进行sparse_slot处理
          recv_queue[recv_queue_size] = i;
          recv_queue_mutex.lock();
          recv_queue_size += 1;
          recv_queue_mutex.unlock();
        }
```

**注：**

​		采用顺时针接收其他实例发送过来的消息。

4、循环所有实例

​		循环所有sockets和threads，获取线程处理的起始位置和终止位置，将线程设置为WORKING状态。并行执行sparse_slot函数，正常执行。对于执行完的线程，执行STEALING，顺时针处理其他线程的节点。获取到线程执行的节点位置，取basic_chunk大小的节点，节点块的节点起始位置为b_i，将指向节点块末尾的值end_b_i设置为b_i+basic_chunk。

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210818144239412.png" alt="image-20210818144239412" style="zoom:80%;" />

​		sparse_slot，将收到的dst设置为活跃节点。在计算连通分量算法中，sparse_slot的作用就是根据收到的msg，判断是否要更新label。

```c++
//sparse_slot 函数
[&](VertexId src, VertexId msg, VertexAdjList<Empty> outgoing_adj){
        VertexId activated = 0;
		//outgoing_adj 邻居节点及边数据
        for (AdjUnit<Empty> * ptr=outgoing_adj.begin;ptr!=outgoing_adj.end;ptr++) {
		  //dst就是邻居节点
          //msg=label[src]
          VertexId dst = ptr->neighbour;
          if (msg < label[dst]) {
			//若write_min比较小函数，msg
		    //返回值为true或false，获取数组第dst个下标的地址，
            write_min(&label[dst], msg);
            active_out->set_bit(dst);
            activated += 1;
          }
        }
        return activated;
      },
```

**注：**

​		push模式下，接收的时候才需要使用STEALING机制。

​		问题：边处理的结果是什么？

​		答：对于连通分量算法，边的处理就是进行发送的过程，并对于dst的节点设置为活跃节点的过程。



接收消息流程

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210824104550500.png" alt="image-20210824104550500" style="zoom:67%;" />

##### 3.4.2 pull

​		只有在发出边的时候才采用STEALING机制。（在操作比较复杂的时候才需要这个机制）

| push                                              | pull                                                         |
| ------------------------------------------------- | ------------------------------------------------------------ |
| 接收队列                                          | 发送队列、接收队列                                           |
| mirror接收消息的时候采用                          | mirror拉取消息的时候采用                                     |
| sparse_signal，master发送消息，mirror接收         | dense_signal，拉取mirror邻居信息，将信息发送出去             |
| sparse_slot将接收的消息传递到邻居上。需要stealing | dense_slot，接收到mirror发送过来的消息，计算速度较快所以不需要STEALING， |



## 二、案例：连通分量

### 1、连通分量原理

​		连通分量：连通子图的个数。

​		数学思想：根据读入的边进行集合的划分，读入的边证明是相连的，那就划分到一个集合中，然后读入新的边就划分到新的集合中，一旦读入的边（两个顶点）都在同一个集合中出现过，那么证明是相互连通的，如果读入的边，两个顶点是出现在两个集合中，那么说明这两个集合至少通过该边是可以相连的，就对集合进行合并。依次类推.......

~~~
示例：输入为边
（1,2）（1,4）
（3,5）
顶点124之间有边相连，35之间有边相连那么就有两个连通分量，分别为：
1,2,4
3,5
~~~

示例：通过深度优先方式进行遍历。

~~~c
#include<stdio.h>
int f[105][105]={0},visited[105]={0}; //f邻接矩阵，a给走过的顶点做标记
int m,n,ans=0;

void DFS(int s)
{
	visited[s]=1; //标记已经访问过
	for(int i=1;i<=n;i++)
	{
		if(!visited[i]&&f[s][i]) //当两点连通并且此点没有被访问过
		{
			m++; //累计总数
			visited[i]=1; //标记
			printf("%d",i);
			DFS(i); //DFS
		}
	}
}

int main()
{   
	for(int i=1;i<=n;i++)
	{
		m=0; //m记录当前顶点连通的所有顶点数
		if(!visited[i]) //当此点没有被访问过
		{
			DFS(i);
		}
	}
}

~~~



### 2、Gemini框架上连通分量

基本思想：

​		将所有顶点都标记为活跃点，如果有边相连，则是连通的，将连通的顶点给相同标记，标记类别数则为连通分量。数据label记录了下标作为节点id所属的类别。active_in：将所有节点标记为活跃点，即还没处理过的节点。

```
将所有节点设置为活跃点，执行处理点函数，得到活跃边数目，确定采用push或pull模式。
1、先进行点处理，将所有点都设置为活跃点。使用label数组记录顶点所属的连通子图。
2、根据活跃点以及活跃点的出度计算出活跃边。
3、根据活跃边数量选择push模式或者pull模式进行边处理。
4、以push为例：将顶点src与邻居dst做对比，若dst的label大于src，则将src写入label[dst]，记录dst所属的连通图。（一个连通图所有顶点的id采用该子图最小的节点id来标记）
5、相同label[]值则为一个连通分量。
```



#### 2.1 点处理

​		从所有节点出发，label初始化为0，当出现一个节点vtx，就将label[vtx]=vtx，给线程分配多个chunk_size大小的节点，处理完后，就去取其他线程的节点进行处理。

```c++
//当节点有出边的时候，处理方法如下：
 unsigned long word = active->data[WORD_OFFSET(v_i)];
		
		//遍历word的每一位，word等于0证明已将64条边均处理完
        while (word != 0) {
		  //如果该位为1，则表示有出边或者入边（转换为二进制求交集为1）
          if (word & 1) {
			 //执行process方法
              //local_reducer
            local_reducer += process(v_i);
          }
          v_i++;
		  //将word右移一位，处理下一个点
          word = word >> 1;
        }

//process()函数如下：
VertexId active_vertices = graph->process_vertices<VertexId>(
	  //将cc函数作为一种参数
    [&](VertexId vtx){
	  //返回1标记活跃节点的个数。
      label[vtx] = vtx;
      return 1;
    },
    active_in   //active_in 是活跃节点
  );
```



#### 2.2 边处理

​		遍历所有活跃点（所有点均为活跃点，处理之后就不再为活跃点，所以判断条件为活跃点数>0）根据活跃点的数目统计活跃边数目，判断使用push或者pull，连通分量案例采用了pull进行计算。

**push模式**

```c++
//如果有出边的话
unsigned long word = active->data[WORD_OFFSET(v_i)];
        while (word != 0) {
		  //假如有出度的话
          if (word & 1) {
			//sparse_signal(push模式下的master，将给mirror的消息放入缓存)
			//----感觉处理的还是根据点去处理信息
            sparse_signal(v_i);
          }
          v_i++;
          word = word >> 1;
        }
      }

//sparse_signal(v_i) push模式下发送消息的函数，将有出边的节点发送出去。
	[&](VertexId src){
		graph->emit(src, label[src]);
	},

void emit(VertexId vtx, M msg) {
	//获得线程id
    int t_i = omp_get_thread_num();
	//拿到该线程的缓存数组
    MsgUnit<M> * buffer = (MsgUnit<M>*)local_send_buffer[t_i]->data;
	//在buffer中新增vtx和msg
    buffer[local_send_buffer[t_i]->count].vertex = vtx;
    buffer[local_send_buffer[t_i]->count].msg_data = msg;
	//计数+1
    local_send_buffer[t_i]->count += 1;
	//当计数达到limit(默认为16)时，执行刷新本地时间函数
    if (local_send_buffer[t_i]->count==local_send_buffer_limit){
      flush_local_send_buffer<M>(t_i);
    }
  }
```



接收消息

```c++
for (b_i=begin_b_i;b_i<end_b_i;b_i++) {
  	VertexId v_i = buffer[b_i].vertex;
	//msg_data也是节点，
    M msg_data = buffer[b_i].msg_data;
    if (outgoing_adj_bitmap[s_i]->get_bit(v_i)) {
	//4.2  sparse_slot(push模式下的mirror点)，mirror接收消息并发送给邻居	  
    local_reducer += sparse_slot(v_i, msg_data, VertexAdjList<EdgeData>(outgoing_adj_list[s_i] + outgoing_adj_index[s_i][v_i], outgoing_adj_list[s_i] + outgoing_adj_index[s_i][v_i+1]));
    }
}


//sparse_slot 函数
[&](VertexId src, VertexId msg, VertexAdjList<Empty> outgoing_adj){
        VertexId activated = 0;
		//outgoing_adj 邻居节点及边数据
        for (AdjUnit<Empty> * ptr=outgoing_adj.begin;ptr!=outgoing_adj.end;ptr++) {
		  //dst就是邻居节点
          //msg=label[src]
          VertexId dst = ptr->neighbour;
          if (msg < label[dst]) {
			//若write_min比较小函数，msg
		    //返回值为true或false，获取数组第dst个下标的地址，
            write_min(&label[dst], msg);
            active_out->set_bit(dst);
            activated += 1;
          }
        }
        return activated;
      },
```

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210818194846729.png" alt="image-20210818194846729" style="zoom:67%;" />



**pull模式如下**：

<img src="C:\Users\catherine\AppData\Roaming\Typora\typora-user-images\image-20210818100956767.png" alt="image-20210818100956767" style="zoom:80%;" />

```c++
dst=1
msg=dst=1
src=dst.nerghbour=0
label[0]=0<1=msg
msg=0<dst=1
//执行发送方法
```



```c++
//mirror向master拉取
//根据dst找到src，将dst于label[src]做比较，如果拉取的值更小，则将dst的label值改为更小的值。
[&](VertexId dst, VertexAdjList<Empty> incoming_adj) {
		//
        VertexId msg = dst;
		//块的起始节点	
        for (AdjUnit<Empty> * ptr=incoming_adj.begin;ptr!=incoming_adj.end;ptr++) {
		  //入边的邻居
          VertexId src = ptr->neighbour;
		  //根据入边的src与msg比较，发送更小的值。
          if (label[src] < msg) {
            msg = label[src];
          }
        }
        if (msg < dst) {	  
          graph->emit(dst, msg);
        }
      },
```



```c++
//接收
[&](VertexId dst, VertexId msg) {
		//当接收到的msg比当前节点的label小（label[dst]），则将当前节点（dst）的label修改为msg。
        if (msg < label[dst]) {
			//返回true 或者false，这个函数的目的是什么
          write_min(&label[dst], msg);
		  //active_out，记录dst是可到达的。
          active_out->set_bit(dst);
		  //返回无符号 1
          return 1u;
        }
		//返回无符号0
        return 0u;
      },
```

​		active_out指根据活跃点可到达的节点，交换active_out，active_in是否能全部可达。

#### 2.3 统计连通分量

怎么计算连通分量：

```c++
for (VertexId v_i=0;v_i<graph->vertices;v_i++) {
      count[label[v_i]] += 1;
    }
//count[0]=5
//count[5]=3
//count[1]=0,count[2]=0,...,count[5]=3,count[6]=0,...
for (VertexId v_i=0;v_i<graph->vertices;v_i++) {
      if (count[v_i] > 0) {
        components += 1;
      }
    }
//统计count[i]>0的个数就是连通分量
```

