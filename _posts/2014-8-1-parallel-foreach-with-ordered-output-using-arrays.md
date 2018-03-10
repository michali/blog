---
layout: post
title:  "Parallel.Foreach with ordered output using arrays"
date:   2014-8-1 00:00
categories:
- Task Parallel Library
excerpt: "One of the problems I've recently had was to process a collection of data and return the outcome of each processing in another collection. To utilise the multiple and/or hyper threaded cores of the machine that executes the processing, I've used the Task Parallel Library."
---
One of the problems I've recently had was to process a collection of data and return the outcome of each processing in another collection. To utilise the multiple and/or hyper threaded cores of the machine that executes the processing, I've used the Task Parallel Library.

The problem is that the output collection had to be ordered, i.e. each element of the collection had to correspond to the same order as the element in the input collection that it came from. A scenario for this is where we need to run a batch operation and need to match the output elements to each input element, such as updating multiple prices in a catalogue of products. One other way of matching input and output would be to assign identifiers to each element but I've decided against adding two temporary fields for an operation that could just be resolved with ordering.

Prototype:

```csharp
public class ParallelNumberIncrementor
{
   internal int[] Increment(int[] input)
   {
      int[] output = new int[input.Length];

      Parallel.ForEach(input, (i, state, index) =>
      {
         output[index] = i + 1;
      });

      return output;
   }
}
```

That's what it is in its simplest form. Arrays are not thread safe but in this instance a thread accesses an array position only once. We're using the overload of ```Parallel.ForEach``` that gives us access to the index of the element being processed in each iteration.

In order to provide live documentation and also to test that our logic does what it's meant to do, we can unit test this. We should remove the Parallel static from the ```ParallelNumberIncrementor``` because we're not testing the Task Parallel Library but only our own logic. So let's isolate the TPL functionality.

```csharp
public interface IParallelLooper
{
   ParallelLoopResult ForEach<TSource>(IEnumerable<TSource> source, Action<TSource, ParallelLoopState, long> body);
}
```

This interface copies the Parallel.ForEach method signature for this example. You can add its overloads into it as per your needs.

The implementing class:

```csharp
public class ParallelLooper
{
   ParallelLoopResult ForEach<TSource>(IEnumerable<TSource> source, Action<TSource, ParallelLoopState, long> body)
   {
       return Parallel.ForEach(source, body);
   }
}
```

So now let's implement a ```ParallelNumberIncrementor``` which is decoupled from the TPL. The logic to test is that even though each iteration can finish at different times on each execution of the loop, the end array will have its elements in the same order as the corresponding elements of the input array.

Let's start by writing our first test for this.

```csharp
[Test]
public void Increment_WhenElementProcessingFinishesInOrder_OutputElementsAppearInOrder()
{
     var parallelNumberIncrementor = new ParallelNumberIncrementor();
     var input = new[]{ 1, 2, 3 };
     var output = parallelNumberIncrementor.Increment(input);

     Assert.AreEqual(new[] { 2, 3, 4 }, output);
}
```

Getting it to pass:

```csharp
public IEnumerable<int> Increment(int[] input)
{
    foreach (var i in input)
       yield return i + 1;
}
```

Ok, nothing fancy here. Also, we want the incrementing operations to happen in parallel iterations and this isn't happening yet. If we could get an IParallelLooper involved that will make a case for parallelism.

Let's re-think the ParallelNumberIncrementor:

```csharp
public class ParallelNumberIncrementor
{
   private readonly IParallelLooper _parallelLooper;

   public ParallelNumberIncrementor(IParallelLooper parallelLooper)
   {
      _parallelLooper = parallelLooper
   }

   internal int[] Increment(int[] input)
   {
      return new int[0];
   }
}
```

The skeleton is in place. This will build but will fail all the tests as it's doing the minimal thing that's required just to build (I prefer to not use null objects in place of collections but this is another topic altogether). We can delegate parallelism to the ParallelLooper and deal with the logic that outputs an array in the desired order. As our unit tests have to be deterministic, i.e. take the system under test from a precondition to post-assertion consistently with each run, we can fake the order the output elements are finished being produced. Then it will be the ```ParallelLooper```'s responsibility to reorder these elements.

Let's rewrite the first test but with a ```ParallelLooper```.

```csharp
[Test]
public void Increment_WhenElementProcessingFinishesInOrder_OutputElementsAppearInOrder()
{
   var parallelLooper = new Mock<IParallelLooper>();
   parallelLooper.Setup(x => x.ForEach(It.IsAny<IEnumerable<int>>(), It.IsAny<Action<int, ParallelLoopState, long>>()))
       .Callback<IEnumerable<int>, Action<int, ParallelLoopState, long>>
       ((numbers, loopBody) =>
       {
          loopBody.Invoke(numbers.ElementAt(0), null, 1);
          loopBody.Invoke(numbers.ElementAt(1), null, 2);
          loopBody.Invoke(numbers.ElementAt(2), null, 3);
       });
   var parallelNumberIncrementor = new ParallelNumberIncrementor(parallelLooper.Object);
   var input = new[]{ 1, 2, 3 };

   var output = parallelNumberIncrementor.Increment(input);

   Assert.AreEqual(new[] { 2, 3, 4 }, output);
}
```

We can mock an ```IParallelLooper``` to return the output elements in reverse order and the ParallelNumberIncrementor will be responsible outputting the correct order.

We can have a design in mind just not be married to it.

```csharp
public class ParallelNumberIncrementor
{
   private readonly IParallelLooper _parallelLooper;

   public ParallelNumberIncrementor(IParallelLooper parallelLooper)
   {
      _parallelLooper = parallelLooper
   }

   internal int[] Increment(int[] input)
   {
      int[] output = new int[input.Length];

      _parallelLooper.ForEach(input, (i, state, index) =>
      {
         output[index] = i + 1;
      });

      return output;
   }
}
```