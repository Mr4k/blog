Title: A Technical Survery of 3D Graphics in Playdate Games
Date: 2024-01-09
Modified: 2024-01-09
Category: graphics, playdate, video games
Tags: graphics, playdate, video games
Slug: playdate3d
Authors: Peter Stefek
Summary: Can the gameboy for hipsters do 3d?

<p align="center">
	<img src="/images/playdate3d/playdate3d-header.png" width="300px"> 
</p>   

This year one of my friends gave me a [Playdate](https://play.date/) for Christmas and I have been fascinated with it for the last couple weeks. As a hobbyist game developer, I wanted to make a game for it. I had a bunch of 3D game ideas but realized that the hardware on the playdate is quite limited compared to my m1 mac. Instead of deterring me, the constraints drew me in and I’ve spent the last couple of weeks exploring the potential for 3D graphics on the playdate.  

To prepare for making a 3D engine I wanted to get a sense of what was already out there. This post will be a review of prior art. I've dug through and categorized all the 3d games/demos/engines for the playdate that I could find.  

Before we dive in, I want to talk about how I categorized these games. It’s difficult to find a hard boundary that defines if a game is 3D or not. As such, the process of deciding if a game is 3D is somewhat a matter of personal opinion. 

A game is 3D to me if it makes me personally feel like I’m looking through a window into some kind of world containing 3 spatial dimensions. For example, some cases I don’t consider 3D are [3/4s view RPGs](https://tvtropes.org/pmwiki/pmwiki.php/Main/ThreeQuartersView) (such as [Casual Birder](https://play.date/games/casual-birder/)) and 2d sidescrollers with [parallax](https://en.wikipedia.org/wiki/Parallax_scrolling) (such as [Midnight Raider](https://lumicreative.itch.io/midnight-raider)).

People who have played a lot of video games also know that there are different kinds of 3D games with different capabilities. Think [Wolfenstien 3D](https://www.youtube.com/watch?v=561sPCk6ByE) vs [Horizon Forbidden West](https://youtu.be/_is7aycnw6k?t=645). Sometimes the words 2.5D or pseudo 3D are used around for games that feel less “3D” than the state of the art. Most games on the playdate fall into this “pseudo” 3D category so these terms are too broad to be useful here. Instead I’m going to categorize games/demos/engines based on how many mechanical degrees of freedom the player has, camera position (first person, third person dynamic, third person static) and projection type ([perspective or orthographic](https://www.scratchapixel.com/lessons/3d-basic-rendering/perspective-and-orthographic-projection-matrix/building-basic-perspective-projection-matrix.html)).  

I’m choosing to focus on these three categories because they all represent tradeoffs between flexibility and performance. 

In general the fewer degrees of freedom there are, the more room there is to “fake” things.  

Other factors play a large role in performance too such as camera type (a static third person camera can be a lot easier to render than a dynamic first person camera), projection type (orthographic projection can be faster than perspective projection) and of course level of graphical fidelity/realism. As we’ll see, different 3D games on the playdate make many different tradeoffs between performance, flexibility and fidelity. 

A last quick note about performance on the playdate, I will give frames per second numbers when available. However it’s important to note that the max fps on the playdate while refreshing the entire screen is 52 fps. I say “refreshing the entire screen” because the playdate sdk also lets you only update part of the screen. In that case you can actually render faster than 52 fps but most 3d games have to redraw the entire screen with every update.  

**Grand Tour Legends**  
Type: Game  
Status: Released  
Link: [https://play.date/games/grand-tour-legends/](https://play.date/games/grand-tour-legends/)   
Degrees of Freedom: 1 (1 combined spatial / rotational degree of freedom)  
Performance On Device: Smooth (no fps counter available)  
Technique: Full Motion Video+  
Camera Position: Third Person Dynamic (Over the Shoulder)  
Camera Projection: Perspective  
<p align="center">
	<img src="/images/playdate3d/grand-tour-legends.gif" width="400px"> 
</p>   

Grand Tour Legends is the first playdate game I played. It’s also the most beautiful 3D I’ve seen on the playdate, but all that shine does not come for free. The game only has one true degree of freedom. You can move forward or roll backwards and that’s it.

Notes  
While the game’s source code is not available there is a little info [here](https://devforum.play.date/t/grand-tour-legends/2306). We know the environment is a pre-rendered video that speeds up based on how fast you are moving (or in some cases goes backwards when you roll downhill). The player is a sprite and the other racers are apparently “fully 3D” (whether this means meshes or [imposters](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/RenderToTextureTools/3/) is not clear to me).  

One thing the above thread doesn’t tell me is how the other racers are occluded by the environment. I suspect when rendering them, each of their pixels is compared against a depth buffer channel of the pre-rendered video to make sure not to draw them in front of the terrain if they are actually behind it.  

I’m also not sure whether the dithering is pre-rendered or not. Either way it looks gorgeous.  

I’m calling this technique full motion video+ because it is substantially more complex than just drawing a video.   

The advantages of pre-rendering a video for the environment are that you can apply all sorts of extra beautification steps (antialiasing, shadows, complex lighting, etc) without any overhead in game. On pc GPU rasterizers are fast enough to make nearly photorealistic images in real time so the benefits or pre-rendering are minimal, but on playdate you can create unmatched beauty.  

My props to [IORAMA](https://www.iorama.studio/) for coming up with a very creative rendering technique.  

**Unnamed Train Game**  
Type: Demo  
Status: Unreleased  
Link: [https://nitter.net/HunterBridges](https://nitter.net/HunterBridges/status/1566650619254882304)   
Degrees of Freedom: 1 (1 spatial degree of freedom, 0 rotation degrees of freedom)  
Performance On Device: Unknown  
Technique: Unknown (not enough info to guess well)  
Camera Position: First Person  
Camera Projection: Perspective 
<p align="center"> 
<video width="400px" controls>
  <source src="/images/playdate3d/TrainGame.mov" type="video/mp4">
  Your browser does not support the video tag.
</video>
</p>

**Tau**  
Type: Game  
Status: Released  
Link: [https://play.date/games/tau/](https://play.date/games/tau/)   
Degrees of Freedom: 1 (1 combined spatial + rotational degree of freedom)  
Performance On Device: Smooth  
Technique: Unknown  
Camera Position: Third Person Dynamic (Over the shoulder)  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/tau.gif" width="400px"> 
</p>  
Notes (conjecture)  
While I’m not sure exactly how they are drawing the cylinder it does look like the game makes heavy use of pre-rendered sprites.

On the playdate, image files are typically tiny because they are often 1 bit per pixel. Because images are cheap, many people choose to pre-render objects from lots of different angles and then display the image that is closest to the true angle between the camera and the pre-rendered sprite during the game. This technique is called [imposters](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/RenderToTextureTools/3/).  

**Skew**  
Type: Game  
Status: Released  
Link: [https://play.date/games/skew/](https://play.date/games/skew/ )  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 0 rotation degrees of freedom)  
Performance On Device: Smooth (exact fps not available)  
Technique: Unknown  
Camera Position: Third Person Dynamic  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/skew.gif" width="400px"> 
</p>  
Notes  
There are a few technical details in this [blog post on Panic’s website](https://news.play.date/news/skew/). I can’t get too much from it though besides the fact that the player is definitely a sprite.  

**Unknown Hat Man Demo**  
Type: Demo  
Status: Unreleased  
Link: [https://nitter.net/timheigames/](https://nitter.net/timheigames/status/1556762636083150852)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 0 rotation degrees of freedom)  
Performance On Device: 50+ fps  
Technique: Triangle Rasterization  
Camera Position: Third Person Dynamic  
Camera Projection: Perspective  
<p align="center"> 
<video width="400px" controls>
  <source src="/images/playdate3d/hatman.mov" type="video/mp4">
  Your browser does not support the video tag.
</video>
</p>
Notes  
We know from the tweet thread linked above that this demo works by rendering triangles. It appears every object in the demo is a true 3D model. As we will talk about later, full 6DOF 3D rasterizers do exist on the playdate, so the author’s claims seem legitimate to me. The author also says performance slows down around 800 triangles. I don’t personally think that is a good metric for measuring draw speed on the playdate but that’s another post. I also suspect this demo uses a full 6DOF capable renderer, but it’s in the 2DOF category for now because I haven’t seen any other degrees of freedom demonstrated.

**The Big Pond**  
Type: Demo  
Status: Released  
Link:[https://okacat.itch.io/the-big-pond](https://okacat.itch.io/the-big-pond)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 0 rotation degrees of freedom)  
Performance On Device: Inconsistent  
Technique: Triangle Rasterization  
Camera Position: Third Person Dynamic  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/bigpond.gif" width="400px"> 
</p>  
Notes:  
This game logs to the console that is loading obj files so I’m pretty sure it is using a triangle rasterizer to render real 3d meshes. Despite not having much on the screen performance is quite inconsistent. I’m not sure why this is. 

**Daily Driver**   
Type: Game  
Status: Demo Released  
Link: [https://gingerbeardman.itch.io/daily-driver](https://gingerbeardman.itch.io/daily-driver)    
Degrees of Freedom: 3 (3 spatial degrees of freedom, 0 rotation degrees of freedom)  
Performance On Device: 52+ fps  
Technique: Isometric Rendering  
Camera Position: Third Person Static  
Camera Projection: Orthographic  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/dailydriver.gif" width="400px"> 
</p>  
Notes  
Daily driver combines sprites that have been pre-rendered from multiple angles ([imposters](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/RenderToTextureTools/3/)) with an isometric view to create a 3D driving experience. Having an orthographic static camera makes 3D rendering a lot easier. It also runs faster than the playdate’s full screen refresh rate of 52 fps because it doesn’t redraw unnecessary raster lines! There are a ton of juicy technical details in the [playdate forum thread](https://devforum.play.date/t/matts-prototypes/826/93?page=5).  

**Mode7Driver**  
Type: Demo  
Status: Released  
Link: https://play.date/dev/ (Once you download the SDK it’s under PlaydateSDK/Examples/Mode7Driver)  
Degrees of Freedom: 3 (2 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: 52 fps  
Technique: Mode7 Subset  
Camera Position: Third Person Over the Shoulder  
Camera Projection: Perspective  
<p align="center"> 
<video width="400px" controls>
  <source src="/images/playdate3d/mode7driver.mov" type="video/mp4">
  Your browser does not support the video tag.
</video>
</p>
Notes    
The original [Mode7](https://sneslab.net/wiki/Mode_7) was a graphics mode on the SNES that allowed developers to draw backgrounds scan line by scan line using special hardware. The pixels of these scan lines were mapped from screen space to texture space by a per scan line affine transformation. This was quite flexible and enabled many different effects, such as the tracks in racing games like [F-Zero](https://www.youtube.com/watch?v=OF1SpvKRMD0) and the [tube from Castlevania IV](https://youtu.be/pafOj9IrtuY?t=78).   

The playdate lua sdk contains a function called [drawSampled](https://sdk.play.date/2.1.1/Inside%20Playdate.html#m-graphics.image.drawSampled) which emulates a subset of Mode7’s functionality to make the player appear to be on textured a 3d plane. 

Unfortunately the drawSampled source code is not available so we cannot know exactly how their technique works.

**Red Terror**  
Type: Game  
Status: Released  
Link: [https://therussianbeargame.itch.io/red-terror](https://therussianbeargame.itch.io/red-terror)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: 20-30 fps  
Technique: Raycasting  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/redterror.gif" width="400px"> 
</p>  
Notes  
Sweet, sweet [raycasting](https://lodev.org/cgtutor/raycasting.html) like Wolfenstein running on the playdate. Looks smooth. More technical details on this [playdate forum post](https://devforum.play.date/t/help-error-search-crash-in-my-raycasting-3d-game/11555/31). One nugget of info is it uses [mip-mapping](https://developer.valvesoftware.com/wiki/Mipmapping) on the walls to make them less noisy when they are far away. Neat! Big props to the dev, this looks pretty smooth.

**Wi-Fi Dungeon: Organism Online**  
Type: Game  
Status: Unreleased  
Link: [Wi-Fi Dungeon: Organism Online (Announce Trailer)](https://www.youtube.com/watch?v=ywhRQ4-nlEg)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: Unknown  
Technique: Raycasting (suspected but not confirmed)  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/ywhRQ4-nlEg?si=124o_rtw-4k-emw8" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</p>


**Unknown Ray Casting Engine #1**  
Type: Demo  
Status: Unreleased  
Link: [https://www.youtube.com/watch?v=ablhAP85NNw](https://www.youtube.com/watch?v=ablhAP85NNw)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 1 rotational degrees of freedom)  
Performance On Device: ~33 fps based on the video  
Technique: Raycasting  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/ablhAP85NNw?si=JAMQCVo2D_QxlclP" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</p>
Notes:  
This raycaster has crisp textures and graphics look very clean compared to other raycasters on the playdate. I can’t tell if this is just the artwork though.

**Unknown Ray Casting Engine #2**  
Type: Demo  
Status: Unreleased  
Link: [https://www.youtube.com/watch?v=rGlUOxdprQM](https://www.youtube.com/watch?v=rGlUOxdprQM )  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: 29 - 45 fps based on the video  
Technique: Raycasting  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/rGlUOxdprQM?si=nu13ohbRHeRwgw-f" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</p>
Notes:  
Fastest raycasting engine I’ve seen so far for playdate. Wonder what makes all these engines different? Textures also look good but can’t tell if this is due to the engine or just the artwork. Would love more details.  

**Wild Dungeons**  
Type: Demo  
Status: Released  
Link: [https://crankworkgames.itch.io/wild-dungeons](https://crankworkgames.itch.io/wild-dungeons)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: 5-15 fps  
Technique: Raycasting  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/wilddungeons.gif" width="400px"> 
</p>  
Notes  
The developer of this game does not have a playdate to test on yet and it seems like the on device fps is suffering a little as a result. At the time of writing they are still pushing improvements out and getting a decent amount of user feedback so it seems like things could improve dramatically.

**Legend of Etad**  
Type: Game  
Status: Released  
Link: [https://play.date/games/legend-of-etad/](https://play.date/games/legend-of-etad/)    
Degrees of Freedom: 2 (2 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: Smooth  
Technique: Pre-rendered composition  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/etad.gif" width="400px"> 
</p>  
Notes  
Another creative, completely out of left field idea, this game pre-renders discrete parts of the world and then overlays the pre-rendered pieces together to achieve a full scene. I’m gonna call this pre-rendered composition. I’m not gonna lie, I don't totally understand it. Many more details in this [gold mine of a twitter thread](https://nitter.net/GarethIW/status/1515778382486724617). All of this comes at a cost of course. The game jumps between positions instead of continuously walking from one place to another.  

**Dioragame**  
Type: Game  
Status: Unreleased  
Link: [https://dioragame.com/devlog/](https://dioragame.com/devlog/)  
Degrees of Freedom: 4 (3 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: [30 fps](https://nitter.net/the3dprintist/status/1561780780275081216?t=2JRksrsKkPYyL4kccMPL8w)    
Technique: Isometric Rendering  
Camera Position: Third Person Dynamic  
Camera Projection: Orthographic  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/diora.gif" width="400px"> 
</p> 
Like Daily Driver but with a rotatable camera. Everything in the scene is an imposters. There’s a pretty detailed engine post if you go to [https://dioragame.com/devlog/](https://dioragame.com/devlog/) and choose the entry “The Diorama Engine”. Like Daily Driver this game gets a large performance boost by having a far away third person orthographic projection.

**P-Racing**  
Type: Game / Engine  
Status: Game Release, Engine Open Sourced  
Game Link: [https://play.date/games/p-racing/](https://play.date/games/p-racing/)     
Engine Link: [https://github.com/risolvipro/playdate-mode7](https://github.com/risolvipro/playdate-mode7)    
Degrees of Freedom: 4 (2 spatial degrees of freedom, 2 rotation degrees of freedom)   
Performance On Device: Smooth  
Technique: Mode7+  
Camera Position: Third Person Dynamic (Over the Shoulder),  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/pracing.gif" width="400px"> 
</p> 
Notes:  
P-Racing is appears to use a combination of Mode7 style rendering (see [here](https://github.com/risolvipro/playdate-mode7/blob/b853a8064d0494c74fc31823ea076c8486ecf5d2/src/pd_mode7.c#L675)) to draw the road similar to Mode7Driver and [imposters](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/RenderToTextureTools/3/) for the cars (see [here](https://github.com/risolvipro/playdate-mode7/blob/b853a8064d0494c74fc31823ea076c8486ecf5d2/src/pd_mode7.c#L428)). I also suspect it’s using non camera facing textured quads for the street lights, and untextured triangle meshes for the buildings (neither appear to be in the open sourced engine sadly). It’s really cool seeing a game use a combination of so many techniques. I’m calling this Mode7+ since it appears more capable than what can be built using the Playdate’s [built-in Mode7 drawing function](https://sdk.play.date/2.1.1/Inside%20Playdate.html#m-graphics.image.drawSampled).

**Super Agent**  
Type: Game  
Status: Unreleased  
Link: [https://nitter.net/RisolviPro](https://nitter.net/RisolviPro/status/1710043502413852815)  
Degrees of Freedom: 3 (3 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: Unknown  
Technique: Unknown  
Camera Position: Third Person Dynamic  
Camera Projection: Perspective  
<p align="center"> 
<video width="400px" controls>
  <source src="/images/playdate3d/super-agent.mov" type="video/mp4">
  Your browser does not support the video tag.
</video>
</p>
Notes  
Just like in the train game, we see that the tree billboards are pre-rendered at different scales to make them look better far away. This is a form of [mip-mapping](https://developer.valvesoftware.com/wiki/Mipmapping).

**Super Agent Mountain Scene**  
Type: Demo  
Status: Unreleased  
Link: [https://nitter.net/RisolviPro](https://nitter.net/RisolviPro/status/1717659562738655470)  
Degrees of Freedom: 2 (2 spatial degrees of freedom, 2 rotation degrees of freedom)  
Performance On Device: Unknown  
Technique: Unknown  
Camera Position: ???  
Camera Projection: Perspective  
<p align="center"> 
<video width="400px" controls>
  <source src="/images/playdate3d/super-agent-mountain.mov" type="video/mp4">
  Your browser does not support the video tag.
</video>
</p>

**PlayDOOM**  
Type: Demo  
Status: Released  
Link: [https://playdate-wiki.com/wiki/PlayDOOM](https://playdate-wiki.com/wiki/PlayDOOM) (Sadly it appears at the time of writing there is not fast build Playdate OS 2.11+)    
Degrees of Freedom: 4 (3 spatial degrees of freedom, 1 rotation degrees of freedom)  
Performance On Device: ~15 fps  
Technique: DOOM (Ray Casting+)  
Camera Position: First Person  
Camera Projection: Perspective  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/pdoom.gif" width="400px"> 
</p> 

Notes  
Some neat porting notes are available on the [playdate forums](https://devforum.play.date/t/doom-on-playdate/852). Also it’s DOOM!  

**Mini3D**  
Type: Engine  
Status: Released  
Link: [https://play.date/dev/](https://play.date/dev/) (Once you download the SDK it’s under PlaydateSDK/C_API/Examples)  
Degrees of Freedom: 6 (3 spatial degrees of freedom, 3 rotational degrees of freedom)  
Performance On Device: Highly scene dependent can be 52, can be 1 if you push it too hard  
Technique: Triangle Rasterizer  
Camera Position: Flexible    
Camera Projection: Supports Perspective and Isometric  
<p align="center"> 
<video width="400px" controls>
  <source src="/images/playdate3d/mini3d.mov" type="video/mp4">
  Your browser does not support the video tag.
</video>
</p>
Notes  
A solid rasterized 3D engine with a full scene hierarchy and Lua bindings that comes hidden in the playdate sdk. It has multiple different settings where you can trade performance for quality especially around depth conflicts ([zbuffer](https://en.wikipedia.org/wiki/Z-buffering)/[ordertable](https://psx.arthus.net/sdk/Psy-Q/DOCS/TECHNOTE/ordtbl.pdf)/per object depth). Honestly, this thing is pretty cool, and its code has taught me a lot about how the playdate works. I’m not sure why this is not used to make 3D games. Maybe people don’t know about it? Maybe it doesn’t scale well to huge worlds? Do playdate games even have huge worlds?

**Mini3D+**  
Type: Engine  
Status: Released  
Link: [https://github.com/nstbayless/mini3D-plus](https://github.com/nstbayless/mini3D-plus)  
Degrees of Freedom: 6 (3 spatial degrees of freedom, 3 rotational degrees of freedom)  
Performance On Device: The kart demo that comes with it runs at 30-40 fps on my device regardless of what the readme says (it claims 20 fps, I have filed an issue on the repo)  
Technique: Triangle Rasterizer  
Camera Position: Flexible  
Camera Projection: Supports Perspective and Isometric  
<p align="center">
	<img loading="lazy" src="/images/playdate3d/mini3d.gif" width="400px"> 
</p> 
Notes  
This library is an open source extension of mini 3D with several large improvements:  

- Mesh clipping at camera (allows rendering faces which are partly behind the camera)
- Textures
- Imposters
- Collision detection
Seems like mostly sphere triangle but the cart demo plays nicely

Downsides  

- Texture projection is approximate to save on speed but only noticeable when triangles are very close to the camera
- Seems abandoned but it also a good starting place / resource for building an even more ambitious engine

**Common themes**  
The biggest thing that stuck out to me when I looked at all these games/demos was how pre-rendering techniques (imposters, mip-mapping, even full motion video) seem to be hugely popular on the playdate. Given its hardware limitation, it does seem like even in triangle-rasterized 3D games, if the scene is large, devs will likely have to use imposters to mimic tiny, even more detailed meshes. I’m personally also curious to see how far you can go with imposter only graphics. Can you make a 6DOF 3d first person shooter using only imposters? I think you might be able to if given the right level design constraints.  

Another thing that struck me was that very few playdate 3d games use triangle rasterization, even though there is a built in engine for it. There are certainly hardware constraints which make triangle rasterization less appealing than on a platform with a gpu, but I suspect this is also due to Playdate developers choosing to make their own 3d engines which emulate retro techniques for the fun of it (and that’s awesome). I am definitely curious about how far triangle rasterization can be pushed on the playdate.  

**What is the future of 3D on the playdate?**  
While all of the games/demos above are wildly impressive, I don’t think people are done pushing the limits of the hardware. I’m excited for what is to come!
