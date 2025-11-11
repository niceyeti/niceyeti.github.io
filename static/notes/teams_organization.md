Date: 12/23/24
Subject: important team organization realizations

The context for this is a severe number of team failures at ****, and some of
the recent professional readings that I have engaged on the side. The most
important of these was Team Topologies, which is not entirely accurate nor
aligns with organizations of all sizes (predominantly it addressed large orgs).
However it provides useful depictions of team roles, characteristics, and
methods of evaluation; predominantly, I like the way it breaks down the
principle components of teams into stream, enabling, tiger, and so on. Many of
these categories flatter the egos of their authors, but I still find them
useful.

Getting to it, some of the finer points I've learned are:

* projects are depth-first: this is absolutely critical, an extremely valuable observation that struck me while running. For example, we recently experienced a
project failure resulting from too much design activity and meetings, with two "experienced" big tech
hires cycling through and failing to deliver. One left a kafka-application with missing distributed
requirements (it cannot scale, nor serves async). By contrast, assert the following:
  * "depth-first" means establish a first line of working (and failing, as needed) deployment
  * then, work ancillary branches to build out required features in priority order, and build chain

This approach works because it prevents needless upstream iterations on design, when per golang "interfaces
should be discovered rather than designed". Software is an empirical process, and it project structure
should follow the same layout; following a depth-first topology, and empirical test-driven
development, the risk distribution is better estimated, interfaces more cleanly designed and identified,
and so on. The same principles of cohesion and modularity in code apply to the architectural components
that emerge by way of a depth-first approach; the alternative are top-down design regimes that
have failed in every single example during my career, whether I participated in them or winced from the
sidelines at an adjacent team. Top-down regimes are not merely "waterfall", they are the middle-management
cringe pattern of every "enterprise agile" organization that is merely implementing the top-down strategy,
which again is proven to fail. But as it fails, its implementors justify their own costs by looking at the relative
costs of savings for mistakes which **never would have occurred** under the alternative depth-first approach.

  Depth-first is important because:

* software is not code: 1) it is the sensors by which its progress and failure modes are detected 2) it is the productivity of the environment in which it was built. The egocentric personality ethic of the software industry is a stupid and self-inflicted lie that propagated
  from arrogant university byproducts, a suburban elitism that permeates the industry and prevents progress beyond its monopolistic homophily.
  As a child, I worked in automotive shops in which legendary mechanics were known to self-teach themselves
  by reading manuals; every tradesman knows the value of a manual, but not engineers, most of whom never
  even cracked their textbooks, and bear out that lazy attitude in th industry itself. If you observe carefully,
  you'll notice that the value-premise of Ai tools is simply to provide quicker (and erroneous) how-to
  feedback, simply to bypass the marketing information that now gatekeeps even the most trivial developer tasks.
  I recently was handed a kafka-streams application, only to
* it rapidly seeks the most valuable components of the risk distribution across a project: languages, ease of testing
development-environment productivity, and all the other real components of software development.
