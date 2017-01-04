# Episode Zero Plan

## Precall

- Talk about intro and overall plan
- Please help me not monologue too much - interject questions and comments. Change of voice!
- I'm going to ask you "why validate in the frontend" and "why in the backend"

## Intro

Intro everyone:
  - Chris Aquino mainly front end, co-wrote "Front-End Web Development: The Big Nerd Ranch Guide" with Todd Gandee
  - Jay Hayes mainly back end
  - me too but I'll be playing a DBA today

- Podcast is an experiment.
- Idea: we may have some informal chat about development, but each episode should includes a concrete explanation of a topic. I personally enjoy podcasts where I learn something, so that's what I wanted to make.
- I'm thinking the topic will typically be brief, but I have kind of a big topic today, so let's jump right in.

## Topic Setup

- Topic: data validation and database constraints.
- We'll talk about when you need constraints and when you *should not* use them.
- I'm gonna cover the basics and some advanced stuff like deferrable exclusion constraints using partial indexes.
- We'll be talking about PostgreSQL; your database may vary.
- The goal here is to get the concepts across. If you understand those, you can look up the exact syntax later.

So to get started:

- Managing data is most of what your code does.
- Some of that data is required. Eg ordering pants: we might LIKE to know how you heard about ExcellentPants.com, but your order actually isn't valid if we don't know what size you want.
- Developers sometimes make too many assumptions in validation - eg everyone has a first and last name. Google "falsehoods programmers believe"
- But still - we gotta validate some stuff. 
- A typical web app has some frontend code - HTML, CSS, maybe JS - some server-side app code, and a database
- So where in the stack should we validate stuff?
- Chris, why validate in the front end? (basically for instant feedback, right?)
- Jay, you're the backend guy. If the frontend validations pass, are you satisfied? (Nope - NoScript, curl)
- So once you've both validated, are we done? Probably not.

## The Rails uniqueness Example

- Let's talk about uniqueness validation. I'll use Rails as an example, but there are frameworks in (at least) PHP, JavaScript and Java that have the same pitfalls.
- Say we're building a social network for people who own ferrets. Everybody needs a unique username, but clearly there are some usernames that will be highly coveted.
- Most Rails devs know that `validates :username, uniqueness: true` isn't enough. You need a unique index in the database. But why?
- The uniqueness validation does a `SELECT` for users with that username. If it (and other validations) pass, we `INSERT`
- The problem is timing. If you're running a production Rails app, you probably handle multiple requests at once.
- Eg, multiple unicorn processes, multiple Puma threads, multiple Heroku dynos, or whatever.
- So two app instance can do the `SELECT` simultaneously, find no conflicts, and both `INSERT` a user with username "ferrets_beuller"
- Or 10 app instances, or 1000! There is no limit to how wrong you can be.
- There is a paper on this called "Feral Concurrency Control". The authors examined popular open-source web applications, found that they mostly didn't use unique indexes, pounded them with concurrent requests, and showed how often the app code failed. Even in this stress test, app validations prevented most (but not all) duplicates. And in the real world, if somebody has already claimed the username you want, it's *likely* that they claimed it more than 10 milliseconds ago, and the validation will actually fail as it should. So this is a real-world problem, but you might not run into it unless you have enough request load.

But:

- Weird thing: if you look at your Rails log, it actually looks like Rails is doing enough to guarantee this will work.
- On `user.save`, Rails starts a db transaction, runs all validations, and if they pass, does an `INSERT`
- So that looks like: start transaction, `SELECT` users with this username, and if no matches, `INSERT`
- That seems like it should work, right? The race condition is: you `SELECT`, I `SELECT`, you `INSERT`, I `INSERT`
- You might think that if you're in a transaction, nothing can happen between the `SELECT` and the `INSERT`.
- And that's true *sometimes*
- Database transactions can have different "isolation levels". This means "how shielded is this transaction from stuff happening in concurrent transactions?"
- If you use something called "serializable" isolation level, the Rails validation thing should work
- PostgreSQL would tell me, "hey, somebody else was running a transaction at the same time as yours, and they ran `COMMIT` first, and theirs MAY conflict with yours. Try again? (literally: 'HINT:  The transaction might succeed if retried.)'"
- But: the "MAY conflict" part is kind of annoying. It gives false positives.
- Retrying is a bit annoying IMO
- Neither Rails nor PostgreSQL defaults to serializable transactions, and using them is a bit of a pain
- Apparently there's a bug in the way PostgreSQL does them right now anyway. I think they can fail at high concurrency, but I'm not sure I understand the bug correctly. It was mentioned in "Feral Concurrency Control" and was filed by the author.
    ALTER TABLE users ADD CONSTRAINT unique_username
    EXCLUDE USING gist (username WITH =);
- So anyway: the Rails approach COULD THEORETICALLY work, but creating a unique index is way easier than figuring out transaction isolation levels

## The General Idea of DB Constraints

- *Only* the database knows what data it has *right now*.
- So any validations that depend on its current state can only be reliably done by the database.
- Examples:
  - Does any user have this username right now?
  - Does post 5 exist still right now, before I comment on it?
  - Does this user's account have enough money to cover this purchase right now?
  - Is this rental property reserved for June 8 right now?
  - Does this store have a manager assigned right now?
- Some of these situations could be handled without constraints. Eg, you can use locks - "nobody else can access this row or this table right now", then do your `SELECT` and your `INSERT/UPDATE`. But fundamentally *we need the database's help* to guarantee consistency in all these cases

## Types of constraints!

Using the term loosely and stealing from my blog post (http://nathanmlong.com/2016/01/protect-your-data-with-postgresql-constraints/). 

### Level 0: column types and limits.

Can't insert "foo" in a date column, or an entire book in a VARCHAR(128)

Note: I swore off MySQL when I saw that, among other weird things, it would truncate strings AND numbers to make them fit. I think that nowadays it won't do that in its default configurations. PostgreSQL will definitely yell at you.

### Level 1: NOT NULL.

(Jay has feelings about this one.)

- This can be done just as well by application code; it knows if it has a `first_name` in memory or not. BUT it can be bypassed.
- Eg, `update_attribute` and `save(validate: false)` bypass validations.
- Also, another application or a SQL query wouldn't use your validations
- The fact that `NOT NULL` can't be bypassed could be good or bad. We'll come back to that.

### Level 2: CHECK constraints (single row)

Says: "what data is valid for a single row?" Eg:

- `ALTER TABLE accounts ADD CONSTRAINT positive_balance CHECK (balance > 0);`
- `ALTER TABLE reservations ADD CONSTRAINT positive_duration CHECK (checkout_time > checkin_time);`

- application code can catch a lot of the problems a check constraint might catch
- However, consider the balance one. If multiple concurrent transactions could decrease the balance, you could get a race condition.
- Eg, a business account has $100 of credit and two users issue $20 purchases at the same time. You don't want both to `SELECT`, see $100, subtract $20, and `UPDATE` to $80.
- You could make each of them lock the row before reading. Or you could do the update in one query `UPDATE accounts SET balance = balance - 20`. If you do that AND have a check constraint to prevent going below 0, the transaction will succeed or fail appropriately.
- I stole that example from a great blog post by Kevin Burke called [Weird Tricks to Write Faster, More Correct Database Queries](https://kev.inburke.com/kevin/faster-correct-database-queries/)

### Level 3: EXCLUDE constraints (row vs rest of table)

Says **"what data is valid for a row, considering the rows that already exist in this table?"**

That sounds a lot like a uniqueness constraint, right? Exclusion constraints are just a bit more general. Eg:
- `ALTER TABLE users ADD CONSTRAINT unique_username EXCLUDE USING gist (username WITH =)`;
- That's the same as a unique constraint. It says "compare the new/updated row with existing ones like this: compare the `username` values with the `=` operation."
- But you can give it a LIST of comparisons, and it can be any commutative operation.
- In other words, `=` is OK because `new_username = old_username` is the same as `old_username = new_username`. But `5 > 4` isn't the same as `4 > 5`.
- We can't specify order, so `>` isn't allowed as one of our criteria.
- The operators you can use (all the ones I know of) are `=`, `<>` (not equal), and `&&` (overlaps)
- Here's one that says "there can't be two overlapping rentals of the same rental property":

    ALTER TABLE reservations ADD CONSTRAINT no_overlapping_rentals
    EXCLUDE USING gist (property_id WITH =,
    tsrange("checkin_time", "checkout_time", '[]') WITH &&);

- So if the two property_ids are equal, and the date ranges overlap, the reservation we're trying to add is invalid; it conflicts with an existing one
- This is like the uniqueness constraint: application validations will catch most cases, but without a constraint, some conflicts can get through because there's a race condition
- Your rental company will have very unhappy customers if two of them get the same condo on the same day
- More fanciness: Exclusion constraints are based on indexes, and PostgreSQL supports "partial indexes", meaning indexes with a `WHERE` clause. A typical use case might be to only include unpaid invoices in an index because you expect to search them more often than paid ones; this results in a smaller index and faster searches for the data you care about.
- You can use this to customize a constraint, too - to say "overlapping rentals that are cancelled don't matter":

    ALTER TABLE reservations ADD CONSTRAINT no_overlapping_rentals
    EXCLUDE USING gist (property_id WITH =,
    tsrange("checkin_time", "checkout_time", '[]') WITH &&)
    WHERE (status != 'cancelled');

- Finally, you can set constraints as "deferrable".

    ALTER TABLE reservations ADD CONSTRAINT no_overlapping_rentals
    EXCLUDE USING gist (property_id WITH =,
    tsrange("checkin_time", "checkout_time", '[]') WITH &&)
    WHERE (status != 'cancelled')
    DEFERRABLE INITIALLY DEFERRED;

- `DEFERRABLE` means "it's OK to wait until the end of a transaction to enforce this", and `INITIALLY DEFERRED` means "make that the default behavior." You could do `INITIALLY IMMEDIATE` to check each statement by default.
- This is useful because it gives you some wiggle room: it allows you to start a transaction, shuffle data around in a way that creates temporary conflicts, fix them, and commit the transaction
- Inside a particular transaction, if a constraint is `DEFERRABLE`, you can use `SET CONSTRAINT no_overlapping_rentals IMMEDIATE` (or `DEFERRED`) to control the value for this transaction

### Level 4: Foreign key constraints (row vs another table)

You probably know about these - you can't comment on a post that doesn't exist, etc.

    #!sql
    ALTER TABLE reservations ADD FOREIGN KEY(property_id)
    REFERENCES properties(id);

- They also let you restrict or cascade deletes - eg, can't delete a post with comments, or if you do, delete the comments also
- You can't put a `WHERE` on a foreign key constraint, but you can make them deferrable:

    ALTER TABLE reservations ADD CONSTRAINT reservations_property_id_fk FOREIGN KEY(property_id) REFERENCES properties(id) DEFERRABLE INITIALLY DEFERRED;

## When NOT to use constraints

- Validations can vary by context - eg maybe an Admin can create a user without supplying a password, but the user has to supply one later. Constraints aren't that flexible.
- Validations can be skipped. This is bad if it's done by accident and good if you actually need to.
- Constraints can't be skipped without altering the database to drop them. This is good if you *really* want a rule enforced, and bad if you might not.

So:

- Only use constraints for iron-clad, inviolable rules, like "user must have an email" and "price can't be negative" and "no way are we going to let two users book this condo at the same time"
- If you *sometimes* need a validation, and it depends on what's in the database *right now*, you could use locks instead 
