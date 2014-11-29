---
layout: post
title:  "Finding optimal university schedules"
tags: python winston classtime

published: true
---

##### The context

[Andrew Hoskins]({{ site.data.github.andrewhoskins }}) and I are working on {{ site.data.names.classtime }}, a university scheduling webapp. You tell {{ site.data.names.classtime }} what courses you're taking, when you're, and he quickly finds you a schedule that fits your life. 

This has been done before: see [Matthew Hoener's UVic schedulebuilder](http://schedulecourses.com/), [Alex Teske's UOttawa schedule-builder](https://github.com/alex-teske/schedule-builder), [Ben Grawi, Ben Russell, and John Resig's Rocherster Institute of Technology ScheduleMaker](http://schedule.csh.rit.edu/), and [Mason Strong]({{ site.data.github.masonstrong }}) and [Peter Crinklaw](http://blackacrebrewing.com/hey.swf)'s UAlberta schedule-builder. 

Its purpose is to generate schedules. But we'll know we've done it right if it also lets someone see their friends more, rest after morning practice, go on longer ski trips, or do anything that makes them happier during the semester.

##### This task

Find an optimal set of schedules given a list of courses.

Each course may have zero or more components (lectures, labs, and seminars). Each component may have zero or more sections (eg Tuesday & Thursday 9:00am-9:50am).

##### The strategy

Start with a blank candidate schedule. In each round, create many new candidates by scheduling a required course in many different ways. In each round, hold onto the best `N` candidates. Continue until all courses are scheduled.
 
##### The implementation

Language: Python 2.7. Code samples are snippets, and exclude details. Anything not explicitly described can be assumed to behave as it is named.

Ok. First, some groundwork needs to be laid.

##### Groundwork

These useful blocks will be written first.

- a representation of a section
- a representation of a schedule
- a way to check for conflicts
- a way to evalute a schedule

Each section is represented as a [python dictionary](https://docs.python.org/2/library/stdtypes.html). This is a fragile solution: a dictionary is hard to debug and has no behaviour associated with it. However, it has low development overhead, and is good for first attempts, experimentation and the like. Later, this will be refactored into a Section [class](http://www.diveintopython.net/object_oriented_framework/defining_classes.html), which will be more robust and testable.

The dictionaries will look like this:

{% highlight python %}
{
    'day': 'TR',
    'startTime': '08:00 AM',
    'endTime': '08:50 AM',
    ...<more fields>
},
{% endhighlight %}

For the timetable, there are 5 lists, each representing one day. Each day is a list of 48 elements, each representing a 30-minute block in the day. Constants are used to denote whether a certain block of time is available or not.  

{% highlight python %}
class Schedule(object):
    NUM_BLOCKS = 24*2
    SCHOOL_DAYS = 5

    BUSY = 1
    OPEN = None

    def __init__(self, sections=[]):
        self.timetable = [[Schedule.OPEN]*Schedule.NUM_BLOCKS
                         for _ in range(Schedule.SCHOOL_DAYS)]
        for section in sections:
            self._add(section)
{% endhighlight %}

Conflicts are detected by adding the section to a blank schedule, then iterating through it and the main schedule at the same time using python's [`zip`](http://www.saltycrane.com/blog/2008/04/how-to-use-pythons-enumerate-and-zip-to/). Each block of time is looked at, and if both schedules are busy, there is a conflict. If a conflict isn't found, then there is no conflict. 

{% highlight python %}
def conflicts(self, section):
    """
    Returns true if there is a conflict between:
    1) this schedule (self), and
    2) other, which is a section dict containing at LEAST
    the properties 'component', day', 'startTime', and 'endTime'
    """
    other = Schedule(section)
    for ourday, theirday in zip(self.timetable, other.timetable):
        for ourblock, theirblock in zip(ourday, theirday):
            if ourblock == Schedule.BUSY and theirblock == Schedule.BUSY:
                return True
    return False
{% endhighlight %}

A contract to hold the system to is also worth specifying. This can be done by writing tests. A list of scenarios are hand-calculated, and have their expected result attached. A method is written to build the given schedule, and [asserts](https://docs.python.org/2/reference/simple_stmts.html#the-assert-statement) that it either has a conflict or does not (whichever is expected). 

{% highlight python %}
def test_conflicts():
    """
    Test conflict detection between a schedule and
    a candidate section.
    """
    scenarios = [
        {
            'expected': True,
            'sections':
            [
                {
                    'day': 'TR',
                    'startTime': '08:00 AM',
                    'endTime': '08:50 AM'
                },
                {
                    'day': 'MTWRF',
                    'startTime': '08:00 AM',
                    'endTime': '08:50 AM'
                }
            ]
        },
        {
            'expected': False,
            'sections':
            [
                {
                    'day': 'TR',
                    'startTime': '08:00 AM',
                    'endTime': '08:50 AM'
                },
                {
                    'day': 'TR',
                    'startTime': '09:00 AM',
                    'endTime': '09:50 AM'
                }
            ]
        },
        ...<more scenarios>
    ]
    
    for scenario in scenarios:
        sections = scenario.get('sections')
        expected = scenario.get('expected')
        yield check_conflict, sections, expected
{% endhighlight %}

These tests are run in the command line with [`$ nosetests`][nosetests-read-the-docs] to keep us honest during refactoring or modifications. C. Titus Brown wrote a great tutorial on [nosetests][nosetests-read-the-docs] which can be found [here][nosetests-tutorial] if you're interested.

[nosetests-read-the-docs]: https://nose.readthedocs.org/en/latest/
[nosetests-tutorial]: http://ivory.idyll.org/articles/nose-intro.html

Lastly for the groundwork, schedules will be naively sorted by how close they are to being completed, which will be measured as the number of different sections already scheduled.

In `class Schedule`, a comparison method, [__lt__](http://jcalderone.livejournal.com/32837.html), is defined. Two constants are also declared in the interest of future readability when more intelligent comparisons are implemented.

{% highlight python %}
SELF_IS_WORSE = True
SELF_IS_BETTER = False
"""Semantic sorting constants"""

def __lt__(self, other):
    if len(self.sections) > len(other.sections):
        return Schedule.SELF_IS_BETTER
    else:
        return Schedule.SELF_IS_WORSE
{% endhighlight %}

Now the main event: generating and sorting schedules. 

### Schedule Generation

To keep track of candidate schedules during the search, we've used python's implementation of a [heap][Wikipedia-heap], called [heapq][python-heapq]. It's a sorted data structure with [log(n) insertion][Wikipedia-big-oh-notation]. It is [constant time][Wikipedia-big-oh-notation] to remove the head, so it is easy to restrict its size by removing the head after inserting a node. And, since python's heapq is a [min-heap][Wikipedia-img-min-heap], the head of the tree is the node with the smallest value. This allows the behaviour of storing "the best N" of a collection. However, the maximum nodes (with the largest values) are hard to find, since they are in the leaves - the whole tree must be traversed. 

In our case, this is a good fit since 

1) we want to avoid wasting time computing bad schedules, where we assume that bad candidate schedules produce bad complete schedules
2) we only need to find the maximums once, at the very end, where the maximums are the best schedules

[Wikipedia-heap]: http://en.wikipedia.org/wiki/Heap_%28data_structure%29
[Wikipedia-img-min-heap]: upload.wikimedia.org/wikipedia/commons/6/69/Min-heap.png
[Wikipedia-big-oh-notation]: http://en.wikipedia.org/wiki/Time_complexity#Table_of_common_time_complexities
[python-heapq]: https://docs.python.org/2/library/heapq.html


