---
layout: post
---
When using an API you must take care to understand not only what it returns, but also for how long the data returned will be valid. This is very important to consider because programs must make either be making decisions on valid and predictable data or have appropriate fallback mechanisms for failure.

I often find bugs because people assumed certain APIs were giving back data about the present or future, but in fact the were giving back information about the past or much shorter present. Part of the problem stems from the fact that API names are usually written in the present tense regardless of what point in time the data returned is valid. For example 'Exists' instead of 'Existed' or 'DidExist'. The language of the API lulls developers into a false sense of security and leads to programming errors.

I like to think of these values as a question of when they are valid. For instance, the data is, was or will be valid.

  * Will ' The data return is valid now and will be valid for the remainder of the program.
  * Is ' The data is valid now and will remain valid until the program or thread takes some action to alter the underlying data source. 
  * Was ' The data was valid at some point in the past but by the time the program receives the result it can no longer be relied upon to be valid. 

Working with APIs in the 'was' category are the trickiest. Because the data we are working with is volatile we must approach them in a different way in order to provide reliable programs. Most algorithms work by validating the present or future state of the system and then executing code with the expectation of success. When working with a 'was' API, there is no way validate the state of system because it cannot provide reliable information about its present or future state. Instead we must execute the algorithm with the expectation of failure and spend our time considering how to account for these failures appropriately. Failure must be the expected case, and not the exception.

Working with 'will' and 'is' APIs is a bit easier. Each has a level of predictability and it's possible to use program state to make reliable decisions and hence produce reliable results.

Lets add some substance to this conversation by considering these types of APIs and the ContainsKey method on various hashtable style collections. This API is often written in the present tense regardless of how long the data is actually valid.  

A 'will' API is the safest to work with because the data will always be valid.  This API cannot be influenced by side effects. Hence the the programmer is free to think only about the algorithm at hand. These APIs are rare and are typically associated with purely immutable data structures. This is the main type of structure for which data does not ever change. For instance consider this set of calls

    
``` csharp
var name = "bob";
ImmutableMap<string, Student> map = GetSomeImmutableMap();
if (map.ContainsKey(name)) {
    SomeOtherCall();
    var student = map[name];
}
```

This call to the indexer will succeed no matter what. The structure cannot be changed irrespective of what actually happens in SomeOtherCall. The ContainsKey method already asserted the key was present in the hashtable.  Because the structure is immutable this assertion will be valid for the remainder of the program and hence the indexer must succeed.

An 'is' API is the most common type of API. It is generally associated with mutable structures completely within the control of the program / thread. The data will remain valid until it is explicitly, or much worse implicitly, invalidated by the user. In a well factored program it is possible to reason about these types of APIs but hidden side effects can get you into trouble.  For instance consider the following code in the context of a single thread.

    
``` csharp
var name = "bob";
Dictionary<string, Student> map = GetSomeDictionary();
if (map.ContainsKey(name)) {
    Console.WriteLine(map[name]);
    SomeOtherCall();
    Console.WriteLine(map[name]);
}
```

The state of a Dictionary<TKey,TValue> structure is completely within the control of the program. As such the first call to Console.WriteLine will work fine. There is nothing which can alter the state of map between the call to ContainsKey and the actual accessing of the map in the first WriteLine call so the assertion is still valid.

However without knowing the hidden side effects of the SomeOtherCall API we cannot know the next call to WriteLine will succeed. It's possible for SomeOtherCall to access the underlying Dictionary reference and Clear it thus making the next call fail. True a programmer must investigate this before writing the above code. But consider the opposite problem. I am the maintainer of SomeOtherCall and I get a bug assigned to me which necessitates clearing out that Dictionary. I must now consider everyone who calls me, directly or indirectly and consider if any of them have an implicit dependency on the state of the underlying Dictionary object. This can get very difficult in a large program.

As previously stated, a 'was' API is the most dangerous type of API. They never give back data for which you can make a reliable decision about. These APIs are associated with a data source that is not completely in the control of the current thread or program. Thus it's possible for the data source to be altered at any point in the execution of the program / thread. Threading is a prime example of this problem.

For an example, consider SynchronizedDictionary to be an implementation of Dictionary<TKey,TValue> which uses locks internally to prevent data corruption from reads / writes on multiple threads but provides no other synchronization capabilities .

``` csharp
var name = "bob";
SynchronizedDictionary<string, Student> map = GetSomeSynchronizedDictionary();
if (map.ContainsKey(name)) {
    var student = map[name];
}
```

In this example it's possible for the call to map[name] to fail because another thread came through and cleared the collection in between the if statement and the call to map[name]. The bug here is that the program was written as if ContainsKey gave information about the present or future but in fact only gave information about the past. This actual return of the API could be made clearer by simply altering the name of the API. A much more applicable name is 'DidContainKey' or 'DidAndMayStillContainKey'. This name is much more representative of the data returned and will hopefully give a developer pause before writing code like the above.

My personal pet peeve in this category is the file system and in particular [File.Exists](http://msdn.microsoft.com/en-us/library/system.io.file.exists.aspx). Notice how this name is written in the present tense indicating a successful return means the file is currently in existence. Lets ignore permissions for a minute[^1] and consider only the files existence. Any program on the machine is at liberty to change the file system and can do so at any point in time. Even the user can affect the file system by doing some thing as simple as yanking out a USB thumb drive or stumbling over a network cable. Any of these operations can alter the state of the file system and hence invalidate the existence of a File with 0 action from your program. Hence File.Exists cannot ever reliably determine if a file exists, it can only determine that a file 'did' exist at some point in the past and may exist at some point in the future. A more representative name would be File.DidExist, or File.DidAndMayStillExist.

So when designing your APIs, take care to consider the lifetime of the data when naming them.

[^1]: Once you consider permissions you find that even File.DidExist in not a representative name. It probably should be called File.DidExistAndHadSomeMeasureOfAccess. Yes, that's a terrible name :(

