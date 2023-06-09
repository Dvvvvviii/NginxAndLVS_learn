# <font color="#20B2AA" size="8">LVS</font> + <font color="#FF7F50" size="8">Nginx</font>实现高可用集群
---------------------------------------
> ## 1.从单体到集群过渡
> 现状：随用户量不断增多，单体面临瓶颈
---------------------------------------
- ### 1.1单体部署
- #### 1.1.1单体架构的优点
> - 小团队成型即可完成开发-测试-上线  
> - 迭代周期短，速度快  
> - 打包方便，运维省事

- #### 1.1.2单体架构面临的挑战

> |<font color="#00dd00" size="4">面临问题</font><br />|<font color="#4682B4" size="4">应对方案</font><br />|
|:---:|:---:|
|<font color="#666600">单节点宕机造成所有服务不可用</font><br />|通过`集群`实现高可用|
|<font color="#666600">单体代码越来越臃肿，耦合度太高，使迭代、测试以及部署更加麻烦</font><br /> |通过业务拆分，即`分布式`或`微服务`进行改进|
|<font color="#666600">单节点并发能力始终有限（服务器并发能力和自身优化及硬件配置相关）</font><br /> |通过`负载均衡(Load Balance)`来降低服务器负载压力，分发用户请求到其他计算机节点|

- ### 1.2集群概念
> - 计算机群体构成整个系统
> - 这个群体构成一个整体，不能独立存在（可以互相ping通）
> - 群体提升并发与可用性，单业务部署到多服务器，降低单服务器压力
> - 只要是多服务器实现了共同业务，那就称之为`集群`
> - 如果每个计算机节点运行的业务不同，则称之为`分布式`

- #### 1.2.1使用集群的优势
> - 随流量增多，集群流量分发给不同节点，轻松`增强系统的整体性能`
> - 不怕单节点宕机，提高系统可用性,即`高可用`
> - 当预测到高流量时，可预先增加计算机节点，流量减少后再减少，集群具有`可扩展性`

- #### 1.2.2使用集群的注意点
> - <font color="#00dd00">用户会话</font><br />当在集群中时，需要使用分布式会话
> - <font color="#00dd00">定时任务</font><br />当设定定时任务时，集群中所有节点都会在该时间运行相同任务，浪费资源，需要单独部署一台服务器进行定时任务或AMQ延时任务
> - <font color="#00dd00">内网互通</font><br />在一个集群中，必须保障互相连通
----------------------------------------------
> ## 2.Nginx入门
----------------------------------------------
- ### 2.1什么是Nginx
> - Nginx是一个高性能的HTTP和反向代理Web服务器，同时也提供`IMAP(Internet Message Access Protocol)`/`POP3(Post Office Protocol-Version 3)`/`SMTP(Simple Mail Transfer Protocol)`服务
> - 主要功能是`反向代理`
> - 通过配置文件可以实现集群和负载均衡
> - 静态资源虚拟化
----------------------------------------------
> - #### 2.1.1在网络中Nginx的作用
![](/docs_pics/NginxProcedure1.png)
![](/docs_pics/NginxProcedure2.png)
> - <font color="#FF1493" size="4">Nginx在此处相当于一个负载均衡器和反向代理器(网关)</font><br />
- ### 2.2常见的Web服务器
---------------------------------
> |<font color="#FF7F50" size="3">Web服务器</font><br />|<font color="#FF7F50" size="3">用途</font><br />|
|:---:|:---:|
|`MS IIS`|发布`asp.net`的项目|
|`Weblogic`与`Jboss`|用于传统行业：`ERP`/`物流`/`电信`/`金融`，两者服务皆是收费的|
|`Tomcat`与`Jetty`|使用J2EE的开发，多数以`Tomcat`为主|
|`Apache`与`Nginx`|都是反向代理服务器，都可以发布静态服务，`Nginx`成本更低，配置简单，支持并发量更高，随时间推移`Nginx`的各项占有率都渐渐追上甚至反超了`Apache`|
|`Netty`|是高性能的，通过编码开发相应服务器|

---------------------------------------
- ### 2.3正向代理与反向代理
----------------------------------------
> - #### 2.3.1正向代理
- 客户端请求目标服务器之间的一个代理服务器
- 客户端发起的请求会先经过代理服务器，之后将请求转发到目标服务器，获得内容后最后响应给客户端
![](/docs_pics/NginxProcedure3.png)

-----------------------------------------
> - #### 2.3.2反向代理
- 用户请求目标服务器，由代理服务器决定访问哪个IP
![](/docs_pics/NginxProcedure4.png)

通过命令行观察反向代理，先`ping`一下淘宝：
```bash
ping www.taobao.com
# 以下为得到结果
正在 Ping www.taobao.com.danuoyi.tbcache.com [111.3.78.233] 具有 32 字节的数据:
来自 111.3.78.233 的回复: 字节=32 时间=10ms TTL=49
来自 111.3.78.233 的回复: 字节=32 时间=10ms TTL=49
来自 111.3.78.233 的回复: 字节=32 时间=8ms TTL=49
来自 111.3.78.233 的回复: 字节=32 时间=10ms TTL=49

111.3.78.233 的 Ping 统计信息:
    数据包: 已发送 = 4，已接收 = 4，丢失 = 0 (0% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 8ms，最长 = 10ms，平均 = 9ms
```
之后再次`ping`一下淘宝：
```bash
ping www.taobao.com
# 以下为得到结果
正在 Ping www.taobao.com.danuoyi.tbcache.com [111.3.78.232] 具有 32 字节的数据:
来自 111.3.78.232 的回复: 字节=32 时间=8ms TTL=49
来自 111.3.78.232 的回复: 字节=32 时间=10ms TTL=49
来自 111.3.78.232 的回复: 字节=32 时间=10ms TTL=49
来自 111.3.78.232 的回复: 字节=32 时间=11ms TTL=49

111.3.78.232 的 Ping 统计信息:
    数据包: 已发送 = 4，已接收 = 4，丢失 = 0 (0% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 8ms，最长 = 11ms，平均 = 9ms
```
- 可以看到两次`ping`操作访问的节点不同，这是被淘宝的反向代理分配了最终请求抵达的服务器；
- 而我们之前所述的`集群`与`负载均衡(LB)`就是通过反向代理实现的，反向代理就是我们讨论的重点。
---------------------------------------------
> #### 2.3.3反向代理之路由
- 通过Nginx可以实现一些虚拟主机、路由的分发和请求
![](/docs_pics/NginxProcedure5.png)
- 如上图，用户请求了三种类型：`shop`、`center`以及`image`
- 后端有3个服务器：**Tomcat 1**、**Tomcat 2**以及**静态资源服务器**
- 图上现有项目的域名后都带有一个端口，通过`Nginx`可以完全规避端口
- 图上策略是通过`Tomcat`虚拟化图片资源，而通过`Nginx`可以发布所有图片，当用户请求图片时，用`Nginx`可以替代`Tomcat`虚拟目录的作用

----------------------------------------------
> ## 3.Nginx实现集群与负载均衡
----------------------------------------------


----------------------------------------------
> ## 4.高可用Nginx方案与实现
----------------------------------------------

----------------------------------------------
> ## 5.生产环境Nginx替换tomcat
----------------------------------------------








