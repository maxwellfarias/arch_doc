How to Think Like an Architect
Speaker: Mark Richards
You don't have to be a software architect to think like a software architect. Doing so gives you several advantages: first, a more effective and well-built system even as a developer; and second, a good head start on a really fun and rewarding career path. That is what I want to show you in this keynote.
The Clouds Metaphor
I want to start by showing you clouds. What are these? Well, they are clouds. But it is fascinating because, to different people, these clouds represent different things.
To a meteorologist, they look up at the sky at those clouds, see the cloud types, and know the weather patterns and what to expect. As an artist, you might look up at those same exact clouds and picture a beautiful, dramatic scene. Of course, all of us are in IT, so when we look up at the sky and see those clouds, we think of cloud-based deployments and the Internet of Things.
When I show you those clouds, everybody sees them with a different eye and a different perspective. That is what I want to show you about architectural thinking. What does it mean to think like an architect? To see a problem with an architect's eye and an architect's point of view?
Why Thinking Like an Architect is Important
Let's look at a situation where we are doing some messaging or event-driven architecture. On the far right-hand side, there is an order placement service. It is going to send a message to payment and inventory. Once I place an order, I have to pay for it, and we have to decrement inventory. Of course, we have a database.
You have a choice to make as a developer. When I send that order, should I include the entire order payload—all 45 different attributes—or should I only send just the key (the ID)?
With a full data payload, I insert the data and send that message over. Payment and inventory have all the data now. But with a key-based approach, I still insert the order into the database, but I only send the key out in that message. Payment and inventory receive that key, but now they have to query the database to get the information they need.
The Developer's View:
"Obviously," says the developer, "the full event payload is the right choice because we get much better scalability and performance. I don't have to go to the database. This is one of the ways of creating scalable systems and high-performance systems. This is the clear choice."
The Architect's View:
"Yeah," says the architect, "but what sort of contract are you going to use? Is it going to be a strict contract with a schema, or are all 45 of those attributes going to be a loose contract with something like JSON? Are you going to pass an object in that message? How are you going to manage it? How are you going to version it when you make changes, so you don't break everything? Oh, and what are you going to do about Stamp Coupling and bandwidth issues?"
The developer asks, "What are you talking about?"
Stamp Coupling is a form of coupling where you pass all the information to a particular service, but it only uses a few pieces of it—maybe one or two attributes. Two problems occur: change and bandwidth issues.
1. Change: If I change a field in the contract that payments and inventory are not using, I still have to change the contract. That might facilitate a change in payment and inventory as well, even though I don't care about that field.
2. Bandwidth: That entire order is 500KB. Payment only needs 350KB; inventory only needs 50KB. I am unnecessarily utilizing bandwidth I don't need. In cloud-based environments, this could get quite expensive.
There is another problem: Multiple Systems of Record. If you are passing the entire payload, everyone has a copy of that data, and they aren't looking at the database where the true system of record is. If I make changes to that order, it may be everywhere, and you might not have the latest copy, leading to data integrity issues.
The Trade-off:
The developer says, "You know, those are really good points. Maybe a key-based event payload would be better. I wouldn't have to worry about stamp coupling, bandwidth issues, and I get better data integrity."
But then the architect says, "Yeah, but what are you going to do about scalability and performance now?"
This is why thinking like an architect is important. It brings us to the First Law of Software Architecture:
