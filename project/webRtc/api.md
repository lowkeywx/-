# 主要api分析

## CreatePeerConnectFactory and Media Engine

- webrtc::CreatePeerConnectionFactory
  - 接口中需要传入一些线程对象和aodio以及video codec。
  - 其中线程对象会直接存放到PeerConnectionFactoryContext中。
  - codec、mixer、process会组装成mediaEngine待创建ChannelManager的时候交给其管理。
  - 通过CreateCallFactory会创建call_factory
  - 通过CreateModularPeerConnectionFactory创建最终的PeerConnectionFactory
  
- CreateCallFactory
  - 创建CallFactory，只有一个接口，createCall。用于创建call，需要传入callConfig

- call
  - //TODO

- CreateMediaEngine
  - 创建WebRtcVoiceEngine，传入aduio codecs process mixer
  - 创建WebRtcVideoEngine，传入video codecs
  - 将创建的voiceEngine和VideoEngine交给CompositeMediaEngine，这个类没什么特别的。专门用来存放voice和video Engine的。

- CreateModularPeerConnectionFactory
  - 如果执行create操作的不是signal线程，则交给signal线程执行。
  - 通过PeerConnectionFactory::Create()创建peecConnectionFactory
  - 将signal_thread、worker_thread、pc_factory交给PeerConnectionFactoryProxy管理。

- PeerConnectionFactory::Create
  - 通过ConnectionContext::Create创建ConnectionContext。将CreatePeerConnectionFactory的所有参数交给ConnectionContext管理。
  - 将刚刚创建的context和CreatePeerConnectionFactory的参数交给PeerConnectionFactory实例。
  - 如果没有创建work_thread、network_thread、signaling_thread则在初始化ConnectionContex的时候创建，类型为thread。

- ConnectionContext::Create
  - 启动network_thread_、worker_thread_、signaling_thread_
  - 通过cricket::ChannelManager::Create创建channel_manager_，传入**webrtc::CreatePeerConnectionFactory**步骤中创建的media_engine和worker_thread等。

- cricket::ChannelManager::Create
  - 构造ChannelManager
  - 调用mediaEngine实例的init

- CompositeMediaEngine::Init()
  - voiceEngine的init
  - 目前没有调用videoEngine的init

- WebRtcVoiceEngine::Init() 
  - 收集音频相关的编码器和解码器
  - 根据CreatePeerConnectionFactory步骤是否创建了audio_Mixer、process、Device决定创建。
  - 调用ApplyOptions，配置音频相关降噪回音消除等
  - 会根据配置和硬件是否支持决定是否硬件支持的回音消除、降噪等功能
  - 将最终决定的配置交给audio_process


---

## CreatePeerConnection

- PeerConnectionFactoryInterface::CreatePeerConnection
  - 有两个重载的版本，四个参数的版本会创建dependence并调用两个参数的版本。
  - 调用PeerConnectionFactory::CreatePeerConnectionOrError

- PeerConnectionFactory::CreatePeerConnectionOrError
  - 创建cert_generator
  - 创建allocator，端口选择器，类型为BasicPortAllocator
  - 创建async_resolver_factory，//TODO
  - 创建ice_transport_factory，//TODO
  - 创建call，//TODO
  - 通过PeerConnection::Create创建peerConnection，接受五步创建的实例作为构造参数。config主要就是turn和stun服务器的地址。

- BasicPortAllocator::BasicPortAllocator
  - 设置session_pool_size=0
  - 调用PortAllocator::SetConfiguration，除了构造会调用，AddTurnServer和Connection的Initialize也会调用

- PortAllocator::SetConfiguration
  - 详情见下文

- PeerConnection::Create
  - 创建dns解析模块
  - 检测ice相关的配置参数
  - 构造peerconnection并且调用Initialize。构造中的context就是peecConnectionFactary中的Context

- PeerConnection::Initialize
  - 解析iceconfig也就是turn和sturn服务器
  - 调用InitializePortAllocator_n初始化PortAllocator
  - 记录configuration_
  - 构造StatsCollector，保存为成员。
  - 构造RTCStatsCollector，保存为成员。
  - 构造SdpOfferAnswerHandler，保存为成员。
  - 构造RtpTransmissionManager，保存为成员。

- PeerConnection::InitializePortAllocator_n
  - 调用PortAllocator::Initialize，内部没什么逻辑，就是设置了一个初始化状态
  - 检查配置信息是否有效
  - 调用PortAllocator::SetConfiguration重设PortAllocator

- PortAllocator::SetConfiguration
  - 记录stun_server到stun_servers_成员
  - 记录turn_server到turn_servers_成员
  - 设置turn server筛选偏好turn_port_prune_policy_
  - 设置condidata_pool_size,对应的是PortAllocatorSession的个数
  - 如果当前的PortAllocatorSession个数少于condidata_pool_size则通过CreateSessionInternal创建新的session对象加入到pooled_sessions_成员，如果多余则从末尾删除多的
  - 调用BasicPortAllocatorSession::StartGettingPorts开始收集port
  
- BasicPortAllocatorSession::StartGettingPorts
  - 创建socket_factory,类型为BasicPacketSocketFactory,BasicPacketSocketFactory需要一个SocketServer来自于networkthread。networkthread维护epoll，出发socket事件。
  - 调用BasicPortAllocatorSession::GetPortConfigurations，异步调用
  
- BasicPortAllocatorSession::GetPortConfigurations
  - 根据strun和turn服务器地址创建PortConfiguration对象
  - 调用BasicPortAllocatorSession::ConfigReady

- BasicPortAllocatorSession::ConfigReady
  - 通过network thread 异步调用BasicPortAllocatorSession::OnConfigReady

- BasicPortAllocatorSession::OnConfigReady
  - 将刚刚创建的config加入到configs成员
  - 调用BasicPortAllocatorSession::AllocatePorts

- BasicPortAllocatorSession::AllocatePorts
  - 调用BasicPortAllocatorSession::OnAllocate

- BasicPortAllocatorSession::OnAllocate
  - 调用BasicPortAllocatorSession::DoAllocate
  - 状态变更为allocation_started_=true

- BasicPortAllocatorSession::DoAllocate
  - 通过AllocationSequence开始收集port

- AllocationSequence::Init
  - 创建udp socket
  - 绑定OnReadPacket回调

- AllocationSequence::Start
  - 通过network thread 异步调用AllocationSequence::Process

- AllocationSequence::Process
  - 根据阶段创建不同的socket，主要为UDPPort、stunPort、RelayPorts、TCPPorts
  - 执行更新阶段计数，进入下一个阶段，即process自身
  - 最终调用BasicPortAllocatorSession::AddAllocatedPort添加port

- BasicPortAllocatorSession::AddAllocatedPort
  - 设置port的属性
  - 加入到ports
  - 绑定OnCandidateReady、OnPortComplete等回调

- StatsCollector
  - //TODO

- RTCStatsCollector
  - //TODO

- SdpOfferAnswerHandler
  - 完成了SessionDescription的构建工作，就是所谓的SDP协议内容。
  - 除了自己，还需要结合MediaSessionOptions和RTCOfferAnswerOptions共同作用
  - 是SdpStateProvider的子类，会传递给WebRtcSessionDescriptionFactory，用来获取SDP中的一些信息。
  - CreateSessionDescriptionRequest会作为最终的参数传入到WebRtcSessionDescriptionFactory，最终由WebRtcSessionDescriptionFactory创建处offer的SDP。
  - sdp的类型为JsepSessionDescription
  - SessionDescription作为sdp的主要参数，由WebRtcSessionDescriptionFactory创建。

- RtpTransmissionManager
  - //TODO

> 到这一步peerConnection的创建就完成了，之后建立p2p的过程以及推拉流相关的操作都是通过此实例完成的。建立链接需要的call ice已经创建并且添加。音视频处理需要的mediaEngine已经创建。

---

## Prepare Audio Track And Transceiver

- PeerConnectionFactory::CreateAudioTrack
  - 线程检测
  - 调用AudioTrack::Create

- AudioTrack::Create
  - 构造AudioTrack

- AudioTrack::AudioTrack
  - 设置了成员audioSource也是唯一的成员。
  - 提供了getSource和addSink方法
  - addSink只是透传Sink给Source
  - 作为AudioSource的观察者

- PeerConnectionFactory::CreateAudioSource
  - 检测线程
  - 调用LocalAudioSource::Create

- LocalAudioSource::Create
  - 构造LocalAudioSource
  - 调用Initialize

- PeerConnection::AddTrack
  - rtp_manager添加track，并且为track创建sender、reciver和transceiver。
  - rtp_manager是peerConnection init的时候初始化的。
  - state添加track，//TODO

- RtpTransmissionManager::AddTrack
  - 根据IsUnifiedPlan决定调用。
  - 通过RtpTransmissionManager::AddTrackUnifiedPlan，创建创建sender、reciver和transceiver。
  - 将新创建的transceiver保存起来。

- RtpTransmissionManager::AddTrackUnifiedPlan
  - 通过RtpTransmissionManager::CreateSender创建sender
  - 通过RtpTransmissionManager::CreateReceiver创建receiver
  - 通过RtpTransmissionManager::CreateAndAddTransceiver创建transmission，传入sender和receiver

- RtpTransmissionManager::CreateSender
  - //TODO

- RtpTransmissionManager::CreateReceiver
  - //TODO

- RtpTransceiver
  - //TODO

---

## Create Offer And Prepare SessionDescription(SDP)

- PeerConnection::CreateOffer
  - SdpOfferAnswerHandler::CreateOffer,感觉peerConnection就像是一个管家，所有工作都转发给不同的功能模块。
  - offer就是根据自身创建peerConnection是传入的参数以及自身的硬件情况，封装SDP数据。具体信息可以参考SDP协议。其中包括seesion描述信息。媒体描述信息。

- SdpOfferAnswerHandler::CreateOffer
  - 将真正的createOffer操作加入到调用莲。下一个调用回以回调参数的形式传递给当前的调用。
  - SdpOfferAnswerHandler::DoCreateOffer

- SdpOfferAnswerHandler::DoCreateOffer
  - 通过SdpOfferAnswerHandler::GetOptionsForOffer获取sdp的值。传入RTCOfferAnswerOptions，传出MediaSessionOptions。这个MediaSessionOptions就是sdp的雏形。
  - 调用WebRtcSessionDescriptionFactory::CreateOffer

- WebRtcSessionDescriptionFactory::CreateOffer
  - 构建CreateSessionDescriptionRequest，数据来源于MediaSessionOptions。并且作为InternalCreateOffer的参数。
  - 调用WebRtcSessionDescriptionFactory::InternalCreateOffer

- WebRtcSessionDescriptionFactory::InternalCreateOffer
  - 根据CreateSessionDescriptionRequest构造SessionDescription
  - 通过CopyCandidatesFromSessionDescription把SdpOfferAnswerHandler中的local_description赋值给SDP。
  - 最后通过onSuccess回调给应用层，SDP作为参数。

- SDP类流程
  - RTCOfferAnswerOptions->MediaSessionOptions->CreateSessionDescriptionRequest->MediaDescriptionOptions->SessionDescription->JsepSessionDescription。通过SdpOfferAnswerHandler和WebRtcSessionDescriptionFactory配合完成SDP的创建工作。

---

## 配置SDP，准备p2p

- PeerConnection::SetLocalDescription
  - 调用SdpOfferAnswerHandler::SetLocalDescription，没啥好说的直接透传了。

- SdpOfferAnswerHandler::SetLocalDescription
  - 加入调用链ChainOperation，调用连前面有说明过。
  - 调用SdpOfferAnswerHandler::DoSetLocalDescription

- SdpOfferAnswerHandler::DoSetLocalDescription
  - 创建各种类型的transprot，最终都交给JsepTransport统一管理和使用。
  - 屌用SdpOfferAnswerHandler::ApplyLocalDescription应用SessionDescription
  - 通过Oncuccese通知observer
  - 通过MaybeStartGathering，主要是端口的探测以及更新ice状态。

- SdpOfferAnswerHandler::ApplyLocalDescription
  - 将新创建的SessionDescription保存给pending_local_description_，然后local_description()就可以使用了。
  - 创建新的videoChannel、audioChannel、dataChannel，清理旧的channel。并将除了dataChannel之外的配置给对应的Transceiver。Transceiver保存在rtc_manager中。rtc_manager()实际使用的是PeerConnection中的。Transceiver在AddTrack的时候创建的。
  - 调用SdpOfferAnswerHandler::PushdownTransportDescription，更新transport_controller中ice过程需要用到的信息。
  - 调用SdpOfferAnswerHandler::UpdateSessionState通知应用层，给channel和Transceiver设置SessionDescription。

- SdpOfferAnswerHandler::PushdownTransportDescription
  - 根据是local还是remote决定调用JsepTransportController::SetLocalDescription，还是JsepTransportController::SetRemoteDescription
  - 最终都会调用JsepTransportController::ApplyDescription_n

- SdpOfferAnswerHandler::UpdateSessionState
  - 通过ChangeSignalingState通知应用层，没什么特别的，透传到PeerConnection->Oberser->ChangeSignalingState，通知应用层。
  - 调用PushdownMediaDescription将SessionDescription设置给channel和Transceiver。

- SdpOfferAnswerHandler::PushdownMediaDescription
  - 遍历所有Transceiver，从SessionDescription中获取所有属于对应流的Contint_info，并通过OnNegotiationUpdate, 更新Transceiver中的变量。
  - 遍历所有channel，通过BaseChannel::SetLocalContent设置rtc协议的相关内容。meidia_channel中设置了rtp协议。协议的相关结构为VoiceMediaInfo和VideoMediaInfo。并且通过GetStat获取。从media_channel中通过setLocalContent设置的。
  - 根据SessionDescription中的信息，更新MediaCHannel中的成员，用于通信使用。

> SdpOfferAnswerHandler中的许多功能都来自于PeerConnection。例如，transportController就是通过peerConnection类型的成员pc获得的。

# 接收到信令服务器的sdp消息

- PeerConnection::SetRemoteDescription
  - 调用SdpOfferAnswerHandler::SetRemoteDescription。没啥好说的，直接透传了。

- SdpOfferAnswerHandler::SetRemoteDescription
  - 组装RemoteDescriptionOperation，SdpOfferAnswerHandler作为参数
  - 将SdpOfferAnswerHandler::DoSetRemoteDescription加入调用链。

- SdpOfferAnswerHandler::DoSetRemoteDescription
  - 初步判断远端sdp的状态是否合法
  - 判断RemoteDescriptionOperation是否有效
  - 调用SdpOfferAnswerHandler::ApplyRemoteDescription

- SdpOfferAnswerHandler::ApplyRemoteDescription
  - 调用RemoteDescriptionOperation::ReplaceRemoteDescriptionAndCheckEror
  - 调用RemoteDescriptionOperation::UpdateChannels，//TODO
  - 调用RemoteDescriptionOperation::UpdateSessionState， //TODO
  - 调用RemoteDescriptionOperation::UseCandidatesInRemoteDescription， //TODO
  - 根据remote sdp中的media信息，创建stream、transceiver相关对象。

- RemoteDescriptionOperation::ReplaceRemoteDescriptionAndCheckEror
  - 调用SdpOfferAnswerHandler::ReplaceRemoteDescription，透传

- SdpOfferAnswerHandler::ReplaceRemoteDescription
  - 获取SessionDescription
  - 调用JsepTransportController::SetRemoteDescription

- JsepTransportController::SetRemoteDescription
  - 判断当前线程是否是网络线程，如果不是，将调用转移到网络线程
  - 调用JsepTransportController::ApplyDescription_n

- JsepTransportController::ApplyDescription_n
  - 检测bundleGroups，//TODO
  - 根据SessionDescription中的media信息创建Transport对象，实际为遍历所有MediaDescription，且逐一调用JsepTransportController::MaybeCreateJsepTransport。
  - 遍历所有MediaDescription，获取刚刚创建的Transport对象。
  - 通过JsepTransportController::CreateJsepTransportDescription创建JsepTransportDescription对象
  - 通过JsepTransport::SetRemoteJsepTransportDescription或者JsepTransport::SetLocalJsepTransportDescription设置给JsepTransport

- JsepTransportController::MaybeCreateJsepTransport
  - 调用JsepTransportController::CreateIceTransport创建P2PTransportChannel对象
  - 根据P2PTransportChannel和mediaDescription创建DtlsTransportInternal
  - 根据配置决定是否创建srtp_transport、sdes_transport、sctp_transport
  - 将创建的P2PTransportChannel和各种transport对象交给JsepTransport
  - 将创建的JsepTransport注册进transports中

- JsepTransportController::CreateIceTransport
  - 调用ice_transport_factory的CreateIceTransport，创建P2PTransportChannel对象

- JsepTransportController::CreateJsepTransportDescription
  - 创建JsepTransportDescription对象，该对象封装了TransportDescription，而TransportDescription就是ice相关的信息，包括iecpwd、iceMode、connection_Role等。

- JsepTransport::SetRemoteJsepTransportDescription
  - 将JsepDescription设置给remote_description_成员
  - 将提取的ice信息设置给各种transport对象，例如：通过JsepTransport::SetRemoteIceParameters设置给ice_transport、通过NegotiateAndSetDtlsParameters设置给其他transport对象。

- JsepTransport::SetLocalJsepTransportDescription
  - 同SetRemoteJsepTransportDescription类似

# add remote ice condidate

- PeerConnection::AddIceCandidate
  - 调用SdpOfferAnswerHandler::AddIceCandidate，直接透传

- SdpOfferAnswerHandler::AddIceCandidate
  - 根据参数决定调用版本，最终都会调用 SdpOfferAnswerHandler::AddIceCandidateInternal

- SdpOfferAnswerHandler::AddIceCandidateInternal
  - 调用SdpOfferAnswerHandler::ReadyToUseRemoteCandidate，检测是否已经保存了该ice_condidate
  - 将新的ice condidate添加到pedding_remote_description中
  - 调用SdpOfferAnswerHandler::UseCandidate

- SdpOfferAnswerHandler::UseCandidate
  - 再次检测condidate
  - 调用PeerConnection::AddRemoteCandidate

- PeerConnection::AddRemoteCandidate
  - 组装任务，添加到任务队列
  - 组装的任务为：调用JsepTransportController::AddRemoteCandidates

- JsepTransportController::AddRemoteCandidates
  - 根据sdp中的media id确定一个Transport，每个媒体类型都会有一个对应的transport
  - 调用JsepTransport::AddRemoteCandidates

- JsepTransport::AddRemoteCandidates
  - 遍历所有condidate，调用transport所管理的transport的AddRemoteCandidate，即P2PTransportChannel::AddRemoteCandidate

- P2PTransportChannel::AddRemoteCandidate
  - 如果是域名需要解析ip地址，调用ResolveHostnameCandidate获得最终的ip，异步调用，获取最终的ip地址
  - 调用P2PTransportChannel::FinishAddingRemoteCandidate

- P2PTransportChannel::FinishAddingRemoteCandidate
  - 如果已经创建过connection，需要更新connection中的remote_candidate
  - 调用P2PTransportChannel::CreateConnections创建connections

- P2PTransportChannel::CreateConnections
  - 遍历之前MaybeStartGathering中探测的端口，逐一调用P2PTransportChannel::CreateConnection
  - 需要根据探测的端口会和session绑定，session通过AddAllocatorSession添加到allocat_sessions中，类型为FakePortAllocatorSession。session_pool中没有可用session的时候才会创建。
  - session用来创建获取可用的port对象。

- P2PTransportChannel::CreateConnection
  - port对象中维护多个connection
  - 从port中获取一个connection，如果没有空闲的就创建一个新的。
  - 调用P2PTransportChannel::AddConnection，设置connection的基本信息，并且添加到ice_controller中

- P2PTransportChannel::AddConnection
  - 设置ice的超时时间等信息
  - 绑定回调
  - 通过调用BasicIceController::AddConnection添加到connections成员中，之后会选取最有的connection
  - 这里没太看明白，以后分析。 //TODO

# 本地condidate收集过程

- StunBindingRequest::OnResponse
  - 从response中解析address
  - 创建SocketAddress对象，并通过回调UDPPort::OnStunBindingRequestSucceeded通知port

- UDPPort::OnStunBindingRequestSucceeded
  - 保存成功通信的stun服务器地址
  - 保存turn地址，一边p2p失败的时候使用
  - 调用Port::AddAddress，通知session

- Port::AddAddress
  - 组装Candidate
  - 调用Port::FinishAddingAddress，传入刚刚创建的condidate

- Port::FinishAddingAddress
  - 保存收集到的condidate
  - 通过SignalCandidateReady通知上层，BasicPortAllocatorSession::OnCandidateReady

- BasicPortAllocatorSession::OnCandidateReady
  - 通过SignalCandidatesReady通知上层
  - MaybeSignalCandidatesAllocationDone

- P2PTransportChannel::OnCandidatesReady
  - 将所有的condidate通过SignalCandidateGathered通知上层

- JsepTransportController::OnTransportCandidateGathered_n
  - 通过signal_ice_candidates_gathered_通知上层
  - 通过SubscribeIceCandidateGathered绑定的回调PeerConnection::OnTransportControllerCandidatesGathered

- PeerConnection::OnTransportControllerCandidatesGathered
  - 组装成JsepIceCandidate
  - 通过SdpOfferAnswerHandler::AddLocalIceCandidate记录下来，并且会更新session description中的信息
  - 通过OnIceCandidate通知observer，其实就是应用层，收到以后通过信令服务器发送到对端，对端再通过addIceCondidate设置

# p2p链接建立过程
- 创建PeerConnection
- 创建BasicPortAllocator
- 创建BasicPortAllocatorSession加入session pool
- 创建UdpPort、TurnPort、StunPort、TcpPort等
- 创建StunPort的同时触发SendStunBindingRequests，发送stun探测消息
- 收到stun服务器的回复，触发socket的读事件，进而触发Udp的OnReadPacket。
- 触发StunBindingRequest::OnResponse剩余参考**本地condidate收集过程**
- 通过信令服务器发送到对断
- 接受到远端Condidate，通过addIceCondatate设置到
- 剩余参考**add remote ice condidate**
- 然后会选择一个connection，发送sturn request出发远端的HandleStunBindingOrGoogPingRequest
- 通过SendStunBindingResponse发送response
- 触发connection的OnReadPacket回调
- 触发Connection::HandleStunBindingOrGoogPingRequest
- 回复response
- 远端出发OnReadPacket
- 触发Connection的OnReadPacket
- 触发SignalStateChange和SignalNominated回调。两者都会出发connection重拍的动作。即P2PTransportChannel::RequestSortAndStateUpdate

# 流的启动

- stream的启动时跟随sdp的创建和接受自动进行的。
- 对于流的一些控制主要是通过WebRtcVoiceMediaChannel，通过ssrc精准控制是哪一个流。
- SdpOfferAnswerHandler::PushdownMediaDescription控制流的创建与启动
- ApplyLocalDescription时本地流创建，同时会检查时候创建了频道，如果没有创建会创建channel
- ApplyRemoteDescription时远端流创建。

# 音频数据流香
- WebRtcAudioSendStream继承自sink，Source通过sink的Ondata传递数据

- WebRtcAudioSendStream::OnData
  - 创建AudioFrame，并通过UpdateFrame接口更新audio数据和一些属性
  - 调用AudioSendStream::SendAudioData处理数据
  - 此处的Source是track，已经是处理完的数据了，就剩下encode和send了

- WebRtcAudioSendStream中Source来源
  - WebRtcVoiceMediaChannel::SetLocalSource
  - WebRtcVoiceMediaChannel::SetAudioSend
  - AudioRtpSender::SetSend中的sink_adapter_
  - RtpSenderBase::SetTrack
  - RtpTransmissionManager::AddTrackUnifiedPlan或者RtpTransmissionManager::CreateSender
  - 来源都是RtpTransmissionManager::AddTrack
  - PeerConnection::AddTrack

- 采集数据流香
  - AudioProcessingImpl::ProcessCaptureStreamLocked调用处理aec、agc等
  - AudioProcessingImpl::ProcessStream
  - audio_frame_proxies文件中的ProcessAudioFrame
  - AudioTransportImpl::ProcessCaptureFrame
  - AudioTransportImpl::RecordedDataIsAvailable，完成了音频数据的process和SendAudioData
  - AudioDeviceBuffer::DeliverRecordedData
  - AudioDeviceWindowsCore::StartRecording采集音频，调用SetRecordedBuffer传递音频数据。
  - WebRtcVoiceEngine::Init创建AudioDeviceModule实例
  - 创建AudioDeviceModule实例的时候会创建AudioDeviceModuleImpl和AudioDeviceBuffer并讲buffer绑定到AudioDeviceModuleImpl的成员AudioDeviceWindowsCore中。


