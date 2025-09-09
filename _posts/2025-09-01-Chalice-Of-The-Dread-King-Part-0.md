---
title: Chalice of the Dread King (Part 0)
date: 2025-09-01 12:16
categories: [Experiences,Game Design]
tags: [.net, asp.net core, blazor, front end, experiential, game design, narrative, puzzles, webassembly, mystery, visual design, storytelling]
image: https://strgdsysburtcher.blob.core.windows.net/burtchernet/images/cotdk-mainmenu.webp
---

> ### About This Series
> In the run up to Christmas 2023, I made a narrative-driven puzzle game, inspired by DOS text adventures, in Blazor WebAssembly. To progress through the experience, players were required to consult a physical photocopied document purporting to be the manual for The Chalice of the Dread King III by Black Lizard Software, which arrived at their home unannounced via post.

> This series of blog posts explores how I went about planning and implementing both the digital and physical sides of the game in just under a month. It's not a tutorial (and almost certainly doesn't represent how you're supposed to use Blazor) but I've tried to demonstrate the stuff I learned with code snippets and explanations where I can.

> **Player, look out:** there are spoilers throughout. I've now made a version of the game free for readers of this blog, and if you have curiosity about the project I suggest you play it first. It might take you about 40 minutes to a couple of hours - or several weeks. Give it a go before reading on.
{: .prompt-warning }

## Part 0: Once Upon A Time
In November 2023, I received a puzzle book advent calendar from my wife.

The book featured a silly adventure story with daily puzzles, the solutions for which you had to verify via a website form. The puzzles were more obscure, escape-room-at-home type exercises rather than crosswords and so on. The tactile experience of solving them on paper before heading over to enter answers online was just good fun. And, it reminded me of something I hadn’t thought of for a couple of decades at least: old DOS games with copy protection codes in their manuals.

Perhaps you remember them too. In the floppy disk era, we definitely had some games (maybe Lemmings was one, and perhaps a flight simulator) which required you to ‘prove ownership’ of the game by getting you to input symbols or commands from the print manual before letting you into the game proper.

This sparked an idea: what if the website verification was more than it seemed? Not just a functional step at the end of a puzzle, but an intrinsic part of the story What if you had to combine the book and the online interface to figure out what was going on? Some of my favourite games have used print manuals to great and varied effect (TIS-1000, Keep Talking and Nobody Explodes) and I couldn't get the idea out of my head.

![Desktop View](https://strgdsysburtcher.blob.core.windows.net/burtchernet/images/cotdk-manual-p4.webp) 
_The finished story spread at the opening of what would become the Manual for COTDK_

I work with code and web stuff. .NET 8, the latest version of my programming platform of choice, had just been released, and I had been looking for a way to get my hands dirty with front-end framework Blazor WebAssembly since it took off in .NET 6. I hadn’t (yet) had any client projects which seemed like a good fit - but this kind of input interactivity seemed like it might just be perfect.

So I set myself a challenge: I’ll learn Blazor in the evenings by making a small prototype of this idea, basically just some jazzy forms and a few riddles with a bit of interactivity sprinkled in, all styled up to look a bit like an old DOS game. My wife suggested I’d be more likely to follow through if I set a deadline to ship it off to a few friends in time to be a little puzzly Christmas gift for them. A simple, neatly contained project to fill a few of those December evenings.

Lol.

If you’ve ever come up with a ‘simple’ project to do in your free time, you know what happened next. The scope spiralled into the stratosphere. Before long, the ‘what if the manual was part of the game’ prototype had evolved into a mystery story centred around a supposedly emulated version of a lost 1990s text adventure, with increasingly strange challenges that blend the game world with real-world interaction. I added a story (for the game and the game within the game), designed and developed a printed manual, conjured a fractious, fictitious game developer with their own backstory and agenda, and increasingly stretched myself to include more dynamic Blazor features to fit the new ideas for challenges that were popping up on a daily basis.

By the end of the journey, I’m sat on the floor of my work shed two days before Christmas, surrounded by multiple iterations of faux-photocopied blurry manuals personalised to their intended recipient, covered in sticky notes, alternately pushing new feature builds and bug fixes into the deployment pipeline while my wife stuffs envelopes to try and hit the last post.

It was exhausting, it was fun, and it worked. I learned a lot about Blazor, and got to revisit some ideas from previous real life escape room and interactive experiences I’ve made in a whole new context.

It’s over a year on and I’ve come to love Blazor development since - so I thought it’d be fun to revisit the project and share a bit about how it works and how I made it for anyone who’s curious.

```csharp
if(_gameState.CurrentMode?.Id == GameModes.EthicsTest && _gameState.CurrentStage?.Id == 3)
{
    if (!_gameState.EthicsLoopBroken)
    {
        _gameState.EthicsLoopCount++;
        if(_gameState.EthicsLoopCount > 2)
        {
            _gameState.UserMayExit = true;
            ErrorConfig error = new("736f756c#LOOP#ConsultManual", "yellow bg-dark bg-opacity-75 ", "type 'quit' to end game");
            errorMessage.Show(error);
        }
        // Return right back to Stage 3.
        stageId = 3;
    }
    else
    {
        // otherwise, move directly to Soul Selling
        modeId = GameModes.SoulSelling;
        stageId = 1;
    }

}
```

This series is an attempt to give an idea of everything that went into it. If you're interested in Blazor development, game design or experiential narratives - or if you're a sucker for punishment who thinks trying to do all three on your own is a good idea, like me - there might be something in here for you. I find it really difficult to completely separate the experience design stuff from the technical implementation stuff but I’ll do my best to make it possible to skip over the bits that aren’t that interesting for you.

In Part 1, I’ll talk more about the game concept in more detail (where it started and where it went), walk through the core mechanics, and give a technical overview of the whole thing as well as some insight into how it evolved throughout.

Thanks for reading! Follow me on [BlueSky](https://bsky.app/profile/batmurtcher.bsky.social) or [Mastodon](https://mastodon.social/@batmurtcher) if you want to hear when the next part comes out.

PS - I can’t emphasize enough that this series will be riddled with spoilers from the off (in fact, if you’ve read this far, some of the surprises and mysteries are already going to be unsurprising and unmysterious) and, if you want to read on, I strongly suggest having a go at the free version of the game I’ve released before Part 1 comes out. Good luck.
