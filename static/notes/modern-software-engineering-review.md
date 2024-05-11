# Book Review

## Review Notes on Modern Software Engineering by David Farley

### Section III: Managing Complexity

#### Modularity

Most people understand modularity, the degree to which a system's components can be separated and to which they vary independently. Moreover, it can be defined in terms of the strength of the borders between components, and by the same token, their orthogonality with respect to one another.

Per some of Farley's language, modular code/design is not:

* overly long functions and non-clean code
* code that is difficult to test independently of other components or state
* microservices which cannot be tested independently

Some rote examples of modularity:

* microservices
* dependency injection
* fractal code

One of the most important points Farley makes is that test-driven development (TDD) drives modularity.
Testability's core requirement is that code and components be testable as standalone artifacts; therefore, test-driven development directly yields modularity.
A strong example of this is the linux tool philosophy of collections of tools that "do one thing well": grep, cut, curl, sed, and so forth form a set of modular tools that container images can include or exclude according to their respective requirements: networking, security, and so forth.

I am a huge fan of test-driven development, for no fashionable reason whatsoever. In fact, TDD will *never* be fashionable.
No matter what any developer or company proclaims, TDD will always experience organizational and interpersonal resistance in the face of time/money resources, inexperience, or sheer mismanagement and bad leadership.
Farley's point about the relationship between TDD and modularity is critical for this precise reason: TDD extends modularity, which in turn extends value and flexibility over time, and thus a more flexible software lifecycle.

More to the point, TDD counteracts and prevents the natural pressures toward short-term reward, such as spaghetti code that checks out on this release and allows a middle manager to check a box saying they released on time. The organizational pressure toward non-TDD development is the classic case of short-term reward derived from killing the golden goose for all of its eggs at the expense of a productive egg cycle. TDD imposes a consistent preventative mechanism that operates at the organizational time scale, rather than the manager/developer timescale of "welp, it works today, surely it will work tomorrow" business recursion.

Deployment pipelines are another example given by Farley.
Deployment pipelines essentially impose requirement of modularity and testability.
However, again there is a higher level epistemic requirement that one know at all times what one's code is doing, its quality, state, etc
This is the value proposition of devops in a nutshell.

Summarizing these points about modularity:

1) code- and design-level modularity is about clean code, testability, and fractal composition
2) organizational modularity requires (1) technically, but operates at a larger organizational timescale, and is implemented via continuous-deployment methodologies

Although Farley didn't split modularity into these two micro/macro forms, in my experience this is the proper way to characterize and measure them.

#### Cohesion

Put simply, one technical definition of cohesion is how well the natural language expression of functions aligns with the implementation.
Its about communication and encapsulation.
Whereas modularity is about something like standardization, cohesion is about the relationship between the code and the domain it is intended to describe.
Farley alludes to two primary concepts of domain-driven development:

* ubiquitous language: aligning domain language with the behavior of our code (including semantics, not just nomenclature)
* bounded-contexts: parts of a system that share a context. For example, a 'passenger' may have a different meaning within a ticket-ordering system than within a physical/spatial safety system.

Applying the techniques of aligning domain language with implementation results in more loosely coupled systems with respect to the domain, and the types of change we expect to handle within a domain's sub-contexts.

One way to distinguish cohesion from modularity, I contend, is that of an async library from the domain code in which it is implemented. For example, golang has a terrific concurrency model atop of which several templated libraries are available to implement arbitrarily complex computational graphs  (in essence). A developer then includes this library (modularity), and implements higher level functions with natural sounding names like "AcceptRequest" or "EnsureDeposit". I choose the term "Ensure*" intentionally because it is often used to communicate a critical operation. Thus, cohesion is one level above modularity.
I have no idea whether or not Farley would agree with this interpretation, but this is the only way to define it such that the definition of "cohesion" does not simply leak back into modularity.
Which is an ironic example of non-cohesion...

In slight disagreement, Farley defines cohesion as being at odds with modularity, based on the concept that cohesion encourages relatedness between components whereas modularity is about strong boundaries.
Place components that change together close to one another in the code; at some point, we must forego modularity in order to combine components.
Frankly, I think this is a jumble of ideas, since cohesion/modularity can be implemented at different layers, i.e. hexagonally, but I understand Farley's line of reasoning.

Summarizing:

* bad cohesion is identified by looking at a piece of code and "not knowing what it does"
* cohesion is about aligning natural domain language with code constructs and their capacity/likelihood of change
* cohesion can be seen as counter to modularity, since it encourages relatedness between similar components

#### Separation of Concerns

Per Farley, separation of concerns operates at every level of granulatity, from code regions to whole systems.
As he calls it, "[separation of concerns] is a take on cohesion and modularity" that "keeps one's focus small".
Dependency injection provides the example of separation of concerns, because it allows components to "do one thing well" and yields testability.

Farley gives two examples of complexity:

* essential complexity: the natural complexity determined the value our code is intended to deliver
* accidental: "everything else", i.e. the incidental complexity of implementations, low level dependencies, process/organizational dependencies (think build systems), security, and so forth.

These distinctions provide the guidance by which to organize our code, for instance to separate domain transactions from the management of an underlying database resource.
The simplest definition here is the separation of business logic from underlying system rules/implementation.
Interfaces (golang, c#, Java) provide good examples of where you put your business logic, and whose implementers deal with the 'accidental' complexity of languages, systems, etc.

Ports and adapters:

* port: an interface through which information flows
* adapter: a translation service for the information provided to it

In the context of code, ports are things like "stores" which are interfaces to things like an S3 store, blob store, file store, etc.
Per Eric Evans, ports and adapters exist wherever you need to "translate information that cross between bounded contexts".
This is a good rule of thumb, and gets to the heart of separation of concerns per readability and testability.
If you implement code in this fashion, then each implementation entails fewer concepts to comprehend as a developer, fewer test cases, and so on.

Summary:

* separation of concerns is about identifying the transition between bounded contexts, a nice heuristic for identifying port/adapter patterns
* separation of concerns is about distinguishing accidental and essential complexity
