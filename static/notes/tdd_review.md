TODO: add summary notes here.
Sections:
- summary notes of TDD
- commentary
- rebuttals and rebuttals to the rebuttals
- patterns and practices: don't let tooling turn testing into a wasteful/fillibustering activity
- organization strategies:
  * surely there is a ratio of test coverage, such that one cannot cover all categories fully, unit, integration, performance, and QA.
  * how much is too much?
  * feedback loops: the secret goal of test-driven developers should be to build larger organizational feedback
    mechanisms by which to control various managerial antipatterns. The example I gave of bad managers at a company
    was inherent to these managers' greedy unwillingness to test their software; testing would have constrained
    the growth of these "personality ethic" charlatans and likely would have kept the company's software higher quality
    and more productive. Testing is a mechanism for beating back organizational demons: the charlatans, the ambitious
    connivers, and so forth. I can't stress this enough. Test-driven development is not simply a technical requirement,
    it has an organization governing property that keeps the loons and goons from scaling the citadel walls, orc-style.
    Test-driven development ensures a more productive software development lifecycle.
    * Possibly explain how code has a longevity component: untested weekend prototypes have a longevity of a few weeks,
    whereas fully abstracted and 100% test coverage libraries may have longevity in the decades or millenia. Your job
    is to find out what the upper confidence bound is, and then make the tradeoffs necessary to get to the highest value
    longevity components, while documenting or defensively-coding for the components that you cannot control.



I'm such a huge fan of test-driven development that it makes it difficult to write about.
A bunch of hyperbole about the "huge"-ness of test driven development makes it difficult to communicate the value something.

Nonetheless, I AM A HUGE ****** FAN OF TEST DRIVEN DEVELOPMENT.

Is this a personal open rebellion against companies, managers, and leadership to whom I've been subordinated that did not support TDD?
Possibly. Okay, almost certainly.
Whoever believes it is safe, prudent, or profitable to produce mission critical software without strong tests is,
well, a goddamn freak. Note that I rarely use such language, and treat my personal writings as more or less off limits
to vulgarity. And yet still, if you have ever worked for such outfits you know what I mean, since they are all the same personalities.
The so-called "leadership" of these institutions are essentially playing a form of surrogate Russian roulette: 
they spin the chamber, you provide the head. Organizations that produce software without tests are essentially
just a gigantic lie. The lies go upward, the responsibility travels downward. But ultimately, the lies prevail
over their denial. Nature abhor a lie. In some professions, lies are tolerated, still others, lies are quite 
possibly the profession's **causa sui**. But in engineering, you can only lie about the sheering properties of steel for so 
long before the entire member cuts loose and real people are harmed, or possibly killed.

While this sounds like personal commentary, I argue that how an organization tests its software tells you nearly every thing
you need to know about it. I mean this sincerely, as someone once deeply involved in security analyses using language and
graphical analyses: there is no greater insight into the health, leadership, and overall longevity of an organization than its test philosophy.

Pretty strong language, eh?

Here's why: there is no such thing as "test-driven development". The moniker portrays test-driven development
as a fad and an optional practice.

It isn't. There is no such thing as "test-driven development" because test-driven development *is* development.
The prefix is redundant because "test-driven" really means "requirements driven", and all software should know and exercise its requirements.
One of the greatest lies in software engineering is that there is a difference between code and tests.
Tests are code; and when written correctly, code should become merely background dressing to the foreground requirements being implemented.

My career started in testing, which at the time was seen as somehow remedial.
As my career progressed, I learned that critiques of testing were actually the remedial sort.

Every critique of test-driven development encapsulates the shortcomings of its proponent:
1) "Not all code is library code--it is difficult to test ____" (fill in the blank with integration/application/microservice code)
  This is the most common refrain I hear from self-described "hero" programmers in almost every organization.
  What no one tells them is that the heroics of writing and thereby owning a lot of code in their organizations is **because it was so poorly written** that no one else can work on it.
  Honor the "hero" programmer stereotype at your peril.
  The fact that a piece of code or a component in a stack is difficult to test is a smell indicating that it needs to be refactored.
  Worse yet, it may be an indicator that one doesn't understand one's own requirements.
  The latter is an easy trap to fall into, and in fact, when building on a new stack/language/etc, one discovers
  requirements through iteration. None of us are the geniuses portrayed on our resumes; iteration and velocity are king.
  And since we will spend most of our time understanding applications and technologies which are new to us,
  conceivably we are spending the majority of our time in which our requirements are unknown to us.
  This ratio between known/unknown and comfort/discomfort is really personal velocity, and each person has their own comfort level with it.
  In precisely this space, test driven development applies because it mitigates the devil-we-don't-know, turning it into the devil we do know.
  
  Do you know Golang? Do you marvel at the comprehensibility of its CSP-based concurrency model? Channels, selects, goroutines, etc.
  As do I.
  Did you know that Golang has a critical design issue that many library authors are unaware of?
  Its most common composite pattern for structuring CSP's is non-deterministic: (give the example).
  I wrote one of the first libraries of generic channel patterns while generics were still in proto.
  In the course of test-driven development, and **only** in the course of test-driven development,
  properties of code, languages, and integration points between technologies become painfully clear.
  Applications that one would assume cooperative suddenly break down, often in the space
  of competing languages, companies, and divided responsibilities.
  As engineers, we simply cannot assume that our industry will ever be mutually cooperative
  in terms of people or machinery; in an organizational sense, our job is essentially to 
  mitigate the unforeseen risks that exist in this space.
  
  
2) "We don't have money/time/resources to test our code"
  Do you have money to fail? It seems that way. 
  
  
  
  










