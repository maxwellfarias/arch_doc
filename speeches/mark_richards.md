How to Think Like an Architect
Speaker: Mark Richards

You don't have to be a software architect to think like a software architect. Doing so gives you several advantages: first, a more effective and well-built system even as a developer; and second, a good head start on a really fun and rewarding career path. That is what I want to show you in this Keynote.


The Clouds Metaphor
I want to start by showing you clouds. What are these? Well, they are clouds. But it is fascinating because, to different people, these clouds represent different things.

To a meteorologist: They look up at the sky at those clouds, see the cloud types, and know the weather patterns and what to expect.

As an artist: You might look up at those same exact clouds and picture a beautiful, dramatic scene.

As IT professionals: When we look up at the sky and see those clouds, we think of cloud-based deployments and the Internet of Things.

When I show you those clouds, everybody sees them with a different eye and a different perspective. That is what I want to show you about architectural thinking. What does it mean to think like an architect? To see a problem with an architect's eye and an architect's point of view.


Why Thinking Like an Architect is Important
Let's look at a situation where we are doing some messaging or event-driven architecture. On the far right-hand side, there is an order placement service. It is going to send a message to payment and inventory. Once I place an order, I have to pay for it, and we have to decrement inventory. Of course, we have a database.



You have a choice to make as a developer. When I send that order, should I include the entire order payload—all 45 different attributes—or should I only send just the key (the ID)? 

Full Data Payload: I insert the data and send that message over. Payment and inventory have all the data now.

Key-Based Payload: I still insert the order into the database, but I only send the key out in that message. Payment and inventory receive that key, but now they have to query the database to get the information they need.

The Perspectives

The Developer's View "Obviously," says the developer, "the full event payload is the right choice because we get much better scalability and performance. I don't have to go to the database. This is one of the ways of creating scalable systems and high-performance systems. This is the clear choice".


The Architect's View "Yeah," says the architect, "but what sort of contract are you going to use? Is it going to be a strict contract with a schema, or are all 45 of those attributes going to be a loose contract with something like JSON?  Are you going to pass an object in that message? How are you going to manage it? How are you going to version it when you make changes, so you don't break everything?".


"Oh, and what are you going to do about Stamp Coupling and bandwidth issues?".

The Problem with Stamp Coupling

Stamp Coupling is a form of coupling where you pass all the information to a particular service, but it only uses a few pieces of it—maybe one or two attributes. Two problems occur:

Change: If I change a field in the contract that payments and inventory are not using, I still have to change the contract. That might facilitate a change in payment and inventory as well, even though I don't care about that field.

Bandwidth: That entire order is 500KB. Payment only needs 350KB; inventory only needs 50KB. I am unnecessarily utilizing bandwidth I don't need. In cloud-based environments, this could get quite expensive.


There is another problem: Multiple Systems of Record. If you are passing the entire payload, everyone has a copy of that data, and they aren't looking at the database where the true system of record is. If I make changes to that order, it may be everywhere, and you might not have the latest copy, leading to data integrity issues.


The Realization

The developer says, "You know, those are really good points. Maybe a key-based event payload would be better. I wouldn't have to worry about stamp coupling, bandwidth issues, and I get better data integrity".


But then the architect says, "Yeah, but what are you going to do about scalability and performance now?".

This brings us to the First Law of Software Architecture:

Everything in software architecture is a trade-off.

Being able to think like an architect allows you to better analyze these trade-offs, to find the right decision, and to seek out those pros and cons.

The Three Core Pillars of Architectural Thinking
Thinking like an architect involves three core things, regardless of your role:

Understanding Business Drivers: Translating business requirements into architectural characteristics (scalability, availability, etc.).

Expanding Technical Breadth: Knowing more solutions to the problem.

Analyzing Trade-offs: Identifying the pros and cons of every decision.

Let's look at each of these.

1. Business Drivers and the "Translation Engine"

The business is concerned with things like user satisfaction, time to market, competitive advantage, mergers and acquisitions, and regulatory compliance. Thinking architecturally is translating these concerns into what the architecture needs to support. We translate those into things like fault tolerance, reliability, recoverability, responsiveness, performance, and availability.



The problem is being "Lost in Translation." The language of an architect is all about fault tolerance; the language of the business is about user satisfaction. These two rarely meet. The brain of an architect must become a translation engine.


Examples of Translation

"User Satisfaction is the most important thing."  We translate that into performance (responsiveness), agility (fixing bugs fast), scalability, availability, security, and recoverability (so users don't lose work). If our architectures don't support these, users won't be happy.


"Time to Market is the most important thing." We translate that into maintainability, testability, and deployability.


"Mergers and Acquisitions." This translates to interoperability (leveraging standards to communicate with acquired systems), extensibility, and scalability (can we support the sudden double in customer base?).


These characteristics help us qualify and make decisions. For example, if you need high scalability, you might look at Microservices, Event-Driven, or Space-Based architectures. If you need simplicity and low cost for a startup, perhaps a Monolithic architecture is the right choice initially.


2. Expanding Technical Breadth (The Triangle of Knowledge)

As an architect, I have to visualize possible solutions. If you only know one solution, it might not be the right one.

There is a concept called the Triangle of Knowledge regarding technology:

Top (Smallest) - Stuff you know: You do this every day. You are an expert (Technical Depth).

Middle - Stuff you know you don't know: You've heard of it (e.g., Clojure), you know what it is generally, but you can't code in it.

Bottom (Largest) - Stuff you don't know that you don't know: The perfect tool or pattern for your problem that you have never heard of.

The "game of life" for an architect is to take stuff from the bottom (Unknown Unknowns) and move it to the middle (Known Unknowns). You do this by talking to people, going to conferences, and attending talks on topics you've never heard of.


Career Evolution through the Triangle

Junior Developer: The "Stuff You Know" is small, but that is appropriate. You are hired for capability in a specific technology (e.g., Java).

Senior Developer: The "Stuff You Know" grows significantly (multiple languages, patterns, frameworks).

Junior Architect: You sacrifice some of that deep expertise ("Stuff You Know") to broaden the "Stuff You Know You Don't Know". You might not know how to code in ten different caching technologies, but you know what they are and their pros/cons.


Senior Architect: The middle section (Known Unknowns) becomes massive. You have significantly broadened your horizon.

The 20-Minute Rule

You might say, "Mark, I work 12 hours a day coding; I don't have time". I don't have time either. So, I use the 20-Minute Rule.


I spend 20 minutes a day—first thing in the morning—on myself and my career. Before I check email (because once you check email, your day is over), I grab my coffee and learn something new. I scan resources like InfoQ, DZone Refcards, or the ThoughtWorks Technology Radar.





If I see a buzzword I don't know (e.g., "Apache Pinot"), I click it. I spend two minutes reading what it is. Now, I have moved it from "Unknown Unknown" to "Known Unknown".

3. Analyzing Trade-offs

Rich Hickey, the creator of Clojure, has a quote:

"Developers know the benefits of everything and the trade-offs of nothing".

We get excited about technology and only see the good stuff. But remember the first law: Everything is a trade-off. In the structural aspect of software architecture, there are no best practices. Everything depends on context.


Example: Shared Library vs. Shared Service

Imagine you have common functionality (calculators, utilities). Should you use a Shared Library (JAR/DLL) or a Shared Service? 

Shared Service Pros: Better for polyglot environments (heterogeneous code), easier to fix bugs instantly without redeploying every client.

Shared Service Cons: High change risk (if you break the service, you break everyone), latency, lower fault tolerance, harder to version.




Shared Library Pros: Compile-time dependency, better performance, no network latency.

Shared Library Cons: If you update the library, every application using it must rebuild, retest, and redeploy.

The "Out of Context" Trap If you just count the checkmarks of pros and cons, you might pick the wrong one. You must apply context.

If your context is "We use 4 different programming languages and change logic frequently," a Shared Service wins, despite the performance hit.

If your context is "High performance is critical," a Shared Library wins.

Pro Tip: Do Not Over-Evangelize Avoid over-evangelizing a technology (e.g., "gRPC is the best thing ever!"). When you evangelize, you hide the trade-offs. Architects are often seen as "grumpy" because we see the trade-offs in everything.



Conclusion
That is architectural thinking:

Having those foundational business drivers drive your decisions.

Expanding your technical breadth so you have more solutions to choose from.

Analyzing trade-offs to ensure the solution fits the specific context.

You don't have to be an architect to think like one. Start practicing this now. It will help you create more effective, efficient, and correct software solutions as a developer, and it will propel your career forward