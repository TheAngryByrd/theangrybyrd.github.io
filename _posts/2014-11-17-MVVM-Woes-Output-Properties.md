---
layout: post
title: "MVVM Woes - Output properties"
published: true
---

## MVVM Woes - Output properties

In MVVM, I regularly have output properties, or properties that usually depend on one or more other properties within the viewmodel.  For example, say we have 2 input fields, one for first name and one for last name.  The output field would be a string.Format() of these two fields showing the full name.  

Simple VM:
{% highlight csharp %}
 public string FirstName
{
    get { return _firstName; }
    set
    {
        if (value == _firstName) return;
        _firstName = value;
        OnPropertyChanged();
    }
}
 
public string LastName
{
    get { return _lastName; }
    set
    {
        if (value == _lastName) return;
        _lastName = value;
        OnPropertyChanged();
    }
}
 
public string FullName
{
    get { return string.Format("Full Name: {0} {1}", FirstName, LastName); }
}
{% endhighlight %}
![Scenario 1]({{ site.url }}/images/MVVMOutput/Scenario1.jpg)

The problem here if you have had experience with MVVM before is fairly obvious.  I need to Notify the UI to update FullName.

{% gist TheAngryByrd/4feb42e5a8ee173e0f77/0b7a8c588f3d43038ad0bacb931cf62a9358c6a4 %}
![Scenario 1-Fixed]({{ site.url }}/images/MVVMOutput/Scenario1-Fixed.jpg)

For a more "exotic" example.  Let's say there is a third input for their favorite color.  Another output property depends now on the FullName and the Favorite Color.

{% gist TheAngryByrd/4feb42e5a8ee173e0f77/1ae4a61ed5e2c114946637c7d076520353f4fc65 %}
![Scenario 1-Fixed]({{ site.url }}/images/MVVMOutput/Scenario2.jpg)

Ok, sure let's just apply the same logic...not so fast.  FullName doesn't have a setter.  So I need to call OnPropertyChanged() in the things that it depends on.  

{% gist TheAngryByrd/4feb42e5a8ee173e0f77/4569b8dd45634359cbf169ece9f21f6614681221 %}
![Scenario 1-Fixed]({{ site.url }}/images/MVVMOutput/Scenario2-Fixed.jpg)

This code is starting to smell.  We have setters becoming full of notify the UI of changes for properties that aren't directly associated with them.  I've seen code with something like:
{% gist TheAngryByrd/4feb42e5a8ee173e0f77/e1ff07217f5b1f6a5cadab8d9b83ad355f74a2ba %}

Just because they (I) couldn't keep track of all the dependencies.

Well there **Is A Better Way**™.  [ReactiveUI](https://github.com/reactiveui/ReactiveUI) which uses [Reactive Extensions](https://github.com/Reactive-Extensions/Rx.NET) (Rx) allows us to create a pipeline effect for our notifications.

I'll show the complete viewmodel before we make any changes:
{% gist TheAngryByrd/4feb42e5a8ee173e0f77/d41f016e14afbbcc6e27f945bbef8452c0b5e0c8 %}

First, we'll get ReactiveUI from NuGet.  Then we'll replace our INotifyChanged with ReactiveObject.  That forces us to change the setters.  That's ok, there's a nice method for checking for update changed and raise all in one call.  this.RaiseAndSetIfChanged().

{% gist TheAngryByrd/4feb42e5a8ee173e0f77/74b80644b268247f820caefd4ca8bbbfa13627f5 %}

Ok, but now we're back to not notifying the FullName or Sentence properties.  Right, we need to talk about some more ReactiveUI first.  Specifically, WhenAnyValue, ObservableAsPropertyHelper and ToProperty.  

WhenAnyValue allows us to get notified when a property changes.  
{% highlight csharp %}
var fullName = this.WhenAnyValue(x => x.FirstName, x => x.LastName, (first, last) => new {first,last})
{% endhighlight %}
Will let us know whenever there are changes to either FirstName or LastName and create a new object that contains both.

Now ObservableAsPropertyHelper and ToProperty go hand in hand. ObservableAsPropertyHelper is an output property in ReactiveUI. ToProperty allows us to set this property.

{% gist TheAngryByrd/4feb42e5a8ee173e0f77/b71729a8122927835ba25f590b1d489238eada9e %}

Now we can see the train of how FullName will get it's value.  Anytime FirstName or LastName update, select a string.Format() of them and update the fullname property.

Ok, well lets fix Sentence:

{% gist TheAngryByrd/4feb42e5a8ee173e0f77/8dce702a6079113be286460f4645e0118e95c17b %}

Now we see the true intent.  Anytime FullName or FavoriteColor is updated, we should change the SentenceProperty.

Full end gist:
{% gist TheAngryByrd/4feb42e5a8ee173e0f77/859288ab239106d13af1663277b32d67f87a6cec %}

[Source code](https://github.com/TheAngryByrd/MVVM-Output-Properties/)







