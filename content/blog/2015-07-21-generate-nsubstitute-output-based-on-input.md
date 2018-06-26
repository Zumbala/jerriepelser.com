---
date: 2015-07-21T00:00:00Z
description: |
  NSubstitute is a mocking framework for .NET. One of the reasons I like NSubstitute is because you can generate the output for a mocked function call, based on the input parameters passed to the function.
tags:
- nsubstitute
- tdd
- unit tests
- unit testing
title: 'Reasons I like NSubstitute: Generate output based on input parameters'
url: /blog/generate-nsubstitute-output-based-on-input/
---

NSubstitute is my current mocking framework of choice. It contains all the features I require in a mocking framework and is well maintained. I previously mentioned as one of my [favourite Nuget packages for .NET](http://www.jerriepelser.com/blog/my-favourite-nuget-packages-for-dotnet) and I also showed how you can [mock abstract classes](http://www.jerriepelser.com/blog/mocking-abstract-class-using-nsubstitute). 

In this blog post I want to talk a little bit about one of the many reasons I like NSubstitute, namely the ability to generate the output of a mocked function call based on the parameters passed into the function. First though let's have a quick look at some basic examples of how you would mock a function call. 

The examples I use here are basically lifted "as is" from the [NSubstitute documentation](http://nsubstitute.github.io/help.html), so I claim no credit for it.

Let us suppose that I have a calculator interface called `ICalculator`:

``` csharp
public interface ICalculator
{
    int Add(int a, int b);
}
```

If another class takes a dependency on `ICalculator` you can easily mock `ICalculator` as follows:

``` csharp
var calculator = Substitute.For<ICalculator>();
``` 

I can also specify which values should be returned from the mock object when the `Add()` method is called. So let us say we want to return the value "3" when the values "1" and "2" are passed in for the `a` and `b` parameters respectively:

``` csharp
calculator.Add(1, 2).Returns(3)
```

You can also specify that it should return a specific value regardless of the parameters passed in:

``` csharp
calculator.Add(Arg.Any<int>(), Arg.Any<int>()).Returns(3)
```

This tells NSubstitute that for any integer value passed in for the `a` and `b` parameters it should always return the value of 3.

The situation may occur though where you want to mock an interface, but want the values returned from a specific method call to be correct. So let us say we want to mock `ICalculator` but the rest of our unit test depends on the return values of the mock object to be correct. As the documentation points out, this may not be best practice, but there are instances where this does make sense.  

Let us quickly look at the parameters the `Returns()` method takes:

![](/assets/images/generate-nsubstitute-output-based-on-input/returns-method-parameters.png)

In the previous examples I simply returned an integer value, but as you can see from the method overloads I can also pass in a `Func<>` which return an integer, but also takes in a paramter of type `CallInfo`. The `CallInfo` class gives us access to the parameters 

``` csharp
public class CallInfo
{
	...
	
	/// <summary>
	/// Gets the nth argument to this call.
	/// 
	/// </summary>
	/// <param name="index">Index of argument</param>
	/// <returns>
	/// The value of the argument at the given index
	/// </returns>
	public object this[int index]
	{
		get
		{
			...
		}
		set
		{
			...
		}
	}
	
	/// <summary>
	/// Get the arguments passed to this call.
	/// </summary>
	/// <returns>
	/// Array of all arguments passed to this call
	/// </returns>
	public object[] Args()
	{
		...
	}
	
	/// <summary>
	/// Gets the types of all the arguments passed to this call.
	/// </summary>
	/// <returns>
	/// Array of types of all arguments passed to this call
	/// </returns>
	public Type[] ArgTypes()
	{
		...
	}
	
	/// <summary>
	/// Gets the argument of type `T` passed to this call. This will throw if there are no arguments
	///             of this type, or if there is more than one matching argument.
	/// 
	/// </summary>
	/// <typeparam name="T">The type of the argument to retrieve</typeparam>
	/// <returns>
	/// The argument passed to the call, or throws if there is not exactly one argument of this type
	/// </returns>
	public T Arg<T>()
	{
		...
	}
	
	/// <summary>
	/// Gets the argument passed to this call at the specified position converted to type `T`.
	///             This will throw if there are no arguments, if the argument is out of range or if it
	///             cannot be converted to the specified type.
	/// 
	/// </summary>
	/// <typeparam name="T">The type of the argument to retrieve</typeparam><param name="position"/>
	/// <returns>
	/// The argument passed to the call, or throws if there is not exactly one argument of this type
	/// </returns>
	public T ArgAt<T>(int position)
	{
		...
	}

	...
}
``` 

Using the `ArgAt<T>(int position)` method we can easily get the values of the the integer parameters at positions 0 and 1 (i.e. the first and second parameters), add them up and return the correct value regardless of the parameters being passed in:

``` csharp
calculator.Add(Arg.Any<int>(), Arg.Any<int>())
    .Returns(callinfo => callinfo.ArgAt<int>(0) + callinfo.ArgAt<int>(1));
```

The above example is fairly simple, but one example where I have used this before was when I used AutoMapper in a ASP.NET Web API controller. I had a mocked database context (or repository) which may return a different number of rows based on certain filter criteria. 

Those rows are then passed to AutoMapper (via the `IMappingEngine` interface) to do the mapping. I am not concerned that the mapping operation returns the correctly mapped objects, but I am interested that it returns the correct **number of objects**.

Let us say I have a `Customer` class which is my Entity Framework class, and a `CustomerDTO` class which is what is being returned by my API call. Also let us assume that I have created an AutoMapper mapping that maps an `IEnumerable<Customer>` to an `IEnumerable<CustomerDTO>`. 

Using the `CallInfo` I can then inspect the number of `Customer` objects passed in to the `Map` method of my mapping engine and generate the corresponding number of `CustomerDTO` objects to return: 

```csharp
var mappingEngine = Substitute.For<IMappingEngine>();

mappingEngine.Map<IEnumerable<CustomerDTO>>(Arg.Any<IEnumerable<Customer>>())
    .Returns(callinfo =>
             {
                 var passedCustomers = callinfo.ArgAt<IEnumerable<Customer>>(0);

                 return Enumerable.Repeat(new CustomerDTO(), passedCustomers.Count());
             });
```

It took me a bit of searching to figure this one out, so I hope that this may help you in the future.
