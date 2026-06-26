---
layout: single
author_profile: true
read_time: true
comments: true
share: true
related: true
title: "The Miracle of Auto Research"
date: 2026-06-16 10:44:00+0000
categories: ["research"]
permalink: "/2026/06/16/the-miracle-of-auto-research"
---

## TL;DR

Andrej Karpathy set out the idea of "Auto Research" in a [GitHub site](https://github.com/karpathy/autoresearch) and spawned a thousand forks (actually more than 12,000 to date).
He applied it to tuning a neural network, but noted that the approach was more general.

The idea is simple: give an AI agent a score to improve, permission to change the code, and a tight loop for testing whether each change helped.
I tried this on an Extended Kalman Filter for core temperature estimation; this post is about what happened.

## Opening: The miracle

A few months ago, if I wanted to improve a research model, I would run an experiment, look at the results, think of a change, edit the code, run it again, and repeat.

Auto-research changes that loop. You give an AI agent a clear goal, a way to measure success, and permission to edit the code. Then it tries an idea, runs the experiment, records the result, and decides what to try next.

That is the miracle. Not that the AI suddenly becomes a scientist, but that it can keep working through the repetitive middle of research: trying variants, rejecting failures, and occasionally finding something useful.

Karpathy described this as setting it going, going to sleep, and waking up eight hours later to find hundreds of experiments performed, the best results saved, and real performance gains produced.

That sounds outrageous. It also works better than I expected.

## How auto-research works

A quick summary of the auto-research approach is as follows:
  1. define a measurable target;
  2. constrain what the agent can change;
  3. run short experiments; 
  4. record results; 
  5. let the agent propose the next intervention; 
  6. keep or revert the change.

The real insight is in establishing a *contract*. This is a document that instructs the agent about the process. 
It asks the agent to change its way of working quite fundamentally---from interacting with the user to interacting with the research problem. 
The contract can be thought of as a clever and detailed prompt. 
Rather than typing it in each time, though, we can ask the agent to read it from a file. 

Markdown is a natural format for talking to agents, as it is close to raw text and converts to tokens efficiently, while also providing some structure and formatting (such as bullet points and headings).
Many modern editors support Markdown, including VS Code and Obsidian. 

I should mention here that we are working with an agent that *can* read a file on your hard disk. It does this by using a local Command Line Interface or CLI (such as, Claude Code, Codex, or Gemini CLI). Open source variants include OpenCode and Pi (my personal favourite). 
Note that the CLI does not necessarily constrain what LLM model is used at the back end. For example, it is possible to use a model that sits on your own laptop if you like (although for this job I found a stronger model made a real difference).

The best way to talk about how to make use of Karpathy's contribution is to explain how I used it, as this allows us to discuss how to change the approach to suit a new problem. 

## Our problem: core temperature prediction

Our team has been working for a while on the problem of using machine learning to provide a real-time core temperature estimator. 
Past work (Buller et al, 2011, 2013) devised an approach that used just heart rate with an Extended Kalman Filter (EKF) to provide a real-time core temperature estimate. 
The logic is that heart rate rises when your core temperature is high as your body is working hard to cool you down. 
Of course, there are many other things that could cause heart rate to rise.
However, core temperature is difficult, expensive, or uncomfortable to measure directly, and so even a potentially error-prone estimate is better than nothing. 
Nonetheless, it would be nice to improve the estimate, and perhaps we can do so with some form of machine learning and a little more information than heart rate alone. 
Some of the other variables include skin temperature, clothing, the user's age, their activity level, and so forth. 
There are lots of possible variables and it is hard to know which ones to select and how to incorporate them into the filter for best effect. 

## Scoring is the hard part

For auto-research, and possibly any sort of research, we need to start by saying what "good" looks like. 
Past work used a measure called Root Mean Squared Error (RMSE), with a lower RMSE being better. 
The logic behind RMSE is that squaring the error makes it positive; we have lots of individual errors over an episode, so we take the mean; taking the square root returns the value to the same units that we started with. 
When judging the quality of a Kalman Filter, though, the average error is not really the whole story. 
The first problem is that we could improve the score greatly by chopping up our episodes into smaller bits---just like predicting the weather one day ahead is easier than 10 days ahead.

The second problem is that a Kalman Filter gives an uncertainty estimate as well as a value---but how good is that uncertainty estimate? 
In the field of core temperature estimation, uncertainty is universally ignored---which is a pity because it is critical in deciding between someone who is definitely at a safe core temperature and someone who may be at risk of exceeding a safe level. 
So for our work, we use well-defined episode lengths and an aggregate score that includes factors for RMSE and uncertainty quality. 
The point is, choosing the metric or scoring method is one of the most important (and perhaps most difficult) parts of the auto-research process. 

## Provide evaluation code

The next step is to provide evaluation code. 
Your agent will be of great help here. 
In my case, I just gave it my scoring equation and asked it to revise the existing EKF evaluation code to only output that one value. 
I'd like to say that I yearn for the days of yore when I had to carefully hand-code, debug, write test cases, and so forth, but it wouldn't be true.
In any case, I suspect that, for many teams, converting existing code to give the summary score (and nothing else) will be as straightforward as it was for me.

## Formulate the `program.md`

Finally, we need to create a revised "program" or script to give the agent. 
Karpathy's [original contract](https://github.com/karpathy/autoresearch/blob/master/program.md) can form the basis, but you should revise it with your AI to suit your problem. 
Start by looking through and working out what *sorts* of things need to be changed.
One thing I wanted to do was to make use of remote servers, so I included instructions on how to use `ssh` to get to those servers. I will discuss this more later on, as it turned out to be critical. 

## Using git as a ratchet mechanism

A ratchet is a device you've probably used when trying to fix a sofa you bought second-hand to the roof of your car. 
It works by having a tape that feeds onto a spool with a ratchet mechanism that only allows the tape to tighten and not loosen, allowing you to get that binding tape to hold the sofa really tightly. 
`program.md` works the same way, but using git. Each new experiment is committed, but when the results come back, a poorer result is reverted while a better result is kept. 
There is also a penalty for complexity. If the improvement is slight but the original is simpler, then the new version is still reverted. 

## Does it work?

So with all this put together, it might sound like it would work immediately. 
I found, however, that Codex was reluctant to follow the instructions without a lot of hand-holding. 
Fortunately, I discovered the `yolo` option (you only live once), which makes Codex much more relaxed. Other agents, such as Pi, are `yolo` by default. 

The next problem I discovered is that it spent all its time making small changes to numerical parameters, sometimes useful, sometimes not. All this for minor gain.
In addition, whenever it tried to change the *structure* of the filter significantly, the parameter set that worked well with the old structure tended to be suboptimal for the new one and it would instantly discard any structure change as a possible path. 

To head that off, I instructed it to do a numerical optimisation each time. If the parameters are already optimised, the agent can't pretend it's doing useful work by tweaking a parameter or two. More importantly, structural changes could be made and tested more effectively. 
This issue with auto-research not being able to get out of local minima is one that Karpathy noted in his original blog post. While adding in the numerical optimiser helps, it doesn't completely solve the problem. Don't expect it to find solutions that require a series of independently counterproductive features but that all work together. 

A further problem that I found is that, when left to its own devices, auto-research can be a bit undirected or can go back and forth over similar unproductive variants. 
To help with this, I asked it to produce a list of ideas, write them to a file, and then explore them one by one. 

## And then miracles start to appear

My starting score at this stage was around 0.86. I set the code to run and left it while I took my son to his tennis lesson. Two hours later and the score was 0.46---almost cutting the metric in half. 
Some of that performance improvement turned out to be due to the limited training set that I was using when I started. When I expanded the training set, the overall model improved further, as expected: more training data gave the optimiser a better chance of finding a robust model. The true improvement due specifically to new ideas and filter structure was still impressive, but more like a 14% reduction when tested on an unseen test set.

## Token budget issues

At this stage, the limiting factor became my ChatGPT Pro account, which, while having generous limits on tokens per period, was not designed for this much attention.
Looking through the log, I quickly realised how wasteful the interaction with the remote server was. There were lots of messages going back and forth about waiting for jobs to complete.
It was simple to fix, though. I just created (with Codex) a script to package up most of the interaction, including checking that the current repo was clean, pushing to GitHub, pulling on the remote server, and then executing the job. That approach ensured that any experiment result could be tied to a specific code version. The waiting could also be scripted.

If you are interested in the detail of this "run job remotely" script, here are the key ideas:

1. Check that the git repo is clean (no outstanding changes, no untracked files) and stop immediately if not.
2. Ensure the git origin is up to date and then pull on the remote server.
3. Our remote servers use SLURM to manage job submission over clusters of computers, so rather than directly run the job, the script sends a command to SLURM to run it later.
4. Then we wait. Since auto-research is set up as a sequential process, we need to wait until we get a result before continuing. The point here is to automate the waiting so it's not up to the agent to worry about.
5. Finally, we extract the result from the logs. The trick here is to keep track of the job number and pull out the experiment output from the corresponding log file.
6. By default, the agent will clutter the place with extra logs. Tell it to resist the temptation. The only output should be the score.

## You can't outsource your understanding

Karpathy recently mentioned a post on X that said "You can outsource your thinking but you can't outsource your understanding."
Two hours of useful auto-research turned into many days of trying to *understand* what the changes were and why they worked.
Fortunately, Codex was able to put together a short research paper, and that really helped.

The agent does not become a scientist. Give it a scoring function, a contract, git, and a plain experimental loop, and it can handle the repetitive middle of research: try variants, record results, and bring back surprises.

Of course, you still have to do the understanding.

It is an exciting time, but not just because tools like this now exist. The exciting part is that we are still learning how to use them well. Auto-research is one pattern; there will be others. We are collectively discovering how AI can amplify serious work, and the best methods are almost certainly still ahead of us.
