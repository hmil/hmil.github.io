---
title: How not to suck at GitHub
date: 2018-05-28 22:00:00
id: 263
categories:
  - software engineering
tags:
  - satire
  - GitHub
---

So you learned a bit of programming and you know how to use git. You are ready to materialize your thoughts through the mighty power of code, and make your art available to the whole world using the most popular open source social network on earth.

But you fear that you will make stupid mistakes and ruin your chance to become the next tech influencer. Rest assured, after reading this post you will know everything you need to know to make the most of Octocat's home.

I will give you 10 tips to climb to the top 1% of GitHubbers faster than [Pemba Dorje](https://en.wikipedia.org/wiki/Pemba_Dorje) on oxygen!


## 1. Don't look for alternatives

If you think of a novel utility, library or a plug-in, you need to start coding your own solution straight away.

You will only lose time looking for alternatives. You will find that prior art does not quite fit your requirements and, of course, you can not bend, even slightly, your problem definition to be solvable by existing solutions. Moreover, during this process, you may taint your original idea by looking at too many similar concepts, which will invariably worsen your design.

It would also be a huge waste of time trying to add a feature to an existing project. The legacy codebase of established projects is massive and messy. Just learning the relevant bits would take as much time as coding the whole solution from scratch anyway. And I'm not even talking about the near-infinite number of hours of work lost in feature requests which end up being rejected by the project owner after 3 months of silence.

## 2. Google and StackOverflow won't help you

Don't waste your precious time googling for a solution. Most of the answers on StackOverflow are garbage anyway, and the library you are working with (and which, of course, is the root cause of your current issue) is not that popular, hence chances of finding the exact solution to your problem online are minimal.

Instead, you better bother that project maintainer, whose lack of QA considerations is causing your troubles. The project maintainer has the mental map of his project engraved in his memory and therefore can figure out the solution to any problem near instantly. Plus, I suspect projet maintainers keep a secret list of known issues which they do not fix out of pure laziness and they actually wait until someone notices it to fix it.

## 3. Don't try to fix it yourself

This tip is similar to number 2. If a project you are using doesn't work, it's probably the maintainer's fault. Don't even bother looking for an answer in the project wiki or investigating the bug yourself. Just go ahead and nag the project maintainer, he or she will probably be able to fix their buggy project just for you.  
Don't forget to check back each day and notify the maintainer if he forgot to respond to your last message.

## 4. Be as vague as possible

When you open an issue for a bug you've encountered (which should be about 2 minutes after you've noticed said bug if you follow the advice above), please do yourself a favor and describe your issue in as vague terms as possible. This way you avoid making a bad diagnosis and mislead the maintainer. Also, don't post a snippet of your own code, system stats or installed libraries as it may leak sensitive information out to the public (remember hackers are constantly scraping GitHub for sensitive information!).

## 5. Don't bother with the markup

You've heard about [markdown](https://en.wikipedia.org/wiki/Markdown#Example) but for your own sake, you couldn't remember how to format code. Is it a &lt;code> tag? Or maybe [code]? Or is it a back tick? Damn it!

Don't worry, no one does. Just forget about formatting your issue altogether. Paste large blobs of code inline with your prose like you're writing lyrics for the next trap banger. After all, if the maintainer is not happy with your format, he can copy and paste it to a text editor.  
As a bonus, this way you don't even need to indent your code properly!

## 6. Refrain from using reactions

It is outrageous that the GitHub team spent so much effort mimicking facebook with its stupid "reactions". Ugh... Anyway, when you feel like you agree (or even disagree) with a comment, be respectful and post an additional comment to voice your opinion while adding absolutely no useful piece of information to the debate. And please, please, please, leave that stupid "thumbs up" button for social-network addicted kids.

## 7. DO NOT, ever, submit a pull request

Everyone knows project maintainers hate it when someone tries to touch their code. After all, why are they constantly finding excuses to reject pull requests?  
Also, in order to preserve your ever so precious time, and avoid a painful public shaming, do not attempt to submit code to an open source project.

## 8. If you ever should submit code, don't ask!

Let's say there is this extremely useful feature you need, and the project seems a bit inactive lately. You **really** crave that feature and you can't wait for Mr. slowpoke to code it himself. As per point 1, you do not have time to write the software from scratch yourself. You realize that it will be a substantial change which will require quite a bit of code, but nothing can discourage you. You **need** that feature.

Then make sure to secretly develop the feature without letting the maintainer know; otherwise, he might try to cut the grass under your feet. Don't let anyone know you are working on this awesome addition until the very last moment when you submit your huge pull request as a single, solid, code change. *Wow* factor guaranteed!

## 9. Do not answer other people's questions

As you contribute to repositories, you will notice other people try to drown your hard work under a deluge of irrelevant issues and comments. Don't think helping them will make them go away; they will invariably come back with more.

Instead, keep a cool head and refrain from helping. If needed, "up" your own issue by asking for updates.

## 10. Use issues as a general purpose Q&A place

As we've covered in point number 2, you will not find answers on StackOverflow. What's more, its elevator system tends to reorder comments in an annoying way which breaks chronological order.  

Therefore, if you have anything to say, which is even remotely connected to a project, then go ahead and express yourself using that project's issues. This will notify **everyone** who watches the repository and guarantee maximum exposure of your comment.

---

Hope you liked these tips. Please feel free to contribute your own in the comments!
