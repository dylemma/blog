A while ago, I learned a new way of thinking about the [observer pattern](http://en.wikipedia.org/wiki/Observer_pattern). Basically, you start thinking about the Publisher of events as a *Collection* of events. Since I'm also thinking about Scala, this means that when I treate a Publisher as a Collection, I also expect that it will come with all sorts of convenient "combinators" (means of creating a transformed version of a collection). The act of attaching a Subscriber to the Publisher becomes the act of traversing the collection.

To demonstrate, here are a few "before" and "after" examples.

	//BEFORE
	val pub = new Publisher[Int]

	//AFTER
	val events = new EventSource[Int]

Nothing special there.. how about listening to events?

	//BEFORE
	val sub = new Subscriber[Int] {
		def receive(event: Int) = { /* handle event */ }
	}
	sub.subscribeTo(pub)

	//AFTER
	for(event <- events) { /* handle event */ }

Okay, that seems to be a bit more concise. Now how about these "combinators"?

	//BEFORE
	val pub2 = new Publisher[Int]
	val pubAdaptor = new Subscriber[Int] {
		def receive(event: Int) = {
			pub2.publish(event * 2)
		}
	}
	pubAdaptor.subscribeTo(pub)
	val sub = new Subscriber[Int] {
		def receive(event: Int) = { /* handle event */ }
	}
	sub.subscribeTo(pub)

	//AFTER
	for(event <- events.map(_ * 2)) { /* handle event */ }

Wow! That certainly saved a lot of space. In case it's not obvious, `pub2` is supposed to be a publisher that fires any event from `pub` after multiplying it by 2. To get that, I had to set up an intermediate subscriber that would tell `pub2` to fire the new event. In the "after" case, all I had to do was call `map(_ * 2)` on `events`, and all of that wiring was taken care of for me.

Assuming that the `EventSource[T]` class has all of the same combinators that you'd find on any other scala collection, you could presumably do something like...

	val transformedEvents = events.takeWhile(_ != 5).filter(_ % 2 == 0).map(List(_ / 2))
	val someOtherEvents = events2 collect {
		case "hello" => List(1,2,3)
		case "goodbye" => List(3,2,1)
	}
	val eitherEvent: transformedEvents.union(someOtherEvents)

There are a few things out there that already offer some of this functionality. C# has the "[Reactive Extensions](http://msdn.microsoft.com/en-us/data/gg577609.aspx)" for LINQ, and there are a few open-source libraries floating around for scala like [reactive-web](http://reactive-web.tk/). I took a stab at making my own implementation of this framework, which I call "Scala FRP." It's [hosted on Github](https://github.com/dylemma/scala.frp), and I've been making occasional additions to it since I got it to a stable state a couple weeks ago. The [docs](http://dylemma.github.com/scala.frp/) are pretty thorough, and you are free to look at the source code.

For further reading on the concept I described, [Deprecating the Observer Pattern](http://lampwww.epfl.ch/~imaier/pub/DeprecatingObserversTR2010.pdf) is where I first encountered it. The paper is a very difficult read, but is definitely worth at least looking at if you are interested in learning more.