---
date: 2026-07-05
title: "Writeup: ProgScape (Nova 2026)"
---

Hello!  
I've had the opportunity to make music for Oni of Chapterhouse on this year's 64k demo at Nova.  
I thought I'd write a quick post about the process for anyone curious.

# Conception

After working together last year, Oni reached out via Discord to work with me on music again.  
He showed me his work in progress and told me he's looking for an 80s prog-rock song.

This raised a challenge: prog rock is acoustic in nature.  
How would I mimic acoustic instruments in a 64k demo? I definitely couldn't record any to fit in there.  
I _could_ try and use samples, but last year we worked using Sointu (a fork of 4klang) because 4klang was supported in Oni's Linux demo toolset.  
However, 64klang and WaveSabre, the two synth I know that can sample, don't run on Linux.  
I still don't have the know-how to code my own synth, so I was out of options.

### But wait...

I remembered that in this year's sizecoding seminars, Gopher gave a talk about how he rewrote 64klang with AI, and I remembered him saying someone offered to make a Linux build.  
So I sent the repo link over to Oni and asked if he could make it work.

Until then, I decided it's probably a good idea to get an initial sketch out, because we didn't have all the time in the world.  
So I started working on my sketch using 64klang.

I used wavetables to sample kicks, snares, a bass and a ride cymbal, and got inspiration from a Phase Plant preset to make an organ patch.  
I sent the sketch over to Oni and he liked it!

So now it was time to try and get everything to compile.  
I sent the relevant song files to Oni, and unfortunately we couldn't get it to work.  
He tried contacting Alcatraz for help with compilation, but the Linux build is extremely new and untested (released only a few weeks prior), so he couldn't get it to work.  
In the end, we decided I'll use Sointu instead, to prevent any long delays and last-minute bugs.

# Composition

Oni suggested Breathe (In the Air) by Pink Floyd as a reference.  
After listening to it, I got the general vibe, but I also had to keep the limitations in mind.  
Guitars wouldn't sound good, so I just mostly avoided them and made the main harmony with an organ.  
I knew the song had to have a lapsteel guitar to get the feel across, but other than that I wanted no guitar sounds at all.

80's prog rock isn't my forte, so to save myself lots of research, I decided to ~~steal~~ be inspired by Pink Floyd's very own Wish You Were Here for the main bassline.  
Also I wanted the song to have a kind of jazzy break in the middle, and after playing around on my piano I found the one I went for.

# Imitating Acoustics

The sound design posed a significant challenge.

Sointu doesn't have sampling capabilities, so I couldn't use real-life instrument sounds at all.  
Sointu can use samples from gm.dls, but of course, this is a Linux production - that isn't available.  
Everything had to be synthesized.

To try and ease the problem, I opted to not use too many guitar sounds in the final piece, because synthetic guitars are quite easy to tell.

I'll break down how I made each instrument, starting with the one I knew would cause me problems from the start, and is actually the main reason I wanted to use samples to begin with:

### The Ride

While authentic cymbals can be achieved with synthesis (using [Au5](https://www.youtube.com/watch?v=2pPfKlXMA7M)'s or [Zion Jaymes](https://www.youtube.com/watch?v=netcpYINyBQ)'s methods, if you want some nice material), it's more complex than what the 4k synth allows.  
I decided to use a dumbed-down version of Au5's method, and just make a metallic noise by throwing a bunch of inharmonic waves.

I made five oscillators with different semitone tunings meant to clash as hard as I could get them.  
Then I activated their unison field, which adds another sine wave with the detune knob going the other way.  
This gives us an additional five oscillators, so ten waves all clashing with each other.  
Here are their relative tunings, in semitones: 36 ± 100 cents, 36 ± 51 cents, 40 ± 64 cents, 38 ± 68 cents, 45 ± 46 cents.  
By the way, all oscillators here are the default - type `sine` with `color` 64.

To get an additional metallic color, I added _another_ unison oscillator, but this one had FM on it, which I accomplished using the `send` module to the oscillator's phase (so technically phase modulation, but that's basically FM).  
The oscillators were at 47 semitones ± 62 cents, phase modulated by an oscillator module at 16 semitones ± 34 cents.

I distorted that result, then added a high-passed noise to round out the cymbal texture, and added an EQ afterwards.  
The new `belleq` module in Sointu is a real life-saver. (now we only need automations!)

The result actually sounds a lot like an 808 cymbal, which I think is nice because I never made anything like that before, but I would still prefer a nice sample to it.  
However, it's recognized as a ride, so it gets the job done.

### The Snare

The snare is actually one of Sointu's presets called `Snare EDM`.  
It sounds relatively acoustic on its own (as far as it could get), but it has quite a hard transient, so I had to modify it to sound softer.  
The modification includes adding an envelope with attack, and reducing the pitch drop element by lowering the envelope's gain.  
Overall not too big a deal.

### The Kick

The kick is also a Sointu preset - the one simply called `Kick`.  
All I added to it was a `distort` module at the very end, I guess so it would be louder in the mix.

### The Bass

The bass is extremely straightforward - it's just a sine wave being FM'd (or PM'd) by another sine wave of the same pitch.  
The sine wave provides a very clean signal, and the FM gives it a little bit of metallic body so it'll sound more like an actual acoustic bass.

### The Organ

As I mentioned earlier, the organ is based entirely on a preset from Phase Plant.  
It's a sine wave being AM'd by a sine wave two octaves up.  
In Sointu, this is easy to do by just using `mul`.  
I didn't use `mulp` because I then added the lower sine wave to the mix using `addp`.

This gives an organ-y sound already, but no organ is complete without the rotary effect.  
I'd have used a chorus, but Sointu doesn't really have one, so I did something similar - I set both waves to have unison, and then modulated their detune using an LFO.  
This imitates the way a chorus works - instead of lowering and raising the pitch with an LFO, I lowered and raised the unison detune.  
The results sound absolutely delightful - I'd recommend this technique if you want the rotary sound.

### The Lead

The first lead is a very basic filtered saw wave, with vibrato added using an LFO on the `detune` field.  
There's only one catch - in some places, I wanted the lead to have a grace note (two notes played in rapid succession), but Sointu's tracker only lets you save 16ths.  
I didn't want to add 32nds to the whole thing, so I added another instrument with the exact same configurations and a `delay` module at the end that offsets it by a fraction of a second, and then played the two instruments simultaneously when I wanted to use a grace note.  
I'm not sure if this is a better method than just expanding the tracker's resolution, but I'm not very well-versed with trackers so that's what I decided to go for.

The lead is one of the compromises that had to be made - normally this kind of lead would be a guitar or a vocal, but there was no way I could see of getting a nice-sounding guitar or vocal, so it's kinda out of place for the time period we were going for, but certainly not unheard of.

### The Lapsteel Guitar

I'm pretty pleased with this one.  
Synthesized guitar sounds entirely fake - it barely passes for the real thing, especially with only the basics in Sointu.  
My solution to this problem was to lean heavily into the lapsteel guitar aesthetic - very long bends, huge reverb and delays, big spacious sound overall.

The patch itself is very simple - it's a skewed triangle (`trisaw color=86`) modulating another skewed triangle with FM, and then passing through two filters with a decay envelope.  
The main thing I'm proud of, however, was the bends.

Since Sointu doesn't have portamento or glide, the way to get these nice bends was to have another instrument (which I called "bend"), which has a big attack envelope multiplied by the `loadnote` module, and then passed through `send` to the transpose knobs on the oscillators in the guitar.  
This means that when I play a note on the "bend" instrument, the guitars will _slowly_ transpose to another pitch value corresponding to that note.  
Somewhere around E5 was the middle, which wouldn't affect the guitar at all. Below E5 the guitar would bend downwards, and above it upwards.  
This allowed me to not only control the bend target, but also the length of the bend - if I wanted a bend to last shorter, I'd simply bend to a more extreme note, and then stop the bend before it gets to its destination.

The only caveat is the guitar can't stay at the end of a bent note - it has to be let go, and either another note will be played or the guitar will remain silent.  
This was a pretty minor inconvenience, and was easily avoided in the composition.

# Conclusion

This pretty much covers everything I did on this project.

This was a fun one to work on!  
I always love collaborating and making music for other people, because it takes me out of my own bubble and shows me new perspectives.  
Also making something for someone requires me to work on a deadline and holds me accountable, which I love, and seeing someone pleased with the result and sharing in creation makes me feel great.  
I hope to collaborate with Oni again and with many more people in the future.  
Thanks for reading!
