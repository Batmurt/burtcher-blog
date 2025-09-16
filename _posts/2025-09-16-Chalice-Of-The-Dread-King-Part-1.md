---
title: The Chalice of the Dread King (Part 1)
date: 2025-09-16 14:14
categories: [The Chalice of the Dread King,Game Design]
tags: [.net, asp.net core, blazor, front end, experiential, game design, narrative, puzzles, webassembly, mystery, visual design, storytelling, dread king]
image: https://strgdsysburtcher.blob.core.windows.net/burtchernet/images/cotdk-mainmenu.webp
---


> **About This Series:** In the run up to Christmas 2023, I made a mixed media narrative-driven puzzle game inspired by DOS text adventures, using Blazor WebAssembly and my printer. To progress through the experience, players were required to consult a physical photocopied document purporting to be the manual for *The Chalice of the Dread King III* by Black Lizard Software, which arrived at their home unannounced via post. This series explores how I went about planning and implementing both the digital and physical sides of the game in just under a month. 
{: .prompt-emphasis }
> **Player, look out:** there are spoilers throughout. I've now made a version of the game free for readers of this blog, and if you have curiosity about the project I suggest you play it first. It might take you about 40 minutes to a couple of hours - or several weeks. Give it a go before reading on.
{: .prompt-warning }

## Part 1: What Was I Thinking
Hello, and welcome to Part One of this series about making a (kind of) text adventure game in Blazor WebAssembly. If you’ve already read Part Zero, you’ll have an idea of where we’re going. In this post, I want to flesh out the concept and inspirations behind the game a little bit more, so you can see what I was getting myself into and follow along with the challenges as they emerge. As such it’s heavily weighted towards big wooly ideas rather than any code implementation - more of that coming in Part 2.

I occasionally have thoughts bordering on design philosophy and methodology which I will endeavour to put in little box-outs so you can skip them if that kind of thing fills you with rage/vomit. I get it. (Look out, there is one coming up immediately.)

<div class="box thought" markdown="1">
### Methodology #1: How My Brain Works (And Yours Too, Probably)


**Disclaimer:** So we’re clear, I’m having to cover topics like “concept” and “gameplay” in a much, much more linear and cogent way than the ideas actually presented themselves. By writing in this format I am necessarily going to give the impression of a much clearer, well-defined vision of every aspect of the game than there ever was in reality while I was making it. The overlapping of gameplay idea with aesthetic ideas with delivery ideas, combined with the endless iterative loop of technical discovery (“Oh, I can make it do this?”) and experience generation (“...then I have to have a scene where the player uses this and that together!) means all those things evolved in parallel. I cannot truthfully claim that I sat down and conjured it all ex nihilio then set about making it. Do people do that? Maybe big teams have to in order to get anything done, but I find it hard to imagine.

It’s a fun way to work, and in my experience represents the best chance at getting creative ‘flow’ going. It is a rehearsal room for one. Allowing yourself to slalom back and forth between ideation and implementation on an ad hoc basis when the mood takes you means something is always getting done, even if it’s not what you had on your list (or, potentially, anything ultimately useful). It is not deadline friendly, as you’ll see, because it’s so open-ended: but you’re making the path yourself, rather than following a map, which is intrinsically more fun.

Anyway: there is just no way I can make a faithful stab at recreating the actual branching threads inside my brain (not least because I didn’t write anything down at that level of detail) so please just take everything with a pinch of salt when it comes to statements like “I thought it should…” and “I came up with the idea to…”. 
</div>

