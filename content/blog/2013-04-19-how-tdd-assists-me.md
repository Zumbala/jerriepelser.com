---
date: 2013-04-19T00:00:00Z
description: |
  My take on how I use Test Driven Development to stay on track and keep moving forward with my work, even when I am offline.
tags:
- fakes
- mocks
- tdd
- unit tests
title: How TDD assists me
url: /blog/how-tdd-assists-me/
---

There are a lot of reasons why people suggest you should do Test Driven Development, and even though I was not a total convert right from the beginning, the practice is growing on me.  My situation is a bit unique from probably most developers out there, so while I buy into a lot of the common reasons why you should do TDD, there are a few unique advantages I gain from it.

Maybe a bit of my background first.  I have been a contractor most of my life, which allowed me to work on a whole range of different projects.  In the last year I work on a team for a large South African organisation who decided to make the shift to Agile methodologies, and the team I worked on was used as the guinea pigs to see if it could be introduced into the wider organisation.  Towards the end of last year I decided to make a big change: I sold all my belongings and now I am traveling the world writing my own software.  (You can read more about me travels on my blog [Jeremia was a Bullfrog](http://www.jeremiawasabullfrog.com)).

Even though I am now working by myself and for myself, a lot of the things I learned from Agile carried over and I am using it on a day to day basis.  One of those are TDD.  So what are the biggest advantages I derive from TDD?

## 1. Procrastination Buster

Like I said, I travel the world and write code.  I try and do this from exciting places, which means that I have a lot of things which can distract me from doing my work.  It is oh so very easy for me to procrastinate and find some other, much more exciting things to do.  So I use TDD as a procrastination buster.  Whenever I don't want to work (but know I have to get some work done), I just start writing a failing unit test.  I then go through the process of getting the unit test green and move on to the next one.  Before I know it an hour of two has passed and I have done a lot of work to move my software product forward. It works so well for me that I sometimes have to force myself to get up from my computer and go see the world outside.

## 2. It enables me moved forward when I am offline

Working in exotic locations has its advantages, but one big disadvantage for me is that the internet connectivity is intermittent.  Right now for example I am working on Koh Tao, and island of the coast of Krabi, Thailand.  As wonderfully quiet as it is here my one big problem is that the internet connectivity is really bad.  This is an especially big problem for me as the application I am developing is working with social media (Facebook, Twitter etc.), so without internet connectivity this poses a bit if a problem for me.

How does TDD help me you may ask?  Mocks and Fakes, that is how.  I can develop large portions of my application using just unit tests, mocks and fakes.  It is of course not 100% foolproof, but I have gone days where I wrote massive pieces of functionality without ever connecting to the internet just by using mocks and fakes.  And each time, once I got to the point of connecting to the internet to test the functionality end to end, it worked basically without any problems.

## 3. It helps me design my application

This you will see in a lot of places where people list the advantages of TDD, but for me it is the one which was probably the most noticeable from the beginning and which I still notice every day.  For me TDD helps me to design a better application.  I feel that by writing the unit tests first I get a better understanding of what the actual code should do.  I also feel that it helps me design better classes, or when I notice a problem in my design I can go back and refactor without fear because I know the unit tests have go my back.

Of course I go along with a lot of the other advantages which people always hold up as reasons for why you should do TDD.  These 3 are just the ones that I find resonates with me the most in my personal situation.

## Sign Up to notify when my application launches!

I am nearing the date when my application launches, so head on over to [www.oneloveapp.com](http://www.oneloveapp.com) and if it sounds like something you may be interested in, please subscribe to the email list to be notified when the application launches.