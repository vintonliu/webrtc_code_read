## 1 PeerConnection 创建

音视频互通需要先创建 PeerConnection，然后通过创建及交换 Sdp，以完成通话建立。

### 1.1 PeerConnectionFactory 创建

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
```java
// PeerConnectionClient.java
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
视频采集源分为视频文件，屏幕共享及摄像头，这几类统一继承了 VideoCapturer 类。

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


class ScreenCapturerAndroid {

}

class FileVideoCapturer {

}

VideoCapturer <|-- CameraVideoCapturer
VideoCapturer <|-- ScreenCapturerAndroid
VideoCapturer <|-- FileVideoCapturer

CameraVideoCapturer <|-- CameraCapturer

interface CameraSession {
    
}

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

private @Nullable VideoCapturer createCameraCapturer(CameraEnumerator enumerator) {
    /* 获取设备摄像头信息 */
    final String[] deviceNames = enumerator.getDeviceNames();

    // First, try to find front facing camera
    Logging.d(TAG, "Looking for front facing cameras.");
    // 创建前置摄像头
    for (String deviceName : deviceNames) {
        if (enumerator.isFrontFacing(deviceName)) {
            Logging.d(TAG, "Creating front facing camera capturer.");
            VideoCapturer videoCapturer = enumerator.createCapturer(deviceName, null);

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
            VideoCapturer videoCapturer = enumerator.createCapturer(deviceName, null);

            if (videoCapturer != null) {
                return videoCapturer;
            }
        }
    }

    return null;
}
```

### 3.1 创建Camera1采集 

摄像头枚举类 Camera1Enumerator 和 Camera2Enumerator 继承于 CameraEnumerator。

#### 3.1.1 创建Camera1枚举(Camera1Enumerator)
```java
public Camera1Enumerator(boolean captureToTexture) {
    this.captureToTexture = captureToTexture;
}
```

#### 3.1.2 创建 CameraCapturer
```java

```

### 3.2 创建Camera2采集
Camera2 必须启用 Texture 才能渲染。

#### 3.2.1 创建 Camera2 枚举(Camera2Enumerator)
```java
public Camera2Enumerator(Context context) {
    this.context = context;
    /* 获取系统摄像头管理服务 */
    this.cameraManager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
}
```