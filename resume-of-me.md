### Abount Me:
熊景
12 年 6 月毕业于湖南工程学院计算机系
现居上海

Email: yuxingfirst@gmail.com
Github: [https://github.com/yuxingfirst](https://github.com/yuxingfirst)
Phone: 185 1610 8950
Blog: [http://coderworm.com/categories.html](http://coderworm.com/categories.html)

### Person Details:
  12年进入 51.com 基础架构部工作。至 13年 8 月在基础架构部负责 51.com 中间层的开发及维护工作。平时的工作平台都是基于 linux，所以懂 Linux 下常用的命令，开发工具(vim)以及一些常用系统监控状态命令,如：top, pstack, strace,free 等等。
  了解常规的网站/后台系统分布式架构方案，如分布式缓存，业务处理模块，代理中间件（webproxy / dbproxy / nosqlproxy）; 
  数据库分库分表设计方案(用户资料存储)；用户认证系统（登陆/注册业务） 。  
  熟悉 linux 下 c/c++编程；tcp 网络编程，常用网络编程模型Reactor；熟悉 linux 下轻量级线程(协程)的原理及实现方式及调度原理
  。了解基于事件驱动的方式下协程调度器的实现。  
  13 年 8 月至今，内部转岗从事页游项目开发(13 年 12 月底上线)，进行游戏服务器逻辑的开发工作。  
  会使用 mysql, oracle 数据库及一些基础的 sql 操作。  
  熟悉基于 github 的项目版本管理，能熟练使用 github；开发工具习惯使用 vim。    
  除 c/c++之外，还学习并使用过 java，php，shell，awk，golang，能写一些小工具。    
  对服务器后端技术及高性能网络程序设计有浓厚兴趣。  
 
### Project experience：
###### Project1: 基于 Redis 的通用排行榜系统
应对网站内部及页游运营平台各种排行榜需求， 需要一套通用的排行榜系统来避免各业
务团队的重复工作， 于是基于 Redis 开发了一套业务无关的排行榜系统，业务团队可自行设定排行规则及各种统计规则。
主要采用Redis的sort set开发业务逻辑以及 master-slave 的结构，来实现服务的数据持久化机制，确保服务的高可用性，避免宕机时的数据丢失。

###### Project2: corosched 协程调度框架
这个项目是近期发布在 github 上的一个开源项目，主要目的是采用 C 语言实现一个高
性能的协程调度框架，主要特性包括：协程调度，协程的多线程运行，事件驱动，协程之间
的消息通信， 异步 socket 操作。

###### Project3: webagent项目
这是一个位于前端php模块和后端中间层之间的代理服务器，实现php模块对后端模块的透明访问，方便后端中间层系统的升级，维护等。采用多进程架构模型，主要特性包括:多种通信协议支持，后端服务的动态添加与减少，心跳机制等等。

###### Project4: 通用数据中间层
本系统主要是在数据库之上的一个代理， 通过制定一定的规则及操作原语， 可以执行位
于配置文件中的 SQL 语句以及缓存查询操作，并且支持通过可配置的规则对数据进行的分
库分表存储。这样对于一些只有数据库操作需求的业务，就可以以零代码纯配置来完成。
数据中间层主要跟缓存跟 DB 交互，db 的查询结果会记录到缓存中，这样下次查询就
可以从缓存中直接获取结果。为了保证数据的一致性，针对一条记录的写操作，会同时清除
缓存中的记录。

###### Project5: 激战海贼王项目(页游)
页游项目，担任服务器程序开发。负责开发了：邮件系统，组队系统，场景管理系统(动
态场景/静态场景)，排行榜系统以及游戏内的其他功能模块。目前游戏已在 51 运营平台，
新浪微游戏运营平台投入运营。
