---
layout: post
title: A story about concurrency and parallelism
date:   2016-02-22 20:50:06
categories: Summary
image: /assets/images/post.jpg
---
<h2>{{ page.title }}</h2>

  <p>This is a story about concurrency and parallelism</p>
  <p>My wife is a teacher. Like most teachers, she's a master of multitasking. At any one
     instant, she's only doning one thing, but she's having to deal with many things concurrently.
     While listening to one child read, she might break off to calm down a rowdy classroom or answern
     a question. This is concurrent, but it's not parallel(there is only one of her).
  </p>
  <p>
     If she's joined by an assistant(one of them listening to an individual reader, the other answering questions),
     we now have something that's both concurrent and parallel.
  </p>
  <p>
     Imagine that the class has designed its own greeting cards and wants to mass-produce them.
     One way to do so would be to give each child each child the task of making five cards.
     This is parallel but not(viewed from a high enough level) concurrent -- only one task is underway(The task is making five cards).
  </p>
  <p>
    Concurrency is about dealing with lots of things at once.
    Parallelism is about doning lots things at once.
  </p>
  <p>The above content is from 《Seven Concurrency Models in Seven Weeks》</p>
  <p>I get:</p>
  <p>
     Concurrency is about a great deal more than just exploiting parallelism—used
     correctly, it allows your software to be responsive, fault tolerant, efficient,
     and simple.
     The world is concurrent, and so should your software be if it wants to interact
     effectively.
     Concurrency is the key to responsive systems.
  </p>
  <p>
     Why the concurrency can work efficiently. I think the design of CPU running 、Thread scheduling and Process scheduling
     are keys. And I can understand the microcosmic concept after I get the Round Robin and Time Slice Manage-ment.
     Yes it is amazing! This is a amazing design after the brain!
  </p>
