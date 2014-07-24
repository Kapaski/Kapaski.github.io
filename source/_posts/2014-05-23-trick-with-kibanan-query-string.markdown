---
layout: post
title: "Trick with Kibanan query string"
date: 2014-05-23 10:41:02 +1000
comments: true
categories: AngularJS, Kibana, ElasticSearch
---
Since we already did this couple of times, maybe put it in a post for others convenience:

[Github Issue Ref](https://github.com/elasticsearch/kibana/issues/168)

This will allow the user can add a ?query=foo to search with keywords 'foo'

![img](https://camo.githubusercontent.com/058cd5d8173b179cd88f942118e06018909aa54b/68747470733a2f2f662e636c6f75642e6769746875622e636f6d2f6173736574732f313235303338372f313038323737362f62643839646664382d313538642d313165332d383165312d3635346161636533633361352e706e67)

You must put above string in one query field and save the dashboard to make it happen, however the defect will be next time when you try to save the dashboard and forget put the samething in the query field, it will be lost.

Quick walkaround will be --  `pin`  it then save, and make your panels listen to this pinned query dedicatedly, and then you will have a dynamic panel showing search result based on http query strings!
