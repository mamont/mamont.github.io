---
layout: post
title:  "Thoughts on C++ async API"
date:   2017-10-14 12:40:42 +0200
categories: c++ async
---

## Preface

Nowadays everything needs to be asynchronous. We all know that. No one suddenly believes that having a dedicated thread for every task is a good idea anymore.
And developing C++ library every programmer has to solve the problem: how to provide library user good way to track and control application state

## Good old times

Classical C++ way to handle asynchronous operation is to provide an object which implements some listener interface.
This approach works well for years and generations of C++ (and other languages) programmers. 
But downsides ot this method are obvious after first glance at the code: it's extremely wordy:
For every possible listener customer of your library should create class implementing listener interface and deal with it's lifetime.
If your class has a several listeners, you're doomed.


Let's consider communication library, which creates persistent connection and allows user to send and receive messages.
So we need (at least) three listeners:
- connection state listener
- incoming message listener
- outgoing message listener


Which means that "classical" flavor of this interface looks like that

{% highlight c++ %}

/**
 * Listener to track message state
 */
class IMessageStateListener 
{
public:
    virtual void onMessageSent() = 0;
    virtual void onMessageFailed() = 0;
    virtual void onMessageCancelled() = 0;
};

/**
 * Library itself
 */
class ILibrary
{
pubic:
    /**
     * Every call of sendMessage starts an async operation, so it needs dedicated listener instance. 
     * Also it needs a possibilty to cancel this operation
     */
    CancellationToken sendMessage(Message message, std::shared_ptr<IMessageStateListener> messageStateListener);   
    void cancelMessage(CancellationToken canellationToken);
};


{% endhighlight %}


It looks not that bad, and quite easy to implement for library developer. But it involves quite a lot of work from library user.
And this is what we need to fix in order to provide easier interface and free up application developer for some actual work, not countless listeners and event routing.


## JavaScript fanboy enters

At this point I have to confess. During last two years I was dealing a lot with javascript. I know, this language is not really well respected in C++ community, but it is all about asynchronousity. So it definitely has something to learn.
And nowadays the most adopted approach of handling asynchronous events in JS is Promise. Yes, they have added async/await, and it is really awesome, but let's keep this theme for another blog post.
So when I got to the problem of building robuts async interface, I evaluated options, and decided to go the way of promises, as most straightforward and easy to implement option.

Here the question arises: we do have std::promise right now. What is the problem then? 
Unfortunately, std::promise/future are lacking few features:
- continuations and chaining. Some of these already are in experimental, but you don't want to expose "experimental" in production api, do you?
- it lacks proper control over thread model which library uses to resolve these promises (can be solved)
- it doesn't support cancellation


So yes, I did it. I wrote my own promises library. 
Though I'm not selling this library here. This article is more about approach, not the particular implementation.

At this point let's see how our interface may look like using promises: 

{% highlight c++ %}

class ILibrary
{
public:
    cancellable_future<Result> sendMessage(Message message);

};

{% endhighlight %}


This interface is quite short and self-exlanatory. But this is not only advantage.
We also do not need to track observer lifetime, we do not need to store canellation token, we don't need to pass cancellation token and method which consumes it to every place where we may want to trigger a cancellation.

{% highlight c++ %}

//Creating a future instance
auto future = libraryInstance
    .sendMessage({ "What a lovely day " })
    .then([](auto response) {
        if (response.code == 200) {
            std::cout << "Message has been delivered: " << response.code << std::endl;
        } else {
            std::cout << "Message has not been delivered: " << response.code << std::endl;
        }
    });
{% endhighlight %}

And remember, you now have a uniformed interface to cancel async operation:
{% highlight c++ %}
// And remember, you can do cancel everywhere!
future.cancel();
{% endhighlight %}


Also, promises are chainable:

{% highlight c++ %}
auto future = libraryInstance
    .sendMessage({ "What a lovely day " })
    .then([](auto response) {
        if (response.code == 200) {
            std::cout << "Message has been delivered: " << response.code << std::endl;
        } else {
            std::cout << "Message has not been delivered: " << response.code << std::endl;
        }
    })
    .then([]() {
        std::cout << "I'm a chained method!" << std::endl;
        promise<int> p;
        // and make sure this promise will be resolved somewhere else

        return p.get_cancellable_future([]() {}); // I'm to lazy to write cancellaton here
    })
    .then([](int answer) {
        std::cout << "I'm asynchronously chained and answer is " << answer << std::endl;
    });

{% endhighlight %}

## Thread model

What is the different in javascript and c++ runtimes? Everything! But one of the most important things - c++ programs usually have multiple threads.
Which means that unlike JS when you just put whatever you want in "then" section, you need to care about non-blocking behaviour, thread safety, etc.
This is the thing which we should care about, since we have a continuation. Imagine, library user has long operation inside the continuation, and we're calling it from some core library thread.
Such situation would cause whole library to stuck until continuation execution finished. 
This is not a kind of behaviour.

Most straightforward approach would be to resolve promise (call set_value in case of c++ promise) in a thread which you want this callback to be called from. This means manual rescheduling the event on some dedicated thread every time we need to resolve the promise.
That is quite possible, but again, involves some manual work inside the library. At least we need to have this dedicated thread, easy way to reschedule event, etc.
Sounds boring.

One of the options is to bundle this functionality inside the promises library.
Futures Java 8 has something like that. In every "thenContinueAsync" or smth like that user can pass a thread pool, which is going to be used to execute continuation method.
This definetely should work for c++ also.
Though it means that it's a library user who has control over thread model. That's also good thing, but not good enough. 
Let's modify this approach a bit. We can still support passing the thread pool to every "then" call, but we also need a possibility to pass it to promise constructor. Or "get_future" method.


{% highlight c++ %}

promise<void> p;
auto future = p.get_future(executor);
future.then([]() {
    std::cout << "Supplied executor decided which thread I'm executed on!" << std::endl;
});

{% endhighlight %}

Where executor can be really anything.
For example, we can implement "synchronous executor", which just calls the continuaton on the same thread
We can implement "single thread executor", which guarantees that all callback are being schedulet on the same thread
And we can implement "multiple thread executor", which just takes threads from threadpool

This method provides enough flexibility and helps to concentrate the knowledge about thread model at the place where promise is created.


## Real life experience

This was a real life problem I had refactoring a c++ library. 
What can I say: 
- using such approach I got much simplier interface
- users of library were happy enough since they don't need to create a gazillion of listeners and track them manually
- continuation got simpler
- thread model became much more uniform

So, don't hesitate, use promises in c++ and be happy!

## Alternatives

There are few other options to facilitate asynchronous operations:
- RxCpp and Observables
- Coroutines
- Probably much more options I'm not aware of

I won't pretend I've tried them all.
Coroutines and async await looked too much pain to build it in to the existing application. Probably works better when you writing app from scratch. 
Observables are cool for series of events. E.g. it, probably, would be better option for incoming messages. 

So promises is simple and robust solution. As every tool it shouldn't be considered as silver bullet. It won't solve all your problems. But if you use it with care, it is extremely useful.
