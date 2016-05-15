---
layout: post
title: Doing autocompletion with Redis
---

## Redis powered search box autocompletion

As a part of my [izadeel](https://www.izadeel.com/) project, I needed to provide our users with a simple to use search.  

My first take at the problem was a simple solution: build a fulltext search index on our MongoDB and query it using simple expressions.  
This worked well enough but it soon revealed a problem:  

**Fulltext search is dumb!**  

Don't take me wrong: MongoDB's fulltext is **very** powerfull. But it's also very dificult to fine tune. You want your users to have many results, as many as possible, but they have to be **meaningfull**.

Since we're indexing over 80.000 products, it was obvious that returning contextualized results would be hard, specially when we couldn't guess what the user was actually looking for.

The answer was using autocomplete to help steer users towards what they're looking for and still rely on fulltext search to provide answers.


My requirements:

  1. Use autocomplete  
  2. Be fast!
  3. Query MongoDB using fulltext search  
  4. Provide incremental suggestions based on the full product title  
  5. Provide suggestions based on database frequency (**Chi**cken vs **Chi**ldren books)  

### 1) Autocomplete
I wasn't set on reinventing the wheel so I grabbed [jQuery-Autocomplete](https://github.com/devbridge/jQuery-Autocomplete) and installed it.  
By now I already had a working suggestion system based on MongoDB's results so adapting my Ajax endpoint was a preety simple and straightforward.

### 2) Be Fast!
Speed is of the utmost importance in a suggest as you type system. So I decided to base the system on Redis.

### 3) Query MongoDB using fulltext search  
The MongoDB fulltext search works very well but needed to be constrained somehow. I hoped to achieve this buy steering the users choices and still relly on MongoDB.


### 4) Provide incremental suggestions based on the full product title  
My users search for products.  
They might be interested in results of the same brand or of similiar quantities but the main search point is the product title.  
Building on the idea of steering the users, the autocomplete will be based on matchin the start of the input to the start of the product title.


### 5) Provide suggestions based on database frequency (**Chi**cken vs **Chi**ldren books)  
If a user types **"Chi"** we should provide suggestions ordered by the largest number of results first. This meant making sure that **Chi**cken would be presented before **Chi**ldren, for instance.



## The solution

Redis has a data type called [Sorted Sets](http://redis.io/topics/data-types#sorted-sets), where every member of a Sorted Set is associated with score.   

This makes it perfect for my intended implementation where the Sorted Set members are the representations of the user input, the keys are the possible hits and their score is the database frequency.   

So, for a query of "chi" we would lookup the "chi" member of the Sorted Set and query its keys by descending score:

{% highlight lua %}
ZREVRANGEBYSCORE 'PREFIX:chi' '+INF' 0
 1) "chicken"
 2) "chia seeds"
 3) "children's books"
{% endhighlight %}

I limit the results displayed back to the user in the ajax endpoint (currently it's displaying 5).


Having defined this solution, all that was left was populating the redis db with results based on my product list.  
This was done with a little python help: a script that iterates every product title and then sequentially increments its path score on the redis db, through a ZINCRBY:

{% highlight python %}
for product in products:

    title = product.title
    title = unidecode(title)

    ## Split this line into words
    words = title.split(" ")

    ## Iterate each word, constructing new phrases
    for word_idx in range(0,len(words)):

        # New phrase with current words
        new_line = " ".join(words[0:word_idx+1])
        line_length = len(new_line)+1

        start = 1 if word_idx < 0 else len( " ".join(words[0:word_idx]) )

        ## Deconstruct current phrase and add to redis
        for idx in range(start,line_length):
            line_to_add = new_line[0:idx]
            redis.zincrby(KEY+line_to_add,new_line,1)


{% endhighlight %}

Now, bear in mind that for my 80.000 products, this generated around 1.5M records in redis.
But the end result is as fast as you get, with **results being served around 60ms!**

![Network log of autocomplete accces](/assets/images/redis_autocomplete.png){: .center-image }

