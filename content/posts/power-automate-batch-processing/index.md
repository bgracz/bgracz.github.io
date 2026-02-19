+++
date = '2026-01-12T08:55:13+01:00'
draft = false
title = 'Processing in Power Automate >5000 rows'
categories = ['Dev', 'Power Automate', 'Tutorial']
tags = ['powerautomate', 'dev', 'automation']
cover = { image = "cover.png", relative = true, alt = "Power automate - header" }
+++
# What we want to achieve?

**Power Automate** from Microsoft is a powerful automation tool. It allows you to really simplify your work by creating a flow that will process large amounts of data. Thanks to the *schedule* option, all the work can be done at night, and in the morning, when you sit down to work, you have the finished result.

The problem is that right from the start, we encounter a significant limitation that, at first glance, may completely disqualify the further use of Power Automate—the limit on the amount of data we can process. While this is not so critical in the case of an **SQL** database, for **Excel**, which remains the most popular *database* in most companies, it is a disqualification right from the start.

But is that really the case? Read on to find out more, as we break down this problem and try to find a solution!

# Where does this restriction come from?

**Power Automate** limits the “List rows present in a table” (Excel Online) action to 5,000 rows by default due to pagination policy and license limits to prevent Microsoft Graph API overload and excessive resource consumption.
In practice, only **256** rows are loaded by default, but we can increase the limit to **5000** in the pagination settings. A higher value will cause an error, and the result of the action will remain the first 5000 records.

# So what are we going to do about it?

Let's take a look at a graph that illustrates an example flow (simplified, of course, without data processing; we only want to understand the logic).

![Visualization of process](/posts/test/diagram.png#center#50)

1. We have a trigger that starts the flow—an action has occurred, or we have simply set it to run at a specific time.

2. We create an integer variable with an initial value of 0.

3. Right after the **SkipToken** variable, we create another one, of type Array, with an initial value of *[]* and call it **resultsArray**. This variable will store all our data from each batch.

4. We will iterate the loop **n * 5000** until the amount of returned data falls below 5000 elements.

```ts {linenos=true}
less(length(body('Get_items')?'value'), 5000)
```

and in the loop:
- the **Data** action, i.e., connecting to our file, from which we will read. We must remember to find the **Skip count** option in **advanced mode** and assign it a value from the **skipToken** variable. The easiest way to do this is as follows:

```ts {linenos=true 
variablem('skipToken')
```

- after the first loop, where we collected the first batch of data, we need to calculate the point at which we will start the next loop. So we increase the value of **skipToken** by another 5000.

```ts {linenos=true}
add(varaibles('skipToken'), 5000)
```

- we don't want to lose the data collected in the current run, so we “send” it outside the loop - we combine what we currently have in resultArray with the data we have just obtained - **MergeCurrentLoopWithResultsArray**

```ts {linenos=true}
union(variables('resultArray), outputs('newData')?['body'])
```

- and here is the correct dump to **resultsArray** from the **UpdateResults** step

```ts {linenos=true}
outputs('MergeCurrentLoopWithResultsArray')
```

# What else is worth to do?

I definitely recommend modifying the **Timeout** value for the **Data** step/action, i.e., where we refer to our **Excel** file.
Keep in mind that in this approach, we focused on loading >5000 *rows*, completely ignoring the number of columns, which can have a significant impact on data processing time. 

How does it work?
`PT1H` is a standard ISO 8601 format meaning 1 hour (Period/Time: 1 Hour). It is used in Power Automate as the default timeout for the *Do until* loop - it means the maximum time the loop can run - after 1 hour, the operation ends automatically, even if the condition is not met (60 iterations OR PT1H by default).

```ts {linenos=true}
PT10M    // 10 minutes testing
PT2H     // 2 hours production  
P1D      // 1 day (maximum for some actions)
```

I recommend experimenting with this. **PT2H** seems to be safe for starters.

Also, remember to add a step at the end that will save our **resultsArray** to an *xlsx*, *csv*, or my favorite - *json* file, so that you don't lose your data and can continue working with it!



