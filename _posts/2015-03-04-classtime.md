---
layout: project
title:  classtime
date:   2015-03-04 02:28:37 -0700
categories: ualberta winston classtime
oneline: HTTP API providing raw UAlberta course data as well as schedule generation.
github: rosshamish/classtime
demo-text: |
    <pre>
    GET http://classtime.herokuapp.com/api/v1/courses-min
    {
        "faculty": "Faculty of Business",
        "subjects": [
            {
              "subject": "ACCTG",
              "subjectTitle": "Accounting",
              "courses": [
                     {
                          "course": "000001",
                          "asString": "ACCTG 300",
                          "courseTitle": "Intermediate Accounting"
                     },
                     { course }
                     ...
               ]
           },
           { subject }
           ...
        ]
    },
    { faculty }
    ...


    GET http://classtime.herokuapp.com/api/v1/generate-schedules?q=?...
    {
        "sections": [
            {
                ...
                course attributes
                section attributes
            },
            { section }
            ...
        ],
        "more_like_this": [schedule-identifier, schedule-identifier, ..]
    },
    { schedule }
    ...
    </pre>
---

Used by the official frontend at http://heywinston.com

### Example

