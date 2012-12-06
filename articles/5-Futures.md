I want to talk about [Futures](http://en.wikipedia.org/wiki/Futures_and_promises), because they really come in handy when you need to do any kind of asynchronous operations in your programs.

On a high level, using Futures is jsut a different style from using callbacks. For example, in Javascript the general way to handle asynchronous calls is something along the lines of

    doSomethingAsync(function(result, err){
      if(err) {...} //handle error
      else {...} //handle result
    })

With futures' style, the same operation would look like

    doSomethingAsync
      .onSuccess(function(result){...})
      .onError(function(result){...})

Now this by itself isn't anything particularly special, but it's the new [advancements for Futures and Promises in Scala 2.10](http://docs.scala-lang.org/sips/pending/futures-promises.html) that got me interested in them.

###Composability

The big thing that I want to highlight is *composability*. Composability in terms of Futures is that you can put them together like little building blocks in a way that makes things much easier. 

Here's an example of how composable futures will make life easier. Let's say I want to make an HTTP request to some URL, and the response will be another URL to which I will need to make a new request.

Let's see what that looks like without composability (Javascript-style).

	function handleError(err){...}
	function handleSuccess(result){...}

	makeHttpRequest(url, function(response, err){
		if(err){
			handleError(err)
			return
		} else {
			var nextUrl = ... //from the response
			makeHttpRequest(nextUrl, function(resp2, err2){
				if(err2){
					handleError(err)
					return
				} else {
					handleSuccess(resp2)
				}
			})
		}
	})

Wow is that ugly! It gets nested four tabs deep, and the error handling needs to get pushed in at every step. Now let's see a composable version of the same code, as written in Scala.

	//assume that our http request function returns a Future
	def makeHttpRequest(url: String): Future[Response]

	val result = for{
		resp <- makeHttpRequest(url)
		resp2 <- makeHttpRequest(getUrl(resp))
	} yield resp2

	result onComplete {
		case Failure(exception) => handleFailure(exception)
		case Success(result) => handleResult(result)
	}

This looks much cleaner to me! Chaining the calculations together is done in 4 lines, and is separate from the handling of the results. Thanks to Scala's syntax sugar for `for-comprehensions`, you could chain a whole bunch of asynchronous calculations together without needing to indent any further than one tab.

Another nice thing is that errors are automatically "trickled down", so that if the first call to `makeHttpRequest` throws an exception, the second call will never happen, and that exception will be wrapped in a `Failure` to be handled in the `onComplete` block.

So to summarize, using Futures helps you handle chaining of operations while managing exceptions.

###Implementing Code with Futures

Scala gives you two convenient ways to create a Future. The easiest way is to use

	import scala.concurrent.Future
	import scala.concurrent.ExecutionContext.Implicits.global

	val futureResult = Future {
		//Do whatever stuff you need to be asynchronous here.
		//Any thrown exceptions will be wrapped in the result
	}

Scala will use the `global` `ExecutionContext` to run the body of your `Future{...}` on another thread, and manage its lifecycle. Any exceptions thrown within the Future's body will be wrapped up in the Future's `result`.

The other way to make a Future is to use a `Promise`. Promises are like the 'write' side of the Future's 'read'. You can use Promises to convert callback-style code into futures-style, or you could use them in classes to handle variables that get initialized late.

	//converting callback-style code
	def makeHttpRequest(url: String): Future[Response] = {
		val p = Promise[Response]()
		makeHttpRequestWithCallback(url,
			(result) => p.success(result), //success callback
			(e) => p.failure(e) //error callback
		)
		p.future
	}

As you can see, the `Promise` can be completed in either a 'success' or 'failure' state, corresponding to results and exceptions. Another important detail is that it can only be completed *once*. Fortunately, Scala 2.10's `Promise` API gives you a `trySuccess` and `tryFailure` that will simply return `false` instead of throwing an exception if you use it more than once.

	class A {
		private val p = Promise[X]()
		def x: Future[X] = p.future

		def prepare: Unit = p.trySuccess(someXValue)
	}

	val a = new A
	a.x onSuccess {
		case xValue => println("got the x value")
	}

	a.prepare
	// this causes the `a.x` to complete, triggering the onSuccess callback
	//prints 'got the x value'

There's a whole bunch of convenience methods that you can use with Futures and Promises. So check them out in the Scala 2.10 API.