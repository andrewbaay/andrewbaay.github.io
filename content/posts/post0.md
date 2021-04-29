+++
title = "Source Engine Cascaded Shadow Implementation"
date = "2021-04-29T15:43:47+08:00"
author = ""
authorTwitter = "baayandrew" #do not include @
cover = ""
tags = ["graphics", "source engine"]
keywords = ["", ""]
description = ""
showFullContent = false
+++


```cpp

//==========================================================================================
// Give shader CSM data
//==========================================================================================
void AGS_SetupCascadedShadowPixelShaderConstants(CBaseVSShader* pShader, IShaderDynamicAPI* pShaderAPI, int constantRegister)
{
	// This must be consistent with what we have on ags_globallight_ps2_3_x.h!
	// CSM + Snapshot takes 13 constant registers after initial
	int initial = constantRegister;

	CSMData_t csmData = AGS_GetRenderState()->csmData;
	CSMStaticData_t staticShadowData = AGS_GetRenderState()->csmStaticData;

	ITexture* pCascadedDepthTexture = NULL;
	ITexture* pStaticDepthTexture = NULL;
	ITexture* pCloudsTexture = NULL;

	pCascadedDepthTexture = csmData.m_depthTexture;
	pStaticDepthTexture = staticShadowData.m_depthTexture;

	// Light forward vector + cloud shadow trigger on alpha
	Vector csmFwd = AGS_GetRenderState()->m_vSunPos;
	pShaderAPI->SetPixelShaderConstant(initial, csmFwd.Base());

	// Light Color
	Vector4D csmLight = AGS_GetRenderState()->m_vSunColor;
	pShaderAPI->SetPixelShaderConstant(initial + 1, csmLight.Base());

	if (pCascadedDepthTexture)
	{
		// Cascaded Depth
		pCascadedDepthTexture = csmData.m_depthTexture;
		pShader->BindTexture(SAMPLER_CASCADED_SHADOW, pCascadedDepthTexture, 0);

		//Far cascade step data
		const Vector vecCascadedStep = csmData.vecCascadedStep;
		float vCascadedStep[3] = { XYZ(vecCascadedStep) };
		pShaderAPI->SetPixelShaderConstant(initial + 2, vCascadedStep);

		//Biasing
		float biasVar[2] = { 0, 0 };
		pShaderAPI->SetPixelShaderConstant(initial + 3, biasVar);

		// Cascaded World To Texture
		VMatrix* worldToTexture0 = csmData.m_worldToTextureMatrix;
		pShaderAPI->SetPixelShaderConstant(initial + 4, worldToTexture0->Base(), 4); // up to 44

		// Cascaded Filter Size
		float csmShadowTweaks[4];
		csmShadowTweaks[0] = pCascadedDepthTexture->GetActualWidth() * r_csm_filterscale.GetFloat();
		csmShadowTweaks[1] = pCascadedDepthTexture->GetActualHeight() * r_csm_filterscale.GetFloat();
		csmShadowTweaks[2] = 1;
		pShaderAPI->SetPixelShaderConstant(initial + 8, csmShadowTweaks);
	}
	else
	{
		pShaderAPI->BindStandardTexture(SAMPLER_CASCADED_SHADOW, TEXTURE_WHITE);
	}
	if (pStaticDepthTexture)
	{
		// Static Depth
		pStaticDepthTexture = staticShadowData.m_depthTexture;
		pShader->BindTexture(SAMPLER_STATIC_SHADOW, pStaticDepthTexture, 0);

		// Static World To Texture
		VMatrix* worldToTexture1 = staticShadowData.m_worldToTextureMatrix;
		pShaderAPI->SetPixelShaderConstant(initial + 9, worldToTexture1->Base(), 4); // up to 44

		// Snapshot Filter Size
		float csmStaticShadowTweaks[4];
		csmStaticShadowTweaks[0] = pStaticDepthTexture->GetActualWidth() * r_csm_filterscale.GetFloat();
		csmStaticShadowTweaks[1] = pStaticDepthTexture->GetActualHeight() * r_csm_filterscale.GetFloat();
		csmStaticShadowTweaks[2] = 1;
		pShaderAPI->SetPixelShaderConstant(initial + 13, csmStaticShadowTweaks);
	}
	else
	{
		pShaderAPI->BindStandardTexture(SAMPLER_STATIC_SHADOW, TEXTURE_WHITE);
	}

	pCloudsTexture = AGS_GetRenderState()->cloudShadow;
	// Setting up framebuffer spec map
	if (pCloudsTexture && AGS_GetRenderState()->m_bCloudShadowEnabled)
	{
		pShader->BindTexture(SAMPLER_CLOUDS_SHADOW, pCloudsTexture, 0);
	}
	else
	{
		pShaderAPI->BindStandardTexture(SAMPLER_CLOUDS_SHADOW, TEXTURE_WHITE);
	}
	// Snapshot Depth
}
```