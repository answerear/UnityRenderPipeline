Unity 引擎剖析笔记

TransformState (TransformState.h) : 

```c++
/**
 * @brief 变换状态，保存着世界变换矩阵、观察变换矩阵、投影变换矩阵，及其对这些矩阵的内部操作
 * @remarks TransformState 也同时作为访问 Shader 中世界变换、观察变换和投影变换的桥梁
 */
struct TransformState
{
public:
    /// 从内置着色器获取对应的变换矩阵（世界观察变换、世界变换、投影变换）并重置为单位矩阵
    void Invalidate(BuiltinShaderParamValues& builtins);
    
    /// 更新世界和观察变换矩阵
    void UpdateWorldViewMatrix(const BuiltinShaderParamValues& builtins) const;
    
    /// 设置世界变换矩阵，这里并不马上传送到 GPU，仅仅是设置其值
    void SetWorldMatrix(const Matrix4x4f& matrix);
    
    /// 设置观察变换矩阵，这里并不马上计算世界观察变换的乘积，
    /// 要调用 UpdateWorldViewMatrix() 才会计算，这里仅仅是设置其值并且设置数据脏标记
    void SetViewMatrix(const Matrix4x4f& matrix, BuiltinShaderParamValues& builtins);
    
    /// 设置投影变换矩阵
    void SetProjectionMatrix(const Matrix4x4f& matrix);

    Matrix4x4f worldMatrix;	// 世界变换
    Matrix4x4f projectionMatrixOriginal; // Originally set from Unity code，投影变换

    mutable Matrix4x4f worldViewMatrix; // Lazily updated in UpdateWorldViewMatrix()，世界变换和观察变换的乘积
    mutable bool worldViewMatrixDirty;	// 世界观察变换是否需要重新计算标识
};
```

遗留问题：

- 哪里把这些矩阵传送给 GPU ？

CbKey (GfxDeviceTypes.h) :

```c++
/**
 * @brief 常量缓冲区的索引 key
 * @remarks 一个简单的 64 位 key，由其 名字 + 大小 构成
 */
// Constant buffers are identified by name+size as a single 64 bit key:
struct CbKey
{
    static const UInt32 kMaxCbSize; // defined in GfxDevice.cpp，这个值为:65536

    union
    {
        struct
        {
            int id;
            UInt32 size;
        };
        UInt64 key;
    };

    CbKey() {}
    explicit CbKey(UInt64 key) : key(key) { DebugAssert(size <= kMaxCbSize); }
    CbKey(int name, size_t size) : id(name), size(size) { DebugAssert(size <= kMaxCbSize); }
    bool operator<(const CbKey& r) const { return key < r.key; }
    bool operator==(const CbKey& r) const { return key == r.key; }
    bool operator!=(const CbKey& r) const { return key != r.key; }
};
```

遗留问题：

- 其成员 id 是怎么定义的？
  - CbKey 来自 ConstantBuffer.m_Name，其是一个 ShaderLab::FastPropertyName 对象，FastPropertyName 里面仅有 index 成员，CbKey.id 就是来自该 index 成员。index 的生成可以参见 ：FastPropertyName.cpp 中的 void FastPropertyName::Init(const char* inName) 。

BuiltinMatrixParamIndex (BuiltinShaderParams.h) :

```c++
struct BuiltinMatrixParamIndex
{
    BuiltinMatrixParamIndex() : gpuIndex(-1), rows(0), cols(0)
#if GFX_SUPPORTS_CONSTANT_BUFFERS
        , cbKey(0)
        , cbIndex(-1)
#endif
#if GFX_SUPPORTS_OPENGL_UNIFIED
        , isVectorized(false)
#endif
    {}
    int     gpuIndex;
    UInt16  rows;
    UInt16  cols;
#if GFX_SUPPORTS_CONSTANT_BUFFERS
    CbKey   cbKey;
    int     cbIndex;
#endif
#if GFX_SUPPORTS_OPENGL_UNIFIED
    bool    isVectorized; // If true, is really a vec4 array
#endif
};
```



BuiltinShaderParamValues (BuiltinShaderParams.h) :

```c++
class BuiltinShaderParamValues
{
public:
    BuiltinShaderParamValues();

    FORCE_INLINE const Vector4f&            GetVectorParam(BuiltinShaderVectorParam param) const     { DebugAssert(param >= 0 && param < kShaderVecCount); return vectorParamValues[param]; }
    FORCE_INLINE const Matrix4x4f&          GetMatrixParam(BuiltinShaderMatrixParam param) const     { DebugAssert(param >= 0 && param < kShaderMatCount); return matrixParamValues[param]; }
    FORCE_INLINE const ShaderLab::TexEnv&   GetTexEnvParam(BuiltinShaderTexEnvParam param) const     { DebugAssert(param >= 0 && param < kShaderTexEnvCount); return texEnvParamValues[param]; }

    FORCE_INLINE Vector4f&          GetWritableVectorParam(BuiltinShaderVectorParam param)          { DebugAssert(param >= 0 && param < kShaderVecCount); isDirty = true; return vectorParamValues[param]; }
    FORCE_INLINE Matrix4x4f&        GetWritableMatrixParam(BuiltinShaderMatrixParam param)           { DebugAssert(param >= 0 && param < kShaderMatCount); isDirty = true; return matrixParamValues[param]; }
    FORCE_INLINE ShaderLab::TexEnv& GetWritableTexEnvParam(BuiltinShaderTexEnvParam param)           { DebugAssert(param >= 0 && param < kShaderTexEnvCount); isDirty = true; return texEnvParamValues[param]; }

    FORCE_INLINE void   SetVectorParam(BuiltinShaderVectorParam param, const Vector4f& val)          { GetWritableVectorParam(param) = val; }
    FORCE_INLINE void   SetMatrixParam(BuiltinShaderMatrixParam param, const Matrix4x4f& val)        { GetWritableMatrixParam(param) = val; }
    FORCE_INLINE void   SetTexEnvParam(BuiltinShaderTexEnvParam param, const ShaderLab::TexEnv& val) { GetWritableTexEnvParam(param) = val; }

    FORCE_INLINE BuiltinParamData GetBuiltinParamData(int nameIndex) const
    {
        const int index = nameIndex & kShaderPropBuiltinIndexMask;
        switch (nameIndex & kShaderPropBuiltinMask)
        {
            case kShaderPropBuiltinVectorMask:
                return BuiltinParamData(&GetVectorParam((BuiltinShaderVectorParam)index), GetBuiltinVectorParamArraySize((BuiltinShaderVectorParam)index));
            case kShaderPropBuiltinMatrixMask:
                return BuiltinParamData(&GetMatrixParam((BuiltinShaderMatrixParam)index), GetBuiltinMatrixParamArraySize((BuiltinShaderMatrixParam)index));
            case kShaderPropBuiltinTexEnvMask:
                return BuiltinParamData(&GetTexEnvParam((BuiltinShaderTexEnvParam)index), 1);
            default:
                DebugAssert(false && "not builtin");
        }

        return BuiltinParamData(0, 0);
    }

    bool                isDirty;

private:
    Vector4f            vectorParamValues[kShaderVecCount];
    Matrix4x4f          matrixParamValues[kShaderMatCount];
    ShaderLab::TexEnv   texEnvParamValues[kShaderTexEnvCount];
};
```

GfxContextData : 

```c++
// Struct to hold all state data that needs propagation to gfx job
// At the point where a graphics job is spawned this gets cached on the main thread
// and a ptr to the cached instance is passed to the job.
struct GfxContextData
{
    GfxContextData()
        : m_Dirty(false)
    {
    }

    // if this returns true we need to create a new cached instance of GfxContextData on the main thread
    // and move the old cached instance to recycle pool
    bool IsDirty() const
    {
        return m_Dirty || m_BuiltinParamValues.isDirty || m_TransformState.worldViewMatrixDirty;
    }

    // call this after creating a new cached instance on the main thread, it makes sure transform state is not stale
    // and resets the dirty flag
    void Update()
    {
        m_TransformState.UpdateWorldViewMatrix(m_BuiltinParamValues);
        m_BuiltinParamValues.isDirty = false;
        m_Dirty = false;
    }

    void SetInsideFrame(bool val) { m_InsideFrame = val; m_Dirty = true; }
    bool GetInsideFrame() const { return m_InsideFrame; }

    void SetActiveCubemapFace(CubemapFace val) { m_ActiveCubemapFace = val; m_Dirty = true; }
    CubemapFace GetActiveCubemapFace() const { return m_ActiveCubemapFace; }

    void SetActiveMipLevel(int val) { m_ActiveMipLevel = val; m_Dirty = true; }
    int GetActiveMipLevel() const { return m_ActiveMipLevel; }

    void SetActiveDepthSlice(int val) { m_ActiveDepthSlice = val; m_Dirty = true; }
    int GetActiveDepthSlice() const { return m_ActiveDepthSlice; }

    void SetStereoActiveEye(StereoscopicEye val) { m_StereoActiveEye = val; m_Dirty = true; }
    StereoscopicEye GetStereoActiveEye() const { return m_StereoActiveEye; }

    void SetSinglePassStereo(SinglePassStereo val) { m_SinglePassStereo = val; m_Dirty = true; }
    SinglePassStereo GetSinglePassStereo() const { return m_SinglePassStereo; }

    void SetInstanceCountMultiplier(int val) { m_InstanceCountMultiplier = val; m_Dirty = true; }
    int GetInstanceCountMultiplier() const
    {
        if (m_InstanceCountMultiplier == 0)
            return (m_SinglePassStereo == kSinglePassStereoInstancing ? 2 : 1);
        return m_InstanceCountMultiplier;
    }

    void SetSinglePassStereoEyeMask(TargetEyeMask val) { m_SinglePassStereoEyeMask = val; m_Dirty = true; }
    TargetEyeMask GetSinglePassStereoEyeMask() const { return m_SinglePassStereoEyeMask; }

    void SetInvertProjectionMatrix(bool val) { m_InvertProjectionMatrix = val; m_Dirty = true; }
    bool GetInvertProjectionMatrix() const { return m_InvertProjectionMatrix; }

    void SetUserBackfaceMode(bool val) { m_UserBackfaceMode = val; m_Dirty = true; }
    bool GetUserBackfaceMode() const { return m_UserBackfaceMode; }

    void SetForceCullMode(CullMode val) { m_ForceCullMode = val; m_Dirty = true; }
    CullMode GetForceCullMode() const { return m_ForceCullMode; }

    void SetGlobalDepthBias(float val) { m_GlobalDepthBias = val; m_Dirty = true; }
    float GetGlobalDepthBias() const { return m_GlobalDepthBias; }

    void SetGlobalSlopeDepthBias(float val) { m_GlobalSlopeDepthBias = val; m_Dirty = true; }
    float GetGlobalSlopeDepthBias() const { return m_GlobalSlopeDepthBias; }

#if UNITY_EDITOR
    void SetUsingAsyncDummyShader(bool val) { m_UsingAsyncDummyShader = val; m_Dirty = true; }
    bool GetUsingAsyncDummyShader() const { return m_UsingAsyncDummyShader; }
#endif

    BuiltinShaderParamValues m_BuiltinParamValues;	// 内置 shader 参数表
    TransformState m_TransformState;	// MVP 变换

private:
    bool                m_InsideFrame;
    CubemapFace         m_ActiveCubemapFace;
    int                 m_ActiveMipLevel;
    int                 m_ActiveDepthSlice;
    StereoscopicEye     m_StereoActiveEye;
    SinglePassStereo    m_SinglePassStereo;		// PC & PS4 上的 VR 应用使用到的一种特性
    TargetEyeMask       m_SinglePassStereoEyeMask;	// PC & PS4 上的 VR 应用使用到的一种特性
    int                 m_InstanceCountMultiplier;
    bool                m_InvertProjectionMatrix;
    bool                m_UserBackfaceMode;
    // Override shader-specified cull mode with this
    // (kCullUnknown does not override)
    CullMode            m_ForceCullMode;
    float               m_GlobalDepthBias;
    float               m_GlobalSlopeDepthBias;
    Vector2f            m_StereoScale;
    Vector2f            m_StereoOffset;
#if UNITY_EDITOR
    bool                m_UsingAsyncDummyShader;
#endif

    bool m_Dirty;
};

```

GfxDevice (GfxDevice.h) : 

```c++
// Base graphics device interface.
// No code in the graphics device should access the ScreenManager or other higher-level Unity components.
// Platform specific implementations should only be concerned with rendering to provided surfaces.
// Avoid adding new variables to this class as platform specific implementations might have different ways of handling state.
// Absolutely no static variables or assumptions about threads - multiple instances of this class will be used on jobs.
class GfxDevice
{
public:

    enum SurfaceFlags
    {
        kSurfaceDefault = 0,
        kSurfaceUseResolvedBuffer = (1 << 2), // Metal: used for MSAA workarounds depending on hasStoreAndMultisampleResolveAction and hasMultiSampleResolveDepth caps support
        // next flag (1<<3)
    };

    enum ReloadResourcesFlags
    {
        kReleaseRenderTextures = (1 << 0),
        kReloadShaders = (1 << 1),
        kReloadTextures = (1 << 2),
    };

    enum RenderTargetFlags
    {
        kFlagDontRestoreColor   = (1 << 0),   // Xbox 360 specific: do not restore old contents to EDRAM
        kFlagDontRestoreDepth   = (1 << 1),   // Xbox 360 specific: do not restore old contents to EDRAM
        kFlagDontRestore        = kFlagDontRestoreColor | kFlagDontRestoreDepth,
        kFlagForceResolve       = (1 << 3),   // Xbox 360 specific: force a resolve to system RAM
        // currently this is used only in editor, so it is implemented only for editor gfx devices (gl/dx)
        kFlagForceSetRT         = (1 << 4),   // force SetRenderTargets call (bypass any caching involved and actually set RT)
        kFlagDontResolve        = (1 << 5),   // Skip AA resolve even when the previous RT was AA. Used when doing shenanigans with default backbuffers.
        kFlagReadOnlyDepth      = (1 << 6), // If set, bind the depth surface as read-only (if supported by the gfxdevice; see GraphicsCaps::hasReadOnlyDepth)
    };

    enum GfxProfileControl
    {
        kGfxProfBeginFrame = 0,
        kGfxProfEndFrame,
        kGfxProfSetActive,
    };

    // tracks if View/Proj matrices need to be reapplied to shader (i.e. after SetPass)
    enum ViewProjMatrixNeedApplyFlag
    {
        kMatricesToApplyFlagView    = 1 << 0,
        kMatricesToApplyFlagProj    = 1 << 1,
        kMatricesToApplyFlagNone    = 0
    };

    enum VertexDeclarationMRUCacheIndex
    {
        kVertexDeclarationMRUCacheGeneral,
        kVertexDeclarationMRUCacheBatching,
        kVertexDeclarationMRUCacheSprites,
        kVertexDeclarationMRUCacheUI,
        kVertexDeclarationMRUCacheCount
    };

    explicit GfxDevice(MemLabelRef label);
    GFX_API ~GfxDevice();

    static CallbackArray CleanupGfxDeviceResourcesCallbacks;
    static CallbackArray InitializeGfxDeviceResourcesCallbacks;

    GFX_API void    InvalidateState();

    // these will be called by real gfx device before/after calling plugin render marker
    // by default we keep old behavior of doing Invalidate in both cases
    GFX_API void    BeforePluginRender()    { InvalidateState(); }
    GFX_API void    AfterPluginRender()     { InvalidateState(); }


    GfxDeviceRenderer GetRenderer() const { return m_Renderer; }
    bool IsDeviceClient() const { return m_IsDeviceClient; } // Returns true if this is an instance of GfxDeviceClient, false otherwise.

#if GFX_SUPPORTS_OPENGL_UNIFIED
    GFX_API GfxDeviceLevelGL GetDeviceLevel() const { return kGfxLevelUninitialized; }
#endif//GFX_SUPPORTS_OPENGL_UNIFIED

    inline GraphicsTier GetActiveTier() const { return GetGraphicsCaps().activeTier; }
    GFX_API void SetActiveTier(GraphicsTier tier);

    GFX_API void SetMaxBufferedFrames(int bufferSize) { m_MaxBufferedFrames = bufferSize; }
    int GetMaxBufferedFrames() const { return m_MaxBufferedFrames; }

    const GfxDeviceStats& GetFrameStats() const { return m_Stats; }
    GfxDeviceStats& GetFrameStats() { return m_Stats; }

    const FrameTimingManager& GetFrameTimingManager() const;
    FrameTimingManager& GetFrameTimingManager();

    const HDROutputSettings& GetHDROutputSettings() const;
    HDROutputSettings& GetHDROutputSettings();

    const BuiltinShaderParamValues& GetBuiltinParamValues() const { return m_GfxContextData.m_BuiltinParamValues; }
    BuiltinShaderParamValues& GetBuiltinParamValues() { return m_GfxContextData.m_BuiltinParamValues; }

    DynamicVBO& GetDynamicVBO();

    int         GetTotalBufferCount() const;
    size_t      GetTotalBufferBytes() const;

    GFX_API void    Clear(GfxClearFlags clearFlags, const ColorRGBAf& color, float depth, UInt32 stencil) GFX_PURE;
    GFX_API void    SetInvertProjectionMatrix(bool enable);
    bool            GetInvertProjectionMatrix() const { return m_GfxContextData.GetInvertProjectionMatrix(); }

    GFX_API void RenderTargetBarrier() {}

    GFX_API size_t GetNumGraphicsJobs(size_t count);
    GFX_API size_t GetMinimumNodesPerGraphicsJob(void);
    GFX_API const DeviceBlendState* CreateBlendState(const GfxBlendState& state) GFX_PURE;
    GFX_API const DeviceDepthState* CreateDepthState(const GfxDepthState& state) GFX_PURE;
    GFX_API const DeviceStencilState* CreateStencilState(const GfxStencilState& state) GFX_PURE;
    GFX_API const DeviceRasterState* CreateRasterState(const GfxRasterState& state) GFX_PURE;

    GFX_API void SetBlendState(const DeviceBlendState* state) GFX_PURE;
    GFX_API void SetRasterState(const DeviceRasterState* state) GFX_PURE;
    GFX_API void SetDepthState(const DeviceDepthState* state) GFX_PURE;
    GFX_API void SetStencilState(const DeviceStencilState* state, int stencilRef) GFX_PURE;
    // When we're skipping regular stencil state application, we still want to remember the stencil reference value.
    // This makes sure it is recorded properly in material display lists of a MT device.
    GFX_API void SetStencilRefWhenStencilWasSkipped(int stencilRef) { m_StencilRefWhenStencilWasSkipped = stencilRef; }
    int GetStencilRefWhenStencilWasSkipped() const { return m_StencilRefWhenStencilWasSkipped; }

    GFX_API void SetSRGBWrite(const bool) GFX_PURE;
    GFX_API bool GetSRGBWrite() GFX_PURE;

    GFX_API void SetUserBackfaceMode(bool enable) GFX_PURE;
    bool GetUserBackfaceMode() const { return m_GfxContextData.GetUserBackfaceMode(); }
    GFX_API void SetForceCullMode(CullMode mode) GFX_PURE;
    GFX_API void SetWireframe(bool wire) GFX_PURE;
    GFX_API bool GetWireframe() const GFX_PURE;

    GFX_API void    SetWorldMatrixAndType(const Matrix4x4f& matrix, TransformType type);
    GFX_API void    SetWorldMatrix(const Matrix4x4f& matrix);
    GFX_API void    SetViewMatrix(const Matrix4x4f& matrix);
    GFX_API void    SetProjectionMatrix(const Matrix4x4f& matrix);

    // VP matrix (kShaderMatViewProj) is updated when setting view matrix but not when setting projection.
    // Call UpdateViewProjectionMatrix() explicitly if you only change projection matrix.
    GFX_API void    UpdateViewProjectionMatrix();

    void ClearViewProjMatrixNeedApplyFlags() { m_ViewProjMatrixNeedApplyFlags = kMatricesToApplyFlagNone; }

    SinglePassStereoSupportExt m_SinglePassStereoSupport;
    SinglePassStereoSupportExt& GetSinglePassStereoSupport() { return m_SinglePassStereoSupport; }
    friend class SinglePassStereoSupportExt;

    GFX_API void    UpdateStereoViewProjectionMatrix(MonoOrStereoscopicEye monoOrStereoEye);
    GFX_API void    SetStereoMatrix(MonoOrStereoscopicEye monoOrStereoEye, BuiltinShaderMatrixParam param, const Matrix4x4f& matrix);
    GFX_API void    GetStereoMatrix(MonoOrStereoscopicEye monoOrStereoEye, BuiltinShaderMatrixParam param, Matrix4x4f& matrixOut);
    GFX_API void SetStereoViewport(StereoscopicEye eye, const RectInt& rect);
    RectInt GetStereoViewport(StereoscopicEye eye) const;
    GFX_API void    SetStereoScissorRects(const RectInt rects[kStereoscopicEyeCount]);
    GFX_API void    SetStereoConstantBuffers(int leftID, int rightID, int monoscopicID, size_t size);
    GFX_API void    SetSinglePassStereoEyeMask(TargetEyeMask eyeMask);
    TargetEyeMask GetSinglePassStereoEyeMask() const;
    GFX_API void    SaveStereoConstants();
    GFX_API void    RestoreStereoConstants();
    void SetCopyMonoTransformsToStereo(bool copy);
    bool GetCopyMonoTransformsToStereo() const;

    GFX_API const Matrix4x4f& GetWorldViewMatrix() const;
    GFX_API const Matrix4x4f& GetWorldMatrix() const;
    GFX_API const Matrix4x4f& GetViewMatrix() const;
    GFX_API const Matrix4x4f& GetProjectionMatrix() const; // get projection matrix as passed from Unity (OpenGL projection conventions)
    GFX_API const Matrix4x4f& GetDeviceProjectionMatrix() const; // get projection matrix that will be actually used

    // Calculate a device projection matrix given a Unity projection matrix, this needs to be virtual so that platforms can override if necessary.
    GFX_API void CalculateDeviceProjectionMatrix(Matrix4x4f& m, bool usesOpenGLTextureCoords, bool invertY) const;

    GFX_API void    SetBackfaceMode(bool backface) GFX_PURE;

    GFX_API void    SetViewport(const RectInt& rect) GFX_PURE;
    GFX_API RectInt GetViewport() const GFX_PURE;
    GFX_API RectInt GetPlatformAdjustedViewport() const { return GetViewport(); }

    GFX_API void    SetScissorRect(const RectInt& rect) GFX_PURE;
    GFX_API void    DisableScissor() GFX_PURE;
    GFX_API bool    IsScissorEnabled() const GFX_PURE;
    GFX_API RectInt GetScissorRect() const GFX_PURE;

    // Set textures used for later draw calls.
    GFX_API void    SetTextures(ShaderType shaderType, int count, const GfxTextureParam* textures) GFX_PURE;

    // Set "inline samplers" (samplers that aren't tied into any particular texture; but with sampling state hardcoded in the shader instead)
    // to be used for later draw calls. Assumes this is called after SetTextures call.
    GFX_API void    SetSamplers(ShaderType shaderType, int count, const GfxSamplerParam* samplers) {}  // empty impl by default, for platforms that don't support separate samplers
    GFX_API void    SetTextureSamplingParams(TextureID texture, const GfxTextureSamplingParams& sp) GFX_PURE;

    // If the properties object is allocated on the stack or is newed/deleted explicitly you should use this API to copy the content to the render thread.
    // Most IntermediateRenderer calls to this API to set their properties.
    GFX_API void    SetShaderPropertiesCopied(const ShaderPropertySheet& properties) GFX_PURE;
    // If the properties object is managed by reference counting i.e. newed and released you may use this API to pass thru only the reference directly to the render thread.
    // Renderer calls to this API.
    GFX_API void    SetShaderPropertiesShared(const ShaderPropertySheet& properties) { SetShaderPropertiesCopied(properties); }

    GFX_API GpuProgram* CreateGpuProgram(ShaderGpuProgramType shaderProgramType, const dynamic_array<UInt8>& source, CreateGpuProgramOutput& output);
    GFX_API void    SetShadersMainThread(const ShaderLab::SubPrograms& programs, const ShaderPropertySheet* localProps, const ShaderPropertySheet* globalProps) GFX_PURE;

    GFX_API bool    IsShaderActive(ShaderType type) const GFX_PURE;
    GFX_API void    DestroySubProgram(ShaderLab::SubProgram* subprogram) GFX_PURE;
    GFX_API void    DestroyGpuProgram(GpuProgram const * const program);

    GFX_API void    UpdateConstantBuffer(CbKey cbKey, const void* data, size_t dataLen) {}
    // Adjust the constant buffer sizes for the next instanced draw with a hint to the maximum array size.
    GFX_API void    AdjustInstancingConstantBufferBindings(const CbKey cbKeys[], const CbKey swapTos[], size_t cbCount, UInt32 arraySizeHint) {}
    // Map a group of constant buffers identified by cbKeys to system memory for write.
    GFX_API void    MapConstantBuffers(void* ptrs[], const CbKey cbKeys[], size_t mapLens[], size_t cbCount);
    // Finish writing to a group of constant buffers identified by cbKeys array. The JobFence will be synced before freeing, to ensure
    // the job/jobgroup scheduled for writing the buffer is completed (done on render thread for GfxDeviceClient) before the memory becomes invalid.
    GFX_API void    UnmapConstantBuffers(JobFence& fence, void* ptrs[], const CbKey cbKeys[], size_t mapLens[], size_t cbCount);


    // -------- Buffer API

    // Allocates a GfxBuffer and fills in the initial information about the buffer
    // This should be callable from any thread at any point, so in most cases the device would only allocate the CPU side struct here
    // and not touch the actual graphics API unless it's completely thread safe
    GFX_API GfxBuffer* AllocateBufferInternal(const GfxBufferDesc& desc) GFX_PURE;

    // Initializes an already allocated GfxBuffer.
    // Must be called from a thread that owns the gfx device at this point.
    GFX_API void InitializeBufferInternal(GfxBuffer* buffer, const void* initData, GfxUpdateBufferFlags flags) GFX_PURE;

    // Create a graphics buffer (e.g. for vertex or index data).
    // Buffer parameters (size, binding targets, etc) are immutable; if you need resizing or to change
    // flags the buffer has to be re-created.
    // Initial buffer data (e.g. for immutable buffers) can be optionally specified in /initData/.
    GfxBuffer* CreateBuffer(const GfxBufferDesc& desc, const void* initData, GfxUpdateBufferFlags flags)
    {
        GfxBuffer* buffer = AllocateBufferInternal(desc);
        InitializeBufferInternal(buffer, initData, flags);
        return buffer;
    }

    // Create a graphics buffer for use with the dynamic VBO. For devices thaat support native graphics jobs, their devices will create temporary buffers using scratch memory.
    GFX_API GfxBuffer* CreateDynamicVBOBuffer(DynamicVBOScratchMemory*& scratchMemory, size_t size, GfxBufferTarget bufferType, GfxBufferMode bufferMode)
    {
        scratchMemory = NULL;
        return AllocateBufferInternal(GfxBufferDesc(size, 0, bufferType, bufferMode, kGfxBufferLabelInternal));
    }

    GFX_API void ReserveDynamicVBOBuffer(DynamicVBOScratchMemory* scratchMemory, GfxBuffer* buffer, UInt32 size)
    {
        AssertMsg(scratchMemory, "DynamicVBO: scratchMemory should not be NULL");
        scratchMemory->Reserve(buffer, size);
    }

    GFX_API void* BeginDynamicVBOBufferWrite(GfxBuffer* buffer)
    {
        return BeginBufferWrite(buffer);
    }

    // Update data of an existing buffer; always full overwrite of the whole buffer contents.
    // Can not be used on immutable buffers.
    GFX_API void    UpdateBuffer(GfxBuffer* buffer, const void* data, GfxUpdateBufferFlags flags) GFX_PURE;

    // Copy data from CPU memory onto a kGfxBufferModeSubUpdates buffer.
    // If kGfxUpdateBufferAcquiredPointer is used then the ranges array and the source pointers within will be sent as pointers instead of full data copy.
    GFX_API void    UpdateBufferRanges(GfxBuffer* buffer, const GfxUpdateBufferRange* ranges, int rangeCount, size_t bufferWriteStart, size_t bufferWriteEnd, GfxUpdateBufferFlags flags);

    // Begin writing ("map" / "lock") to part of a dynamic or circular buffer.
    // Returns pointer where to write data into, or NULL on error.
    GFX_API void*   BeginBufferWrite(GfxBuffer* buffer, size_t offset = 0, size_t size = 0) GFX_PURE;

    // End writing ("unmap" / "unlock") to part of a dynamic or circular buffer.
    GFX_API void    EndBufferWrite(GfxBuffer* buffer, size_t bytesWritten) GFX_PURE;

    // Delete a graphics buffer.
    GFX_API void    DeleteBuffer(GfxBuffer* buffer) GFX_PURE;

    // Draw a batch with the new batcher
#if GFX_SUPPORTS_SRP_BATCHER
    GFX_API void    DrawBuffersBatchMode(const GfxBatchHeader& params) {}
#endif
    GFX_API VertexDeclaration* GetVertexDeclaration(const VertexChannelsInfo& declKey) { return NULL; }
    VertexDeclaration*         GetVertexDeclaration(const VertexChannelsInfo& declKey, VertexDeclarationMRUCacheIndex cacheIndex);

    GFX_API GfxBuffer*  GetDefaultVertexBuffer(GfxDefaultVertexBufferType type, size_t size);

    GFX_API void    DrawBuffers(GfxBuffer* indexBuf, UInt32 indexStride,
        GfxBuffer* const* vertexBufs, const UInt32* vertexStrides, int vertexStreamCount,
        const DrawBuffersRange* drawRanges, int drawRangeCount,
        VertexDeclaration* vertexDecl) {}

    void    DrawBuffers(GfxBuffer* indexBuf,
        GfxBuffer* const* vertexBufs, int vertexStreamCount,
        const DrawBuffersRange* drawRanges, int drawRangeCount,
        VertexDeclaration* vertexDecl)
    {
        DrawBuffers(indexBuf, 0, vertexBufs, NULL, vertexStreamCount, drawRanges, drawRangeCount, vertexDecl);
    }

    GFX_API void    DrawBuffersIndirect(GfxBuffer* indexBuf, UInt32 indexStride,
        GfxBuffer* const* vertexBufs, const UInt32* vertexStrides, int vertexStreamCount,
        VertexDeclaration* vertexDecl, GfxPrimitiveType topology,
        ComputeBufferID indirectBuffer, UInt32 bufferOffset) {}

    void    DrawBuffersIndirect(GfxBuffer* indexBuf,
        GfxBuffer* const* vertexBufs, int vertexStreamCount,
        VertexDeclaration* vertexDecl, GfxPrimitiveType topology,
        ComputeBufferID indirectBuffer, UInt32 bufferOffset)
    {
        DrawBuffersIndirect(indexBuf, 0, vertexBufs, NULL, vertexStreamCount, vertexDecl, topology, indirectBuffer, bufferOffset);
    }

    /// Geometry Job API usage: See GeometryJob.h

    // Allocates a GeometryJobFence. Must be passed to GeometryJobInstruction.
    static GeometryJobFence CreateGeometryJobFence(bool allowMainThreadQueue = true) { return s_GeometryJobs.CreateFence(allowMainThreadQueue); }
    static GeometryJobTasks& GetGeometryJobs() { return s_GeometryJobs; }

    // Schedules geometry jobs in batch.
    template<typename T>
    void ScheduleGeometryJobs(void jobFunc(T*), const GeometryJobInstruction* jobDatas, UInt32 jobCount)
    {
        CompileTimeAssert((IsBaseOfType<GeometryJobData, T>::result), "Expected jobFunc to take a jobData derived from GeometryJobData");
        ScheduleGeometryJobsInternal(reinterpret_cast<GeometryJobFunc*>(jobFunc), jobDatas, jobCount);
    }

    template<typename T>
    void ScheduleComputeBufferJobs(void jobFunc(T*), GeometryJobScheduledFunc* scheduledFunc, const ComputeBufferJobInstruction* jobDatas, UInt32 jobCount)
    {
        CompileTimeAssert((IsBaseOfType<ComputeBufferJobData, T>::result), "Expected jobFunc to take a jobData derived from GeometryJobData");
        ScheduleComputeBufferJobsInternal(reinterpret_cast<ComputeBufferJobFunc*>(jobFunc), scheduledFunc, jobDatas, jobCount);
    }

    bool ScheduleSharedGeometryJobs(GeometryJobFence fence, SharedGeometryJobFunc* jobFunc, SharedGeometryJobFreeFunc* freeMemFunc, GeometryJobScheduledFunc* scheduledFunc, SharedGeometryJobData* jobData, UInt32 jobCount, const DynamicVBOBuffer* vb, const DynamicVBOBuffer* ib)
    {
        return ScheduleSharedGeometryJobsInternal(fence, jobFunc, freeMemFunc, scheduledFunc, jobData, jobCount, vb, ib);
    }

    GFX_API void AcquireSharedDynamicVBOChunk(GfxBufferTarget bufferType, size_t elementCount, UInt32 stride)
    {
        DynamicVBOBufferManager::AcquireSharedInternal(*this, bufferType, elementCount, stride);
    }

    GFX_API void DrawSharedGeometryJobs(const DynamicVBOBuffer& vertexBuffer, UInt32 vertexStride, const DynamicVBOBuffer& indexBuffer, UInt32 indexStride, GeometryJobFence fence, const DrawBuffersRange* params, size_t paramsCount, VertexDeclaration* vertexDecl);
    void DrawSharedGeometryJobs(const DynamicVBOBuffer& vertexBuffer, UInt32 vertexStride, GeometryJobFence fence, const DrawBuffersRange* params, size_t paramsCount, VertexDeclaration* vertexDecl)
    {
        DrawSharedGeometryJobs(vertexBuffer, vertexStride, DynamicVBOBuffer(), 0, fence, params, paramsCount, vertexDecl);
    }

protected:
    GFX_API void    ScheduleGeometryJobsInternal(GeometryJobFunc* jobFunc, const GeometryJobInstruction* jobDatas, UInt32 jobCount);
    void    ScheduleComputeBufferJobsInternal(ComputeBufferJobFunc* jobFunc, GeometryJobScheduledFunc* scheduledFunc, const ComputeBufferJobInstruction* jobDatas, UInt32 jobCount);
    GFX_API bool    ScheduleSharedGeometryJobsInternal(GeometryJobFence fence, SharedGeometryJobFunc* jobFunc, SharedGeometryJobFreeFunc* freeMemFunc, GeometryJobScheduledFunc* scheduledFunc, SharedGeometryJobData* jobData, UInt32 jobCount, const DynamicVBOBuffer* vb, const DynamicVBOBuffer* ib);

public:

    // Waits for geometry job to complete on the render thread.
    // Unmaps vertex buffer after the geometry job has completed
    // IMPORTANT:
    // - This does not wait for the job to complete on the main thread.
    //   Thus it is NOT safe to deallocate userData input for the job on the main thread. EVER!
    GFX_API void    PutGeometryJobFence(GeometryJobFence& fence);

    // Synchronizes all rendering
    GFX_API void    EndAsyncJobFrame();
    void            EndDynamicVBOFrame();
    GFX_API void    EndGeometryJobFrame();
#if UNITY_DEFER_GRAPHICS_JOBS_SCHEDULE
    GFX_API void    StartAsyncJobFrame();
#endif // #if UNITY_DEFER_GRAPHICS_JOBS_SCHEDULE

#if GFX_ENABLE_DRAW_CALL_BATCHING
    static  int     GetMaxStaticBatchIndices();

#if ENABLE_PROFILER
    GFX_API void    AddBatchStats(GfxBatchStatsType type, int batchedTris, int batchedVerts, int batchedCalls, TimeFormat batchedDt, int batchCount = 1);
#endif


    GFX_API void    BeginDynamicBatching(ShaderChannelMask shaderChannels, ShaderChannelMask channelMask, UInt32 stride, VertexDeclaration* vertexDecl, size_t maxVertices, size_t maxIndices, GfxPrimitiveType topology);
    GFX_API void    DynamicBatchMesh(const Matrix4x4f& matrix, const UInt8* vertices, UInt32 firstVertex, UInt32 vertexCount, const UInt16* indices, UInt32 indexCount, GfxTransformVerticesParams params, GfxTransformVerticesFlags flags, UInt32 defaultColor = 0xFFFFFFFF);
    GFX_API void    EndDynamicBatching(TransformType transformType);
#endif

    GFX_API void    AddSetPassStat();

    GFX_API void    ReleaseSharedMeshData(const SharedMeshData* data);
    GFX_API void    ReleaseSharedTextureData(const SharedTextureData* data);   // use in conjunction with kTextureUploadAcquiredPointer
    GFX_API void    ReleaseAsyncCommandHeader(GfxDeviceAsyncCommand::Arg* header);

    GFX_API GPUSkinPoseBuffer* AllocateGPUSkinPoseBufferInternal() { return NULL; }
    GFX_API void InitializeGPUSkinPoseBufferInternal(GPUSkinPoseBuffer* poseBuffer) {}  // TODO: looks like this is not needed anywhere
    GFX_API GPUSkinPoseBuffer* CreateGPUSkinPoseBuffer()
    {
        GPUSkinPoseBuffer* poseBuffer = AllocateGPUSkinPoseBufferInternal();
        InitializeGPUSkinPoseBufferInternal(poseBuffer);
        return poseBuffer;
    }

    GFX_API void    DeleteGPUSkinPoseBuffer(GPUSkinPoseBuffer* poseBuffer) { Assert(false); }
    GFX_API void    UpdateSkinPoseBuffer(GPUSkinPoseBuffer* poseBuffer, MatrixArrayJobOutput* matrixArray) { Assert(false); }

    /**
    * SkinOnGPU - Perform GPU-assisted skinning into the destination vertex buffer.
    *
    * @param skinBuffer Skin indices and weights
    * @param poseBuffer Bone matrices
    * @param destBuffer Target vertex buffer for the skinned data
    */
    GFX_API void    SkinOnGPU(GfxBuffer* const* vertexBuffers, int vertexStreamCount, GPUSkinPoseBuffer* poseBuffer, GfxBuffer* destBuffer, int vertexCount, int bonesPerVertex, VertexDeclaration* vertexDecl, ShaderChannelMask channelMask) { Assert(false); }
    GFX_API void    ComputeSkinning(GfxBuffer* const* vertexBuffers, int vertexStreamCount, GfxBuffer* poseBuffer, GfxBuffer* destBuffer, int vertexCount, int bonesPerVertex, ShaderChannelMask channelMask);
    GFX_API void    ApplyBlendShape(GfxBuffer* destBuffer, GfxBuffer* blendShapeBuffer, int blendShapeFirstVertex, int blendShapeVertexCount, ShaderChannelMask blendShapeChannelMask, float blendWeight);
    GFX_API void    UpdateComputeSkinPoseBuffer(GfxBuffer* buffer, MatrixArrayJobOutput* matrixArray);

    // Platform specific API extensions are kept in the
    // PlatformDependent folder and implemented there.
    #define GFX_EXT_POSTFIX GFX_PURE
    #include "Runtime/GfxDevice/PlatformSpecificGfxDeviceExt.h"
    #undef  GFX_EXT_POSTFIX


    // Render Surfaces / Render Targets

    // Sets up render target(s), including various flags and load/store actions.
    // Note, this is not virtual on purpose; does some common work and then calls a virtual SetRenderTargetsImpl
    void SetRenderTargets(const GfxRenderTargetSetup& rt);

    // Renderpass API. Default implementation (in overrideable *Impl functions) uses SetRenderTargets.
    // The common API facade performs error checking and tracks state (compute etc are not allowed when in renderpass)
    void BeginRenderPass(const RenderPassSetup &setup);
    void NextSubPass();
    void EndRenderPass();

    // Currently active render target face/mip/slice
    CubemapFace GetActiveCubemapFace() const { return m_GfxContextData.GetActiveCubemapFace(); }
    int GetActiveMipLevel() const { return m_GfxContextData.GetActiveMipLevel(); }
    int GetActiveDepthSlice() const { return m_GfxContextData.GetActiveDepthSlice(); }

    GFX_API RenderSurfaceHandle CreateRenderColorSurface(TextureID textureID, int width, int height, int samples, int depth, TextureDimension dim, GraphicsFormat format, SurfaceCreateFlags createFlags);

    // Takes an explicit mip count. Setting mipCount=-1 is equivalent to calling the default overload (i.e. the full mip chain will be created if mip generation is specified in createFlags).
    // Passing a higher count than is possible is safe (and equivalent to -1).  Ignored on platforms that do not support partial mip chains.
    GFX_API RenderSurfaceHandle CreateRenderColorSurface(TextureID textureID, int width, int height, int samples, int depth, int mipCount, TextureDimension dim, GraphicsFormat format, SurfaceCreateFlags createFlags);
    GFX_API RenderSurfaceHandle CreateRenderDepthSurface(TextureID textureID, int width, int height, int samples, int depth, TextureDimension dim, DepthBufferFormat format, SurfaceCreateFlags createFlags);
    GFX_API bool CreateStencilTexture(TextureID textureID, RenderSurfaceHandle& depthRs, GraphicsFormat stencilFormat);
    GFX_API void DestroyRenderSurface(RenderSurfaceHandle& rs);

    GFX_API void ResolveColorSurface(RenderSurfaceHandle srcHandle, RenderSurfaceHandle dstHandle) GFX_PURE;
    GFX_API void ResolveDepthIntoTexture(RenderSurfaceHandle /*colorHandle*/, RenderSurfaceHandle /*depthHandle*/) {}

    GFX_API void DiscardContents(RenderSurfaceHandle& rs) GFX_PURE;
    // Do not produce a warning for next unresolve of current RT; it is expected and there's nothing we can do about it
    GFX_API void IgnoreNextUnresolveOnCurrentRenderTarget() {}
    GFX_API void IgnoreNextUnresolveOnRS(RenderSurfaceHandle rs) {}

    GFX_API RenderSurfaceHandle GetActiveRenderColorSurface(int index) const GFX_PURE;
    GFX_API RenderSurfaceHandle GetActiveRenderDepthSurface() const GFX_PURE;
    GFX_API int GetActiveRenderTargetCount() const GFX_PURE;

    GFX_API int GetActiveRenderTargets(RenderSurfaceSet &colorRTs, RenderSurfaceHandle &depthRT) const;

    GFX_API int GetActiveRenderSurfaceWidth() const { return GetActiveRenderColorSurface(0).object->width; }
    GFX_API int GetActiveRenderSurfaceHeight() const { return GetActiveRenderColorSurface(0).object->height; }

    GFX_API bool IsRenderingToBackBuffer() const { return GetActiveRenderColorSurface(0).object->backBuffer; }

    // The main contract is: pointer to RenderSurface may be kept.
    // For RenderTextures that means that on RT destroy we will call RenderManager::OnRenderSurfaceDestroyed
    //   so cameras still referencing this surface would be switched to "draw to screen"
    // For backbuffer handling we have more complicated workflow, as it might be useful to customize "backbuffer" with RenderSurface
    //   for that we provide Clone functions

    // TODO: we might need to extend it in the future, e.g. for multi-display
    GFX_API RenderSurfaceHandle GetBackBufferColorSurface()    { return m_BackBufferColor; }
    GFX_API RenderSurfaceHandle GetBackBufferDepthSurface()    { return m_BackBufferDepth; }

    // This will copy whole platform render surface descriptor (and not just store argument pointer)
    GFX_API void SetBackBufferColorDepthSurface(RenderSurfaceBase* color, RenderSurfaceBase *depth);

    // Used with dynamic resolution to indicate the current scale of dynamic render targets
    GFX_API void SetScalableBufferFrameData(ScalableBufferManager::FrameData* frameData) {}

    //
    // low level RenderSurface interface.
    //

    // sizeof of appropriate platform struct
    GFX_API size_t              RenderSurfaceStructMemorySize(bool colorSurface) GFX_PURE;
    // simple alloc/dealloc - do not touch platform-specific parts of render surface
    GFX_API RenderSurfaceBase*  AllocRenderSurface(bool colorSurface);
    GFX_API void                DeallocRenderSurface(RenderSurfaceBase* rs);
    // init platform-specific parts of render surface (rs will already have RenderSurfaceBase members filled)
    // in case of failure (returns false) surface will be marked as dummy (kSurfaceCreateNeverUsed)
    GFX_API bool                CreateColorRenderSurfacePlatform(RenderSurfaceBase* rs, GraphicsFormat format) GFX_PURE;
    GFX_API bool                CreateDepthRenderSurfacePlatform(RenderSurfaceBase* rs, DepthBufferFormat format) GFX_PURE;
    // switch surface out of platform specific fast memory if applicable (such as when RenderBufferManager adds it to the free list)
    GFX_API void                SwitchColorRenderSurfaceOutOfFastMemoryPlatform(RenderSurfaceBase* rs, bool copyContents) {}
    GFX_API void                SwitchDepthRenderSurfaceOutOfFastMemoryPlatform(RenderSurfaceBase* rs, bool copyContents) {}
    // switch surface into platform specific fast memory if applicable (such as when RenderBufferManager removes it from the free list)
    GFX_API void                SwitchColorRenderSurfaceIntoFastMemoryPlatform(RenderSurfaceBase* rs, SurfaceUsage usage, FastMemoryFlags flags, bool copyContents, float spillPercentage) {}
    GFX_API void                SwitchDepthRenderSurfaceIntoFastMemoryPlatform(RenderSurfaceBase* rs, bool includeStencil, SurfaceUsage usage, FastMemoryFlags flags, bool copyContents, float spillPercentage) {}
    // resize to larger/smaller placement resource, for dynamic resolution support
    GFX_API void                ResizeRenderSurface(RenderSurfaceBase* rs, float widthScale, float heightScale) {}
    // destroys platform-specific part of render surface
    GFX_API void                DestroyRenderSurfacePlatform(RenderSurfaceBase* rs) GFX_PURE;
    // creates the platform specific part of a stencil view associated with a given depth surface
    GFX_API bool                CreateStencilViewPlatform(TextureID textureID, RenderSurfaceBase* rs, GraphicsFormat stencilFormat) { return false; }
    // destroys the platform specific part of a stencil view
    GFX_API void                DestroyStencilViewPlatform(TextureID textureID, RenderSurfaceBase* rs) {}
    // copy render surface desc (will copy platform-specific parts too)
    GFX_API void                CopyRenderSurfaceDesc(RenderSurfaceBase* dst, const RenderSurfaceBase* src);
    // helper: alloc+copy
    RenderSurfaceBase*          CloneRenderSurface(RenderSurfaceBase* src);

    GFX_API void                AliasRenderSurfacePlatform(RenderSurfaceBase* rs, TextureID origTexID);

    // Creates a new render surface that will share the backing store of the source render surface.
    RenderSurfaceBase*          AliasRenderSurface(TextureID newTextureID, RenderSurfaceBase* src);

    GFX_API void SetSurfaceFlags(RenderSurfaceHandle surf, UInt32 flags) {}  // empty impl for most platforms

    GFX_API TextureID   CreateTextureID(); // persists across devices (if device switching is supported)
    GFX_API void        CreateTextureIdIfNeeded(RenderSurfaceBase* rs) {}
    TextureID           CreateAutoFreedTextureID(); // automatically freed on this device's destruction
    GFX_API void        FreeTextureID(TextureID texture);

    GFX_API void        RegisterNativeTexture(TextureID texture, intptr_t nativeTex, TextureDimension dim);
    GFX_API void        UnregisterNativeTexture(TextureID texture);

    // The 2 textures must match the texture dimension passed for this to function correctly
    GFX_API void        TransferNativeTexture(TextureDimension textureDimension, TextureID targetTextureID, TextureID sourceTextureID);

    GFX_API bool CanCreateTexture2DThreaded(TextureID tid, TextureDimension dimension, int width, int height, GraphicsFormat format, ThreadedTextureCreationSettings& threadedTextureCreationSettings) { return false; }
    GFX_API TextureCreateData* CreateTexture2DThreaded(TextureID tid, TextureDimension dimension, const UInt8* srcData, int srcSize, int width, int height, GraphicsFormat format, int mipCount, TextureUploadFlags uploadFlags, TextureUsageMode usageMode) { return NULL; }

    GFX_API bool AcquireTexture2DUploadMemory(TextureCreateData& textureCreateData, TextureUploadMemory*& uploadMemory) { uploadMemory = NULL; return false; }
    GFX_API void CopyToTextureMemory2DThreaded(TextureCreateData* textureCreateData, TextureUploadMemory& uploadMemory) {}
    GFX_API void ReleaseTexture2DUploadMemory(TextureUploadMemory*& uploadMemory) {}
    GFX_API void IntegrateThreadedTexture(TextureCreateData* textureCreateData) {}
    static bool         IsCreateTextureThreadedEnabled() { return (GetGraphicsCaps().threadedResourceCreationSupport && s_ThreadedTextureCreationEnabled); }
    static void         EnableCreateTextureThreaded(bool enabled) { s_ThreadedTextureCreationEnabled = enabled; }

    GFX_API void UploadTexture2D(TextureID texture, TextureDimension dimension, const UInt8* srcData, int srcSize, int width, int height, GraphicsFormat format, int mipCount, TextureUploadFlags uploadFlags, TextureUsageMode usageMode) GFX_PURE;
    GFX_API void UploadTextureSubData2D(TextureID texture, const UInt8* srcData, int srcSize, int mipLevel, int x, int y, int width, int height, GraphicsFormat format, TextureUploadFlags uploadFlags) GFX_PURE;
    GFX_API void UploadTextureCube(TextureID texture, const UInt8* srcData, int srcSize, int faceDataSize, int size, GraphicsFormat format, int mipCount, TextureUploadFlags uploadFlags) GFX_PURE;
    GFX_API void UploadTexture3D(TextureID texture, const UInt8* srcData, int srcSize, int width, int height, int depth, GraphicsFormat format, int mipCount, TextureUploadFlags uploadFlags) GFX_PURE;
    GFX_API void DeleteTexture(TextureID texture) GFX_PURE;

    GFX_API void UploadTexture2DArray(TextureID texture, const UInt8* srcData, size_t elementSize, int width, int height, int depth, GraphicsFormat format, int mipCount, TextureUploadFlags uploadFlags) {}  // empty impl for devices that don't support them
    GFX_API void UploadTextureCubeArray(TextureID texture, const UInt8* srcData, size_t elementSize, int width, int numCubeMaps, GraphicsFormat format, int mipCount, TextureUploadFlags uploadFlags) {}  // empty impl for devices that don't support them

    GFX_API void GenerateRenderSurfaceMips(RenderSurfaceBase* rs) {}  // empty impl for devices that don't support them

    // Note: Error checking for mip/element indices is done from the calling code
    GFX_API void CopyTexture(TextureID src, TextureID dst) {}  // empty impl in base class
    GFX_API void CopyTexture(TextureID src, int srcElement, int srcMip, int srcMipCount, TextureID dst, int dstElement, int dstMip, int dstMipCount) {}  // empty impl in base class
    GFX_API void CopyTexture(TextureID src, int srcElement, int srcMip, int srcMipCount, int srcX, int srcY, int srcZ, int srcWidth, int srcHeight, int srcDepth, TextureID dst, int dstElement, int dstMip, int dstMipCount, int dstX, int dstY, int dstZ) {}  // empty impl in base class

    // Create a sparse ("tiled") texture.
    GFX_API SparseTextureInfo CreateSparseTexture(TextureID texture, int width, int height, GraphicsFormat format, int mipCount)
    {
        SparseTextureInfo info = {1, 1};
        return info;
    }

    // Upload data for a single sparse texture tile. Unload the tile if data pointer is null.
    GFX_API void UploadSparseTextureTile(TextureID texture, int tileX, int tileY, int mip, const UInt8* srcData, int srcSize, int srcPitch) {}

    GFX_API void DebugMessage(const char* msg);

    GFX_API GfxDevicePresentMode GetPresentMode() GFX_PURE;

    // when commandline argument -batchmode is used, we never call present, which causes the resources to be leaked for the frame.
    // this one is used to notify the gfxdevice that the frame resources needs to be recycled.
    GFX_API void    EndBatchModeUpdate() {}

    GFX_API void    BeginFrame() GFX_PURE;
    GFX_API void    EndFrame() GFX_PURE;
    GFX_API void    ExecuteCallback(GfxDeviceCallback callback) { (*callback)(*this, kGfxDeviceCallbackWorker); }
    inline bool     IsInsideFrame() const { return m_GfxContextData.GetInsideFrame(); }
    inline void     SetInsideFrame(bool v) { m_GfxContextData.SetInsideFrame(v); }
    GFX_API void    PresentFrame() GFX_PURE;
    GFX_API void    PresentFrame(ShaderChannelMask /*blitChannels*/) { PresentFrame(); }
    // Check if device is in valid state. E.g. lost device on D3D9; in this case all rendering should be skipped.
    GFX_API bool    IsValidState() { return true; }
    GFX_API bool    HandleInvalidState() { return true; }

    // Asynchronously send all queued-up commands in the GPU driver's command buffer when this command is executed.  Does not block or flush Unity's command queue.
    GFX_API void Flush() {}
    // Submit all commands in Unity's command queue.  Fully finish any queued-up rendering (including on the GPU).  This blocks until all commands are submitted to the driver and executed.
    GFX_API void    FinishRendering() GFX_PURE;
    // Insert CPU fence into command queue (if threaded)
    GFX_API UInt32  InsertCPUFence() { return 0; }
    // Without blocking, check if the CPU fence has been passed by the worker thread (if threaded)
    GFX_API bool    IsCPUFencePassed(UInt32 fence) const { return true; }
    // Finish any threaded commands before CPU fence
    GFX_API void    WaitOnCPUFence(UInt32 /*fence*/) {}
    GFX_API void    RecreatePrimarySwapchain() {}

    // Are we recording graphics commands?
    inline bool     IsRecording() const
    {
#if !GFX_SUPPORTS_CLIENT_WORKER_GRAPHICS_THREADS
        DebugAssert(CurrentThread::IsMainThread());
#endif
        return m_IsRecording;
    }

    // Does this device derive from GfxThreadableDevice?
    inline bool     IsThreadable() const { return m_IsThreadable; }

    // Acquire thread ownership on the calling thread. Worker releases ownership.
    GFX_API void    AcquireThreadOwnership() {}
    // Release thread ownership on the calling thread. Worker acquires ownership.
    GFX_API void    ReleaseThreadOwnership() {}

    // Immediate mode rendering
    GFX_API void    ImmediateVertex(float x, float y, float z);
    GFX_API void    ImmediateNormal(float x, float y, float z);
    GFX_API void    ImmediateColor(float r, float g, float b, float a);
    GFX_API void    ImmediateTexCoordAll(float x, float y, float z);
    GFX_API void    ImmediateTexCoord(int unit, float x, float y, float z);
    GFX_API void    ImmediateBegin(GfxPrimitiveType type, VertexInputMasks inputMasks);
    GFX_API void    ImmediateEnd();

    // Recording display lists
#if GFX_SUPPORTS_DISPLAY_LISTS
    GFX_API bool    BeginRecording() { return false; }
    GFX_API bool    EndRecording(GfxDisplayList** outDisplayList, const ShaderPropertySheet& globalProperties) { return false; }
#endif

    // Capturing screen shots / blits
    GFX_API bool    CaptureScreenshot(int left, int bottom, int width, int height, UInt8* rgba32) GFX_PURE;
    GFX_API bool    ReadbackImage(ImageReference& image, int left, int bottom, int width, int height, int destX, int destY) GFX_PURE;
    GFX_API void    GrabIntoRenderTexture(RenderSurfaceHandle rs, RenderSurfaceHandle rd, int x, int y, int width, int height) GFX_PURE;

    // Any housekeeping around draw calls
    GFX_API void    BeforeDrawCall() GFX_PURE;
    GFX_API void    AfterDrawCall() {}

    GFX_API void    SetActiveContext(void* /*ctx*/) {}

    GFX_API void    ResetFrameStats();
    GFX_API void    BeginFrameStats();
    GFX_API void    EndFrameStats();
    GFX_API void    SaveDrawStats();
    GFX_API void    RestoreDrawStats();
    GFX_API void    SynchronizeStats();

    GFX_API void    SetGlobalDepthBias(float bias, float slopeBias) { m_GfxContextData.SetGlobalDepthBias(bias); m_GfxContextData.SetGlobalSlopeDepthBias(slopeBias); }
    GFX_API void    GetGlobalDepthBias(float& bias, float& slopeBias) const { bias = m_GfxContextData.GetGlobalDepthBias(); slopeBias = m_GfxContextData.GetGlobalSlopeDepthBias(); }

    static void     OnProfilerFrameChanged(UInt32 frameIndex, void* context);

    GFX_API void    BeginProfileEvent(profiling::Marker* info) {}
    GFX_API void    EndProfileEvent(profiling::Marker* info) {}
    GFX_API void    SetDeviceIdleProfilingMarker(profiling::Marker* info) {}
    GFX_API void    ProfileControl(GfxProfileControl /*ctrl*/, unsigned /*param*/) {}
    GFX_API HDROutputSettings* CreateHDROutputSettings() { return nullptr; }

    GFX_API GfxTimerQuery*      CreateTimerQuery() { return NULL; }
    GFX_API void                DeleteTimerQuery(GfxTimerQuery* query) {}
    GFX_API void                BeginTimerQueries() {}
    GFX_API void                EndTimerQueries() {}
    GFX_API bool                TimerQueriesIsActive() { return false; }

#if GFX_HAS_OBJECT_LABEL
    GFX_API void    SetTextureName(TextureID /*tex*/, const char* /*name*/)                 {}
    GFX_API void    SetRenderSurfaceName(RenderSurfaceBase* /*rs*/, const char* /*name*/)   {}
    GFX_API void    SetBufferName(GfxBuffer* /*vb*/, const char* /*name*/)                  {}
    GFX_API void    SetGpuProgramName(GpuProgram* /*prog*/, const char* /*name*/)           {}
#endif //#if GFX_HAS_OBJECT_LABEL

    // Editor-only stuff
#if UNITY_EDITOR
    void DrawUserPrimitives(GfxPrimitiveType type, int vertexCount, ShaderChannelMask vertexChannels, VertexInputMasks vertexInput, const void* data, int stride);

    GFX_API GfxDeviceWindow* CreateGfxWindow(GfxPlatformWindowHandle window, int width, int height, DepthBufferFormat depthFormat) GFX_PURE;

    inline bool     IsUsingAsyncDummyShader() const { return m_GfxContextData.GetUsingAsyncDummyShader(); }
    inline void     SetUsingAsyncDummyShader(bool v) { m_GfxContextData.SetUsingAsyncDummyShader(v); }
#endif

#if FRAME_DEBUGGER_AVAILABLE
    GFX_API void SetNextDrawCallType(GfxDrawCallType type) {} // overridden in GfxDeviceClient when frame debugger is enabled
    GFX_API void SetNextComputeInfo(InstanceID csInstanceID, const ShaderLab::FastPropertyName& kernelName, int threadGroupsX, int threadGroupsY, int threadGroupsZ, ShaderPropertySheet& props) {} // overridden in GfxDeviceClient when frame debugger is enabled
    GFX_API void SetNextShaderInfo(InstanceID shaderInstanceID, int subshaderIndex, int passIndex, const ShaderLab::Pass& pass) {}
    GFX_API void SetNextDrawCallInfo(InstanceID componentInstanceID, InstanceID meshInstanceID, int meshSubset) {}
    GFX_API void BeginGameViewRenderingOffscreen() {}
    GFX_API void EndGameViewRenderingOffscreen() {}

    GFX_API void BeginGameViewRendering() {}
    GFX_API void EndGameViewRendering() {}
#else
    inline  void SetNextDrawCallType(GfxDrawCallType) {} // Non-virtual so this doesn't cost anything
    inline  void SetNextComputeInfo(InstanceID, const ShaderLab::FastPropertyName&, int, int, int) {}
#endif // #if FRAME_DEBUGGER_AVAILABLE

    static void CommonReloadResources(UInt32 flags);

    // These return native api device/texture/buffer.
    GFX_API void* GetNativeGfxDevice() { return NULL; }
    GFX_API void* GetNativeTexturePointer(TextureID /*id*/) { return NULL; }
    GFX_API void* GetNativeBufferPointer(GfxBuffer* buffer) { return NULL; }

#if GFX_SUPPORTS_RENDERING_EXT_PLUGIN
    GFX_API void InsertCustomMarkerCallbackAndData(UnityRenderingEventAndData callback, int eventId, void* data, size_t dataSize = 0) { callback(eventId, data); }
    GFX_API void InsertCustomBlitCallbackAndData(UnityRenderingEventAndData callback, UnityRenderingExtCustomBlitParams& data) { InsertCustomMarkerCallbackAndData(callback, kUnityRenderingExtEventCustomBlit, &data); }
    GFX_API void InsertPluginTextureUpdateCallback(UnityRenderingEventAndData callback, UnityRenderingExtTextureUpdateParamsInternal& data) { ErrorString("IssuePluginCustomTextureUpdate is not supported by this GfxDevice"); }
#endif

    GFX_API void InsertCustomMarker(int eventId);  // Obsolete
    GFX_API void InsertCustomMarkerCallback(UnityRenderingEvent callback, int eventId);

    GFX_API ComputeBufferID CreateComputeBufferID();
    GFX_API void FreeComputeBufferID(ComputeBufferID id);

    GFX_API void SetComputeBufferData(GfxBuffer* /*buffer*/, const void* /*data*/, size_t /*size*/, size_t /*dstOffset*/) {}
    GFX_API void GetComputeBufferData(GfxBuffer* /*buffer*/, void* /*dest*/, size_t /*destSize*/, size_t /*srcOffset*/) {}
    GFX_API void SetComputeBufferCounterValue(ComputeBufferID /*buffer*/, UInt32 /*counterValue*/) {}
    GFX_API void CopyComputeBufferCount(ComputeBufferID /*srcBuffer*/, ComputeBufferID /*dstBuffer*/, UInt32 /*dstOffset*/) {}

    GFX_API void CopyBuffer(GfxBuffer* /*srcBuffer*/, GfxBuffer* /*dstBuffer*/) { ErrorString("CopyBuffer is not supported by this GfxDevice"); }

    // Async Readback interface

    // Create a device dependent async readback data that holds the status of the request + potential device dependent state
    GFX_API GfxAsyncReadbackData* CreateAsyncReadbackData();

    // Delete an async readback data
    GFX_API void DeleteAsyncReadbackData(GfxAsyncReadbackData* /*data*/);

    // Request a new readback on an async readback data with the given description
    // source, destination and other settings for the readback are passed through the request desc
    // Requests can be made several times on the same async readback data. But only one request at a time is possible, the latest one prevails.
    // Status can be anything after this call and is independent from previous status
    GFX_API void RequestAsyncReadbackData(GfxAsyncReadbackData* /*data*/, const GfxAsyncReadbackRequestDesc& /*request*/);

    // Update the state of the async readback data
    // if no request is pending (status is not kGfxAsyncReadbackPending), it is a no-op
    // wait allows to make this call synchronous and blocks until the request has been fulfilled (data status cannot be kGfxAsyncReadbackPending after a call with wait to true)
    // Status can be anything after this call but cannot be changed to kGfxAsyncReadbackPending
    GFX_API void UpdateAsyncReadbackData(GfxAsyncReadbackData* /*data*/, bool /*wait*/) {}

    GFX_API void SetRandomWriteTargetTexture(int /*index*/, TextureID /*tid*/) {}
    GFX_API void SetRandomWriteTargetBuffer(int /*index*/, ComputeBufferID /*bufferHandle*/) {}
    GFX_API void ClearRandomWriteTargets() {}

    GFX_API bool HasActiveRandomWriteTarget() const  { return false; }

    GFX_API ComputeProgramHandle CreateComputeProgram(const UInt8* /*code*/, size_t /*codeSize*/, const char* /*name*/) { ComputeProgramHandle cp; return cp; }
    GFX_API void DestroyComputeProgram(ComputeProgramHandle& /*cpHandle*/) {}
    GFX_API void ResolveComputeProgramResources(ComputeProgramHandle /*cpHandle*/, ComputeShaderKernel& /*kernel*/, dynamic_array<ComputeShaderCB>& /*constantBuffers*/, dynamic_array<ComputeShaderParam>& /*uniforms*/, bool /*preResolved*/) {}
    GFX_API void CreateComputeConstantBuffers(unsigned /*count*/, const UInt32* /*sizes*/, ConstantBufferHandle* /*outCBs*/) {}
    GFX_API void DestroyComputeConstantBuffers(unsigned /*count*/, ConstantBufferHandle* /*cbs*/) {}
    GFX_API void SetComputeUniform(ComputeProgramHandle /*cpHandle*/, ComputeShaderParam& /*uniform*/, size_t /*byteCount*/, const void* /*data*/) {}

    // Update and bind an array of constant buffers used in the compute shader.
    // If the highest bit of bindPoint is set, the ConstantBufferHandle.object is actually a GfxResourceIDType pointing to ComputeBuffer to be bound in to that slot.
    // In such cases, cbOffset and cbSize represent offset/size within that constant buffer, not in the data buffer, and data upload is skipped for that buffer.
    GFX_API void UpdateComputeConstantBuffers(unsigned /*count*/, ConstantBufferHandle* /*cbs*/, UInt32 /*cbDirty*/, size_t /*dataSize*/, const UInt8* /*data*/, const UInt32* /*cbSizes*/, const UInt32* /*cbOffsets*/, const int* /*bindPoints*/) {}
    GFX_API void UpdateComputeResources(const ComputeShaderResources& /*resources*/) {}
    GFX_API void SetComputeProgram(ComputeProgramHandle /*cpHandle*/) {}
    GFX_API void DispatchComputeProgram(ComputeProgramHandle /*cpHandle*/, unsigned /*threadGroupsX*/, unsigned /*threadGroupsY*/, unsigned /*threadGroupsZ*/) {}
    GFX_API void DispatchComputeProgram(ComputeProgramHandle /*cpHandle*/, ComputeBufferID /*indirectBuffer*/, UInt32 /*argsOffset*/) {}

    GFX_API RayTracingProgramHandle CreateRayTracingProgram(const RayTracingShaderDesc& /*desc*/) { RayTracingProgramHandle rp; return rp; }
    GFX_API void DestroyRayTracingProgram(RayTracingProgramHandle& /*rpHandle*/) {}
    GFX_API void DispatchRayTracingProgram(RayTracingProgramHandle& /*rpHandle*/, const char* /*name*/, UInt32 /*width*/, UInt32 /*height*/, UInt32 /*depth*/) {}
    GFX_API void CreateRayTracingConstantBuffers(unsigned /*count*/, const UInt32* /*sizes*/, ConstantBufferHandle* /*outCBs*/) {}
    GFX_API void DestroyRayTracingConstantBuffers(unsigned /*count*/, ConstantBufferHandle* /*cbs*/) {}
    GFX_API void SetRayTracingGlobalProperties(RayTracingProgramHandle& /*rpHandle*/, const ShaderPropertySheet& /*properties*/) {}
    GFX_API void SetRayTracingShaderResources(RayTracingProgramHandle& /*rpHandle*/, RayTracingShaderResourceScope /*resourceScope*/, const RayTracingShaderFunctionResources& /*resources*/) {}
    GFX_API void SetRayTracingShaderConstantBuffers(RayTracingProgramHandle& /*rpHandle*/, UInt32 /*count*/, ConstantBufferHandle* /*cbs*/, UInt32 /*cbDirty*/, size_t /*dataSize*/, const UInt8* /*data*/, const UInt32* /*cbSizes*/, const UInt32* /*cbOffsets*/, const int* /*bindPoints*/) {}
    GFX_API void SetRayTracingGeometryCount(RayTracingProgramHandle& /*rpHandle*/, UInt32 /*geometryCount*/) {}
    GFX_API void SetGeometryRayTracingShaderThreadable(RayTracingProgramHandle& /*rpHandle*/, UInt32 /*geometryIndex*/, GpuProgram* /*gpuProgram*/, const GpuProgramParameters* /*gpuProgramParameters*/, UInt8 const * const /*paramsBuffer*/) {}
    GFX_API void SetGeometryRayTracingShaderMainThread(RayTracingProgramHandle& /*rpHandle*/, UInt32 /*geometryIndex*/, const ShaderLab::SubProgram* /*subProgram*/, const ShaderPropertySheet* /*localShaderProperties*/, const ShaderPropertySheet* /*globalShaderProperties*/) {}
    GFX_API void BuildRayTracingAccelerationStructures(RayTracingAccelerationStructure& /*as*/) {}
    GFX_API void UpdateRayTracingAccelerationStructures(RayTracingAccelerationStructure& /*as*/) {}

    //Set the type of queue this device is currently targeting (e.g the graphics queue or one of the async compute queues). Default implementation does nothing and assumes the graphics queue is always the target
    GFX_API void SetTargetQueueType(const ComputeQueueType /*queueType*/) {}

    GFX_API void DrawNullGeometry(GfxPrimitiveType /*topology*/, int /*vertexCount*/, int /*instanceCount*/) {}
    GFX_API void DrawNullGeometryIndirect(GfxPrimitiveType /*topology*/, ComputeBufferID /*bufferHandle*/, UInt32 /*bufferOffset*/) {}

    GFX_API void DrawIndexedNullGeometry(GfxPrimitiveType /*topology*/, GfxBuffer* /*indexBuffer*/, int /*indexCount*/, int /*instanceCount*/, int /*startIndex*/) {}
    GFX_API void DrawIndexedNullGeometryIndirect(GfxPrimitiveType /*topology*/, GfxBuffer* /*indexBuffer*/, ComputeBufferID /*bufferHandle*/, UInt32 /*bufferOffset*/) {}

    DepthBufferFormat GetFramebufferDepthFormat() const { return m_FramebufferDepthFormat; }
    void SetFramebufferDepthFormat(DepthBufferFormat depthFormat) { m_FramebufferDepthFormat = depthFormat; }

    // Set the correct texture array index etc. for target eye.
    GFX_API void SetStereoTarget(StereoscopicEye eye) {}
    // Called after rendering is finished on an eye texture.
    GFX_API void EndStereoEye(StereoscopicEye eye) {}
    // Set which eye we are logically rendering. Affects GetStereoActiveEye() and kShaderVecStereoEyeIndex builtin param.
    void SetStereoActiveEye(StereoscopicEye eye);
    StereoscopicEye GetStereoActiveEye() const { return m_GfxContextData.GetStereoActiveEye(); }
    GFX_API void SetSinglePassStereo(SinglePassStereo mode); // By default do nothing
    SinglePassStereo GetSinglePassStereo() const { return m_GfxContextData.GetSinglePassStereo(); }
    GFX_API void SendVRDeviceEvent(UInt32 eventID, UInt32 data = 0);
    void SendVRDeviceEvent(UInt32 eventID, float data);

    int GetDrawIterations() const { return (m_GfxContextData.GetSinglePassStereo() == kSinglePassStereoSideBySide) ? 2 : 1; }
    GFX_API void SetInstanceCountMultiplier(int multiplier) { m_GfxContextData.SetInstanceCountMultiplier(multiplier); }
    int GetInstanceCountMultiplier() const { return m_GfxContextData.GetInstanceCountMultiplier(); }
    bool ShouldIssueDrawCall(MonoOrStereoscopicEye activeSinglePassStereoEye, TargetEyeMask singlePassEyeMask, int iteration) const;

    GFX_API size_t GetMultiGPUCount() { return 1; }
    GFX_API size_t GetCurrentGPU() { return 0; }

#if SUPPORT_MULTIPLE_DISPLAYS
    GFX_API bool ActivateDisplay(const UInt32 /*displayId*/) { return false; }
    GFX_API bool SetDisplayTarget(const UInt32 /*displayId*/) { return false; }
#endif // #if SUPPORT_MULTIPLE_DISPLAYS

    // Upload Asynchronous Data Time Sliced.
    GFX_API void AsyncResourceUpload(const int timeSliceMS, const AsyncUploadManagerSettings& settings);
    GFX_API void SyncAsyncResourceUpload(AsyncFence fence, const AsyncUploadManagerSettings& settings);

    // Renderloop multithreading API
    GFX_API void    StoreContextData(GfxContextData *dest) const;
    GFX_API void    ApplyContextData(const GfxContextData *from);
    GFX_API void    CopyContextDataFrom(const GfxDevice* device);

    // New jobified rendering API
    GFX_API void    UpdateDeviceThreadID(ThreadId threadId) {}
    GFX_API void    ExecuteAsync(int count, GfxDeviceAsyncCommand::Func* func, GfxDeviceAsyncCommand::ArgScratch** scratches, const GfxDeviceAsyncCommand::Arg* arg, const JobFence& depends);
#if UNITY_DEFER_GRAPHICS_JOBS_SCHEDULE
    void ScheduleAsyncJob(AsyncCommandJobFunc* jobFunc, GfxDeviceAsyncCommand* cmd, const JobFence& depends, JobBatchDispatcher& dispatcher);
#else
    JobFence& ScheduleAsyncJob(AsyncCommandJobFunc* jobFunc, GfxDeviceAsyncCommand* cmd, const JobFence& depends, JobBatchDispatcher& dispatcher);
#endif // #if UNITY_DEFER_GRAPHICS_JOBS_SCHEDULE

    // Is it allowed to get a threadable gfx device client.
    GFX_API bool    CanGetPreparedThreadedDeviceClient();

    // Get a threadable gfx device client to record a command list that will be inserted into the main cmd list, at the moment this method was called.
    GFX_API GfxDeviceClient* GetPreparedThreadedDeviceClient();

    // Internal API for worker thread
    GPUParamBuffer&     GetParamsScratchBuffer(void) { return m_ParamsScratchBuffer; }

    GFX_API GfxDeviceGraphicsJobsSyncPoint GetDefaultGraphicsJobsSyncPoint() const { return kGfxDeviceGraphicsJobsSyncPointAfterScriptUpdate; }
    void SetGraphicsJobsSyncPoint(GfxDeviceGraphicsJobsSyncPoint syncPoint) { m_GraphicsJobsSyncPoint = syncPoint; }
    GfxDeviceGraphicsJobsSyncPoint GetGraphicsJobsSyncPoint() const { return m_GraphicsJobsSyncPoint; }
    static void EndGraphicsJobs(GfxDeviceGraphicsJobsSyncPoint currentPoint);

    void SetWaitForPresentSyncPoint(GfxDeviceWaitForPresentSyncPoint syncPoint) { m_WaitForPresentSyncPoint = syncPoint;  }
    GfxDeviceWaitForPresentSyncPoint GetWaitForPresentSyncPoint() const { return m_WaitForPresentSyncPoint; }

    struct DebugSettings
    {
        float sleepAtStartOfGraphicsJobs;
        float sleepBeforeTextureUpload;

        DebugSettings()
            : sleepAtStartOfGraphicsJobs(0.0f)
            , sleepBeforeTextureUpload(0.0f)
        {
        }
    };

    GFX_API void SetDebugSettings(const DebugSettings& settings) { m_DebugSettings = settings; }
    const DebugSettings& GetDebugSettings() const { return m_DebugSettings; }

#if ENABLE_UNIT_TESTS
    GFX_API void PerformTestWithData(const void* data, size_t dataSize) {}
#endif

    GFX_API RenderSurfaceHandle CreateRenderSurfaceWrapper() { return RenderSurfaceHandle(); }
    GFX_API void UpdateRenderSurfaceWrapper(RenderSurfaceHandle& platformSurface, RenderSurfaceBase* updateToThisSurface, DepthBufferFormat depthFormat) { platformSurface.object = updateToThisSurface; }
    GFX_API void CleanupRenderSurfaceWrapper(RenderSurfaceHandle& platformSurface) { if (platformSurface.IsValid()) platformSurface.Reset(); }

    MemLabelId GetMemoryLabel() { return m_MemoryLabel; }

    // Create a PlatformNativeGPUFence appropriate for fencing against the last batch issued and write it to the GPUFenceInternals instance passed.
    // We need whatever data is stored in PlatformNativeGPUFence to allow us to subsequently instruct the GPU to wait for that fence to pass (via WaitOnGPUFence), and to be able to determine on the CPU if the fence has passed (via HasGPUFencePassed).
    GFX_API void CreateGPUFence(GPUFenceInternals* fence, GPUFenceType ftype, SynchronisationStage stage) {}

    // Have the GPU wait on the PlatformNativeGPUFence contained within the provided GPUFenceInternals instance. Only valid for fences created with kGPUFenceTypeAsync type.
    GFX_API void WaitOnGPUFence(GPUFenceInternals* fence, SynchronisationStage stage) {}

    // Destroy the given PlatformNativeGPUFence or release to a pool as appropriate.
    static  void ReleaseGPUFence(PlatformNativeGPUFence fence);

    // Evaluate on the CPU if the provided PlatformNativeGPUFence has passed on the GPU.
    static  bool HasGPUFencePassed(const PlatformNativeGPUFence fence);

    bool IsInsideRenderPass() const { return m_ActiveSubPassIndex != -1; }
    GfxBuffer*     GetProceduralQuadIndexBuffer(int quadCount);

    // The end of frame callbacks are called once per frame from EndDynamicVBOFrame.
    void AddPresentFrameCallback(GfxDeviceCallback callback);
    GFX_API void SubmitPresentFrameCallbacks();

    GFX_API void SetPaperWhiteInNits(float paperWhiteInNits);
protected:
    // Platform-dependent part of SetRenderTargets
    GFX_API void SetRenderTargetsImpl(const GfxRenderTargetSetup& rt) GFX_PURE;

    // Platform dependent implementation of GPU fence related functions, needs to be thread-safe as may get called off the main thread
    GFX_API bool HasGPUFencePassedImpl(const PlatformNativeGPUFence fence) const { return true; }
    GFX_API void ReleaseGPUFenceImpl(PlatformNativeGPUFence fence) const {}

    // Overrideable renderpass implementation
    GFX_API void BeginRenderPassImpl(const RenderPassSetup &setup);
    GFX_API void NextSubPassImpl();
    GFX_API void EndRenderPassImpl();

protected:
    void OnCreate();
    void OnDelete();
    void OnCreateBuffer(GfxBuffer* buffer);
    void OnDeleteBuffer(GfxBuffer* buffer);
    void CreateDefaultVertexBuffers();
    void CleanupSharedBuffers();


    static const int kDefaultMaxBufferedFrames = 2; // -1 means no limiting, default is 2

    MemLabelId          m_MemoryLabel;

    // Mutable state
    GfxContextData  m_GfxContextData;

    GfxDeviceStats      m_Stats;
    GfxDeviceStats      m_SavedStats;
    bool                m_IsRecording;
    bool                m_IsThreadable;
    GfxBuffer*          m_DefaultVertexBuffers[kGfxDefaultVertexBufferCount];
    GfxBuffer*          m_ProceduralQuadIndexBuffer;
    GfxBuffer*          m_ProceduralQuadIndexBuffer32;
    int                 m_ProceduralQuadMax32;
    int                 m_StencilRefWhenStencilWasSkipped;
    ViewProjMatrixNeedApplyFlag m_ViewProjMatrixNeedApplyFlags;

    // Immutable data
    GfxDeviceRenderer   m_Renderer;
    bool                m_IsDeviceClient;
    int                 m_MaxBufferedFrames;
    DepthBufferFormat   m_FramebufferDepthFormat;

    RenderSurfaceHandle m_BackBufferColor;
    RenderSurfaceHandle m_BackBufferDepth;

    // Utility class for immediate rendering
    DrawImmediate*      m_DrawImmediate;
    FrameTimingManager* m_FrameTimingManager;

    HDROutputSettings* m_HDROutputSettings;

    static GeometryJobTasks s_GeometryJobs;

    dynamic_array<JobFence> m_AsyncJobFences;
#if UNITY_DEFER_GRAPHICS_JOBS_SCHEDULE
    dynamic_array<AsyncCommandJobFunc*> m_AsyncJobFunctions;
    dynamic_array<GfxDeviceAsyncCommand*> m_AsyncJobCommands;
    dynamic_array<JobFence> m_AsyncJobDepends;
#endif // #if UNITY_DEFER_GRAPHICS_JOBS_SCHEDULE

    // Currently active subpass index within the renderpass, or -1 if not inside a Begin/EndRenderPass()
    int                 m_ActiveSubPassIndex;
    RenderPassSetup     m_ActiveRenderPass;

    DynamicVBO*         m_DynamicVBO;

    dynamic_array<GfxDeviceCallback> m_PresentFrameCallbacks;

private:
    static UInt32 CalculateDefaultVertexBufferStride(GfxDefaultVertexBufferType type);
    GfxBuffer* CreateDefaultVertexBuffer(GfxDefaultVertexBufferType type, size_t size);

    void CleanupTextureIDs();

    GPUParamBuffer          m_ParamsScratchBuffer;
    GfxBufferList*          m_BufferList;
    VertexDeclarationMRUCache m_VertexDeclMRUCaches[kVertexDeclarationMRUCacheCount];

    dynamic_array<TextureID> m_AutoFreedTextureIDs;

    static volatile int ms_ComputeBufferIDGenerator;

    typedef std::map<TextureID, size_t> TextureIDToSizeMap;
    TextureIDToSizeMap m_TextureSizes;

    // Fallback implementation of RenderPasses needs to calculate the load/store actions for each color attachment plus depth in each subpass
    struct SubPassActions
    {
        dynamic_array<GfxRTLoadAction> loadActions;
        dynamic_array<GfxRTStoreAction> storeActions;
        GfxRTLoadAction depthLoad;
        GfxRTStoreAction depthStore;
    };
    // This cannot be a dynamic_array because SubPassActions is not POD
    std::vector<SubPassActions> m_RenderPassActions;

    // Storage for the fast propertynames for the input framebuffer names, in the fallback impl.
    ShaderLab::FastPropertyName m_InputFBNames[8];

    GfxDeviceGraphicsJobsSyncPoint m_GraphicsJobsSyncPoint;
    GfxDeviceWaitForPresentSyncPoint m_WaitForPresentSyncPoint;

    DebugSettings m_DebugSettings;

    struct DynamicBatch
    {
        bool isActive;
        TimeFormat startTime;
        ShaderChannelMask shaderChannels;
        ShaderChannelMask channelMask;
        size_t maxVertices;
        size_t maxIndices;
        size_t vertexCount;
        size_t indexCount;
        size_t meshCount;
        GfxPrimitiveType topology;
        size_t destStride;
        DynamicVBOMemory vboMemory;
        VertexDeclaration* vertexDecl;
    } m_DynamicBatch;

    static bool s_ThreadedTextureCreationEnabled;

public:
    static void FlipRectForSurface(const RenderSurfaceBase* surface, RectInt& rect);
    static void FlipRectForHeight(int height, RectInt& rect);
};
```

