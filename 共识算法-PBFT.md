# 区块基础-PBFT

磨链输出计划

---
## PBFT算法


### 算法概要
* 拜占庭问题衍生而来的PBFT算法，算法提出一个主要解决拜占庭容错的状态机副本复制。在保证了算法活性和安全性的前提下提供了（n-1）/3的容错性。
* 这个算法在只读过程中只使用一次消息往返，在只写过程中只使用两次消息往返。且在正常操作中使用消息验证码（MAC），公钥加密只在发生失效的过程中使用。

* 参考：https://www.jianshu.com/p/fb5edf031afd 
https://m.2cto.com/kf/201607/527570.html
整理说明PBFT算法概念和过程。
### 算法前提
> > * 系统是异步分布式。
> > * 节点失效独立。
> > * 系统中恶意节点不能无限期延迟正确节点，且无法破解加密算法。
> > * 算法提供不超过（n-1）/3的情况下保证安全性和活性。
> > * 系统中通过访问控制来限制失效客户端造成的破坏，审核阻止客户端的操作。
> > * 失效副本不超过（n-1）/3，延迟不会无限增长。这个时间是发送消息到最后接收的时间间隔。
>

> ### 算法中使用的相关词汇
> > * 公钥签名（RSA算法）
> > * 消息验证编码（MAC）
> > * hash函数生成的消息摘要（message digest）
> > * m表示消息
> > * mi表示节点i签名的消息
> > * D(m)表示消息摘要
> > * 状态（state）
> > * 多个操作（operations）
> > * 副本服务：算法实现确定性的副本复制服务，操作可读可写，基于状态和操作进行任意确定性的计算，副本的复制由n个节点组成。
> > * 安全性：副本复制服务满足线性一致性，理解中心化系统的原子化操作。
> > * 活性：依靠同步提供活性，客户端最终都会收到请求的回复。
> > * 副本集合R
> > * |R|=3f+1 |R|表示副本集合个数
> > * f失效的副本
> > * view：所有副本在一个view视图中
> > * primary：主节点
> > * backups：其他的备份副本
>

> ### n>3f
> > **f为失效节点，那么系统中必须存在3f+1个节点，原因如下：**
> > > * 系统中在n-f个节点通讯后，必须要做正确的判断。
> > > * f个副本失效，不返回响应
> > > * f个副本没有失效，但不返回响应
> > > * 系统需要足够的支持响应，那么响应必须超过失效的，n-2f>f
> > > * f个失效节点，那么在剩余的节点中必须要有一半以上的节点有正确回应，那么加起来就是3f+1个节点中，有f+1回应才能保证消息正确。
> >
>

> ###算法准备说明：
> > * 首先：主节点由公式p = v mod|R|计算得到，这里v是视图编号，p是副本编号，|R|是副本集合的个数。在主节点失效后启动视图更换。
> > * 三个阶段（预准备、准备、确认）
> > * 客户端client：发送请求
> > * 主节点primary:负责给所有客户端请求排序，然后按序发送给备份节点。
> > * 备份节点backups:检查序号合法性，并通过超时机制判断主节点异常。
>
## PBFT算法过程

> ###算法过程：
> > * 主节点（primary）和备节点（backups），系统整体有一个视图（view）的概念。
> > * 首先所有的副本（replica）中选择一个主节点（primary），主节点负责把所有客户端（client）的请求进行排序，然后按排序发送给备节点。在主节点出现故障，如：不分配序号、分配相同的序号等情况，那么备节点主动检查序号的合法性，通过一个timeout的机制检测主节点是否已经失效，出现这种情况，备节点触发view change协议来选择一个新的主节点。
> > * 定义所有副本（replice）集为|R|,那么|R|=3f+1.f就是一直在说的最大容忍失效节点个数。
> > * 通过公式：p = v mod |R|选择出来一个replica，为主节点。v是view全局视图的编号。再每一次view change触发的时候，replica依次转换角色。
> > * 客户端（client）发送请求给主节点（primary），发送一个【REQUEST,o,t,c】，o就来表示具体的操作，t为时间戳（timestamp），给请求加一个时间戳，也对后来的请求有一个约束验证。
> > * 总体上客户端和主节点交互，即验证请求后主节点写到自己的log中，然后回复客户端【REPLY,v,t,c,i,r】v是当前的view的序号，t还是时间戳，i是replica节点的编号（主节点副本编号）、r是执行结果。整个算法中，客户端接收到f+1个replicas回复，且t和r一致，那么就认为是合法的。这个过程中replica和客户端（client）共享一份秘钥。
> > ###具体三阶段：（pre-prepare, prepare, commit）
> > * 三阶段说明：pre-prepare和prepare阶段在系统同一个view里发送的请求给确定序列，各个replica节点都认可这个序列。下图说明：
> > ![屏幕快照 2018-03-13 下午4.41.21.png-278.3kB][1]
> > * 一个client和4个replica节点。replica-0是主节点，其余为备节点。系统中存在3种情况：1.所有节点都是正常节点。2.主节点故障，剩下的三个节点正常。3.主节点是正常节点，三个备节点中有一个是故障节点。上述三种情况下都可以保证整个系统正常。
**注：**在prepare和commit阶段中，view change新的view后，之前的序列保持不变。pre-prepare和prepare阶段中view change新的view后，在新的view之前的序列不作数。（后续说明为什么prepare两者都有）
> > * pre-prepare阶段：主节点收到客户端请求，分配编号，主节点在系统中广播一条PRE-PREPARE到每一个备节点。
PRE-PREPARE包括信息请求编号n、view编号v、信息的消息摘要d（digest）。这条消息到每一个备节点。
备节点验证，验证主节点分配的编号n，如果发现备节点之前已经接收到了一条在同一view下并且编号也是n，但是digest不同的PRE-PREPARE消息，那么就拒绝。验证通过过就备节点就进入prepare阶段。
> > * prepare阶段：备节点在确认主节点信息后，那么进入prepare阶段，同时将PREPARE信息广播给主节点和其他两个备节点（参考上图）。每个备节点会互相收到其他备节点的PREPARE信息，接收后对PREPARE信息验证，先整理其他两个备节点的PREPARE信息和自身的PREPARE信息，如果备节点和其他两个备节点都同意主节点分配的n编号，view编号v、请求消息m。那么就说明这个节点的状态是prepare，这时候生成一个prepared certificate。
**注：** 排序工作在这个时候理论上完成，但是存在漏洞，比如1节点收集完成验证后，到了prepare状态，但是2.3节点并未验证完成，这个时候view change又刚好触发，进入到了一个新的view编号中，那么1的编号可以纳入序列。2和3的编号n则不作数，那么这时候发现单单有1的编号，无法全网认可，所以在新的view，之前1的编号也作废，重新发起提议。但是节点1会把pre-prepare和prepare阶段中的收到和发送的信息记录到log中。
> > * commit阶段：prepare阶段后节点确定分配编号n后，那么在系统中广播一条commit信息，告诉所有节点获得prepared certificate，同时节点也会收到相关其他节点的commit信息，如果节点收到了2f+1条commit消息，并且验证不同节点发来的commit消息中的编号n和view的v值是一致的，那么该节点就拥有一个committed certificate，在这个节点上状态就被commit了。那么观察一个节点的状态commit，也说明了整个系统中处于了prepared状态，其余节点等待到达commit状态。


| 步骤 | 具体步骤 |说明|
|:----:|:----------------:|:-----------:|
|1.客户端发送请求|客户端发送【REQUEST,o,t,c】请求|o是操作、t是时间戳时间戳保证请求只执行一次|
|2.主节点收到客户请求|主节点收到客户端的请求||
|3.预准备阶段|主节点分配一个序列号n给收到的请求，并发送所有备节点预准备消息。【PRE-PREPARE,v,n,d>,m】|v是视图编号，m是客户端发送的请求消息，d是请求消息m的摘要|
|4.备份节点接受预准备消息|请求和预准备消息的签名正确，并且d与m的摘要一致、当前视图编号是v。备份节点从未在视图v中接受过序号为n但是摘要d不同的消息m。预准备消息的序号n必须在水线上下限h和H之间|水线存在的意义在于防止一个失效节点使用一个很大的序号消耗序号空间。|
|5.准备阶段开始|备份节点进入准备阶段，然后向其他副本节点发送准备消息【PREPARE,v,n,d,i】|将预准备和准备消息写入消息日志|
|6.准备阶段|所有节点收到准备消息后，验证签名，视图编号和消息序号。|验证完成写入日志|
|7.准备阶段完成|副本节点将【m,v,n,i】写入消息日志|m是请求内容，预准备消息m在视图v中的编号n，以及2f个从不同副本节点收到的与预准备消息一致的准备消息|
|8.确认阶段开始|副本节点将【COMMIT,v,n,D(m),i】向其他节点广播。接受确认消息条件：签名正确、消息的视图编号与节点的当前视图编号一致、消息的序号n满足水线条件|确认消息写入消息日志|
|8.确认阶段|任意f+1个正常副本节点集合中的所有副本其prepared(m,v,n,i)为真，同时本地确认，prepared(m,v,n,i)为真，并且已经接受了2f+1个确认（包括自身在内）与预准备消息一致。确认与预准备消息一致的条件是具有相同的视图编号、消息序号和消息摘要|确认消息写入消息日志|

> ### 垃圾回收机制
> > * 分布式系统较中心化系统复杂，更多的不确定性。整个过程中，节点执行完操作后，需要有一个清理，清理之前各个阶段的相关信息，但不是盲目清理，比如在prepared阶段，生成的prepared certificat就有可能被再次使用。故清理机制也是很重要的一个部分。
> > * 清理机制，在执行完成每一条请求后，节点再次发送一次广播，在全网对清理信息达成一致。比如在执行多次请求后，约定在规定数量次数请求后，全网发起一次清理，如：连续执行K条操作，全网反馈K条操作已完成，那么删除K条记录。接下来在执行K次，重读上述操作。这个X的点成为checkpoint，重读的这个操作形成一个stable checkpoint（记录在第K条的编号）。
> > * 上述的都是理想化的状态，实际运行过程中会有响应不及时，节点之间步伐不一致等问题，那么就加上一个上文提到过的高低水位。对某个节点来说，它的低水位h等于它上一个stable checkpoint的编号，高水位H=h+L，L是我们指定的数值，它一般是checkpoint周期K的常数倍（这个常数是比较小的，比如2倍）
> 

* 大致介绍了PBFT算法的概念和三阶段过程。参考上面两篇文章，大致整合了下，如果有理解不对的地方请及时指出。

  [1]: http://static.zybuluo.com/JackyJin/eonxfhu24j3skr34ccp2pj97/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-03-13%20%E4%B8%8B%E5%8D%884.41.21.png
