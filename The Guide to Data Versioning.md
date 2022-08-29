# The Guide to Data Versioning
> **“I have never lied to you, I have always told you some version of the truth.”**

> **“The truth doesn’t have versions, okay?”** — Something’s Gotta Give (2003)

![](https://lakefs.io/wp-content/uploads/2021/11/the-guide-to-data-versioning-lakefs-1536x777.jpg)

Jack Nicholson and Diane Keaton discuss data versioning in Something's Gotta Give.

A **version** of something is defined as “a particular form in which some details are different from earlier or later forms.” In the digital world, versioning is a luxury we are fortunate to indulge — maintaining multiple versions of pretty much anything, from small objects to whole systems. 

Many things we interact with are versioned automatically — word documents, codebases, the software that runs our precious phones. We rarely think twice about it.

The reason so many things are versioned is that it **produces an invaluable record of incremental changes made and when they occurred**. The ability to inspect this log is super helpful when understanding why the value of a certain datapoint is what it is.

What’s more: the ability to navigate between different versions is a superpower that acts as a virtual form of time travel. It’s no surprise that every software bug report starts with the same question: _which version are you running?_

Data is one area where versioning is still in its relative infancy. Why is this? 

Well, data can be quite large — arguably larger than anything else in the digital world — and it is non-trivial to maintain multiple versions of something so heavy.

Non-trivial… but not impossible. 

As we’ll explore, a versioning system for any size of data can exist with the right data structures and abstractions in place to efficiently map data objects to the versions they are a part of.

In business-critical data environments, versioning is increasingly seen as a vital component and not simply a nice-to-have premium. Before we dive into why this is, let’s take a step back and define what versioning is in the domain of data.

Fundamentally **to version data means to create a unique reference for a collection of data**. This reference can take the form of a query, an ID, or also commonly a datetime identifier.

This general definition qualifies a range of approaches as being “data versioning.” It includes something like saving an entire copy of the data under a new name or filepath every time you want to create a version of it.

It also includes more advanced versioning solutions that optimize storage usage between versions and expose special operations to manage them.

We’ll discuss how these work in more detail in the [How Data Versioning Is Implemented](https://lakefs.io/data-versioning/#elementor-toc__heading-anchor-3) Section below.

Why is Data Versioning Important?
---------------------------------

Data versioning is important because it allows for quicker development of data products while reducing errors.

Accidentally deleted a petabyte of production data? Restoring a previous version of a dataset is a lot easier than re-running a backfill job that can take a whole day to complete.

Need to identify records that changed in a table without a reliable `last_updated` field or [change-data-capture](https://en.wikipedia.org/wiki/Change_data_capture#:~:text=In%20databases%2C%20change%20data%20capture,taken%20using%20the%20changed%20data.) (CDC) log? Saving multiple versions of the data as snapshots and querying for differences will do the trick.

As these examples show, minimizing the cost of mistakes and exposing how data has changed over time are two ways to increase the development speed of a data team. Data versioning is the catalyst making this possible.

### Taming Complexity in Modern Data Environments

Versioning data has always had value. But it has particular significance in modern data environments that are being asked to do more than feed internal reporting. In an increasing number of organizations, data supports a myriad of mission-critical business processes, which brings increased responsibility and complexity.

Nick Schrock, CEO of Elementl, states in an episode of the [MAD Data podcast](https://mad-data.simplecast.com/episodes/hello-big-complexity-is-your-modern-data-stack-ready) how he sees data environments evolving as a result of this shift (_minor paraphrasing and emphasis_):

> It is about the embrace of cloud technologies, the embrace of managed services, and as importantly, **the embrace of engineering practices to tackle and control that complexity**… Integrating software engineering practices throughout the data ecosystem is the way to manage Big Complexity and will dominate the next 10 years of development.

The software engineering practices referred to? Things like unit tests, integration tests, CI/CD deployment, and of course, versioning. **We see this theme continuing to play out and data versioning gaining adoption across the data ecosystem in the next generation of data platforms.**

Next, let’s look at a few ways we can implement data versioning, starting with basic forms and working our way up to more advanced solutions.

How Is Data Versioning Implemented?
-----------------------------------

There are several ways to implement data versioning. We’ll cover three instructive approaches below:

### Versioning Approach #1: Full Duplication

Have a dataset you want to see how it’s changing over time? One option is to save a full copy of it under a new location each time you want a version of it. This works best for smaller datasets with something like a daily versioning frequency.

![](data:image/svg+xml,%3Csvg%20xmlns='http://www.w3.org/2000/svg'%20viewBox='0%200%20554%20251'%3E%3C/svg%3E)

Versioning via saving a full copy of an example users dataset daily.

While this approach does create versioned data, it does so in the least space-efficient way. In the illustration above, any block that stays green is an example of a data object that hasn’t changed but is now duplicated across each version. 

Furthermore, code or queries that interact with this versioned data will be error-prone with the correct date value having to be manually hardcoded in different places.

Although not the most elegant solution, it is an easy way to get started versioning data.

### Versioning Approach #2: "Valid\_from/to" Metadata

A more space-efficient and incremental approach to versioning works by adding and maintaining two metadata fields in a tabular dataset, often named `valid_from` and `valid_to`. When updating a record in this dataset, we make sure to **never** overwrite an existing record. Instead, we append new records and update the `valid_to` field to the current timestamp for any record that would have been overwritten.

Besides being something you can implement in your own ETL scripts, it is also notably the approach SQL Server uses for its [Temporal Tables](https://docs.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables?view=sql-server-ver15) feature and what dbt uses for [dbt Snapshots](https://docs.getdbt.com/docs/building-a-dbt-project/snapshots).

![](data:image/svg+xml,%3Csvg%20xmlns='http://www.w3.org/2000/svg'%20viewBox='0%200%201024%20441'%3E%3C/svg%3E)

Using query filters to get the state of the Orders table on Oct. 17.

This approach works quite well for “time traveling” throughout a single collection of tabular data. However, it provides only one method of interacting with the versions — which is to add filters to queries on the metadata fields as shown above.

Yes, we can materialize the table as it was at various points in time, but this also what we are limited to doing.

### Versioning Approach #3: First-class Data Version Control

If the first two approaches can be summarized as “Let me add a bit of versioning to the data I already have”, now it’s time to change mindsets. Instead, we should think of versioning as a first-class citizen of our data environment. An inherent property of any data we introduce into the system.

To make this possible — as made clear by the limitations of the above approaches — we need to solve a few challenges.

1.  **Minimize the storage footprint of data versioning.** This means not creating copies of data objects that remain unchanged between versions.
2.  **Expose operations that let us interact directly with the versions.** Things like “create a version”, “delete a version”, “compare two versions” are examples you might think of off the top of your head.
3.  **Work the same over any scale of data, data format, and both structured and unstructured data.**

Now, these are not simple problems. And you probably won’t hack your way to a solution in an afternoon or even a weekend. 

One of the more popular approaches to solving these extends the [git version control](https://www.freecodecamp.org/news/what-is-git-learn-git-version-control/) model to data. We see this in projects like [lakeFS](https://docs.lakefs.io/), [DVC](https://dvc.org/), and [git LFS](http://atlassian.com/git/tutorials/git-lfs). Let’s take a closer look at how the open-source lakeFS solves the above challenges.

Borrowing useful abstractions from git, lakeFS lets you creates versions of data via **commits**, which in turn belong to **branches**. In essence, “creating a commit” is synonymous with “creating a version” _(Challenge #2)_. 

![](data:image/svg+xml,%3Csvg%20xmlns='http://www.w3.org/2000/svg'%20viewBox='0%200%201024%20574'%3E%3C/svg%3E)

Data Model of lakeFS to enable scalable versioning.

The above diagram shows the full relationships between the actual datafiles being versioned in lakeFS (bottom row), all the way up to the commit and branch abstractions exposed to users (top). 

The important concept is that duplication of datafiles is **minimized** between commits _(Challenge #1)_. This is depicted by the arrows going from one Metarange to multiple Ranges and the arrows from one Range to multiple objects.

An even more detailed look at these relationships can be seen below:

![](data:image/svg+xml,%3Csvg%20xmlns='http://www.w3.org/2000/svg'%20viewBox='0%200%20782%20592'%3E%3C/svg%3E)

Pointers between ranges and objects prevent data duplication.

What’s the point of all this? When it comes to time travel, we now have a super-easy way to navigate amongst the different data versions (_by using the unique generated lakeFS commit\_id in each commit_).

All of the commands available for interacting with the versions can be found [here](https://docs.lakefs.io/reference/commands.html).

Examples Using Data Versioning
------------------------------

Let’s give a clearer picture of how data versioning is useful in different contexts by walking through some examples.

### Data Versioning in Machine Learning

Say you work for a company that uses machine learning algorithms to enhance grainy video footage and identify objects. The users of the product use the enhanced footage for a variety of commercial uses.

![](data:image/svg+xml,%3Csvg%20xmlns='http://www.w3.org/2000/svg'%20viewBox='0%200%20700%20378'%3E%3C/svg%3E)

Image Source: https://dzone.com/articles/the-most-insightful-computer-vision-project

A new and improved algorithm is developed by the data scientists that improve the classification accuracy of its outputs – the enhanced footage. However, it is not possible to roll out the new algorithm to all users at once. Instead, for a period of time, we need to let users switch between the classifications of both algorithms.

We could save the outputs of both versions of the algorithm to different paths in the object store. For one algorithm with just two versions, you could get away with this “hardcoding filepaths” type of approach. When the numbers of both algorithms and versions increase though, developers will start to make a greater number of mistakes — forgetting to update the path of a version or losing track of what parameters were used for a particular version.

Incorporating data versioning raises useful abstractions (e.g. commit messages, branch names) to manage the outputs in a more sane manner.

### Data Versioning in Analytics

The foundational process of analytics is creating metrics that contain the logic to evaluate a business and user behavior. The logic for important concepts like “Active users”, “Sessions”, “Churn Rate” are often defined in SQL and calculated on a regular cadence.

A common problem in analytics is that a query that ran fine a day ago might start causing errors because of a change in the data. The most effective way to figure out what is causing the issue is to run the same query over the data **as it was when the error first occurred.**

Having a version of the data available at this time, [simplifies the debugging process](https://lakefs.io/solving-data-reproducibility/) and results in data errors getting resolved faster.

Data Versioning Best Practices
------------------------------

When implementing data versioning, here are some tips we’ve found helpful!

### Use TTL's to expire old versions

It may be the case that there’s no need to retain versions older than 30 days or a year, for example. But no one wants to play housekeeper and be responsible for deleting older versions of data. And so they start to accumulate.

If there’s a time duration you know versions are no longer relevant, a TTL ([time to live](https://en.wikipedia.org/wiki/Time_to_live#:~:text=Time%20to%20live%20%28TTL%29%20or,data%20is%20discarded%20or%20revalidated.)) policy is a great way to have older versions of data be deleted automatically. If using a versioning system that doesn’t support TTLs, periodically running a version clean-up script can achieve the same effect.

### Version semantically, not temporally

There’s nothing wrong with using a consistent daily or hourly cadence to create new versions of a dataset. In many cases, finding the version with a `created_at` time closest to when you want is good enough to find what you’re looking for.

What makes data versions even more meaningful is tying versioning to the start and/or completion of data pipeline tasks. And ETL script finished running? Create a version. About to send an email to your “highly engaged” users? Save a version of the dataset first.

This lets you include more meaningful metadata around your versions beyond the time they were created and lets you figure out what happened much faster if something goes wrong.

### Leverage versioning for collaboration

One of the challenges in data environments is to not step on the toes of your teammates. Often data assets are treated as a sort of shared folder that anyone can read from, write to, or make modifications to.

One way to avoid these problems is to create personal versions of the data when developing. This prevents the chance that a change you make inadvertently affects another member of the team.

There’s a noticeable movement in the data space to adopt the mindset of treating “[Data as a Product](https://medium.com/@itunpredictable/data-as-a-product-vs-data-as-a-service-d9f7e622dc55)”. We believe this is a positive trend for data orgs and it requires a leveling up of the way many data teams operate.

Development best practices like CI/CD, testing, and version control are features you need to be thinking about if you want your data team to confidently take on these types of projects.
