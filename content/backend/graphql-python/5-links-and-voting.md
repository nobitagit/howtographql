---
title: Links and Voting
question: In which Python class is defined the arguments for a Mutation?
answers: ["Input", "Query", "Mutation", "Schema"]
correctAnswer: 0
description: Enable Users to create Links and to Vote on them
---

### Attaching Users to Links
With sign in power, you can now create you *own* links, posted by you. To make it possible, let's integrate the Links and Users models:

<Instruction>

On the Link models file, add the `posted_by` field at the very end:

```python(path=".../graphql-python/hackernews/links/models.py")
posted_by = models.ForeignKey('users.User', null=True, on_delete=models.deletion.CASCADE)
```

</Instruction>

<Instruction>

Run the Django commands to reflect the changes on the database:

```bash
python manage.py makemigrations
python manage.py migrate
```

</Instruction>

<Instruction>

On the `CreateLink` mutation, grab the user from the client's session to use it in the new created field:

```python(path=".../graphql-python/hackernews/links/schema.py")
# ...code
# Add the following imports
from users.schema import get_user, UserType


# ...code
# Change the CreateLink mutation
class CreateLink(graphene.Mutation):
    id = graphene.Int()
    url = graphene.String()
    description = graphene.String()
    posted_by = graphene.Field(UserType)

    class Arguments:
        url = graphene.String()
        description = graphene.String()

    def mutate(self, info, url, description):
        user = get_user(info) or None

        link = Link(
            url=url,
            description=description,
            posted_by=user,
        )
        link.save()

        return CreateLink(
            id=link.id,
            url=link.url,
            description=link.description,
            posted_by=link.posted_by,
        )
```

</Instruction>

To test it, send a mutation to the server:

![](http://i.imgur.com/9JMnRWf.png)

Neat!

### Adding Votes
One of the Hackernews' features is to vote on links, making ones more popular than others. To create this, you'll need to create a Vote model and a `CreateVote` mutation. Let's begin:

<Instruction>

Add the Vote model on the `links/models.py`:

```python(path=".../graphql-python/hackernews/links/schema.py")
class Vote(models.Model):
    user = models.ForeignKey('users.User', on_delete=models.deletion.CASCADE)
    link = models.ForeignKey('links.Link', related_name='votes', on_delete=models.deletion.CASCADE)
```

</Instruction>

<Instruction>

And reflect the changes on the database:

```bash
python manage.py makemigrations
python manage.py migrate
```

</Instruction>

<Instruction>

Finally, add a new mutation for voting:

```python(path=".../graphql-python/hackernews/links/schema.py")
# ...code
# In the same import line, add the Vote model
from links.models import Link, Vote


# ...code
# Add the CreateVote mutation
class CreateVote(graphene.Mutation):
    user = graphene.Field(UserType)
    link = graphene.Field(LinkType)

    class Arguments:
        link_id = graphene.Int()

    def mutate(self, info, link_id):
        user = get_user(info) or None
        if not user:
            raise Exception('You must be logged to vote!')

        link = Link.objects.filter(id=link_id).first()
        if not link:
            raise Exception('Invalid Link!')

        Vote.objects.create(
            user=user,
            link=link,
        )

        return CreateVote(user=user, link=link)


# ...code
# Add the mutation to the Mutation class
class Mutation(graphene.ObjectType):
    create_link = CreateLink.Field()
    create_vote = CreateVote.Field()
```

</Instruction>

Voting time! Try to vote for the first link:

![](http://i.imgur.com/ih72ZmP.png)

### Relating Links and Votes
You can already vote, but you can't see it! A solution for this is being able to get a list of all the votes and a list of votes from each link. Follow the next steps to accomplish it:

<Instruction>

Add the `VoteType` and a `votes` field to get all votes:

```python(path=".../graphql-python/hackernews/links/schema.py")
# ...code
# Add after the LinkType
class VoteType(DjangoObjectType):
    class Meta:
        model = Vote
```

</Instruction>

<Instruction>

And add the `votes` field and the `resolve_links` method:

```python(path=".../graphql-python/hackernews/links/schema.py")
# ...code
# Add the votes field
class Query(graphene.ObjectType):
    links = graphene.List(LinkType)
    votes = graphene.List(VoteType)

    def resolve_links(self, info, **kwargs):
        return Link.objects.all()

    def resolve_votes(self, info, **kwargs):
        return Vote.objects.all()
```

</Instruction>

Awesome! On GraphiQL, try to fetch the list of votes:

![](http://i.imgur.com/mkb5w1z.png)

To close this chapter, make a query for all the links and see how smoothly the votes become available:

![](http://i.imgur.com/uGMWHxV.png)

Yay!
