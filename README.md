
# How to Recover the Model from a Subquery with SQLAlchemy

**Disclaimer:** This guide offers an alternative explanation to the official SQLAlchemy [documentation on subqueries](https://docs.sqlalchemy.org/en/13/orm/tutorial.html#using-subqueries), providing more visual examples and simpler explanations. The information is based on `SQLAlchemy==2.0.30`, and newer versions or features may affect its relevance over time.

## Solution Overview

If you're only interested in the solution, here it is:

```python
# Build the main query
query: Query = session.query(Post)

# Build the subquery
subquery = (
    self.session.query(CheckReaction)
    .filter(CheckReaction.employee_id == user_id)
    .subquery()
)

# Join with the subquery
query = query.outerjoin(subquery, Post.id == subquery.c.post_id)
alias_reaction = aliased(Reaction, subquery)
query = query.add_entity(alias_reaction)
```

## Background

In many backend systems, we have different models for validation [(e.g., Pydantic)](https://docs.pydantic.dev/latest/) and for models in database representation [(e.g., SQLAlchemy)](https://www.sqlalchemy.org). Although some libraries like [SQLModel](https://sqlmodel.tiangolo.com) attempt to bridge this gap, they are still in early stages.

Consider a scenario where you have two SQLAlchemy models: _Post (P)_ and _Reaction (R)_. Here, P represents a social network post, and R represents a user's reaction to a post.

Suppose you need to query all P along with the R of a specific _User (U)_. Directly joining all posts with all reactions and filtering by `user_id` may be inefficient due to the large volume of data. Instead, you can optimize this by filtering reactions first and then performing the join. The SQL for this optimized approach looks like:

```sql
SELECT *
FROM 
  posts p
LEFT JOIN 
  (SELECT * 
   FROM reactions 
   WHERE user_id = {{user_id}}
  ) r
ON 
  r.post_id = p.id
```


The query itself it's not hard, but what I have found difficult was to found a way of keeping both models when using SQLAlchemy.

First we do a subquery, this would be equivalent to the subquery we mentioned before to speed up the join operation.
```python
# Build the main query
query: Query = session.query(Post)

# Build the subquery
subquery = (
    self.session.query(CheckReaction)
    .filter(CheckReaction.employee_id == user_id)
    .subquery()
)
```
This part it's not something hard and we can easily find [documentation](https://docs.sqlalchemy.org/en/20/orm/queryguide/query.html#sqlalchemy.orm.Query.subquery) about it. The tricky part comes now, I had a hard momment finding a way to return both models in one single return. I could have returned the model P and the all the fields from R, but remember, we want a return with only both models, so it's clean and concise.

After some investigation I discovered that you need to alias the subquery, what I mean by this is that you need to specify the model that will be returned and then add the entity to the original query. Below you can see how it's done.

```python
# Join with the subquery
query = query.outerjoin(subquery, Post.id == subquery.c.post_id)
alias_reaction = aliased(Reaction, subquery)
query = query.add_entity(alias_reaction)
return query.all()
```

Then we just do:
```python
post_models: list[PostModel] = [PostModel.model_validate(post) for post, _ in posts]
reactions_models: list[ReactionModel] = [ReactionModel.model_validate(reaction) for _, reaction in reactions]
```

It's interesting to understand why this happens, in SQL when you do `SELECT *` behind it deconstructs the model and returns you all the fields. But now in SQLAlchemy you don't want to lose these models as you may want to return them, and that's basically the reason.

It was an interesting investigation that was not easy to find and I wanted to share with all of you.
