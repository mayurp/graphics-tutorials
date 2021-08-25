# Graphics Tutorials


## Water Effect

The aim here is to create the appearance of a relatively still water surface. 

The water will have the following properties:

- Small waves of varying frequencies.
- Reflection of objects above the surface.
- Refraction of objects below the surface.


The final result can be seen below: 

<iframe width="425" height="355" src="https://www.youtube.com/embed/0SPX0h42xCs" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Most of this can be done within a pixel shader. No vertex displacement is needed here. 

### Reflection

A reflection map can be created by rendering the entire scene reflected about the water plane onto an off screen render target. Any objects below the water plane should be culled. This image is then used as a texture when rendering reflective objects.

We could multiply each object's world matrix by a reflection matrix R. We would also need to reflect the position and orientation of the camera and light(s).

An easier solution is to simply pre-multiply the view matrix by R and render the scene objects normally. This renders the scene from the reflected camera position and also reflects the coordinate system of the resulting image. Lighting calculations are done using the original world space coordinates so there's no need to reflect the light position!

### Refraction

The appearance of refraction through a transparent object can be produced by first rendering the scene behind the object to a texture. This is then used to texture the surface of the object when the scene is rendered to the back buffer in a second pass. Each texture coordinate is perturbed by a small amount to simulate the bending of light. These offsets can be loaded from a bump map and scaled accordingly.

```hlsl
float4 refractColour = RefractiveMap.Sample(sampler, texCoord + offset);
```

The downside is that we have to render the entire scene twice. Once to create the refraction texture and once to render the scene to the backbuffer.

### Multiple Render Targets

However, It is possible to achieve the effect by rendering the main scene once using multiple render targets i.e. rendering to multiple textures simultaneously.

We use 2 render targets in this case. The first is associated with the backbuffer and the second with a texture map. We need to create a pixel shader which outputs 2 color values. We can just create call our regular shader function and output the result twice.

First be render the regular objects in the scene. For anything behind the water plane we set the pixels's alpha channel to 0.  We then render the water using the texture map only sampling values where the alpha is 0. This ensures objects above the water don't contribute to (or bleed into)  the water color.

DirectX 10 Code Extract:

```c++
ID3D10RenderTargetView* renderTargets[2] = {m_pBackBufferRTV, m_pTextureMapRTV};
m_pd3dDevice->OMSetRenderTargets(2, renderTargets, m_pDepthStencilView);
```

Shader Code Extract:

```hlsl
struct PS_OUT_Main_MRT
{
    // The pixel shader can output 2 values simulatanously 
    float4 RGBColor1    : SV_Target0;  // for backbuffer
    float4 RGBColor2    : SV_Target1;  // for texture
};

PS_OUT_Main_MRT PS_Main_MRT(VS_OUT_Main pIn)
{
    PS_OUT_Main_MRT pOut;
    pOut.RGBColor1 = PS_Main(pIn);    // Call regular pixel shader function
    pOut.RGBColor2 = pOut.RGBColor1;  // Copy result to second output 

    pOut.RGBColor2.a = 1.0f;          // alpha is 1 by default;

    // Set alpha mask to 0 for objects below the water plane
    float d = dot (mPlaneNormal,(pIn.posW - mPlanePoint));
    if(d < 0)
    {
        pOut.RGBColor2.a = 0.0f;
    }
    return pOut;
}
```

### Putting It All Together

So now we have 2 textures to render the water  - a reflective map and a refractive map. 
To produce waves of different frequencies we calculate 4 sets of texture coordinates in the vertex shader. Each is multiplied by different frequency factor. A time based component is then added to animate the waves. 

We combine the refractive and reflective colours based on the angle between the water surface bump normal and the eye look at vector. Looking at the water at a shallow angle (i.e. almost at 90 degrees to the normal) means the reflective colour is at it's maximum and the refractive component is at  it's minimum. Looking straight into the water means the reflective colour is at it's minimum and we see the maximum refractive colour. 


Shader Code Extract:

```hlsl
VS_OUT_Water VS_Water(VS_IN_Water vIn)
{
    .....

    // Transform to homogeneous clip space.
    vOut.posH = mul(float4(vIn.posL, 1.0f), mWorldViewProj);

    vOut.posTexC = vOut.posH;

    // Texture coordinates to sample water bump map at different frequencies
    vOut.waterC1 = vin.texC*WAVE_FREQ1 + gTime*WAVE_VELOCITY1;
    vOut.waterC2 = vin.texC*WAVE_FREQ2 + gTime*WAVE_VELOCITY2;
    vOut.waterC3 = vin.texC*WAVE_FREQ3 + gTime*WAVE_VELOCITY3;
    vOut.waterC4 = vin.texC*WAVE_FREQ4 + gTime*WAVE_VELOCITY4;
    
    .....
}

float4 PS_Water(VS_OUT_Water pIn) : SV_Target
{

    // Convert from NDC space to texture space [0,0] to [1,1] 
    float2 postex  = NDCtoTextureSpace(pIn.posTexC.xyw); 

    .....       

    // Get displacement vectors from bump map.
    // Sampler state should be wrap or mirror
    float3 normalT1 = g_txNormalMap.Sample( g_samWaterBump, pIn.waterC1 ).xyz;
    float3 normalT2 = g_txNormalMap.Sample( g_samWaterBump, pIn.waterC2 ).xyz;
    float3 normalT3 = g_txNormalMap.Sample( g_samWaterBump, pIn.waterC3 ).xyz;
    float3 normalT4 = g_txNormalMap.Sample( g_samWaterBump, pIn.waterC4 ).xyz;

    float3 normalT = normalT1 + normalT2 + normalT3 + normalT4;

    // Uncompress each component from [0,1] to [-1,1].
    normalT = 2.0f*normalT - 4.0f;

    .....

    // Scale displacement vectors
    float3 vRefrDisp = normalT.xyz * REFRACT_DISP_FACTOR; 
    float3 vReflDisp = normalT.xyz * REFLECT_DISP_FACTOR;

    // Get reflective colour
    float4 reflecCol = g_txMirrorMap.Sample(gTriLinearSam, postex + vReflDisp);

    // Get refractive colour
    float4 refracCol = g_txPostProcMap.Sample(g_samPoint, postex + vRefrDisp);
    
    // if alpha equals one 1 displaced sample is above water so do not use it
    if (refracCol.a == 1.0f)
    {
      refracCol = g_txPostProcMap.Sample(g_samPoint, postex);
    }

    ......

    float3 eyeVector = normalize(mEyeW - pIn.posW);
    float NdotL = max(dot(eyeVector, bumpedNormalW), 0);
    // Calculate fresnel for reflected colour.
    float  fresnel  = max(0.0, 0.2 + 0.8 * pow(1-NdotL, 5); 
    float3 watercolor = reflecCol * fresnel + refracCol * NdotL ;

    return watercolor;                         
}        
```

