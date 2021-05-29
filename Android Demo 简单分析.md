# Android Demo 简单分析
----
[TOC]

## 1 PeerConnection 创建

音视频互通需要先创建 PeerConnection，然后通过创建及交换 Sdp，以完成通话建立。

### 1.1 创建 PeerConnectionFactory 

```java
// CallActivity.java
CallActivity.onCreate() {
    // Create peer connection client.
    // 创建 peer connection client, 并调用 PeerConnectionFactory.initialize() 加载动态库及上下文环境等.
    peerConnectionClient = new PeerConnectionClient(
        getApplicationContext(), eglBase, peerConnectionParameters, CallActivity.this);

    /* 创建 PeerConnectionFactory */
    PeerConnectionFactory.Options options = new PeerConnectionFactory.Options();
    PeerConnectionClient.createPeerConnectionFactory(options)
    
    /* 开始通话建立 */
    startCall()
}
```

```java
// PeerConnectionClient.java
public void createPeerConnectionFactory(PeerConnectionFactory.Options options) {
    if (factory != null) {
        throw new IllegalStateException("PeerConnectionFactory has already been constructed");
    }
    executor.execute(() -> createPeerConnectionFactoryInternal(options));
}

createPeerConnectionFactoryInternal() {
    /* 创建音频设备模块 */
    final AudioDeviceModule adm = createJavaAudioDevice();

    /* 视频编码是否使能 H264 high level profile */
    final boolean enableH264HighProfile =
        VIDEO_CODEC_H264_HIGH.equals(peerConnectionParameters.videoCodec);
    final VideoEncoderFactory encoderFactory;
    final VideoDecoderFactory decoderFactory;

    /* 创建视频编解码工厂, 软编或硬编 */
    if (peerConnectionParameters.videoCodecHwAcceleration) {
      encoderFactory = new DefaultVideoEncoderFactory(
          rootEglBase.getEglBaseContext(), true /* enableIntelVp8Encoder */, enableH264HighProfile);
      decoderFactory = new DefaultVideoDecoderFactory(rootEglBase.getEglBaseContext());
    } else {
      encoderFactory = new SoftwareVideoEncoderFactory();
      decoderFactory = new SoftwareVideoDecoderFactory();
    }

    /* 构建 PeerConnectionFactory */
    factory = PeerConnectionFactory.builder()
                  .setOptions(options)
                  .setAudioDeviceModule(adm)
                  .setVideoEncoderFactory(encoderFactory)
                  .setVideoDecoderFactory(decoderFactory)
                  .createPeerConnectionFactory();
    
    adm.release();
}

// sdk/android/api/org/webrtc/PeerConnectionFactory.java
// PeerConnectionFactory.Builder
public PeerConnectionFactory createPeerConnectionFactory() {
    // 通过 Jni 创建 Native 层 PeerConnectionFactory 对象
    return nativeCreatePeerConnectionFactory(ContextUtils.getApplicationContext(), options,
        /* 获取音频设备 Native C++指针 */
        audioDeviceModule.getNativeAudioDeviceModulePointer(),
        /* 创建音频编码工厂类 Native C++ 指针 */
        audioEncoderFactoryFactory.createNativeAudioEncoderFactory(),
        /* 创建音频解码工厂类 Native C++ 指针 */
        audioDecoderFactoryFactory.createNativeAudioDecoderFactory(), 
        /* 视频编码器工厂类对象 */
        videoEncoderFactory,
        /* 视频解码器工厂类对象 */
        videoDecoderFactory,
        /* 音频增强处理，此处为 null */
        audioProcessingFactory == null ? 0 : audioProcessingFactory.createNative(),
        /* fec 控制器，此处为 null */
        fecControllerFactoryFactory == null ? 0 : fecControllerFactoryFactory.createNative(),
        /* 此处为 null */
        networkControllerFactoryFactory == null
            ? 0
            : networkControllerFactoryFactory.createNativeNetworkControllerFactory(),
        /* 此处为 null */
        networkStatePredictorFactoryFactory == null
            ? 0
            : networkStatePredictorFactoryFactory.createNativeNetworkStatePredictorFactory(),
        /* 此处为 null */
        mediaTransportFactoryFactory == null
            ? 0
            : mediaTransportFactoryFactory.createNativeMediaTransportFactory(),
        /* 此处为 null */
        neteqFactoryFactory == null ? 0 : neteqFactoryFactory.createNativeNetEqFactory());
}
```

```c++
// out/android_arm64_Debug/gen/sdk/android/generated_peerconnection_jni/PeerConnection_factory_jni.h
// 该文件根据 sdk/android/api/org/webrtc/PeerConnectionFactory.java 在编译时动态生成
/* 创建 Native 层 PeerConnectionFactory 对象 */
JNI_GENERATOR_EXPORT jobject
    Java_org_webrtc_PeerConnectionFactory_nativeCreatePeerConnectionFactory(
    JNIEnv* env,
    jclass jcaller,
    jobject context,
    jobject options,
    jlong nativeAudioDeviceModule,
    jlong audioEncoderFactory,
    jlong audioDecoderFactory,
    jobject encoderFactory,
    jobject decoderFactory,
    jlong nativeAudioProcessor,
    jlong nativeFecControllerFactory,
    jlong nativeNetworkControllerFactory,
    jlong nativeNetworkStatePredictorFactory,
    jlong mediaTransportFactory,
    jlong neteqFactory) {
    return JNI_PeerConnectionFactory_CreatePeerConnectionFactory(env,
        base::android::JavaParamRef<jobject>(env, context), 
        base::android::JavaParamRef<jobject>(env, options), 
        nativeAudioDeviceModule, 
        audioEncoderFactory, audioDecoderFactory,
        base::android::JavaParamRef<jobject>(env, encoderFactory),
        base::android::JavaParamRef<jobject>(env, decoderFactory),
        nativeAudioProcessor,
        nativeFecControllerFactory, nativeNetworkControllerFactory,
        nativeNetworkStatePredictorFactory, mediaTransportFactory, neteqFactory).Release();
}

// sdk/android/src/jni/pc/peer_connection_factory.cc
static ScopedJavaLocalRef<jobject>
JNI_PeerConnectionFactory_CreatePeerConnectionFactory(
    JNIEnv* jni,
    const JavaParamRef<jobject>& jcontext,
    const JavaParamRef<jobject>& joptions,
    jlong native_audio_device_module,
    jlong native_audio_encoder_factory,
    jlong native_audio_decoder_factory,
    const JavaParamRef<jobject>& jencoder_factory,
    const JavaParamRef<jobject>& jdecoder_factory,
    jlong native_audio_processor,
    jlong native_fec_controller_factory,
    jlong native_network_controller_factory,
    jlong native_network_state_predictor_factory,
    jlong native_media_transport_factory,
    jlong native_neteq_factory) {
  rtc::scoped_refptr<AudioProcessing> audio_processor =
      reinterpret_cast<AudioProcessing*>(native_audio_processor);
  return CreatePeerConnectionFactoryForJava(
      jni, jcontext, joptions,
      // Native 音频设备对象
      reinterpret_cast<AudioDeviceModule*>(native_audio_device_module),
      TakeOwnershipOfRefPtr<AudioEncoderFactory>(native_audio_encoder_factory),
      TakeOwnershipOfRefPtr<AudioDecoderFactory>(native_audio_decoder_factory),
      jencoder_factory, jdecoder_factory,
      // 创建音频增强处理对象
      audio_processor ? audio_processor : CreateAudioProcessing(),
      TakeOwnershipOfUniquePtr<FecControllerFactoryInterface>(
          native_fec_controller_factory),
      TakeOwnershipOfUniquePtr<NetworkControllerFactoryInterface>(
          native_network_controller_factory),
      TakeOwnershipOfUniquePtr<NetworkStatePredictorFactoryInterface>(
          native_network_state_predictor_factory),
      TakeOwnershipOfUniquePtr<MediaTransportFactory>(
          native_media_transport_factory),
      TakeOwnershipOfUniquePtr<NetEqFactory>(native_neteq_factory));
}

// Following parameters are optional:
// |audio_device_module|, |jencoder_factory|, |jdecoder_factory|,
// |audio_processor|, |media_transport_factory|, |fec_controller_factory|,
// |network_state_predictor_factory|, |neteq_factory|.
ScopedJavaLocalRef<jobject> CreatePeerConnectionFactoryForJava(
    JNIEnv* jni,
    const JavaParamRef<jobject>& jcontext,
    const JavaParamRef<jobject>& joptions,
    rtc::scoped_refptr<AudioDeviceModule> audio_device_module,
    rtc::scoped_refptr<AudioEncoderFactory> audio_encoder_factory,
    rtc::scoped_refptr<AudioDecoderFactory> audio_decoder_factory,
    const JavaParamRef<jobject>& jencoder_factory,
    const JavaParamRef<jobject>& jdecoder_factory,
    rtc::scoped_refptr<AudioProcessing> audio_processor,
    std::unique_ptr<FecControllerFactoryInterface> fec_controller_factory,
    std::unique_ptr<NetworkControllerFactoryInterface>
        network_controller_factory,
    std::unique_ptr<NetworkStatePredictorFactoryInterface>
        network_state_predictor_factory,
    std::unique_ptr<MediaTransportFactory> media_transport_factory,
    std::unique_ptr<NetEqFactory> neteq_factory) {
    // talk/ assumes pretty widely that the current Thread is ThreadManager'd, but
    // ThreadManager only WrapCurrentThread()s the thread where it is first
    // created.  Since the semantics around when auto-wrapping happens in
    // webrtc/rtc_base/ are convoluted, we simply wrap here to avoid having to
    // think about ramifications of auto-wrapping there.
    rtc::ThreadManager::Instance()->WrapCurrentThread();

    // 创建 network 线程
    std::unique_ptr<rtc::Thread> network_thread =
        rtc::Thread::CreateWithSocketServer();
    network_thread->SetName("network_thread", nullptr);
    RTC_CHECK(network_thread->Start()) << "Failed to start thread";

    // 创建 worker 线程
    std::unique_ptr<rtc::Thread> worker_thread = rtc::Thread::Create();
    worker_thread->SetName("worker_thread", nullptr);
    RTC_CHECK(worker_thread->Start()) << "Failed to start thread";

    // 创建 signaling 线程
    std::unique_ptr<rtc::Thread> signaling_thread = rtc::Thread::Create();
    signaling_thread->SetName("signaling_thread", NULL);
    RTC_CHECK(signaling_thread->Start()) << "Failed to start thread";

    rtc::NetworkMonitorFactory* network_monitor_factory = nullptr;

    const absl::optional<PeerConnectionFactoryInterface::Options> options =
        JavaToNativePeerConnectionFactoryOptions(jni, joptions);

    // Do not create network_monitor_factory only if the options are
    // provided and disable_network_monitor therein is set to true.
    if (!(options && options->disable_network_monitor)) {
        network_monitor_factory = new AndroidNetworkMonitorFactory();
        rtc::NetworkMonitorFactory::SetFactory(network_monitor_factory);
    }

    PeerConnectionFactoryDependencies dependencies;
    dependencies.network_thread = network_thread.get();
    dependencies.worker_thread = worker_thread.get();
    dependencies.signaling_thread = signaling_thread.get();
    dependencies.task_queue_factory = CreateDefaultTaskQueueFactory();
    dependencies.call_factory = CreateCallFactory();
    dependencies.event_log_factory = std::make_unique<RtcEventLogFactory>(
        dependencies.task_queue_factory.get());
    dependencies.fec_controller_factory = std::move(fec_controller_factory);
    dependencies.network_controller_factory =
        std::move(network_controller_factory);
    dependencies.network_state_predictor_factory =
        std::move(network_state_predictor_factory);
    dependencies.media_transport_factory = std::move(media_transport_factory);
    dependencies.neteq_factory = std::move(neteq_factory);

    cricket::MediaEngineDependencies media_dependencies;
    media_dependencies.task_queue_factory = dependencies.task_queue_factory.get();
    // 外部创建音频设备
    media_dependencies.adm = std::move(audio_device_module);
    media_dependencies.audio_encoder_factory = std::move(audio_encoder_factory);
    media_dependencies.audio_decoder_factory = std::move(audio_decoder_factory);
    media_dependencies.audio_processing = std::move(audio_processor);
    media_dependencies.video_encoder_factory =
        absl::WrapUnique(CreateVideoEncoderFactory(jni, jencoder_factory));
    media_dependencies.video_decoder_factory =
        absl::WrapUnique(CreateVideoDecoderFactory(jni, jdecoder_factory));
    // 创建 MediaEngine
    dependencies.media_engine =
        cricket::CreateMediaEngine(std::move(media_dependencies));

    // 创建 PeerConnectionFactory，此调用与其他平台相同
    rtc::scoped_refptr<PeerConnectionFactoryInterface> factory =
        CreateModularPeerConnectionFactory(std::move(dependencies));

    RTC_CHECK(factory) << "Failed to create the peer connection factory; "
                            "WebRTC/libjingle init likely failed on this device";
    // TODO(honghaiz): Maybe put the options as the argument of
    // CreatePeerConnectionFactory.
    if (options)
        factory->SetOptions(*options);

    return NativeToScopedJavaPeerConnectionFactory(
        jni, factory, std::move(network_thread), std::move(worker_thread),
        std::move(signaling_thread), network_monitor_factory);
}

ScopedJavaLocalRef<jobject> NativeToScopedJavaPeerConnectionFactory(
    JNIEnv* env,
    rtc::scoped_refptr<webrtc::PeerConnectionFactoryInterface> pcf,
    std::unique_ptr<rtc::Thread> network_thread,
    std::unique_ptr<rtc::Thread> worker_thread,
    std::unique_ptr<rtc::Thread> signaling_thread,
    rtc::NetworkMonitorFactory* network_monitor_factory) {
    // OwnedFactoryAndThreads 保存维护 Native Factory 对象
    OwnedFactoryAndThreads* owned_factory = new OwnedFactoryAndThreads(
        std::move(network_thread), std::move(worker_thread),
        std::move(signaling_thread), network_monitor_factory, pcf);

    // 创建 Java 层的 PeerConnectionFactory 实例
    ScopedJavaLocalRef<jobject> j_pcf = Java_PeerConnectionFactory_Constructor(
        env, NativeToJavaPointer(owned_factory));

    // 回调 PeerConnectionFactory.onNetworkThreadReady()
    PostJavaCallback(env, owned_factory->network_thread(), RTC_FROM_HERE, j_pcf,
                    &Java_PeerConnectionFactory_onNetworkThreadReady);
    // 回调 PeerConnectionFactory.onWorkerThreadReady()
    PostJavaCallback(env, owned_factory->worker_thread(), RTC_FROM_HERE, j_pcf,
                    &Java_PeerConnectionFactory_onWorkerThreadReady);
    // 回调 PeerConnectionFactory.onSignalingThreadReady()
    PostJavaCallback(env, owned_factory->signaling_thread(), RTC_FROM_HERE, j_pcf,
                    &Java_PeerConnectionFactory_onSignalingThreadReady);

    return j_pcf;
}

static base::android::ScopedJavaLocalRef<jobject> Java_PeerConnectionFactory_Constructor(JNIEnv*
    env, jlong nativeFactory) {
    jclass clazz = org_webrtc_PeerConnectionFactory_clazz(env);
    CHECK_CLAZZ(env, clazz,
        org_webrtc_PeerConnectionFactory_clazz(env), NULL);

    jni_generator::JniJavaCallContextChecked call_context;
    call_context.Init<
        base::android::MethodID::TYPE_INSTANCE>(
            env,
            clazz,
            "<init>",
            "(J)V",
            &g_org_webrtc_PeerConnectionFactory_Constructor);

    // 构造 Java 层 PeerConnectionFactory 实例
    jobject ret =
        env->NewObject(clazz,
            call_context.base.method_id, nativeFactory);
    return base::android::ScopedJavaLocalRef<jobject>(env, ret);
}
```

### 1.2 创建 PeerConnection
1. 连接上房间服务器；
2. 若使能视频，则先创建视频采集源；
3. 创建 PeerConnection；

```java
// CallActivity.java
/* 信令层连接房间服务器成功, 创建 PeerConnection, 若是主叫端则创建 offer sdp，若是被叫端则需设置远端 Sdp 及创建本端 Sdp */
private void onConnectedToRoomInternal(final SignalingParameters params) {
    final long delta = System.currentTimeMillis() - callStartedTimeMs;

    signalingParameters = params;
    logAndToast("Creating peer connection, delay=" + delta + "ms");
    /* 创建视频采集实例，可为：视频文件、屏幕共享、摄像头采集 */
    VideoCapturer videoCapturer = null;
    if (peerConnectionParameters.videoCallEnabled) {
        videoCapturer = createVideoCapturer();
    }
    /* 创建 PeerConnection */
    peerConnectionClient.createPeerConnection(
        localProxyVideoSink, remoteSinks, videoCapturer, signalingParameters);

    if (signalingParameters.initiator) {
        logAndToast("Creating OFFER...");
        // Create offer. Offer SDP will be sent to answering client in
        // PeerConnectionEvents.onLocalDescription event.
        peerConnectionClient.createOffer();
    } else {
        if (params.offerSdp != null) {
            peerConnectionClient.setRemoteDescription(params.offerSdp);
            logAndToast("Creating ANSWER...");
            // Create answer. Answer SDP will be sent to offering client in
            // PeerConnectionEvents.onLocalDescription event.
            peerConnectionClient.createAnswer();
        }
        if (params.iceCandidates != null) {
            // Add remote ICE candidates from room.
            for (IceCandidate iceCandidate : params.iceCandidates) {
                peerConnectionClient.addRemoteIceCandidate(iceCandidate);
            }
        }
    }
}
```

```java
// PeerConnectionClient.java
public void createPeerConnection(final VideoSink localRender, final VideoSink remoteSink,
      final VideoCapturer videoCapturer, final SignalingParameters signalingParameters) {
    if (peerConnectionParameters.videoCallEnabled && videoCapturer == null) {
        Log.w(TAG, "Video call enabled but no video capturer provided.");
    }
    createPeerConnection(
        localRender, Collections.singletonList(remoteSink), videoCapturer, signalingParameters);
}

public void createPeerConnection(final VideoSink localRender, final List<VideoSink> remoteSinks,
      final VideoCapturer videoCapturer, final SignalingParameters signalingParameters) {
    if (peerConnectionParameters == null) {
        Log.e(TAG, "Creating peer connection without initializing factory.");
        return;
    }
    // 本地预览的 sink
    this.localRender = localRender;
    // 远端渲染的 sink
    this.remoteSinks = remoteSinks;
    // 视频源实例
    this.videoCapturer = videoCapturer;
    this.signalingParameters = signalingParameters;
    executor.execute(() -> {
      try {
            createMediaConstraintsInternal();
            createPeerConnectionInternal();
            maybeCreateAndStartRtcEventLog();
      } catch (Exception e) {
            reportError("Failed to create peer connection: " + e.getMessage());
            throw e;
      }
    });
}

private void createPeerConnectionInternal() {
    if (factory == null || isError) {
      Log.e(TAG, "Peerconnection factory is not created");
      return;
    }
    Log.d(TAG, "Create peer connection.");

    queuedRemoteCandidates = new ArrayList<>();

    PeerConnection.RTCConfiguration rtcConfig =
        new PeerConnection.RTCConfiguration(signalingParameters.iceServers);
    // TCP candidates are only useful when connecting to a server that supports
    // ICE-TCP.
    rtcConfig.tcpCandidatePolicy = PeerConnection.TcpCandidatePolicy.DISABLED;
    rtcConfig.bundlePolicy = PeerConnection.BundlePolicy.MAXBUNDLE;
    rtcConfig.rtcpMuxPolicy = PeerConnection.RtcpMuxPolicy.REQUIRE;
    rtcConfig.continualGatheringPolicy = PeerConnection.ContinualGatheringPolicy.GATHER_CONTINUALLY;
    // Use ECDSA encryption.
    rtcConfig.keyType = PeerConnection.KeyType.ECDSA;
    // Enable DTLS for normal calls and disable for loopback calls.
    rtcConfig.enableDtlsSrtp = !peerConnectionParameters.loopback;
    rtcConfig.sdpSemantics = PeerConnection.SdpSemantics.UNIFIED_PLAN;

    /* 创建 PeerConnection */
    peerConnection = factory.createPeerConnection(rtcConfig, pcObserver);

    if (dataChannelEnabled) {
        /* create data channel */
        ...
    }
    isInitiator = false;

    // Set INFO libjingle logging.
    // NOTE: this _must_ happen while |factory| is alive!
    Logging.enableLogToDebugOutput(Logging.Severity.LS_INFO);

    List<String> mediaStreamLabels = Collections.singletonList("ARDAMS");
    if (isVideoCallEnabled()) {
        /* 创建并添加视频源到 PeerConnection, 创建 VideoTrack 时将启动视频源采集 */
        peerConnection.addTrack(createVideoTrack(videoCapturer), mediaStreamLabels);

        // We can add the renderers right away because we don't need to wait for an
        // answer to get the remote track.
        remoteVideoTrack = getRemoteVideoTrack();
        remoteVideoTrack.setEnabled(renderVideo);
        for (VideoSink remoteSink : remoteSinks) {
            remoteVideoTrack.addSink(remoteSink);
        }
    }
    peerConnection.addTrack(createAudioTrack(), mediaStreamLabels);
    if (isVideoCallEnabled()) {
        findVideoSender();
    }

    ...

    Log.d(TAG, "Peer connection created.");
}
```

```java
// sdk/android/api/org/webrtc/PeerConnectionFactory.java
public PeerConnection createPeerConnection(
      PeerConnection.RTCConfiguration rtcConfig, PeerConnection.Observer observer) {
    return createPeerConnection(rtcConfig, null /* constraints */, observer);
}

PeerConnection createPeerConnectionInternal(PeerConnection.RTCConfiguration rtcConfig,
      MediaConstraints constraints/* null */, PeerConnection.Observer observer,
      SSLCertificateVerifier sslCertificateVerifier/* null */) {
    checkPeerConnectionFactoryExists();
    // 创建 PeerConnection.Observer 的 Native 层关联对象
    long nativeObserver = PeerConnection.createNativePeerConnectionObserver(observer);
    if (nativeObserver == 0) {
        return null;
    }
    long nativePeerConnection = nativeCreatePeerConnection(
        nativeFactory, rtcConfig, constraints, nativeObserver, sslCertificateVerifier);
    if (nativePeerConnection == 0) {
        return null;
    }
    return new PeerConnection(nativePeerConnection);
}

// sdk/android/api/org/webrtc/PeerConnection.java
PeerConnection(long nativePeerConnection) {
    this.nativePeerConnection = nativePeerConnection;
}
```

绑定 Java 层 PeerConnection Observer 对象，后续通过该 Observer 将 Native 层事件回调到 Java 层。
其中，PeerConnectionObserverJni 为Android端 PeerConnectionObserver 的实现类。
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_peerconnection_jni/PeerConnection_jni.h
// 该文件根据 PeerConnection.java 生成
JNI_GENERATOR_EXPORT jlong Java_org_webrtc_PeerConnection_nativeCreatePeerConnectionObserver(
    JNIEnv* env,
    jclass jcaller,
    jobject observer) {
  return JNI_PeerConnection_CreatePeerConnectionObserver(env,
      base::android::JavaParamRef<jobject>(env, observer));
}

// sdk/android/src/jni/pc/peer_connection.cc
static jlong JNI_PeerConnection_CreatePeerConnectionObserver(
    JNIEnv* jni,
    const JavaParamRef<jobject>& j_observer) {
  return jlongFromPointer(new PeerConnectionObserverJni(jni, j_observer));
}

PeerConnectionObserverJni::PeerConnectionObserverJni(
    JNIEnv* jni,
    const JavaRef<jobject>& j_observer)
    // 创建 Observer Object 的 global 引用
    : j_observer_global_(jni, j_observer) {}
```

创建 Native 层 PeerConnection 对象
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_peerconnection_jni/PeerConnection_factory_jni.h
// 该文件根据 sdk/android/api/org/webrtc/PeerConnectionFactory.java 在编译时动态生成
/* 创建 Native 层 PeerConnection 对象 */
JNI_GENERATOR_EXPORT jlong Java_org_webrtc_PeerConnectionFactory_nativeCreatePeerConnection(
    JNIEnv* env,
    jclass jcaller,
    jlong factory,
    jobject rtcConfig,
    jobject constraints, // null
    jlong nativeObserver,
    jobject sslCertificateVerifier) {
  return JNI_PeerConnectionFactory_CreatePeerConnection(env, factory,
      base::android::JavaParamRef<jobject>(env, rtcConfig),
      base::android::JavaParamRef<jobject>(env, constraints), nativeObserver,
      base::android::JavaParamRef<jobject>(env, sslCertificateVerifier));
}

// sdk/android/src/jni/pc/peer_connection_factory.cc
static jlong JNI_PeerConnectionFactory_CreatePeerConnection(
    JNIEnv* jni,
    jlong factory,
    const JavaParamRef<jobject>& j_rtc_config,
    const JavaParamRef<jobject>& j_constraints, /* null */
    jlong observer_p,
    const JavaParamRef<jobject>& j_sslCertificateVerifier /* null */) {
    /* PeerConnectionObserver */
    std::unique_ptr<PeerConnectionObserver> observer(
        reinterpret_cast<PeerConnectionObserver*>(observer_p));

    // RTCConfiguration 拷贝
    ...

    // rtc::RTCCertificate 生成
    ...

    // MediaConstraints 拷贝
    ...

    PeerConnectionDependencies peer_connection_dependencies(observer.get());

    // SSLCertificateVerifierWrapper 创建
    ...

    /* 创建 PeerConnection，该接口调用与各平台一致 */
    rtc::scoped_refptr<PeerConnectionInterface> pc =
        PeerConnectionFactoryFromJava(factory)->CreatePeerConnection(
            rtc_config, std::move(peer_connection_dependencies));
    if (!pc)
        return 0;

    /* 返回 PeerConnection 封装对象 */
    return jlongFromPointer(
        new OwnedPeerConnection(pc, std::move(observer), std::move(constraints)));
}

// pc/peer_connection_factory.cc
rtc::scoped_refptr<PeerConnectionInterface>
PeerConnectionFactory::CreatePeerConnection(
    const PeerConnectionInterface::RTCConfiguration& configuration,
    PeerConnectionDependencies dependencies) {
    
    ...
    // 创建 PeerConnection
    rtc::scoped_refptr<PeerConnection> pc(
        new rtc::RefCountedObject<PeerConnection>(this, std::move(event_log),
                                                    std::move(call)));
    ActionsBeforeInitializeForTesting(pc);
    // PeerConnection 初始化
    if (!pc->Initialize(configuration, std::move(dependencies))) {
        return nullptr;
    }
    return PeerConnectionProxy::Create(signaling_thread(), pc);
}
```

### 1.3 创建 VideoTrack
1. 创建视频渲染处理 SurfaceTextureHelper；
2. 创建 VideoSource;
3. 初始化视频采集源并启动，设置视频数据回调监听为 VideoSource 内实现的 CapturerObserver；
4. 创建 VideoTrack，并作为 VideoSink 注册到 VideoSource 内；
5. VideoTrack 使能视频渲染；
6. VideoTrack 添加本地预览Sink。

```java
// PeerConnectionClient.java
private VideoTrack createVideoTrack(VideoCapturer capturer) {
    // 视频源纹理数据渲染实例，经过该实例纹理处理后的数据通过 CapturerObserver 回调出来用于编码或预览(Camera2)
    surfaceTextureHelper =
        SurfaceTextureHelper.create("CaptureThread", rootEglBase.getEglBaseContext());
    
    // 创建 VideoSource
    videoSource = factory.createVideoSource(capturer.isScreencast());

    // 初始化及启动视频采集，摄像头的话，调用 CameraCapturer.initialize() 接口初始化，CapturerObserver 为 VideoSource 创建
    capturer.initialize(surfaceTextureHelper, appContext, videoSource.getCapturerObserver());

    capturer.startCapture(videoWidth, videoHeight, videoFps);

    // 创建 VideoTrack
    localVideoTrack = factory.createVideoTrack(VIDEO_TRACK_ID, videoSource);
    // 使能视频
    localVideoTrack.setEnabled(renderVideo);
    // 设置预览 Sink
    localVideoTrack.addSink(localRender);
    return localVideoTrack;
}

// sdk/android/api/org/webrtc/SurfaceTextureHelper.java
public static SurfaceTextureHelper create(
      final String threadName, final EglBase.Context sharedContext) {
    return create(threadName, sharedContext, /* alignTimestamps= */ false, new YuvConverter(),
        /*frameRefMonitor=*/null);
}

private SurfaceTextureHelper(Context sharedContext, Handler handler, boolean alignTimestamps,
      YuvConverter yuvConverter, FrameRefMonitor frameRefMonitor) {
    if (handler.getLooper().getThread() != Thread.currentThread()) {
        throw new IllegalStateException("SurfaceTextureHelper must be created on the handler thread");
    }
    this.handler = handler;
    this.timestampAligner = alignTimestamps ? new TimestampAligner() : null;
    this.yuvConverter = yuvConverter;
    this.frameRefMonitor = frameRefMonitor;

    eglBase = EglBase.create(sharedContext, EglBase.CONFIG_PIXEL_BUFFER);
    try {
        // Both these statements have been observed to fail on rare occasions, see BUG=webrtc:5682.
        eglBase.createDummyPbufferSurface();
        eglBase.makeCurrent();
    } catch (RuntimeException e) {
        // Clean up before rethrowing the exception.
        eglBase.release();
        handler.getLooper().quit();
        throw e;
    }

    oesTextureId = GlUtil.generateTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES);
    surfaceTexture = new SurfaceTexture(oesTextureId);
    setOnFrameAvailableListener(surfaceTexture, (SurfaceTexture st) -> {
        hasPendingTexture = true;
        tryDeliverTextureFrame();
    }, handler);
}
```

## 2 音频设备模块(AudioDeviceModule)

外部音频设备仅支持 java 层级，不支持 OpenSLES. 借由 JavaAudioDeviceModule 创建音频采集和音频播放实例，并通过 Jni 创建 native C++ 层 AudioDeviceModule 对象。

### 2.1 创建 JavaAudioDeviceModule
```java
// PeerConnectionClient.java
AudioDeviceModule createJavaAudioDevice() {
    /* 采样频率在 Builder 构造时从 WebRtcAudioManager 获取设置 */
    return JavaAudioDeviceModule.builder(appContext)
        /* 音频采集数据回调，用于保存到文件 */
        .setSamplesReadyCallback(saveRecordedAudioToFile)
        /* 是否开启系统回声消除，需设备支持 */
        .setUseHardwareAcousticEchoCanceler(!peerConnectionParameters.disableBuiltInAEC)
        /* 是否开启系统噪声抑制，需设备支持 */
        .setUseHardwareNoiseSuppressor(!peerConnectionParameters.disableBuiltInNS)
        /* 设置错误回调及状态回调 */
        .setAudioRecordErrorCallback(audioRecordErrorCallback)
        .setAudioTrackErrorCallback(audioTrackErrorCallback)
        .setAudioRecordStateCallback(audioRecordStateCallback)
        .setAudioTrackStateCallback(audioTrackStateCallback)
        /* 构建音频设备模块 */
        .createAudioDeviceModule();
}

// sdk/android/api/org/webrtc/JavaAudioDeviceModule.java
// JavaAudioDeviceModule.Builder
public AudioDeviceModule createAudioDeviceModule() {
    /* 创建音频采集实例 sdk/android/src/java/org/webrtc/audio/WebRtcAudioRecord.java */
    final WebRtcAudioRecord audioInput = new WebRtcAudioRecord(context, audioManager, audioSource,
        audioFormat, audioRecordErrorCallback, audioRecordStateCallback, samplesReadyCallback,
        useHardwareAcousticEchoCanceler, useHardwareNoiseSuppressor);
    
    /* 创建音频播放实例 sdk/android/src/java/org/webrtc/audio/WebRtcAudioTrack.java */
    final WebRtcAudioTrack audioOutput = new WebRtcAudioTrack(
        context, audioManager, audioTrackErrorCallback, audioTrackStateCallback);
    
    /* 创建音频设备模块 */
    return new JavaAudioDeviceModule(context, audioManager, audioInput, audioOutput,
        inputSampleRate, outputSampleRate, useStereoInput, useStereoOutput);
}

// JavaAudioDeviceModule 是 AudioDeviceModule 接口的派生类
private JavaAudioDeviceModule(Context context, AudioManager audioManager,
      WebRtcAudioRecord audioInput, WebRtcAudioTrack audioOutput, int inputSampleRate,
      int outputSampleRate, boolean useStereoInput, boolean useStereoOutput) {
    this.context = context;
    this.audioManager = audioManager;
    this.audioInput = audioInput;
    this.audioOutput = audioOutput;
    this.inputSampleRate = inputSampleRate;
    this.outputSampleRate = outputSampleRate;
    this.useStereoInput = useStereoInput;
    this.useStereoOutput = useStereoOutput;
}

/* 创建音频设备 Native 对象，在创建 PeerConnectionFactory 时调用 */
public long getNativeAudioDeviceModulePointer() {
    synchronized (nativeLock) {
        if (nativeAudioDeviceModule == 0) {
            nativeAudioDeviceModule = nativeCreateAudioDeviceModule(context, audioManager, audioInput,
                audioOutput, inputSampleRate, outputSampleRate, useStereoInput, useStereoOutput);
        }
        return nativeAudioDeviceModule;
    }
}
```

### 2.2 创建 Native 层 AudioDeviceModule
通过JNI 调用创建 AudioDeviceModule
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_java_audio_jni/JavaAudioDeviceModule_jni.h
// 此文件为编译过程中动态生成
JNI_GENERATOR_EXPORT jlong
    Java_org_webrtc_audio_JavaAudioDeviceModule_nativeCreateAudioDeviceModule(
    JNIEnv* env,
    jclass jcaller,
    jobject context,
    jobject audioManager,
    jobject audioInput,
    jobject audioOutput,
    jint inputSampleRate,
    jint outputSampleRate,
    jboolean useStereoInput,
    jboolean useStereoOutput) {
  return JNI_JavaAudioDeviceModule_CreateAudioDeviceModule(env,
        /* Java 层 Context 引用 */
        base::android::JavaParamRef<jobject>(env, context), 
        /* Java 层 AudioManager 引用*/
        base::android::JavaParamRef<jobject>(env, audioManager), 
        /* Java 层 WebRtcAudioRecord 引用 */
        base::android::JavaParamRef<jobject>(env, audioInput),
        /* Java 层 WebRtcAudioTrack 引用 */
        base::android::JavaParamRef<jobject>(env, audioOutput), 
        inputSampleRate, outputSampleRate,
        useStereoInput, useStereoOutput);
}

// sdk/android/src/jni/audio_device/java_audio_device_module.cc
static jlong JNI_JavaAudioDeviceModule_CreateAudioDeviceModule(
    JNIEnv* env,
    const JavaParamRef<jobject>& j_context,
    const JavaParamRef<jobject>& j_audio_manager,
    const JavaParamRef<jobject>& j_webrtc_audio_record,
    const JavaParamRef<jobject>& j_webrtc_audio_track,
    int input_sample_rate,
    int output_sample_rate,
    jboolean j_use_stereo_input,
    jboolean j_use_stereo_output) {
    AudioParameters input_parameters;
    AudioParameters output_parameters;
    /* 通过 AudioManager 获取音频采集及播放的相关参数 */
    GetAudioParameters(env, j_context, j_audio_manager, input_sample_rate,
                        output_sample_rate, j_use_stereo_input,
                        j_use_stereo_output, &input_parameters,
                        &output_parameters);

    /* 创建音频采集 AudioRecordJni 对象 */
    auto audio_input = std::make_unique<AudioRecordJni>(
        env, input_parameters, kHighLatencyModeDelayEstimateInMilliseconds,
        j_webrtc_audio_record);

    /* 创建音频播放 AudioTrackJni 对象 */
    auto audio_output = std::make_unique<AudioTrackJni>(env, output_parameters,
                                                        j_webrtc_audio_track);


    return jlongFromPointer(CreateAudioDeviceModuleFromInputAndOutput(
                                // AudioDeviceModule::AudioLayer
                                AudioDeviceModule::kAndroidJavaAudio,
                                // 是否双声道
                                j_use_stereo_input, j_use_stereo_output,
                                // 播放延时估计，150ms
                                kHighLatencyModeDelayEstimateInMilliseconds,
                                // 音频采集及播放对象
                                std::move(audio_input), std::move(audio_output))
                                .release());
}

// sdk/android/src/jni/audio_device/audio_device_module.cc
// 创建 AudioDeviceModule(modules/audio_device/include/audio_device.h)，
// AndroidAudioDeviceModule 继承于该类.
rtc::scoped_refptr<AudioDeviceModule> CreateAudioDeviceModuleFromInputAndOutput(
    AudioDeviceModule::AudioLayer audio_layer,
    bool is_stereo_playout_supported,
    bool is_stereo_record_supported,
    uint16_t playout_delay_ms,
    std::unique_ptr<AudioInput> audio_input,
    std::unique_ptr<AudioOutput> audio_output) {
  RTC_LOG(INFO) << __FUNCTION__;
  return new rtc::RefCountedObject<AndroidAudioDeviceModule>(
      audio_layer, is_stereo_playout_supported, is_stereo_record_supported,
      playout_delay_ms, std::move(audio_input), std::move(audio_output));
}

AndroidAudioDeviceModule(AudioDeviceModule::AudioLayer audio_layer,
                           bool is_stereo_playout_supported,
                           bool is_stereo_record_supported,
                           uint16_t playout_delay_ms,
                           std::unique_ptr<AudioInput> audio_input,
                           std::unique_ptr<AudioOutput> audio_output)
      : audio_layer_(audio_layer),
        is_stereo_playout_supported_(is_stereo_playout_supported),
        is_stereo_record_supported_(is_stereo_record_supported),
        playout_delay_ms_(playout_delay_ms),
        task_queue_factory_(CreateDefaultTaskQueueFactory()),
        input_(std::move(audio_input)),
        output_(std::move(audio_output)),
        initialized_(false) {
    RTC_CHECK(input_);
    RTC_CHECK(output_);
    RTC_LOG(INFO) << __FUNCTION__;
    thread_checker_.Detach();
}
```

### 2.3 Native 获取音频采集播放参数
```c++
// sdk/android/src/jni/audio_device/audio_device_module.cc
void GetAudioParameters(JNIEnv* env,
                        const JavaRef<jobject>& j_context,
                        const JavaRef<jobject>& j_audio_manager,
                        int input_sample_rate,
                        int output_sample_rate,
                        bool use_stereo_input,
                        bool use_stereo_output,
                        AudioParameters* input_parameters,
                        AudioParameters* output_parameters) {
  const int output_channels = use_stereo_output ? 2 : 1;
  const int input_channels = use_stereo_input ? 2 : 1;
  const size_t output_buffer_size = Java_WebRtcAudioManager_getOutputBufferSize(
      env, j_context, j_audio_manager, output_sample_rate, output_channels);
  const size_t input_buffer_size = Java_WebRtcAudioManager_getInputBufferSize(
      env, j_context, j_audio_manager, input_sample_rate, input_channels);
  output_parameters->reset(output_sample_rate,
                           static_cast<size_t>(output_channels),
                           static_cast<size_t>(output_buffer_size));
  input_parameters->reset(input_sample_rate,
                          static_cast<size_t>(input_channels),
                          static_cast<size_t>(input_buffer_size));
  RTC_CHECK(input_parameters->is_valid());
  RTC_CHECK(output_parameters->is_valid());
}

// out/android_arm64_Debug/gen/sdk/android/generated_audio_device_module_base_jni/WebRTCAudioManager_jni.h
// 该文件根据 sdk/android/src/java/org/webrt/audio/WebRtcAudioManager.java 编译时生成
// 获取 AudioTrack 的 getMinBufferSize
static jint Java_WebRtcAudioManager_getOutputBufferSize(JNIEnv* env, const
    base::android::JavaRef<jobject>& context,
    const base::android::JavaRef<jobject>& audioManager,
    JniIntWrapper sampleRate,
    JniIntWrapper numberOfOutputChannels) {
    jclass clazz = org_webrtc_audio_WebRtcAudioManager_clazz(env);
    CHECK_CLAZZ(env, clazz,
        org_webrtc_audio_WebRtcAudioManager_clazz(env), 0);

    jni_generator::JniJavaCallContextChecked call_context;
    call_context.Init<
        base::android::MethodID::TYPE_STATIC>(
            env,
            clazz,
            "getOutputBufferSize",
            "(Landroid/content/Context;Landroid/media/AudioManager;II)I",
            &g_org_webrtc_audio_WebRtcAudioManager_getOutputBufferSize);

    /* 调用 sdk/android/src/java/org/webrt/audio/WebRtcAudioManager.java 的 getOutputBufferSize() 方法 */
    jint ret =
        env->CallStaticIntMethod(clazz,
            call_context.base.method_id, context.obj(), audioManager.obj(), as_jint(sampleRate),
                as_jint(numberOfOutputChannels));
    return ret;
}

// 获取 AudioRecord 的 getMinBufferSize
static jint Java_WebRtcAudioManager_getInputBufferSize(JNIEnv* env, const
    base::android::JavaRef<jobject>& context,
    const base::android::JavaRef<jobject>& audioManager,
    JniIntWrapper sampleRate,
    JniIntWrapper numberOfInputChannels) {
    jclass clazz = org_webrtc_audio_WebRtcAudioManager_clazz(env);
    CHECK_CLAZZ(env, clazz,
        org_webrtc_audio_WebRtcAudioManager_clazz(env), 0);

    jni_generator::JniJavaCallContextChecked call_context;
    call_context.Init<
        base::android::MethodID::TYPE_STATIC>(
            env,
            clazz,
            "getInputBufferSize",
            "(Landroid/content/Context;Landroid/media/AudioManager;II)I",
            &g_org_webrtc_audio_WebRtcAudioManager_getInputBufferSize);

    /* 调用 sdk/android/src/java/org/webrt/audio/WebRtcAudioManager.java 的 getInputBufferSize() 方法 */
    jint ret =
        env->CallStaticIntMethod(clazz,
            call_context.base.method_id, context.obj(), audioManager.obj(), as_jint(sampleRate),
                as_jint(numberOfInputChannels));
    return ret;
}
```

### 2.4 音频采集 AudioRecordJni
Native 层音频采集对象，在构建时绑定 Java 层的 WebRtcAudioRecord 实例，以实现音频采集启动及读取音频采集数据进行编码发送。

```c++
AudioRecordJni::AudioRecordJni(JNIEnv* env,
                               const AudioParameters& audio_parameters,
                               int total_delay_ms,
                               const JavaRef<jobject>& j_audio_record)
    : j_audio_record_(env, j_audio_record), // 保存 Java 层音频采集 WebRtcAudioRecord 实例引用
      audio_parameters_(audio_parameters),
      total_delay_ms_(total_delay_ms),
      direct_buffer_address_(nullptr),
      direct_buffer_capacity_in_bytes_(0),
      frames_per_buffer_(0),
      initialized_(false),
      recording_(false),
      audio_device_buffer_(nullptr) {
    RTC_LOG(INFO) << "ctor";
    RTC_DCHECK(audio_parameters_.is_valid());

    /* 调用 Java层WebRtcAudioRecord的setNativeAudioRecord 方法设置 Native 对象，在启动音频采集和将采集数据回调C++时使用 */
    Java_WebRtcAudioRecord_setNativeAudioRecord(env, j_audio_record_,
                                                jni::jlongFromPointer(this));
    // Detach from this thread since construction is allowed to happen on a
    // different thread.
    thread_checker_.Detach();
    thread_checker_java_.Detach();
}

// out/android_arm64_Debug/gen/sdk/android/generated_java_audio_device_module_native_jni/WebRtcAudioRecord_jni.h
// 该文件根据 org/webrtc/audio/WebRtcAudioRecord.java 在编译时生成
static void Java_WebRtcAudioRecord_setNativeAudioRecord(JNIEnv* env, const
    base::android::JavaRef<jobject>& obj, jlong nativeAudioRecord) {
  jclass clazz = org_webrtc_audio_WebRtcAudioRecord_clazz(env);
  CHECK_CLAZZ(env, obj.obj(),
      org_webrtc_audio_WebRtcAudioRecord_clazz(env));

  jni_generator::JniJavaCallContextChecked call_context;
  call_context.Init<
      base::android::MethodID::TYPE_INSTANCE>(
          env,
          clazz,
          "setNativeAudioRecord",
          "(J)V",
          &g_org_webrtc_audio_WebRtcAudioRecord_setNativeAudioRecord);

     env->CallVoidMethod(obj.obj(),
          call_context.base.method_id, nativeAudioRecord);
}
```
Java 层调用
```java
// sdk/android/src/java/org/webrtc/audio/WebRtcAudioRecord.java
@CalledByNative
public void setNativeAudioRecord(long nativeAudioRecord) {
    this.nativeAudioRecord = nativeAudioRecord;
}
```

### 2.5 音频播放 AudioTrackJni
Native 层音频播放对象，在构建时绑定到 Java 层的 WebRtcAudioTrack 实例，实现音频播放初始化及音频播放数据读取。

```c++
AudioTrackJni::AudioTrackJni(JNIEnv* env,
                             const AudioParameters& audio_parameters,
                             const JavaRef<jobject>& j_webrtc_audio_track)
    : j_audio_track_(env, j_webrtc_audio_track), // 保存 Java 层音频采集 WebRtcAudioTrack 实例引用
      audio_parameters_(audio_parameters),
      direct_buffer_address_(nullptr),
      direct_buffer_capacity_in_bytes_(0),
      frames_per_buffer_(0),
      initialized_(false),
      playing_(false),
      audio_device_buffer_(nullptr) {
    RTC_LOG(INFO) << "ctor";
    RTC_DCHECK(audio_parameters_.is_valid());

    /* 调用 Java层 WebRtcAudioTrack 的 setNativeAudioTrac 方法设置 Native 对象，在启动音频播放和从C++层获取播放数据时使用 */
    Java_WebRtcAudioTrack_setNativeAudioTrack(env, j_audio_track_,
                                                jni::jlongFromPointer(this));
    // Detach from this thread since construction is allowed to happen on a
    // different thread.
    thread_checker_.Detach();
    thread_checker_java_.Detach();
}

// out/android_arm64_Debug/gen/sdk/android/generated_java_audio_device_module_native_jni/WebRtcAudioTrack_jni.h
// 该文件根据 org/webrtc/audio/WebRtcAudioTrack.java 在编译时生成
static void Java_WebRtcAudioTrack_setNativeAudioTrack(JNIEnv* env, const
    base::android::JavaRef<jobject>& obj, jlong nativeAudioTrack) {
  jclass clazz = org_webrtc_audio_WebRtcAudioTrack_clazz(env);
  CHECK_CLAZZ(env, obj.obj(),
      org_webrtc_audio_WebRtcAudioTrack_clazz(env));

  jni_generator::JniJavaCallContextChecked call_context;
  call_context.Init<
      base::android::MethodID::TYPE_INSTANCE>(
          env,
          clazz,
          "setNativeAudioTrack",
          "(J)V",
          &g_org_webrtc_audio_WebRtcAudioTrack_setNativeAudioTrack);

     env->CallVoidMethod(obj.obj(),
          call_context.base.method_id, nativeAudioTrack);
}
```
Java 层调用
```java
// sdk/android/src/java/org/webrtc/audio/WebRtcAudioTrack.java
@CalledByNative
public void setNativeAudioTrack(long nativeAudioTrack) {
    this.nativeAudioTrack = nativeAudioTrack;
}
```

## 3 视频图像采集模块(VideoCapturer)
视频采集源分为视频文件，屏幕共享及摄像头，这几类视频源统一继承了 VideoCapturer 类。其中摄像头采集源的通过 CameraEnumerator、CameraSession、CameraCapturer 连接不同 Camera API， 其中 Camera1Enumerator、Camera1Session， Camera1Capturer 连接 Camera V1 Api(即Android 5之前的摄像头接口) ；而 Camera2Enumerator、Camera2Session， Camera2Capturer 连接 Camera V2 接口。这两套接口封装了新旧 Camera Api的差异性，使外部调用保持一致。

```puml
@startuml

interface CapturerObserver {
    + void onCapturerStarted(boolean success)
    ..
    + void onCapturerStopped()
    ..
    + void onFrameCaptured(VideoFrame frame)
}

interface VideoCapturer {
    + void initialize(SurfaceTextureHelper , Context , CapturerObserver )
    ..
    + void startCapture(int width, int height, int framerate)
    ..
    + void stopCapture()
    ..
    + void changeCaptureFormat(int width, int height, int framerate)
    ..
    + void dispose()
    ..
    + boolean isScreencast()
}

interface CameraVideoCapturer {
    + void switchCamera(CameraSwitchHandler )
    ..
    + void switchCamera(CameraSwitchHandler, String cameraName)
}

abstract class CameraCapturer {
    # {abstract} void createCameraSession()
    ..
    - void createSessionInternal(int delayMs)
    ..
    - void switchCameraInternal()
    -- field --
    - {field} CameraEnumerator cameraEnumerator
    ..
    - {field} CameraSession.CreateSessionCallback createSessionCallback
    ..
    - {field} CameraSession.Events cameraSessionEventsHandler
    ..
    - {field} CapturerObserver capturerObserver
    ..
    - {field} CameraSession currentSession
    ..
    - {field} CameraSwitchHandler switchEventsHandler
    ..
    - {field} CameraStatistics cameraStatistics
}
CameraCapturer::capturerObserver *-- CapturerObserver
CameraCapturer::cameraEnumerator *-- CameraEnumerator
CameraCapturer::createSessionCallback *-- CameraSession.CreateSessionCallback
CameraCapturer::cameraSessionEventsHandler *-- CameraSession.Events
CameraCapturer::currentSession *-- CameraSession

class ScreenCapturerAndroid {

}

class FileVideoCapturer {

}

VideoCapturer <|-- CameraVideoCapturer
VideoCapturer <|-- ScreenCapturerAndroid
VideoCapturer <|-- FileVideoCapturer

CameraVideoCapturer <|-- CameraCapturer

class Camera1Capturer {
    + Camera1Capturer()
    ..
    # {method} void createCameraSession()
}

class Camera2Capturer {
    + Camera2Capturer()
    ..
    # {method} void createCameraSession()
}

CameraCapturer <|-- Camera1Capturer
CameraCapturer <|-- Camera2Capturer

interface CameraSession {
    + void stop()
    ..
    + {static} int getDeviceOrientation(Context context)
    ..
    + {static} VideoFrame.TextureBuffer createTextureBufferWithModifiedTransformMatrix(
      TextureBufferImpl buffer, boolean mirror, int rotation)
}

class Camera1Session {
    + {static} void create()
    ..
    - void startCapturing()
    ..
    - void listenForTextureFrames()
    ..
    - void listenForBytebufferFrames()
    ..
    - int getFrameOrientation()
}

class Camera2Session {
    + {static} void create()
    ..
    - void start()
    ..
    - int getFrameOrientation()
}
CameraSession <|-- Camera1Session
CameraSession <|-- Camera2Session

interface CameraSession.CreateSessionCallback {
    + void onDone(CameraSession session)
    ..
    + void onFailure(FailureType failureType, String error)
}

interface CameraSession.Events {
    + void onCameraOpening()
    ..
    + void onCameraError(CameraSession session, String error)
    ..
    + void onCameraDisconnected(CameraSession session)
    ..
    + void onCameraClosed(CameraSession session)
    ..
    + void onFrameCaptured(CameraSession session, VideoFrame frame)
}

interface CameraEnumerator {
    + String[] getDeviceNames()
    ..
    + boolean isFrontFacing(String deviceName)
    ..
    + boolean isBackFacing(String deviceName)
    ..
    + List<CaptureFormat> getSupportedFormats(String deviceName)
    ..
    + CameraVideoCapturer createCapturer(String deviceName, CameraVideoCapturer.CameraEventsHandler )
}

class Camera1Enumerator {
    - {static} CameraInfo getCameraInfo(int index)
    ..
    - {static} List<CaptureFormat> getSupportedFormats(int cameraId)
    ..
    - {static} String getDeviceName(int index)
}

class Camera2Enumerator {
    - {method} CameraCharacteristics getCameraCharacteristics(String deviceName)
    ..
    + {static} boolean isSupported(Context context)
}

CameraEnumerator <|-- Camera1Enumerator
CameraEnumerator <|-- Camera2Enumerator
@enduml
```

### 3.1 创建摄像头采集源(CameraVideoCapturer)

创建视频采集源， Demo 端调用入口。
```java
// CallActivity.java
private @Nullable VideoCapturer createVideoCapturer() {
    final VideoCapturer videoCapturer;
    String videoFileAsCamera = getIntent().getStringExtra(EXTRA_VIDEO_FILE_AS_CAMERA);
    /* 视频文件作为视频采集源 */
    if (videoFileAsCamera != null) {
        try {
            videoCapturer = new FileVideoCapturer(videoFileAsCamera);
        } catch (IOException e) {
            reportError("Failed to open video file for emulated camera");
            return null;
        }
    } else if (screencaptureEnabled) {
        /* 屏幕共享 */
        return createScreenCapturer();
    } else if (useCamera2()) {
        /* Camera2 Api */
        if (!captureToTexture()) {
            reportError(getString(R.string.camera2_texture_only_error));
            return null;
        }

        Logging.d(TAG, "Creating capturer using camera2 API.");
        videoCapturer = createCameraCapturer(new Camera2Enumerator(this));
    } else {
        /* Camera Api */
        Logging.d(TAG, "Creating capturer using camera1 API.");
        videoCapturer = createCameraCapturer(new Camera1Enumerator(captureToTexture()));
    }
    if (videoCapturer == null) {
        reportError("Failed to open camera");
        return null;
    }
    return videoCapturer;
}

// 创建摄像头采集源
private @Nullable VideoCapturer createCameraCapturer(CameraEnumerator enumerator) {
    /* 获取设备摄像头信息 */
    final String[] deviceNames = enumerator.getDeviceNames();

    // First, try to find front facing camera
    Logging.d(TAG, "Looking for front facing cameras.");
    // 创建前置摄像头
    for (String deviceName : deviceNames) {
        if (enumerator.isFrontFacing(deviceName)) {
            Logging.d(TAG, "Creating front facing camera capturer.");
            VideoCapturer videoCapturer = enumerator.createCapturer(deviceName, /* CameraEventsHandler */null);

            if (videoCapturer != null) {
                return videoCapturer;
            }
        }
    }

    // Front facing camera not found, try something else
    Logging.d(TAG, "Looking for other cameras.");
    // 若无前置摄像头，则创建后置摄像头
    for (String deviceName : deviceNames) {
        if (!enumerator.isFrontFacing(deviceName)) {
            Logging.d(TAG, "Creating other camera capturer.");
            VideoCapturer videoCapturer = enumerator.createCapturer(deviceName, /* CameraEventsHandler */null);

            if (videoCapturer != null) {
                return videoCapturer;
            }
        }
    }

    return null;
}
```

#### 3.1.1 摄像头 Camera1 采集(Camera1Capturer)
1. 创建 Camera1Enumerator；
2. 通过 Camera1Enumerator 接口 createCapturer() 创建 Camera1Capturer

```java
// sdk/android/api/org/webrtc/Camera1Enumerator.java
public Camera1Enumerator(boolean captureToTexture) {
    this.captureToTexture = captureToTexture;
}

@Override
public CameraVideoCapturer createCapturer(
    String deviceName, CameraVideoCapturer.CameraEventsHandler eventsHandler) {
    return new Camera1Capturer(deviceName, eventsHandler, captureToTexture);
}
```

3. Camera1Capturer 构造函数调用父类 CameraCapturer 构造函数，保存相关参数等操作
```java
// sdk/android/api/org/webrtc/Camera1Capturer.java
public Camera1Capturer( String cameraName, CameraEventsHandler eventsHandler, boolean captureToTexture) {
    /* 调用父类构造函数，注意此处重建了 Camera1Enumerator  */
    super(cameraName, eventsHandler, new Camera1Enumerator(captureToTexture));

    this.captureToTexture = captureToTexture;
}

// sdk/android/src/java/org/webrt/CameraCapturer.java
public CameraCapturer(String cameraName, @Nullable CameraEventsHandler eventsHandler,
      CameraEnumerator cameraEnumerator) {
    // 此处 eventsHandler 为 null
    if (eventsHandler == null) {
        eventsHandler = new CameraEventsHandler() {
            @Override
            public void onCameraError(String errorDescription) {}
            @Override
            public void onCameraDisconnected() {}
            @Override
            public void onCameraFreezed(String errorDescription) {}
            @Override
            public void onCameraOpening(String cameraName) {}
            @Override
            public void onFirstFrameAvailable() {}
            @Override
            public void onCameraClosed() {}
        };
    }

    this.eventsHandler = eventsHandler;
    this.cameraEnumerator = cameraEnumerator;
    this.cameraName = cameraName;
    List<String> deviceNames = Arrays.asList(cameraEnumerator.getDeviceNames());
    uiThreadHandler = new Handler(Looper.getMainLooper());
}
```

#### 3.1.2 摄像头 Camera2 采集(Camera2Capturer)
1. 创建 Camera2Enumerator；
2. 通过 Camera2Enumerator 接口 createCapturer() 创建 Camera2Capturer
```java
// sdk/android/api/org/webrtc/Camera2Enumerator.java
public Camera2Enumerator(Context context) {
    this.context = context;
    /* 获取系统摄像头管理服务 */
    this.cameraManager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
}

@Override
public CameraVideoCapturer createCapturer(String deviceName, CameraVideoCapturer.CameraEventsHandler eventsHandler) {
    return new Camera2Capturer(context, deviceName, eventsHandler);
}
```

3. Camera2Capturer 构造函数调用父类 CameraCapturer 构造函数，保存相关参数等操作
```java
// sdk/android/api/org/webrtc/Camera1Capturer.java
public Camera2Capturer(Context context, String cameraName, CameraEventsHandler eventsHandler) {
    /* 调用父类构造函数，注意此处新建了 Camera2Enumerator  */
    super(cameraName, eventsHandler, new Camera2Enumerator(context));

    this.context = context;
    /* 获取系统摄像头管理服务 */
    cameraManager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
}

// sdk/android/src/java/org/webrt/CameraCapturer.java
public CameraCapturer(String cameraName, @Nullable CameraEventsHandler eventsHandler,
      CameraEnumerator cameraEnumerator) {
    // 此处 eventsHandler 为 null
    if (eventsHandler == null) {
        eventsHandler = new CameraEventsHandler() {
            @Override
            public void onCameraError(String errorDescription) {}
            @Override
            public void onCameraDisconnected() {}
            @Override
            public void onCameraFreezed(String errorDescription) {}
            @Override
            public void onCameraOpening(String cameraName) {}
            @Override
            public void onFirstFrameAvailable() {}
            @Override
            public void onCameraClosed() {}
        };
    }

    this.eventsHandler = eventsHandler;
    this.cameraEnumerator = cameraEnumerator;
    this.cameraName = cameraName;
    List<String> deviceNames = Arrays.asList(cameraEnumerator.getDeviceNames());
    uiThreadHandler = new Handler(Looper.getMainLooper());
}
```

### 3.2 创建 VideoSource
创建 Java 层 VideoSource，由 PeerConnectionClient.createVideoTrack() 内调用。

```java
// sdk/android/api/org/webrtc/PeerConnectionFactory.java
public VideoSource createVideoSource(boolean isScreencast) {
    return createVideoSource(isScreencast, /* alignTimestamps= */ true);
}

public VideoSource createVideoSource(boolean isScreencast, boolean alignTimestamps) {
    checkPeerConnectionFactoryExists();
    // 需先创建 Native 层 VideoSource，进行绑定
    return new VideoSource(nativeCreateVideoSource(nativeFactory, isScreencast, alignTimestamps));
}

// sdk/android/api/org/webrtc/VideoSource.java
public VideoSource(long nativeSource) {
    // 调用父类 MediaSource 构造
    super(nativeSource);
    // 创建 AndroidVideoTrackSource 的 java 层映射
    this.nativeAndroidVideoTrackSource = new NativeAndroidVideoTrackSource(nativeSource);
}

// // sdk/android/src/java/org/webrtc/NativeAndroidVideoTrackSource.java
public NativeAndroidVideoTrackSource(long nativeAndroidVideoTrackSource) {
    this.nativeAndroidVideoTrackSource = nativeAndroidVideoTrackSource;
}

// sdk/android/api/org/webrtc/MediaSource.java
public MediaSource(long nativeSource) {
    refCountDelegate = new RefCountDelegate(() -> JniCommon.nativeReleaseRef(nativeSource));
    this.nativeSource = nativeSource;
}
```

创建 VideoSource 对应 Native 层 AndroidVideoTrackSource。
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_peerconnection_jni/PeerConnectionFactory_jni.h
JNI_GENERATOR_EXPORT jlong Java_org_webrtc_PeerConnectionFactory_nativeCreateVideoSource(
    JNIEnv* env,
    jclass jcaller,
    jlong factory,
    jboolean is_screencast,
    jboolean alignTimestamps) {
  return JNI_PeerConnectionFactory_CreateVideoSource(env, factory, is_screencast, alignTimestamps);
}

// sdk/android/src/jni/pc/peer_connection_factory.cc
static jlong JNI_PeerConnectionFactory_CreateVideoSource(
    JNIEnv* jni,
    jlong native_factory,
    jboolean is_screencast,
    jboolean align_timestamps) {
  OwnedFactoryAndThreads* factory =
      reinterpret_cast<OwnedFactoryAndThreads*>(native_factory);
  return jlongFromPointer(CreateVideoSource(jni, factory->signaling_thread(),
                                            factory->worker_thread(),
                                            is_screencast, align_timestamps));
}

// sdk/android/src/jni/pc/video.cc
void* CreateVideoSource(JNIEnv* env,
                        rtc::Thread* signaling_thread,
                        rtc::Thread* worker_thread,
                        jboolean is_screencast,
                        jboolean align_timestamps) {
    rtc::scoped_refptr<AndroidVideoTrackSource> source(
        new rtc::RefCountedObject<AndroidVideoTrackSource>(
            signaling_thread, env, is_screencast, align_timestamps));
    return source.release();
}

// sdk/android/src/jni/android_video_track_source.cc
// AndroidVideoTrackSource 继承于 AdaptedVideoTrackSource, AdaptedVideoTrackSource 又继承于 VideoTrackSourceInterface
// AdaptedVideoTrackSource 实现了对视频裁剪，角度旋转，丢帧等处理
AndroidVideoTrackSource::AndroidVideoTrackSource(rtc::Thread* signaling_thread,
                                                 JNIEnv* jni,
                                                 bool is_screencast,
                                                 bool align_timestamps)
    : AdaptedVideoTrackSource(kRequiredResolutionAlignment),
      signaling_thread_(signaling_thread),
      is_screencast_(is_screencast),
      align_timestamps_(align_timestamps) {
    RTC_LOG(LS_INFO) << "AndroidVideoTrackSource ctor";
}
```

### 3.3 摄像头采集初始化及启动

```java
// sdk/android/src/java/org/webrtc/CameraCapturer.java
@Override
public void initialize(SurfaceTextureHelper surfaceTextureHelper, Context applicationContext,
      org.webrtc.CapturerObserver capturerObserver) {
    this.applicationContext = applicationContext;
    // capturerObserver 视频数据回调接口
    this.capturerObserver = capturerObserver;
    this.surfaceHelper = surfaceTextureHelper;
    this.cameraThreadHandler = surfaceTextureHelper.getHandler();
}

@Override
public void startCapture(int width, int height, int framerate) {
    Logging.d(TAG, "startCapture: " + width + "x" + height + "@" + framerate);
    if (applicationContext == null) {
        throw new RuntimeException("CameraCapturer must be initialized before calling startCapture.");
    }

    synchronized (stateLock) {
        if (sessionOpening || currentSession != null) {
        Logging.w(TAG, "Session already open");
        return;
        }

        this.width = width;
        this.height = height;
        this.framerate = framerate;

        sessionOpening = true;
        // 摄像头启动尝试次数
        openAttemptsRemaining = MAX_OPEN_CAMERA_ATTEMPTS;
        createSessionInternal(0);
    }
}

private void createSessionInternal(int delayMs) {
    // 启动摄像头启动超时定时器，以便重启摄像头
    uiThreadHandler.postDelayed(openCameraTimeoutRunnable, delayMs + OPEN_CAMERA_TIMEOUT);
    cameraThreadHandler.postDelayed(new Runnable() {
        @Override
        public void run() {
            // 调用子类 Camera1Capturer 或 Camera2Capturer 的 createCameraSession() 实现
            createCameraSession(createSessionCallback, cameraSessionEventsHandler, applicationContext,
                surfaceHelper, cameraName, width, height, framerate);
        }
    }, delayMs);
}
```

#### 3.3.1 启动 Camera1 采集
在调用 CameraCapturer 的 startCapture() 时启动，先创建 Camera1Session，并在 Camera1Session 内启动摄像头。

1. 创建 Camera1Session，并打开摄像头，设置摄像头参数。
```java
// sdk/android/api/org/webrtc/Camera1Capturer.java
@Override
protected void createCameraSession(CameraSession.CreateSessionCallback createSessionCallback,
    CameraSession.Events events, Context applicationContext,
    SurfaceTextureHelper surfaceTextureHelper, String cameraName, int width, int height,
    int framerate) {
    // 创建 Camera1Session 并启动摄像头
    Camera1Session.create(createSessionCallback, events, captureToTexture, applicationContext,
        surfaceTextureHelper, Camera1Enumerator.getCameraIndex(cameraName), width, height,
        framerate);
}

// sdk/android/src/java/org/webrtc/Camera1Session.java
public static void create(final CreateSessionCallback callback, final Events events,
      final boolean captureToTexture, final Context applicationContext,
      final SurfaceTextureHelper surfaceTextureHelper, final int cameraId, final int width,
      final int height, final int framerate) {
    final long constructionTimeNs = System.nanoTime();
    Logging.d(TAG, "Open camera " + cameraId);
    // 状态上报
    events.onCameraOpening();

    final android.hardware.Camera camera;
    try {
        // 打开摄像头
        camera = android.hardware.Camera.open(cameraId);
    } catch (RuntimeException e) {
        // 摄像头打开失败，回调触发重启
        callback.onFailure(FailureType.ERROR, e.getMessage());
        return;
    }

    if (camera == null) {
        // 摄像头打开失败，回调触发重启
        callback.onFailure(FailureType.ERROR,
            "android.hardware.Camera.open returned null for camera id = " + cameraId);
        return;
    }

    try {
        // 设置摄像头预览
        camera.setPreviewTexture(surfaceTextureHelper.getSurfaceTexture());
    } catch (IOException | RuntimeException e) {
        camera.release();
        // 摄像头预览设置失败，回调触发重启
        callback.onFailure(FailureType.ERROR, e.getMessage());
        return;
    }

    // 摄像头 参数设置
    ...

    // 若以非纹理方式捕捉，则通过 Camera.PreviewCallback 回调采集数据，需设置缓冲区
    if (!captureToTexture) {
        final int frameSize = captureFormat.frameSize();
        for (int i = 0; i < NUMBER_OF_CAPTURE_BUFFERS; ++i) {
            final ByteBuffer buffer = ByteBuffer.allocateDirect(frameSize);
            camera.addCallbackBuffer(buffer.array());
        }
    }

    // Calculate orientation manually and send it as CVO insted.
    camera.setDisplayOrientation(0 /* degrees */);

    callback.onDone(new Camera1Session(events, captureToTexture, applicationContext,
        surfaceTextureHelper, cameraId, camera, info, captureFormat, constructionTimeNs));
}

private Camera1Session(Events events, boolean captureToTexture, Context applicationContext,
      SurfaceTextureHelper surfaceTextureHelper, int cameraId, android.hardware.Camera camera,
      android.hardware.Camera.CameraInfo info, CaptureFormat captureFormat,
      long constructionTimeNs) {
    Logging.d(TAG, "Create new camera1 session on camera " + cameraId);

    this.cameraThreadHandler = new Handler();
    this.events = events;
    this.captureToTexture = captureToTexture;
    this.applicationContext = applicationContext;
    this.surfaceTextureHelper = surfaceTextureHelper;
    this.cameraId = cameraId;
    this.camera = camera;
    this.info = info;
    this.captureFormat = captureFormat;
    this.constructionTimeNs = constructionTimeNs;

    // 设置采集数据输出分辨率
    surfaceTextureHelper.setTextureSize(captureFormat.width, captureFormat.height);

    // 开始摄像头采集
    startCapturing();
}
```

2. 启动摄像头采集，设置视频采集数据回调。
```java
// sdk/android/src/java/org/webrtc/Camera1Session.java
private void startCapturing() {
    Logging.d(TAG, "Start capturing");
    checkIsOnCameraThread();

    state = SessionState.RUNNING;

    camera.setErrorCallback(new android.hardware.Camera.ErrorCallback() {
        @Override
        public void onError(int error, android.hardware.Camera camera) {
            String errorMessage;
            if (error == android.hardware.Camera.CAMERA_ERROR_SERVER_DIED) {
                errorMessage = "Camera server died!";
            } else {
                errorMessage = "Camera error: " + error;
            }
            Logging.e(TAG, errorMessage);
            stopInternal();
            if (error == android.hardware.Camera.CAMERA_ERROR_EVICTED) {
                events.onCameraDisconnected(Camera1Session.this);
            } else {
                events.onCameraError(Camera1Session.this, errorMessage);
            }
        }
    });

    // 设置采集数据监听
    if (captureToTexture) {
        listenForTextureFrames();
    } else {
        listenForBytebufferFrames();
    }
    try {
        // 正式启动摄像头
        camera.startPreview();
    } catch (RuntimeException e) {
        stopInternal();
        events.onCameraError(this, e.getMessage());
    }
}
```

#### 3.3.2 启动 Camera2 采集
在调用 CameraCapturer 的 startCapture() 时启动，先创建 Camera2Session，并在 Camera2Session 内启动摄像头。

1. 创建 Camera2Session
```java
// sdk/android/api/org/webrtc/Camera2Capturer.java
@Override
protected void createCameraSession(CameraSession.CreateSessionCallback createSessionCallback,
      CameraSession.Events events, Context applicationContext,
      SurfaceTextureHelper surfaceTextureHelper, String cameraName, int width, int height,
      int framerate) {
    Camera2Session.create(createSessionCallback, events, applicationContext, cameraManager,
        surfaceTextureHelper, cameraName, width, height, framerate);
}

// sdk/android/src/java/org/webrtc/Camera2Session.java
public static void create(CreateSessionCallback callback, Events events,
      Context applicationContext, CameraManager cameraManager,
      SurfaceTextureHelper surfaceTextureHelper, String cameraId, int width, int height,
      int framerate) {
    new Camera2Session(callback, events, applicationContext, cameraManager, surfaceTextureHelper,
        cameraId, width, height, framerate);
}

private Camera2Session(CreateSessionCallback callback, Events events, Context applicationContext,
      CameraManager cameraManager, SurfaceTextureHelper surfaceTextureHelper, String cameraId,
      int width, int height, int framerate) {
    Logging.d(TAG, "Create new camera2 session on camera " + cameraId);

    constructionTimeNs = System.nanoTime();

    this.cameraThreadHandler = new Handler();
    this.callback = callback;
    this.events = events;
    this.applicationContext = applicationContext;
    this.cameraManager = cameraManager;
    this.surfaceTextureHelper = surfaceTextureHelper;
    this.cameraId = cameraId;
    this.width = width;
    this.height = height;
    this.framerate = framerate;

    start();
}
```

2. 打开摄像头
```java
private void start() {
    checkIsOnCameraThread();
    Logging.d(TAG, "start");

    try {
        cameraCharacteristics = cameraManager.getCameraCharacteristics(cameraId);
    } catch (final CameraAccessException e) {
        reportError("getCameraCharacteristics(): " + e.getMessage());
        return;
    }
    cameraOrientation = cameraCharacteristics.get(CameraCharacteristics.SENSOR_ORIENTATION);
    isCameraFrontFacing = cameraCharacteristics.get(CameraCharacteristics.LENS_FACING)
        == CameraMetadata.LENS_FACING_FRONT;

    findCaptureFormat();
    openCamera();
}

private void openCamera() {
    checkIsOnCameraThread();

    Logging.d(TAG, "Opening camera " + cameraId);
    events.onCameraOpening();

    try {
        // 设置摄像头状态回调，打开摄像头
        cameraManager.openCamera(cameraId, new CameraStateCallback(), cameraThreadHandler);
    } catch (CameraAccessException e) {
        reportError("Failed to open camera: " + e);
        return;
    }
}
```

### 3.4 创建 VideoTrack
调用入口为 PeerConnectionClient.createVideoTrack()， 详见 [1.3 创建 VideoTrack](#13-创建-videotrack)。
在创建 VideoSource 及打开摄像头之后，调用 PeerConnectionFactory.createVideoTrack()。

#### 3.4.1 创建 VideoTrack

localVideoTrack = factory.createVideoTrack(VIDEO_TRACK_ID, videoSource);

```java
// sdk/android/api/org/webrtc/PeerConnectionFactory.java
// id: "ARDAMSv0"
public VideoTrack createVideoTrack(String id, VideoSource source) {
    checkPeerConnectionFactoryExists();
    return new VideoTrack(
        nativeCreateVideoTrack(nativeFactory, id, source.getNativeVideoTrackSource()/* Native VideoSource 指针地址 */));
}

// sdk/android/api/org/webrtc/VideoTrack.java
public VideoTrack(long nativeTrack) {
    // 调用父类 MediaStreamTrack 构造函数，保存 native 指针
    super(nativeTrack);
}
```
创建 Native 层 VideoTrack
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_peerconnection_jni/PeerConnectionFactory_jni.h
JNI_GENERATOR_EXPORT jlong Java_org_webrtc_PeerConnectionFactory_nativeCreateVideoTrack(
    JNIEnv* env,
    jclass jcaller,
    jlong factory,
    jstring id,
    jlong nativeVideoSource) {
    return JNI_PeerConnectionFactory_CreateVideoTrack(env, factory,
        base::android::JavaParamRef<jstring>(env, id), nativeVideoSource);
}

// sdk/android/src/jni/pc/peer_connection_factory.cc
static jlong JNI_PeerConnectionFactory_CreateVideoTrack(
    JNIEnv* jni,
    jlong native_factory,
    const JavaParamRef<jstring>& id,
    jlong native_source) {
    rtc::scoped_refptr<VideoTrackInterface> track =
        PeerConnectionFactoryFromJava(native_factory)
            ->CreateVideoTrack(
                JavaToStdString(jni, id),
                reinterpret_cast<VideoTrackSourceInterface*>(native_source));
    return jlongFromPointer(track.release());
}

// pc/peer_connection_factory.cc
rtc::scoped_refptr<VideoTrackInterface> PeerConnectionFactory::CreateVideoTrack(
    const std::string& id,
    VideoTrackSourceInterface* source) {
  RTC_DCHECK(signaling_thread_->IsCurrent());
  rtc::scoped_refptr<VideoTrackInterface> track(
      VideoTrack::Create(id, source, worker_thread_));
  return VideoTrackProxy::Create(signaling_thread_, worker_thread_, track);
}

// pc/video_track.cc
rtc::scoped_refptr<VideoTrack> VideoTrack::Create(
    const std::string& id,
    VideoTrackSourceInterface* source,
    rtc::Thread* worker_thread) {
    // 构造 VideoTrack
    rtc::RefCountedObject<VideoTrack>* track =
        new rtc::RefCountedObject<VideoTrack>(id, source, worker_thread);
    return track;
}

VideoTrack::VideoTrack(const std::string& label,
                       VideoTrackSourceInterface* video_source,
                       rtc::Thread* worker_thread)
    : MediaStreamTrack<VideoTrackInterface>(label),
      worker_thread_(worker_thread),
      // 保存 VideoSource，在 set_enabled() 的时候添加 sink。
      video_source_(video_source),
      content_hint_(ContentHint::kNone) {
    
    // 注册 VideoSource 观察者
    video_source_->RegisterObserver(this);
}

// api/notifier.h
// AndroidVideoTrackSource 继承于 AdaptedVideoTrackSource，
// AdaptedVideoTraceSource 继承于 webrtc::Notifier<webrtc::VideoTrackSourceInterface>
virtual void RegisterObserver(ObserverInterface* observer) {
    RTC_DCHECK(observer != nullptr);
    observers_.push_back(observer);
}
```

#### 3.4.2 使能 VideoTrack

localVideoTrack.setEnabled(renderVideo);

```java
// sdk/android/api/org/webrtc/MediaStreamTrack.java
// VideoTrack 是 MediaStreamTrack 的扩展类，setEnabled() 的实现位于 MediaStreamTrack
public boolean setEnabled(boolean enable) {
    checkMediaStreamTrackExists();
    // 调用 Native 层 VideoTrack 函数
    return nativeSetEnabled(nativeTrack, enable);
}
```
转入 Native 层调用
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_peerconnection_jni/MediaStreamTrack_jni.h
JNI_GENERATOR_EXPORT jboolean Java_org_webrtc_MediaStreamTrack_nativeSetEnabled(
    JNIEnv* env,
    jclass jcaller,
    jlong track,
    jboolean enabled) {
  return JNI_MediaStreamTrack_SetEnabled(env, track, enabled);
}

// sdk/android/src/jni/pc/media_stream_track.cc
static jboolean JNI_MediaStreamTrack_SetEnabled(JNIEnv* jni,
                                                jlong j_p,
                                                jboolean enabled) {
    return reinterpret_cast<MediaStreamTrackInterface*>(j_p)->set_enabled(enabled);
}

// pc/video_track.cc
bool VideoTrack::set_enabled(bool enable) {
    RTC_DCHECK(signaling_thread_checker_.IsCurrent());
    worker_thread_->Invoke<void>(RTC_FROM_HERE, [enable, this] {
        RTC_DCHECK(worker_thread_->IsCurrent());
        for (auto& sink_pair : sink_pairs()) {
            rtc::VideoSinkWants modified_wants = sink_pair.wants;
            modified_wants.black_frames = !enable;
            video_source_->AddOrUpdateSink(sink_pair.sink, modified_wants);
        }
    });

    // 调用基类函数
    return MediaStreamTrack<VideoTrackInterface>::set_enabled(enable);
}

// pc/media_stream_track.h
bool set_enabled(bool enable) override {
    bool fire_on_change = (enable != enabled_);
    // 保存使能状态
    enabled_ = enable;
    if (fire_on_change) {
        // 此处 T 为 VideoTrackInterface
        Notifier<T>::FireOnChanged();
    }
    return fire_on_change;
}
```

#### 3.4.3 添加渲染 Sink
添加预览的 VideoSink 到 VideoTrack。

localVideoTrack.addSink(localRender);
```java
// sdk/android/api/org/webrtc/VideoTrack.java
public void addSink(VideoSink sink) {
    if (sink == null) {
        throw new IllegalArgumentException("The VideoSink is not allowed to be null");
    }
    // We allow calling addSink() with the same sink multiple times. This is similar to the C++
    // VideoTrack::AddOrUpdateSink().
    if (!sinks.containsKey(sink)) {
        /* Java 层 VideoSink 与 Native 层 VideoSink 的映射 */
        final long nativeSink = nativeWrapSink(sink);
        /* 保存映射关系，在 删除Sink时，释放 Native 层的 VideoSink */
        sinks.put(sink, nativeSink);
        nativeAddSink(getNativeMediaStreamTrack(), nativeSink);
    }
}
```

1. 映射 Java 层 VideoSink 到 Native 层 VideoSink（VideoSinkWrapper）。
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_video_jni/VideoTrack_jni.h
JNI_GENERATOR_EXPORT jlong Java_org_webrtc_VideoTrack_nativeWrapSink(
    JNIEnv* env,
    jclass jcaller,
    jobject sink) {
    return JNI_VideoTrack_WrapSink(env, base::android::JavaParamRef<jobject>(env, sink));
}

// sdk/android/src/jni/video_track.cc
static jlong JNI_VideoTrack_WrapSink(JNIEnv* jni,
                                     const JavaParamRef<jobject>& sink) {
  return jlongFromPointer(new VideoSinkWrapper(jni, sink));
}

// sdk/android/src/jni/video_sink.cc
// 保存 Java 层 VideoSink 实例，后续视频帧通过该实例回调给 Java 层。
// VideoSinkWrapper 父类是 rtc::VideoSinkInterface<VideoFrame>，重写了 OnFrame(VideoFrame)
VideoSinkWrapper::VideoSinkWrapper(JNIEnv* jni, const JavaRef<jobject>& j_sink)
    : j_sink_(jni, j_sink) {}
```

2. 添加 VideoSink 到 VideoTrack 上
```c++
// out/android_arm64_Debug/gen/sdk/android/generated_video_jni/VideoTrack_jni.h
JNI_GENERATOR_EXPORT void Java_org_webrtc_VideoTrack_nativeAddSink(
    JNIEnv* env,
    jclass jcaller,
    jlong track,
    jlong nativeSink) {
    return JNI_VideoTrack_AddSink(env, track, nativeSink);
}

// sdk/android/src/jni/video_track.cc
static void JNI_VideoTrack_AddSink(JNIEnv* jni,
                                   jlong j_native_track,
                                   jlong j_native_sink) {
    reinterpret_cast<VideoTrackInterface*>(j_native_track)
        ->AddOrUpdateSink(
            reinterpret_cast<rtc::VideoSinkInterface<VideoFrame>*>(j_native_sink),
            rtc::VideoSinkWants());
}

// pc/video_track.cc
void VideoTrack::AddOrUpdateSink(rtc::VideoSinkInterface<VideoFrame>* sink,
                                 const rtc::VideoSinkWants& wants) {
    RTC_DCHECK(worker_thread_->IsCurrent());
    /* 基类保存 Sink */
    VideoSourceBase::AddOrUpdateSink(sink, wants);
    rtc::VideoSinkWants modified_wants = wants;

    // 是否已使能 VideoTrack，未使能的话用黑屏帧代替，因之前已设置 enable，所以此处 black_frames 为false
    modified_wants.black_frames = !enabled();
    // VideoSource 添加回调 Sink，即调用 AdaptedVideoTrackSource::AddOrUpdateSink(）
    video_source_->AddOrUpdateSink(sink, modified_wants);
}

// media/base/video_source_base.cc
void VideoSourceBase::AddOrUpdateSink(
    VideoSinkInterface<webrtc::VideoFrame>* sink,
    const VideoSinkWants& wants) {
    RTC_DCHECK(sink != nullptr);

    SinkPair* sink_pair = FindSinkPair(sink);
    if (!sink_pair) {
        sinks_.push_back(SinkPair(sink, wants));
    } else {
        sink_pair->wants = wants;
    }
}

// 添加 VideoSink 到 VideoSource
// AndroidVideoTrackSource 继承于 AdaptedVideoTrackSource,
void AdaptedVideoTrackSource::AddOrUpdateSink(
    rtc::VideoSinkInterface<webrtc::VideoFrame>* sink,
    const rtc::VideoSinkWants& wants) {
    broadcaster_.AddOrUpdateSink(sink, wants);
    OnSinkWantsChanged(broadcaster_.wants());
}
```

#### 3.4.4 添加 VideoTrack
添加 VideoTrack 到 PeerConnection

```java

```

### 3.5 摄像头视频数据流
```
(Camera2 或 Camera1 使能 captureToTexture) (SurfaceTextureHelper.java) SurfaceTextureHelper.listener.onFrame() -->
Camera1Session/Camera2Session --> (CameraCapturer.java) CameraSession.Events.onFrameCaptured() 
--> (VideoSource.java) CapturerObserver.onFrameCaptured()
--> (NativeAndroidVideoTrackSource.java) NativeAndroidVideoTrackSource.onFrameCaptured() 
--> (android_video_track_source.cc) AndroidVideoTrackSource.onFrameCaptured()
--> (adapted_video_track_source.cc) AdaptedVideoTrackSource::OnFrame() --> broadcaster_.OnFrame(frame)
// 视频预览
--> (sdk/android/src/jni/video_sink.cc) VideoSinkWrapper::::OnFrame(VideoFrame)
--> (gen/sdk/android/generated_video_jni/VideoSink_jni.h) Java_VideoSink_onFrame(VideoFrame) // 映射到 Java 层 VideoFrame
--> (CallActivity.java) ProxyVideoSink::onFrame() --> target.onFrame(VideoFrame)
--> (SurfaceViewRenderer.java) SurfaceViewRenderer::onFrame(VideoFrame)
```