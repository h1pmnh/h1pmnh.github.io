---
title: "Advanced sqlmap Case Study"
date: 2022-05-06T23:07:00-04:00
draft: false
tags: ["sqlmap", "sqli", "advanced"]
---

## Summary
Many new bug bounty hunters will blindly rely on the output of tools to magically find them bugs. As most experienced hunters know, the key to long-term success is to understand how to effectively use the many great tools and fine-tune these tools to achieve results in the form of valuable, challenging bugs.

Since I joined the Synack Red Team, I have been digging into `sqlmap` and the intricacies involved in finding SQL injections "in the wild". This will be the first of hopefully several posts on how to use `sqlmap` to work around challenging real-world scenarios where SQL injections exist. I hope you find this useful in your research!

## On Manual Testing
In almost all cases I **never** run `sqlmap` without first manually confirming the presence of a vulnerability! Don't be an irresponsible hunter - do your research manually or with low-impact scanning first!

## Scenario
In this scenario I was able to find a SQL injection vulnerability through source code auditing of the application I was testing. The application took a certain parameter and issued 2 SQL queries with it. The first query looked like this (partial query):

```
... select OID from pg_namespace where nspname='{parameter}' ...
```

If this query succeeded (returned a row), the code would issue a 2nd query as below:

```
select distinct object from {parameter} ...
```

If the first query did not return a row, the code would instead throw a Python error:

```
AttributeError: 'NoneType' object has no attribute
```

So this means that in order to pass both queries the `{parameter}` needed to be a valid string in the first query _and_ a valid table or view name in the 2nd query! Unfortunately this is inherently incompatible with the need to be able to construct a valid boolean true/false expression. I had to find another approach.

## The Approach
I determined that it was not going to be possible to get a payload that would successfully pass both queries, therefore I determined that I had to construct a payload that would satisfy the first query enough to generate a true/false result. "True" would mean the code would successfully execute the first query, and fail in the 2nd query. "False" would mean the code would successfully execute the first query, but because a row was not returned from the query, it would generate the `AttributeError` message. This got me set up with the "True" and "False" conditions I needed for `sqlmap` to work properly:

 * "True": Error message contains `select distinct object from`
 * "False": Error message contains `AttributeError: 'NoneType'`

## One other Wrinkle
Because of the nature of the code involved, the SQL injection payload needed to be placed _in the middle of the query parameter string_ in a specific location. Let's say the query parameter looked like this:

```
param=canary.bluebird
```

The position of the SQL injection payload had to be here:

```
param=canary.bluebird
          ^^
```

Basically, it had to be placed before the `.` character, in the middle of the string, in order for the first SQL query to successfully execute.

## Validating the Approach
I tested this approach by the following query:

```
param=`cana'||(select/**/'r'/**/where/**/1=1)||'y.bluebird`
```

I confirmed this corresponded to the "true" result (getting past the first SQL query and a syntax error on the 2nd). Changing the `1=1` to `0=1` confirmed the behavior of the "false" result (`AttributeError`). Great!

Now, on to getting `sqlmap` to follow this same manual approach.

## A Couple more Wrinkles
A few other wrinkles came up in my manual testing:

 * Both "true" and "false" resulted in an error page with HTTP status `500`
 * The URL parameter had to be issued without URL encoding (i.e. `'` instead of `%27`)
  * Because of this, the inclusion of space characters would break the request, so they had to be replaced with comments `/**/`

I had to make sure that `sqlmap` followed these as well.

## Wrangling sqlmap
Let's get the easy stuff out of the way first:

 * Don't encode URL parameters: use the `--skip-urlencode` flag 
 * Change spaces to comments: use the `--tamper=space2comment` flag to invoke this tamper script
  * Tamper scripts modify the payload _before_ it is sent to the target site!
 * Match HTTP code `500` as part of the "true" result: use the `--code=500` flag
  * As an interesting side note: without this, `sqlmap` will assume a non-2xx status code is a failure (neither true nor false) and will ignore the result

Now we need to get the positioning of the payload correct. We can use the `--prefix` and `-suffix` flags for this as follows:

```
--prefix="cana'||" --suffix="||'y.bluebird"
```

If you are following along, you might wonder where the `r` character in `canary` went. Remember the injection point needs to be before the `.` in the valid parameter string. Remember a valid query "true/false" will look like this:

```
param=`cana'||(select/**/'r'/**/where/**/1=1)||'y.bluebird`
```

You can see the `r` is being returned from the inner `select` statement if the `where` clause evaluates to "true". If it evaluates to "false", no value is returned, and we know that the parameter value of `canay.bluebird` (no `r`) will return 0 rows. This type of SQL injection corresponds to the `sqlmap` payload with "original value" in the name.

 * Tip: `sqlmap` has two types of payloads: one where the boolean condition returns the original value inline in the query, and the other where the boolean result is _appended_ to the original value (the classic `AND 1=1` type of payload)

In this case, trying to get an `AND 1=1` type payload in the middle of our parameter would make no sense at all! Therefore we're looking only for blind boolean payloads that use the **original value** in the constructed query string.

The final piece of the puzzle is helping `sqlmap` determine a true vs false result. In this case we use the `--string` parameter to suggest to `sqlmap` text that is guaranteed only to appear when the boolean value is true. In our analysis we determined the displayed error message will contain the string `select distinct object from`.

Putting it all together, the final `sqlmap` command line looked like this:

```
...&param=r -p param --prefix="cana'||" --suffix="||'y.bluebird" --tamper space2comment --level=3 --risk=2 --string "select distinct object from ucon_p" --code=500 --dbms PostgreSQL --skip-urlencode --no-escape --no-cast --banner --technique=B
```

With this combination of parameters I was able to translate my manual confirmation of the SQLi with the automatic data gathering magic of `sqlmap`

## Feedback?
Hopefully this article was helpful in outlining how to reason through a complex SQL injection vulnerability. If you have any comments or want to collaborate on a report feel free to DM me on Twitter, I'm always open to collaboration with trusted people.

## Links
A couple other resources that cover this sort of advanced `sqlmap` content well:

 * [https://cybr.com/ethical-hacking-archives/sqlmap-cheat-sheets-to-help-you-find-sql-injections/](https://cybr.com/ethical-hacking-archives/sqlmap-cheat-sheets-to-help-you-find-sql-injections/)
 * [https://thegreycorner.com/2017/01/05/exploiting-difficult-sql-injection.html](https://thegreycorner.com/2017/01/05/exploiting-difficult-sql-injection.html)

