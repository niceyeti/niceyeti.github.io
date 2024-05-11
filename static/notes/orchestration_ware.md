# Orchestrationware: Notes on Graphical Orchestration Libs

These notes are an evaluation of existing orchestration systems and their reqs.

Description: orchestration is any graphical workflow process, most of which are
a core business responsibility, but which also end up being encoded across a range
of software artifacts: cdk projects, build scripts, etc. The best example I can provide
is the inevitable decay of almost any build system into a mush of glue-code: while a graphical
description of a build system is an obvious responsibility, integration points and coupling
(by applicationware stacks like Gitlab, artifactory, etc) erode a central, onion-like description
of its orchestration. The popular DevSecOps lemniscate and "shift left" promise good
clean orchestration of build systems, and yet arguably no such system exists!

A good orchestrator is the distributed-system aggregation of few common patterns (loosely):

* observers
* chain-of-responsibility
* computational graphs
* automatic differentiation: not directly applicable to workflow systems, but the implementation
  of AD is an example of the kind of self-inspection features that a workflow system should provide.
* automated analysis: e.g. petri nets, searchability, verification properties

Some examples of orchestration libraries and frameworks:

* AWS StepFunctions
* Tilt (more later on why I like Tilt as a workflow/orchestration example)
* Kubeflow
* AirFlow
* Pegasus
* Dagster
* Flow
* ArgoCD?

A colleague of mine mentioned that all such "workflow" systems (aka orchestrators) become esoteric
to their narrow set of requirements and usecases because they all make it 80% of the way
to full orchestration, but never achieve the remaining 20% where most of the value of a
truly generic workflow system lies.

Desired properties:

* Simple one-file language-based description (e.g. Tiltfile)
* DAG construction
* versioning: rewind through previous iterations of a workflow, functioning at each semver
* distributed guarantees: scaling (e.g. AWS lambda or k8s HPAs)
* visualization: optional, but should be capable
* Inferred compilation: like Tilt, changes to inputs and resources should trigger redeployment automatically.
  A more familiar example is simply continuous-deployment systems.
  
## Dagster

A good way to think about Dagster is in terms of its similarity to observer patterns in front-end libraries,
such as various state management libraries involving composable "Stores" by which one can build
computational graphs describe the desired state of data, and its input/outputs. I mention this
because once you notice the similarity, grasping Dagster (and perhaps even its shortcomings,
w.r.t. the advancement of js state libs) is a lot more familiar.

Dagster allows you to specify "Assets" can be included into one or more "Jobs" which execute
based on some "Scheduling" parameters. Orthogonal components are also included,
such as a Context object for recording information (job health, stats, etc.).

* Asset: Assets declare their own dependencies on one another (IMO a drawback, since it is coupling)
  Assets represent a coherent set of actions for a particular task; its okay to think of them as "jobs"
  but this is introduces ambiguity w.r.t the actual "Job" abstraction. An asset may have quite a bit
  of unfactored spaghetti: read some file, make an http request based on it, generate a plot,
  output a csv file or pandas frame.
* IO Managers: use these for side effects during execution, such as persisting data that won't fit in memory.

Criticism: the Dagster classes are weirdly factored, and the api feels tightly coupled to their abstraction.
A reorganization of the layers would give the generic workflow tool that I have in mind.
There are also typical pythonic anti-patterns, like binding resources to strings of the same name
as some function/resource, such that it can be retrieved via this hard-coded value elsewhere in
some other file (which won't propagate change, and therefore is guaranteed to break).
So the system is kind of a first crack at the problem, specific to pythonic data pipeline domains
and fishy ui requirement.
Benefits: despite its factorings, I am confident that Dagster could be the orchestration engine
atop of which you could implement your own domain specific language. For example, think of implementing
the Tiltfile language using Dagster: once you learn Dagster, you can see almost immediately
how it extends such a case. The "case" being that of 1) specifying some domain language
2) implementing it under some engine, ie Dagster. There is a clean separation between domain
and engine, and Dagster provides a great example of an engine, despite its warts.
