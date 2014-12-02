---
layout: post
title:  "Finding optimal university schedules"
tags: university schedules python winston classtime heaps

published: true
---

#### The context

[Andrew Hoskins]({{ site.data.github.andrewhoskins }}) and I are working on {{ site.data.names.classtime }}, a university scheduling webapp. You tell {{ site.data.names.classtime }} what courses you're taking and what you want in a schedule. He then quickly finds you a schedule that fits your life. 

#### The task

Starting from scratch, find an optimal set of schedules given a list of courses.

Each course may have zero or more components (lectures, labs, and seminars). Each component may have zero or more sections (eg Tuesday & Thursday 9:00am-9:50am).

---

#### The bigger picture

This has been done before: see [Matthew Hoener's UVic schedulebuilder](http://schedulecourses.com/), [Alex Teske's UOttawa schedule-builder](https://github.com/alex-teske/schedule-builder), [Ben Grawi, Ben Russell, and John Resig's Rochester Institute of Technology ScheduleMaker](http://schedule.csh.rit.edu/), and [Mason Strong]({{ site.data.github.masonstrong }}) and [Peter Crinklaw](http://blackacrebrewing.com/hey.swf)'s UAlberta schedule-builder.

Its purpose is to generate schedules. But we'll know we've done it right if it also lets someone

- rest and eat after morning practice
- see their friends more
- go on longer ski trips
- do anything that makes them happier during the semester.

##### The strategy

Start with a blank candidate schedule. 

In each round, create many new candidates by scheduling a required course in many different ways. In each round, hold onto the best `N` candidates.

Continue until all courses are scheduled.
 
##### The implementation

Language: Python 2.7

Code samples are snippets, and exclude details. Anything not explicitly described can be assumed to behave as it is named.

Ok. First, some groundwork needs to be laid.

##### Groundwork

These useful pieces will be written first.

- a representation of a section
- a representation of a schedule
- a way to check for conflicts
- a way to evalute a schedule

Each section is represented as a [python dictionary](https://docs.python.org/2/library/stdtypes.html). This is a fragile solution: a dictionary is hard to debug and has no error-checking or interface capabilities. However, it has low development overhead, and is good for first attempts, experimentation and the like.

Later, this will be refactored into a Section [class](http://www.diveintopython.net/object_oriented_framework/defining_classes.html), which will be more robust and testable.

The dictionaries will look like this:

{% highlight python %}
section = {
              'day': 'TR',
              'startTime': '08:00 AM',
              'endTime': '08:50 AM',
              ...<more fields like course, Lec/Lab/Sem, instructor, location>
          }
{% endhighlight %}

The timetable is implemented as a list of 5 lists, each representing one day in the week. Each day is a list of 48 elements, each representing one 30-minute block in the day. Constants are used to denote whether a certain block of time is available or not.

The list is built using a neat feature in python called a [list comprehension](http://www.pythonforbeginners.com/basics/list-comprehensions-in-python).

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

Conflicts are detected by adding the section to a blank schedule, then iterating through it and the main schedule at the same time using python's [`zip`](http://www.saltycrane.com/blog/2008/04/how-to-use-pythons-enumerate-and-zip-to/). Each block of time is looked at on each day. If both schedules are busy at any one time, there is a conflict. If no conflicts are found after looking at all days, then there is no conflict.

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

A contract to specify the system's behaviour is also worth writing. This can be done by writing unit tests. Here, a number of scenarios are collected, along with their expected result. A method is written to build the schedule in each scenario and [assert](https://docs.python.org/2/reference/simple_stmts.html#the-assert-statement) that the expected result is correct.

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

Lastly for the groundwork, schedules will be naively sorted by comparing how close each is to being completed. This will be computed as the number of sections already added to a schedule.

The work of sorting is done by defining a [comparison method](http://jcalderone.livejournal.com/32837.html) for `Schedule` objects. Two constants are also declared in the interest of future readability when more intelligent comparisons are implemented.

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

Now, the main event: generating schedules.

### Schedule Generation

Python's implementation of a [heap][Wikipedia-heap], called [heapq][python-heapq], is used to keep track of candidate schedules during the search.

A [heap][Wikipedia-heap] is a sorted [recursive data structure][Wikipedia-recursive-data-structure] with [log(n) insertion][Wikipedia-big-oh-notation]. The head can be removed in [constant time][Wikipedia-big-oh-notation], so its size can be restricted by removing the head after each insertion. And, since python's heapq is a [min-heap][Wikipedia-img-min-heap], the head is the minimum, and thus the "worst" node will be discarded after each insertion. This makes it ideal for storing "the best N" of a collection. The downside is that "best" nodes are hard to find, since maximum nodes are in the leaves - the whole tree must be traversed.

In our case, this is a good fit since

1. If we assume that bad half-done schedules are likely to become bad complete schedules, we can avoid computing bad complete schedules by throwing out the bad half-done schedules. This can be done by restricting the heap size.
2. The best schedules only need to be found once, at the very end. This means we only pay the price of finding maximum nodes once, which is acceptable.

[Wikipedia-recursive-data-structure]: http://en.wikipedia.org/wiki/Recursive_data_type
[Wikipedia-heap]: http://en.wikipedia.org/wiki/Heap_%28data_structure%29
[Wikipedia-img-min-heap]: upload.wikimedia.org/wikipedia/commons/6/69/Min-heap.png
[Wikipedia-big-oh-notation]: http://en.wikipedia.org/wiki/Time_complexity#Table_of_common_time_complexities
[python-heapq]: https://docs.python.org/2/library/heapq.html

An empty schedule is used as the initial candidate.

{% highlight python %}
candidates = [Schedule()]
{% endhighlight %}

For each section, iterate through a **copy** of the candidate list using `[:]`. This allows us to push new candidates onto the heap while we iterate through it.

{% highlight python %}
for section in sections:
    for candidate in candidates[:]:
{% endhighlight %}

If the section conflicts with the candidate, do nothing. If the section does not conflict, [deep copy](https://docs.python.org/2/library/copy.html) the candidate, add the section, and push the new candidate onto the heap.

If the heap is at its max size, discard the worst schedule with `heapq.heapreplace()`.

{% highlight python %}
if candidate.conflicts(section):
    continue
new_candidate = candidate.add_section_and_deepcopy(section)
if len(candidates) >= HEAP_SIZE:
    heapq.heapreplace(candidates, new_candidate)
else:
    heapq.heappush(candidates, new_candidate)
{% endhighlight %}

Once all sections have been scheduled, exclude candidates with incomplete schedules.

{% highlight python %}
candidates = [candidate for candidate in candidates
              if len(candidates[0].sections) == len(sections)]
{% endhighlight %}

Return only the best schedules by sorting the list in best-first order with `reverse=True` and then [slicing the list](http://stackoverflow.com/questions/509211/explain-pythons-slice-notation).

{% highlight python %}
return sorted(candidates, reverse=True)[:RETURN_SIZE]
{% endhighlight %}

Here's the full code

{% highlight python linenos %}

def generate_schedules(sections):
    HEAP_SIZE = 50
    RETURN_SIZE = 10

    candidates = [Schedule()]
    for section in sections:
        for candidate in candidates[:]:
            if candidate.conflicts(section):
                continue
            new_candidate = candidate.add_section_and_deepcopy(section)
            if len(candidates) >= HEAP_SIZE:
                heapq.heapreplace(candidates, new_candidate)
            else:
                heapq.heappush(candidates, new_candidate)

    if not candidates:
        return []
    candidates = [candidate for candidate in candidates
                  if len(candidates[0].sections) == len(sections)]
    return sorted(candidates, reverse=True)[:RETURN_SIZE]

{% endhighlight %}

Adios, until next time.
