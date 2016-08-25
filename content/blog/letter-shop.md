+++
date = "2016-08-24T18:30:50+02:00"
draft = false
title = "letter shop"

+++

# What
This post describes a made up application - Letter Shop, why I created it and how it was done.

# tl;dr 
* [TCK code](https://github.com/lukeindykiewicz/letter-sh)
* [API code](https://github.com/lukeindykiewicz/letter-shop-api)

# Why
Letter Shop is an attempt to make comparison of concepts possible, of course in a very practical way. I wanted to have some api/tests/problem that I can implement with different approaches (like using monads, using event sourcing, etc.). Each approach is best in a particular situation and simple examples, "hello worlds" show exactly those cases. Then it comes to a real world example and sometimes it's not so sweet. There is [TodoMVC](http://todomvc.com/) and [Todo-Backend](http://www.todobackend.com/), but I wanted to have more control over the problem, to make it evolve according to the concepts I'm exploring. 
There is also another reason, I want to have this api/tck as my kata for further exploring of programming concepts and languages.

There are two main parts of Letter Shop:

* API - this is yaml (swagger) description of endpoints available in the Letter Shop
* TCK - tests for all main features, green means a proper implementation of an API

# How

## Idea
I wanted to model something that is a little bit more complicated than todo, but also is easy enough to not loose interest in building it and be able to quickly explain what it does. The main goal is to train/check new programming ideas, not explore business domain. So I took a shop case, as it is a well know situation for everyone. We have cart, things we want to buy and some price. I didn't want to implement a lot of different products in the shop, so I decided that the shop sell letters, like: a,b,c. Sounds stupid, but for this purpose suits very good.

## API
Api ([swagger ui](http://lukeindykiewicz.com/letter-shop-api/#/default)) contains just few endpoints - basic features of Letter Shop. We can:

* add letters to the cart
* replace existing cart with new letters (also clean cart)
* get cart info
* check cart (just checks the price of current cart)
* checkout cart (price, receipt id and cleans cart)
* set price for given letter (default price is 10)
* get all receipts from the system

Yes, everything is public, no user/admin login. It has to be simple. 

This is pure CRU (create, read, update) stuff, so I decided to add some price logic. So, as in every shop, there are promotions:

* Three for two (only for letter 'a' and 'X') - when you buy three letters you pay only for two
* Promo code to get 10 percent discount

Read [CheckCartPriceTest](https://github.com/lukeindykiewicz/letter-shop-tck/blob/master/src/test/scala/lettershop/CheckCartPriceTest.scala) to check promotion rules exactly.

## TCK
[Tests](https://github.com/lukeindykiewicz/letter-shop-tck/tree/master/src/test/scala/lettershop) are written in Scala and specs2. They describe high level functionalities for Letter Shop.

# What is missing
I decided to put Letter Shop online, although I don't know what should be there to make it 1.0 ready. As I mentioned earlier this will probably evolve. On the other hand I think it is enough to start implementing Letter Shop. Things that are planned:

* add validation, as this is always a case in business applications
* think of a way to run tests in some CI (tck is a separate project).
* write free monads implementation

# Implementations
Very simple implementation is already there, just to test the tests. [Check it](https://github.com/lukeindykiewicz/letter-shop-classic-scala) if you want.

# Summary
This is a first blog about Letter Shop, the definition that is going to be implemented and descibed in future blog posts.
