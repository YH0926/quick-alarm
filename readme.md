# 一个可扩展的报警系统Quick-Alarm

## I. 背景

日常的系统中，报警是不可缺少的一环，目前报警方式很多，最常见的有直接打日志，微信报警，短信报警，邮件报警等；而涉及到报警，一般不可避免的需要提前设置一些基本信息，如报警方式，报警频率，报警用户，开关等；

另外一个常见的问题是一般采用的是单一的报警方式，比如不管什么类型的报警全部都用短信方式触达，然后就会发现手机时常处于被淹没的状态了，久而久之对报警短信就不会敏感了


## II. 目标

因此我们准备设计一个通用的报警框架

- 可以自由选择报警方式，
- 支持用户自定义报警方式拓展
- 支持动态的报警配置，
- 支持用户自定义报警规则拓展
- 支持报警方式自动切换规则设定
- 支持报警方式自定义自动切换规则拓展


通过做一个东西，当然是希望可以带来一些用处，或者能学习到什么东西，才不枉花费精力来折腾一下，那么我们这个报警系统，究竟有什么用，或者可以从中学习到什么东西呢？

**用途：**

- 支持灵活可配的报警规则，以及具体报警业务的自定义拓展
- 目标就是统一报警的使用姿势，也就是不管什么报警，都一个姿势，但是内部可以玩出各种花样，对使用者而言就方便简洁了

**学习：**

抛开特有的知识点，可以抽象一些公共可用的地方，大概就下面这两点了

- 我们可以如何支持功能的动态可拓展
- 线程池的使用

## III. 设计

整体来说，报警主要可以划分为三个步骤，如下：

![IMAGE](https://s17.mogucdn.com/mlcdn/c45406/180209_3f276k99cb3k1kec5g184f6c4hb7f_2030x996.jpg)

- 提交报警：对外部使用者提供的接口
- 选择报警：根据报警相关信息，选择具体的报警执行单元
- 执行报警：实现具体的报警逻辑


从任务划分上来看，比较清晰简单，但是每一块的内容又必须可以拓展，

- 选择报警：
  - 报警规则的制定
  - 报警规则加载器  `ConfLoader`
  - 报警规则变更的触发器 `ConfChangeTrigger`
  - 报警规则解析器
    - `ConfParse` ： 解析文本格式报警规则为业务对象
    - `AlarmSelector` ：根据报警规则和报警类型，选择具体报警执行器 `AlarmExecute`

- 执行报警：
  - 线程池执行（以防止影响主业务流程）
  - AlarmExecute的动态拓展（支持用户自定义的报警器实现）
  - 实际的报警逻辑

根据上面的拆解，在应用启动的时候，就有一些事情必须去做了

1. ConfLoader的选择
2. 报警规则加载
3. AlarmExecute的加载（包括默认的+自定义实现的）


下图显示在应用启动时，报警规则解析的相关步骤

![应用启动.png](https://s17.mogucdn.com/mlcdn/c45406/180209_41ccjhcg1ag35i36ikel3jekf8ld9_868x608.png)


至于报警执行器的加载就比较简单了，如下图

![IMAGE](https://s17.mogucdn.com/mlcdn/c45406/180209_5jii7f1ed2j3f8e0di3aalhgji114_1666x402.jpg)


因此，整个的工作流程如下图

![alarm-arch.jpg](https://s17.mogucdn.com/mlcdn/c45406/180209_5eh16796bg6gk4622dj44diaa09bd_1078x620.jpg)


## IV. 任务拆解

通过前面的任务设计之后，对需要做的东西有了一个大概的脉络了，因此在正式操刀实现之前，下对整个架构进行任务拆解，看下可以具体的执行步骤可以怎么来

- 最直接的就是设计报警执行器`AlarmExecute`
  - 定义基本接口
  - 制定自定义扩展规则
- 接下来就是设计报警规则
  - 如何加载报警规则？
  - 报警规则具体的定义细则
  - 报警规则的解析：即根据报警类型来获取报警执行器
  - 报警规则动态更新支持
- 报警线程池
  - 维护报警队列
  - 报警的计数与频率控制
- 封装对外使用接口

所以，通过上面的分析可以看出，这个系统的结构还是蛮简单的，整个只需要四个部分就可以搞定，其中最主要的就是前面两个了，后面将分别说明

## V. 整体说明

### 1. 相关文档

1. [报警系统QuickAlarm总纲](https://liuyueyi.github.io/hexblog/2018/02/09/%E6%8A%A5%E8%AD%A6%E7%B3%BB%E7%BB%9FQuickAlarm%E6%80%BB%E7%BA%B2/)
2. [报警系统QuickAlarm之报警执行器的设计与实现](https://liuyueyi.github.io/hexblog/2018/02/09/%E6%8A%A5%E8%AD%A6%E7%B3%BB%E7%BB%9FQuickAlarm%E4%B9%8B%E6%8A%A5%E8%AD%A6%E6%89%A7%E8%A1%8C%E5%99%A8%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/)
3. [报警系统QuickAlarm之报警规则的设定与加载](https://liuyueyi.github.io/hexblog/2018/02/09/%E6%8A%A5%E8%AD%A6%E7%B3%BB%E7%BB%9FQuickAlarm%E4%B9%8B%E6%8A%A5%E8%AD%A6%E8%A7%84%E5%88%99%E7%9A%84%E8%AE%BE%E5%AE%9A%E4%B8%8E%E5%8A%A0%E8%BD%BD/)

### 2. 历程

- [x] 2018.02.07
    - 整体框架搭建完成
    - 基本功能实现 
- [x] 2018.02.13
    - 项目开发技术文档完成
    - 开发使用手册完成


### 3. todoList

- 剥离报警规则和解析器，支持用户自定义
- 包装邮件报警方式
- 包装微信报警方式



## VI. 其他

### 个人博客： [Z+|blog](https://liuyueyi.github.io/hexblog)

基于hexo + github pages搭建的个人博客，记录所有学习和工作中的博文，欢迎大家前去逛逛


### 声明

尽信书则不如，已上内容，纯属一家之言，因本人能力一般，见识有限，如发现bug或者有更好的建议，随时欢迎批评指正，我的微博地址: [小灰灰Blog](https://weibo.com/p/1005052169825577/home)

### 扫描关注

![QrCode](https://s17.mogucdn.com/mlcdn/c45406/180209_74fic633aebgh5dgfhid2fiiggc99_1220x480.png)