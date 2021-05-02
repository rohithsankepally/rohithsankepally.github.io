---
layout: post
title: Writing Custom ES Plugin
blog : true
published: false
date: April 04, 2021
comments: true
---

In this blog post, we will discuss about Elastic Search plugins and its usages. Additionally, we will also implement a simple elastic search plugin and see it in action.

# What are plugins?

Elastic search supports plugins which allows us to enhance its core functionality. Through plugins we can add custom functionality without messing with the source code of elastic search. There are a wide range of plugins already available. Some of them are part of elastic search source code while some exist as standalone plugins that have their own licensing. To find out more about plugins go through https://www.elastic.co/guide/en/elasticsearch/plugins/current/intro.html.


# Why should we write a plugin?
Some of us might think, is it really necessary to write a plugin. Can't we just implement logic in application instead?

Yes, you can implement any kind of logic in your application code. But you might end up writing a lot of boilerplate code and also increase complexity of your application.

A plugin with a custom functionality, helps you reduce the amount of application code and improves usability. Computation will be entirely carried out on the elastic search server, thereby reducing the complexity on application side. Moreover, if there is a change in logic, we just need to update the plugin and restart our elastic search cluster without disturbing the application.

# Writing a custom plugin : Currency Converter

We are going to implement a plugin which performs currency conversion. 

## Possible Use Case
Let's say we have e-commerce data of orders that are successfully placed by customers stored in elastic search. Every order has an associated amount paid by the customer in a given currency. Given such data, let's say one would like to see the total amount of all the orders placed within a given month in USD based on current conversion rate.

## Data & Query Pattern

Below is the sample data for two orders.
```bash
{
    "timestamp" : 1618625586000,
    "amount": 123.4,
    "currency" : "INR",
    "userId": 1546,
    "orderId": 1249393833
},
{
    "timestamp" : 1618625587000,
    "amount": 234.5,
    "currency" : "EUR",
    "userId": 1544,
    "orderId": 1249393834
}
```
Now lets say, we would like know the total amount of all the orders. We can do a simple elastic search sum aggregation on the `amount` field and get the total amount. The query for this is given below.
```bash
POST /orders/_search?size=0
{
    "aggs" : {
        "total_amount" : { "sum" : { "field" : "amount" } }
    }
}
```

The output of this as given below 
```bash
{
    "aggregations": {
        "total_amount": {
           "value": 357.9
        }
    }
}
```

But it doesn't totally solve our problem as the orders have been placed in multiple currencies and a simple addition of values would not work. We want the total amount in a specified currency.  

In this case, if the specified currency is say USD, then the expected total amount is as follows

```bash
Total Amount = 123.4 / 74 + 234.5 * 1.20
```

## Currency Converter 



