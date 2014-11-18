---
layout: post
title: "MVVM Woes - Output properties"
published: true
---

## MVVM Woes - Output properties

In MVVM, I regularly have output properties, or properties that usually depend on one or more other properties within the viewmodel.  For example, say we have two _input_ properties, 'FirstName' and 'LastName'.  The output field would be a string.Format() of these two fields showing the full name.  

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

The problem here, if you have had experience with MVVM before, is fairly obvious.  I need to Notify the UI to update FullName.

{% highlight csharp %}
 public string FirstName
{
    get { return _firstName; }
    set
    {
        if (value == _firstName) return;
        _firstName = value;
        OnPropertyChanged();
        OnPropertyChanged("FullName");
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
        OnPropertyChanged("FullName");
    }
}

public string FullName
{
    get { return string.Format("Full Name: {0} {1}", FirstName, LastName); }
}
{% endhighlight %}
![Scenario 1-Fixed]({{ site.url }}/images/MVVMOutput/Scenario1-Fixed.jpg)

For a more "exotic" example, let's say there is a third input for their favorite color.  Another output property now depends on the FullName and the Favorite Color.

{% highlight csharp %}
public string FirstName
{
    get { return _firstName; }
    set
    {
        if (value == _firstName) return;
        _firstName = value;
        OnPropertyChanged();
        OnPropertyChanged("FullName");
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
        OnPropertyChanged("FullName");
    }
}

public string FullName
{
    get { return string.Format("Full Name: {0} {1}", FirstName, LastName); }
}

public string FavoriteColor
{
    get { return _favoriteColor; }
    set
    {
        if (value == _favoriteColor) return;
        _favoriteColor = value;
        OnPropertyChanged();
    }
}

public string Sentence
{
    get { return string.Format("{0}'s favorite color is: {1}", FullName, FavoriteColor); }
}
{% endhighlight %}
![Scenario 2]({{ site.url }}/images/MVVMOutput/Scenario2.jpg)

OK, sure, let's just apply the same logic... not so fast.  FullName doesn't have a setter.  So I need to call OnPropertyChanged() in the things that it depends on.  

{% highlight csharp %}
public string FirstName
{
    get { return _firstName; }
    set
    {
        if (value == _firstName) return;
        _firstName = value;
        OnPropertyChanged();
        OnPropertyChanged("FullName");
        OnPropertyChanged("Sentence");
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
        OnPropertyChanged("FullName");
        OnPropertyChanged("Sentence");
    }
}

public string FullName
{
    get { return string.Format("Full Name: {0} {1}", FirstName, LastName); }
}

public string FavoriteColor
{
    get { return _favoriteColor; }
    set
    {
        if (value == _favoriteColor) return;
        _favoriteColor = value;
        OnPropertyChanged();
        OnPropertyChanged("Sentence");
    }
}

public string Sentence
{
    get { return string.Format("{0}'s favorite color is: {1}", FullName, FavoriteColor); }
}
{% endhighlight %}
![Scenario 2-Fixed]({{ site.url }}/images/MVVMOutput/Scenario2-Fixed.jpg)

This code is starting to smell.  We have setters becoming full of code to notify the UI of changes to properties that aren't directly associated with them.  I've seen code with something like:
{% highlight csharp %}
public void NotifyAllTheThings()
{
    //Reflect over all properties and notifyChanges
}
{% endhighlight %}

Just because they (I) couldn't keep track of all the dependencies.

Well, there **Is A Better Way**™.  [ReactiveUI](https://github.com/reactiveui/ReactiveUI) which uses [Reactive Extensions](https://github.com/Reactive-Extensions/Rx.NET) (Rx) allows us to create a pipeline effect for our notifications.

I'll show the complete viewmodel before we make any changes:
{% highlight csharp %}

using System.ComponentModel;
using System.Runtime.CompilerServices;
using OutputProperties.Annotations;

namespace OutputProperties
{
    public class MainWindowViewModel : INotifyPropertyChanged
    {
        private string _firstName;
        private string _lastName;
        private string _favoriteColor;
        public event PropertyChangedEventHandler PropertyChanged;

        [NotifyPropertyChangedInvocator]
        protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
        {
            PropertyChangedEventHandler handler = PropertyChanged;
            if (handler != null) handler(this, new PropertyChangedEventArgs(propertyName));
        }

        public string FirstName
        {
            get { return _firstName; }
            set
            {
                if (value == _firstName) return;
                _firstName = value;
                OnPropertyChanged();
                OnPropertyChanged("FullName");
                OnPropertyChanged("Sentence");
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
                OnPropertyChanged("FullName");
                OnPropertyChanged("Sentence");
            }
        }

        public string FullName
        {
            get { return string.Format("Full Name: {0} {1}", FirstName, LastName); }
        }

        public string FavoriteColor
        {
            get { return _favoriteColor; }
            set
            {
                if (value == _favoriteColor) return;
                _favoriteColor = value;
                OnPropertyChanged();
                OnPropertyChanged("Sentence");
            }
        }

        public string Sentence
        {
            get { return string.Format("{0}'s favorite color is: {1}", FullName, FavoriteColor); }
        }

    }
}
{% endhighlight %}

First, we'll get ReactiveUI from NuGet.  Then we'll replace our INotifyChanged with ReactiveObject.  That forces us to change the setters.  That's OK, there's a nice method for checking for update changed and raise all in one call: this.RaiseAndSetIfChanged().

{% highlight csharp %}
using ReactiveUI;

namespace OutputProperties
{
    public class MainWindowViewModel : ReactiveObject
    {
        private string _firstName;
        private string _lastName;
        private string _favoriteColor;

        public string FirstName
        {
            get { return _firstName; }
            set { this.RaiseAndSetIfChanged(ref _firstName, value); }
        }

        public string LastName
        {
            get { return _lastName; }
            set { this.RaiseAndSetIfChanged(ref _lastName, value); }
        }

        public string FullName
        {
            get { return string.Format("Full Name: {0} {1}", FirstName, LastName); }
        }

        public string FavoriteColor
        {
            get { return _favoriteColor; }
            set { this.RaiseAndSetIfChanged(ref _favoriteColor, value); }
        }

        public string Sentence
        {
            get { return string.Format("{0}'s favorite color is: {1}", FullName, FavoriteColor); }
        }

    }
}
{% endhighlight %}

OK, but now we're back to not notifying the FullName or Sentence properties.  Right, we need to talk about some more ReactiveUI first.  Specifically, WhenAnyValue, ObservableAsPropertyHelper and ToProperty.  

WhenAnyValue allows us to get notified when a property changes.  
{% highlight csharp %}
var fullName = this.WhenAnyValue(x => x.FirstName, x => x.LastName, (first, last) => new {first,last})
{% endhighlight %}
This will let us know whenever there are changes to either FirstName or LastName and create a new object that contains both.

Now ObservableAsPropertyHelper and ToProperty go hand in hand. ObservableAsPropertyHelper boiler plate for an output property in ReactiveUI. ToProperty allows us to set this property.

{% highlight csharp %}
ObservableAsPropertyHelper<string> _fullName;
public string FullName
{
    get { return _fullName.Value; }
}
public MainWindowViewModel()
{
    this.WhenAnyValue(x => x.FirstName, x => x.LastName, (first, last) => new {first,last})
        .Select(name => string.Format("Full Name: {0} {1}", name.first, name.last))
        .ToProperty(this, x => x.FullName, out _fullName);
}
{% endhighlight %}

Now we can see the train of how FullName will get its value.  Any time FirstName or LastName update, select a string.Format() of them and update the FullName property.

OK, well, let's fix Sentence:

{% highlight csharp %}   ObservableAsPropertyHelper<string> _fullName;
public string FullName
{
    get { return _fullName.Value; }
}


ObservableAsPropertyHelper<string> _sentence;
public string Sentence
{
    get { return _sentence.Value; }
}
public MainWindowViewModel()
{
    this.WhenAnyValue(x => x.FirstName, x => x.LastName, (first, last) => new {first,last})
        .Select(name => string.Format("Full Name: {0} {1}", name.first, name.last))
        .ToProperty(this, x => x.FullName, out _fullName);

    this.WhenAnyValue(x => x.FullName, x => x.FavoriteColor, (full,color) => new {full,color})
        .Select(x => string.Format("{0}'s favorite color is: {1}", x.full, x.color))
        .ToProperty(this, x => x.Sentence, out _sentence);
}
{% endhighlight %}

Now we see the true intent.  Any time FullName or FavoriteColor is updated, we should change the SentenceProperty.

Full end result:
{% highlight csharp %}
using System.Reactive.Linq;
using ReactiveUI;

namespace OutputProperties
{
    public class MainWindowViewModel : ReactiveObject
    {
        private string _firstName;
        private string _lastName;
        private string _favoriteColor;

        ObservableAsPropertyHelper<string> _fullName;
        public string FullName
        {
            get { return _fullName.Value; }
        }

        ObservableAsPropertyHelper<string> _sentence;
        public string Sentence
        {
            get { return _sentence.Value; }
        }

        public MainWindowViewModel()
        {
            this.WhenAnyValue(x => x.FirstName, x => x.LastName, (first, last) => new {first,last})
                .Select(name => string.Format("Full Name: {0} {1}", name.first, name.last))
                .ToProperty(this, x => x.FullName, out _fullName);

            this.WhenAnyValue(x => x.FullName, x => x.FavoriteColor, (full,color) => new {full,color})
                .Select(x => string.Format("{0}'s favorite color is: {1}", x.full, x.color))
                .ToProperty(this, x => x.Sentence, out _sentence);
        }

        public string FirstName
        {
            get { return _firstName; }
            set { this.RaiseAndSetIfChanged(ref _firstName, value); }
        }

        public string LastName
        {
            get { return _lastName; }
            set { this.RaiseAndSetIfChanged(ref _lastName, value); }
        }

        public string FavoriteColor
        {
            get { return _favoriteColor; }
            set { this.RaiseAndSetIfChanged(ref _favoriteColor, value); }
        }

    }
}
{% endhighlight %}

[Source code](https://github.com/TheAngryByrd/MVVM-Output-Properties/)
