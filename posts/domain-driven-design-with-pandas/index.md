<!--
.. title: Domain-Driven Design architecture with Pandas
.. slug: domain-driven-design-with-pandas
.. date: 2019-09-06 15:29:14 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-- status: private
-->

Hi guys. Today I would like talk about implementing Domain-Driven Design (DDD)
to build a Business Intelligence system with Python and Pandas. There are many 
articles talking about DDD already but 
this article aims to show you an example of what you can do with it.

## My Background

I work in finance where I need to handle data with complex relationship on a daily basis.
The raw data that I received is usually too "raw" such that extracting any useful information from it
often requires a lot of steps. Codes became impossible to maintain and a lot of codes
are duplicated because of the mismatch of raw data structure and the business logic.

#### The solution
 
Soon I realized that I needed a DDD layer on top of the raw data to handle all the complexity for me. 
The reason I use the term **DDD layer** here is that DDD is essentially an abstraction layer of a system. 
**It allows you to focus on business logic only, instead of having to worry about how to implement business from raw data.**
Really, architecture is all about reducing your cognitive burden when you try to edit a codebase.

Now I will demonstrate a simple design and slowly build on top of that as the codebase grow.

# faesfase




