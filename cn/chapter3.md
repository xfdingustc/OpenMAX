#3 OpenMAX IL控制API
OpenMAX IL API允许IL客户端控制音频、视频、图像领域上的组件。“其他”领域还包括额外的一些功能，例如音视频同步。OpenMAX IL API的使用者往往是一个多媒体框架。在本文档的其他部分，OpenMAX IL API的使用者指的就是IL客户端。

OpenMAX IL API定义了一组头文件，他们的名称是：

-  `OMX_Types.h`: OpenMAX IL使用的数据类型
-  `OMX_Core.h`: OpenMAX IL 核心 API
-  `OMX_Component.h`: OpenMAX 组件 API
-  `OMX_Audio.h`: OpenMAX 音频数据结构
-  `OMX_IVCommon.h`: OpenMAX 视频和图像通用的数据结构
-  `OMX_Video.h`: OpenMAX 视频数据结构
-  `OMX_Image.h`: OpenMAX 图像据结构
-  `OMX_Other.h`: OpenMAX 其他的数据结构 (包括音视频同步)
-  `OMX_Index.h`: OpenMAX定义的数据结构的索引

本节介绍了如果配置OpenMAX Core和OpenMAX组件的操作。

首先介绍了OpenMAX的数据类型。其次，阐述了OpenMAX Core的方法。组件的实现的方法在第3.3节中讨论。最后，第3.4节介绍了一些操作的调用顺序，包括组件的初始化，普通数据流，数据管道的建立，数据管道中数据流。这些时序图介绍了IL客户端，IL Core和OpenMAX组件之间的交互。

下面约定用于记录接口方法参数：

-  <参数名> [输入] 指定一个输入参数，由函数调用者设置并被函数读取。
-  <参数名> [输出] 指定一个输出参数，由函数本身设置并返回给调用者。当函数返回时，调用者读取通过引用传递的参数的新值。
-  <参数名> [输入输出] 指定一个输入/输出参数，由函数调用者设置。函数改变这个参数的值并返回给调用者。

参数的分类可以在OpenMAX的头文件中找到，里面定义了空的宏： `OMX_IN`, `OMX_OUT` 和 `OMX_INOUT`。`OMX_IN`对应 <参数名> [输入]， `OMX_OUT`对应<参数名> [输出]， `OMX_INOUT` 对应<参数名> [输入输出]

##3.1  OpenMAX 类型
###3.1.1 枚举
`OMX_Core.h`中定义了5个32位整型枚举数

- `OMX_ERRORTYPE` 为每一个OpenMAX IL API方法的返回值（见3.1.1.3小节）
- `OMX_COMMANDTYPE` 包含了所有IL客户端发往组件的命令（见小节3.1.1.1）
- `OMX_EVENTTYPE` 包括了OpenMAX组件产生并传递给IL客户端的消息（见3.1.1.4节）。
- `OMX_BUFFERSUPPLIERTYPE` 包括了管道端口中所有可能的buffer供应者。3.1.1.5小节可以看到这个枚举类型用法的描述。
- `OMX_STATETYPE`, 在3.1.1.2中描述。

图 3-1 显示了`OMX_Core.h`定义的枚举类型

![](img/3_1.png)


**图 3-1. OMX_Core.h中定义的枚举类型**

####3.1.1.1  OMX_COMMANDTYPE
表3-1展示了IL客户端可以向OpenMAX组件发送的消息类型。由于消息是非阻塞的，当消息处理完毕后， OpenMAX组件会生成一个消息完成回调。
回调是在一个专门的结构中定义，见3.1.2.7小节。


| 字段名称 | 描述 |
| ------------- | ------------- |
| OMX_CommandStateSet | 切换组件状态 |
| OMX_CommandFlush | 清空组件上一个端口的buffer队列|
| OMX_CommandPortDisable | 禁用组件上一个端口 |
| OMX_CommandPortEnable | 启用组件上一个端口|
| OMX_CommandMarkBuffer | 标记一块buffer并指定接受标记时间的组件|

表 3-2 描述了每一个命令需要的参数。

| 命令代码 | 参数 | 数据 |
| ------------- | ------------- |  ------------- |
| OMX_CommandStateSet |OMX_STATETYPE – 要转移的状态 | 无 |
| OMX_CommandFlush | OMX_U32 – 目标端口ID | 无 |
| OMX_CommandPortDisable |OMX_U32 – 目标端口ID  | 无 |
| OMX_CommandPortEnable | OMX_U32 – 目标端口ID  | 无 |
| OMX_CommandMarkBuffer | OMX_U32 – 目标端口ID  | OMX_MARKTYPE* - 标记数据和目标组件 |

**表 3-2. 命令语法**
####3.1.1.2  OMX_STATETYPE
表3-2展示了IL客户端调用了一系列`OMX_SendCommand`(`OMX_StateSet`, <状态>)后的状态转移，新的状态当参数传递给组件。尖括号包围的转移名表示转换不是由IL客户端命令触发的，而是由一系列组件内部事件的结果。

![](img/3_2.png)

**图 3-2. OpenMAX 组件状态转移**

这个小节描述了组件的状态。IL客户端通过调用`OMX_SendCommand`发送`OMX_CommandStateSet`命令来切换组件状态。

表 3-3 展示了OpenMAX组件的状态

| 字段名 | 描述 | 是否获取资源 | buffer位置 |
| ------------- |-------------| ------------- | ------------- |
| OMX_StateInvalid | 组件已损坏或遇到无法回复的错误 | 未知 | 未知 |
| OMX_StateLoaded | 组件已加载但没有获得资源 | 否 | 无 |
| OMX_StateIdle | 组件已获得资源但没有转递任何buffer或开始处理数据 | 是 | 只有供应者 |
| OMX_StateExecuting | 组件以开始转递buffer并处理数据 | 是 | 供应者和非供应者 |
| OMX_StatePause | 组件暂停处理数据但可能会从暂停点恢复 | 是 | 供应者和非供应者 |
| OMX_StateWaitForResources | 组件在等待可用资源| 否 | 无 |

**表 3-3. OpenMAX 组件状态**
######3.1.1.2.1  OMX_StateLoaded
在调用`OMX_GetHandle`创建组件之后，分配资源之前，组件处于`OMX_StateLoaded`状态。在这个状态，IL客户端可以通过`OMX_SetParameter`改变组件参数，创建组件端口上的数据通道，或者切换组件状态至`OMX_StateIdle`或`OMX_StateWaitForResources`。

IL客户端可以选择一个处于`OMX_StateLoaded`的组件转移到`OMX_StateWaitForResources`状态，例如，组件未能获得切换到`OMX_StateIdle`状态的资源。

###### 3.1.1.2.1.1  OMX_StateLoaded 到 OMX_StateIdle
如果IL客户端请求状态由`OMX_StateLoaded`切换到`OMX_StateIdle`，组件必须在完成状态切换前获得所有的资源，包换buffer。此外，在状态切换完成之前，buffer的提供者（在非管道模式时为IL客户端），必须保证非提供者拥有他所有的buffer。如果一个端口连接到IL客户端，IL客户端可以自己分配buffer并通过调用端口上的`OMX_UseBuffer`方法转递给端口，或者调用端口上`OMX_AllocateBuffer`命令让端口直接分配。

当端口出于管道状态，供应端口要么自己分配buffer，要么当端口实现了buffer共享时，复用同组件上的其他端口上的buffer。管道供应端口则通过调用非供应者的`OMX_UseBuffer` 将buffer转递给非供应者。

端口上的buffer数量由端口的定义（见`OMX_IndexParamPortDefinition`）确定，默认是最小值（见同一个数据结构）。但在提供者可以调用`OMX_UseBuffer` 和`OMX_AllocateBuffer`之前可以通过调用 `OMX_SetParameter`修改此值。

#####3.1.1.2.2  OMX_StateIdle
在`OMX_StateIdle`状态时，组件已经可以被使用，这意味着所有必要的资源已经分配。但是，提供者仍然保留着buffer，并没有发生buffer交换或处理。因此，如果这个状态由`OMX_StateExecuting`或`OMX_StatePause`转移而来，组件必须归还他正在处理的所有buffer给他的提供者。IL客户端可能会转移到除了`OMX_StateInvalid`和`OMX_StateWaitForResources`任何其他状态。

######3.1.1.2.2.1  OMX_StateIdle 到 OMX_StateLoaded
在从`OMX_StateIdle` 到 `OMX_StateLoaded`的转移过程中，每一个buffer提供者必须为非提供者端口上的每一块buffer调用`OMX_FreeBuffer`方法。如果提供者分配了buffer，他必须在调用`OMX_FreeBuffer`之前释放buffer。如果非供应端口分配了buffer，他必须收到`OMX_FreeBuffer`调用时释放内存。此外，非供应端口总是必须收到`OMX_FreeBuffer`调用时释放buffer头。当所有的buffer被移出组件时，状态转移完成。组件通过一个回调时间表示调用`OMX_SendCommand`完成。

######3.1.1.2.2.2  OMX_StateIdle 到 OMX_StateExecuting
如果IL客户端请求将状态由`OMX_StateIdle`切换至`OMX_StateExecuting`，组件应该开始转移并处理数据。和IL客户端通信的端口，IL客户端会通过`OMX_EmptyThisBuffer`和`OMX_FillThisBuffer`初始化数据传输。在管道端口中，任何输入端口也是供应端口，应该把它的空buffer通过调用`OMX_FillThisBuffer`转移给他的管道输出端口。

#####3.1.1.2.3  OMX_StateExecuting
在这个状态中，OpenMAX组件传输并处理数据。组件应该接受其输入端口的`OMX_EmptyThisBuffer` 调用和输出端口的`OMX_EmptyThisBuffer`。任何与IL端口通信的端口应该调用回调函数`EmptyBufferDone`和`FillBufferDone`返回空或满的buffer给IL客户端。管道端口应该调用`OMX_FillThisBuffer`或`OMX_EmptyThisBuffer`返回空或满的buffer给管道端口的另一段。IL客户端可以将组件从`OMX_StateIdle`或`OMX_StatePaused`转移至`OMX_StateExecuting`。

######3.1.1.2.3.1  OMX_StateExecuting 到 OMX_StateIdle
如果IL客户端请求状态由`OMX_StateExecuting`转移到`OMX_StateIdle`，组件应该在转移完成之前归换所有的buffer给他的提供者，并且接受所有自身提供者端口上的buffer。任何与IL客户端通信的端口应该通过`OMX_EmptyBufferDone`和`OMX_FillBufferDone`返回自己持有的buffer，这些buffer本来是分别给输入或输出端口使用的。任何管道端口应该通过`OMX_EmptyBufferDone`和`OMX_FillBufferDone`返回自己持有的buffer给管道另一端的端口。同理，非供应管道端口应该等待他的管道端口返回所有的buffer。

#####3.1.1.2.4  OMX_StatePause
在这个状态下，OpenMAX组件不传输或者处理数据，但buffer也不会返回给供应者。`OMX_StatePause`转移到`OMX_StateExecuting`，执行可以继续并且可能不会丢失数据。组件在自己的输入端口上可能继续接受数据，但这些buffer仅仅存放到队列中但不会进一步处理。 IL客户端可能将组将由`OMX_StatePause`转移至`OMX_StateIdle`或`OMX_StateExecuting`。在`OMX_StatePause`向`OMX_StateIdle`转移时,组件应该想他的供应者归还所有的buffer，方式描述参见3.1.1.2.3.1小节。

#####3.1.1.2.5  OMX_StateWaitForResources
在这个状态中，组件等待一个或多个需要的资源。这个状态和资源管理器有关。假设系统有一个或多个硬件特有的资源管理器来管理资源。OpenMAX组件和资源管理器之间的交互不再本标准讨论范围内。

如果处于`OMX_StateLoaded`状态的组件由于非buffer资源不足而无法切换至`OMX_StateIdle`状态是，IL客户端如果希望知道什么时候资源变得可用，那么可以将组件至于`OMX_StateWaitForResources`状态。IL客户端可以通过由`OMX_StateWaitForResources`状态转移至 `OMX_StateLoaded`命令组件停止等待资源。如果在`OMX_StateWaitForResources`状态的组件得到了所有等待的资源，它应该开始转移至`OMX_StateIdle`。

######3.1.1.2.5.1  OMX_StateWaitForResources 到 OMX_StateIdle
当组件开始从`OMX_StateWaitForResources`向`OMX_StateIdle`转移，它应该通过事件`OMX_EventResourcesAcquired`向IL客户端进行通知。当IL客户端收到`OMX_EventResourcesAcquired`时间，它应该调用`OMX_UseBuffer` 和`OMX_AllocateBuffer`，和从`OMX_StateLoaded`向 `OMX_StateIdle`一样，同理，除非组件获得了所有的资源，包括buffer，他不能完成到`OMX_StateIdle`的状态转移。

#####3.1.1.2.6  OMX_StateInvalid
在这个状态时，组件发现内部损坏或遇到无法恢复的错误。当它检测到这个情况是，组件将自己转移到`OMX_StateInvalid`并通知IL客户端产生一个值为`OMX_ErrorInvalidState`的`OMX_ErrorEvent`事件。 但客户端收到这个时间。它应该释放组件关联的所有资源并且最后调用`OMX_FreeHandle`释放组件关联的句柄。

位于`OMX_StateInvalid`状态的组件应该除了 `OMX_GetState`, `OMX_FreeBuffer`, 或`OMX_ComponentDeinit`方法调用外，其他的调用都失败并返回`OMX_ErrorStateInvalid`错误信息。IL组件应该同样明确的通过`OMX_SendCommand`将组件转移到`OMX_StateInvalid`状态。组件可以从任意状态转移到`OMX_StateInvalid`。

####3.1.1.3  OMX_ERRORTYPE
表3-4描述了枚举类型`OMX_ERRORTYPE`，它定义了每一个OpenMAX IL API返回的OpenMAX的标准错误。这些错误可以覆盖大多数的普通错误。但是硬件厂商可以根据下面的原则自由的加入额外的错误类型：

-  厂商的错误消息范围是0x90000000到0x9000FFFF。
-  厂商错误消息应该定义在和组件一起提供的头文件中。未定义的错误消息是不允许的。

|字段名称| 值 | 描述 |
|------------- |:-------------:|  ------------- |
|OMX_ErrorNone| 0 | 函数返回正确|
|OMX_ErrorInsufficientResources | 0x80001000 |执行请求操作没有足够的资源|
|OMX_ErrorUndefined | 0x80001001 | 未知原因的错误 |
|OMX_ErrorInvalidComponentName |0x80001002 | 组件名错误 |
|OMX_ErrorComponentNotFound | 0x80001003 | 没有找到指定名称的组件 |
|OMX_ErrorInvalidComponent | 0x80001004 | 指定组件没有`OMX_ComponentInit`入口，或者组件没有完成`OMX_ComponentInit`调用 |
|OMX_ErrorBadParameter | 0x80001005 | 一个或多个参数非法|
|OMX_ErrorNotImplemented | 0x80001006 | 请求功能未实现 |
|OMX_ErrorUnderflow | 0x80001007 | 下一个buffer准备好之前目前的buffer已空 |
|OMX_ErrorOverflow | 0x80001008 | 需要buffer的时候不可用|
|OMX_ErrorHardware | 0x80001009 | 硬件响应错误 |
|OMX_ErrorInvalidState | 0x8000100A | 组件处于`OMX_StateInvalid`状态 |
|OMX_ErrorStreamCorrupt | 0x8000100B | 发现流损坏 |
|OMX_ErrorPortsNotCompatible | 0x8000100C | 建立管道的端口不兼容 |
|OMX_ErrorResourcesLost | 0x8000100D | 处于`OMX_StateIdle` 状态的组件丢失了分配的资源， 导致组件回到`OMX_StateLoaded`状态 |
|OMX_ErrorNoMore | 0x8000100E | 没有更多的索引可以枚举。 |
|OMX_ErrorVersionMismatch | 0x8000100F | 组件检测到版本不匹配。 |
|OMX_ErrorNotReady | 0x80001010 |组件此时没有准备好返回数据。|
|OMX_ErrorTimeout | 0x80001011 | 发生超时. |
|OMX_ErrorSameState | 0x80001012 | 组件试图切换至当前正处于的状态|
|OMX_ErrorResourcesPreempted | 0x80001013 |处于 `OMX_StateExecuting`或 `OMX_Pause`状态的组件所分配的资源被抢占，导致组件回到`OMX_StateIdle`状态|
|OMX_ErrorPortUnresponsiveDuringAllocation |0x80001014|非供应端口认为等待供应端口调用`OMX_UseBuffer`来分配buffer时间过长。非供应端口在loaded向idle进行状态切换时或启用某一端口时通过`EventHandler`回调发送此错误给IL客户端|
|OMX_ErrorPortUnresponsiveDuringDeallocation|0x80001015|非供应端口认为等待供应端口调用`OMX_FreeBuffer`来释放buffer时间过长。非供应端口在idle向loaded进行状态切换时或禁用某一端口时通过`EventHandler`回调发送此错误给IL客户端|
|OMX_ErrorPortUnresponsiveDuringStop |0x80001016|供应端口认为等待非供应端口调用`EmptyThisBuffer`或`FillThisBuffer`返回时间过长。供应端口在idle向loaded进行状态切换时或禁用某一端口时通过`EventHandler`回调发送此错误给IL客户端|
|OMX_ErrorIncorrectStateTransition|0x80001017|试图进行不允许的状态转移|
|OMX_ErrorIncorrectStateOperation|0x80001018|试图调用的命令和方法在当前状态是不支持的|
|OMX_ErrorUnsupportedSetting| 0x80001019|一个或多个参数或配置结构不正确|
|OMX_ErrorUnsupportedIndex|  0x8000101A|给出的索引参数或配置不支持|
|OMX_ErrorBadPortIndex|  0x8000101B|给出得端口索引不正确|
|OMX_ErrorPortUnpopulated | 0x8000101C | 端口丢失了一个或多个buffer|

**Table 3-4. OpenMAX 错误代码**

####3.1.1.4  OMX_EVENTTYPE
枚举类型`OMX_EVENTTYPE`如表3-5所示，它包括了OpenMAX组件产生的事件类型。3.1.2.7小节描述了OpenMAX组件产生事件并通过回调传送给IL客户端。与事件关联的参数也一并通过回调传递。

| 字段名 | 说明 |
| ------------- | ------------- |
| OMX_EventCmdComplete | 组件完成命令执行。 |
| OMX_EventError | 组件检测到错误。|
| OMX_EventMark | 一个标记的buffer到达目标组件，IL客户端收到此带有指向私有数据指针的事件。|
| OMX_EventPortSettingsChanged | 组件改变了端口设置。例如，组件根据比特流的解析相应的改变了端口设置。|
| OMX_EventBufferFlag | 组件检测到码流结束（EOS）时发送的事件。|
| OMX_EventResourcesAcquired | 组件得到资源并将从`OMX_StateWaitForResources`切换到`OMX_StateIdle`|

**表 3-5. OpenMAX 事件类型**

#####3.1.1.4.1  OMX_EventCmdComplete
组件完成命令执行后会立刻产生`OMX_EventCmdComplete`事件传递给IL客户端。如果是组件状态改变，新状态会作为事件的参数。组件转移到`OMX_StateInvalid`不会产生此事件。

#####3.1.1.4.2  OMX_EventError
组件检测到下面的情况的错误时会产生｀OMX_EventError｀事件，错误事件类型会放在事件参数中，并使用`OMX_ERRORTYPE`中定义的值。组件应该通过`OMX_EventError`发送下面的错误：

-  组件转移到`OMX_StateInvalid`状态时会发送`OMX_ErrorInvalidState`错误。
-  组件由于资源不足时从`OMX_StateExecuting` 或 `OMX_StatePause` 切换到 `OMX_StateIdle`时会发送`OMX_ErrorResourcesPreempted`错误。
-  组件由于资源丢失而从`OMX_StateIdle` 切换到 `OMX_StateLoaded`时会发送`OMX_ErrorResourcesLost`错误。

#####3.1.1.4.3  OMX_EventMark
组件受到一块标记过的buffer时会产生`OMX_EventMark`事件。组件收到buffer时，他应该比较自身指针和buffer中`pMarkTargetComponent`字段。如果指针相等，组件处理完buffer后应该立即发送一个包含`pMarkData`参数的标记事件。IL客户端可以使用使用此标记事件来计算组件链上的传输延时，或通知组件一个快特殊的buffer已经到达目的地。

#####3.1.1.4.4  OMX_EventPortSettingsChanged
组件改变端口设置时会立刻产生`OMX_EventPortSettingsChanged`事件。例如，视频解码器可能不知道输出视频的帧大小和帧率，因为这些参数在输入比特流中编码。一旦这些组件被解析了，组件改变输出组件上的配置结构并且传递`OMX_EventPortSettingsChanged`事件给IL客户端。

#####3.1.1.4.5  OMX_EventBufferFlag
当一个输出端口发出一个在字段`nFlags`带有`OMX_BUFFERFLAG_EOS`标识的buffer时，组件产生`OMX_EventBufferFlag`事件。事件处理程序中的`nData1`字段指示了输出端口的索引， `nData2`字段指示了包含码流结束（EOS）标志的不可变的｀nFlags｀字段。如果组件不再传递流（例如，组件时一个视频或视频sink），组件处理完带有`OMX_BUFFERFLAG_EOS`的buffer后，应该为此流发送一个`OMX_EventBufferFlag`事件。事件处理程序的`nData1`字段指定了接受buffer的输入端口，｀nData2｀字段指定了含有EOS标志的不可变的`nFlags`字段。

#####3.1.1.4.6  OMX_EventResourcesAcquired
组件处于`OMX_StateWaitForResources`状态时，资源管理器检测到所需资源可用时，组件产生`OMX_EventResourcesAcquired`事件。组件收到这个事件时，它便可以转移状态到`OMX_StateIdle`，并且会所有端口上的buffer分配。

####3.1.1.5  OMX_BUFFERSUPPLIERTYPE
表3-6中的枚举类型｀OMX_BUFFERSUPPLIERTYPE｀指明了管道端口中的供应端口。一个供应端口要么自己分配buffer，要么复用同一组件下的另一个端口的buffer。

| 字段名称 | 值 | 说明 |
| ------------- | ------------- | ------------- |
| OMX_BufferSupplyUnspecified | 0x0 | 提供buffer的端口未指定，或没有优先的提供者。 |
| OMX_BufferSupplyInput | | 输入端口提供buffer。 |
| OMX_BufferSupplyOutput | | 输出端口提供buffer。|

**表 3-6. OpenMAX管道建立时Buffer提供类型**

###3.1.2 结构
本小节讨论了OpenMAX core中定义的数据结构。每个OpenMAX组件的钱两个字段指明了结构的大小和小节3.1.2.4中定义的版本号`OMX_VERSIONTYPE`。分配OpenMAX结构的实例负责填充着两个值。

####3.1.2.1  OMX_COMPONENTREGISTERTYPE
`OMX_COMPONENTREGISTERTYPE｀结构用于组件静态链接到core中时。Core使用此结构加载和运行特定的组件初始化方法。

`OMX_COMPONENTREGISTERTYPE`定义如下.

``` C
typedef struct OMX_COMPONENTREGISTERTYPE
{
  const char * pName;
  OMX_COMPONENTINITTYPE pInitialize;
} OMX_COMPONENTREGISTERTYPE;
```

####3.1.2.2  OMX_COMPONENTINITTYPE Type Definition
`OMX_COMPONENTINITTYPE`类型定义了组件初始化入口的程序指针。定义如下：

```C
typedef OMX_ERRORTYPE (* OMX_COMPONENTINITTYPE)(OMX_IN OMX_HANDLETYPE hComponent);
```
#####3.1.2.2.1  pName
`pName`包含了组件的名称，最大不超过128个字节（包括‘\0’）

#####3.1.2.2.2  pInitialize
`pInitialize`包括了组件初始化函数的指针。

####3.1.2.3  OMX_ComponentRegistered[]
任何静态链接组件的core应该在`OMX_COMPONENTREGISTERTYPE`字段中声明它所有注册的全局组件列表。

####3.1.2.4  OMX_VERSIONTYPE
`OMX_VERSIONTYPE`类型指示了组件或结构的版本。每个结构使用`OMX_VERSIONTYPE`字段指定了结构的OpenMAX版本。对OpenMAX IL 1.0版本，协议版本时1.0.0.0。组件结构也包含了一个厂商特定的组件版本号的`OMX_VERSIONTYPE`字段。


`OMX_VERSIONTYPE`定义如下：

``` C
typedef union OMX_VERSIONTYPE
{
  struct
  {
    OMX_U8 nVersionMajor;
    OMX_U8 nVersionMinor;
    OMX_U8 nRevision;
    OMX_U8 nStep;
  } ;
  OMX_U32 nVersion;
} OMX_VERSIONTYPE;
```

#####3.1.2.4.1  nVersionMajor
`nVersionMajor` 标识主要版本号.

#####3.1.2.4.2  nVersionMinor
`nVersionMinor` 标识次要版本号.

#####3.1.2.4.3  nRevision
`nRevision` 标识修订号.

#####3.1.2.4.4  nStep
`nStep` 步骤号。

####3.1.2.5  OMX_PRIORITYMGMTTYPE
`OMX_PRIORITYMGMTTYPE`类型指示了组件组的被指定的优先级。组件组指的是与一个相同的功能关联的相互依赖的一组组件。组里的所有组件使用相同的组ID和优先级。如果组里的一个组件丢失了资源并停止运行，他们所共同提供的功能也会结束。这种情况下，同一组的所有其他组件应该转移到`OMX_StateLoaded`。组件仅有一个特定的`nGroupID`的行为时原子的。

`OMX_PRIORITYMGMTTYPE`定义如下：

``` C
typedef struct OMX_PRIORITYMGMTTYPE {
  OMX_U32 nSize;
  OMX_VERSIONTYPE nVersion;
  OMX_U32 nGroupPriority;
  OMX_U32 nGroupID;
} OMX_PRIORITYMGMTTYPE;
```

#####3.1.2.5.1  nGroupPriority
｀nGroupPriority｀的值为组件组的优先级。如果组件被指定了这个类型的参数，那么组件所在的组的组件优先级也是这个值。根据定义，0代表了组件组的最高优先级。

指定组件组的具体机制不再本文档讨论范围。

#####3.1.2.5.2  nGroupID
｀nGroupID｀的值是同一个组件组里所有组件的唯一ID。

####3.1.2.6  OMX_BUFFERHEADERTYPE
在一个单一端口的上下文中，每一个数据buffer拥有一个相关联的头，包括了buffer的元信息。IL客户端与每一个与之通信的端口共享buffer头。同理，每一个管道两头的两个端口共享buffer头。另外，如果一个buffer传输到多个端口将buffer头分到每一个端口上去。buffer头的定义如下。

``` C
typedef struct OMX_BUFFERHEADERTYPE
{
  OMX_U32 nSize;
  OMX_VERSIONTYPE nVersion;
  OMX_U8* pBuffer;
  OMX_U32 nAllocLen;
  OMX_U32 nFilledLen;
  OMX_U32 nOffset;
  OMX_PTR pAppPrivate;
  OMX_PTR pPlatformPrivate;
  OMX_U32 nOutputPortPrivate;
  OMX_U32 nInputPortPrivate;
  OMX_HANDLETYPE hMarkTargetComponent;
  OMX_PTR pMarkData;
  OMX_U32 nTickCount;
  OMX_TICKS nTimeStamp;
  OMX_U32 nFlags;
  OMX_U32 nOutputPortIndex;
  OMX_U32 nInputPortIndex;
} OMX_BUFFERHEADERTYPE;
```

#####3.1.2.6.1  pBuffer
`pBuffer`为buffer中数据存储的真实指针，但并不一定时是有效数据的起始位置。更多信息参考3.1.2.6.4描述的`nOffset`。

#####3.1.2.6.2  nAllocLen
`nAllocLen`为buffer中分配的总大小，包括有效的和未用的字节。

#####3.1.2.6.3  nFilledLen
`nFilledLen`为buffer中有效数据的总大小，从`pBuffer`和`nOffset`指定的位置开始。

#####3.1.2.6.4  nOffset
`nOffset`为从buffer开始计算的有效数据的偏移位置。有效数据的指针可以从`nOffset`和`pBuffer`相加得到。

#####3.1.2.6.5  pAppPrivate
`pAppPrivate`为指向IL客户端私有结构的指针。

#####3.1.2.6.6  pPlatformPrivate
`pPlatformPrivate`为指向平台私有结构的指针。分配buffer头结构的core使用这个指针。

#####3.1.2.6.7  pOutputPortPrivate
`pOutputPortPrivate`为使用buffer的输出端口的私有指针。如果buffer头用于输入端口与IL客户端之间的通信，buffer的`pOutputPortPrivate`则不用定义。

#####3.1.2.6.8  pInputPortPrivate
`pInputPortPrivate`为使用buffer的输入端口的私有指针。如果buffer头用于输出端口与IL客户端之间的通信，buffer的`pInputPortPrivate`则不用定义。

#####3.1.2.6.9  hMarkTargetComponent
`hMarkTargetComponent`为处理buffer时需要发出｀OMX_EventMark｀消息的组件的句柄。一个空的句柄表面buffer没有携带任何标记。`OMX_CommandMarkBuffer`命令将次句柄传递给标记的组件。标记的组件，将此句柄复制到标记的buffer中。每个处理此buffer的组件应该使用这个句柄和自己向比较，如果相同，则发出标记消息。组件应该在输入buffer和相应的输出buffer中传递这个字段。

#####3.1.2.6.10  pMarkData
`pMarkData`指针指向IL客户端特定的数据，与`OMX_EventMark`发出的消息标志相关联。当收到这个标记时，IL客户端可以使用这个数据来和其他的标记相区别。命令`OMX_CommandMarkBuffer`提供此指针来标记组件。标记的组件，将此句柄复制到标记的buffer中。组件应该在输入buffer和相应的输出buffer中传递这个字段。

#####3.1.2.6.11  nTickCount
`nTickCount`为一个组件和IL客户端可以更新的计时器，为可选条目，不是所有的组件会更新它。｀nTickCount｀的值以微秒为单位。由于这个值是一个任意起始点的相对值，它不能用于确定绝对时间。


#####3.1.2.6.12  nTimeStamp
`nTimeStamp`为buffer中第一个逻辑单元的时间戳。Buffer中后续数据的时间戳可以通过buffer的持续时间和此时间戳相加得到。组件应该在输入buffer和相应的输出buffer中传递这个字段。

#####3.1.2.6.13  nFlags
`nFlags`字段包含了buffer的特定的标志，例如EOS标志。组件应该在输入buffer和相应的输出buffer中传递这个字段。标志的列表如下：


```C
#define OMX_BUFFERFLAG_EOS 0x00000001
#define OMX_BUFFERFLAG_STARTTIME 0x00000002
#define OMX_BUFFERFLAG_DECODEONLY 0x00000004
#define OMX_BUFFERFLAG_DATACORRUPT 0x00000008
#define OMX_BUFFERFLAG_ENDOFFRAME 0x00000010
```

######3.1.2.6.13.1  OMX_BUFFERFLAG_EOS
如果组件的输出端口没有更多的数据发出，则会设置EOS。因此，一个输出端口应该在最后一个发出的buffer上设置EOS，输出端口什么时候停止发送数据由具体的实现决定。

######3.1.2.6.13.2  OMX_BUFFERFLAG_STARTTIME
流的源（例如，分离器组件）设置包含流的起始时间戳的buffer`OMX_BUFFERFLAG_STARTTIME`标志。起始时间戳对应了起始或跳转操作后第一帧数据显示时间。

流的第一个时间戳不一定是起始时间。例如，在搜索一个特定视频帧的情况下，目标帧可能是一个帧间帧。因此，流的第一帧应该是在目标帧之前的帧内帧。在目标帧所依赖的帧重建完毕后，才能发生目标帧的起始时间。

`OMX_BUFFERFLAG_STARTTIME`标志直接和buffer的时间戳向关联。因此，buffer数据和`OMX_BUFFERFLAG_STARTTIME`标志的关系和传输和时间戳完全一致。

时钟组件收到一个带有`STARTTIME`标志的buffer应该在它的同步端口上使用`OMX_ConfigTimeClientStartTime`调用`OMX_SetConfig`来传递buffer的时间戳。

######3.1.2.6.13.3  OMX_BUFFERFLAG_DECODEONLY
流的源头（例如，一个分离器组件）设置一个只解码不显示的buffer`OMX_BUFFERFLAG_DECODEONLY` 标志。这个标志用于，源跳转到一个帧间帧时，需要首先解出目标所依赖的帧。在这个例子中，源需要将目标所依赖的帧发出但标记他们只能被解码。

`OMX_BUFFERFLAG_DECODEONLY`标记和buffer数据相关联，传输的行为和时间戳完全一致。显示数据的组件应该忽略所有设置了`OMX_BUFFERFLAG_DECODEONLY`标志的buffer。

######3.1.2.6.13.4  OMX_BUFFERFLAG_DATACORRUPT
当IL客户端识别buffer相关的数据损坏时设置`OMX_BUFFERFLAG_DATACORRUPT`标志位。

######3.1.2.6.13.5  OMX_BUFFERFLAG_ENDOFFRAME
`OMX_BUFFERFLAG_ENDOFFRAME`是一个可选的标志位，buffer playload中包含帧结束的最后一个字节时由输出端口设置。任何一个在输出端口上实现了设置`OMX_BUFFERFLAG_ENDOFFRAME`标志的组件应该为输出端口发出的每一个包含EOF的buffer设置这个标志。没有buffer payload可以包含两个独立的帧。

这些限制保证了从输出端口接受到数据的输入端口能够不通过额外的处理检测到EOF。也保证了如果输出端口支持这个标志的话，输入端口能轻易的通过标志位的有或无检测第一帧是否传输完。

####3.1.2.6.14  nOutputPortIndex
`nOutputPortIndex`包含了使用buffer的输出端口的索引。如果一个buffer头用于和IL端口上的输入端口通信的话，此值不用定义。

#####3.1.2.6.15  nInputPortIndex
`nInputPortIndex`包含了使用buffer的输入端口的索引。如果buffer头用于和IL客户端通信的输出端口，此值不用定义。

####3.1.2.7  OMX_PORT_PARAM_TYPE
组件用`OMX_PORT_PARAM_TYPE`结构来定义特定域上的端口数量和起始端口索引。

`OMX_PORT_PARAM_TYPE`定义如下：

``` C
typedef struct OMX_PORT_PARAM_TYPE {
  OMX_U32 nSize;
  OMX_VERSIONTYPE nVersion;
  OMX_U32 nPorts;
  OMX_U32 nStartPortNumber;
} OMX_PORT_PARAM_TYPE;
```

#####3.1.2.7.1  nPorts
`nPorts`为组件给定端口域（音频，视频，图像或其他）上的端口数。

#####3.1.2.7.2  nStartPortNumber
`nStartPortNumber`为组件给定端口域（音频，视频，图像或其他）上的端口索引。给定域的后续端口按顺序编号从`nStartNumber`开始。

#####3.1.2.8  OMX_CALLBACKTYPE
OpenMAX IL包含了一个回调机制，允许组件可以和IL客户端进行下面的通信：

-  IL客户端发出的异步命令成功，失败或产生错误。命令包括通过`OMX_SendCommand`和IL客户端发出的`EmptyThisBuffer`和`FillThisBuffer`。
-  和命令无关的错误发生。例如，组件进入了无法回复的错误并且转移到`OMX_StateInvalid`状态。

为了实现回调，OpenMAX IL定义了3个回调函数：一个通用的事件处理程序和两个和数据流相关的回调（`EmptyBufferDone`和`FillBufferDone`）

IL客户端负责用回调函数入口填充`OMX_CALLBACKTYPE`结构并在初始化的时候传递给OpenMAX core，通常在函数`OMX_GetHandle`中。


`OMX_CALLBACKTYPE`定义如下：

```C
typedef struct OMX_CALLBACKTYPE
{
  OMX_ERRORTYPE (*EventHandler)(
  	OMX_IN OMX_HANDLETYPE hComponent,
    OMX_IN OMX_PTR pAppData,
    OMX_IN OMX_EVENTTYPE eEvent,
    OMX_IN OMX_U32 nData1,
    OMX_IN OMX_U32 nData2,
    OMX_IN OMX_PTR pEventData);
  OMX_ERRORTYPE (*EmptyBufferDone)(
    OMX_IN OMX_HANDLETYPE hComponent,
    OMX_IN OMX_PTR pAppData,
    OMX_IN OMX_BUFFERHEADERTYPE* pBuffer);
  OMX_ERRORTYPE (*FillBufferDone)(
    OMX_IN OMX_HANDLETYPE hComponent,
    OMX_IN OMX_PTR pAppData,
    OMX_IN OMX_BUFFERHEADERTYPE* pBuffer);
} OMX_CALLBACKTYPE;
```

#####3.1.2.8.1  EventHandler
组件使用事件处理函数方法来通知IL客户端什么时候一个所感兴趣的事件在组件内部发生了。枚举类型`OMX_EVENTTYPE`定义了OpenMAX IL事件的集合，可以参看每中事件的定义。`nData1`携带了完成事件的`OMX_COMMANDTYPE` 值或是`OMX_ERRORTYPE`的错误类型。`nData2`携带了更多的事件参数，例如，`OMX_STATETYPE`。`pEventData`包含了事件具体的数据。`pEventData`指针可能包含了额外的与时间有关的数据（例如，标记特定数据）。事件处理函数的调用是阻塞的，所有IL客户端应该在5个毫秒内完成响应，以免长时间阻塞住组件。

方法`EventHandler`定义如下：

``` C
OMX_ERRORTYPE(* OMX_CALLBACKTYPE::EventHandler)(
  OMX_IN OMX_HANDLETYPE hComponent,
  OMX_IN OMX_PTR pAppData,
  OMX_IN OMX_EVENTTYPE eEvent,
  OMX_IN OMX_U32 nData1,
  OMX_IN OMX_U32 nData2,
  OMX_IN OMX_PTR pEventData)
```
参数定义如下：

| 参数 | 说明 |
| -------- | -------- |
| *hComponent* | 调用此函数的组件句柄。 |
| ̛*eEvent* | 组件和IL客户端通信的事件。 |
| *nData1* | 第一个事件特定的整型参数。表3-7描述了在每个事件上下文中的意义。|
| *nData2* | 第二个事件特定的整型参数。表3-7描述了在每个事件上下文中的意义。如果没有用，默认值为0.|
| *pEventData* | 指向事件特定数据的指针。表3-7描述了在每个事件上下文中的意义。|
**表 3-7 每一种事件所用的参数列表**

| eEvent | nData1 | nData2 | pEventData |
| ------- | ------- | ------- | ------- |
| OMX_EventCmdComplete | OMX_CommandStateSet | 状态切换完成 | 无 |
| | OMX_CommandFlush | 端口索引 | 无 |
| | OMX_CommandPortDisable | 端口索引 | 无 |
| | OMX_CommandPortEnable | 端口索引 | 无 |
| | OMX_CommandMarkBuffer | 端口索引 | 无 |
| OMX_EventError | 错误代码 | 0 | 无 |
| OMX_EventMark | 0 | 0 | 标记相连的数据（如果有的话） |
| OMX_EventPortSettingsChanged | 端口索引 | 0 | 无 |
| OMX_EventBufferFlag | 端口索引 |`nFlags`不可变 | 无 |
| OMX_EventResourcesAcquired | 0 | 0 | 无 |
**表 3-7. 事件参数用法**

#####3.1.2.8.2  EmptyBufferDone
组件使用回调`EmptyBufferDone`从一个输入端口返回传递一个buffer给IL客户端。组件设置buffer头中的`nOffset`和`nFilledLength`值来反应buffer中被消耗的部位。例如，如果完全被消耗，则`nFilledLength`设置为0。

为了加快执行组件和IL客户端之间的数据流动，组件在下面情况下使用`EmptyBufferDone`方法把输入buffer返回给IL客户端：

- IL客户端命令状态由`OMX_StateExecuting`或`OMX_StatePause`转移到`OMX_StateIdle` 或`OMX_StateInvalid`。
- IL客户端清空或禁用端口。

`EmptyBufferDone`为阻塞方法，应该在5毫秒以内返回。因此， IL客户端在调用期间可能不选择填充缓冲区，而在调用之外排队处理。

方法`EmptyBufferDone`定义如下：

``` C
OMX_ERRORTYPE(* OMX_CALLBACKTYPE::EmptyBufferDone)(
  OMX_OUT OMX_HANDLETYPE hComponent,
  OMX_OUT OMX_PTR pAppData,
  OMX_OUT OMX_BUFFERHEADERTYPE* pBuffer)
```

The parameters are as follows.

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* | 调用此函数的组件句柄。 |
| *pAppData* | 指向IL客户端定义数据的指针。 |
| *pBuffer* | 指向消耗或返回的`OMX_BUFFERHEADERTYPE`结构类型的指针。 |

#####3.1.2.8.3  FillBufferDone
组件使用回调`FillBufferDone`从输出端口返回数据给IL客户端。组件设置buffer头中的`nOffset`和`nFilledLength`值来反应buffer中被填充的部位。例如，如果没有数据，则`nFilledLength`设置为0。

为了加快执行组件和IL客户端之间的数据流动，组件在下面情况下使用此方法把输出buffer返回给IL客户端：

- IL客户端命令状态由`OMX_StateExecuting`或`OMX_StatePause`转移到`OMX_StateIdle` 或`OMX_StateInvalid`。
- IL客户端清空或禁用端口。

`FillBufferDone`为阻塞方法，应该在5毫秒以内返回。因此， IL客户端在调用期间可能不选择填充缓冲区，而在调用之外排队处理。

`FillBufferDone`定义如下：

``` C
OMX_ERRORTYPE(* OMX_CALLBACKTYPE::FillBufferDone)(
  OMX_OUT OMX_HANDLETYPE hComponent,
  OMX_OUT OMX_PTR pAppData,
  OMX_OUT OMX_BUFFERHEADERTYPE* pBuffer)
```

参数定义如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* | 访问组件的句柄。此句柄为方法`GetHandle`返回的组件句柄。 |
| *pAppData* | 指向IL客户端定义数据的指针。 |
| *pBuffer* | 指向填充或返回的`OMX_BUFFERHEADERTYPE`结构类型的指针。 |

####3.1.2.9  OMX_PARAM_BUFFERSUPPLIERTYPE
`OMX_PARAM_BUFFERSUPPLIERTYPE`结构用于传输buffer提供者的设置或偏好。

`OMX_PARAM_BUFFERSUPPLIERTYPE` 定义如下.

``` C
typedef struct OMX_PARAM_BUFFERSUPPLIERTYPE {
  OMX_U32 nSize;
  OMX_VERSIONTYPE nVersion;
  OMX_U32 nPortIndex;
  OMX_BUFFERSUPPLIERTYPE eBufferSupplier;
} OMX_PARAM_BUFFERSUPPLIERTYPE;
```

#####3.1.2.9.1  nPortIndex
`nPortIndex`为结构适用的组件端口

#####3.1.2.9.2  eBufferSupplier
`eBufferSupplier`字段包括了buffer提供者的索引，如果是一个输入或输出端口。

####3.1.2.10  OMX_TUNNELSETUPTYPE
当IL客户端调用`OMX_SetupTunnel`来连接端口时，方法`ComponentTunnelRequest`使用`OMX_TUNNELSETUPTYPE`结构体来在两个端口之间传输数据。

`OMX_TUNNELSETUPTYPE` 定义如下.

``` C
typedef struct OMX_TUNNELSETUPTYPE
{
  OMX_U32 nTunnelFlags;
  OMX_BUFFERSUPPLIERTYPE eSupplier;
} OMX_TUNNELSETUPTYPE;
```

#####3.1.2.10.1  nTunnelFlags
`nTunnelFlags`整型参数包括了接受此结构的端口所适用的一个或多个比特标志，这些标志包括：

``` C
#define OMX_PORTTUNNELFLAG_READONLY 0x00000001
```

如果标志设置为只读，接受到结构体的输入端口不能改变管道上buffer的内容。

#####3.1.2.10.2  eSupplier
`eSupplier`定义了是输出还是输入端口提供buffer。3.4.1.2小节描述了建立管道的具体调用步骤。

####3.1.2.11  OMX_PARAM_PORTDEFINITIONTYPE
`OMX_PARAM_PORTDEFINITIONTYPE`结构提包含了组件中每个端口的一些通用的字段。一些字段在每个域上都是通用的而其他的字段是每个域特有的。IL客户端使用这个结构体来获取每个端口的通用信息。

`OMX_PARAM_PORTDEFINITIONTYPE` 定义如下.

``` C
typedef struct OMX_PARAM_PORTDEFINITIONTYPE {
  OMX_U32 nSize;
  OMX_VERSIONTYPE nVersion;
  OMX_U32 nPortIndex;
  OMX_DIRTYPE eDir;
  OMX_U32 nBufferCountActual;
  OMX_U32 nBufferCountMin;
  OMX_U32 nBufferSize;
  OMX_BOOL bEnabled;
  OMX_BOOL bPopulated;
  union {
    OMX_AUDIO_PORTDEFINITIONTYPE audio;
    OMX_VIDEO_PORTDEFINITIONTYPE video;
    OMX_IMAGE_PORTDEFINITIONTYPE image;
    OMX_OTHER_PORTDEFINITIONTYPE other;
  } format;
} OMX_PARAM_PORTDEFINITIONTYPE;
```

#####3.1.2.11.1  nPortIndex
`nPortIndex` 是一个标识端口的只读字段。他的值为一个组件上唯一的32比特的数。 同一个组件上的两个不同的端口不能拥有相同的端口号。而不同组件上的端口可以有相同的端口号。

#####3.1.2.11.2  eDir
`eDir`是一个指明端口方向（`OMX_DirInput`或`OMX_DirOutput`）的只读域。

#####3.1.2.11.3  nBufferCountActual
`nBufferCountActual` 表明了端口在poplated之前（如字段bPopulated所示）所需要的buffer的数量。组件应该为这个字段设置一个不小于`nBufferCountMin`的缺省值。

#####3.1.2.11.4  nBufferCountMin
`nBufferCountMin` 是一个指定了端口所需最少buffer数的只读字段。组件应定义一个非零的缺省值。

#####3.1.2.11.5  nBufferSize
`nBufferSize` 是一个指定了端口所需分配的buffer的最小buffer字节数的只读字段。

#####3.1.2.11.6  bEnabled
`bEnabled`为指定了端口是否启用的只读布尔型端口。默认值为`OMX_TRUE`，并可以通过`OMX_SendCommand`方法发送`OMX_CommandPortEnable`和`OMX_CommandPortDisable`来启用/禁用。

端口被禁用是不可以被populated。

#####3.1.2.11.7  bPopulated
`bPopulated`为一个指示端口是否populated的只读bool型字段。当端口上所有`nBufferCountActual`指明的数量`nBufferSize`指明的大小的buffer被分配的时候，端口才是populated的。一个populated的端口应该被启用。启用的端口应该通过转移到`OMX_StateIdle`状态populated和切换到`OMX_StateLoaded`来unpopulated。

#####3.1.2.11.8  eDomain
`eDomain`为指定端口域的只读字段。决定了共同体中的内容，见小节3.1.2.11.9的解释。

#####3.1.2.11.9  format
`format`字段为一个域参数的共同体。具体信息可以见第4章。

###3.1.3 OMX_PORTDOMAINTYPE
表3-8枚举了`OMX_PARAM_PORTDEFINITIONTYPE`结构体中使用的字段，用于定义端口的作用域。

| 字段名称 | 说明 |
| ------- | ------- |
| OMX_PortDomainAudio | 指定了字段格式类型为`OMX_AUDIO_PORTDEFINITIONTYPE`|
| OMX_PortDomainVideo | 指定了字段格式类型为`OMX_VIDEO_PORTDEFINITIONTYPE`|
| OMX_PortDomainImage | 指定了字段格式类型为`OMX_IMAGE_PORTDEFINITIONTYPE`|
| OMX_PortDomainOther | 指定了字段格式类型为`OMX_OTHER_PORTDEFINITIONTYPE`|

**表 3-8. 端口域名称                                  **

###3.1.4 OMX_HANDLETYPE
`OMX_HANDLETYPE`结构体定义了IL客户端可见的组件句柄。组件句柄用于访问所有组件的方法。组件句柄也包含了组件私有数据域的指针。OpenMAX core在加载组件的过程中分配并初始化组件。组件加载完毕后，IL客户端可以安全的访问所有组件的公有方法，虽然某些方法会由于状态不对而返回错误。

##3.2  OpenMAX Core 方法/宏
OpenMAX core实现了IL客户端使用组件的主要接口。为了效率，OpenMAX IL定义了一系列的宏来完成对组件方法一对一的映射。一些宏和方法建议函数在5毫秒或20号码内返回，这取决与函数。5毫秒的超时被标准认为是不需要buffer处理的命令的合理的响应事件。标准认为需要处理buffer的命令处理的合理超时时间为20号码。这里的假设是最长的buffer处理应该低于30号码，对应域每秒30帧视频。这些超时的主要目的是为了使组件集成者可以通过一致性测试很容易的得到组件的响应延时。

这些宏包含下面的内容：

- 获取组件信息（版本，能力）
- 在初始化时设置/获取组件参数。
- 在运行是设置/获取组件参数。
- 分配/释放buffer。
- 发送给OpenMAX组件端口一个充满数据的buffer。
- 发送给OpenMAX组件端口一个空的buffer。
- 发送命令给组件。
- 获得组件的实际状态。
- 获取OpenMAX组件专有参数的引用。

OpenMAX Core也实现了下面的方法：

- 初始化/析构整个OpenMAX IL core。
- 获得一个OpenMAX组件句柄
- 释放一个OpenMAX组件句柄
- 运行时探测系统上所有可用的OpenMAX组件。
- 在OpenMAX组件之间建立数据管道。

当一个方法执行的时间限制被指定时，它并不打算是组件标准一致性的硬性规定。但如果不遵守这个限制，组件文档中应该标明。


###3.2.1 方法的返回值
表3-9列举了每一个方法的所有可能的返回错误值。致命错误指的是组件无法恢复。发生致命错误时组件应该转移到`OMX_StateInvalid`状态。除了最后两列，左右的列对应着调用组件方法的错误返回。最后两列为内部错误时发出的异步错误。

![](img/t3_9.png)

###3.2.2 宏
本小节描述了OpenMAX Core中的宏

表3-10定义了在每一种状态下哪一个宏应该被调用。

![](img/t3_10.png)

表3-10. 合法的组件调用

####3.2.2.1  OMX_GetComponentVersion
宏`OMX_StateInvalid`查询组件并返回具体信息。这是一个阻塞调用。组件应该在5毫秒内返回这个调用。

The macro 定义如下.

``` C
#define OMX_GetComponentVersion (
  hComponent,
  pComponentName,
  pComponentVersion,
  pSpecVersion,
  pComponentUUID )
  ((OMX_COMPONENTTYPE*)hComponent)->GetComponentVersion( \
    hComponent, \
    pComponentName, \
    pComponentVersion, \
    pSpecVersion, \
    pComponentUUID)
```

参数定义如下：

| 参数 | 输入 |
| ------- | ------- |
| *hComponent* [输入] | 执行命令的组件句柄。|
| *pComponentName* [输出] | 组件名字符串的指针。组件名必须是小于127字节加结尾即最大长度为128字节的字符串。例如，一个合法的组件名可以是"OMX.<厂商名>.AUDIO.DSP.MIXER\0"。名字由厂商指定，但需要以"OMX."开始加上厂商指定的字符串。|
| *pComponentVersion* [输出] | 组件所填充的OpenMAX版本结构体的指针。组件会填入版本值。注意组件的版本和在所有结构中的OpenMAX协议的版本不是一样的。厂商组件自行定义组件的版本来确定这个值。|
| *pSpecVersion* [输出] | 组件所填充的OpenMAX版本结构体的执政。`SpecVersion`是组件对应的协议的版本。注意这个版本和结构体的版本可能一样也可能不一样。例如，如果组件是根据2.0版本的协议实现的，但创建IL的客户端是由1.0版本的协议建立的，两者是不同的。|
| *pComponentUUID* [输出] | 组件通用唯一识别码（UUID）的指针，由组件填入。UUID是一个唯一的识别码，由组件在运行期间设置，并且每个组件实例都是唯一的。|

#####3.2.2.1.1  先决条件2
这种方法没有先决条件。

#####3.2.2.1.2  调用顺序实例代码
下面的实例代码展示了调用顺序：

```C
/* detect mismatch between IL client's and component's spec version */
OMX_GetComponentVersion(
  hComp,
  &CompName,
  &CompVersion,
  &CompSpecVersion,
  &CompUUID);
  if (CompSpecVersion != IlClientVersion){
    printf("ERROR: version mismatch\n");
  }
```

####3.2.2.2  OMX_SendCommand
宏`OMX_SendCommand`会调用组件上的一个命令。这是一个非阻塞的调用，有效的命令参数应该在最小5毫秒内返回。通常来说，组件并不在此调用的上下文中执行命令，但如果方案中没有多线程的话可能会选择在上下文中执行。在这两种情况下，组件使用事件回调来通知IL客户端命令执行完毕的结果。如果组件成功执行命令，组件会生成`OMX_EventCmdComplete`回调。如果组件执行命令失败，则生成`OMX_EventError`并将适当的错误以参数传回。

组件可能选择将命令传入队列以便后续执行。唯一的限制就是完成的顺序应该和请求的顺序一致。

宏定义如下：

``` C
#define OMX_SendCommand (
  hComponent,
  Cmd,
  nParam,
  pCmdData)
  ((OMX_COMPONENTTYPE*)hComponent)->SendCommand( \
    hComponent, \
    Cmd, \
    nParam,
    pCmdData)
```

参数如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* [输入] | 执行命令的组件句柄 |
| *Cmd* [输入] | 待组件执行的命令 |
| *nParam* [输入] | 待执行命令的整型参数 |
| *pCmdData*[输入] | 一个指针，包含具体实现的无法用数值型参数来描述的数据|

3.3.6小节描述了每个组件实现的对应的方法。

####3.2.2.3  OMX_CommandStateSet
IL客户端调用此命令来请求组件切换到`nParam`指定的状态中。只有合法的状态转移并且所有先决条件都已满足，组件才可以完成老状态到新状态之间的成功切换。更多信息参见3.1.1.2小节。

如果组件成功转移到新的状态，他会通过事件`OMX_EventCmdComplete` 来通知IL客户端，`nData1`中存放`OMX_CommandStateSet`，`nData2`放入新的状态。如果转移失败，组件应该通过事件`OMX_EventError`通知IL客户端错误。相关错误包括但不限于以下内容：

- `OMX_ErrorSameState`: 组件已经在此状态
- `OMX_ErrorIncorrectStateTransition`: 转移请求非法
- `OMX_ErrorInsufficientResources`: 组件获取转移所需的资源失败

####3.2.2.4  OMX_CommandFlush
IL调用此命令来清空组件上一个或多个组件。`nParam`指定了待刷新端口的缩影。如果值为-1，则应该清空所有端口。

当IL客户端清空一个非供应端口时，端口应该返回所有拥有的buffer给供应端口。如果供应端口为IL客户端，被清空的组件使用`EmptyBufferDone` 和`FillBufferDone`（分别对应输入或输入端口）返回buffer。如果供应端口为一个管道端口，被清空的端口使用使用`EmptyThisBuffer` 和`FillThisBuffer`（分别对应输入或输入端口）返回buffer。

对于每个组件成功清空的端口，组件应该发送一个`OMX_EventCmdComplete`事件， `nData1`中放入`OMX_CommandFlush`，`nData2`中放入独立的端口索引，即使`nParam`使用的是-1。如果清空失败，组件应该使用`OMX_EventError`事件来通知IL客户端错误。

####3.2.2.5  OMX_CommandPortDisable
命令`OMX_CommandPortDisable`禁用一个端口。`nParam`指定了禁用端口的索引。如果`nParam`值为-1，组件应该禁用所有的端口。一个被禁用的端口没有buffer也不连接到IL客户端或是通过管道连接到其他端口。一个被禁用的端口从`OMX_StateLoaded`或`OMX_StateWaitForResources` 转移到`OMX_StateIdle`是不分配buffer。IL客户端可以通过`OMX_SetParameter`改变一个禁用端口的设置或是忽略状态来建立一个管道。因此命令`OMX_CommandPortDisable`配合`OMX_CommandPortEnable`，可以用于动态的改变端口配置或重新连接管道。

端口收到`OMX_CommandPortDisable`必须立刻清除端口定义结构体上的`bEnabled`字段。如过IL客户端禁用的端口是一个非供应端口，IL客户端应该通过`OMX_EmptyThisBuffer`/`OMX_FillThisBuffer`（管道） 或`EmptyBufferDone`/`FillBufferDone`（非管道）返回所有的拥有的buffer。然后，IL客户端应该等待供应端口通过`OMX_FreeBuffer`释放buffer来完成禁用命令。如果被禁用的端口是一个供应端口，分配了buffer， IL客户端应该等待非供应端口通过`OMX_EmptyThisBuffer`或`OMX_FillThisBuffer`返回buffer。然后，IL客户端应该等待供应端口通过`OMX_FreeBuffer`释放buffer来完成禁用命令。

对于每一个组件成功禁用的端口，组件应该发送`OMX_EventCmdComplete`事件，`nData1`中放入`OMX_CommandPortDisable`，`nData2`中放入独立的端口索引，`nParam`使用的是-1。如果禁用失败，组件应该使用`OMX_EventError`事件来通知IL客户端错误。

####3.2.2.6  OMX_CommandPortEnable
`OMX_CommandPortEnable`命令启用一个端口。`nParam`指定了启用端口的索引。如果`nParam`值为-1，组件应该启用所有的端口。启用端口应该遵循组件状态的所有要求。因此，端口应该：

- 组件如果在`OMX_StateLoaded`或`OMX_StateWaitForResources`应该没有buffer分配，而在其他状态应该所有buffer都已分配
- 从 `OMX_StateLoaded`或`OMX_WaitForResources`转移到`OMX_IdleState`时分配buffer。
- 在`OMX_StateExecuting`状态时传输buffer到数据流
- 除了 `OMX_StateLoaded`以外所有其他状态不允许通过`OMX_SetParameter`改变参数。

命令`OMX_CommandPortEnable`配合`OMX_CommandPortDisable`，可以用于动态的改变端口配置或重新连接管道。

端口收到`OMX_CommandPortEnable`后必须立刻设置端口定义结构体中的`bEnabled`字段。如果IL客户端在组件为除`OMX_StateLoaded`或 `OMX_WaitForResources`以外的任何状态启用端口，端口应该通过和`OMX_StateLoaded`到`OMX_StateIdle`相同的调用顺序分配buffer。如果IL客户端当组件为`OMX_Executing`时启用，那么端口应该开始传输buffer。

针对每一个组件成功启用的端口，组件应该发送`OMX_EventCmdComplete`事件，`nData1`中放入`OMX_CommandPortEnable`，`nData2`放入独立的端口索引，即使使用`nParam`为-1来启用。如果端口启用操作失败，组件应该通过`OMX_EventError`事件通知IL客户端。

####3.2.2.7  OMX_CommandMarkBuffer
命令`OMX_CommandMarkBuffer`指示给定的端口标记buffer。`nParam`持有进行标记的端口索引。`OMX_SendCommand`的`pCmdData`参数指向`OMX_MARKTYPE`结构体。结构体中`pMarkTargetComponent`字段持有目标组件的指针，当处理完标记的buffer后会发送事件给目标组件。`pMarkData`字段持有一个指针，指向与标记相关的应用程序特定的数据， 用来给应用程序在一个标记事件中唯一标识这个标记（标识数据的名称）。

当指示标记一个buffer时，组件将在接受到标记命令后标记接受的下一个buffer。源组件是一个特殊情况，它将标记加到其输出buffer队列的下一个buffer。非源组件的情况下，nParam中的端口索引值持有了标记下一个buffer的输入端口的索引。源组件的情况下，nParam中的端口索引值持有了标记下一个buffer的输出端口的索引。

在下列情况下， 多个标记可能会竞争同一个buffer：

- 组件连续收到两个或更多标记命令，两次标记之间没有buffer.
- 两个或多个输入buffer，每个都有一个标记，合并成一个输出buffer，（例如，在一个mixer中）
- 组件收到一个标记命令但下一个buffer已经被标记。

如果多个标记竞争同一个buffer，组件使用第一个收到的标记来标记buffer，并且按收到标记的顺序将剩余的标记应用到后续的buffer中。如果后面没有buffer，组件可以把剩余的标记应用到一个或多个空的buffer中去。

组件成功标记buffer的每一个端口，组件应该发送`OMX_EventCmdComplete`事件，`nData1`中放置`OMX_CommandPortMarkBuffer`，`nData2`放置独立的端口索引。如果标记操作失败，组件应该通过`OMX_EventError`事件通知IL客户端。

buffer头包含了`pMarkTargetComponent`和`pMarkData`字段，意义和`OMX_MARKTYPE`中的字段相同。组件通过从标记命令拷贝`pMarkTargetComponent`和`pMarkData`字段来标记buffer。默认情况下这两个字段是空的（例如，标记buffer之前）。一个组件根据buffer标志位和时间戳简历的元数据规则来从输入buffer向输出buffer传播标记。目标组件不传播标记，而是将两个字段清楚为NULL。

但组件收到buffer，他应该自身指针和pMarkTargetComponent。如果指针匹配，buffer离开组件或成功处理而不需要离开组件后，组件应该立马发送标记事件，包含了pMarkData作为参数。

#####3.2.2.7.1  先决条件
这种方法没有先决条件。


#####3.2.2.7.2  调用顺序实例代码
下面的实例代码展示了调用顺序：

``` C
/* disable every audio port of a component*/
OMX_GetParameter(hComp, OMX_IndexParamAudioInit, &oParam);
for (i=0;i<oParam.nPorts;i++) {
  OMX_SendCommand(
    hComp,
    OMX_CommandPortDisable,
    Param.nStartPortNumber + i,
    0);
}
```

####3.2.2.8  OMX_GetParameter
宏`OMX_GetParameter`取得组件的一个参数。参数`nParamIndex`指示了请求组件的哪一个结构体。调用者在调用此宏之前应该提供结构体的内存并填充`nSize`和`nVersion`字段。如果参数是来自一个端口，调用者也应该在调用此宏之前在`nPortIndex`字段中提供一个有效的端口号。所有组件应该支持每个参数的一组默认值，这样调用者可以得到有效值的结构体。

这个调用为一个阻塞调用。组件应该在20毫秒以内返回这个调用。

宏OMX_GetParameter定义如下：

```C
#define OMX_GetParameter (
  hComponent,
  nParamIndex,
  ComponentParameterStructure)
  ((OMX_COMPONENTTYPE*)hComponent)->GetParameter( \
    hComponent, \
    nParamIndex, \
    ComponentParameterStructure)
```

参数描述如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* [输入] | 执行调用的组件句柄 |
| *nParamIndex* [输入] | 填充的结构体索引。 这个值来自`OMX_INDEXTYPE`结构体 |
| *ComponentParameterStructure* [输入,输出] | 指向IL客户端分配的结构体的指针，由组件填充 |

3.3.7小节描述了每个组件实现的相应的方法。

#####3.2.2.8.1  先决条件
这个宏可以当组件在除`OMX_StateInvalid`外的任何状态时被调用。

#####3.2.2.8.2  调用顺序实例代码
下面的实例代码展示了调用顺序：

```C
/* disable every audio port of a component*/
OMX_GetParameter(hComp, OMX_IndexParamAudioInit, &oParam);
for (i=0;i<oParam.nPorts;i++) {
  OMX_SendCommand(
    hComp,
    OMX_CommandPortDisable,
    oParam.nStartPortNumber + i,
    0);
}
```

####3.2.2.9  OMX_SetParameter
宏`OMX_SetParameter`将发送一个参数结构体给组件。参数`nParamIndex`指示了传递给组件的是哪一个结构体。

调用者应该提供正确的结构体的内存，并在调用宏之前填充结构体中`nSize`和`nVersion`字段。调用者可以在调用之后可以自由的处理该结构，因为组件需要拷贝它需要保留的任何数据。

一些参数结构体包含了只读字段。`OMX_SetParameter`方法将保留只读字段，并且调用者试图改变只读字段时也不会产生错误。

此调用为阻塞调用。组件应该在20毫秒以内返回。

宏`OMX_SetParameter`定义如下：

```C
#define OMX_SetParameter (
  hComponent,
  nParamIndex,
  ComponentParameterStructure)
  ((OMX_COMPONENTTYPE*)hComponent)->SetParameter( \
    hComponent, \
    nParamIndex, \
    ComponentParameterStructure)
```

参数定义如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* [输入] | 执行调用的组件句柄|
| *nIndex* [输入] | 发送的结构体缩影。这个值来自`OMX_INDEXTYPE`枚举类型|
| *ComponentParameterStructure* [输入] | 指向IL客户端分配的结构体的指针，组件用于初始化|


3.3.8小节描述了每个组件实现的相应方法。

#####3.2.2.9.1  先决条件
宏`OMX_SetParameter`仅当组件在`OMX_StateLoaded`或组件被禁用时被调用。

#####3.2.2.9.2  调用顺序实例代码
下面的实例代码展示了调用顺序：

```C
/* force a port to be the supplier */
OMX_GetParameter(hComp, OMX_IndexParamPortDefinition, &oPortDef);
if (oPortDef.eDir == OMX_DirInput){
  oSupplier.eBufferSupplier = OMX_BufferSupplyInput;
} else {
  oSupplier.eBufferSupplier = OMX_BufferSupplyOutput;
}
oSupplier.nPortIndex = nPortIndex;
OMX_SetParameter(hComp, OMX_IndexParamCompBufferSupplier, &oSupplier);
```

####3.2.2.10  OMX_GetConfig
宏`OMX_GetConfig`将从一个组件获得一个配置结构体。此红可以在组件被加载后任何事件被调用。参数`nParamIndex`指示了组件的哪一个结构体被请求。调用者应该在调用这个宏之前提供此结构体的内存并填充`nSize`和`nVersion`字段。如果是配置一个端口，调用者也应该在调用之前在`nPortIndex`字段提供一个有效的端口号。所有的组件应该支持每个配置的默认值，这样调用者可以获得填充正确值的结构体。

此调用为一个阻塞调用。组件应该在5毫秒内返回。

宏`OMX_GetConfig`定义如下：

```C
#define OMX_GetConfig (
  hComponent,
  nConfigIndex,
  ComponentConfigStructure)
  ((OMX_COMPONENTTYPE*)hComponent)->GetConfig( \
    hComponent, \
    nConfigIndex, \
    ComponentConfigStructure)
```

参数定义如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent*[输入] | 执行调用的组件句柄 |
| *nIndex*[输入] | 需要填充的结构体索引。此值来自OMX_INDEXTYPE枚举类型|
| *ComponentConfigStructure*[输入,输出] | 指向IL客户端分配的结构体指针，由组件填充 |


3.3.9小节描述了每个组件实现的相应方法。


#####3.2.2.10.1  先决条件
此宏可以当组件在除了`OMX_StateInvalid`外的任意状态被调用。

#####3.2.2.10.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* Wait until a certain playback position */
do {
  OMX_GetConfig(hClockComp, OMX_IndexConfigTimeCurrentMediaTime,
    oMediaTime);
} while (oMediaStamp.nTimestamp < nTargetTimeStamp);
```

####3.2.2.11  OMX_SetConfig
宏`OMX_SetConfig`将给组件一个配置值。组件加载完后，这个宏可以在任何时间被调用。

调用者应该在调用宏之前提供正确的结构体的内存，并填充结构体中`nSize`和`nVersion`字段。调用者可以在调用之后可以自由的处理该结构，因为组件需要拷贝它需要保留的任何数据。

一些配置结构体包含了只读字段。`OMX_SetConfig`方法将保留只读字段，并且调用者试图改变只读字段时也不会产生错误。

这个调用为一个阻塞调用。组件应该在5毫秒以内返回这个调用。

宏`OMX_SetConfig`定义如下：

```C
#define OMX_SetConfig (
  hComponent,
  nConfigIndex,
  ComponentConfigStructure )
  ((OMX_COMPONENTTYPE*)hComponent)->SetConfig( \
    hComponent, \
    nConfigIndex, \
    ComponentConfigStructure)
```

参数定义如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* [输入] |执行调用的组件句柄|
| *nIndex* [输入] | 发送的结构体索引。此值来自OMX_INDEXTYPE枚举类型|
| *ComponentConfigStructure* [输入] | 指向IL客户端分配的结构体的指针，组件用于初始化 |

3.3.10小节描述了每个组件实现的相应方法。

#####3.2.2.11.1  先决条件
此宏可以当组件在除了`OMX_StateInvalid`外的任意状态被调用。

#####3.2.2.11.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* Change the time scale of the clock component*/
oScale.xScale = 0x00020000; /*2x*/
OMX_SetConfig(hClockComp, OMX_IndexConfigTimeScale, (OMX_PTR)&oScale);
```

####3.2.2.12  OMX_GetExtensionIndex
宏`OMX_GetExtensionIndex`将调用组件从一个标准OpenMAX或厂商扩展的配置或参数翻译为OpenMAX结构体索引。厂商不需要在那些已经在OMX_INDEXTYPE中定义的索引支持这个命令，从而可以降低内存占用。组件可以支持`OMX_INDEXTYPE`中没有的任何标准OpenMAX或厂商扩展的索引。


这个调用为一个阻塞调用。组件应该在5毫秒以内返回这个调用。

宏`OMX_GetExtensionIndex`定义如下：

```C
#define OMX_GetExtensionIndex (
  hComponent,
  cParameterName,
  pIndexType )
  ((OMX_COMPONENTTYPE*)hComponent)->GetExtensionIndex( \
    hComponent, \
    cParameterName, \
    pIndexType)
```

参数定义如下：

| 参数 | 说明|
| ------- | ------- |
| *hComponent* [输入] |执行调用的组件句柄|
| *cParameterName*[输入] | 一个OMX_STRING值，小于128个字符（包括了结尾的null字节）。组件将把这个字符串翻译为一个配置索引|
| *pIndexType* [输出] | 指向OMX_INDEXTYPE结构体的指针，用于接受索引值|

3.3.11小节描述了每个组件实现的相应方法。

#####3.2.2.12.1  先决条件
此宏可以当组件在除了`OMX_StateInvalid`外的任意状态被调用。

#####3.2.2.12.2  调用顺序实例代码
下面的实例代码展示了调用顺序：

```C
/* Set the vendor-specific filename parameter on a reader */
OMX_GetExtensionIndex(
  hFileReaderComp,
  "OMX.CompanyXYZ.index.param.filename",
  &eIndexParamFilename);
OMX_SetParameter(hComp, eIndexParamFilename, &oFileName);
```
####3.2.2.13  OMX_GetState
宏`OMX_GetState`将调用组件来获得组件的当前状态并将组件值放入`pState`指向的地址。组件应该在5毫秒内返回。

宏`OMX_GetState`定义如下：

```C
#define OMX_GetState (
  hComponent,
  pState )
  ((OMX_COMPONENTTYPE*)hComponent)->GetState( \
    hComponent, \
    pState)
```

参数定义如下：

| 参数 | 定义 |
| ------- | ------- |
| *hComponent* [输入] | 执行调用的组件句柄 |
| *pState*[输出]| 指向接受状态的指针。返回的值应该是`OMX_STATETYPE`成员之一。|

3.3.12小节描述了每个组件实现的相应方法。

#####3.2.2.13.1  先决条件
这种方法没有先决条件。

#####3.2.2.13.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
OMX_SendCommand(hComp, OMX_CommandStateSet, OMX_StateIdle, 0);
do {
  OMX_GetState(hComp, &eState);
} while (OMX_StateIdle != eState);
```

####3.2.2.14  OMX_UseBuffer
宏`OMX_UseBuffer`请求组件使用IL客户端已经分配好或一个管道的组件提供的buffer。`OMX_UseBuffer`的实现应该分配buffer头，填充给定的参数，并通过输出参数ppBufferHdr传递回来。

宏`OMX_UseBuffer`应该在下面的情况下执行：

- 当组件在`OMX_StateLoaded`并已经发送了转移到`OMX_StateIdle`的请求。
- 当组件在`OMX_StateWaitForResources`状态，所需求资源可用，并组件已经准备转移到`OMX_StateIdle`状态。
- 当组件处于`OMX_StateExecuting`, `OMX_StatePause`, 或`OMX_StateIdle`在禁用的端口上

这个调用为一个阻塞调用。组件应该在20毫秒以内返回这个调用。

宏`OMX_UseBuffer`定义如下：
```C
#define OMX_UseBuffer(\
  hComponent,\
  ppBufferHdr,\
  nPortIndex,\
  pAppPrivate,\
  nSizeBytes,\
  pBuffer)\
  ((OMX_COMPONENTTYPE*)hComponent->UseBuffer(\
    hComponent,\
    ppBufferHdr,\
    nPortIndex,\
    pAppPrivate,\
    nSizeBytes,\
    pBuffer)
```

参数定义如下：

| 参数 | 说明 |
| --------| ------- |
| *hComponent* [输入] | 执行调用的组件句柄 |
| *ppBufferHdr* [输出] | 指向一个OMX_BUFFERHEADERTYPE结构体的指针的指针，用于接受buffer头的指针。|
| *nPortIndex* [输入] | 指定buffer的端口索引。这个索引与拥有端口的组件相对应。 |
| *pAppPrivate* [输入] | 指针，指向和具体实现内存空间，由buffer提供者负责。|
| *nSizeBytes* [输入] | buffer大小，由字节标识|
| *pBuffer* [输入]| 指向使用的内存buffer空间的指针|

3.3.14小节描述了每个组件实现的相应方法。

#####3.2.2.14.1  先决条件
组件应该处于`OMX_StateLoaded`或`OMX_StateWaitForResources`状态，或调用的端口被禁用。

#####3.2.2.14.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* supplier port allocates buffers and pass them to non-supplier */
for (i=0;i<pPort->nBufferCount;i++)
{
  pPort->pBuffer[i] = malloc(pPort->nBufferSize);
  OMX_UseBuffer(pPort->hTunnelComponent,
                &pPort->pBufferHdr[i],
                pPort->nTunnelPort,
                pPort,
                pPort->nBufferSize,
                pPort->pBuffer[j]);
}
```

####3.2.2.15  OMX_AllocateBuffer
宏OMX_AllocateBuffer将请求组件分配一块新的buffer和buffer头。组件将分配buffer和buffer头并返回buffer头的指针。这个调用为阻塞调用，并且应该在下列条件下进行：

- While the component is in the OMX_StateLoaded state and has already sent a request for the state transition to OMX_StateIdle
- While the component it is in the OMX_StateWaitForResources state, the resources needed are available, and the component is ready to go to the OMX_StateIdle state
- On a disabled port when the component is the OMX_StateExecuting, the OMX_StatePause, or the OMX_StateIdle states.

The OMX_AllocateBuffer macro allocates buffers on a specific port for communication with the IL client only. This macro cannot be used to allocate buffers for tunneled ports. Buffers allocated before a port was configured for tunneling will result in the component failing OMX_SetupTunnel calls to the port.

组件应该在5毫秒以内返回这个调用。

宏`OMX_AllocateBuffer`定义如下：

```C
#define OMX_AllocateBuffer (
  hComponent,
  pBuffer,
  nPortIndex,
  pAppPrivate,
  nSizeBytes )
  ((OMX_COMPONENTTYPE*)hComponent)->AllocateBuffer( \
      hComponent, \
      pBuffer, \
      nPortIndex, \
      pAppPrivate, \
      nSizeBytes)
```

参数定义如下：

| Paramter | Description |
| ------- | ------- |
| hComponent [in] | 执行调用的组件句柄 |
| ppBufferHdr [out] | A pointer to a pointer of an OMX_BUFFERHEADERTYPE structure that receives the pointer to the buffer header.|
| nPortIndex [in] | Selects the port on the component that the buffer will be used with. The port can be found by using the nPortIndex value as an index into the port definition array of the component. |
| pAppPrivate [in] | Initializes the pAppPrivate member of the buffer header structure. |
| nSizeBytes [in] |The size of the buffer to allocate. |

3.3.15小节描述了每个组件实现的相应方法。

#####3.2.2.15.1  先决条件
组件应该处于`OMX_StateLoaded`或`OMX_StateWaitForResources`状态，或调用的端口被禁用。

#####3.2.2.15.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* IL client asks component to allocate buffers */
for (i=0;i<pClient->nBufferCount;i++)
{
  OMX_AllocateBuffer(hComp,
      &pClient->pBufferHdr[i],
      pClient->nPortIndex,
      pClient,
      pClient->nBufferSize);
}
```

####3.2.2.16  OMX_FreeBuffer
宏`OMX_FreeBuffer`将从一个组件中释放一块buffer和buffer头。如果组件只分配buffer，则只释放buffer头。如果组件分配buffer和buffer头，则应该释放buffer和buffer头。因此，组件应该记录哪些buffer是自己分配的，以便执行响应的释放。

这个调用应该在下面的情况下执行：

- 当组件在`OMX_StateIdle`且IL客户端已发送了一个向OMX_StateLoaded转移的请求（例如，在组件停止时）。
- 当组件在`OMX_StateExecuting`, `OMX_StatePause`, 或`OMX_StateIdle`状态时禁用端口。


这个方法可以在任何事件被调用，但如果调用没有按上面描述的规则执行，则可能端口发送`OMX_ErrorPortUnpopulated`错误。在管道中，这个方法由供应端口调用，用于释放供应端口管道上的buffer头。


这个调用为一个阻塞调用。组件应该在20毫秒以内返回这个调用。

宏OMX_FreeBuffer定义如下。

```C
#define OMX_FreeBuffer (
hComponent,
nPortIndex,
pBuffer )
((OMX_COMPONENTTYPE*)hComponent)->FreeBuffer( \
hComponent, \
nPortIndex,
pBuffer)
```
参数定义如下：

| 参数 | 说明 |
| ------- | ------- |
| *hComponent* [输入] | 执行调用的组件句柄 |
| *nPortIndex* [输入] | The index of the port that is using the specified buffer |
| *pBuffer* [输入] | A pointer to an OMX_BUFFERHEADERTYPE structure used to provide or receive the pointer to the buffer header.|

3.3.16小节描述了每个组件实现的相应方法。

#####3.2.2.16.1  先决条件
组件应处于`OMX_StateIdle`状态或端口被禁用。

#####3.2.2.16.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* supplier port frees buffers */
for (i=0;i<pPort->nBufferCount;i++)
{
free(pPort->pBuffer[i]);
pPort->pBuffer[i] = 0;
OMX_FreeBuffer(pPort->hTunnelComponent,
pPort->nTunnelPort,
pPort->pBufferHdr[i]);
pPort->pBufferHdr[j] = 0;
}
```

####3.2.2.17  OMX_EmptyThisBuffer
The OMX_EmptyThisBuffer macro will send a filled buffer to an input port of a component. When the buffer contains data, the value of the nFilledLength field of the buffer header will not be zero. If the buffer contains no data, the value of nFilledLength is 0x0. The OMX_EmptyThisBuffer macro is invoked to pass buffers containing data when the component is in or making a transition to the OMX_StateExecuting or in the OMX_StatePaused state.

When a port is non-tunneled, buffers sent to OMX_EmptyThisBuffer are returned to the IL client with the EmptyBufferDone callback once they have been emptied.

When a port is tunneled, buffers sent to OMX_EmptyThisBuffer are sent to the tunneled port once they are emptied so long as the component is in the OMX_StateExecuting state. Buffers are returned to the input port that supplied them using OMX_EmptyThisBuffer whenever the tunneled port is flushed or disabled. Buffers are also returned to the input port that supplied them when the component calling OMX_FillThisBuffer is transitioning from the OMX_StateExecuting state or the
OMX_StatePaused state to the OMX_StateIdle state.

This call is a non-blocking call since the component will queue the buffer and return immediately. The buffer will be emptied later at the proper time. If the parameter nInputPortIndex in the buffer header does not specify a valid input port, the component returns OMX_ErrorBadPortIndex. The component should return from this call within five msec.

宏`OMX_EmptyThisBuffer`定义如下：
```C
#define OMX_EmptyThisBuffer (
hComponent,
pBuffer )
((OMX_COMPONENTTYPE*)hComponent)->EmptyThisBuffer( \
hComponent, \
pBuffer)
```
参数定义如下：

| Parameter | Description |
| ------- | ------- |
| hComponent [in] | 执行调用的组件句柄|
| pBuffer [in] | A pointer to an OMX_BUFFERHEADERTYPE structure that is used to provide or receive the pointer to the buffer header. The buffer header shall specify the index of the input port that receives the buffer |

3.3.17小节描述了每个组件实现的相应方法。

#####3.2.2.17.1  先决条件
The component must be in the appropriate state as shown in Table 3-10.

#####3.2.2.17.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* deliver full buffer */
if (pPort->hTunnelComponent)
OMX_EmptyThisBuffer(pPort->hTunnelComponent, pBuffer);
else
pCallbacks->FillBufferDone(hComp, pBuffer,
pPort->pCallbackAppData);
```

####3.2.2.18  OMX_FillThisBuffer
The OMX_FillThisBuffer macro will send an empty buffer to an output port of a component. The OMX_FillThisBuffer macro is invoked to pass buffers containing no data when the component is in or making a transition to the OMX_StateExecuting
state or is in the OMX_StatePaused state.

When a port is non-tunneled, buffers sent to OMX_FillThisBuffer return to the IL client with the FillBufferDone callback once they have been filled.

When a port is tunneled, buffers sent to OMX_FillThisBuffer are sent to the tunneled port once they are filled so long as the component is in the OMX_StateExecuting state. Buffers are returned to the output port that supplied them using OMX_FillThisBuffer whenever the tunneled port is flushed or disabled. Buffers are also returned to the output port that supplied them when the component that calls OMX_FillThisBuffer is transitioning from the OMX_StateExecuting state or OMX_StatePaused state to the OMX_StateIdle state.

This call is a non-blocking call since the component will queue the buffer and return immediately. The buffer will be filled later at the proper time. If the parameter nOutputPortIndex in the buffer header does not specify a valid output port, the component returns OMX_ErrorBadPortIndex. The component should return from this call within five msec.

宏`OMX_FillThisBuffer`定义如下：

```C
#define OMX_FillThisBuffer (
hComponent,
pBuffer )
((OMX_COMPONENTTYPE*)hComponent)->FillThisBuffer( \
hComponent, \
pBuffer)
```

参数定义如下：

| Parameter | Description |
| ------- | ------- |
| hComponent [in] | 执行调用的组件句柄|
| pBuffer [in] | A pointer to an OMX_BUFFERHEADERTYPE structure used to provide or receive the pointer to the buffer header. The buffer header shall specify the index of the input port that receives the buffer. |

3.3.18小节描述了每个组件实现的相应方法。

#####3.2.2.18.1  先决条件
The component must be in the appropriate state as shown in Table 3-10.

#####3.2.2.18.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* On a port enable, if tunneling and an input and not supplier */
/* then give buffers to supplier port */
if (pPort->hTunnelComponent &&
(pPort->oPortDef.eDir == OMX_DirInput) &&
(pPort->eSupplierSetting == OMX_BufferSupplyInput) )
{
for (i=0;i<pPort->nBuffers;i++){
OMX_FillThisBuffer(pPort->hTunnelComponent,
pPort->ppBufferHdrs[i]);
}
}
```
###3.2.3 Functions
This section describes the functions in the OpenMAX IL API.

####3.2.3.1  OMX_Init
The OMX_Init method initializes the OpenMAX core. OMX_Init shall be the first call made into OpenMAX and should be executed only one time without an intervening OMX_Deinit call. If OMX_Init is called twice, OMX_ErrorNone is returned but the
init request is ignored. The core should return from this call within 20 msec.

The usage of OMX_Init() is as follows.

OMX_API OMX_ERRORTYPE OMX_APIENTRY OMX_Init()

#####3.2.3.1.1  先决条件
This method has no prerequisites.

#####3.2.3.1.2  Results/Outputs for This Method
If the command successfully executes, the return code will be OMX_ErrorNone. Otherwise, the appropriate OpenMAX error will be returned. The OpenMAX core functions are ready to be used when this function returns successfully.

#####3.2.3.1.3  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* Initialize OpenMax and create some components */
OMX_Init();
OMX_GetHandle(hMp3Decoder, "OMX.CompanyXYZ.mp3.decoder", pAppData, pCallbacks);
OMX_GetHandle(hAudioMixer, "OMX.CompanyXYZ.audio.mixer", pAppData, pCallbacks);
```

####3.2.3.2  OMX_Deinit
The OMX_Deinit method de-initializes the OpenMAX core. OMX_Deinit should be the last call made into the OpenMAX core after all OpenMAX-related resources have been released. The core should return from this call within 20 msec. While it may be
preferable to have the core command each of the components back to the loaded state and then de-initialize them, doing so may require more than the recommended 20 msec call time. It further requires the OpenMAX core to track all component handles, which may add unnecessary complexity for some platforms.

The OMX_Deinit method usage is as follows.

``` C
OMX_API OMX_ERRORTYPE OMX_APIENTRY OMX_Deinit()
```

#####3.2.3.2.1  先决条件
The use of OMX_Deinit requires that all component handles in the system have been released, implying that all resources associated with components have been freed.

####3.2.3.2.2  Results/Outputs for This Method
The use of OMX_Deinit returns OMX_ERRORTYPE. If the command successfully executes, the return code will be OMX_ErrorNone. Otherwise, the appropriate OpenMAX error will return.

#####3.2.3.2.3  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* Determine if a component of a particular name exists. */
OMX_Init();
eError = OMX_ErrorNone;
for (i=0; OMX_ErrorNone == eError; i++)
{
eError = OMX_ComponentNameEnum(szCompEnumName, 256, i);
if ((OMX_ErrorNone == eError) &&
(!strcmp(szCompEnumName, szComponentName))
{
OMX_Deinit();
return OMX_TRUE;
}
}
OMX_Deinit();
return OMX_FALSE;
```
####3.2.3.3  OMX_ComponentNameEnum
The OMX_ComponentNameEnum method will enumerate through all the names of recognized components in the system to detect all the components in the system run-time. There is no strict ordering to the enumeration of component names, although each name shall be enumerated only once. If the OpenMAX core supports run-time installation of new components, it is required to detect newly installed components only when the first call to enumerate component names occurs (i.e., when the value of nIndex is 0x0).

方法`OMX_ComponentNameEnum`定义如下：

```C
OMX_API OMX_ERRORTYPE OMX_APIENTRY OMX_ComponentNameEnum(
OMX_OUT OMX_STRING cComponentName,
OMX_IN OMX_U32 nNameLength,
OMX_IN OMX_U32 nIndex
)
```

参数定义如下：

| Parameter | Description |
| ------- | ------- |
| cComponentName [out] | A pointer to a null-terminated string with the component name. Component names are strings limited to less than 127 bytes in length plus the trailing null for a maximum length of 128 bytes. An example of a valid component name is "OMX.<vendor_name>.AUDIO.DSP.MIXER\0". The name shall start with "OMX." concatenated to a vendor-specified string. |
| nNameLength [in] | The number of characters in the cComponentName string. Since all component name strings are restricted to less than 128 characters, not including the trailing null, the caller should provide an input string of at least 128 characters.|
| nIndex [in] | A number containing the enumeration index for the component. Multiple calls to OMX_ComponentNameEnum with increasing values of nIndex will enumerate through the component names in the system until OMX_ErrorNoMore returns. The value of nIndex is 0 to N-1, where N is the number of installed components in the system. |

#####3.2.3.3.1  先决条件
OMX_ComponentNameEnum can be called after the OMX_Init function.

#####3.2.3.3.2  Results/Outputs for This Method
If OMX_ComponentNameEnum successfully executes, the return code will be OMX_ErrorNone. When the value of nIndex exceeds the number of components in the system minus 1, OMX_ErrorNoMore will be returned. Otherwise, the appropriate OpenMAX error will be returned.

#####3.2.3.3.3  调用顺序实例代码
下面的实例代码展示了调用顺序：

```C
/* print a list of all components */
eError = OMX_ErrorNone;
for (i=0; OMX_ErrorNoMore != eError; i++)
{
eError = OMX_ComponentNameEnum(szCompName, 256, i);
if (OMX_ErrorNone == eError)
printf("Component %i: %s\n", szCompName);
}
```

####3.2.3.4  OMX_GetHandle
The OMX_GetHandle method will locate the component specified by the component name given, load that component into memory, and validate it. If the component is valid, OMX_GetHandle will invoke the component's methods to fill the component handle
and set up the callbacks. The OMX_GetHandle method will allocate the actual OMX_HANDLETYPE structure, ensures it is populated correctly, and then updates the value of *pHandle with a pointer to the newly created handle. The component should return from this call within 20 msec.

Each time the OMX_GetHandle function returns successfully, a new component instance is created. The IL client shall configure the newly created component, which is in the OMX_StateLoaded state, before the component can be used.

Since components are requested by name, a naming convention is defined. OpenMAX component names are NULL terminated strings with the following format:

“OMX.<vendor_name>.<vendor_specified_convention>”.

No standardization among component names is dictated across different vendors.

`OMX_GetHandle`定义如下：

```C
OMX_API OMX_ERRORTYPE OMX_APIENTRY OMX_GetHandle(
OMX_OUT OMX_HANDLETYPE * pHandle,
OMX_IN OMX_STRING cComponentName,
OMX_IN OMX_PTR pAppData,
OMX_IN OMX_CALLBACKTYPE * pCallBacks
)
```

参数定义如下：

| Parameter |  Description |
| ------ | ------ |
| pHandle [out] | A pointer to OMX_HANDLETYPE to be filled in by this method. |
| cComponentName [in] | A pointer to a null-terminated string with the component name. Component names are strings limited to less than 128 bytes in length plus the trailing null for a maximum length of 128 bytes. An example of a valid component name is "OMX.<vendor_name>.AUDIO.DSP.MIXER\0". The name shall start with "OMX." concatenated to a vendor-specified string. |
| pAppData [in] | A pointer to an IL client-defined value that will be returned during callbacks so that the IL client can identify the source of the callback. |
| pCallBacks [in] |A pointer to an OMX_CALLBACKTYPE structure containing the  callbacks that the component will use for this IL client.|

#####3.2.3.4.1  先决条件
The OpenMAX core shall be initialized.

#####3.2.3.4.2  Results/Outputs for This Method
If successful, the function returns a valid component handle to the IL client.

#####3.2.3.4.3  调用顺序实例代码
下面的实例代码展示了调用顺序：

```C
/* determine maximum number of instantiations of a component */
eError = OMX_ErrorNone;
for (i=0; OMX_ErrorNone == eError; i++)
{
  eError = OMX_GetHandle(&hComp[i],
  szComponentName,
  pAppData,
  pCallbacks);
}
printf("Created %i instantiations.\n",i);
```

####3.2.3.5  OMX_FreeHandle
The OMX_FreeHandle method will free a handle allocated by the OMX_GetHandle method. The component should return from this call within 20 msec. The IL client should call OMX_FreeHandle only when the component is in the OMX_StateLoaded or the OMX_StateInvalid state; calling OMX_FreeHandle from any other state may result in the component taking longer than the recommended 20 msec execution time, and is provided only as a failure recovery mechanism.

`OMX_FreeHandle`定义如下：

```C
OMX_API OMX_ERRORTYPE OMX_APIENTRY OMX_FreeHandle(
  OMX_IN OMX_HANDLETYPE hComponent )
```

The single parameter is as follows.

| Parameter | Description |
| ------ | ------ |
| hComponent [in] | The handle of the component to freed. |

#####3.2.3.5.1  先决条件
The component should be in the OMX_StateLoaded or the OMX_StateInvalid state when this method is called.

#####3.2.3.5.2  Results/Outputs for This Method
All resources associated with the components are freed.

#####3.2.3.5.3  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* stop executing component and clean up component */
OMX_SendCommand(hComp, OMX_CommandStateSet, OMX_StateIdle, 0);
OMX_SendCommand(hComp, OMX_CommandStateSet, OMX_StateLoaded, 0);
do {
OMX_GetState(hComp, &eState);
} while (OMX_StateLoaded != eState);
OMX_FreeHandle(hComp);
```

####3.2.3.6  OMX_SetupTunnel
The OMX_SetupTunnel method sets up tunneled communication between an output port and an input port. This method is an actual method and not a defined macro. The OMX_SetupTunnel method will make calls to the component’s
ComponentTunnelRequest() method to set up the tunnel.

When setting up non-tunneled communication for an input port, the value of the hOutput parameter shall be 0x0. When setting up non-tunneled communication for an output port, the value of hInput shall be 0x0.

When setting up tunneled communication between an output port and an input port, the method first issues a call to ComponentTunnelRequest() on the component with the output port. If the call is successful, a second call to ComponentTunnelRequest() on the component with the input port is made. Should either call to ComponentTunnelRequest() fail, the method will set up both the output and input ports for non-tunneled communication.

The components may negotiate proprietary communication in place of tunneled communication so long as both the output and input ports can support proprietary communication. An IL client cannot disambiguate between tunneled and proprietary
communication.

The component should return from this call within 20 msec.

This method is unsupported by base profile components, which shall return OMX_ErrorNotImplemented .
For a detailed description of the process to set up a data tunnel between two components, see section 3.4.1.2.
OMX_SetupTunnel is defined as follows.

```C
OMX_API OMX_ERRORTYPE OMX_APIENTRY OMX_SetupTunnel(
OMX_IN OMX_HANDLETYPE hOutput,
OMX_IN OMX_U32 nPortOutput,
OMX_IN OMX_HANDLETYPE hInput,
OMX_IN OMX_U32 nPortInput
)
```
参数定义如下：
| Parameter | Description |
| ------ | ------ |
| hOutput [in] | The handle of the component containing the output port used in the tunnel, where the output port is identified by the nPortOutput parameter. By definition, an output port has the direction OMX_DirOutput. If the value of this parameter is 0x0, the hPortInput port on the hInput component will be set up for non-tunneled communication.|
| nPortOutput [in] | Indicates the output port of the component specified by hOutput that is to be used for tunneled or proprietary communication.|
| hInput [in] |The handle of the component containing the input port used in the tunnel, where the input port is identified by the nPortInput parameter. By definition, an input port has the direction OMX_DirInput. If the value of this parameter is 0x0, the hPortOutput port on the hOutput component will be set up for non-tunneled communication. |
| nPortInput [in] | Indicates the input port of the component specified by hInput that is to be used for tunneled or proprietary communication. |

#####3.2.3.6.1  先决条件
Each component that is being tunneled shall be in the OMX_StateLoaded state, or its port shall be disabled.

#####3.2.3.6.2  Results/Outputs for This Method
If the method returns successfully when both an output and input component are supplied, tunneled or proprietary communication has been set up between the specified output and input ports. When only an output or an input component is supplied or if an error occurs during processing, the ports are set up for non-tunneled communication.

#####3.2.3.6.3  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* set up tunnel between two components then transition to idle */
OMX_SetupTunnel(hCompA, nCompAOutPort, hCompB, nCompBInPort);
OMX_SendCommand(hCompA, OMX_CommandStateSet, OMX_StateIdle, 0);
OMX_SendCommand(hCompB, OMX_CommandStateSet, OMX_StateIdle, 0);
```

##3.3  OpenMAX Component Methods and Structures
OpenMAX components are defined in the OMX_Component.h header file. The structure OMX_COMPONENTTYPE holds the data fields and function entry points for a component.

###3.3.1 nSize
nSize is the size of the structure in bytes. This value shall be specified when this structure is used as either an input to or an output from a function.

###3.3.2 nVersion
nVersion is the version of the OpenMAX specification that the structure is built against. The creator of this structure is responsible for initializing this value. Every user of this structure should verify that it knows how to use the exact version of this structure.

###3.3.3 pComponentPrivate
pComponentPrivate is a pointer to the component private data area. The component allocates and initializes this member when the component is first loaded. The application should not access this data area.

###3.3.4 pApplicationPrivate
pApplicationPrivate is a pointer to the application private data area. The component initializes this field during the call to OMX_SetCallbacks, as this field is provided back to the IL client when the component issues callbacks..

###3.3.5 GetComponentVersion
The IL client calls the GetComponentVersion component method via the OMX_GetComponentVersion core macro. See the definition of OMX_GetComponentVersion in section 3.2.2.1 for a description of its semantics.

`GetComponentVersion`定义如下：

```C
OMX_ERRORTYPE (*GetComponentVersion)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_OUT OMX_STRING pComponentName,
OMX_OUT OMX_VERSIONTYPE* pComponentVersion,
OMX_OUT OMX_VERSIONTYPE* pSpecVersion,
OMX_OUT OMX_UUIDTYPE* pComponentUUID);
```
###3.3.6 SendCommand
The IL client calls the SendCommand component method via the OMX_SendCommand core macro. See the definition of OMX_SendCommand in section 3.2.2.2 for a description of its semantics.

`SendCommand`定义如下：
```C
OMX_ERRORTYPE (*SendCommand)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_COMMANDTYPE Cmd,
OMX_IN OMX_U32 nParam,
OMX_IN OMX_PTR pCmdData);
```

###3.3.7 GetParameter
The IL client or a tunneled component calls the GetParameter component method via the OMX_GetParameter core macro. See the definition of OMX_GetParameter in section 3.2.2.8 for a description of its semantics.

GetParameter is defined as follows.
```C
OMX_ERRORTYPE (*GetParameter)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_INDEXTYPE nParamIndex,
OMX_INOUT OMX_PTR ComponentParameterStructure);
```
###3.3.8 SetParameter
The IL client or a tunneled component calls the SetParameter component method via the OMX_SetParameter core macro. See the definition of OMX_SetParameter in section 3.2.2.9 for a description of its semantics.

`SetParameter`定义如下：
```C
OMX_ERRORTYPE (*SetParameter)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_INDEXTYPE nIndex,
OMX_IN OMX_PTR ComponentParameterStructure);
```

###3.3.9 GetConfig
The IL client calls the GetConfig component method via the OMX_GetConfig core macro. See the definition of OMX_GetConfig in section 3.2.2.10 for a description of its semantics.
`GetConfig`定义如下：

```C
OMX_ERRORTYPE (*GetConfig)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_INDEXTYPE nIndex,
OMX_INOUT OMX_PTR pComponentConfigStructure);
```

###3.3.10  SetConfig
The IL client calls the SetConfig component method via the OMX_SetConfig core macro. See the definition of OMX_SetConfig in section 3.2.2.11 for a description of its semantics.

`SetConfig`定义如下：

```C
OMX_ERRORTYPE (*SetConfig)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_INDEXTYPE nIndex,
OMX_IN OMX_PTR pComponentConfigStructure);
```
###3.3.11  GetExtensionIndex
The IL client calls the GetExtenstionIndex component method via the OMX_GetExtensionIndex core macro. See the definition of
OMX_GetExtensionIndex in section 3.2.2.12 for a description of its semantics.

`GetExtensionIndex`定义如下：

```C
OMX_ERRORTYPE (*GetExtensionIndex)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_STRING cParameterName,
OMX_OUT OMX_INDEXTYPE* pIndexType);
```

###3.3.12  GetState
The IL client calls the GetState component method via the OMX_GetState core macro. See the definition of OMX_GetState in section 3.2.2.13 for a description of its semantics.

`GetState`定义如下：
```C
OMX_ERRORTYPE (*GetState)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_OUT OMX_STATETYPE* pState);
```
###3.3.13  ComponentTunnelRequest
The OMX_ComponentTunnelRequest method will interact with another OpenMAX component to determine if tunneling is possible and to set up the tunneling if it is possible. The return codes for this method can determine if tunneling is not possible or if proprietary communication or tunneling is used.

The interop profile-conformant component shall support tunneling to a component with compatible parameters. The component may also support proprietary communication. If proprietary communication is supported, the negotiation of proprietary communication is performed in a vendor-specific way. The only requirement is that the proper result be returned. The details of the proprietary communication setup are left to the vendor’s component implementer.


The ComponentTunnelRequest method is invoked on both components that support the tunneling communication. When this method is invoked on the component that provides the output port, the component will do the following:

1. Indicate its supplier preference in pTunnelSetup.
2. Set the OMX_PORTTUNNELFLAG_READONLY flag to indicate that buffers from this output port are read-only and that the buffers cannot be shared through components or modified.

When this method is invoked on the component that provides the input port, the component will do the following:

1. Check the data compatibility between the ports using one or more GetParameter calls.
2. Review the buffer supplier preferences of the output port and use OMX_SetParameter with index OMX_IndexParamCompBufferSupplier to inform the output port of which port supplies the buffers.

If this method is invoked with a NULL parameter for the pTunnelComp parameter, the port should be set up for non-tunneled communication with the IL client.

The component should return from this call within five msec.

`ComponentTunnelRequest`定义如下：
``` C
OMX_ERRORTYPE (*ComponentTunnelRequest)(
OMX_IN OMX_HANDLETYPE hComp,
OMX_IN OMX_U32 nPort,
OMX_IN OMX_HANDLETYPE hTunneledComp,
OMX_IN OMX_U32 nTunneledPort,
OMX_INOUT OMX_TUNNELSETUPTYPE* pTunnelSetup);
```
参数定义如下：

| Parameter | Description |
| ------- | ------- |
| hComp [in] | The handle of the target component of the RequestTunnel call and one of the components that will participate in the tunnel. |
| nPort [in] | The index of the port belonging to hComp that will participate in the tunnel. |
| hTunneledComp [in] | The handle of the other component that participates in the tunnel. When this parameter is NULL, the port specified in nPort should be configured for non-tunneled communication with the IL client. |
| nTunneledPort [in] | The index of the port belonging to hTunneledComp that participates in the tunnel.|
| pTunnelSetup [in,out] | The structure that contains data for the tunneling negotiation between components. The supplier field can be filled by both components; the callbacks field is filled by the output port component. The read-only flag can be applied by both components.|

####3.3.13.1  先决条件
The component shall be in the OMX_StateLoaded state.

####3.3.13.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* Translate a SetupTunnel call to two ComponentTunnelRequest calls */
pCompOut = (OMX_COMPONENTTYPE *)hOutput;
pCompIn = (OMX_COMPONENTTYPE *)hInput;
pCompOut->ComponentTunnelRequest(hOutput, nPortOutput, hInput,
nPortInput, &oTunnelSetup);
pCompIn->ComponentTunnelRequest(hInput, nPortInput, hOutput,
nPortOutput, &oTunnelSetup);
```

###3.3.14  UseBuffer
The IL client or a tunneled component calls the UseBuffer component method via the OMX_UseBuffer core macro. See the definition of OMX_UseBuffer in section 3.2.2.14 for a description of its semantics.

`UseBuffer`定义如下：

```C
OMX_ERRORTYPE (*UseBuffer)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_INOUT OMX_BUFFERHEADERTYPE** ppBufferHdr,
OMX_IN OMX_U32 nPortIndex,
OMX_IN OMX_PTR pAppPrivate,
OMX_IN OMX_U32 nSizeBytes,
OMX_IN OMX_U8* pBuffer);
```

###3.3.15  AllocateBuffer
The IL client calls the AllocateBuffer component method via the OMX_AllocateBuffer core macro. See the definition of OMX_AllocateBuffer in section 3.2.2.15 for a description of its semantics.

`AllocateBuffer`定义如下：
```C
OMX_ERRORTYPE (*AllocateBuffer)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_INOUT OMX_BUFFERHEADERTYPE** pBuffer,
OMX_IN OMX_U32 nPortIndex,
OMX_IN OMX_PTR pAppPrivate,
OMX_IN OMX_U32 nSizeBytes);
```

###3.3.16  FreeBuffer
The IL client or a tunneled component calls the FreeBuffer component method via the OMX_FreeBuffer core macro. See the definition of OMX_FreeBuffer in section 3.2.2.16 for a description of its semantics.

`FreeBuffer`定义如下：

```C
OMX_ERRORTYPE (*FreeBuffer)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_U32 nPortIndex,
OMX_IN OMX_BUFFERHEADERTYPE* pBuffer);
```

###3.3.17  EmptyThisBuffer
The IL client or a tunneled component calls the EmptyThisBuffer component method via the OMX_EmptyThisBuffer core macro. See the definition of OMX_EmptyThisBuffer in section 3.2.2.17 for a description of its semantics.

`EmptyThisBuffer`定义如下：
```C
OMX_ERRORTYPE (*EmptyThisBuffer)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_BUFFERHEADERTYPE* pBuffer);
```
###3.3.18  FillThisBuffer
The IL client or a tunneled component calls the FillThisBuffer component method via the OMX_FillThisBuffer core macro. See the definition of OMX_FillThisBuffer in section 3.2.2.18 for a description of its semantics.

`FillThisBuffer`定义如下：

```C
OMX_ERRORTYPE (*FillThisBuffer)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_BUFFERHEADERTYPE* pBuffer);
```

###3.3.19  SetCallbacks
The SetCallbacks method will allow the core to transfer the callback structure from the IL client to the component. This is a blocking call. The component should return from this call within five msec.

`SetCallbacks`定义如下：
```C
OMX_ERRORTYPE (*SetCallbacks)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_CALLBACKTYPE* pCallbacks,
OMX_IN OMX_PTR pAppData);
```
参数定义如下：

| Parameter | Description |
| ------ | ------ |
| hComponent [in] |The handle of the component that executes the call. |
| pCallbacks [in] | A pointer to an OMX_CALLBACKTYPE structure that is used to provide the callback information to the component. |
| pAppData [in] | A pointer to a value that the IL client has defined (for example, a pointer to a data structure) that allows the callback in the IL client to determine the context of the call. |

####3.3.19.1  先决条件
The component shall be in the OMX_StateLoaded state.

####3.3.19.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* On GetHandle (for statically linked components):
create component, initialize it, and set its callbacks */
pComp = (OMX_COMPONENTTYPE *)malloc(sizeof(OMX_COMPONENTTYPE));
hHandle = (OMX_HANDLETYPE)pComp;
pComp->nVersion = version_1_0;
pComp->nSize = sizeof(OMX_COMPONENTTYPE);
OMX_ComponentRegistered[i].pInitialize(hHandle);
pComp->SetCallbacks(hHandle, pCallBacks, pAppData);
```

###3.3.20  ComponentDeinit
The core calls the ComponentDeinit function when the core needs to dispose of a component.

`ComponentDeinit`定义如下：

```C
OMX_ERRORTYPE (*ComponentDeInit)(
OMX_IN OMX_HANDLETYPE hComponent);
```

The single parameter is as follows.

| Parameter | Description |
| ------ | ------ |
| hComponent [in] | The handle of the component that executes the call. |

There are no prerequisites for this method. The IL client may execute this function regardless of component state so that de-initialization is guaranteed even on components that are unresponsive to state changes. However, executing ComponentDeinit when the component is in the OMX_StateLoaded state is recommended for proper shutdown.

####3.3.20.2  调用顺序实例代码
下面的实例代码展示了调用顺序：


```C
/* On FreeHandle: de-initialize component and destroy it */
pComp = (OMX_COMPONENTTYPE*)hComponent;
(pComp->ComponentDeInit)(hComponent);
OMX_OSAL_Free(pComp);
```

##3.4  Calling Sequences
This section describes how the IL client, the OpenMAX core, and the components dynamically interact in a few meaningful use cases, namely initialization, de-initialization, data flow, data tunneling setup, and data flow in the case of data tunneling and dynamic port reconfiguration. The interaction between the core, the components, and the possible implementation of a resource manager is also described.

###3.4.1 Initialization
This section describes the operations for initializing the OpenMAX components. The components can be handled directly by the IL client, can be tunneled to each other, or both. The tunneled and non-tunneled cases are distinguished for clarity, but the two cases can be both present in the component framework.

####3.4.1.1  Non-tunneled Initialization
Figure 3-3 shows how an IL client should initialize an OpenMAX component.

![](img/3_3.png)

**Figure 3-3. Component Initialization**

First, the IL client shall call the OMX_GetHandle function, which activates the actual component creation (1.1) by the core. Also, all of the configuration resources of the component are loaded into memory. The core passes IL client callback functions to the component by means of the SetCallbacks method (1.2). If previous steps are
successful, a valid handle is returned in step 1.3 and the component will be in the
OMX_StateLoaded state.

The IL client shall configure the component and its ports. For this purpose, the IL core macro OMX_SetParameter shall be used; it may be called multiple times (step 1.4) if needed.

When the client has completed the configuration phase, it can request the component to make the state transition to OMX_StateIdle. Only after this request shall the IL client set up buffers for the component to use for all of its ports. The IL client shall use either OMX_AllocateBuffer or OMX_UseBuffer to set up buffers. If the IL client asks components for a tunnel, it does not allocate setup buffers because the tunneled components allocate any buffers. See section 3.4.1.2 for more details on tunneling.

This process may be repeated multiple times, depending on the number of ports and the total number of buffers needed on each port. If OMX_UseBuffer is used, the IL client shall have allocated a buffer and passed it to the component. Alternatively, the IL client may ask the component to allocate a buffer and a buffer header using the OMX_AllocateBuffer method. In the latter case, the component will allocate both a buffer and its related header and return it to the IL client by reference.

As soon as these initial configuration steps are completed, the component shall complete the state transition and return an event to the client for the SendCommand request completion (step 2.8).

The component is now ready to be used by the IL client.

####3.4.1.2  Tunneled Initialization
To avoid moving data buffers back and forth among the IL client and OpenMAX components, data tunnels can be set up so that the output buffer of one component is passed directly to the input port of the next component in the chain.

Consider the example shown in Figure 3-4, where an IL client generates data for a chain of three tunneled components identified as A, B, and C. Component C is a sink and does not return data to the IL client.

![](img/3_4.png)

**Figure 3-4. Example of Data Tunneling Among OpenMAX Components**

Note that all callbacks are always directed to and managed by the IL client when ports communicate using proprietary or tunneled communication. The tunneling setup and initialization require a detailed description, based on the following steps:

- The components are constructed with the calls to OMX_GetHandle.
- The components are tunneled, linking an output port of the first component to an input port of the second component. The port that shall supply the buffer is decided in this phase.
- The IL client may override the input ports’ choice of buffer supplier after OMX_SetupTunnel has completed by setting the buffer supplier into the input port, which in turn will reprogram the supplier to the output port..

During the transition from OMX_StateLoaded to OMX_StateIdle, each component shall not transition until the required buffers on all enabled ports have been allocated.

OMX_SetupTunnel shall be executed only when the components are in the OMX_StateLoaded state or when ports are disabled. Figure 3-5 illustrates the setup process:

![](img/3_5.png)

**Figure 3-5. Tunnel Setup**
The IL client shall start the data setup process by calling the OMX_SetupTunnel
function of the IL core when the components that are being tunneled are in the
OMX_StateLoaded state (step 1.0).

As a result, the IL core shall call the ComponentTunnelRequest methods of
component A and B in sequence. The structure OMX_TUNNELSETUPTYPE defined in
section 3.1.2.9 shall be passed by the IL core to the component with the output port first.
The component receiving such a call shall fill in the structure and return it to the core. If
the ComponentTunnelRequest call returns successfully, the IL core shall call the
same function on the second component (1.3), passing the OMX_TUNNELSETUPTYPE
structure that was filled in by the first component. The component also shall check that
the output port of the peer component is compatible with its input port (i.e., the data type
should be the same) (1.4). If the tunnel setup parameters included in the structure are
agreed to by the second component, the ComponentTunnelRequest call will send
back to the first component the result of negotiation (1.5) and returns successfully (1.6).
The IL core shall check that both calls of ComponentTunnelRequest did not return
errors. If so, the initial OMX_SetupTunnel will return successfully.

If the call to ComponentTunnelRequest on component B fails, component A will
be set to not tunnel by a second call to ComponentTunnelRequest with a pointer to
NULL in place of the component B handle and pTunnelSetup parameter.


After the successful tunnel setup, the IL client may override the buffer supplier
negotiation with the procedure illustrated in Figure 3-6:

![](img/3_6.png)

**Figure 3-6. IL Client Buffer Supplier Override**

If the IL client wants to override the negotiation of tunneled components that specifies
which component is the buffer supplier, it shall call the function SetParameter on the
component that provides the input port. That component is responsible for signaling to
the other tunneled component the new buffer supplier, with the same call to
SetParameter.

The last step of the tunnel initialization phase is the state transition from
OMX_StateLoaded to OMX_StateIdle that also involves the buffer allocation and
assignment. Figure 3-7 illustrates the state transition behavior in which the tunnels are
already created and configured.


![](img/3_7.png)

**Figure 3-7. Tunneling Example**

Component A is tunneled with component B, and component B is the buffer supplier.

Component B is tunneled with component C, and component C is the buffer supplier.

Figure 3-8 illustrates the behavior of each tunneled component during the state transition.

![](img/3_8.png)

**Figure 3-8. State Transition to Idle in the Case of Tunneled Component s**

Each supplier port on a component shall pass its buffers to the non-supplier port it is
tunneling with via OMX_UseBuffer. After all of its supplier ports have passed buffers,
the component waits until all of its non-supplier ports have received all of their buffers
via OMX_UseBuffer.

In Figure 3-8, component A receives the state transition request from the IL client.
Component A is tunneled with component B. The input port of B is set as buffer supplier
for the tunnel. In this case, component A shall wait until its output port receives all of the
needed buffers.

Meanwhile, the IL client asks component B to change its state. In this case, component B
has a port that is a buffer supplier, the input port, and it shall call UseBuffer on the
output port of component A. Then, component B waits for all of the needed buffers on its
output port.

Now component A has all of the needed buffers, so it can perform the state transition to
OMX_StateIdle. The exact sequence of transitions can be different, since it depends on
the platform, the operating system, and the implementation. The only rule is to wait until
all the resources are available.

The IL client requests that component C change its state. Component C behaves like
component B: Component C gives the buffers needed to component B, and then can
change its state, since it does not need any other buffers.

Finally, component B can change its state to OMX_StateIdle since it has obtained all of
the needed buffers.

###3.4.2 Data Flow
OpenMAX defines two means of data communication:

- Tunneled communication, where a port exchanges data directly with a port on another component
- Non-tunneled communication, where a port exchanges data only with the IL client

A port may implement data tunneling via proprietary communication, taking advantage of platform-specific features. The following sections describe the data flow inherent to each means of communication.

####3.4.2.1  Non-tunneled Data Flow
An IL client that has a data buffer to deliver to a component input port shall issue an OMX_EmptyThisBuffer call.

Conversely, for the component output port, the IL client shall initially provide one or
more empty buffers into which the component can write output data; the
OMX_FillThisBuffer call accomplishes this task. As soon as one buffer is available
from the component output port, the component shall send an OMX_FillBufferDone
callback. The component is aware of the callback entry point from the earlier SetBacks
call.

Note that the IL client is entirely responsible for moving data buffers among components
if data tunneling is not used.

Figure 3-9 illustrates the dynamic behavior related to data flow.

![](img/3_9.png)

**Figure 3-9. Data Flow Between Non-tunneled Components**

####3.4.2.2  Tunneled Data Flow
In data tunneling, OpenMAX components directly pass data buffers among themselves
without returning them to the IL client. This data flow uses a different convention from
the situation where all data buffers are exchanged with the IL client.

If the buffer supplier is the output component, it shall call OMX_EmptyThisBuffer on
the other tunneled component to pass the buffer that is to be emptied. When the input
component has terminated the operation, it shall return the buffer to the output
component by calling OMX_FillThisBuffer on it.

If the buffer supplier is the input component, the communication mechanism is the same
but is initiated by calling OMX_FillThisBuffer on the output component. Figure 3-
10 illustrates this process.

![](img/3_10.png)

**Figure 3-10. Data Flow Between Tunneled Components**

####3.4.2.3  Proprietary Communication
On some platforms data tunneling among components can be optimized by proprietary
communication mechanisms, which can be based on specific hardware such as DMA or
shared memory. Such resources are set up in a proprietary manner during the standard
data tunneling setup phase. Although the IL client uses the standard
OMX_SetupTunnel call, platform-specific optimizations can prepare optimized
transport channels among components.

Assuming a chain of components A, B, and C that support proprietary communication,
the resulting data flow would appear as illustrated in Figure 3-11.

![](img/3_11.png)

**Figure 3-11. Data Flow with Proprietary Communication Between Components**
Assuming that all components are in the OMX_StateExecuting state, the IL client sends
two buffers to component A using the OMX_EmptyThisBuffer call (steps 1.0 and
1.1). Given the data tunnel setup, the output of component A is sent to the input port of
component B. The output of component B is sent to the input port of component C, which
is the sink.

No callbacks will be invoked since the components will use their proprietary mechanisms
to move data.

The OMX_EmptyBufferDone callback will be issued to the IL client only when
component A has finished processing buffers.

Even though buffer-related callbacks are not used in this use case, note that components
may still generate events to the IL client using the OMX_EventHandler callback entry
point.

###3.4.3 De-Initialization
This section describes tunneled and non-tunneled component de-initialization.

####3.4.3.1  Non-tunneled De-initialization
When the IL client decides to stop the execution and dispose of the components, it should
first switch the components to the OMX_StateIdle state so that all buffers are returned to
their suppliers.

When the transition to OMX_StateIdle is completed, the IL client can request the
component to change its state to OMX_StateLoaded. The IL client shall free all of the
component’s buffers by calling OMX_FreeBuffer for each buffer. The
OMX_FreeBuffer function requires that the component remove the specified buffer
from the specified port. If the component allocated the buffer with an
OMX_AllocateBuffer call, the component shall also free the buffer memory. If the
IL client allocated the buffer and assigned it to the component with an OMX_UseBuffer call, then the IL client shall de-allocate the buffer memory after
calling OMX_FreeBuffer.

When all of the buffers have been freed, the component shall complete the state transition.
Finally, the IL client calls the OMX_FreeHandle function that disposes of the
component.

This procedure is performed for each non-tunneled port. Figure 3-12 illustrates non-
tunneled de-initialization.

![](img/3_12.png)

**Figure 3-12. De-initialization of Non-tunneled Components**

A port that is tunneled shall follow the component de-initialization procedure illustrated
in section 3.4.3.2.

####3.4.3.2  Tunneled De-Initialization
Figure 3-13 illustrates the component de-initialization for a port that is tunneled.

![](img/3_13.png)

**Figure 3-13. De-initialization of Tunneled Components**

###3.4.4 Port Disablement and Enablement
Disabling a port causes it to behave as if its component transitioned to the
OMX_StateLoaded state. Thus, all of the port’s buffers are returned to their suppliers,
and any buffers the disabled port allocated are freed. The act of enabling a port inverts
this process, putting a port that is effectively in the OMX_StateLoaded state into the
component’s state. Thus, if the component is in a state where its ports have buffers, then
an enabled port will acquire buffers. Likewise, if the component is exchanging buffers, an
enabled port will begin exchanging buffers.

Note that if a port is disabled when the component is in the OMX_StateLoaded state, the
port’s effective state is still made disjoint from the component’s state. Thus, when a
component transitions from OMX_StateLoaded to OMX_StateIdle, any disabled port will
not acquire buffers but, instead, will effectively remain in OMX_StateLoaded.
The description of port disablement and enablement is divided into tunneling and non-
tunneling cases.

####3.4.4.1  Tunneled Ports Disablement and Enablement
Figure 3-14 illustrates the behavior of enabling and disabling tunneled ports.

![](img/3_14.png)

**Figure 3-14. Disablement and Enablement of Tunneled Ports**

####3.4.4.2  Non-tunneled Port Disablement and Enablement
Figure 3-15 illustrates the case of the disablement and enablement procedure for a non-
tunneled port. A detailed discussion of OMX_AllocateBuffer, OMX_UseBuffer,
and OMX_FreeBuffer is omitted here; for more detailed descriptions of the use of
these functions, see sections 3.3.15, 3.3.14, and 3.3.16, respectively.

![](img/3_15.png)

Figure 3-15. Disablement and Enablement of Non-tunneled Ports

###3.4.5 Dynamic Port Reconfiguration
This section describes how a component may change its port settings dynamically.

The following examples show where this functionality is typically needed:

- A video decoder parses a sequence header and discovers the frame size of the output pictures, so buffers associated with its output ports shall be rearranged.
- The parameters of an audio stream vary dynamically, and a decoder should change its port settings.

Figure 3-16 shows how a video decoder and a video renderer, both of which exchange
data through the IL client, should dynamically change their port settings.

![](img/3_16.png)

**Figure 3-16. Dynamic Port Reconfiguration**

The sequence starts with the IL client putting a video renderer and a video decoder in the
OMX_StateExecuting state (1.0 through 1.3). At this stage, the output port of the video
decoder and the input port of the renderer are not yet configured, since the dimension of the output frame is unknown a priori. The decoder needs to start parsing the input bit
stream to derive such information.

In fact, the IL client sends the first buffer to the decoder in step 1.4. Assuming that the
video sequence header is included in that first buffer, the OpenMAX decoder component
will parse it and change its output port settings accordingly.
The OpenMAX decoder component shall then notify the IL client by generating the
OMX_PortSettingsChanged event (step 1.5). As soon as the IL client receives this
callback, it shall disable the output port of the video decoder and the input port of the
video renderer (steps 1.6 through 1.11).

The IL client shall then read the new port settings with OMX_GetConfig and allocate
one or more buffers with the right dimensions for the output port. Once the buffers are
allocated, they will be also communicated to the video renderer using OMX_UseBuffer
(1.17). The input port of the video renderer shall also be set up with OMX_SetConfig
(1.18).

Finally, ports can be enabled and normal processing resumes.

###3.4.6 Resource Management
This section describes the entry points for resource management. The interface between
components and the resource manager are presented only as an example. Only the
interface between the IL client and the components is part of the OpenMAX standard
definition. An IL client may use the resource manager entry points.

Figure 3-17 proposes the behavior of an IL client that ignores the resource manager. The
resource manager handles the component internally only, and the IL client has to take no
special action.

![](img/3_17.png)

**Figure 3-17. Transition from Loaded to Idle with Resource Management**

In Figure 3-17, the IL client is unaware of the existence of a resource manager. In the
implementation of the OpenMAX component, an asynchronous call to the resource
manager is implemented.


The OpenMAX component provides a callback to the resource manager, which receives
the signal for the completion of the request.

Figure 3-17 represents a possible implementation of a resource manager, and shows how
it can be transparent to the client. The functions AcquireResourceRequest and
AcquireResourceResponse are examples. This specification is concerned only
about the interface between the IL client and the components. Details of the interactions
between the components and the vendor/specific manager(s) are outside the scope of this
specification.

Figure 3-18 presents a more complex use case.


![](img/3_18.png)

**Figure 3-18. Busy Resource Management**

In Figure 3-18, two different OpenMAX components, A and B, need the same resource to
work, and they have different priorities. Here, as in the preceding example, the IL clients
use the standard transition from Loaded to Idle to set up the component and allocate all of
the required resources.

The first component, component A, takes ownership of the resource, requesting it from
the resource manager. Component A switches to the idle state and is ready to execute.

The second component, component B, asks for the same resource, but in this case the
resource manager denies it since a higher priority component, component A, has that
resource. This event is reported to the IL client with an error message including the value
OMX_ErrorInsufficientResources. If IL client Y decides that it needs to be
notified when this resource becomes available again, it may direct component B to
change state to OMX_StateWaitForResources. This action puts component B in a
waiting queue until the resource X will become available. Alternatively, IL client Y may
request component B to switch back to the Loaded state.

Figure 3-18 also shows the behavior of components when resource X becomes available.
Component A changes state to Loaded and releases all of the resources. The resource
manager becomes aware of the available resource and calls Component B, which is
already in the waiting queue.

When the resource manager provides the component with all the resources it is waiting
on, the component informs the IL client that all resources needed are available with an
OMX_EventResourcesAcquired event. The IL client shall now provide all of the
needed buffers to the component. Then, the component can change state by itself to
OMX_StateIdle and alert the client about the state change. This waiting queue represents
a unique case of automatic state change.

In Figure 3-18, the priorities of components A and B are not compared within the IL
layer, and no preemption mechanism is implemented or proposed; an external policy
manager, which should communicate with the resource manager, should have this
responsibility. The description of such a policy manager is outside the scope of this
document and the OpenMAX standard in general.

Figure 3-19 presents an example of a client that actively uses the resource management
API.

![](img/3_19.png)

**Figure 3-19. State Change from Loaded to WaitForResources**

The IL client may request a state change from OMX_StateLoaded to
OMX_StateWaitForResources in case the IL client wants to be notified when the
resource becomes available again. For an explanation of OMX_StateWaitForResources,
see section 3.1.1.2.5.

In this case, the client puts the component into a waiting queue, handled by the resource
manager; the change to the idle state happens effectively when the resource will become available or if it is available immediately. In any case, the client receives two different
OMX_EventHandler callbacks that correspond to two different state changes.

The two functions WaitForResourceRequest and
WaitForResourceResponse in Figure 3-19 are not defined in this specification but
are examples of an interaction between components and the resource manager.

The IL client may decide to stop waiting at a certain time. In this case, it shall request the
component to change state back to Loaded, as shown in Figure 3-20.

![](img/3_20.png)
Figure 3-20. Remove Component from Waiting Status		
