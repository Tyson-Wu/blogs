---
title: URP Water
date: 2021-04-06 10:01:00
categories:
- [Unity, URP]
tags:
- Render
- Shader
---


``` C#
public WaterFeature :ScriptableRendererFeature
{
    [Serializable]
    public class Settings
    {
        public string TextureName = "_NormalGrabTexture";
        public LayerMask LayerMask;
        public Material normalGrabMaterial;
        public int normalGrabMaterialPassIndex = 0;
        public LayerMask WaterLayerMask;
    }
    class NormalGrabPass : ScriptableRenderPass
    {
        Settings settings;
        Material normalGrabMaterial;
        int normalGrabMaterialPassIndex = 0;
        RenderTargetHandle tempNormalTarget;
        List<ShaderTagId> m_ShaderTagIdList = new List<ShaderTagId>();
        FilteringSettings m_FilteringSettings;
        RenderStateBlock m_RenderStateBlock;
        public NormalGrabPass(Settings settings)
        {
            this.settings = settings;
            normalGrabMaterial = new Material(Shader.Find("Unlit/NormalTexture"));
            normalGrabMaterialPassIndex = 0;
            renderPassEvent = RenderPassEvent.AfterRenderingOpaques;
            tempNormalTarget.Init(settings.TextureName);
            m_ShaderTagIdList.Add(new ShaderTagId("UniversalForward"));
            m_ShaderTagIdList.Add(new ShaderTagId("LightweightForward"));
            m_ShaderTagIdList.Add(new ShaderTagId("SPRDefaultUnlit"));
            m_FilteringSettings = new FilteringSettings(RenderQueueRange.all, settings.LayerMask);
            m_RenderStateBlock = new RenderStateBlock(RenderStateMask.Nothing);
        }
        public overide void Configure(CommandBuffer cmd, RenderTextureDescriptor cameraTextureDescriptor)
        {
            cmd.GetTemporaryRT(tempNormalTarget.id, cameraTextureDescriptor);
            cmd.SetGlobalTexture(settings.TextureName, tempNormalTarget.Identifier());
            ConfigureTarget(tempNormalTarget.Identifier()); //sceen view
            cmd.SetRenderTarget(tempNormalTarget.Identifier()); // game view
        }
        public overrider void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get();
            context.ExecuteCommandBuffer(cmd);
            cmd.Clear();
            Matrix4x4 viewMatrix = renderingData.cameraData.camera.worldToCameraMatrix;
            Vector3 normal = viewMatrix.MultiplyVector(new Vector3(0,1,0));
            cmd.ClearRenderTarget(false, true, new Color(normal.x, normal.y, normal.z));
            context.ExecuteCommandBuffer(cmd);
            cmd.Clear();

            DrawingSettings drawingSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, SortingCriteria.CommonOpaque);
            drawingSettings.overrideMaterial = normalGrabMaterial;
            drawingSettings.overrideMaterialPassIndex = normalGrabMaterialPassIndex;
            contex.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref m_FilteringSettings, ref m_RenderStateBlock);

            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
        public override void FrameCleanup(CommandBuffer cmd)
        {
            cmd.ReleaseTemporaryRT(tempNormalTarget.id);
        }
    }
    class WaterPass : ScriptableRenderPass
    {
        List<ShaderTagId> m_ShaderTagIdList = new List<ShaderTagId>();
        RenderStateBlock m_RenderStateBlock;
        FilteringSettings m_FilteringSettings;
        public WaterPass(Settings settings)
        {
            renderPassEvent = RenderPassEvent.BeforeRenderingTransparents;
            m_ShaderTagIdList.Add(new ShaderTagId("Water"));
            m_RenderStateBlock = new RenderStateBlock(RenderStateMask.Nothing);
            m_FilteringSettings = new FilteringSettings(RenderQueueRange.transparent, settings.WaterLayerMask);
        }
        public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
        {
            CommandBuffer cmd = CommandBufferPool.Get();
            context.ExecuteCommandBuffer(cmd);
            cmd.Clear();

            DrawingSettings drawingSettings = CreateDrawingSettings(m_ShaderTagIdList, ref renderingData, SortingCriteria.CommonOpaque);
            contex.DrawRenderers(renderingData.cullResults, ref drawingSettings, ref m_FilteringSettings, ref m_RenderStateBlock);

            context.ExecuteCommandBuffer(cmd);
            CommandBufferPool.Release(cmd);
        }
    }
    [SerializeField] Settings settings = new Settings();
    NormalGrabPass normalGrabPass;
    WaterPass waterPass;
    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        renderer.EnqueuePass(normalGrabPass);
        renderer.EnqueuePass(waterPass);
    }
    public override void Create()
    {
        normalGrabPass = new NormalGrabPass(settings);
        waterPass = new WaterPass(settings);
    }
}
```

``` C#
Shader "Unlit/NormalTexture"
{
    Properties{}
    SubShader
    {
        Tags {"RenderType" = "Opaque" "RenderPipeline" = "UniversalPipeline"}
        ZWrite Off
        ZTest Always
        Pass
        {
            Name "NormalTexture"
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Corel.hlsl"
            struct VertexInput
            {
                float4 posotionCS : POSITION;
                float3 normal : NORMAL;
            };
            struct FragInput
            {
                float4 positionCS : SV_POSITION;
                float3 normal : TEXCOORD0;
            };
            FragInput vert(VertexInput i)
            {
                FragInput o;
                float3 worldPos = TransformObjectToWorld(i.positionOS.xyz);
                float4 clipPos = TransformWorldToHClip(worldPos);
                o.positionCS = clipPos;
                float3 worldNormal = TransformObjectToWorldNormal(i.normal);
                o.normal = TransformWorldToViewDir(worldNormal);
                return o;
            }
            half4 frag(FragInput i) : SV_TARGET
            {
                return half4(i.normal, 1);
            }
            ENDHLSL
        }
    }
}
```


```c#
Shader "Unlit/ToonWater"
{
    Properties
    {
        _DepthGradientShallow("Depth Gradient Shallow", Color) = (0,3,0.8,0.9,0.4)
        _DepthGradientDeep("Depth Gradient Deep", Color) = (0.08, 0.4, 0.1, 0.7)
        _DepthMaxDistance("Depth Maximum Distance", Float) = 1

        _SurfaceNoise("Surface Noise", 2D) = "white"{}
        _SurfaceNoiseCutoff("Surface Noise Cutoff", Range(0,1)) = 0.77
        _FoamColor("Foam Color", Color) = (1,1,1,1)
        _FoamMaxDistance("Foam Max Distance", Float) = 1
        _FoamMinDistance("Foam Min Distance", Float) = 0.4
        _SurfaceNoiseScroll("Surface Noise Scroll Amount", Vector) = (0.03, 0.03, 0, 0)
        _SurfaceDistortion("Surface Distortion", 2D) = "white"{}
        _SurfaceDistortionAmount("Surface Distortion Amount", Range(0,1)) = 0.27
    }
    SubShader
    {
        Tags { "RenderType" = "Transparent" "Queue" = "Transparent" "RenderPipeline" = "UniversalPipeline" }
        Blend SrcAlpha OneMinusSrcAlpha
        ZWrite Off
        Pass
        {
            Tags {"LightMode" = "Water" }
            HLSLPROGRAM
            #define SMOOTHSTEP_AA 0.01
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"
            float4 _DepthGradientShallow;
            float4 _DepthGradientDeep;
            float _DepthMaxDistance;
            TEXTURE2D(_SurfaceNoise);   SAMPLER(sampler_SurfaceNoise);
            float4 _SurfaceNoise_ST;
            float _SurfaceNoiseCutoff;
            float _FoamMaxDistance;
            float _FoamMinDistance;
            float2 _SurfaceNoiseScroll;
            TEXTURE2D(_SurfaceDistortion);  SAMPLER(sampler_SurfaceDistortion);
            float4 _SurfaceDistortion_ST;
            float _SurfaceDistortionAmount;
            TEXTURE2D(_NormalGrabTexture); SAMPLER(sampler_NormalGrabTexture);
            float4 _FoamColor;
            struct VertexInput
            {
                float4 positionOS : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };
            struct FragInput
            {
                float4 positionCS : SV_POSITION;
                float4 screenPos : TEXCOORD0;
                float3 viewPos : TEXCOORD1;
                float2 noiseUV : TEXCOORD2;
                float2 distortUV : TEXCOORD3;
                float3 viewNormal : NORMAL;
            };
            FragInput vert(VertexInput i)
            {
                FragInput o;
                float3 worldPos = TransformObjectToWorld(i.positionOS.xyz);
                float4 clipPos= TransformWorldToHClip(worldPos);
                float4 screenPos = ConputeScreenPos(clipPos);
                float3 viewPos = TransformWorldToView(worldPos);
                o.positionCS = clipPos;
                o.screenPos = screenPos;
                o.viewPos = viewPos;
                o.noiseUV = TRANSFORM_TEX(i.uv, _SurfaceNoise);
                o.distortUV = TRANSORM_TEX(i.uv, _SurfaceDistortion);
                float3 worldNormal = TransformObjectToWorldNormal(i.normal);
                o.viewNormal = TransformWorldToViewDir(worldNormal);
                return o;
            }
            float4 AlpahBelnd(float4 top, float4 bottom)
            {
                float3 color = top.rgb * top.a + (1 - top.a) * bottom.rgb;
                float alpha = top.a + bottom.a * (1 - top.a);
                return float4(color, alpha);
            }
            half4 frag(FragInput i) : SV_TARGET
            {
                half4 color = float4(1,1,1,1);
                float2 screenUV = i.screenPos.xy / i.screenPos.w;
                float sceeneRawDepth = SampleSceneDepth(sceenUV);
                float sceenEyeDepth = LinearEyeDepth(sceneRawDepth, _ZBufferParams);
                float depthDifference = sceneEyeDepth - i.scenePos.w;

                float waterDephtDifference01 = saturate(depthDifference / _DepthMaxDistance);
                float4 waterColor = lerp(_DepthGradientShallow, _DepthGradientDeep, waterDepthDifference01);

                float2 distortSample = SAMPLE_TEXTURE2D(_SurfaceDistortion, sampler_SurfaceDistortion, i.distorUV).xy;
                distortSample = (distortSample * 2 - 1) * _SurfaceDIstortionAmount;

                float noiseUV = float2(i.noiseUV.x + _Time.y * _SurfaceNoiseScroll.x,
                                        i.noiseUV.y + _Time.y * _SurfaceNoiseScroll.y);
                float surfaceNoiseSample = SAMPLE_TEXTURE2D(_SurfaceNoise, sampler_SurfaceNoise, noiseUV);

                float3 existingNormal = SAMPLE_TEXTURE2D(_NormalGrabTexture, sampler_NormalGrabTexture, screenUV).rgb;
                float normalDot = saturate(do(existingNormal, i.viewNormal));
                float foamDistance = lerp(_FoamMaxDistance, _FoamMinDistance, normalDot);
                float foamDipthDifference01 = saturate(depthDifference / foamDistance);

                float surfaceNoiseCutoff = foamDepthDifference01 * _surfaceNoiseCutoff;
                float surfaceNoise = smoothstep(surfaceNoiseCutoff - SMOOTHSTEP_AA, surfaceNoiseCutoff + SMOOTHSTEP_AA, surfaceNoiseSample);

                float4 surfaceNoiseColor = _FoamColor * surfaceNoise;
                color = ALphaBlend(surfaceNoiseColor, waterColor);
                return color;
            }
            ENDHLSL
        }
    }
}
```

