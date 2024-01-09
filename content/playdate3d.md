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
	<img src="/images/playdate3d/tau.gif" width="400px"> 
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
	<img src="/images/playdate3d/skew.gif" width="400px"> 
</p>  
Notes  
There are a few technical details in this [blog post on Panic’s website](https://news.play.date/news/skew/). I can’t get too much from it though besides the fact that the player is definitely a sprite.
