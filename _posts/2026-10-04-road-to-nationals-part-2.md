---
layout: post
title: "A Paradigm Shift in Cyber Competitions"
date: 2026-10-04 00:00:00 -0400
author: Alex Schneider
categories: [road-to-nationals]
tags: [ccdc, blue-team, ai, opencode, competition]
series: Road to Nationals
series_part: 2
series_url: /road-to-nationals.html
permalink: /road-to-nationals-part-2.html
description: What our first WRCCDC Invitational taught us about AI, and how we want to approach the 2027 CCDC season.
---

Yesterday, we competed in our first WRCCDC Invitational of the year. As always, there was a lot to learn. We went into it as we usually do: unprepared, with no real coordination amongst ourselves, and a free-for-all in the first ten minutes as everyone tried to get ahold of their systems and understand them. We did, however, take away something major for how we intend to approach the 2027 season.

Most of us started by manually enumerating and threat hunting on our machines. We found a backdoor here and there, along with some domain users with DCSync rights (Recycle Bin, haha). We used OpenCode more and more as the invitational went on. The room got quieter too, since we were spending less time typing commands ourselves and more time reading what the agents were doing.

It was quite frightening. By the end of the competition, we all kind of sat back and discussed what we had done. We came to the conclusion that two possible paths lay in front of us:

1. Refuse to use AI in subsequent invitationals and competitions, and attempt to learn everything by hand.
2. Commit to the dark side and accept that the future of cyber competitions, and perhaps cybersecurity as a whole, hinges on practical applications of AI.

Such a realization is quite a grim one, but indeed it seems that's where things are going. In [*The CTF scene is dead*](https://kabir.au/blog/the-ctf-scene-is-dead), Kabir Acharya, a CTF competitor who has played with TheHackersCrew, writes:

> Teams that refused to use AI were not just missing a convenience; they were playing a slower version of the competition.

This trend isn't limited to CTFs. I believe refusing AI in RvB competitions where it's allowed puts you at a competitive disadvantage, and orchestration itself has become a competitive skill.

Where we go from here requires some reflection. Our team leads and I are becoming AI-pilled, and we're thinking about what we could create to compete in this new RvB landscape.

## The Dark Solution

Our plan is to build a lightweight harness around OpenCode, orchestrating a swarm of instances running on the student competitors' machines to harden a CCDC environment. Since we can't use paid AI services, we'd use free models available through OpenCode. The orchestrator would itself be an OpenCode instance, with access to the scoring website, service dashboards, and inject portal.

The main features we brainstormed are:

1. Injects. The orchestrator would complete injects using a template we'd created, gathering relevant findings from the agents. We'd review and submit the responses ourselves to ensure they're valid.
2. Service-specific skills and playbooks. Each OpenCode instance would have skills geared toward its assigned systems: Windows, AD, Linux, routers, etc. It would also have playbooks for tasks like rotating credentials at the beginning of competition. We'd run the instances in VMs, taking advice from [Krauq's AI Power User Guide](https://blog.krauq.com/post/krauqs-ai-power-user-guide), so their tools and working files stay off our main machines.
3. Scoreboard monitoring. The orchestrator would scrape the scoreboard to see which services are down and direct the relevant agents to investigate and restore them.
4. A shared discussion board. Agents would share findings and coordinate changes, since some services rely on others. A Linux service using AD authentication, for instance, would need its agent to coordinate with the AD agent before changing the account it uses.
5. Credential management. A shared credential manager would be updated whenever an agent rotated credentials, keeping the team and other agents working with the current passwords.

[![Mermaid diagram showing the orchestrator, OpenCode workers, service skills, shared discussion board, credential manager, and inject review.]({{ '/assets/images/road-to-nationals/agent-harness.svg' | relative_url }})]({{ '/assets/images/road-to-nationals/agent-harness.svg' | relative_url }})

*High-level architecture, admittedly drawn by AI*

## References

- [OtterSec: Announcing the Save CTFs Fund](https://osec.io/blog/save-ctfs-fund/)
- [Kabir Acharya: The CTF scene is dead](https://kabir.au/blog/the-ctf-scene-is-dead)
- [Krauq's AI Power User Guide](https://blog.krauq.com/post/krauqs-ai-power-user-guide)
