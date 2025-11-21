# The Principles of Clean Architecture
**Speaker:** Robert C. "Uncle Bob" Martin

## The Mathematics of Ancestry

How many biological parents do you have? Generally, two. Yeah, actually, always two; it's one of those laws of nature. Even in Norfolk, it's going to be two. But how many biological grandparents do you have? Four. Generally, almost always four. Yes, four would be very close to the appropriate number.

Great-grandparents? Not to exceed eight. Yes, this is a sequence we programmers know, isn't it? This is powers of two. Great-great-grandparents would be... well, if you've lived in the same area for a long time and there are cousins that live nearby, it might be less, but still probably 16. Although there's a good chance it's something less than 16.

But let's not dilly-dally around this region. Let's go out 10 generations—200 years. That's $2^{10}$, so the maximum would be 1,024 possible ancestors. But probably not; in fact, there is a really good chance that it's not 1,024 because the whole "cousin thing" starts to really take effect at that point.

If we were to go back another 200 years, right to 400 years ago—say, to 1615—you'd have a million possible ancestors. But it's almost certainly nowhere near a million. And if you were to go back another 200 years to 1415 (somewhere before Columbus sailed), you would have a billion *possible* ancestors. Although there probably weren't a billion people on the planet—or maybe just a billion. One more 200-year jump would take you back to the year the Magna Carta was signed, in 1215, and the maximum possible number of ancestors would be a trillion. But there haven't been a trillion people on the planet, even including all the dead ones.

So, this "power of two" thing doesn't work. It only works in the first couple of generations, and then it sort of curves around. You can imagine this growth curve of your possible ancestors, and then the *real* ancestors kind of curve around. Then, another effect takes place.

Let's go back 600 years. All right, there were people alive back then, and they had children. But if you were to look at their lines—their progeny—some of them died out. Some of their children didn't make it to 2015. The lines just died out. In fact, every line has a positive probability of dying out. Therefore, if you go back far enough, you eliminate every line except one.

So, you and I would have a single person somewhere in the deep, dark past—a single pair of people, one man, one woman—who were the parents of everybody in the room. Our little private Adam and Eve, so to speak. If we did this analysis with everyone on the planet, we'd have to go back further in time, but we could find a single man and a single woman who were the sole parents of everyone alive on the planet. They probably didn't know each other. They probably didn't live within a few hundred miles of each other. They very likely did not live within a few thousand years of each other. But these two individuals are the mother and father of everyone alive, and this is a mathematical certainty.

## Written in the Molecules

What can we know about them? Well, it turns out that there are certain molecules in our body that must have come from them. These molecules are relatively undisturbed. Most of our DNA gets chopped up into little bits and mixes left and right, so you couldn't possibly trace it out. But there is a strand of DNA in your body which comes only from your mother, and it lives in a little organelle in every one of your cells. This organelle is called the mitochondria (not the "midichlorian").

The mitochondria is a cute little hot-dog-shaped organelle which takes in glucose and oxygen and emits water, carbon dioxide, and energy. This is the little device within our cells which does the chemical reaction of respiration, of life. It has its own DNA and reproduces on its own schedule. It's almost a little bacterium; in fact, it very closely resembles a little bacterium. Apparently, a billion and a half years ago, one cell swallowed another and failed to digest it. The swallowed cell thought, "This is nice in here," and produced energy. The swallowing cell thought, "Gee, it's nice to have someone produce energy for me." A symbiosis was set up that led to you and me.

Now, if you were to take your mitochondrial DNA—which you got from your mother, not from your father (because egg cells have mitochondria and sperm cells don't)—and compare it to your mother's, it would be identical. Almost certainly. If you were to compare it to your maternal grandmother, it would be identical. However, if you were to go back far enough in time, there is a small chance that there would be a mutation. There are lots of possible mutations; a cosmic ray can come along and hit an egg cell at just the right moment and flip one little DNA nucleotide. Most of the time, when you flip a nucleotide, it doesn't do any damage because the genetic code is worked out so that single-bit errors don't accumulate.

So, that's interesting. We could probably lay them aside and see how long it took a mutation to take place. If you were to take your mitochondrial DNA and lay it next to mine, we might see a few differences. The number of differences would tell us how soon we had a common mother. We could do this analysis with everyone on the planet. You don't have to do everyone, just a spot check—a few people from here and there—and you could build a tree of the descendants of everybody. You could tell how far back in time we all had common mothers.

This work has been done, of course. The "Mitochondrial Eve"—the woman who is the mother of everyone alive—lived probably 120,000 years ago in East Africa. We know it's East Africa because this is where the mitochondrial DNA has the most diversity; it's had the most time to change, mutate, and spread. If you go to Iceland, all the mitochondrial DNA is exactly the same. But in East Africa, it's all really different.

So, she lived in East Africa 120,000 years ago, at a time when the weather was warm but about to get cold. The last Ice Age started shortly after that and only ended 14,000 years ago. We can trace what happened to her children. We know that her children left East Africa, went North, and then moved to the West into Western Europe. A completely different cohort moved North and then went to the East into Eastern Europe, over the steppes into Russia. A different batch entirely went around the oceans all the way down into Australia. There was another batch that managed to leave Russia, cross the Bering Strait, and go into North America about 14,000 years ago, winding up at the tip of South America a thousand years later.

Men in the room, you have a stretch of DNA that comes from your father: the Y chromosome. The Y chromosome only comes from Dad; Mom's DNA does not mix with it. It is a hunk of pure testosterone coming from your father, and his father, and his father, and so on—a single inheritance hierarchy going all the way back to "Y-Chromosome Adam," who lived 70,000 years ago at the height of the last Ice Age in East Africa. We can trace the migration of his sons, and it's roughly the same (oddly, not quite the same, because they had very different timings) as the migrations of Mitochondrial Eve. The map of these migrations is very precise and detailed.

I find that very interesting—that the history of the human race is written in the molecules of our body. But that's not why we're here, is it? I need to talk about architecture instead.

## Architecture and Intent

I guess it was about eight years ago now, my son came to me. He had written a Ruby on Rails app and showed it to me. I looked at it and immediately recognized, "Oh yeah, that's a Ruby on Rails app." Because you can tell, right? It's got the `app` directory, it's got the `controllers`, `models`, `views`. This is a Ruby on Rails app. Anybody who has done Rails would look at that and go, "Yep, Ruby on Rails app."

Then it occurred to me, for the first time in many years: **Why do I know that this is a Ruby on Rails app?** I'm looking at the high-level directory structure. Why does the high-level directory structure of this application tell me the framework that I'm using? Why doesn't it tell me what the application *does*?

That bothered me. I thought, "Wait, the highest-level directory structure in the system—shouldn't it tell me what the application does? Isn't that what's important about this application?" Why is the framework standing out in front of everything, hiding what this application is supposed to do? I thought, "Well, that's just not right."

What is Rails? Does anybody do Ruby on Rails here? Okay. You probably use some other web framework. Ruby on Rails is a web framework. I thought, "Ruby on Rails is a web framework. Why is it so obviously in front of everything when the Web is an I/O device?" Why would the Web be considered something so significant that it would shadow what the application does?

There are reasons for that. When the Web came out in the late 90s, it changed everything. The Internet is here, the Web is here! It took us close to 20 years to realize it didn't change *anything*. The Web was just an I/O device like any other I/O device. It takes input from a user and delivers output to the screen. Okay, it uses this odd HTML language to do that, but so what?

If you remember those heady days of the Dot-com era, everyone was making a trillion dollars by putting up a "Save a Pet" website. That was the way things were until everybody lost the trillion dollars, and then we had programmers holding signs saying "I will code for food." Those were bad days. The Web had dominated us to the point where we thought the Web was an architecturally significant element of a system. But, of course, it's not. It's an I/O device.

Rails, and every other web framework out there, took an architecturally significant position in all of our systems. To the extent where you would describe your application as a "Rails app." Why would you do that?

"I'm working on a Rails app."
"No, you're working on an accounting app. You may happen to be using Rails, but you're working on an accounting app."

"I'm using Spring."
"What is the architecture of your application?"
"Well, the architecture of my application is that we're using Java, Spring, and Hibernate."
"That's not an architecture; that's a bunch of tools. What is the *architecture* of your application?"

I started thinking about this. What do architectures tell us? I got some blueprints out. I looked at a library blueprint. You can tell it's a library: it's got bookshelves, desks, computers sitting over here, tables you can sit on. That looks like a library. The blueprint is trying to tell you: **"I am a library."**

If you look at a blueprint of a church, you can tell it's a church. It's screaming at you: **"I am a church."**

It occurred to me that **architecture is about intent**. It's about the intent of the system. It's not about the tools; it's not about the frameworks. Those are details—details that should be hidden, not exposed.

## Rediscovering Ivar Jacobson

That reminded me of decades long past. It reminded me of this guy who had the same message in 1992: Ivar Jacobson. In 1992, he published this book (actually, it was a bunch of his employees who wrote the book and he took credit for it, but that's okay): *Object-Oriented Software Engineering: A Use Case Driven Approach*.

I remember how excited we were in the early 90s when this book was about to come out. It was a classic; it remains a classic today. Does anybody remember Use Cases? And how excited we were, and then how awful it was later?

What happened to Use Cases was the invasion of the consultants. The consultants came in, and every consultant has to outdo the last consultant. At first, they said, "Oh yes, I can do Jacobson's Use Cases." Then they had to create forms so you could download them on the Internet. Then every consultant had to have a better form than the one before him. The forms got more and more elaborate; eventually, you had to fill out primary actors, secondary actors, tertiary actors, preconditions, postconditions... it turned into a nightmare. The poor people who were supposed to be doing Use Cases said, "No, we can't do this."

That was just at the point where the Agile Revolution started, and everybody said, "Let's do User Stories instead." Nobody quite understood that there's no difference between a User Story and a Use Case—they are the same thing—but at least we didn't have the form.

A "Jacobsonian" Use Case looks like this: a very simple, straightforward kind of thing. This is the "Create Order" Use Case in an order entry system. It tells you what the incoming data looks like (Customer ID, shipment destination, payment information). Notice that I am not describing the structure of this data; I'm just saying it exists. The Use Case is not the appropriate place to identify the structure of the data. Notice I'm also not saying it's coming from the Web, although this might be a web-driven system. Why would the Use Case say a thing like that? It's just telling you that this data is going to come in from somewhere.

Then the Use Case describes the "Primary Course"—what happens if everything goes right:
1. The order clerk issues the "Create Order" command with the above data.
2. The system validates all the data. (I'm not going to tell you *how* it validates or give criteria; that's for later).
3. The system creates the order and determines the Order ID.
4. The system delivers the Order ID to the clerk.

This is a very typical Jacobsonian Use Case.

## Interactors and Boundaries

Jacobson said you can take that Use Case and turn it into an object. He called these objects **Control Objects**. I call them **Interactors** because I didn't like "Control Object" (too close to Model-View-Controller). The Interactor object contains **application-specific business rules**.

What is an application-specific business rule? There are two kinds of business rules:
1. **Business rules that are true across many applications:** For example, pricing an order (adding up prices) might be in an order system and an order processing system. This is *not* application-specific. Jacobson put these into objects he called **Entities** (or Business Objects).
2. **Business rules specific to a particular application:** For example, the process for creating an order is specific to the order entry system. Interactors are always application-specific; they are the Use Cases of the application.

Usually, an Interactor will talk to several Entity objects (Order entity, Line Item entity, Customer entity).

Jacobson said there's one more kind of object: **Boundary Objects**. At first, he called them Interface Objects. I have drawn them as interfaces (Java-like interfaces). Data comes into the Interactor through the Input Boundary (implemented by the Interactor) and goes out through the Output Boundary (implemented by something else).

### The Flow of Data
Here is the flow:
1. The user pushes a button (maybe a web form submit).
2. The **Delivery Mechanism** (web, console, thick client—doesn't matter) takes the data and sticks it into a data structure called a **Request Model**.
    * *Note:* This Request Model should be a plain old object (POJO/POCO) with no dependencies on any framework. No `HttpServletRequest`.
3. The Request Model gets passed into the **Input Boundary**.
4. The **Interactor** receives the Request Model, decodes it, and uses that data to control the "Dance of the Entities." It tells the entities to create an order, add line items, etc.
5. When done, the Interactor gathers results into a **Result Model**.
6. The Result Model is passed out through the **Output Boundary** to the Delivery Mechanism.

**Can you test those Interactors?** Yes. You don't need the Web running. You don't need the database running. All you need is the Interactor and its Entities. You pass data in through the boundaries and collect data out.

This is a fundamental goal of writing tests: you do not want to have the whole system running, especially slow parts like the Web server or Database. You want to talk right to the business rules.

## Model-View-Controller (MVC)

You might be wondering, "What about Model-View-Controller?"

I met the inventor of Model-View-Controller a couple of years ago: **Trygve Reenskaug**. I met him in Norway. I was looking for a place to plug my laptop in, and this old, grizzled guy hands me a power strip. Our fingers barely touched. I haven't washed that hand.

The MVC architecture he came up with for Smalltalk looks like this:
* **Model:** Hides business rules. Not allowed to know about the Controller or View.
* **Controller:** Gathers input from a device and translates gestures into method calls on the Model.
* **View:** Registers with the Model. The Model calls back to the View (Observer pattern) to say "I've changed." The View then presents the data.

Input -> Process -> Output.

In the Web era, we lost the original idea. People thought, "MVC is good, my web app is good, therefore my web app is MVC." Nowadays, Web MVC looks like this:
* The Web hits a Controller.
* The Controller unpacks the URL/Form data.
* The Controller calls business objects (Model).
* The Controller passes control to the View.
* The View queries business objects to display data.

The problem is **no solid boundaries**. Controller logic leaks into business objects; business logic leaks into Views. It turns into a mess because we aren't good at keeping discipline.

## The Plugin Architecture

In Clean Architecture, there is a double black line. This represents a **hard architectural boundary**. All dependencies must point across this line in **one direction**: always pointing *towards* the business rules.

This is the fundamental architecture of a **Plugin**.

Think about Visual Studio and ReSharper (a plugin). They are built by different teams (Microsoft vs. JetBrains). Who can screw whom? Microsoft can put ReSharper out of business just by changing their interface. Microsoft is immune to ReSharper; ReSharper is vulnerable to Microsoft.

**Plugins are vulnerable to the things they plug into, but the things they plug into are immune to the plugins.**

Don't you want your business rules to be immune from the strange, bizarre changes that occur in the UI? Of course. That means the UI should be a plugin to the business rules. The UI changes a lot; business rules change less often.
* **Rule:** If something changes a lot, it should be a plugin. If something doesn't change often, it should be plugged into.

### The Presenter
The Interactor creates the Result Model (raw data: Date objects, Money objects). It passes this to a **Presenter**.
* The Presenter's job is to take the Result Model and translate it into a **View Model**.
* The View Model contains only strings and simple flags (booleans).
* The Presenter formats the date string. It adds currency symbols. It determines if a number should be red (negative). It decides if a button should be grayed out.
* The **View** object is so stupid it has no logic. It just moves data from the View Model to the screen.
* **Benefit:** You don't have to test the View (test it with your eyes). You *can* test the Presenter logic via unit tests without the GUI.

## The Database is a Detail

Does the database call you in your sleep? Is it the great god of all applications?

DBAs would like that view. But where did DBAs come from? **Oracle.** Oracle went to CEOs in the 70s and asked, "Are you protecting your data assets?" The CEO said, "I didn't know I had data assets." Oracle said, "You need a Database Administrator."

It is ironic because databases are just **I/O devices**. You store bits on a disk. They have tools (indexing, searching), but they are details. If the database is the center of your architecture, you have made the same mistake as Rails.

Why do we have relational databases? Because **disks are a pain in the ass.**
A disk is a spinning platter. To read data, you have to move a head (seek time, ~100ms back in the day), wait for rotation (latency), and read a huge 4K sector just to get one byte. It was slow. We invented indexes and tables to make getting data off this spinning magnetic surface efficient.

**But who has a spinning disk in their laptop today?** Almost no one. Disks are going away. We have huge arrays of RAM (SSDs/Flash) that look like RAM.
If you had persistent RAM, would you use a relational table? No. You would use linked lists, hash tables, and binary trees—the structures programmers actually like. We only unpack data into those structures after fighting with SQL anyway.

If I were Oracle, I'd be scared. The reason for their existence (managing spinning disks) is evaporating. This is why the **NoSQL** movement exists—we don't need the heavy relational machinery for everything anymore.

### Abstracting the Database
You should treat the database as a plugin.
* The Interactors use an interface called an **Entity Gateway**.
* The Entity Gateway has methods for every query you need (`getOrdersSince(date)`, `saveOrder(order)`).
* The implementation of this interface lives behind the boundary (in the database layer). This is where the SQL lives.
* **Result:** There is no SQL in the business rules. The database plugs *into* the business rules.

This allows you to replace the database with "Stubs" for testing. Your tests can run in microseconds because they aren't hitting a disk.

## The Importance of Fast Tests

If your test suite takes too long (more than a couple of minutes), you won't run it. You'll defer it to the nightly build. If you don't run the tests yourself, you can't make decisions.

If the test suite passes, and you trust it, you can **ship it**.
If you have a test suite you trust, you can **clean the code**.
How many of you have seen ugly code, wanted to clean it, but thought, "I'm not touching it because I might break it"? If that is your reaction, the code will rot. You will only make it worse. The only way to keep code clean is to have a test suite you trust so you can refactor without fear.

## The Fitness Story

In 2002, my son and I began a project called **Fitness** (a Wiki). We needed a database to store pages. We were about to install MySQL when someone said, "We don't need MySQL yet. Let's focus on the Wiki-to-HTML translation."

We mocked the database (`MockWikiPage`). We stored pages in a hash map in memory. We worked for a year this way. We could run tests, but we couldn't save anything permanently. Eventually, we needed persistence. Michael Feathers said, "Just write the hash map to a flat file." We did that. It worked.

Three months later, we realized: **We don't need a database at all.** The file system was fine. We pushed the database decision right off the end of the project.

**Good architectures allow major decisions to be deferred.**
You don't need to decide which database, which web framework, or which tools to use at the beginning. You can build the business logic first.

## Conclusion

A good architecture maximizes the number of decisions *not* made. You achieve that by using a **Plugin Model**.
* The UI is a plugin.
* The Database is a plugin.
* Frameworks are plugins.

**A note on Frameworks:** Framework authors are not out to screw you, but they *will* screw you. They will change the framework to suit *their* needs, not yours. You must keep frameworks at arm's length. Don't let framework code (like `@Autowired` or specific base classes) leak into your business rules. Inject dependencies via Factories into the plugin layer, not the core.

Architect your system so that if a framework becomes a liability, you can break away from it.

Any questions?

**Audience Member:** "Stored procedures can be used to abstract the database so programmers can't access tables."
**Uncle Bob:** "Exactly. Entities are not instances of database tables; they are constructions. Good point."

All right, beer. Thank you.
