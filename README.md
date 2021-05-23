# TL2TS

## TL2TS是什么？

TL2TS的全称是TraceLogsToTrainingSet，这是我们DNNs-T故障检测构件的数据预处理组件。TL2TS的功能是从繁杂的基于OpenTracing规范的服务监测日志中检索关键的数据字对，从而将其还原为一条条调用链。

## TL2TS是怎样工作的？

TL2TS包括对正常请求trace logs和注入故障实验请求trace logs的两类处理行为：

### 对正常请求trace logs

TL2TS的工作包括以下几个流程：

#### 1. 清洗原始trace logs

原始trace logs往往包含了大量的监控日志，而在TL2TS和DNNs-T的设计上，我们只需特定时间上的某一段日志。因此，通过设定起始时间戳和结束时间戳，我们可以截取特定时间段的trace logs，以备后续使用。
在这一阶段，TL2TS的产物为：training（.json）

#### 2. 还原整条调用链

还原一整条调用链的关键在于traceID。对于一整条调用链来说，traceID就是它的唯一标识符。这一阶段将把所有具有相同traceID的调用放在一起进行组合，以此表现整条调用链。对于并发调用，其排列的顺序取决于产生调用的时间（即使是并发调用，他们仍有发起调用请求的先后顺序）。
在这一阶段，TL2TS的产物为：atraining（.txt）

#### 3. 将从trace logs中找出每一次调用，并进行匹配

trace logs中服务之间每一次相互的调用都会产生一条日志并以spanID作为唯一标识。这一阶段将会找出具有调用关联关系的两条日志进行匹配，反映服务间一次调用。
在这一阶段，TL2TS的产物为：rtraining（.txt）

#### 4. 生成正常请求字符串

请求字符串是我们设计的一种记录请求执行序列的数据结构，是故障检测实验的关键数据之一。TL2TS会首先找出所有调用链中的所有服务，依次使用哈希函数进行映射。之后，会根据对每一条调用链进行转化，生成请求字符串。所有正常请求的请求字符串会在去重后进行记录。
在这一阶段，TL2TS的产物为：allService、norstr（.txt）

#### 5. 生成正常请求训练集

TL2TS会计算每种正常请求间调用所消耗的平均时间。之后，TL2TS会综合之前的数据，生成我们实验所需要的训练集，这同时也是我们DNNs-T故障检测构件所需要的数据。
在这一阶段，TL2TS的产物为：avgDuration、training（.txt）

### 对注入故障请求trace logs

TL2TS的工作包括以下几个流程：

#### 1. 分割原始trace logs

在设计之初，TL2TS是用于处理包含注入故障实验的trace logs，因此TL2TS的首要功能就是将原始日志分割成为三个部分：故障注入前、故障正在发生、故障被移除。
在这一阶段，TL2TS的产物为：faultN-1、faultN-2、faultN-3（.json）

#### 2. 还原整条调用链

与处理正常调用请求相同，这一阶段将把所有具有相同traceID的调用放在一起进行组合，以此表现整条调用链。
在这一阶段，TL2TS的产物为：aN-1fault、aN-2fault、aN-3fault（.txt）

#### 3. 将从trace logs中找出每一次调用，并进行匹配

与处理正常调用请求相同，这一阶段将会找出具有调用关联关系的两条日志进行匹配，反映服务间一次调用。
在这一阶段，TL2TS的产物为：rN-1fault、rN-2fault、rN-3fault（.txt）

#### 4. 读取请求字符串，计算编辑距离

请求字符串间的编辑距离是我们实验中设计所需要的数据之一，用以衡量工作流异常程度。TL2TS读取正常请求字符串，计算每一段调用链与正常请求调用链间的编辑距离，并标记是否有匹配字符串。
在这一阶段，TL2TS的产物为：fault（.txt）

### 5. 生成注入故障训练集

TL2TS会综合之前的数据，生成我们实验所需要的训练集，这同时也是我们DNNs-T故障检测构件所需要的数据。
在这一阶段，TL2TS的产物为：fN-1training、fN-2training、fN-3training（.csv）


## TL2TS的产物有什么？


### 1. training、faultN-1、faultN-2、faultN-3（.json）

training是起始时间戳对原始trace logs截取的产物，为正常请求日志。
faultN-1、faultN-2、faultN-3是以起始时间戳对原始trace logs分割的产物，并且以traceID和起始时间戳进行排序。
根据我们的设计，-1表示注入故障前，-2表示注入的故障正在影响服务，-3表示注入故障被移除，N为故障实验编号（后同）。

### 2. atraining、aN-1fault、aN-2fault、aN-3fault（.txt）

还原的请求调用链，其中每一行都是一整条调用链。一条调用链记录了服务的一次完整响应流程。其中每一次调用间以逗号为分隔符，并根据产生调用的时间戳作为排列依据。

### 3. rtraining、rN-1fault、rN-2fault、rN-3fault（.txt）

rtraining、rN-1fault、rN-2fault、rN-3fault记录每一次调用的关键信息，也是我们产生数据集的关键信息。数据含义如下：
| 编号 | 名称 | 类型 | 含义 |
| ------ | ------ | ------ | ------ |
| 1 | traceID | string | 所属调用链的编号 |
| 2 | callerServiceName | string | 发起调用的服务名称 |
| 3 | callerSpanID | string | 调用发起端span编号 |
| 4 | calledServiceName | string | 被调用的服务名称 |
| 5 | calledSpanID | string | 调用响应端span编号 |
| 6 | duration | int | 调用持续时间 |
| 7 | TOF | bool | 调用是否出现错误 |

### 4. allService（.txt）

所有正常请求可能会调用的服务名称列表，已去重。

### 5. norstr、fault（.txt）

所有正常/故障请求所对应的请求字符串列表，已去重。

### 6. avgDuration（.csv）

所有正常请求可能会产生调用的消耗时间信息，数据含义如下：
| 编号 | 名称 | 类型 | 含义 |
| ------ | ------ | ------ | ------ |
| 1 | callerServiceName | string | 发起调用的服务名称 |
| 2 | calledServiceName | string | 被调用的服务名称 |
| 3 | avg | float | 平均调用持续时间 |
| 4 | std | float | 调用持续时间标准差 |

### 7. training、fN-1training、fN-2training、fN-3training（.csv）

训练集，数据含义如下：
| 名称 | 类型 | 含义 |
| ------ | ------ | ------ |
| traceID | string | 所属调用链的编号 |
| E | --- | 调用时间矩阵，每一个元素表示相应的两个服务调用所需平均时间 |
| minNormalEditDis | int | 与正常请求的最小编辑距离 |
| minFaultEditDis | int | 与故障请求的最小编辑距离 |
| error | bool | 是否出现错误码 |
| TOF | bool | 调用是否出现错误 |
| faultService | --- | 实际出现故障的服务 |

## 如何使用TL2TS？
* 导入trace logs (.json)
* 设定参数
    *  starttime: 目标trace logs中需要调用链的起始时间戳
    *  endtime: 目标trace logs中需要调用链的最晚时间戳
    *  fstarttime: 目标trace logs中注入故障的起始时间戳
    *  fendtime: 目标trace logs中注入故障的结束时间戳
    *  fnum: trace logs文件数量
* 选择功能 (通过更改ftype参数，选择功能)
    *  0: 建立正常请求训练集
    *  1: 建立故障请求训练集
    *  2: 数据清洗
* 取出训练集


