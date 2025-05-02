---  
date: 2025-05-02  
title: "Writeup: Perspective"  
---  

This is a writeup about my 8K intro ["Perspective"](https://youtu.be/24RZGnTAIVM?si=ksqmiKKP6HYafikL&t=1094).  

### Introduction  
Going into Revision this year, I wanted to put in a bunch of effort. So besides making 2 music entries (which both got 6th place!) and a [video entry](https://www.youtube.com/watch?v=JIvileqNhNQ) (which won third place), I decided to also put in a bunch of effort into my coding productions.  
For the better part of a year, I toiled away trying to make the nicest intro I could, and keeping a devlog of it in the process. It took 3 months to make and I thought the result was great.  

That intro was Vitrage, my 4K entry. It took 11th place\*.  
Then I made Perspective in a 9 days and it took 4th.  
People liked it so much some were asking me how it was done, which is crazy! I wanted to write about my demos for a while so here's the full explanation starting from the concept.  

<sup>\*I'm still really proud of my 4K! I think it represented a significant improvement from the previous years, and it deserved 11th because the competition was extremely fierce this year.</sup>

### Concept  
I was expecting the concept of Perspective to carry the demo.  
Some people assumed it was based off the games Superliminal or Viewfinder, but it's actually based on data-moshing clips I've seen around, like [this one](https://www.youtube.com/watch?v=kIBsAvUwkUQ) (I can't find the original but it's the same idea).  

Video clips are compressed by having only one frame, then telling the program which pixels to move around for the next frames.  
If you then take out the first frame, you can make the previous frame move around in weird ways, which can be then used to manipulate perspective.  
My thinking was similar, but instead of moving the pixels around, I can just project the frame on the next scene from where the camera isand move the camera around from there, which is exactly what I did.  
From then on, the ideas just flourished in my head and everything came naturally.
```glsl
else if(scene == 6) {
    // Fuck it, revision logo, I'm out of ideas
    return revisionScene(p);
}
```

Let's start with the simplest part:  

### Soundtrack  
The soundtrack was written using [`sointu`](https://github.com/vsariola/sointu) by pestis, a fork of 4klang which is so much easier to work with. It has a nicer UI, built-in unison and polyphony on oscillators, and most importantly - it **saves the state** in your DAW project, which is the biggest drawback of 4klang - numerous times I forgot to save its state and then the program crashed.  
So I highly recommend it.  

Most of the drum sounds were presets and I programmed the lead and pad patches.  
There are a few tricks I used which could be helpful:  
- To modulate the reverb stopping (at the end of each scene), I added an instrument which sends a negative volume to the reverb aux with an envelope, and then played a note in it whenever I wanted the reverb to stop.  
    - Because it sounded better, I also added a release to the negative envelope so that the reverb gradually comes back during the next scene. Otherwise the previous note just shows up.  
- To make the hats more interesting, I linked the decay of the envelope to the note that was playing (using `loadnote`) and then played higher hat notes on beats where I wanted them to be more open.  
  This works because hats are noise which doesn't have a note, so they're not really linked to the note in any way, enabling this modulation.  

### Effect  
So, to get the perspective warp effect, it's actually extremely simple.  
The code is comprised of one raymarching shader. While initially I wanted to use kind of a reverse shadow map, I gave that up because I didn't have much time to learn that and the raymarching solution worked well enough.  

The shader gets these 7 parameters:  
- The camera position vector  
- The camera target vector (where the camera is looking at)  
- The projector position and target vectors  
- The projection texture  
- Which shot to render  
- 2 special shot parameters (used to switch up the sky and object colors in some of the shots)

Then, we want the projection texture to be "projected" from the projector point, pretty similar to an actual projector, and the camera is free to move around independently of that point.  
How do we actually do that?  
Here's the relevant code.  

We start by getting our projection directions just like you would for a normal raymarching camera:  
```glsl  
vec3 forward = normalize(projectionTarget - projectionPoint);  
vec3 left = normalize(cross(forward, vec3(0, 1, 0)));  
vec3 up = normalize(cross(left, forward));  
```  

Then, we take our ray direction and get out each component from it:  
```glsl  
vec2 uv = vec2(dot(rayDirection, left), dot(rayDirection, up));  
```  

This doesn't look right, so after trial and error I found this formula to fix it:  
```glsl  
float forwardFactor = dot(forward, rayDirection) / FOV;  
uv /= forwardFactor;  
```  

Now the image was stretched out, so there's some aspect ratio fixing:  
```glsl  
uv.x *= _res.y / _res.x;  
uv += .5;  
```  

And finally, sample the projection texture (making sure to stay in bounds):  
```glsl  
// If forwardFactor is negative, the direction is behind the projector  
bool inBounds = clamp(uv.x, 0., 1.) == uv.x && clamp(uv.y, 0., 1.) == uv.y;  
return (forwardFactor > 0. && inBounds) ? texture(_tex0, uv) : vec4(0);  
```  

You can see the full function in the [repository](https://github.com/altigamusic/Perspective/blob/83c0a1cfbe5bc689be834655b6e048a89d94bd62/src/FragmentShader.glsl#L422).  

We run this function on every point we hit in the scene, and then blend the resulting color into the actual scene using its alpha.  
If there's no projection, the alpha is 0 and we get the scene, otherwise we get our result.  

Now there are a few problems with this, so let's address them:  

**What if an object is behind another object? It shouldn't get projected on, then.**  
For that, we use the same trick we use for shading: we raymarch from our hit point to the projection point, and if the resulting distance is smaller than the distance to the projector, that means we hit some other thing on the way there and we return 0.  
You can see that in [the `getProjection` function](https://github.com/altigamusic/Perspective/blob/83c0a1cfbe5bc689be834655b6e048a89d94bd62/src/FragmentShader.glsl#L448).  

**What if we don't hit the scene?**  
To take care of that, it's actually pretty simple: we do the exact same projection trick on the sky.  
Except there is not hit point to calculate, so we just use our actual ray direction.  
Because the sky doesn't change based on which point you're on in the scene, it's the same as projecting from the projection point.  

Of course, we march in the scene first to make sure we didn't hit any object.  
You can see this in [the `getSkyProjection` function](https://github.com/altigamusic/Perspective/blob/83c0a1cfbe5bc689be834655b6e048a89d94bd62/src/FragmentShader.glsl#L440).  

This is in general how the effect works.  

### Composition
How do we switch between scenes then?  
To do that, I wrote a little struct that contains all the parameters passed to the shader.  
Then, each scene is a function that gets the time as a value from 0-1, and returns this struct.  
The time being 0-1 is useful because we can use easing functions on it and we can set the scene's progression without worrying about the amount of time it takes up in the demo.  

The intro loop calls the `intro_do` function and passes it which scene to render and the current time.  
The `intro_do` function then calls the function for the appropriate scene.  

However, we still need to render the projection texture!  
So to do that, the `intro_do` function remembers which scene was called last time, and when the scenes switch up, it triggers the rerender.  
The scene is rendered to a framebuffer based on which scene we're on - for most scenes this renders the last scene at time 1 (when it ended), but on the first scene it actually renders the *next* scene at time 0 (when it starts), because we move *into* the projection instead of away from it.  

To do the sphere and star scenes, we render the scene itself to the texture every beat and then rewind it to the previous position.  

Here's a list of scenes and how they are rendered:
- "Perspective" text - this one is just a bunch of rectangles and circles added and subtracted from each other. I got the idea from LJ, after he told me that's how he did the text on [Dubplate](https://www.youtube.com/watch?v=d2xD27Hkg1w) (but much more rushed).
- Room of spheres - negative cube with limited-domain-repeated spheres inside.
- Pyramids - these are actually domain-repeated octahedra which are cut off by the floor.
- "Road" - these are just domain-repeated cylinders on the side of the camera.
- "Jungle" - these are domain-repeated segments connecting every point on the 3D grid with all its neighbors.  
  I initially wanted to have only some of those segments show up but I actually liked it better with all of them. It reminded me of these [playground towers](https://assets.kompan.com/cdn-cgi/image/width%3D640%2Cformat%3Dauto%2Cquality%3D70/$web/prod/43fe8844-17b5-414e-bfcd-2b4d59fedc18/CRP302501-1101_CAD1.png) I used to like as a child.
- The room of spheres appears again - the special parameter is used to switch up each beat. It only goes up to 3, then it repeats itself.
- Stars - 3 octahedra, one for each axis, stretched in the direction of the axis then overlayed on each other.
  The special parameters are used to switch up the sky and how many/where the stars are.
- "Perspective" text appears again with a different sky thanks to the special shot parameter.
- Revision logo - this was entirely stolen from [pestis](https://www.shadertoy.com/view/7sjfRR) (and extended to the 3rd dimension) basically on the last hour because I needed something in the final 16 beats of the track.  
  There's a white cylinder above the logo so that it remains hidden until the camera dips under it for the big surprise.

### Tooling  
For both my 4K and 8K, I made some debug tools using ImGui and messages.  
The tools allowed me to set shader parameters for easier iteration, and to freely move around the scene to test the camera and shader.  
This proved extremely useful and I recommend everyone to do so as well. Next year, I'll invest even more into my tooling as it helps immensely.  

### Conclusion  
I submitted this 8K at 4AM on Saturday, about 5 hours before the deadline.  

I really felt it was unfinished - I can point to a million problems I have with it, but I wanted to get it out and not have to wait for next year.  
Imagine my surprise that people also really enjoyed it and said it was a party standout for them!  

To me this serves as an important reminder that we really don't know what people will like, and you should really get as many ideas out as possible to see what sticks.  
That was my motivation behind submitting 5 entries this year, and hopefully next year I'll submit more :)

Thanks for reading!