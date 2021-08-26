# Light Shafts

This method is based on a _GPU Gems Chapter 13: Volumetric Light scattering as a post process._

A "volume" or shafts of light are produced when dust particles in the air causes light to scatter.
This effect can be recreated purely as a post process by applying a type of radial blur (centered around the light position) to an image.
The results are shown below:

<image src="https://user-images.githubusercontent.com/713970/130870678-ce027006-a6a8-4a19-94e3-6d51b780d443.png" width="400" height="320"> <image src="https://user-images.githubusercontent.com/713970/130870813-04b400cf-73fe-4002-b81f-1a760cc8e54e.png" width="400" height="320">
  
Note that the sun in this image was just a white coloured sphere before the post processing was applied.
The blur has an added bonus of producing a natural looking glow effect with streaks.

The method involves rendering the scene normally to an off screen texture T. 
We then sample T and use a radial blur effect to generated light shafts on texture L. 
The final result is given by rendering a full screen quad with using an additive blend of textures T and L.

When we first render the scene to T within the pixel shader we use the alpha channel to store information about each fragment. 
In the method I have chosen light sources (the sun) and occluding objects have an alpha value of 1.0, and the sky a value of 0.0. 
Then when we apply the blur process to create L only pixels with an alpha value of 1 contribute to the blend. The result can be seen in the image on the right. We discard the color information to get a uniform light shaft color.

## Issues

A problem highlighted in the GPU gems article is that light shafts flicker as occluding objects pass the edge of the screen.
This is because the objects fall outside the range of possible samples. I applied a simple fix for this by changing the sampler state addressing mode to "mirror".

```hlsl
SamplerState g_samLinearMirror
{
    Filter = MIN_MAG_MIP_LINEAR;
    AddressU = Mirror;
    AddressV = Mirror;
};
```
  
The other issue mentioned is that as we approach a viewing angle at 90 degrees to the direction to the light source, 
the space between samples approaches infinity which causes artefacts. I chose resolve this by fading the effect out according to this angle.
The sum of the samples is multiplied by the dot product of E (eye look at vector) and L (vector from the eye position to the light source position).

## Performance

Texture T is created with dimensions screenWidth, screenHeight since it will be used to colour a full screen quad.
For the above scene 30 samples are taken from T to produce each pixel in texture L.
If L were for example a 1024x1024 texture it would result in a huge number of required samples.
We therefore need to keep the dimensions of L to a minimum. Here I have used the dimensions screenWidth / 4 , screenHeight / 4 
which greatly reduces the cost of blur.

  
Shader Code Extract:

```hlsl
float4 PS_RadialBlur(VS_OUT pIn) : SV_Target
{
    half2 ScreenLightPos = CalculateNDC(mLightPosH.xyw);
    half2 texCoord = CalculateNDC(float3(pIn.texPos.xy,1.0f));

    // Calculate vector from pixel to light source in screen space.   
    half2 deltaTexCoord = (texCoord.xy - ScreenLightPos.xy);  

    // Divide by number of samples and scale by control factor.  
    deltaTexCoord *= 1.0f / NUM_SAMPLES * DENSITY;  

    // initial raycolor is 0;
    half4 raycolor = (float4) 0.0f;

    // Set up illumination decay factor.  
    half illuminationDecay = 1.0f;  

    // Evaluate summation from Equation 3 NUM_SAMPLES iterations.  
    for (int i = 0; i < NUM_SAMPLES; i++)  
    {  
        // Step sample location along ray.  
        texCoord -= deltaTexCoord;  
        // Retrieve sample at new location.  
        half4 sample = gColorMap.Sample(g_samLinearMirror, texCoord);  

        if(sample.a == 1.0)
        {
            // Apply sample attenuation scale/decay factors.  
            sample *= illuminationDecay * WEIGHT;  
            // Accumulate combined color.  
            raycolor += sample;
        }
 
        // Update exponential decay factor.  
        illuminationDecay *= DECAY; 
    }

    // scale by angle to light source and a constant
    raycolor *= mEdotL * EXPOSURE; 
    return raycolor;
}
```
