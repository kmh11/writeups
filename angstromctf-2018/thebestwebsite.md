# The Best Website, 230 pts

You are provided with a seemingly useless website. Upon further inspection, it is a legitimately useless website. However, in the source of index.html you see a comment directing developers to record their changes in `log.txt`. Visiting `log.txt`, you see that a super secret flag was added to the database, and there is a timestamp. This will be important later.

Continuing your inspection of the website, you see it makes a network request to `/boxes?ids=<id1>,<id2>,<id3>`. From either the hint or previous knowledge, you can determine that these are MongoDB object ids. Googling what makes up a MongoDB object id, you find [how it is made](https://docs.mongodb.com/manual/reference/method/ObjectId/). The machine and process ID are shared, the counter can just be incremented by one, and the timestamp can be gotten through [this useful website](https://steveridout.github.io/mongo-object-time/) (be careful of time zones though).

After reconstructing the object id, substituting it for one of the current ids gives the flag.
