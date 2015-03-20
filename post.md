In [Here Be Monsters](https://apps.facebook.com/herebemonsters/), we have a story-driven, episodic MMORPG that has over 5000 items and 1500 quests. It has more text than the first three Harry Potter books combined, so it represented a fairly sizable challenge when we decided to localize the whole game!

## The Challenge

From a technical point of view, the shear volume of words is of little consequence, although it is a significant cost concern - localization is a time-consuming and expensive process.

It’s the number of places that require localization that represents a **maintainability headache**.

With a conventional approach, the client application would consume a [gettext](http://en.wikipedia.org/wiki/Gettext) file containing all the translations. Anywhere it needs to display some text it’ll substitute the original text with the localized text instead.

We found a number of issues with this approach:
1. large number of files – Domain Objects/DTOs/game logic/view – need to change during implementation
2. all future changes need to take localization into account
3. need to replicate changes across client platforms (Flash, iOS, etc.)
4. hard to get good test coverage given the scope, especially across all client platforms
5. easy for regression to creep in during our frequent release cycles
6. complicates and lengthens testing and puts more pressure on already stretched QA resources

## Localize on Server

Instead, we decided to perform localization on the server as part of the pipeline that validates and publishes the data (quests, achievements, items, etc.) captured in our custom CMS. 

The publishing process first generates domain objects that are consumable by our game servers, then converts them to DTOs for the clients.

This approach partially addresses points 3 and 4 above as it centralizes the bulk of the localization work. But it still leaves plenty of unanswered questions, the most important was the question of how to implement a solution that is:
* **simple**
* **clean** – it shouldn’t convolute our codebase
* **maintainable** – it should be easy to modify/extend and hard to make mistakes even as we continue to evolve our codebase
* **scalable** – it should continue to work well as we add more languages and localized DTO types

### AOP via PostSharp

To answer these questions, we derived a simple and yet effective solution:

1. ingest the *gettext* translation file (the Nuget package [SecondLanguage](https://www.nuget.org/packages/SecondLanguage/) comes in handy here)
2. use a [PostSharp](https://www.postsharp.net/) attribute to intercept string property setters on DTOs to replace input string with the localized version
3. repeat for each language to generate a language specific version of the DTOs

[PostSharp](https://www.postsharp.net/) is an [Aspect-Oriented Programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming) (AOP) framework for .Net, similar to AspectJ for Java.

Whilst most of the discussion around AOP has been about eliminating [cross-cutting concerns](http://en.wikipedia.org/wiki/Cross-cutting_concern) such as logging and validation, you can achieve so much more with PostSharp. 

What PostSharp gives you is a well-defined way to inject behaviour at key points - property getter/setter, event invokation, method invokation, etc. The mindset in the PostSharp community has shifted towards [Automating Design Patterns](http://www.infoq.com/articles/Design-Pattern-Automation) in a displined and safe way.

### Localize attribute

Here is a simplified version of what our `Localize` attribute looks like:

```csharp
[Serializable]
[AttributeUsage(AttributeTargets.Class 
             | AttributeTargets.Assembly 
             | AttributeTargets.Field 
             | AttributeTargets.Property)]
[MulticastAttributeUsage(MulticastTargets.Property)]
public sealed class LocalizeAttribute : LocationInterceptionAspect
{
    public override bool CompileTimeValidate(LocationInfo locationInfo)
    {
        // ignore if not a string property
        return locationInfo.LocationKind == LocationKind.Property &&
               locationInfo.LocationType == typeof(string);
    }
 
    public override void OnSetValue(LocationInterceptionArgs args)
    {
        var str = args.Value as string;
        if (string.IsNullOrWhiteSpace(str))
        {
            args.ProceedSetValue();
            return;
        }
 
        // short-circuit using a custom 'context' that captures the 
        // intention to localize for a language
        var localizationCtx = LocalizationContext.GetCurrentContext();
        if (localizationCtx == null)
        {
            args.ProceedSetValue();
            return;
        }
 
        // again, use the custom localization context which contains 
        // information from the gettext file to localize the original string
        args.Value = localizationCtx.Translate(str);
        args.ProceedSetValue();
    }
} 
```

### One-liner localization!

To automatically apply localization to all present and future DTO types (assuming that all the DTO types are defined in one project), simply multicast the attribute and target all types that follows our naming convention:

`[assembly: Localize(AttributeTargetTypes = “*DTO”)]`

and voila, we have **localized over 95% of the game with one line of code**!

![](http://img3.wikia.nocookie.net/__cb20130410210519/glee/images/0/0c/HELL-YEAH.jpg)

and here’s an exam­ple of how an almanac page in the game looks in both Eng­lish and Brazil­ian Portuguese:

![](http://theburningmonk.com/WordPress/wp-content/uploads/2014/05/image.png)

![](http://theburningmonk.com/WordPress/wp-content/uploads/2014/05/image1.png)

## Conclusion

I hope you find this story of how we localized our MMORPG interesting, and the morale of the story is that there is much more to AOP than the same old examples you might have heard so many times before – logging, validation, etc.

With a powerful framework like PostSharp, you are able to do meta-programming on the .Net platform in a structured and disciplined way and tackle a whole range of problems that would otherwise be difficult to solve. To name a few that pops into mind:

* [Undo/Redo](http://www.postsharp.net/blog/post/New-in-PostSharp-32-UndoRedo)
* [String interning](http://theburningmonk.com/2013/02/aop-string-interning-with-postsharp-attribute/)
* [Caching](http://doc.postsharp.net/example-cache)
* [Auto-implement INotifyPropertyChanged](https://www.postsharp.net/model/inotifypropertychanged)
* [Auto add DataContract and DataMember attributes](http://doc.postsharp.net/example-dataattributes)
* [UI thread dispatching](https://www.postsharp.net/threading)
* [Performance monitoring](http://theburningmonk.com/2012/03/recording-for-my-webinar-with-postsharp/)
* Transaction handling
* [Backing property with a registry value](http://doc.postsharp.net/example-registryvalues)
* [Making an event asynchronous](http://doc.postsharp.net/example-asyncevent)
* [Raise event when object is Finalized](http://doc.postsharp.net/example-finalizedevent)
* [Dynamically introducing an interface](http://doc.postsharp.net/example-interface)

the list goes on, and many of these are available as part of the PostSharp patterns library too so you even get them out of the box.


## Links

[Read about how we automated game processing for *Here Be Monsters* using Neo4j](https://www.airpair.com/neo4j/posts/modelling-game-economy-with-neo4j)

[Read the Design Pattern Automation whitepaper on InfoQ](http://www.infoq.com/articles/Design-Pattern-Automation)

[Read about how else we're using PostSharp at Gamesys](http://theburningmonk.com.s3.amazonaws.com/Gamesys%20Case%20Study.pdf)
