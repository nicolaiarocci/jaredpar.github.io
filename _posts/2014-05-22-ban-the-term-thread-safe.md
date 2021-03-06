---
layout: post
title: We should ban the phrase "thread safe"
---

There few are phrases in programming that make me shudder more than 

> This type is thread safe

Reading this phrase is like hearing nails on a chalk board.  It often makes me physically cringe. 

The phrase *thread safe* makes it sound like thread safety is on / off property of a type.  Nothing could be further from the truth.  There is a wide variety of multi-threaded usage scenarios for types that simply cannot be decribed in a binary fashion.  Describing it in such a way serves to do nothing other than confuse users.  

A much better way to describe the thread safety of a type is to enumerate the scenarios in which the type can or can't be used.  Giving users concrete information on how the type will behave in specific circumstances gives them a much better chance of correctly using the type in their own programs.  

Even better once the scenarios are enumerated you'll often find these types fall into exitsing well known patterns which the user is more likely to have experience with.  This gives them a much better chance of succesfully using the type    

Here is a non-exhausting sampling of the most common multithreaded patterns I  encounter in my day to day programming

Data Race Free
===
A type which is data race free will never corrupt its internal state no matter how many threads are reading from and writing to it.  Sharing such a type between threads is safe in that the internal data structures will never be corrupted.  A data race free list for example will never have a count different than the actual number of elements stored inside it. 

Examples of data race free types are this include Java vectors, .Net 1.0 [synchronized collections](http://msdn.microsoft.com/en-us/library/3azh197k(v=vs.110).aspx), and .Net 4.0 [concurrent collections](http://msdn.microsoft.com/en-us/library/dd381779(v=vs.110).aspx).  

Such a type is still fraught with usability problems though.  For example here is an incorrect sample using a synchronized version of [ArrayList](http://msdn.microsoft.com/en-us/library/vstudio/system.collections.arraylist)

``` csharp
static object GetFirstOrDefault(ArrayList synchronizedList) { 
  if (synchronizedList.Count > 0) {
    return synchronizedList[0];
  }

  return null;
}
```

None of the individual statements here are an incorrect usage of synchronizedList yet the function itself is wrong.  It's possible for any number of threads to execute in between the initial `if` block and the `return` statement.  Any of which could clear the list thus causing an execption on the `return` statement [^1]. 

Multiple Reader, Single Writer
===
This class of types can be safely read from mulitple threads or written to by any single thread.  However these operations cannot be done at the same time or the data structure will possibly corrupt its internal state.  

This pattern allows for useful parallel operations that simply need to read from a type.  For example projecting the contents of one collection to a new one.  

In my experience this is the most common pattern for types because it is the natural default.  There is simply no danger in reading the same memory from multiple threads so long as it is consistent for the duration of the reads.  This is how properties and query methods naturally behave.  Unless a type goes out of its way to silently mutate on reads or has a particular thread affinity it will function in this manner 

Thread Affinity
===
Thread affinitized types can only be safely accessed from a specific thread in the program.  Any attempt to access instances of the type from code executing on a different thread is an error.  Such code must somehow transfer execution to the correct thread in order to safely inspect the object.

This pattern is probably most common in UI frameworks.  It is in fact the source of one of the most frequently asked questions on [stackoverflow](http://stackoverflow.com).  

> Why does an exception get thrown when I change a property on my control? 

``` csharp
var worker = new BackgroundWorker();
worker.DoWork += DoWork;

private void DoWork(object sender, DoWorkEventArgs e) {
  /* background caclulation */
  _theLabel.Text = theResult;
}
```

All WinForm controls are affinitized to the UI thread.  Any attempt to modify them from another thread will be met with an exception.  In this case the BackgroundWorker callback executes on a different thread and hence this is a violation of the types threading contract.  

Lucikly Winform types are one of the few types that enforce their thread safety contract at runtime. 

Not Even Read Safe 
===
The ability to safely read from a type on multiple threads depends on the read operations either a) being non-mutating or b) having internal, and potentially costly, synchronization around the mutations.  Types which do neither of these simply can't be accessed in this manner.  

This may seem rather strange but it's actually something the majority of C# users consume every day: LINQ queries.  An IEnumerable<T> produced from a LINQ query aren't actually evualuated until they are read from.  Queries like Distinct must use a table in order to filter out previously returned values.  Hence every read operation is reading from and potentially writing to the table.  This mutation disqualifies it from being succesful queried from multiple threads 

Less well known but particularily interesing are types like [Splay Trees](http://en.wikipedia.org/wiki/Splay_tree).  This binary search tree which optimizes lookups for recently accessed nodes.  It does this by mutating the tree to push recently accessed nodes closer to the root on read operations (mutation on read).  

Immutable Types
===
This is the one class of types to which *thread safe* actually can be validly applied.  A type which never changes satisfies any rational expectation a developer has for safe use amongst multiple threads.  Seemingly random operations like thread ordering have no effect on the outcome of operations on an immutalbe type

But if you have an immutable type calling it as such is far more descriptive than *thread safe*.  It's a rock solid guarantee that exists through the lifetime of the program.  Don't bother with ambiguous terminology when far more descriptive alternatives exist. 

This wide variety of scenarios is why the term *thread safe* simply needs to be banned.  It is an ambiguous description of a type that serves no purpose other than to cause confusion.    

Don't ever let another program get away with saying *thread safe*.  Call out this ambiguous description of a type and ask them to enumerate the specific threading scenarios for the type.  And most importantly, once they enumerate the details make sure it is documented in the type itself and not just an email.

[^1]: For the curious I already covered this problem in detail in a [previous post]({% post_url 2009-02-11-why-are-thread-safe-collections-so-hard %}) 
