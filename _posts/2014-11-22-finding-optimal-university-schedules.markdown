---
layout: post
title:  "Finding optimal university schedules"
tags: python threading multiprocessing winston classtime

published: true
---

##### The context

[Andrew Hoskins]({{ site.data.github.ahoskins }}) and I are working on {{ site.data.names.classtime }}, a university scheduling assistant. You tell {{ site.data.names.classtime }} what courses you're taking, and he quickly finds you a schedule that fits your life. 

This has been done before: see [Matthew Hoener's UVic schedulebuilder](http://schedulecourses.com/), [Alex Teske's UOttawa schedule-builder](https://github.com/alex-teske/schedule-builder), [Ben Grawi, Ben Russell, and John Resig's Rocherster Institute of Technology ScheduleMaker](http://schedule.csh.rit.edu/), not to mention our classmate 

##### The task

Find an optimal set of schedules given a list of courses.

Each course may zero or more components (lectures, labs, and seminars). Each component may have zero or more sections (eg Tuesday & Thursday 9:00am-9:50am).

##### The strategy so far

Start with a blank candidate schedule. In each round, create many new candidates by scheduling a required course in many different ways. In each round, hold onto the best `N` candidates. Continue until all courses are scheduled.

##### The implementation

Note: code samples are sinppets, and exclude details. Anything not explicitly described can be assumed to behave as it is named.

Ok. First, we'll need a way to represent the schedule.

{% highlight python linenos %}
class Schedule(object):
    NUM_BLOCKS = 24*2
    SCHOOL_DAYS = 5

    BUSY = 1
    OPEN = None

    def __init__(self, sections=[]):
        self.schedule = [[Schedule.OPEN]*Schedule.NUM_BLOCKS
                         for _ in range(Schedule.SCHOOL_DAYS)]
        for section in sections:
            self._add(section)
{% endhighlight %}

And a way to check for conflicts

...to be continued
