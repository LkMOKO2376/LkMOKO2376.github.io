---
layout: post
title:  "[UE] 尝试优化虚幻大气颜色和Mie散射效果"
date:   2026-10-5 3:34:00 +0800
categories: post
math: true
image: /assets/images/CustomRayleighColorSpace/imgH.png
---

之前在[Jasmin Patry对马岛的分享]发现一个简单好用的大气渲染颜色改进方法。
![alt text](/assets/images/LearnAtmosphereSpectralRenderApprox/screenshot-20260329-121356.png)
![screenshot-20260329-220922.png](/assets/images/CustomRayleighColorSpace/screenshot-20260329-220922.png)
用了一个自定义ColorSpace 让结果的颜色更加接近光谱渲染。看着效果不错于是打算在虚幻试试。但他们只给了瑞利散射的系数和色彩空间，Ozone的吸收系数要自己再想办法拟合一下（蓝色肥鱼神力巨大帮助搞了可微渲染，方案是基于[Eradiate]的光谱渲染对Sebastien Hillaire实现的参数拟合，Eradiate是基于[mitsuba3]的）。 

另外还有GT7天空渲染[Realistic Real-time Sky Dome Rendering in Gran Turismo 7] 对于Mie散射的改进(之前发过笔记)，虽然他们的离线计算比较复杂，不过以效果为导向其实能用Nvidia的这个公式简单近似一下[An Approximate Mie Scattering Function for Fog and Cloud Rendering]。或者单独搞个Lut像虚幻的体积云渲染一样，目前还是用的NV文章的公式。也用了GT7的一些图来做对比，效果图全在文章结尾。

后文代码是比较早的时候手写的，后来又大量用AI优化和修了一些bug，以及补上了臭氧的吸收拟合。

本文并不考虑性能优化，性能上是负优化，有些背景知识（其实只是改也不需要知道太多，本文改的都是比较简单的东西）一个是Sebastien Hillaire大佬的[Physically Based and Scalable Atmosphere in Unreal Engine] 还有 Eric Bruneton 和 Fabrice Neyret的[Precomputed Atmospheric Scattering]


## 修改大气渲染色彩

虚幻引擎里大气的色彩主要由瑞利Rayleigh散射和臭氧Ozone的吸收决定，Mie默认是白色，跟粒子特性相关不详细说了。因此优化色彩主要是改这两个的系数，当然只是为了艺术表现可以随意调整，这里还是想让默认值比较接近真实，作为一个调整的基础。

### UE 源码修改渲染色彩空间

默认UE的源码实现，大气的参数是sRGB空间的，转换到引擎的WokingColorSpace来计算。这样如果WorkingColorSpace变化理论上计算结果会变，即使是一般sRGB下的计算效果也和光谱渲染有区别。
（UE的路径追踪参考大气是有光谱渲染，基础是对齐实时这一套的，渲染结果也比较像，UE的这套参数又部分继承自Bruneton的论文，和网上以及GT7的一些其他光谱渲染结果不太像，这里没有作为参考）

参考对马岛的思路，也为了不过大修改UE的渲染方法，打算是把全部大气计算都转到对马岛这个LMS空间，大气渲染就不受workingColorSpace影响，输入系数本来是sRGB也直接用LMS。

最后在AerialPerspective和SkyView以及RenderSkyAtmosphereRayMarchingPS等把色彩空间转换回WorkingColorSpace。TransmittanceLut用了workingColorSpace来存，因为有些其他模块要用。

**代码没全贴出来**，有点多，大部分写下思路。显示出来缩进有点问题忽略吧

首先是加个ConsoleVariable来控制开关，加在UI也行，这里图方便了
```
//SkyAtmosphereRendering.cpp

static TAutoConsoleVariable<int32> CVarSkyAtmosphereUseGotLmsColorSpace(
	TEXT("r.SkyAtmosphere.UseGotLmsColorSpace"), 0,
	TEXT("Enable the Ghost of Tsushima LMS colorspace for the atmosphere scattering calculation."),
	ECVF_RenderThreadSafe | ECVF_Scalability);
```

做色彩空间转换，系数不转到workingColorSpace，GroundAlbedo也不额外转，就当输入颜色是LMS的，也符合UE对颜色在WorkingColorSpace下不转的设计，相当于大气有个独立的workingColorSpace。不过光源颜色我还是转换了，考虑对齐其他地方的渲染（平行光默认是1没什么影响）。这里要注意改下把ColorSpace的修改加进ComputeAtmosphereVersion，保证修改时Lut更新。

```
// SkyAtmosphereCommonData.cpp

void FAtmosphereSetup::ComputeAtmosphereVersion()
{
	uint32 Crc = 0;
	Crc = FCrc::MemCrc32((void*)&BottomRadiusKm,					sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&TopRadiusKm,						sizeof(float),		Crc);
	Crc = FCrc::MemCrc32((void*)&MultiScatteringFactor,				sizeof(float),		Crc);
	const int32 ColorSpaceMode = GetSkyAtmosphereColorSpaceModeValue();
	Crc = FCrc::MemCrc32((void*)&ColorSpaceMode,					sizeof(int32),		Crc);
	
	......
	
// Rayleigh scattering
{
    RayleighScattering = (SkyAtmosphereComponent.RayleighScattering * SkyAtmosphereComponent.RayleighScatteringScale).GetClamped(0.0f, 1e38f);
    RayleighScattering = bUseGotLmsColorSpace ? RayleighScattering : ConvertCoefficientsFromSRGBToWorkingColorSpace(RayleighScattering);
    
    ......
    
// Mie scattering
{

    MieScattering = (SkyAtmosphereComponent.MieScattering * SkyAtmosphereComponent.MieScatteringScale).GetClamped(0.0f, 1e38f);
    MieScattering = bUseGotLmsColorSpace ? MieScattering : ConvertCoefficientsFromSRGBToWorkingColorSpace(MieScattering);

    MieAbsorption = (SkyAtmosphereComponent.MieAbsorption * SkyAtmosphereComponent.MieAbsorptionScale).GetClamped(0.0f, 1e38f);
    MieAbsorption = bUseGotLmsColorSpace ? MieAbsorption : ConvertCoefficientsFromSRGBToWorkingColorSpace(MieAbsorption);
    
    ......
 
// Ozone
{
    AbsorptionExtinction = (SkyAtmosphereComponent.OtherAbsorption * SkyAtmosphereComponent.OtherAbsorptionScale).GetClamped(0.0f, 1e38f);
    AbsorptionExtinction = bUseGotLmsColorSpace ? AbsorptionExtinction : ConvertCoefficientsFromSRGBToWorkingColorSpace(AbsorptionExtinction);   

```

接下来就是把要传到Shader的色彩空间转换和Shader的Permutation等设置好

```

//SkyAtmosphereRendering.h
// 把在 LMS 色彩空间里积分得到的颜色转回工作色彩空间，以及把工作色彩空间里的光源颜色转进 LMS。
// LMS 色彩空间关闭时都是单位矩阵。存的是转置后的矩阵，因为 shader 侧用 mul(Matrix, Vector)。

SHADER_PARAMETER(FMatrix44f, LMSToWorkingColorSpace)
SHADER_PARAMETER(FMatrix44f, WorkingColorSpaceToLMS)
```

```
//SkyAtmosphereRendering.cpp

static void CopyAtmosphereSetupToUniformShaderParameters(FAtmosphereUniformShaderParameters& out, const FAtmosphereSetup& Atmosphere)
{
#define COPYMACRO(MemberName) out.MemberName = Atmosphere.MemberName 
	......
	COPYMACRO(GroundAlbedo);
	COPYMACRO(LMSToWorkingColorSpace);
	COPYMACRO(WorkingColorSpaceToLMS);
#undef COPYMACRO
}

class FUseGotLmsColorSpace : SHADER_PERMUTATION_BOOL("USE_GOT_LMS_COLOR_SPACE");

// 除了这个class，FRenderTransmittanceLutCS，FRenderMultiScatteredLuminanceLutCS，FRenderDistantSkyLightLutCS，FRenderSkyViewLutCS，FRenderCameraAerialPerspectiveVolumeCS，FRenderDebugSkyAtmospherePS 也要写
class FRenderSkyAtmospherePS : public FGlobalShader
{
    ......
    
    using FPermutationDomain = TShaderPermutationDomain<FSampleCloudSkyAO, FFastSky, FUseGotLmsColorSpace, FFastAerialPespective, FSecondAtmosphereLight,
                                                    FRenderSky, FSampleOpaqueShadow, FSampleCloudShadow,
                                                    FAtmosphereOnClouds, FMSAASampleCount, FHighQualityMie>;

    ......
    
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
	    SHADER_PARAMETER_STRUCT_REF(FAtmosphereUniformShaderParameters, Atmosphere)
	
		......
		
    END_SHADER_PARAMETER_STRUCT()
    
    ......

}

void FSceneRenderer::RenderSkyAtmosphereLookUpTables(FRDGBuilder& GraphBuilder, class FSkyAtmospherePendingRDGResources& PendingRDGResources, FDynamicShadowsTaskData* FilterDynamicShadowTaskData)
{
    ......
    
	const bool bSeparatedAtmosphereMieRayLeigh = VolumetricCloudWantsSeparatedAtmosphereMieRayLeigh(Scene);
	const bool bUseGotLmsColorSpace = CVarSkyAtmosphereUseGotLmsColorSpace.GetValueOnRenderThread() > 0;
	
	......
	
	if (bEvaluateTransmittanceAndMultiScatteringLUTs && CVarSkyAtmosphereTransmittanceLUT.GetValueOnRenderThread() > 0)
	{
		RDG_EVENT_SCOPE(GraphBuilder, "SkyAtmosphere::TransmittanceLut");
				
		FRenderTransmittanceLutCS::FPermutationDomain PermutationVector;
		PermutationVector.Set<FUseGotLmsColorSpace>(bUseGotLmsColorSpace);
		TShaderMapRef<FRenderTransmittanceLutCS> ComputeShader(GlobalShaderMap,PermutationVector);
		......
    }
    // 后面接下来几个Lut类似，省略了

	......
}

void FSceneRenderer::RenderSkyAtmosphereInternal(
	FRDGBuilder& GraphBuilder,
	const FSceneTextureShaderParameters& SceneTextures,
	FSkyAtmosphereRenderContext& SkyRC)
{
    const bool bRenderSkyPixel = SkyRC.bRenderSkyPixel || (SkyAtmosphereOutputsAlpha && !SkyRC.bSceneHasSkyMaterial);	// In this case we need to write alpha holdout values in the sky pixels. If there is no IsSky dmoe meshes.
    const bool bUseGotLmsColorSpace = CVarSkyAtmosphereUseGotLmsColorSpace.GetValueOnRenderThread() != 0;
    
    PsPermutationVector.Set<FUseGotLmsColorSpace>(bUseGotLmsColorSpace);
    
}
```

接下来是shader。加两个 helper 统一处理色彩空间转换。

```
// SkyAtmosphere.usf

#ifndef USE_GOT_LMS_COLOR_SPACE 
#define USE_GOT_LMS_COLOR_SPACE 0 
#endif

float3 AtmosphereColorToWorkingColorSpace(float3 Color)
{
#if USE_GOT_LMS_COLOR_SPACE
	return mul((float3x3)Atmosphere.LMSToWorkingColorSpace, Color);
#else
	return Color;
#endif
}

// 反向。LMS 的基和工作色彩空间不同，高饱和的颜色转过来可能落在 LMS 色域外（出现负坐标），
// 而 LMS 区域内一律要求非负，所以夹一下
float3 WorkingColorToAtmosphereColorSpace(float3 Color)
{
#if USE_GOT_LMS_COLOR_SPACE
	return max(mul((float3x3)Atmosphere.WorkingColorSpaceToLMS, Color), 0.0f);
#else
	return Color;
#endif
}
```

输出要转Working，光源 color 转Lms（对齐场景）：

```
// SkyAtmosphere.usf

// 输出例子：
float3 L = AtmosphereColorToWorkingColorSpace(ss.L);
float3 T = AtmosphereColorToWorkingColorSpace(ss.Transmittance);

// 输入例子：
WorkingColorToAtmosphereColorSpace(View.AtmosphereLightIlluminanceOuterSpace[0].rgb) * SkyAtmosphere.SkyAndAerialPerspectiveLuminanceFactor,
```

要转的地方是 RenderSkyAtmosphereRayMarchingPS、RenderSkyViewLutCS、RenderDistantSkyLightLutCS、RenderCameraAerialPerspectiveVolumeCS，注意每个类的调用点都要 Set 这个 permutation。

TransmittanceLut 要特殊处理一下。Lut里存WorkingColorSpace的透过率，体积云、光照这些别的模块要用。写在Encode前Decode后。

```
// SkyAtmosphere.usf  RenderTransmittanceLutCS

float3 transmittance = AtmosphereColorToWorkingColorSpace(exp(-ss.OpticalDepth));
transmittance = EncodeTransmittance(transmittance);
```

```
// SkyAtmosphereCommon.ush  GetAtmosphereTransmittance

// 取用时按原路转回 LMS，注意是在 DecodeTransmittance 之后
#if USE_GOT_LMS_COLOR_SPACE
	Transmittance = mul((float3x3)Atmosphere.WorkingColorSpaceToLMS, Transmittance);
#endif
```

CPU 侧的 GetTransmittanceAtGroundLevel 同理也要转。它算出来的透过率会被拿去当方向光的颜色（PrepareSunLightProxy，UE注释写的 // See explanation in "Physically Based Sky, Atmosphere	and Cloud Rendering in Frostbite" page 26），所以也要是工作色彩空间的。矩阵行列有变换，虚幻cpu是行主序。

```
// SkyAtmosphereCommonData.cpp

FLinearColor OpticalDepthRGB = OpticalDepth(WorldPos, WorldDir);
const FLinearColor Transmittance = FLinearColor(Exp(-R), Exp(-G), Exp(-B));

const auto LMSToWorking = LMSToWorkingColorSpace.GetTransposed();
const FVector4f WorkingTransmittance = LMSToWorking.TransformVector(FVector3f(Transmittance.R, Transmittance.G, Transmittance.B));
return FLinearColor(WorkingTransmittance.X, WorkingTransmittance.Y, WorkingTransmittance.Z);
```

### 系数重新拟合

即使不去管这几个系数怎么来的，很好看出来引擎scattering的系数就是把论文数据的33.1作为scale了，其他数值对应做了除法，m转km。
![alt text](/assets/images/CustomRayleighColorSpace/screenshot-20260329-164532.png)
**左：论文里提到的系数 右侧：虚幻编辑器的默认系数**

可以照样套到对马岛分享的系数上，因为好奇对马岛的数据怎么来的，也因为我把全部大气计算都改LMS了，Ozone的也要改，还是自己搞了下拟合。

首先自然是找一个能够作为参考的实现，我用的是Eradiate, 这是一个为了验证遥测数据的项目，有光谱渲染和完整的路径追踪，它基于的Mitsuba3基本上已经常规GroundTruth了），Eradiate在大气这块增强了，有支持真实世界的数据。 过程中还发现了SkyTracer项目（之前写的笔记有），也可以参考。

这块拟合主要是靠deepseek写的。简单说下思路，实现就不写了捏（也可能以后单独写），后面放结果。

首先Eradiate有很多个参数模型，用的是最基础的afgl_1986-us_standard，额外用了CKD的absorption数据库。其他季节海拔的也有内置，等如果以后做TOD和季节变换可以再考虑下。这些差异可以看GT7的分享他们都有提到。
然后是把虚幻Sebastien Hillaire的方案移植到了python，用pytorch，修改一些了使渲染可微，然后用L-BFGS方法，Ictcp做loss计算（ΔE_ITP）。

对臭氧的Tent分布的Tip Altitude、width和瑞利散射exp distribution，以及multiScatter的，以及系数拟合。太阳高度我设置了2个略低于地平线的位置-0.5和-1度（再低拟合误差很大），0-30度13个+4个更高角度的，Hillaire方案大气高度用的100km。

把exp distribution和multiScatterLut的参数也塞进拟合了，实际可能不太符合物理，但是整体颜色有改进，相比参考的ΔE_ITP均值误差大概在4%以下，对马岛Jasmin Patry分享的这个色彩空间确实有用，能够减少误差，但主要还是拟合参数带来的收益。最终结果（对马岛LMS色彩空间的）：

- 瑞利参数 0.02750789 × (0.280417, 0.516265, 1.000000) 。也就是0.007713677, 0.01420137, 0.02750789  $$ km^{-1} $$，跟对马岛的还是比较接近的，感觉他们做的近似应该比我的好。我拟合同时也拟合了瑞利Exponential Distribution，不太好直接数值对比。
- 臭氧吸收 0.00366248 × (1.000000, 0.717798, 0.298517)，这里虚幻是G高，拟合的是R高，应该是他们和我的拟合目标不一样。虚幻明显偏紫色，我这个偏蓝色，文末对比图可见。
- Tip Altitude 18.8092 ， Width  12.7364
- 瑞利Exponential Distribution 7.2753
- multiScatterFactor 1.003533 （UE的multiScattering参数）

## 修改Mie散射

原本是打算GT7一样多个log-Normal分布混合的思路，简化成每组接近的粒子大小算个平均直径，再用平均直径去算相函数，再混合在一起。简化到两层就是,一层Crose Mie，一层FineMie，多一套参数，但是这样参数很多很麻烦，性能变差不说物理上写对也挺费力，效果也一般。用真实数据也试过，但是很难对应上哪个数据应该是什么条件的效果，不太直观。
![img3.png](/assets/images/CustomRayleighColorSpace/img3.png){:.img-h-md}

既然效果提升不大，还是简单点（至少代码上简单点），比如改个Mie相位函数。虚幻的 Mie 相位是 Henyey-Greenstein（HG），只有一个 g 参数，光晕是个很平滑的圆，实际粒子尺度小的时候会出现很明显的彩色光环。GT7 用的BHMIE离线算 Mie 散射相位，虚幻做体积云的时候考虑了这个用MiePlot算了Lut，来做第一次散射。
![img2.png](/assets/images/CustomRayleighColorSpace/img2.png)

Nvidia 那篇文章给了个省事的近似：输入只有粒子直径 d（单位微米）一个，把 Draine 相位和 HG 相位按 blend 混合。d 超过 50 直接 clamp 掉，可以整个写进 shader，不用额外做 Lut（虽然分段导致多了些判断，计算量也变大了）。从结果上来说效果还是比UE自带的好些，也更接近GT7的。

放下代码：
相函数和参数，注意cosTheta的正负

```
// ParticipatingMediaCommon.ush


// reduces to HG for a = 0, to Rayleigh for g = 0, a = 1 and to CS(Cornette-Shanks) for a = 1.
float DrainePhase(float CosTheta, float g, float a)
{
	float mu = -CosTheta;
	float Denom = 1.0 + g * g - 2.0 * g * mu;
	return ((1.0 - g * g) * (1.0 + a * mu * mu)) / (4.0 * (1.0 + (a * (1.0 + 2.0 * g * g)) / 3.0) * PI *
		Denom * sqrt(Denom));
}

// d 是粒子直径，单位微米 um，Nvidia 拟合出来的参数
float4 mie_parameters(float d)
{
	if (d <= 0.1)
	{
		return float4(
			13.8 * d * d,
			1.1456 * d * sin(9.29044 * d),
			250.0,
			0.252977 - 312.983 * pow(d, 4.3));
	}
	else if (d <= 1.5)
	{
		return float4(
			0.862 - 0.143 * log(d) * log(d),
			0.379685 * cos(
				1.19692 * cos((log(d) - 0.238604) * (log(d) + 1.00667) / (0.507522 - 0.15677 * log(d))) + 1.37932 *
				log(d) + 0.0625835) + 0.344213,
			250.0,
			0.146209 * cos(3.38707 * log(d) + 2.11193) + 0.316072 + 0.0778917 * log(d));
	}
	else if (d <= 5.0)
	{
		return float4(
			0.0604931 * log(log(d)) + 0.940256,
			0.500411 - 0.081287 / (-2.0 * log(d) + tan(log(d)) + 1.27551),
			7.30354 * log(d) + 6.31675,
			0.026914 * (log(d) - cos(5.68947 * (log(log(d)) - 0.0292149))) + 0.376475);
	}
	else
	{
		return float4(
			exp(-0.0990567 / (d - 1.67154)),
			exp(-2.20679 / (d + 3.91029) - 0.428934),
			exp(3.62489 - 8.29288 / (d + 5.52825)),
			exp(-0.599085 / (d - 0.641583) - 0.665888));
	}
}

float DraineHenyeyGreensteinPhase(float d, float CosTheta)
{
	d = clamp(d, 0, 50);
	float4 mie_params = mie_parameters(d);
	float HG = HenyeyGreensteinPhase(mie_params[0], CosTheta);
	// UE 的 HenyeyGreensteinPhase 方向不同，要取反
	float D = DrainePhase(-CosTheta, mie_params[1], mie_params[2]);
	return lerp(HG, D, mie_params[3]);
}
```

然后把 SkyAtmosphere.usf 里的相位函数换掉，加个 cvar 和 permutation 控制开关，还加了个直径参数在UI上：

```
// SkyAtmosphere.usf

float SkyAtmosphereEvaluateMiePhase(float CosTheta)
{
#if SKYATMOSPHERE_HIGH_QUALITY_MIE
	// 高品质模式：Mie 相位由气溶胶粒径决定，MieAnisotropy 不参与计算
	return DraineHenyeyGreensteinPhase(SkyAtmosphere.AerosolDiameter, CosTheta);
#else
	return HenyeyGreensteinPhase(Atmosphere.MiePhaseG, CosTheta);
#endif
}
```

```
// SkyAtmosphereRendering.cpp

class FHighQualityMie : SHADER_PERMUTATION_BOOL("SKYATMOSPHERE_HIGH_QUALITY_MIE");

static TAutoConsoleVariable<int32> CVarSkyAtmosphereHighQualityMie(
	TEXT("r.SkyAtmosphere.HighQualityMie"), 0,
	TEXT("Enable high quality Mie phase function. 0: HenyeyGreenstein (default), 1: DraineHenyeyGreenstein."),
	ECVF_RenderThreadSafe | ECVF_Scalability);
```


```
// SkyAtmosphereComponent.h

UPROPERTY(EditAnywhere, BlueprintReadOnly, interp, Category = "Atmosphere - Mie|High Quality", meta = (DisplayName = "Aerosol Diameter", UIMin = 0.01, UIMax = 50.0, ClampMin = 0.0, ClampMax = 50.0, SliderExponent = 1.0))
float AerosolDiameter;
```

```
// SkyAtmosphereRendering.cpp

SHADER_PARAMETER(float, AerosolDiameter)

......

InternalCommonParameters.AerosolDiameter = FMath::Max(0.0f, AtmosphereSetup.AerosolDiameter);
```
 
上面 clamp 到 50 是跟着 `DraineHenyeyGreensteinPhase` 里的 clamp 来的。
用这个相位的问题是高频成分多，走 SkyViewLut 和 AerialPerspectiveVolume 的话分辨率不够会有严重的方块走样，这也是开头说这两个 LUT 得大幅加分辨率的原因。

## 效果图对比

截图都没有用虚幻的fastSkyLut模式，用的逐像素raymarching，multiScatter也开了高质量。用了GT7的SDR tonemap（250nit parperWhite）不是默认的Filmic，虚幻改Tonemap可能以后水一篇文章记录下。手动曝光，没有开Bloom暗角等。大气高度都是100km（这项其实差距很小无所谓）。

对比虚幻原参数，原版Mie散射+lms修改的系数等。左原版。
![img4.png](/assets/images/CustomRayleighColorSpace/img4.png)
![img.png](/assets/images/CustomRayleighColorSpace/img.png)

原版色彩空间和系数对比Mie散射（原版参数没法完全对应，调到近似范围，调了曝光），也是左虚幻原版。
![img9.png](/assets/images/CustomRayleighColorSpace/img9.png)
![img8.png](/assets/images/CustomRayleighColorSpace/img8.png)
![img7.png](/assets/images/CustomRayleighColorSpace/img7.png)
可以看到虚幻的只有集中或者发散二选一，修改过的有更多变化。 看这个参考图也能看出和Measure的那项更接近了。

![img6.png](/assets/images/CustomRayleighColorSpace/img6.png)
![img5.png](/assets/images/CustomRayleighColorSpace/img5.png)

对比GT7，根据画面调整了Mie和曝光，对马岛LMS色彩空间，其他参数都是拟合结果的。
![img10.png](/assets/images/CustomRayleighColorSpace/img10.png)
![img12.png](/assets/images/CustomRayleighColorSpace/img12.png)
![img11.png](/assets/images/CustomRayleighColorSpace/img11.png)

[Jasmin Patry对马岛的分享]: https://advances.realtimerendering.com/s2021/jpatry_advances2021/index.html#/87/0/3
[Sébastien Hillaire]: https://sebh.github.io/publications/index.html
[Eradiate]:https://www.eradiate.eu/site/
[mitsuba3]:https://www.mitsuba-renderer.org/
[Realistic Real-time Sky Dome Rendering in Gran Turismo 7]: https://www.gdcvault.com/play/1029434/Advanced-Graphics-Summit-Realistic-Real
[An Approximate Mie Scattering Function for Fog and Cloud Rendering]:https://research.nvidia.com/labs/rtr/approximate-mie/
[Physically Based and Scalable Atmosphere in Unreal Engine]:https://blog.selfshadow.com/publications/s2020-shading-course/hillaire/s2020_pbs_hillaire_slides.pdf
[Precomputed Atmospheric Scattering]: https://hal.science/inria-00288758/en/