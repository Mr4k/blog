Title: Learn You a NeRF
Date: 2023-01-10
Modified: 2022-01-10
Category: graphics, pytorch, deep learning
Tags: graphics, pytorch, deep learning
Slug: nerf
Authors: Peter Stefek
Summary: Radical Radiance

<p align="center">
	<img src="/images/nerf/excavator.png" width="80%"> 
</p>   

**A Computer Vision Problem**  
Over the years computer graphics has gotten really good at rendering virtual worlds known as forward rendering. Forward rendering means taking some description of a 3d world stored in a computer and turning it into a realistic looking 2d image captured from a particular perspective inside that world.  

The inverse problem, turning an image or collection of images into a 3d scene, is also very interesting. Up until recently these problems were solved in unrelated ways but new techniques called inverse graphics have tied the two problems together. NeRF (Neural Radiance Fields) is a wildly popular[ref]Over 50 papers derived from NeRF were submitted to CVPR in 2022. This number is courtesy of [Frank Dellart](https://dellaert.github.io/NeRF22/) [/ref] inverse graphics technique which beautifully formulates the problem in terms of forward rendering.  

**Forward Rendering via Ray Tracing**  
Before we talk about NeRF, I want to quickly go over the basic ideas of ray tracing, a forward rendering technique. NeRF relies on a very simplified version of ray tracing so understanding the basic ideas behind ray tracing is useful for understanding NeRF. If you already are familiar with ray tracing feel free to skip ahead! For a more comprehensive explanation please see the wonderful [Scratchapixel](https://www.scratchapixel.com/lessons/3d-basic-rendering/introduction-to-ray-tracing/how-does-it-work.html).  

The basic idea behind ray tracing is to use a virtual pinhole camera to capture imaginary light rays and turn them into an image. This approximates how cameras work in real life. Below is a diagram of an ideal pinhole camera:  
<p align="center">
	<img src="/images/nerf/camdiagram1.png" width="70%"> 
</p>  
Light from the sun or other light source bounces off of an object and into the pinhole of the camera projecting the object upside down. In our virtual pinhole camera we usually assume the pinhole is infinitely small and has sufficient light exposure so there is no need for a lens and therefore no depth of field effects.  

An important notational point is that graphics programmers prefer to talk about rendering in terms of an "eye" placed at the pinhole and an "image plane" placed in front of the camera one focal length unit away. This has no real life analog but is the right side up version of the image captured by the pinhole camera so it is easier to work with. Here is a diagram of the image plane:
<p align="center">
	<img src="/images/nerf/camdiagram2.png" width="70%"> 
</p> 
One way you might try to render an image would be to sample rays of light from the light sources in the scene (the imaginary sun or other lights) and bounce them around until you have accumulated enough inside the pinhole camera to see a finished image. However given the fact that our virtual pinhole is infinitely tiny this would take forever. Even if it wasn't this would be super inefficient as most samples would miss the camera and therefore be useless. 
<p align="center">
	<img src="/images/nerf/camdiagram3.png" width="50%"> 
</p> 
Instead of casting rays from the light source to the camera, we can be more efficient by casting "reverse" rays out from the camera. We can do this because we know that the color of each point[ref]I maintain this is technically true for a single point. However in ray tracing we are estimating the color for an entire pixel which is a rectangle, therefore serious ray tracers use techniques such as Monte Carlo sampling to sample many points inside each pixel's rectangle and merge them together to get a final color[/ref] on the image plane is uniquely determined by the light coming into the camera's pinhole from a certain direction. These reverse rays will bounce around the scene until they hit a light source. By the time they have done that they will have followed the exact paths that the incoming light ray from that source would follow and therefore we can determine how much energy is left when that ray of light finally hits our sensor[ref]If you are scratching your head at how this is accomplished you are absolutely correct. In fact I'd be a little worried if you weren't confused. The answer to this question is incredibly complex and a good chunk of the subfield of computer graphics called Physically Based Rendering is dedicated to creating more and more accurate approximations. A great place to start is the [Pbr book](https://www.pbr-book.org/3ed-2018/contents)[/ref].  

One important property of this version of ray tracing I want to highlight is that we can terminate "reverse" rays early and make an approximation of the amount of light coming from the remainder of the tracing process instead of continuing to trace. All ray tracers I'm aware of do this to some extent. Usually there is a max bounce depth before early termination.  

The simplest form of this is to terminate immediately when hitting the first object instead of doing any bouncing and make a guess as to the amount of light hitting that object at that particular angle. This is the type of ray tracing NeRF relies on.  

**So what is NeRF anyway?**  
Neural Radiance Fields (NeRF) takes in a set of cameras with the following known parameters: position, orientation and field of view as well as the corresponding ground truth photographs taken from each of those cameras. It then learns a 3d representation of the underlying scene. This 3d representation can be used to render the scene from novel viewpoints or turned into a mesh.  

NeRF represents the 3d scene with a so-called radiance field. A radiance field can be viewed as two different functions $\mathbf{\sigma(pos)}$ and $\mathbf{c(pos, dir)}$ which together represent a volume.  

$\mathbf{\sigma(pos)}$ is a function which takes a position in 3d space and returns the probability density of encountering a particle at that location. This quantity is sometimes called the attenuation but the NeRF authors refer to it as the volume density.  

$\mathbf{c(pos, dir)}$ is a function which takes a position in 3d space and a direction then returns the color of light reflecting off that particle. The return type of $\mathbf{c(pos, dir)}$ is a 3 component RGB vector with components from $[0, 1]$.  

One thing you might notice is that $\mathbf{\sigma(pos)}$ does not depend on the direction of the ray while the color function does. This is an explicit choice by the NeRF authors, who say that the angle at which you look at an object should not determine whether or not it is there. In contrast the author's acknowledge that the color of the object can be influenced by the ray direction especially if its surface is reflective. For example, consider a mirror, you can look at the same part of it from different angles and you will see different images.  

To create a 2d view from this radiance field the authors use a forward rendering technique from an area of computer graphics called volume rendering.

