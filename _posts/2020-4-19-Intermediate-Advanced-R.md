---
layout: post
title: "Digging deeper: online resources for intermediate to advanced R users"
excerpt_separator: <!--more-->
---

Anybody wanting to learn R from scratch will find an incredible wealth of tutorials, interactive learning websites, and high-quality videos at their disposal—almost to a point where it's difficult to know where to start! This is of course a good thing, and is mainly due to R's quickly growing popularity, with a constant stream of new users from both industry and academia wanting to learn the fundamentals.

But I've found that once you reach a certain level of confidence with the language, it becomes more difficult to find material for intermediate/advanced users who wish to become really good at R programming. But these materials do exist—they just tend to be mentioned and highlighted less frequently by the community.

Hence this post, where I've tried to gather a variety of books, courses and resources that should be beneficial to you, if you're at that level where you don't need another tidyverse tutorial, but wish you could get advanced insights from seasoned R programmers. 

<!--more-->

Note that I've only listed here material that I've actually tested and used (maybe not entirely, but at least enough to judge its benefits). I'm sure there are many more resources out there that are as valuable!

---

## Books

There are actually quite a few books available on intermediate/advanced R programming ([O'Reilly](https://www.oreilly.com/search/?query=R&extended_publisher_data=true&highlight=true&include_assessments=false&include_case_studies=true&include_courses=true&include_playlists=true&include_collections=true&include_notebooks=true&is_academic_institution_account=false&source=user&formats=book&sort=relevance&facet_json=true&page=0&include_facets=false&include_scenarios=true&include_sandboxes=true&json_facets=true) is usually a good source for this), but I've chosen the four books below in particular because:

* They cover broad areas of R programming, rather than very narrow issues that will only be interesting to specific users (such as advanced statistical models, large-scale Shiny apps, etc.);
* All can be read for free on a dedicated website—but of course, buying a physical copy is the best way to thank their authors if you found the contents useful (which you'll most likely do!).

![Advanced R books](https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/bookshelf.png)

### [_Advanced R_, by Hadley Wickham](http://adv-r.had.co.nz/)

I feel like this almost shouldn't be listed here, since it hardly fits my definition of material "less frequently mentioned by the community"! Hadley Wickham's book is the main reference on advanced R programming. It takes you all the way back to the fundamentals, teaching you the intricacies of variable assignment, vectors and matrices—not because you don't know what they are, but because you don't _entirely_ know what they are. Contents also cover object-oriented programming, debugging, performance/profiling, and a highly-useful style guide.

### [_Efficient R programming_, by Colin Gillespie & Robin Lovelace](https://csgillespie.github.io/efficientR/)

This book seems to be somewhat less known in the R community (I found it through a random Google search), but it is the perfect resource if you feel like you know how to get things done in R, but want to learn how to get things done "properly" and efficiently. The concept of efficiency has a broad definition here, and includes obvious aspects such as code optimisation and profiling, but also workflow, data input/output, collaboration, learning, and many others. Highly recommended!

### [_Data Visualization: A practical introduction_, by Kieran Healy](http://socviz.co/)

This circulated a lot on Twitter last year when it was published online, and I read the whole thing before it was subsequently published as a book. This holds extremely valuable information for anybody producing data visualizations for their projects. I found it to have the perfect balance between theoretical aspects (principles of perception, data representation, ease of understanding) and practical learning on how to implement things in ggplot2. Note that Part 2 (_"Get started"_) can be skipped almost entirely if you're already an experienced R user.

### [_R packages_, by Hadley Wickham](http://r-pkgs.had.co.nz/)

I haven't read _R packages_ as thoroughly as the other books above, but it is the main reference book on how (and why) to write R packages. Many people equate packages with CRAN; and while it is of course very useful to make your code available to other R users around the world, packages can also simply be a way to better organise your projects, and to make sure that your entire workflow is very tidy and fully reproducible.

---

## Courses

Online courses are usually much more expensive than the one-off price of a book, but of course they offer a lot more interactivity. If you know you're the kind of person who will read through a book but won't have the motivation to re-type the examples and do the exercises, you might want to check out those resources.

### [Mastering Software Development in R Specialization, by Johns Hopkins University on Coursera.org](https://www.coursera.org/specializations/r)

**Cost:** Free for 7 days, then ~$40/£28/32€ per month

**Duration:** 3 to 6 months

This series of courses aims at teaching users how to _"design software for data tooling, distribute R packages, and build custom visualizations"_. The first course ("The R Programming Environment") probably won't be needed for intermediate R users, but subsequent courses ("Advanced R Programming", "Building R Packages", and "Building Data Visualization Tools") should be a great way to extend your knowledge. Note that I haven't tested this new version of the specialization, but the former version (already taught by Roger Peng) was the main way I learned R a few years ago.

### [R courses on Datacamp.com](https://www.datacamp.com/courses/tech:r)

**Cost:** ~$29/£20/23€ per month

**Duration:** each course takes between 3 and 6 hours

Another well-known resource in the R community, Datacamp provides one of the best platforms for beginners to learn R, but also a great way to dig deeper and widen your skillset. Intermediate and advanced courses include "Writing Functions" (taught by Hadley Wickham), "Building Web Applications with Shiny" (Mine Cetinkaya-Rundel), "Data Analysis, the data.table Way" (Matt Dowle), "Working with Web Data" (Charlotte Wickham), "Object-Oriented Programming" (Richie Cotton), "Writing Efficient Code" (Colin Gillespie), and "Scalable Data Processing" (Michael Kane).

### [Code Clinic: R, by Mark Niemann-Ross on Lynda.com](https://www.lynda.com/R-tutorials/Code-Clinic-R/372541-2.html)

**Cost:** Free for 1 month, then ~$30/£21/24€ per month

**Duration:** 3 hours of video + a few more hours to go through the problems

This course is very different from everything else mentioned on this list. The idea of "code clinics" on Lynda.com is to solve a series of general computer science problems by using a specific programming language. This means that those problems are very different from those in typical data-related R tutorials: you'll learn about image processing, chess simulations, and even how to interact with your computer peripherals to produce sound when you move your mouse.

### [Posit webinars](https://posit.co/resources/videos/)

**Cost:** Free

**Duration:** around 40 minutes to 1 hour

Posit (ex-RStudio) runs regular webinars, i.e. live teaching videos covering a very specific topic in R programming... so specific that you might not see any possible use in some of them. But I highly recommend going through their list of past webinars and watching whatever seems relevant to your particular job/role.

---

## What else?

If you've been through all the material above, it's probably time to delve into more specific subtopics. I find [this Quora answer by Benjamin Paul Rollert](https://www.quora.com/How-do-I-become-an-expert-in-R-if-I%E2%80%99m-an-intermediate-now-Any-good-books-lectures-or-blogs/answer/Benjamin-Paul-Rollert) to be quite complete. More generally, you can search for ["advanced R courses"](https://www.google.com/search?q=advanced+r+course) being taught in universities or elsewhere. The topics covered by the course are often listed on the booking page, so that's another great way to find new areas to explore.
