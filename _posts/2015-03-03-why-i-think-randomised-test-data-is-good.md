---
layout: post
title: "Why I Think Randomised Test Data Is Good"
modified:
categories:
excerpt:
tags: ["TDD","unit testing","testing","randomised unit test data","random","randomised input data","random unit tests"]
image:
  feature:
date: 2015-03-03T09:05:42+00:00
comments: true
---

So far when I have brought up the idea of generating random input data for unit tests, there have been two different reactions: confusion and horror.

The confused have never heard of this technique whilst the horrified have seen this technique used and have only bad things to say about it. 

I am writing this blog, to hopefully target both of these developers and explain why I think randomised test input dataDogBreed is a good thing and how it can be beneficial to the TDD  approach. [TL;DR](#tldr)

##What is randomised test input data?

Utilising test driven development in order to write your code is undoubtably a great thing. Sometimes when writing software we need to check some output of our system under test versus an input from a known set. For the examples in this article I will use dog breeds. 
{% highlight java %}
public interface Dog {
	DogBreed getBreed();
	String getName();
}

@Test
public static void testLabradorGetBreedIsCorrect() throws Exception{
	Dog colin = new Labrador(“Colin”);
	assertEqual(colin.getBreed(),DogBreed.LABRADOR);
}
{% endhighlight %}
In this case we have a finite set (DogBreed) and we have initialised out Labrador object which is a Dog and we know the breed type must be in the DogBreed set. For cases such as this standard TDD techniques are fine.

However when we do not have an infinite(ish) sized set then we run into an interesting area when writing tests to *drive* our code. Taking our Labrador example from above:
{% highlight java %}
@Test
public static void testLabradorGetNameReturnsColin() throws Exception{
	String name = “Colin”
	Dog colin = new Labrador(name);
	assertThat(colin.getName(),equalTo(name));
}
{% endhighlight %}
From this test I am *driven* to write the following code:
{% highlight java %}
public class Labrador implements Dog {
	public Labrador(String name){}

	@Override
	public String getName(){
		return “Colin”;
	}
	// getBreed left out for sake of brevity
}
{% endhighlight %}
This is great as our tests are green and all is good. Unless we want a second labrador with another name...

Now there are two possible solutions to this problem (excluding randomised test input).


### Refactoring

TDD a has 3 steps: Red, Green and Refactor. We have been Red (when we wrote the test and ran it). We have been green, now that we are returning “Colin” meaning we can now refactor. So if we set name as a field property and return it from getName, job's a good'un and everything is still green after we re-run the tests. 

### Triangulation

If we want our tests to drive our code, then let's simply write another test to force us to return the property we passed in.
{% highlight java %}
@Test
public static void testLabradorGetNameReturnsColin() throws Exception{
	String name = “Colin”;
	Dog colin = new Labrador(name);
	assertThat(colin.getName(),equalTo(name));
}

@Test
public static void testLabradorGetNameReturnsFrank() throws Exception{
	String name = “Frank”;
	Dog colin = new Labrador(name);
	assertThat(colin.getName(),equalTo(name));
}
{% endhighlight %}
Okay now we're talking. The tests are forcing us to write the code for dynamic names. However we now have a new problem. We have two tests which are essentially doing the same work but with different inputs. We also have a larger test suite which will take longer to run (though in this case it is negligible). So following DRY we can then delete one of the tests. This way we have ended up in the same place as refactoring, but with a step in-between.

### The alternative 

The alternative to these solutions is to generate a random name to input into our test. This means that instead of writing our first test against a dog called “Colin” we could do the following:
{% highlight java %}
@Test
public static void testLabradorGetNameReturnsCorrectName() throws Exception{
	String name = generateName();
	Dog colin = new Labrador(name);
	assertThat(colin.getName(),equalTo(name));
}
{% endhighlight %}
In this instance, by generating a randomised name, we have short-circuited the former approaches. As this test will generate a new name for each run, we are *driven* to write the correct code the first time round.
{% highlight java %}
public class Labrador implements Dog {
	final private String name;
	public Labrador(String name){
		this.name = name;
	}

	@Override
	public String getName(){
		return name;
	}
	// getBreed left out for sake of brevity
}
{% endhighlight %}

## The benefits of Randomised test input data

For me, the major benefit of randomised test input data is this ability to short circuit the triangulation  phase when writing code. However this approach also brings another major benefit, lets imagine for a second the following scenario:

You are working on a multi-phased software project where in the first iteration, you only care about 1 specific type of dog. So in your code which serializes the Dog object to be sent on to another system you simply hardcode the dog type to “Labrador”.

You might write a test which looks like the following:
{% highlight java %}
@Test
public static void testDogBreederializesWithBreed() throws Exception{
	Dog colin = new Dog(”Colin”, DogBreed.LABRADOR);
	XmlDogSerializer serializer = new XmlDogSerializer();
	assertThat(serializer.serialize(colin), containsString(<breed>Labrador</breed>));
}
{% endhighlight %}
This works just fine for the first iteration. 

Now, lets say in the second iteration we want to handle any of the breeds from DogBreed. If we were to simply refactor, or triangulate and delete to get to this code then we would end up with not much change to our test code, and a serialize which serializes from the dog object.

However the subtle problem with this test it that it is not *enforcing* your code design. Lets say you commit the code and everything is fine. But one of your colleagues has a merge conflict, and somewhere in the merge process your colleague doesn't pull in your changes to the serializer. All the tests pass and they are none the wiser, the code could potentially ship if it is not caught downstream.


## It's not all sunshine and rainbows

One of the main arguments that I have come across with randomised test input is that tests can pass or fail randomly and are not reproducible. 

To these arguments, I would say that the reason test pass or fail randomly is not due to randomised test input, but rather that a flaw in the production code has been highlighted due to the randomness of the input data, or that the test has some incorrect presumptions about how the system should work.

If a test is randomly passing or failing when using randomised test input data, then it is likely that the generation technique is either generating data which is invalid based on your assumptions (e.g. in our dog example, it may be generating dog names which are not in the DogBreed enum) or the code is not able to handle the data (e.g. a function which falls over when a randomised value is negative is indicative of a problem in the production code).

On the note of a test not being reproducible, I would argue that this can come down to poor assertions which do not highlight the problems. In the examples above where randomness was used the assertions used should show the expected vs. actual values also [see the next section](#don-t-randomise-all-the-things) about where it is appropriate to randomise.

In cases where good assertions are not possible, then you should use a seeded random value, where the seed can be logged or output with the test value. This ensures if a failure occurs, you can rerun the test with the given seed and this ensures the tests are repeatable.

## Don't randomise all the things

I am not arguing that every unit test should generate random input data. Sometimes generating random data can make a test's intention less clear or in some cases specific examples may be more apt at showing intentions. 

Take for example a method which validates a dog has an age >= 0 and less than 15.

Using randomised test input you might write your tests in the following way:
{% highlight java %}
@Test
public static void testDogIsValidWhenAgeIsBetweenZeroAndFifteen() throws Exception{
	int age = generateRandomAgeBetween(0,15);
	String name = generateRandomName();
	DogBreed breed = generateRandomDogBreed();
	Dog dog = new Dog(name,breed,age);
	assertThat(dogValidators.valid(dog),isTrue());
}

@Test
public static void testDogIsnotValidWhenAgeIsNotBetweenZeroAndFifteen() throws Exception{
	int age = generateRandomAgeOutsideOf(0,15);
	String name = generateRandomName();
	DogBreed breed = generateRandomDogBreed();
	Dog dog = new Dog(name,breed,age);
	assertThat(dogValidators.valid(dog),isFalse());
}
{% endhighlight %}
This test has too much randomness. This test could potentially fail for any reason, e.g. the name if invalid, or the breed is an incorrect type. Here we should fix all the values we are not interested in for this test so we know what is the reason for the failure. This can be seen as the rule “a unit test should test one thing”.

Another problem with this test is that this could potentially fail randomly if the code has not been written correctly. Lets say for example the developer on this story accidentally used > 0 to check the dog age is valid. This test may pass several times before anyone knows there is a fault in the code (although they will eventually know).
 
In this particular case not using randomisation here could be more effective:
{% highlight java %}
private static String name = “Colin”;
private static DogBreed breed = DogBreed.Labrador;

@Test
public static void testDogIsNotValidWhenAgeIsLessThanZero() throws Exception{
	int age = -1;
	Dog dog = new Dog(name,breed,age);
	assertThat(dogValidators.valid(dog),isFalse());
}

@Test
public static void testDogIsNotValidWhenAgeIsFifteen() throws Exception{
	int age = 15;
	Dog dog = new Dog(name,breed,age);
	assertThat(dogValidators.valid(dog),isFalse());
}


@Test
public static void testDogIsValidWhenAgeIsZero throws Exception{
	int age = 0;
	Dog dog = new Dog(name,breed,age);
	assertThat(dogValidators.valid(dog),isTrue());
}

@Test
public static void testDogIsValidWhenAgeIsFourteen throws Exception{
	int age = 14;
	Dog dog = new Dog(name,breed,age);
	assertThat(dogValidators.valid(dog),isTrue());
}
{% endhighlight %}
Here no randomisation was used, and I think the intention of this test suite is far more clear and well defined.

##TL;DR

In conclusion I believe randomised input data is beneficial to the TDD cycle, because it allows the developer to *drive* their software in the correct direction and also *enforces* the code to stay that way until a change is required.

Randomisation is not always the best solution however as it can create more brittle tests when used in the wrong way. It should be used as and when needed, and should not be the go-to for every solution. I believe refactoring towards generic tests using randomised test input can be a happy medium which should prevent brittle tests whilst still utilising non-random input data for where it is needed.