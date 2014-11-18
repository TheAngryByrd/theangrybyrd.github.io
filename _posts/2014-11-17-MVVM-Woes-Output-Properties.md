---
published: false
---

## MVVM Woes - Output properties

In MVVM, I regularly have output properties, or properties that usually depend on one or more other properties within the viewmodel.  For example, say we have 2 input fields, one for first name and one for last name.  The output field would be a string.Format() of these two fields showing the full name.  

Simple VM:
{% gist TheAngryByrd/4feb42e5a8ee173e0f77 %}

Simple demo:
![Scenario 1]({{ site.url }}/images/MVVMOutput/Scenario1.jpg)


For a more "exotic" example.  Let's say there is a third input for their favorite color.  Another output property depends now on the 
