#3 OpenMAX Integration Layer Control API
The OpenMAX Integration Layer API allows integration layer clients to control multimedia components in the audio, video and image domains. An “other” domain is also included to provide for extra functionality, such as audio-video (A/V) synchronization. The user of the OpenMAX Integration Layer API is usually a multimedia framework. In the rest of this document, the user of the OpenMAX Integration Layer API will be referred to as the IL client.

The OpenMAX Integration Layer API is defined in a set of header files, namely:

-  OMX_Types.h: Data types used in the OpenMAX IL
-  OMX_Core.h: OpenMAX IL core API
-  OMX_Component.h: OpenMAX component API
-  OMX_Audio.h: OpenMAX audio domain data structures
-  OMX_IVCommon.h: OpenMAX structures common to image and video domains
-  OMX_Video.h: OpenMAX video domain data structures
-  OMX_Image.h: OpenMAX image domain data structures
-  OMX_Other.h: OpenMAX other domain data structures (includes A/V synchronization)
-  OMX_Index.h: Index of all OpenMAX-defined data structures
  
This section describes how the OpenMAX core and OpenMAX components are configured for operation.

First, the OpenMAX data types are introduced. Next, the methods of the OpenMAX core are described. The methods that components implement are discussed in section 3.3. Finally, section 3.4 shows calling sequences for a few meaningful operations, including component initialization, normal data flow, data tunnel setup, and data flow in the presence of data tunneling. Such sequence diagrams aim at describing the dynamic interactions between the IL client, the IL core, and the OpenMAX components.

When documenting functions, the following convention is used for function parameters:

-  <param_name> [in] specifies an input parameter, which is set by the function caller and read by the function implementation.
-  <param_name> [out] specifies an output parameter, which is set by the function implementation and passed back to the caller. When the function returns, the caller can read the new value of the parameter, which is passed as a reference.
-  <param_name> [inout] specifies an input/output parameter, which the function caller can set. The function implementation can modify the parameter before returning it back to the function caller.

This parameter classification can also be found in the OpenMAX header files, where the null macros OMX_IN, OMX_OUT and OMX_INOUT are defined. OMX_IN corresponds to the function parameter <param_name> [in]. OMX_OUT corresponds to the function parameter <param_name> [out], and OMX_INOUT corresponds to the function parameter <param_name> [inout].
 
##3.1  OpenMAX Types
###3.1.1 Enumerations
Five 32-bit integer enumerations are defined in OMX_Core.h:

- `OMX_ERRORTYPE` is returned by each function defined in the OpenMAX Integration Layer API (see section 3.1.1.3).
- `OMX_COMMANDTYPE` includes the possible commands that an IL client can send to an OpenMAX component (see section 3.1.1.1).
- `OMX_EVENTTYPE` includes events that can be generated inside an OpenMAX component and that are passed to the IL client through a callback function (see section 3.1.1.4).
- `OMX_BUFFERSUPPLIERTYPE` includes all the possibilities for the buffer supplier in the case of tunneled ports. A description of the use of this enumerative type can be found in section 3.1.1.5.
- `OMX_STATETYPE`, which is described in section 3.1.1.2.Figure 3-1 shows the enumerations defined in `OMX_Core.h`.

![](img/3_1.png)


**Figure 3-1. Enumerations Defined in OMX_Core.h**
####3.1.1.1  OMX_COMMANDTYPE
Table 3-1 represents the possible commands that an IL client can send to an OpenMAX component. Since commands are non-blocking, the OpenMAX component generates a command completion event via a callback function when the command has completed.
Callbacks are defined in a dedicated structure; see section 3.1.2.7.

| Field Name | Description |
| ------------- |
| OMX_CommandStateSet | Change the component state OMX_CommandFlush  Flush the queue(s) of buffers on a port of a component|
| OMX_CommandPortDisable | Disable a port on a component |
| OMX_CommandPortEnable | Enable a port on a component |
| OMX_CommandMarkBuffer | Mark a buffer and specify which other component will raise the event mark received|

Table 3-2 describes the parameters to be used for each command.

| Command code | nParam | pCmdData |
| ------------- |
| OMX_CommandStateSet |OMX_STATETYPE – state to transition to | NULL |
| OMX_CommandFlush | OMX_U32 – target port ID | NULL |
| OMX_CommandPortDisable |OMX_U32 – target port ID | NULL |
| OMX_CommandPortEnable | OMX_U32 – target port ID | NULL |
| OMX_CommandMarkBuffer | OMX_U32 – target port ID |OMX_MARKTYPE* - mark data and target component |

**Table 3-2. Command Syntax**
####3.1.1.2  OMX_STATETYPE
Figure 3-2 illustrates the transitions among states that occur as a consequence of the IL client calling `OMX_SendCommand`(`OMX_StateSet`, <state>), where the new state for the component is passed as a parameter. A transition name surrounded by curly braces indicates that the transition is not triggered by a command sent by the IL client but is a consequence of internal component events

![](img/3_2.png)

**Figure 3-2. OpenMAX Component State Transitions**

This section describes component states. An IL client commands a component to change states via the `OMX_SendCommand` function using the `OMX_CommandStateSet` command.

Table 3-3 represents the states of an OpenMAX component.

| Field Name | Description | Resources Allocated |Location of buffer |
| ------------- |:-------------:|
| OMX_StateInvalid | Component is corrupt or has encountered an error from which it cannot recover. | Unknown | Unknown |
| OMX_StateLoaded | Component has been loaded but has no resources allocated. | No | Not available |
| OMX_StateIdle | Component has all resources but has not transferred any buffers or begun processing data. | Yes | Supplier only |
| OMX_StateExecuting | Component is transferring buffers and is processing data (if data is available). | Yes | Supplier or non-supplier |
| OMX_StatePause | Component data processing has been paused but may be resumed from the point it was paused. | Yes |Supplier or non-supplier |
| OMX_StateWaitFor | Resources Component is waiting for aresource to become available.| No | Not available |

**Table 3-3. OpenMAX Component States**
######3.1.1.2.1  OMX_StateLoaded
A component is in the OMX_StateLoaded state after it has been created via an `OMX_GetHandle` call and before allocation of its resources. In this state, the IL client may modify the component’s parameters via `OMX_SetParameter`, set up data tunnels on the component’s ports with `OMX_SetupTunnel`, or transition the component to either the `OMX_StateIdle` state or the OMX_StateWaitForResources state.

The IL client may elect to transition a component that is currently in the OMX_StateLoaded state into the OMX_StateWaitForResources state if, for example, the component failed to acquire all of its resources on an attempted transition to the `OMX_StateIdle` state.

###### 3.1.1.2.1.1  OMX_StateLoaded to OMX_StateIdle
If the IL client requests a state transition from `OMX_StateLoaded` to `OMX_StateIdle`, the component must acquire all of its resources, including buffers, before completing the transition. Furthermore, before the transition can complete, the buffer supplier, which is always the IL client when not tunneling, must ensure that the non-supplier possesses all of its buffers. For a port connected to the IL client, the IL client may allocate the buffers itself and then pass them to the port via an `OMX_UseBuffer` call on the port, or it may direct the port to perform the allocation via an `OMX_AllocateBuffer` call on the port.

When a port is tunneling, the supplier port either allocates buffers itself or, if the port implements buffer sharing, re-uses buffers from a port on the same component. A tunneling supplier port then passes the buffers to the non-supplier port via an `OMX_UseBuffer` call on the non-supplier.

The number of buffers used on a port is specified in its port definition (see `OMX_IndexParamPortDefinition`), which defaults to the minimum (specified in the same structure) but which may be modified by the supplier before the sequence of `OMX_UseBuffer` and `OMX_AllocateBuffer` calls via a call to `OMX_SetParameter`.

#####3.1.1.2.2  OMX_StateIdle
In the `OMX_StateIdle` state, the component is ready to be used, meaning that all necessary resources have been properly allocated. However, the suppliers retain all their buffers, and no buffer exchange or processing is taking place. Thus, if this state is entered from an `OMX_StateExecuting` or `OMX_StatePause` state, the component shall have returned all buffers it was processing to their respective suppliers. The IL client may transition the component to any states other than the `OMX_StateInvalid` and `OMX_StateWaitForResources` states.

######3.1.1.2.2.1  OMX_StateIdle to OMX_StateLoaded
On a transition from OMX_StateIdle to OMX_StateLoaded, each buffer supplier must call `OMX_FreeBuffer` on the non-supplier port for each buffer residing at the non-supplier port. If the supplier allocated the buffer, it must free the buffer before calling `OMX_FreeBuffer`. If the non-supplier port allocated the buffer, it must free the buffer upon receipt of an `OMX_FreeBuffer` call. Furthermore, a non-supplier port must always free the buffer header upon receipt of an OMX_FreeBuffer call. When all of the buffers have been removed from the component, the state transition is complete; the component communicates that the initiating `OMX_SendCommand` call has completed via a callback event.

######3.1.1.2.2.2  OMX_StateIdle to OMX_StateExecuting
If the IL client requests a state transition from OMX_StateIdle to `OMX_StateExecuting`, the component shall begin transferring and processing data. For ports that communicate with the IL client, the IL client will initiate buffer transfers via `OMX_EmptyThisBuffer` and `OMX_FillThisBuffer`. Among tunneling ports, any input port that is also a supplier shall transfer its empty buffers to the tunneled output port via `OMX_FillThisBuffer`.

#####3.1.1.2.3  OMX_StateExecuting
In this state, an OpenMAX component is transferring and processing data buffers. The component shall accept calls to `OMX_EmptyThisBuffer` on its input ports and `OMX_FillThisBuffer` on its output ports. Any port that communicates with the IL client shall call the `EmptyBufferDone` and `FillBufferDone` callbacks to return an empty or full buffer, respectively, back to the IL client. Any tunneling port shall call `OMX_FillThisBuffer` or `OMX_EmptyThisBuffer` on its corresponding tunneled port to return an empty or full buffer, respectively, back to its tunneled port. An IL client may transition a component in the `OMX_StateExecuting` state to either the `OMX_StateIdle` state or the `OMX_StatePaused` state.

######3.1.1.2.3.1  OMX_StateExecuting to OMX_StateIdle
If the IL client requests a state transition from `OMX_StateExecuting` to `OMX_StateIdle`,the component shall return all buffers to their respective suppliers and receive all buffers belonging to its supplier ports before completing the transition. Any port communicating with the IL client shall return any buffers it is holding via `OMX_EmptyBufferDone`
and OMX_FillBufferDone callbacks, which are used by input and output ports, respectively. Any non-supplier port shall return all buffers it is holding to the input port or output port it is tunneling with using `OMX_EmptyThisBuffer` or `OMX_FillThisBuffer`, respectively. Likewise, any supplier tunneling port shall wait for all of its buffers to be returned from its tunneled port.

#####3.1.1.2.4  OMX_StatePause
In this state, an OpenMAX component is not transferring or processing data but buffers are not necessarily returned to their suppliers. From the `OMX_StatePause` state, execution may be resumed via a transition to `OMX_StateExecuting`, preferably without dropping data. The component may still accept data buffers at its input, but such buffers will be queued only and not processed further. The IL client may transition a component in the `OMX_StatePause` state to `OMX_StateIdle` or `OMX_StateExecuting`. On a transition from `OMX_StatePause` to `OMX_StateIdle`, the component shall return all buffers to their respective suppliers in a manner identical to the `OMX_StateExecuting` to `OMX_StateIdle` transition described in section 3.1.1.2.3.1.

#####3.1.1.2.5  OMX_StateWaitForResources
In this state, the component is waiting for one or more of its required resources to become available. This state is related to resource management. The assumption is that one or more hardware-specific resource managers exist on the platform to handle available resources. The interaction among OpenMAX components and resource managers is outside the scope of this specification.

If a component in the `OMX_StateLoaded` state fails to enter the `OMX_StateIdle` state because resources other than buffers are insufficient, the IL client may put the component in the `OMX_StateWaitForResources` state if the IL client wants to be notified when the needed resources become available. The IL client may command the component to discontinue waiting for resources by transitioning it from the `OMX_StateWaitForResources` state to the `OMX_StateLoaded` state. If a component in the `OMX_StateWaitForResources` state acquires all the resources upon which it is waiting, it shall initiate a transition to the `OMX_StateIdle` state.

######3.1.1.2.5.1  OMX_StateWaitForResources to OMX_StateIdle
When a component initiates a transition from the OMX_StateWaitForResources state to the `OMX_StateIdle` state, it shall communicate the initiation of this transition to the IL client via an `OMX_EventResourcesAcquired` event. When the IL client receives the `OMX_EventResourcesAcquired` event, it shall call `OMX_UseBuffer` and `OMX_AllocateBuffer` in the manner of a transition from `OMX_StateLoaded` to `OMX_StateIdle`. Likewise, the component cannot complete its transition to
`OMX_StateIdle` until it acquires all of its resources, including buffers.

#####3.1.1.2.6  OMX_StateInvalid
In this state, the component has suffered internal corruption or an error from which it cannot recover. When it detects such a condition, the component transitions itself to `OMX_StateInvalid` and informs the IL client by generating an `OMX_ErrorEvent` event with the value `OMX_ErrorInvalidState`. When the IL client receives `OMX_EventError` indicating a transition to `OMX_StateInvalid`, it shall free all resources associated with that component and eventually call `OMX_FreeHandle` to release the handle associated with the component.

A component in the `OMX_StateInvalid` state shall fail every call made upon it and return an `OMX_ErrorStateInvalid` error message except for `OMX_GetState`, `OMX_FreeBuffer`, or `OMX_ComponentDeinit`. The IL client may also command a transition to the `OMX_StateInvalid` state explicitly via `OMX_SendCommand`. A component may transition between any state and the `OMX_StateInvalid` state.

####3.1.1.3  OMX_ERRORTYPE
The OMX_ERRORTYPE enumeration shown in Table 3-4 defines the standard
OpenMAX errors that all functions defined in the OpenMAX IL API return. These errors
should cover most of the common failure cases. However, vendors are free to add
additional error messages of their own as long as they follow these rules:

-  Vendor error messages shall be in the range of 0x90000000 to 0x9000FFFF.
-  Vendor error messages shall be defined in a header file provided with the component. No error messages are allowed that are not defined.

|Field Name| Value | Description |
|------------- |:-------------:|
|OMX_ErrorNone| 0 | The function returned successfully. |
|OMX_ErrorInsufficientResources | 0x80001000 |There were insufficient resources toperform the requested operation. |
|OMX_ErrorUndefined | 0x80001001 | There was an error but the cause of the error could not be determined. |
|OMX_ErrorInvalidComponentName |0x80001002 | The component name string wasinvalid. |
|OMX_ErrorComponentNotFound | 0x80001003 | No component with the specified name string was found. |
|OMX_ErrorInvalidComponent | 0x80001004 | The component specified did not have a OMX_ComponentInit entry point, or the component did not correctly complete the OMX_ComponentInit call. |
|OMX_ErrorBadParameter | 0x80001005 | One or more parameters were invalid.|
|OMX_ErrorNotImplemented | 0x80001006 | The requested function is notimplemented. |
|OMX_ErrorUnderflow | 0x80001007 | The buffer was emptied before the next buffer was ready. |
|OMX_ErrorOverflow | 0x80001008 | The buffer was not available when it was needed.|
|OMX_ErrorHardware | 0x80001009 | The hardware failed to respond as expected. |
|OMX_ErrorInvalidState | 0x8000100A | The component is in the OMX_StateInvalid state. |
|OMX_ErrorStreamCorrupt | 0x8000100B |The stream is found to be corrupt. |
|OMX_ErrorPortsNotCompatible | 0x8000100C | Ports being set up for tunneled communication are incompatible. |
|OMX_ErrorResourcesLost | 0x8000100D | Resources allocated to a component inthe OMX_StateIdle state have been lost, which has resulted in the component returning to the OMX_StateLoaded state. |
|OMX_ErrorNoMore | 0x8000100E | No more indices can be enumerated. |
|OMX_ErrorVersionMismatch | 0x8000100F |The component detected a versionmismatch. |
|OMX_ErrorNotReady | 0x80001010 |The component is not ready to returndata at this time. |
|OMX_ErrorTimeout | 0x80001011 | A timeout occurred. |
|OMX_ErrorSameState | 0x80001012 |The component tried to transition into the state that it is currently in. |
|OMX_ErrorResourcesPreempted | 0x80001013 |Resources allocated to a component in the OMX_StateExecuting or OMX_Pause states have been pre- empted, causing the component to return to the OMX_StateIdle state. |
|OMX_ErrorPortUnresponsiveDuringAllocation |0x80001014|The non-supplier port deemed that it had waited an unusually long time for the supplier port to send it an allocated buffer via an OMX_UseBuffer call. A non-supplier port sends this error to the IL client via the EventHandler callback during the allocation of buffers on a transition from the LOADED to the IDLE state or on a port enable.|
|OMX_ErrorPortUnresponsiveDuringDeallocation|0x80001015|The non-supplier port deemed that it had waited an unusually long time for the supplier port to request the de-allocation of a buffer header via a OMX_FreeBuffer call. A non-supplier port sends this error to the IL client via the EventHandler callback during the de-allocation of buffers on a transition from the IDLE to LOADED state or on a port disablement.|
|OMX_ErrorPortUnresponsiveDuringStop |0x80001016|The supplier port deemed that it had waited an unusually long time for the non-supplier port to return a buffer via an EmptyThisBuffer or FillThisBuffer call. A supplier port sent this error to the IL client via the EventHandler callback during the disabling of a port, either on a transition from the IDLE to LOADED state or on a port disablement.|
|OMX_ErrorIncorrectStateTransition|0x80001017|A state transition was attempted that is not allowed.|
|OMX_ErrorIncorrectStateOperation|0x80001018|A command or method was attempted that is not allowed during the present state. |
|OMX_ErrorUnsupportedSetting| 0x80001019|One or more values encapsulated in the parameter or configuration structure are unsupported.|
|OMX_ErrorUnsupportedIndex|  0x8000101A|The parameter or configuration indicated by the given index is unsupported.|
|OMX_ErrorBadPortIndex|  0x8000101B|The port index that was supplied is incorrect.|
|OMX_ErrorPortUnpopulated | 0x8000101C | The port has lost one or more of its buffers and is thus unpopulated.|

**Table 3-4. OpenMAX Error Codes**
####3.1.1.4  OMX_EVENTTYPE
The OMX_EVENTTYPE enumeration shown in Table 3-5 includes the event types that
an OpenMAX component can generate. Section 3.1.2.7 describes events that the
OpenMAX component generates and passes to the IL client by means of the callback
mechanism. Events have associated parameters that are also passed in the callback.

| Field Name | Description |
| ------------- |
| OMX_EventCmdComplete | Component has completed the execution of a command. |
| OMX_EventError | Component has detected an error condition. |
| OMX_EventMark | A buffer mark has reached the target component, and the IL client has received this event with the private data pointer of the mark. |
| OMX_EventPortSettingsChanged | Component has changed port settings. For example, the component has changed port settings resulting from bit stream parsing. |
| OMX_EventBufferFlag | The event that a component sends when it detects the end of a stream. |
| OMX_EventResourcesAcquired | The component has been granted resources and is transitioning from the OMX_StateWaitForResources state to the OMX_StateIdle state. |
**Table 3-5. OpenMAX Event Types**

#####3.1.1.4.1  OMX_EventCmdComplete
A component generates an OMX_EventCmdComplete event as soon as a command sent by the IL client has completed its execution. In case of a component state change, the new state that the component has entered is returned as an event parameter. A component that transitions to the OMX_StateInvalid state does not generate this event.

#####3.1.1.4.2  OMX_EventError
A component generates the OMX_EventError event when the component detects an error condition; the type of error detected is returned as an event parameter and will use values defined in `OMX_ERRORTYPE`. A component shall send the following errors via `OMX_EventError`:

-  A component sends the `OMX_ErrorInvalidState` error if the component transitions to the `OMX_StateInvalid` state.
-  A component sends the `OMX_ErrorResourcesPreempted` error if the component transitions from `OMX_StateExecuting` or `OMX_StatePause` to `OMX_StateIdle` due to the loss of a resource.
-  A component sends the `OMX_ErrorResourcesLost` error if the component transitions from `OMX_StateIdle` to `OMX_StateLoaded` due to the loss of a resource.]

#####3.1.1.4.3  OMX_EventMark
A component generates the `OMX_EventMark` event when it receives a marked buffer. When a component receives a buffer, it shall compare its own pointer to the pMarkTargetComponent field contained in the buffer. If the pointers match, then the
component shall send a mark event including pMarkData as a parameter, immediately after the component has finished processing the buffer. The IL client can use the mark event to measure the propagation delay of a data buffer through a chain of components, or to notify a component that a particular buffer has reached the given destination.

#####3.1.1.4.4  OMX_EventPortSettingsChanged
A component generates the `OMX_EventPortSettingsChanged` event as soon as component port settings change. For example, a video decoder may not know a priori the output frame size and frame rate, as these parameters are coded in the input bit stream. As soon as such parameters are parsed, the component changes the values of the configuration structures of its output port and sends the `OMX_EventPortSettingsChanged` event to the IL client.

#####3.1.1.4.5  OMX_EventBufferFlag
A component generates the `OMX_EventBufferFlag` event when an output port emits a buffer with the `OMX_BUFFERFLAG_EOS` flag set in the `nFlags` field. The `nData1` field of EventHandler specifies the value of the output port’s portindex field. The nData2 field of `EventHandler` specifies the unaltered nFlags field containing the end-of-stream (EOS) flag. If a component does not propagate a stream further (e.g., the component is an audio or video sink), then the component shall send an OMX_EventBufferFlag event for that stream when it has finished processing a buffer with OMX_BUFFERFLAG_EOS set. The nData1 field of EventHandler specifies the input port that received the buffer. The nData2 field of EventHandler specifies the unaltered nFlags field containing the EOS flag.

#####3.1.1.4.6  OMX_EventResourcesAcquired
A component generates the `OMX_EventResourcesAcquired` event when it is in the `OMX_StateWaitForResources` state, and the resource manager detects that the needed resources are available. When the component receives this event, it is ready to change state into the `OMX_StateIdle`, and it waits for all the buffers to be allocated and assigned to its ports.

####3.1.1.5  OMX_BUFFERSUPPLIERTYPE
The OMX_BUFFERSUPPLIERTYPE enumerative type shown in Table 3-6 specifies the port in the tunnel that is the supplier port. A buffer supplier port either may allocate its buffers or reuse buffers provided by another port within the same component.

| Field Name | Value | Description |
| ------------- |
| OMX_BufferSupplyUnspecified | 0x0 | The port supplying the buffers is unspecified, or no supplier is preferred. |
| OMX_BufferSupplyInput | | The input port supplies the buffers. |
| OMX_BufferSupplyOutput | | The output port supplies the buffer.|
**Table 3-6. OpenMAX Buffer Supplier Type Used in Tunnel Setup**

###3.1.2 Structures
This section discusses the data structures defined in the OpenMAX core. The first two fields of each OpenMAX data structure denote the size of the structure and the version of type `OMX_VERSIONTYPE`, which is defined in section 3.1.2.4. The entity that allocates an OpenMAX structure is responsible for filling in these two values.

####3.1.2.1  OMX_COMPONENTREGISTERTYPE
The `OMX_COMPONENTREGISTERTYPE` structure is used in the case of static linking of components to the core. The core optionally uses it to load the component and run the specific component initialization functions.

`OMX_COMPONENTREGISTERTYPE` is defined as follows.

``` C
typedef struct OMX_COMPONENTREGISTERTYPE
{
const char * pName;
OMX_COMPONENTINITTYPE pInitialize;
} OMX_COMPONENTREGISTERTYPE;
```

####3.1.2.2  OMX_COMPONENTINITTYPE Type Definition
The `OMX_COMPONENTINITTYPE` type definition is the type of function pointer for the
component initialization entry point. The definition is as follows:

```C
typedef OMX_ERRORTYPE (* OMX_COMPONENTINITTYPE)(OMX_IN OMX_HANDLETYPE hComponent);
```
#####3.1.2.2.1  pName
`pName` contains the string name of the component and has limit of 128 bytes (including‘\0’).

#####3.1.2.2.2  pInitialize
`pInitialize` contains the pointer to the initialization function of the component.

####3.1.2.3  OMX_ComponentRegistered[]
Any core that statically links its components shall define this global array containing the list of all registered components in the form of `OMX_COMPONENTREGISTERTYPE` fields.

####3.1.2.4  OMX_VERSIONTYPE
The `OMX_VERSIONTYPE` type indicates the version of a component or structure. Each structure uses an `OMX_VERSIONTYPE` field to indicate the OpenMAX specification version under which the structure is defined. For OpenMAX IL version 1.0, the specification version is 1.0.0.0. The component structure also includes an `OMX_VERSIONTYPE` field to indicate a vendor-specific component version.

``` C
OMX_VERSIONTYPE is defined as follows.
typedef union OMX_VERSIONTYPE
{
struct
{
OMX_U8 nVersionMajor;
OMX_U8 nVersionMinor;
OMX_U8 nRevision;
OMX_U8 nStep;
} s;
OMX_U32 nVersion;
} OMX_VERSIONTYPE;
```

#####3.1.2.4.1  nVersionMajor
`nVersionMajor` identifies the major version number.

#####3.1.2.4.2  nVersionMinor
`nVersionMinor` identifies the minor version number.

#####3.1.2.4.3  nRevision
`nRevision` identifies the revision number.

#####3.1.2.4.4  nStep
`nStep` identifies the step number.]

####3.1.2.5  OMX_PRIORITYMGMTTYPE
The `OMX_PRIORITYMGMTTYPE` type describes the priority assigned to a set of components. A component group identifies a set of co-dependent components associated with the same feature. All components in the same group share the same group ID and
priority. If one component in a group loses resources and stops running, the entire feature they collectively contribute to is lost. In this case, all of the other components in the same group shall transition to `OMX_StateLoaded`. A component that is the only one with a certain `nGroupID` acts atomically.

`OMX_PRIORITYMGMTTYPE` is defined as follows.

``` C
typedef struct OMX_PRIORITYMGMTTYPE {
OMX_U32 nSize;
OMX_VERSIONTYPE nVersion;
OMX_U32 nGroupPriority;
OMX_U32 nGroupID;
} OMX_PRIORITYMGMTTYPE;
```

#####3.1.2.5.1  nGroupPriority
The value of nGroupPriority is the priority value associated with a group of components. If a parameter of this type is assigned to a component, that component belongs to the group identified with nGroupID and has a priority equal to
nGroupPriority. By definition, the value 0 represents the highest priority for a group of components.

The exact mechanism to assign priorities to groups of components is outside the scope of this document.

#####3.1.2.5.2  nGroupID
The value for nGroupID is a unique ID for all components in the same component group.

####3.1.2.6  OMX_BUFFERHEADERTYPE
In the context of a single port, each data buffer has a header associated with it that contains meta-information about the buffer. The IL client shares buffer headers with each port with which it is communicating. Likewise, each pair of tunneling ports share buffer headers; otherwise, the same buffer transferred over multiple ports will have distinct buffer headers associated with it for each port. The definition of the buffer header is shown as follows.

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
pBuffer is a pointer to the actual buffer where data is stored but not necessarily the start of valid data; for more information, see the description of nOffset in section 3.1.2.6.4.

#####3.1.2.6.2  nAllocLen
nAllocLen is the total size of the allocated buffer in bytes, including valid and unused byte.

#####3.1.2.6.3  nFilledLen
nFilledLen is the total size of valid bytes currently in the buffer starting from the location specified by pBuffer and nOffset.

#####3.1.2.6.4  nOffset
nOffset is the start offset of valid data in bytes from the start of the buffer. A pointer to the valid data may be obtained by adding nOffset to pBuffer.

#####3.1.2.6.5  pAppPrivate
pAppPrivate is a pointer to an IL client private structure.

#####3.1.2.6.6  pPlatformPrivate
pPlatformPrivate is a pointer to a platform private structure. The core that allocated this buffer header structure uses this pointer.

#####3.1.2.6.7  pOutputPortPrivate
pOutputPortPrivate is a private pointer of the output port that uses the buffer. If a buffer header is used on an input port communicating with the IL client, the value of the buffer’s pOutputPortPrivate is undefined.

#####3.1.2.6.8  pInputPortPrivate
pInputPortPrivate is a private pointer of the input port that uses the buffer. If a buffer header is used on an output port communicating with the IL client, the value of the buffer’s pInputPortPrivate is undefined.

#####3.1.2.6.9  hMarkTargetComponent
hMarkTargetComponent is the handle of the component that should emit an `OMX_EventMark` event upon processing this buffer. A NULL handle indicates that the buffer carries no mark. The `OMX_CommandMarkBuffer` command provides this handle to the marking component. The marking component, in turn, copies this handle to the marked buffer. Each component that is processing a buffer should compare its own handle to this handle and emit the mark if the handles match. A component should
propagate this field from an input buffer to its associated output buffer.

#####3.1.2.6.10  pMarkData
The pMarkData pointer refers to IL client-specific data associated with the mark that is sent on `OMX_EventMark` when emitted. Upon receipt of a mark, the IL client may use this data to disambiguate this mark from others. The `OMX_CommandMarkBuffer` command provides this pointer to the marking component. The marking component, in turn, copies this pointer to the marked buffer. A component should propagate this field from an input buffer to its associated output buffer.

#####3.1.2.6.11  nTickCount
nTickCount is an optional entry that the component and IL client can update with a tick count when they access the component; not all components will update it. The value of nTickCount is in microseconds. Since this is a value relative to an arbitrary starting point, nTickCount cannot be used to determine absolute time.

#####3.1.2.6.12  nTimeStamp
nTimeStamp is a timestamp corresponding to the sample starting at the first logical sample boundary in the buffer. Timestamps of successive samples within the buffer may be inferred by adding the duration of the preceding buffer to the timestamp of the preceding buffer. A component should propagate this field from an input buffer to its associated output buffer.

#####3.1.2.6.13  nFlags
The nFlags field contains buffer specific flags, such as the EOS flag. A component should propagate this field from an input buffer to its associated output buffer. The list of flags is as follows:

```C
#define OMX_BUFFERFLAG_EOS 0x00000001
#define OMX_BUFFERFLAG_STARTTIME 0x00000002
#define OMX_BUFFERFLAG_DECODEONLY 0x00000004
#define OMX_BUFFERFLAG_DATACORRUPT 0x00000008
#define OMX_BUFFERFLAG_ENDOFFRAME 0x00000010
```

######3.1.2.6.13.1  OMX_BUFFERFLAG_EOS
A component sets EOS when it has no more data to emit on a particular output port. Thus, an output port shall set EOS on the last buffer it emits. The determination by a component of when an output port should cease sending data is implementation specific.

######3.1.2.6.13.2  OMX_BUFFERFLAG_STARTTIME
The source of a stream (e.g., a de-multiplexing component) sets the `OMX_BUFFERFLAG_STARTTIME` flag on the buffer that contains the starting timestamp for the stream. The starting timestamp corresponds to the first data that should be displayed at startup or after a seek operation. 

The first timestamp of the stream is not necessarily the start time. For instance, in the case of a seek to a particular video frame, the target frame may be an interframe. Thus the first buffer of the stream will be the intraframe preceding the target frame, and the start time will occur with the target frame along with any other required frames required to reconstruct the target intervening.

The `OMX_BUFFERFLAG_STARTTIME` flag is directly associated with the buffer timestamp. Thus, the association of the OMX_BUFFERFLAG_STARTTIME flag to buffer data and its propagation is identical to that of the timestamp.


A clock component client that receives a buffer with the STARTTIME flag shall perform an OMX_SetConfig call on its sync port using `OMX_ConfigTimeClientStartTime` and pass the timestamp for the buffer.

######3.1.2.6.13.3  OMX_BUFFERFLAG_DECODEONLY
The source of a stream (e.g., a de-multiplexing component) sets the `OMX_BUFFERFLAG_DECODEONLY` flag on any buffer that should be decoded but not rendered. This flag is used, for instance, when a source seeks to a target interframe that requires decoding of frames preceding the target to facilitate reconstruction of the target. In this case, the source would emit the frames preceding the target downstream but mark them as decode only.

The `OMX_BUFFERFLAG_DECODEONLY` flag is associated with buffer data and propagated in a manner identical to that of the buffer timestamp. A component that renders data should ignore all buffers with the `OMX_BUFFERFLAG_DECODEONLY` flag set.

######3.1.2.6.13.4  OMX_BUFFERFLAG_DATACORRUPT
The `OMX_BUFFERFLAG_DATACORRUPT` flag is set when the IL client identifies the data in the associated buffer as corrupt.

######3.1.2.6.13.5  OMX_BUFFERFLAG_ENDOFFRAME
`OMX_BUFFERFLAG_ENDOFFRAME` is an optional flag that is set by an output port when the last byte that a buffer payload contains is an end-of-frame. Any component that implements setting the `OMX_BUFFERFLAG_ENDOFFRAME` flag on an output port
shall set this flag for every buffer sent from the output port containing an end-of-frame.No buffer payload can contain data from two separate frames.

These restrictions enable input ports that receive data from the output port to detect an end-of-frame without requiring additional processing. These restrictions also enable an input port to easily detect if an output port supports this flag by its presence or absence on completion of the first frame.

####3.1.2.6.14  nOutputPortIndex
nOutputPortIndex contains the port index of the output port that uses the buffer. If a buffer header is used on an input port that is communicating with the IL client, the value of nOutputPortIndex is undefined.

#####3.1.2.6.15  nInputPortIndex
nInputPortIndex contains the port index of the input port that uses the buffer. If a buffer header is used on an input port that is communicating with the IL client, the value of nInputPortIndex is undefined.

####3.1.2.7  OMX_PORT_PARAM_TYPE
A component uses the OMX_PORT_PARAM_TYPE structure to identify the number and starting index of ports of a particular domain.

`OMX_PORT_PARAM_TYPE` is defined as follows.

``` C
typedef struct OMX_PORT_PARAM_TYPE {
OMX_U32 nSize;
OMX_VERSIONTYPE nVersion;
OMX_U32 nPorts;
OMX_U32 nStartPortNumber;
} OMX_PORT_PARAM_TYPE;
```

#####3.1.2.7.1  nPorts
nPorts is the number of ports of a given port domain (audio, video, image, or other) for the component.

#####3.1.2.7.2  nStartPortNumber
nStartPortNumber is the index of the first port of a given port domain (audio, video, image, or other) for the component . Subsequent ports of the given domain are numbered sequentially from nStartNumber.

#####3.1.2.8  OMX_CALLBACKTYPE
The OpenMAX IL includes a callback mechanism that allows a component to communicate the following with the IL client:

-  An asynchronous command triggered by the IL client has completed successfully or failed and generated an error. Commands include those sent by OMX_SendCommand and those implied by IL client calls to EmptyThisBuffer or FillThisBuffer.
-  An error unassociated with a command triggered by the IL client has occurred. For example, the component has suffered an unrecoverable error and is transitioning to the `OMX_StateInvalid` state.

To accomplish a callback, the OpenMAX IL has three callback functions defined: a generic event handler and two callbacks related to the dataflow (`EmptyBufferDone` and `FillBufferDone`).

The IL client is responsible for filling in an `OMX_CALLBACKTYPE` structure with itscallback entry points and passing the structure to the OpenMAX core at initialization(init) time, usually in the `OMX_GetHandle` function.

`OMX_CALLBACKTYPE` is defined as follows.

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
A component uses the EventHandler method to notify the IL client when an event of interest occurs within the component. The OMX_EVENTTYPE enumeration defines the set of OpenMAX IL events; refer to the definition of this enumeration for the meaning of each event. nData1 carries the value of `OMX_COMMANDTYPE` that has been completed or `OMX_ERRORTYPE`. nData2 carries further event parameters, e.g., `OMX_STATETYPE`. pEventData contains event specific data. The pEventData pointer may contain additional data associated with the event (e.g., mark-specific data). A call to EventHandler is a blocking call, so the IL client should respond within five msec to avoid blocking the component for an excessively long time period.

The `EventHandler` method is defined as follows.
``` C
OMX_ERRORTYPE(* OMX_CALLBACKTYPE::EventHandler)(
OMX_IN OMX_HANDLETYPE hComponent,
OMX_IN OMX_PTR pAppData,
OMX_IN OMX_EVENTTYPE eEvent,
OMX_IN OMX_U32 nData1,
OMX_IN OMX_U32 nData2,
OMX_IN OMX_PTR pEventData)
```
The parameters are as follows.

| Parameter | Description |
| -------- |
| hComponent | The handle of the component that calls this function. |
| eEvent | The event that the component is communicating to the IL client. |
| nData1 | The first integer event-specific parameter. See Table 3-7 for the meaning in the context of each event. |
| nData2 | The second integer event-specific parameter. See Table 3-7 for the meaning in the context of each event. The default value is 0 if not used. |
| pEventData | A pointer to additional event-specific data. See Table 3-7 for the meaning in the context of each event. |
**Table 3-7 lists the parameters used in each event.**

| eEvent | nData1 | nData2 | pEventData |
| ------- |
| OMX_EventCmdComplete | OMX_CommandStateSet | State reached | Null |
| | OMX_CommandFlush | Portindex | Null |
| | OMX_CommandPortDisable | Portindex | Null |
| | OMX_CommandPortEnable | Portindex | Null |
| | OMX_CommandMarkBuffer | Portindex | Null |
| OMX_EventError | Error code | 0 | Null |
| OMX_EventMark | 0 | 0 | Data linked to the mark, if any |
| OMX_EventPortSettings | Changed | port index | 0 | Null |
| OMX_EventBufferFlag | port index |nFlags unaltered | Null |
| OMX_EventResourcesAcquired | 0 | 0 | Null |
**Table 3-7. Event Parameter Usage**

#####3.1.2.8.2  EmptyBufferDone
A component uses the EmptyBufferDone callback to pass a buffer from an input port back to the IL client. A component sets the nOffset and nFilledLength values of the buffer header to reflect the portion of the buffer it consumed; for example,
nFilledLength is set equal to 0x0 if completely consumed.

In addition to facilitating normal data flow between an executing component and the ILclient, a component uses the EmptyBufferDone function to return input buffers to the IL client in the following cases:

- The IL client commands a transition from `OMX_StateExecuting` or `OMX_StatePause` to `OMX_StateIdle` or to `OMX_StateInvalid`.
- The IL client flushes or disables a port.
 
The `EmptyBufferDone` call is a blocking call that should return from within five msec.Therefore, the IL client may elect not to fill the buffers during this call but queue them for processing outside this call.

The `EmptyBufferDone` call is defined as follows.

``` C
OMX_ERRORTYPE(* OMX_CALLBACKTYPE::EmptyBufferDone)(
OMX_OUT OMX_HANDLETYPE hComponent,
OMX_OUT OMX_PTR pAppData,
OMX_OUT OMX_BUFFERHEADERTYPE* pBuffer)
```

The parameters are as follows.

| Parameter | Description |
| ------- |
| hComponent | The handle of the component that is calling this function. |
| pAppData | A pointer to IL client-defined data. |
| pBuffer | A pointer to an OMX_BUFFERHEADERTYPE structure that was consumed or returned. |

#####3.1.2.8.3  FillBufferDone
A component uses the FillBufferDone callback to pass a buffer from an output port back to the IL client. A component sets the nOffset and nFilledLength of the buffer header to reflect the portion of the buffer it filled; for example, nFilledLength is equal to 0x0 if it contains no data).

In addition to facilitating normal dataflow between an executing component and the IL client, a component uses this function to return output buffers to the IL client in the following cases:

- The IL client commands a transition from `OMX_StateExecuting` or `OMX_StatePause` to `OMX_StateIdle` or to `OMX_StateInvalid`.
- The IL client flushes or disables a port.

The `FillBufferDone` call is a blocking call that should return from within five msec. The IL client may elect not to empty the buffers during this call but queue them for consumption outside this call.

`FillBufferDone` is defined as follows.

``` C
OMX_ERRORTYPE(* OMX_CALLBACKTYPE::FillBufferDone)(
OMX_OUT OMX_HANDLETYPE hComponent,
OMX_OUT OMX_PTR pAppData,
OMX_OUT OMX_BUFFERHEADERTYPE* pBuffer)
```

The parameters are as follows.

| Parameter | Description |
| ------- |
| hComponent | The handle of the component to access. This handle is the component handle returned by the call to the GetHandle function. |
| pAppData | A pointer to IL client-defined data |
| pBuffer | A pointer to an OMX_BUFFERHEADERTYPE structure that was filled or returned. |

####3.1.2.9  OMX_PARAM_BUFFERSUPPLIERTYPE
The `OMX_PARAM_BUFFERSUPPLIERTYPE` structure is used to communicate buffer supplier settings or buffer supplier preferences.

`OMX_PARAM_BUFFERSUPPLIERTYPE` is defined as follows.

``` C
typedef struct OMX_PARAM_BUFFERSUPPLIERTYPE {
OMX_U32 nSize;
OMX_VERSIONTYPE nVersion;
OMX_U32 nPortIndex;
OMX_BUFFERSUPPLIERTYPE eBufferSupplier;
} OMX_PARAM_BUFFERSUPPLIERTYPE;
```

#####3.1.2.9.1  nPortIndex
`nPortIndex` represents the port that this structure applies to.

#####3.1.2.9.2  eBufferSupplier
`eBufferSupplier` is a field that contains the index of the buffer supplier, if input or
output.

####3.1.2.10  OMX_TUNNELSETUPTYPE
The ComponentTunnelRequest function uses the `OMX_TUNNELSETUPTYPE` structure to pass data between two ports when an IL client connects these ports via an `OMX_SetupTunnel` call.

`OMX_TUNNELSETUPTYPE` is defined as follows.

``` C
typedef struct OMX_TUNNELSETUPTYPE
{
OMX_U32 nTunnelFlags;
OMX_BUFFERSUPPLIERTYPE eSupplier;
} OMX_TUNNELSETUPTYPE;
```

#####3.1.2.10.1  nTunnelFlags
The nTunnelFlags integer parameter contains one or more bit flags applied to the port that receives this structure. Flags include:
``` C
#define OMX_PORTTUNNELFLAG_READONLY 0x00000001
```

If the flag is set as read only, the input port that receives this structure cannot alter the contents of buffers supplied on the tunnel.

#####3.1.2.10.2  eSupplier
The `eSupplier` field defines whether the input port or the output port provides the buffers. The exact sequence of calls to set up a tunnel is specified in section 3.4.1.2.

####3.1.2.11  OMX_PARAM_PORTDEFINITIONTYPE
The `OMX_PARAM_PORTDEFINITIONTYPE` structure contains a set of generic fields that characterize each port of the component. Some of these fields are common to all domains while other fields are specific to their respective domains. The IL client uses this structure to retrieve general information from each port.

`OMX_PARAM_PORTDEFINITIONTYPE` is defined as follows.

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
`nPortIndex` is a read-only field the identifies the port. The value of nPortIndex is a unique 32-bit number for the component. No two ports on a single component may share the same port number, but ports on different components may have the same port number.

#####3.1.2.11.2  eDir
eDir is a read-only field that indicates the direction (`OMX_DirInput` or `OMX_DirOutput`) for the port.

#####3.1.2.11.3  nBufferCountActual
`nBufferCountActual` represents the number of buffers that are required on this port before it is populated, as indicated by the bPopulated field of this structure. The component shall set a default value no less than nBufferCountMin for this field.

#####3.1.2.11.4  nBufferCountMin
`nBufferCountMin` is a read-only field that specifies the minimum number of buffers that the port requires. The component shall define this non-zero default value.

#####3.1.2.11.5  nBufferSize
`nBufferSize` is a read-only field that specifies the minimum size in bytes for buffers that are allocated for this port. .

#####3.1.2.11.6  bEnabled
`bEnabled` is a read-only Boolean field that indicates if the port is enabled. Ports default to bEnabled = `OMX_TRUE` and are enabled/disabled by sending the `OMX_CommandPortEnable` and `OMX_CommandPortDisable` commands with the `OMX_SendCommand` method.

A port shall not be populated when it is not enabled.

#####3.1.2.11.7  bPopulated
`bPopulated` is a read-only Boolean field that indicates if a port is populated. A port is populated when all of the buffers indicated by nBufferCountActual with a size of at least nBufferSize have been allocated on the port. A populated port shall be enabled. Enabled ports shall be populated on a transition to `OMX_StateIdle` and unpopulated on a transition to `OMX_StateLoaded`.

#####3.1.2.11.8  eDomain
`eDomain` is a read-only field that indicates the domain of the port. This field determines the contents of the format union explained in section 3.1.2.11.9.

#####3.1.2.11.9  format
The `format` fields are a union of domain-specific parameters. For more information on parameters for audio, video, image, and other domains, see section 4.

###3.1.3 OMX_PORTDOMAINTYPE
Table 3-8 enumerates the fields used in the `OMX_PARAM_PORTDEFINITIONTYPE` structure to define the domain of the port.
| Field Name | Description |
| ------- |
| OMX_PortDomainAudio | Specifies that the field format is a structure of the OMX_AUDIO_PORTDEFINITIONTYPE type. |
| OMX_PortDomainVideo | Specifies that the field format is a structure of the OMX_VIDEO_PORTDEFINITIONTYPE type. |
| OMX_PortDomainImage | Specifies that the field format is a structure of the OMX_IMAGE_PORTDEFINITIONTYPE type. |
| OMX_PortDomainOther | Specifies that the field format is a structure of the OMX_OTHER_PORTDEFINITIONTYPE type. |

**Table 3-8. Port Domain Names**

###3.1.4 OMX_HANDLETYPE
The `OMX_HANDLETYPE` structure defines the component handle as seen by the IL client. The component handle is used to access all of the public methods of the component. The component handle also contains pointers to the private data area of the component. The OpenMAX core allocates and initializes the component handle with help from the component during the process of loading the component. After the component is successfully loaded, the IL client can safely access any of the public functions of the component, although some may return an error because the state is inappropriate for the access.

##3.2  OpenMAX Core Methods/Macros
The OpenMAX core implements the main interface for an IL client that wants to use OpenMAX components. For efficiency, OpenMAX IL defines a set of OpenMAX core macros that map on one-to-one basis to most OpenMAX component methods. Some macros and methods recommend that the function return within either five milliseconds or 20 milliseconds, depending on the function. The 5-millisecond timeout was deemed by the standards body to be a reasonable response time for commands that
may not require buffer processing. The standards body identified the 20-millisecond timeout to be a reasonable response time for commands that may require buffer processing to be completed; the assumption here is that the longest buffer processing would be less than 30 milliseconds, which corresponds to 30-frames per second video. These timeouts are intended primarily to enable component integrators to get a good idea of component response latency via conformance testing.

The macros include the following:

- Get component information (version, capabilities).
- Set/Get component parameters at init time.
- Set/Get component parameters at run time.
- Allocate/De-allocate buffers.
- Send a buffer full of data to an OpenMAX component port.
- Send an empty buffer to an OpenMAX component port.
- Send commands to a component.
- Get the actual state of the component.
- Get references to OpenMAX component-proprietary parameters.

The OpenMAX Core also implements methods for the following:

- Initializing/de-initializing the whole OpenMAX IL Core
- Getting an OpenMAX component handle
- Releasing an OpenMAX component handle
- Detecting all OpenMAX components available on the platform at run time
- Setting up data tunnels among OpenMAX components

When a time limit for the execution of a method is specified, it is not intended as a hard restriction for the conformance of the component to the standard, but if the limit is not respected, a note shall appear in the description document related to the component.

###3.2.1 Return Codes for the Functions
Table 3-9 lists all of the possible return error codes for each function. A critical error denotes an error from which the component cannot recover. The component should transition to the `OMX_StateInvalid` state when a critical error occurs. All columns but the last two correspond to errors returned from a call to the component. The rightmost two columns denote errors sent asynchronously as the result of an internal error.

![](img/t3_9.png)

###3.2.2 Macros
This section describes the OpenMAX core macros.

Table 3-10 defines which macros may be called on a component in each component state.

![](img/t3_10.png)