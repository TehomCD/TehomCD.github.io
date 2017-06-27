---
layout: post
title: "Tehom Releases New Blog: Thousands Dead"
date: 2017-06-25
---

Self-obsessed narcissist that I am, it occurred to me to start a dev blog series for Arx, documenting my struggles both in fixing technical issues, and in the more interesting parts of game design. 
I decided to try out [Jekyll](http://jekyllrb.com), so we'll see how that goes.

For my first post, I'd like to talk about a technical issue that plagued us for six months, and how we wound up fixing
it.

Evennia uses a clever django model called Attributes to store pickled data that when retrieved can become arbitrary python 
objects, and these are then associated with objects inside the game. This gives tremendous flexibility: if you want to say
an in-game object has a strength score of 5, you create an Attribute that might have the db_key of "strength" and stores an
integer of 5. Want to store a list of other in-game objects? It can serialize and store those too, no problem. So all was well,
and then Evennia decided to add another index that indicated, as a CharField, what type of objects these Attributes were attached to, 
as an optimization.

Everything broke. *Everything*.

A query before that would take a fraction of a second suddenly took 30 seconds long. It was a complete mystery why this would 
happen, and I was left with no choice but to put our production server on a different branch of Evennia that had the query
altered so things would be playable, and just handle merge conflicts whenever Evennia updated anything.

Eventually I decided to do more research into the issue. Testing was pretty painful to do, given how long the queries took, but
I was able to print out the raw SQL that django issues into log files from testing, and then had SQLite explain its query plan
for the different types of queries, with the index added or removed. I noticed something interesting - everything worked just
fine as long as the field was used as an __iexact query rather than an exact query. What gives?

So what I learned on that glorious day is that __iexact queries in django are LIKE queries, which probably won't use the index,
and sqlite uses one index per table, which its query planner is responsible for choosing. I also found out that when I originally
populated my database, I was on a windows machine that used sqlite version 3.6, and we had then later migrated to a Linux environment,
which uses 3.8. Sqlite changed their query planner in 3.8, so we were bitten by it doing an incredibly poor choice in the query planner
due to not running ANALYZE. It would choose to use a query plan that more or less resulted in it scanning the entire table of 200k rows rather than looking at a handful. After swearing for a solid ten minutes straight once I realized that's all that was necessary, I just ran ANALYZE, and the issue vanished completely.