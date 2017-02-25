### 概述 
  提供各种语义的队列服务，以分布式方式运行，数据在多个节点之间复制，通过master节点自动选举实现高可用性。  
  所有请求由master节点处理，异步复制到其他节点，客户端可以使用集群中任意选择一个服务器节点发起请求，请求会自动转发给master节点，失败则重试下一个。
  功能点： 
  1. 通过json报文格式实现消息的生产、消费、确认。
  2. 支持消息的最多处理一次，最少处理一次语义。
  3. 支持消息的延迟、重试和有效期。
  4. 系统以集群方式运行，数据在多个节点间自动复制。
  5. 少于一半节点失效时，服务不受影响，数据不会丢失。


### 实现
 1. 使用c++实现，采用单进程多线程异步模型，工作线程负责处理请求，主线程负责在节点间同步数据。
 2. 所有节点都可以接收请求，队列数据相关请求转发给master节点处理。
 3. 对外接口支持tcp/udp协议，请求中数据量过大时(超过2k)建议使用tcp协议，报文使用json格式。
 4. 节点间的数据复制通过内部tcp连接通道进行，异步复制，推拉结合。
 5. 对消息的增/删操作都会记录在消息日志中，使用递增的trans_id保证全局顺序，通过消息日志回放方式实现数据复制。
 6. 消息队列包含三种数据结构，消息存储结构、按时间索引的待处理队列和按时间索引的待确认(重试)队列。
 7. 节点选举算法采用简化的paxos算法 。

 * 内存程序数据结构   
   主要由队列和队列消息日志构成:   
   队列内部有三个数据结构，消息列表，待处理队列，待重入队列。  
   消息列表使用unordered_map保存，维护消息id和消息内容的对应关系。  
   待处理队列使用multimap保存,维护消息的出队列顺序。  
   待重入队列使用multimap保存，维护消息的重入队列顺序。  
   队列消息日志维护所有队列的数据变化(增删)列表，用于在节点间同步数据，每条日志具有唯一且递增的trans_id，使用map保存以保证有序。  

 * 选举算法   
   集群中的节点分为master节点和slave节点，一个集群中有奇数个节点组成，只有一个master节点，其他都是slave节点。   
   master节点处理所有请求，slave节点同步数据，当master节点失效时，从所有slave节点中重新选举一个作为master。  
   为了实现数据一致性，选举算法将节点的队列消息日志中的最大trans_id作为选举的一个条件。  
   选举使用(trans_id,vote_id,node_id)组成的三元组表示选举值，每次有数据更新，trans_id会增加，每次发起选举，vote_id会增加，保证数据最新的节点选举值最高。  
   当节点连接到其他节点数量超过半数时，才能形成有效集群。  
   当有效集群中不存在master节点时，集群中的节点发起选举，获得大多数同意的节点赢得选举，成为新master节点，通知其他节点。  

 * 队列算法   
   生产消息时，通过消息列表存储消息内容，根据消息的delay插入待处理队列相应位置。  
   消费消息时，从待处理队列中按顺序出队列，若消息需要确认(retry>0)，再加入待重入队列，若不需要确认则删除消息。  
   确认消息时，直接删除消息。  
   队列内存有定时器，如果消息一直没有确认，在适当时间将待重入队列中的消息重新放入待处理队列。   
   消息的生产和删除事件会记录到消息日志中。   

 * 同步算法   
   数据同步发生在master节点和slave节点之间，通过消息日志的push同步方式和pull同步方式互相配合实现高效的节点间数据同步。   
   当slave节点启动或选举结束时，主动向master节点发起pull同步请求，带上本地最大的trans_id,master节点返回trans_id的后一条数据，slave持续重复这个过程，直到同步完成。     
   为了防止数据同步占用过多master节点资源，pull同步可以限制同步速率。   
   当master节点上发生数据变更时，master节点写入消息日志，push同步给所有slave节点，slave节点判断收到的数据和本地数据是否连续，如果不连续，忽视此数据并发起pull方式同步。   
   如果连续，接受此数据。  
   
 * 虚拟队列     
   多个真实队列可以绑定在一起，形成一个虚拟队列，向虚拟队列中写入时，会同时向每个真实队列复制。虚拟队列只处理写入，其他操作都通过真实队列进行。   
   
### 演进方向
 1. 消息日志持久化存储。  

### 性能测试
   使用php进行简单的性能测试，同网段2台机器，一台运行客户端，一台运行服务端，
   服务器为4核E5-2620 v2 @ 2.10GHz虚拟机，系统未做优化。  
   客户端为8个php并发进程，每个进程不停发送1万个请求，测试结果：  
total:10000 fail:0 min:0.000230 max:0.003858 avg:0.000540  
total:10000 fail:0 min:0.000226 max:0.004506 avg:0.000542  
total:10000 fail:0 min:0.000231 max:0.005618 avg:0.000542  
total:10000 fail:0 min:0.000270 max:0.004635 avg:0.000581  
total:10000 fail:0 min:0.000281 max:0.003218 avg:0.000582  
total:10000 fail:0 min:0.000280 max:0.004616 avg:0.000582  
total:10000 fail:0 min:0.000275 max:0.008703 avg:0.000583  
total:10000 fail:0 min:0.000278 max:0.004305 avg:0.000584     
   说明  min : 最小耗时(秒) max : 最大耗时(秒) avg : 平均耗时(秒)  
   服务器TPS达到近14000/秒时，平均延迟在0.6毫秒。  

### 依赖   
  依赖protobuf 2.4及以上    

### 编译安装    
  $make && make install    
  编译后的程序生成在./deploy目录下    

### 配置项
  日志配置     
  ```
  <log prefix="queue_server" level="5" />
  ```
  节点配置   
  ```
  <node_info node_type="10" node_id="1" host="0.0.0.0" port="1111" />  
  ```
  队列配置   
  ```
  <queue_config queue_size="10000"  log_size="100000"  sync_rate="3000" />  
  ```
  集群配置   
  ```
   <cluster>  
       <node id="1" host="127.0.0.1" port="1101" />  
       <node id="2" host="127.0.0.1" port="1102" />
       <node id="3" host="127.0.0.1" port="1103" />
   </cluster>
   ```

   虚拟队列配置
   ```   
   <virtual_queue name="test">   
       <queue name="test1" />   
       <queue name="test2" />   
   </virtual_queue>   
   ```   

### 接口
 数据格式为json对象,字段定义：
#### 请求
 action: 请求类型，数字， 1:生产消息  2:消费消息 3:确认消息 4:监控  7:队列列表 104:本节点监控 107:本节点队列列表 。   
 queue: 队列名字，字符串 。   
 delay : 消息延迟处理时间，数字 。   
 ttl :  消息过期时间，数字 ， 应大于delay。  
 retry : 消息未确认时重新进入队列时间，数字，0表示不重试，即最多处理一次，>0表示重试间隔，直到超时，即最少处理1次。  
 data :  消息内容，字符串。  
 seq  : 请求序列号，可选项。  
 msg_id : 消息id 。  

#### 响应
 code: 响应码， 0表示成功 , -1表示系统错误，-2表示重定向。  
 reason : 原因， 失败时表示失败原因 。  
 size : 队列长度 。  
 max_size : 队列最大长度。  
 max_id : 当前消息id 。    
 leader_id ： master节点id 。    
 node_id : 处理请求节点id 。   
 log_size : 消息日志数量。   
 max_log_size ： 消息日志最大数量。    
 trans_id ： 当前trans_id值。   
 wait_status : 队列拥塞情况。     
