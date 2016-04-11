---
layout: post
title:  "When your machine lies to you, Part 2: A conspiracy of database and ORM"
categories: java hibernate debugging db2 null fail usertype everybodylies
---

To quote the great debugging genius Dr House:

> Everybody lies.

We would like the tools we use to have useful, non-lying debugging mechanisms.
Unfortunately, these tools are built by programmers. Programmers, being a
subset of Everybody, also lie, and thus the tools they build often
have a somewhat fleeting affair with what we think of as truth.

Sometimes, these lies are accidental, or just a matter of not thinking things
through. Sometimes, it seems like the lies are almost intentional.

## A conspiracy of database and ORM

Some years ago, I was given the task of building a small [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)
application that was to serve as one of the back-ends for another, larger
application. At first glance, it was nothing out of the ordinary: JBoss, some
web services as the front interface, a DB2 database in the back, and JPA in
the form of Hibernate 3 inbetween, and just a bit of validation and business
logic. Not overly exciting, but fairly quick to implement, and the basic
functions were working and available to the consuming application in a
short time.

Some time later, the team building one of the consumers of our webservices
started complaining about data disappearing from their objects. They were
putting entries into a `Map<String, String>` in one of the objects.
Upon loading the containing object again, some of these map entries were missing.

I found this very odd, because we were doing zero processing on this map. We
simply handed the whole object off to Hibernate to store it in the database,
and at some later point loaded it again, the same way.

With a closer look, we found that the missing entries all had an empty `String` ""
as their values. Now one might question whether storing empty `String`s is a good
design, but it's certainly legal. And in any case, our backend should not eat
any data.

### Data! \*omnomnom\*

Since I had already had a bad impression of DB2, I decided to investigate in there
first. Open up a SQL console, experiment a bit... and of course. DB2 goes and says
"Eh, between friends, what's the real difference between `''` and `NULL`?" Wherever I
wrote an empty string, DB2 gave me back a `NULL`. So much for data integrity.

While this was good to know, it didn't yet fully explain what was going on - we weren't
seeing null values, we saw map entries disappearing completely. For that to happen on the
DB level, entire rows would have to vanish, and even though DB2 was lying about
my empty Strings, it wasn't quite eating rows.

The next logical layer up is Hibernate. Let's play with that a bit. Writing a key-value
pair with an empty `String` as the value into a local (non-DB2) database simply works,
as does reading it. Okay, so let's try using a null value. It does not show up in the
database.

    myBean.getMap().put("foo", "bar"); // this gets written to the table
    myBean.getMap().put("huh", null); // this does *not* get written

Grrr. Granted, the `java.util.Map` interface doesn't outrirght _require_ implementations
to accept null values. But it is also not prohibited, and I think any framework should
not impose additional restrictions unless really necessary. And if it does, it should
give an error as soon as possible instead of _silently eating data_.

But this is still not quite what we're seeing, since we're not actually writing `null`
values. We're writing empty `String`s that DB2 helpfully turns into `NULL`s. Surely
Hibernate doesn't filter out things that are already in the database, does it?

    insert into bean_map (bean_id, key, value) values (123, 'foo', NULL);
    ...
    MyBean bean = session.get(MyBean.class, 123);
    bean.getMap().containsKey("foo"); // this is false. wtf.

### Lies, damn lies, and opinionated frameworks

So yes, Hibernate actively ignores a row that is already in your database and will
pretend it is not there. A bug? No. This is deliberate.
[Quoting Gavin King](https://hibernate.atlassian.net/browse/HHH-772?focusedCommentId=18944&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-18944):

> Now ignore null elements during loading.

Why, you ask? [Well...](https://hibernate.atlassian.net/browse/HHH-772?focusedCommentId=19005&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-19005)

> A "sane" schema does not have "null values in a map". please.

This may or may not be true, I think `null`s are a bad idea in general, but:
Out of all bad practices in databases, Hibernate chooses to actually enforce not having this one. Argh. And yet they go to great lengths to support all kinds of crazy stuff elsewhere.

King argues that having a `null` value is indistinguishable from not having a key/value pair
at all, because `Map.get(key)` will return `null` in either case. That is, admittedly, not
entirely wrong. If we had simpler maps in Java, I might even agree. However, this argument
does not take into account that `java.util.Map` is a _huge_ interface.

There is, for example, the `containsKey` method. As shown in the example above, with this
the two cases of having a key with a null `value` ("I know this key, but I have no value
for it") and not having the key at all ("I have never heard of this") are easily
distinguishable. Hibernate could have openly declared that they don't support this and
simply thrown an exception. Instead, the implementation _deliberately lies to you_.

So this is where we are: Hibernate doesn't want me to store `null`s in a Map, and I'm not
even trying to do that. But then DB2 decides to lie and turn my empty Strings into `null`s.
Hibernate wants to get in on the action and lies as well, completely hiding existing rows
in the map table. Sometimes I really hate computers.

But of course I can make them do what I want, even if it involves disproportionate effort
sometimes.

I tried using a Hibernate UserType to mask the empty `String` as something like "$empty$"
so DB2 wouldn't eat it. Then I tried hooking into Hibernate 3's event listeners. Long story
short: Nothing worked. A bit of hunting around in Hibernate's sources revealed why:

    // PersistentMap.readFrom(...):
    if ( element!=null ) map.put(index, element);

This is at almost the lowest level, where `PersistentMap` loads its contents from
the JDBC `ResultSet`, going through a `UserType` if applicable.
There's no way for any user code to prevent this line from
helpfully discarding entries where the element is null, even if the key (here called
`index`) is present.

While the `UserType` can stop empty `String`s from going into the database by converting
back and forth between `""` and `"$empty"`, the same thing does not work for actual
`null` values. Since I can't rule out `NULL`s showing up in the database, this is not enough.

### Doing it the hard way

So, in the end, I fell back on doing it the hard way. I wrote a wrapper implementation
of `java.util.Map` that would intercept `null`s and empty `String`s and replace them with
`"$null"` and `"$empty"`, respectively. Hibernate would only get to see the inner map,
so it would never see a `null`, and DB2 would thus never see an empty `String`.

Any code outside the DB access layer would only see the wrapped map, which would behave
normally, so `null`s and empty `String`s were simply there.

Additionally, a `UserType` intercepts actual `NULL`s in the database, "tunneling" through
Hibernate's stupidity as `"$null"` so they can then be decoded to `null` by the wrapper
`Map`.

This setup is not pretty, but it works. The ugliness is compounded by the fact that
`Map` is a gigantic interface. If you want to implement it, you also have to write a
`Set` for `entrySet()`, another `Set` for `keys()` and a `Collection` for `values()`.
You can't just instantiate e.g. a `HashSet`, because changes to the `Map` need to be instantly
visible in the other collections and vice versa. Hooray for mutable collections, I guess.

### Lessons learned

1. Don't use a database that will silently modify your data.
2. Java's collection interfaces are useful, but way too large.
3. Unless you have a really good reason for it, do not use an ORM. They seem useful at first glance, but add a lot of incidental complexity that _will_ bite you.
