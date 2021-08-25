# Dynamic Cube Map

## Cubemap Textures

Cubemaps are a type of texture made up of 6 faces. They can represent a 3 dimensional rendering of an area. 
Rather than 2D texture coordinates a 3 dimensional direction vector is used to look up each texel.
They are often used for skybox rendering but they can also be used to model reflective objects by projecting the environment onto the faces of the cube.
For a given point on a reflective object's surface we reflect the eye to point direction vector by the surface normal  and use this as the lookup value. 


## Cubemap Generation

In a relatively open scene we might be able to get away with just reflecting the sky or distant objects baked into a cubemap texture file.
However if we need to reflect local objects which are not part of the background, we would need to dynamically generate a cubemap from the scene.

The large sphere below appears to reflect both the sky and nearby geometry. Before the main rendering pass the entire scene is rendered onto 6 faces of a cube.
Each of the 6 faces of the cubemap is created by rendering the scene with the eye point at C and with view direction in the positive and negative x ,y and z directions.
All 6 faces were rendered in a single pass via a render target array and the geometry shader (available in DirectX 10).

Note that the sky is rendered using a static cubemap texture loaded from a file. it is used both in the main render pass and within the dynamic cubemap render pass.

In a dynamic scene where  object positions, lighsources etc. are changing the cubemap generate must be done periodically - i.e. every N frames.
In the below scene the large sphere orbits around the centre of the scene and the cubemap is updated every 6 frames.

<img src="https://user-images.githubusercontent.com/713970/130867298-0cbda54b-2eac-4064-bc1c-4e3c3822bdbb.png" width="400" height="320"> <img src="https://user-images.githubusercontent.com/713970/130868519-d5cbfe45-7190-47ec-b233-a08990707494.png" width="400" height="320">


## Reflections Within Reflections

Note that on the left image the smaller reflective orbs appear black (their diffuse colour) in the reflection. We have 3 options to render reflective objects within the creation of the dyanamic cube map.

1. Use of a static cubemap texture
2. Maintain 2 iterations of the dynamic cube map (current and previous iterations)
3. Recursive render pass

The image on the right shows the use of the first most simple technique.  If you look closely at the reflections within the reflections they do not contain the surrounding scene. This is the easiest method.

The second method can be used If more accurate results are required with little performance loss.  The idea is to maintain 2 iterations of the dynamic cubemap and use the older iteration for 2nd order reflections. The disadvantages are that this requires double the memory resources to store the extra iteration and that the second order reflections will lag behind the scene by 2N where N is the number of frames per cubemap render.

Finally, the third method is a recursive render pass for cube map generation. This would provide the most accurate results however the performance hit is probably too high. 

## Performance

The bottleneck for the cubemap generation using this method lies in the geometry shader.
We need to minimise the amount of data passing through the geometery shader function and the calculations done within it.
We can achieve this via frustum and back face culling and also pre calculating the set of 6 view projection matrices (common for all vertices in the scene).

The biggest performance gain came from reducing the [maxvertexcount] parameter of the geometry shader. Initially this was set to 18 since each input triangle is made of 3 vertices and we are rendering to 6 faces:  6x3 = 18.
However in practice each triangle is likely to only appear from at most 3 of the 6 view directions. Only an extremely large triangle or a triangle close to the cube map centre position C will be visible from 4 or 5 view directions.
It would have to lie exactly on C to be visible from all 6. 

Since our reflective objects have a certain radius we don't have to worry about triangles very close to C.
Also In this case we have no huge triangles in the scene, but if we did we could simply  break them up into smaller ones when constructing our meshes.

In the above scene I reduced the maxvertexcount to to 9 (3 x 3) which is the minimum needed to render the cubemap correctly.

Shader Code Extract:

```hlsl
struct GS_IN_CUBEMAP
{
    float4 PosW : POSITION;
    float4 Normal : NORMAL;
    float2 Tex : TEXCOORD0;
};

struct GS_OUT_CUBEMAP
{
    float4 PosH : SV_POSITION; // Projection coord
    float4 PosW : POSITION;    // World coord 
    float4 Normal : NORMAL; 
    float2 Tex : TEXCOORD0;    // Texture coord
    uint RTIndex : SV_RenderTargetArrayIndex;
};

[maxvertexcount(9)]
void GS_CubeMap( triangle GS_IN_CUBEMAP input[3], inout TriangleStream<PS_IN_CUBEMAP> CubeMapStream )
{
    // per cubemap face f  
    for( int f = 0; f < 6; ++f )
    {
        GS_OUT_CUBEMAP output;
        output.RTIndex = f;

        // Compute screen coordinates
        float4 tpos[3];
        tpos[0] = mul(input[0].PosW, m_mViewProjCM[f]);
        tpos[1] = mul(input[1].PosW, m_mViewProjCM[f]);
        tpos[2] = mul(input[2].PosW, m_mViewProjCM[f]);

        // Frustum culling
        float4 t0 = saturate(tpos[0].xyxy * float4(-1,-1,1,1) - tpos[0].w);
        float4 t1 = saturate(tpos[1].xyxy * float4(-1,-1,1,1) - tpos[1].w);
        float4 t2 = saturate(tpos[2].xyxy * float4(-1,-1,1,1) - tpos[2].w);
        float4 t = t0 * t1 * t2;

        if (!any(t))
        {
            //Back face culling
            float2 d0 = tpos[1].xy / abs(tpos[1].w) - tpos[0].xy / abs(tpos[0].w);
            float2 d1 = tpos[2].xy / abs(tpos[2].w) - tpos[0].xy / abs(tpos[0].w);
            if (d1.x * d0.y > d0.x * d1.y)
            {
                for (int v = 0; v < 3; v++)
                {
                    output.PosH = tpos[v];
                    output.Normal = input[v].Normal;
                    output.PosW = input[v].PosW; 
                    output.Tex  = input[v].Tex;     
                    CubeMapStream.Append(output);
                }
                CubeMapStream.RestartStrip();
            }
        }
    }
}
