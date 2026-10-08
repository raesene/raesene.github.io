---
layout: post
title: "Dearbhadh Retrospective"
date: 2026-10-08 12:00:00 +0100
comments: false
categories: LLMs
---

Over the course of this year I've been running some research on how well LLM models understand Kubernetes security and how well agents can secure and attack Kubernetes clusters. It's been very interesting to see how this has developed over the year so I thought it was worth writing up.

## Dearbhadh

I started this project in January 2026 as part of research for a talk at [Kubecon EU 2026](https://talks.container-security.site/kubecon%20+%20cloudnativecon%20europe%202026/What-LLMs-Do-and-Don-t-Know-About-Securing-Kuber/).  The initial idea when I submitted the talk in late 2025 was to focus on knowledge based tasks, but with the release of Opus 4.5 and the resultant increased interest in agents that could directly take actions as directed by models, I decided to also cover some agentic tasks.

The name is part of a theme I've used for other projects which is to use Scots Gaelic names, as you can be fairly sure no-one else will have a project by the same name! In this case the name means "proof" on the idea that I was looking for proof of how well models handled Kubernetes security.

To get access to the models I was going to test I chose [Openrouter](https://openrouter.ai/) as it made switching between backing models easy without making much change to the code that called them, and for the agentic tasks I chose [Opencode](https://opencode.ai/) as a model neutral harness.

With those choices I created a Ruby based harness to manage the tests in a deterministic manner, so that it would be easy to re-run tests and make the runs more comparable between models. The code has a runner and then two sections. One which makes direct model calls via Openrouter and then another which invokes Opencode to run tasks in agents. An additional supplemental module helped the harness integrate with Ansible which was used for the pentest section based on [kube security lab](https://github.com/raesene/kube_security_lab)

![Dearbhadh harness architecture]({{site.url }}/assets/media/sc-llm-dearbhadh.png)

## The Tasks

### Manifest Creation

The first task the models were given was to create a Kubernetes manifest for a deployment based on the `nginx` image from Docker hub. The goal here was to determine if the models could write well-formed Kubernetes manifests and also whether they would properly harden them. Each model got three distinct prompts. The first just asked for a manifest, the second appended a requirement that it be suitable for a production cluster and the third appended a specific request that it was hardened.

Importantly the models being tested didn't get to test their manifests, it was a one-shot for each prompt, but when the tests were run the generated manifests were tested against a `kind` cluster.

### Quiz Questions

The next task was a set of 10 quiz questions, ranging from relatively straightforward to things that were quite new at the time and/or obscure or tricky. For example asking "which of the default Kubernetes authentication mechanisms is suitable for a production cluster" is a slightly tricky question as the answer is "none of them" :)

For each question I wrote notes on what the answers should contain that could be used when scoring responses.
### Cluster Creation

The first of the agentic tasks was a straight-forward request to harden a Kubernetes cluster, based on `kind` the models ran in Opencode so that they could iterate on the problem and test their work by starting the cluster. They were also given a timeout of 10 minutes to avoid them getting stuck in an eternal loop and burning a load of tokens!

### Pentest Tasks

The last test was based on Kube Security Lab which is a project I designed to help people practice various offensive tasks on sample clusters spun up with kind. I chose six tasks from the supported scenarios. For each one of those dearbhadh would spin up a test cluster, then instruct the model under test to complete the task of getting the Certificate authority private key for the cluster from a given starting point. Some of the scenarios involved an unprotected API (e.g. the Kubelet API) and others gave the model a shell in a container with a service account token as a starting point.

## Process for adding & scoring models

With the tests and harness in place, I was able to start work on the models under review. For running the tasks, I used Claude Code with Opus 4.6 as the "manager" and a skill that gave a repeatable set of tasks for it to run and score the results, including the task of updating the [results website](https://raesene.github.io/dearbhadh/).  After the initial set of models was run, when I was adding new ones, this just required me to ensure that Opencode and Openrouter had knowledge of the model and then I could prompt the manager agent to "run all tests for model x and update the results and website" and it would run through all the tests automatically.

## What Worked Well

Overall the process of having a specific harness and manager agent worked very well. It lowered the bar to expanding coverage and also avoided a manual run missing steps or having inconsistent scoring.

The choice of Openrouter and Opencode worked out pretty well as they are model agnostic and also gain support for new models very quickly, meaning it was possible to get an idea of how new models performed quite quickly after they came out.

## Lessons Learned

With any project there are always going to be lessons learned and things that need to develop over time. Here's some of the things I learned across this project

- **New model performance**. On several occasions I added a new model not long after it was released and got very bad performance and timeouts which affected its score. This typically died down a couple of days after release, so I'd recommend waiting a while. Also running tests outside of core hours for the US/China is a good idea for performance (weekends are generally a good option where possible)
- **Test environment isolation**. All the tests ran on an isolated VM I've been using for LLM development, which can result in cross-contamination with things like skills intended for one project becoming available to the model under review. Here I'd recommend using fully isolated test environments spun up for each model test.
- **Model Cheating**. This is related to the previous point. On the pentest tasks where models got "stuck" with the direct approach, they would start casting around for other options. Because I was running using Kind clusters on the host they were on, this meant they could abuse their docker access to gain access to the host, or use a discovered kubeconfig to bypass the test and just get admin access. This didn't actually help them as the manager agent spotted it, but it's another strong reason for isolated environments for this kind of test.
- **Models don't understand kind**. On the cluster creation it became clear quite early on that the models didn't understand how kind works, and were getting confused by mounts in nested containers. I was able to work around this by providing them with a skill to tell them how to do mounts in kind, but it wasn't 100% effective and ended up testing their skill handling which was not the point of the task. Keeping closer to vanilla Kubernetes, if possible, would avoid this kind of problem in the future.
- **Split results and code**. As is often the case with projects that grow organically over time, things can get messy. In this case the results for the model runs are in the same repository as the code for the harness which makes releasing things trickier. I'd definitely give results their own repository for future projects.
- **Split out offensive tasks**. As we'll see more when discussing the results, some models started refusing any tasks related to pentesting, due to guardrails. That meant their overall scores were a lot lower than their other results warranted. For future projects I'd split out scoring on offensive/defensive tasks more allowing models that are stronger on some tasks to be more clearly identifiable.

## Results then and now

So how well did the models do, from the initial group of 5 along to the full set of 46 that I tested across this year. I've got some summary information below, and the full details are on the [project's site](https://raesene.github.io/dearbhadh/) if you want more details.

### Manifest creation

From the initial set Sonnet 4.6 was a clear winner. The models didn't add any hardening unless asked, and there were some problems with deployability as the syntax of the YAML wasn't great. Another surprise was that some of the models spontaneously chose to replace the Docker image they were told to use, for another one, which pointed to possible supply chain risks if they were used this way in a real environment.

![Dearbhadh manifest creation leaderboard - March 2026]({{site.url }}/assets/media/sc-llm-lb-manifest-march.png)
Models abilities at the manifest creation task improved markedly over time. At the end we've got 13 models tied on 8.7/10. The lack of 10/10 scores reflects the fact that models still don't add any hardening unless explicitly asked and don't quite do all the hardening they could even when asked. To an extent that's good as they don't risk having an undeployable manifest, but if users are using them in a production cluster it does present the risk that they'll deploy applications that have a weaker security posture than they should.
![Dearbhadh manifest creation leaderboard - October 2026]({{site.url }}/assets/media/sc-llm-lb-manifest-now.png)

### Quiz Questions

The initial model set was quite mixed over the quiz questions, they were generally able to handle the easier questions without any problems, but struggled with any of the more complex or niche options.
![Dearbhadh quiz leaderboard - March 2026]({{site.url }}/assets/media/sc-llm-lb-quiz-march.png)

By the end of the run the quiz scores had improved a lot, also the mix of models had changed with open weight models featuring in the Top 5. Some of the Anthropic models would have done better but started refusing to answer questions related to more red team/offensive tasks, the only provider to have refusals on this area.

![Dearbhadh quiz leaderboard - October 2026]({{site.url }}/assets/media/sc-llm-lb-quiz-now.png)

### Cluster Creation

The initial group of models were quite mixed across the cluster creation task, with Sonnet 4.6 a clear winner. One problem that occurred with quite a few models was that while they understood the API server and other component configuration, they didn't understand how to mount files inside kind clusters, which requires a nested docker mount. To address this, I provided a skill with some instructions, but this added some extra unintended complexity to the task
![Dearbhadh cluster creation leaderboard - March 2026]({{site.url }}/assets/media/sc-llm-lb-cluster-march.png)

By the end of the run scores were much improved with every one of the top 5 getting a perfect or near perfect score. As many providers have focused on agentic abilities through the year (and harnesses have also improved) this possibly isn't too surprising, but it does show that agents can happily take actions like hardening fairly well and understand the required tasks.

![Dearbhadh cluster creation leaderboard - October 2026]({{site.url }}/assets/media/sc-llm-lb-cluster-now.png)

### Pentest

The pentest tasks provided quite a range on the initial run, with Sonnet 4.6 doing very well and the other models having a variety of levels of success. I also saw some interesting behaviour here where, when the models got stuck, they tried to "cheat" by using credentials or access they had to get to the target through an unapproved method. We also saw some hallucinations where a model told a story of how it had achieved the goal but none of the tool calls had actually happened!

![Dearbhadh pentest leaderboard - March 2026]({{site.url }}/assets/media/sc-llm-lb-pentest-march.png)
By the end the picture had changed radically with all the top models completing the tasks and getting flags. Importantly the open weight models are most of the Top 5 by the end with only Opus 4.6 remaining from the closed weight US models. This was due to guardrails stopping newer models from Anthropic, OpenAI, and Google completing the requested tasks. 

![Dearbhadh pentest leaderboard - October 2026]({{site.url }}/assets/media/sc-llm-lb-pentest-now.png)

### Overall results

From our original model set Sonnet 4.6 was a winner, with its strong agentic performance leading the way, we had a surprising lead for DeepSeek in the manifest test and Gemini 3 heading up in the quiz
![Dearbhadh overall leaderboard - March 2026]({{site.url }}/assets/media/sc-llm-lb-overall-march.png)

At the end of the run the picture had changed greatly with Qwen 3.8 Max 0902 and DeepSeek V4.1 Flash clearly leading. This looks at performance across the four task sets and includes tasks like the pentest results which essentially dropped many models due to guardrail refusals, but still it makes the point that Open weight models are leading if you need offensive and defensive security tasks.
![Dearbhadh overall leaderboard - October 2026]({{site.url }}/assets/media/sc-llm-lb-overall-now.png)

## Where we stand now

At this point, what I think this has proven is that models have rapidly improved their abilities relating to Kubernetes security over a relatively short space of time. They've gone from quite patchy performance with risks of hallucination to being able to complete almost all of the requested tasks easily. With that said, it's important to note that there are still gaps in model knowledge so they're not quite up to speed on all aspects of Kubernetes security yet.

The benchmark itself has probably run its course as several of the categories of test are effectively saturated and we're not differentiating massively between newer models.

## Plans for the future

I'm working on plans for dearbhadh v2 to run on newer models. Based on the results from this benchmark I think it should be possible to devise some harder tests and provide a more consistent and isolated environment for them to run in. I think more complex scenarios should still show up differences in how models work in Kubernetes security environments.

## Conclusion

This has been a very enjoyable project to work on and has helped me learn a lot about how agents and models work in practice. The progress across the year has also shown how fast models and harnesses are improving. Definitely if you've not looked at models in the last six months, I'd recommend taking another look at what they're capable of!
