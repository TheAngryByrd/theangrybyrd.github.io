---
published: true
---

## MVVM Woes - Output properties

In MVVM, I regularly have output properties, or properties that usually depend on one or more other properties within the viewmodel.  For example, say we have 2 input fields, one for first name and one for last name.  The output field would be a string.Format() of these two fields showing the full name.  

Simple VM:
{% gist TheAngryByrd/4feb42e5a8ee173e0f77/948bbc0c5ec29f4d3bea5b6abc5cabc119543a86 %}

Simple demo:
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

This code is starting to smell.
