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
  - 创建allocator，端口选择器
  - 创建async_resolver_factory，//TODO
  - 创建ice_transport_factory，//TODO
  - 创建call，//TODO
  - 通过PeerConnection::Create创建peerConnection，接受五步创建的实例作为构造参数。config主要就是turn和stun服务器的地址。

- PeerConnection::Create
  - 创建dns解析模块
  - 检测ice相关的配置参数
  - 构造peerconnection并且调用Initialize。构造中的context就是peecConnectionFactary中的Context

- PeerConnection::Initialize
  - 解析iceconfig也就是turn和sturn服务器
  - 记录configuration_
  - 构造StatsCollector，保存为成员。
  - 构造RTCStatsCollector，保存为成员。
  - 构造SdpOfferAnswerHandler，保存为成员。
  - 构造RtpTransmissionManager，保存为成员。

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
  - 开始ice的过程

- SdpOfferAnswerHandler::ApplyLocalDescription
  - 将新创建的SessionDescription保存给pending_local_description_，然后local_description()就可以使用了。
  - 创建新的videoChannel、audioChannel、dataChannel，清理旧的channel。并将除了dataChannel之外的配置给对应的Transceiver。Transceiver保存在rtc_manager中。rtc_manager()实际使用的是PeerConnection中的。Transceiver在AddTrack的时候创建的。
  - 调用SdpOfferAnswerHandler::UpdateSessionState通知应用层，给channel和Transceiver设置SessionDescription。
  
- SdpOfferAnswerHandler::UpdateSessionState
  - 通过ChangeSignalingState通知应用层，没什么特别的，透传到PeerConnection->Oberser->ChangeSignalingState，通知应用层。
  - 调用PushdownMediaDescription将SessionDescription设置给channel和Transceiver。

- SdpOfferAnswerHandler::PushdownMediaDescription
  - 遍历所有Transceiver，从SessionDescription中获取所有属于对应流的Contint_info，并通过OnNegotiationUpdate, 更新Transceiver中的变量。
  - 遍历所有channel，通过BaseChannel::SetLocalContent设置rtc协议的相关内容。meidia_channel中设置了rtp协议。协议的相关结构为VoiceMediaInfo和VideoMediaInfo。并且通过GetStat获取。从media_channel中通过setLocalContent设置的。
  - 根据SessionDescription中的信息，更新MediaCHannel中的成员，用于通信使用。



