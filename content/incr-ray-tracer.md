Title: Slightly Incremental Ray Tracing In One Weekend
Date: 2020-10-13
Modified: 2020-10-13
Category: ray tracing, self adjusting computation
Tags: ray tracing, self adjusting computation
Slug: incr-ray-tracer
Authors: Peter Stefek
Summary: Reflecting on an experiment
Status: draft

**Incremental Computation**  
Incremental computation is a programming abstraction which gives users a way of writing calculations such that, when some of the inputs change only a relevant subset of the calculation has to be recomputed. 
 
The above definition is kind of vague and honestly it's best to start with an example. Let's take a look at the following mathematical expression.  

$$ (a+b)*c $$  

We can evaluate this expression in two steps, First by adding a and b together, then by multiplying the result by c. We can even write this computation out as a dependency graph:
<p align="center">
  <img src="/images/incr-ray-tracer/comp-dag.png" width=50%> </img>
</p>
Here the variables a,b,c are referred to as leaves because they have no dependencies. Now what if after we do out initial computation, the value of c changes?
<p align="center">
  <img src="/images/incr-ray-tracer/comp-dag-2.png" width=50%> </img>
</p>
We can see from this graph that since (a+b) does not depend on c, we do not need to recompute it. Now imagine scaling this idea up to larger, more complex expressions. If you are familiar with spreadsheet software, you might have flashbacks to the huge interlinked computations created there. This brings us to a second, more intuitive definition of incremental computation: Microsoft Excel on steroids.
<p align="center">
  <img src="/images/incr-ray-tracer/excel-on-steroids.png" width=50%> </img>
</p>
There are several different frameworks and projects that seek to give the programmer these incremental computing abilities. These include several rust libraries, Facebook's Skip Lang and an ocaml library called Incremental. I choose play around with Incremental for my experiments, because it seems to be the most battle tested option.

**Pros/Cons**  
So far it may seem like incremental computation frameworks are magical abstractions which allow programmers to write incredibly efficient programs with ease. So why don’t we all use them? Well like most things in software, incremental computation frameworks come with a few trade offs. 
Pros:


-  An incremental computation framework provides a simple abstraction around optimizing computations which allow programmers to write more efficiently computable programs with low mental overhead
- The abstraction provided by these frameworks generalizes well to many different types of problems

Cons:
 
- There the framework has non negligible overhead both in speed and memory. We need to make sure each incrementalize step is heavy enough or else the overhead of the framework could dwarf the actual time it takes to recompute the expression from scratch.
- Similar to the above point, the initial computation of the algorithm will be slower. You need to decide if you will be recomputing often enough to justify this slow down.
- There are always limitations to any particular abstraction. Therefore you will likely be able to come up with a custom algorithm for a specific problem that is much faster than the result you get using an incremental computation framework

Those interested in exploring these tradeoffs in detail or wanting to peek behind the curtain will enjoy this talk (TODO add link).

**How do people use Incremental Computation**  
So where are people actually using incremental computation? As far as I know, there are a few different areas where these ideas are being applied:
 
- Finance, especially to deal with large spreadsheet-like calculations. Most finance companies are a little vague about what they actually do, so I don’t know too much about this use case.
- Compliers, the rust compiler explored these techniques when building their incremental compilation features
- Databases, the company Materialize uses technology based on the Timely Dataflow Model developed by Microsoft Research to create a better streaming database
- User Interface, user interfaces are a perfect place for incremental computation because it's important to figure out which components you actually need to re-render when something on the page changes.

**How people shouldn't use Incremental Computation**  
Now we can get to the fun part. Over the last week, I tried to play around with Incremental (and picked up scraps of OCaml) to get a better idea of how to actually build incremental computations. 
 
Of course the first thing I did was to break the library, but after that I decided I wanted to write a small but non trivial project which used it. Since I don't have legions of highly paid quants to create large spreadsheets, I decided that use case was off the table. I felt that building a toy compiler or database would be at least a week long project by itself without incrementalizing it, so I ruled those out as well. I'm also not a huge frontend person, so a ui framework didn't seem appeal to me that much. Instead I decided to try something different:

<p align="center">
  <img src="/images/incr-ray-tracer/in-one-weekend.jpg" width=50%> </img>
</p>

I decided to implement (3/4s of) Peter Shirley's Ray Tracing in One Weekend tutorial in OCaml and then try to incrementalize a part of the ray tracing step. I will be honest, this experiment was not the most amazing thing in the world, but I think I learned a lot about what not to do with incremental computation. 

Is this a terrible idea? Absolutely! But we can learn a lot but understanding why exactly it's terrible.

So how does the graph based incremental computation I talked about above even fit in with something like ray tracing to being with? It turns out shooting light rays into space can be thought of as a graph.

<p align="center">
  <img src="/images/incr-ray-tracer/ray-trace-graph.png" width=75%> </img>
</p>

In the image above, the blue nodes are nodes in an incremental computation graph and the arrows are directed edges between them. We can also fit additional metadata about our scene into the graph. For example, each object that a light ray hits will have a material associated with it.
 
In my model, these materials have color but also a "fuzziness" parameter that represents how reflective the material is. If the material is perfectly reflective, all light rays are reflected precisely over the collision normal, otherwise there is a random jitter applied to the outgoing direction of the ray. The size of this jitter is controlled by the "fuzziness" parameter. 
 
So the material parameters affect the final image color, as well as the direction of the outgoing ray. When the fuzziness of a material changes, we must recast all subsequent rays bouncing off of any objects which use that material. We can represent this dependency in our graph as follows: 

<p align="center">
  <img src="/images/incr-ray-tracer/render-graph-with-materials.png" width=75%> </img>
</p>
In the above graph, if the fuzziness of a material changes, we will recast all subsequent rays.
 
Handling changing material parameters will be the focus of our render. The idea is that after an expensive render an artist could tweak a few material parameters without having to re-render the entire screen.
 
But what about moving objects around? For this experiment, I decided to not to implement this (fairly important) feature b/c I could not find a way to make it fit as nicely into the abstraction provided by incremental. If I really wanted to do this, I'd basically just store all the rays inside an spartial acceleration structure like a octtree. Then whenever a sphere moved I'd query the set of rays intersecting it before it moved and the new set of rays interesting it after it moved. I'd then fire each of the nodes associated with these rays.  
 
So anyways does this material editing approach actually work? The answer is somewhat.
 
Here's a gif of it in action (in real time)!
<p align="center">
  <img src="/images/incr-ray-tracer/trace-reload.gif" width=50%> </img>
</p>

In the above gif, each material's color and fuzziness parameters are being changed one sphere at a time. Instead of doing a full render each time something changes, incremental figures out which rays need to be recomputed. The image above looks grainy because I'm casting only about 10 rays per pixel (for reference Shirley's code which this is based off does 100 rays per pixel). To see why I'm only taking 10 samples per pixel, let's look at some performance numbers: 

<table style="width:90%">
  <tr>
    <th># Rays Cast (x129600)</th>
    <th>Non incremental render time (s) </th>
    <th>Inital incremental render time (s) </th>
    <th>50th percentile edit time (s)</th>
    <th>99th percentile edit time (s)</th>
  </tr>
  <tr>
    <td>1x</td>
    <td>0.864982</td>
    <td>2.475083</td>
    <td>0.010557</td>
    <td>0.077140</td>
  </tr>
  <tr>
    <td>2x</td>
    <td>1.559165</td>
    <td>5.213949</td>
    <td>0.015602</td>
    <td>0.157500</td>
  </tr>
  <tr>
    <td>5x</td>
    <td>4.088101</td>
    <td>12.362720</td>
    <td>0.032054</td>
    <td>0.405897</td>
  </tr>
  <tr>
    <td>10x</td>
    <td>7.763904</td>
    <td>24.869927</td>
    <td>0.088754</td>
    <td>0.901066</td>
  </tr>
  <tr>
    <td>20x</td>
    <td>14.815373</td>
    <td>121.544599</td>
    <td>0.840104</td>
    <td>11.20906</td>
  </tr>
</table>
</p>
The benchmarks above are all taken with 100 spheres, each with its own unique material. Each sphere's material was changed once and I recorded the time it took to rerender after each sphere's material had changed. Each ray cast bounced a maximum of 10 times through the scene. The non incremental version is my implementation of Peter Shirley's code. Both versions of the code are single threaded.
 
Okay let's talk though this data quickly. The first four rows of the table tell a consistent story. The initial render has about a 3x to 4x overhead and editing the sphere material results is significantly faster than a full redraw. All the numbers seem to scale linearly with the number of rays cast as expected.
 
But what about that fifth row! Runtimes seem to explode in the incremental version. I was really surprised by this result. I don't have a definitive answer to this question but it did make me rethink the overhead that a library like Incremental brings to a program.

**Overhead of Incremental**  
At first glance, accounting for the runtime overhead of incremental seems like it would be pretty simple. Each node that is fired requires a little bookkeeping on the incremental side. As far as I know this overhead is typically on the order of ~50-150ns per node on commodity hardware and should grow linearly. The operations we are wrapping are a lot slower than that, so the firing overhead should not be a problem.  
 
While the bookkeeping overhead is minimal, there is a second kind of overhead introduced by using this library. This is the memory overhead. Incremental nodes are not super light weight by themselves and we have at least one node per bounce of light. In fact there were 20678306 nodes in the row 5 computation. Each node weights at least 216 bytes without counting the data it is pointing to. I didn't think about this memory overhead at first because I was concerned with runtime, not mememory usage.  

Of course, things are always more complex than they appear and I now believe that the runtime spike we saw earlier is caused by the memory overhead. 

Basically, the memory that most programs have access to is not real memory, but an abstraction known as virtual memory. That virtual memory is divided into "pages" of about 4KB each (on my computer at least). These pages are typically stored in RAM. But when memory gets tight, the operating system uses a variety of tricks to create the illusion of having more RAM then it really does. These tricks include temporarily storing pages to disk, temporarily swapping entire programs onto disk, and compressing memory. The drawback of these tricks is a considerable overhead when accessing affected memory. 

It's worth saying here, I'm not an operating systems expert by any means. The evidence I have leads to less of an open and shut case and more to this:
<p align="center">
  <img src="/images/incr-ray-tracer/charlie.jpg" width=50%> </img>
</p>

**Take Aways**  
So was incremental a good fit for ray tracing? The answer is definitely not. For one, ray tracing is better done in high parallel environments, and incremental is single threaded. A second reason is that ray tracing is a really well studied problem and there are already faster incremental approaches which make use of a lot of problem specific domain knowledge.
 
So why did I do this project? For me, it was a good way to play around with the limits of incremental computation. I think I learned a lot about structuring these graph computations and some of the specific pain of the Incremental library. However I'm still pretty new to all of this so if you have any tips for me I'd love to hear from you!

Have questions / comments / corrections?  
Get in touch: <a href="mailto:pstefek.dev@gmail.com">pstefek.dev@gmail.com</a>   
