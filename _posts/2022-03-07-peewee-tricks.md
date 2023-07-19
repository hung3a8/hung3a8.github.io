---
layout: post
title: "Relationships workarounds in Peewee"
date: 2022-03-07
categories: coding
---

Peewee is a lightweight ORM for Python. Compared to SQLAlchemy, Peewee is much more simpler, and compared to Django ORM, it is much more explicit. Here are some Peewee tricks you can use to improve your code.

## Ultilize DefferedForeignKeyField to avoid circular imports

Like any other ORM, Peewee has ForeignKeyField and ManyToManyField to work with relationships. Although these fields is really convenient, they are not very powerful in some cases.

The most common issue is circular imports. Take a look at the following code:

```python
class User(Model):
    username = CharField()
    favorite_tweet = ForeignKeyField(Tweet, null=True)  # NameError!!

class Tweet(Model):
    message = TextField()
    user = ForeignKeyField(User, backref='tweets')
```

While it is possible to solve this issue by changing the order of the classes, it is not pratical and may cause further problems. To fix this, the simplest solution is to use a `DeferredForeignKey` field:

```python
class User(Model):
    username = CharField()
    favorite_tweet = DeferredForeignKey('Tweet', null=True)  # OK

class Tweet(Model):
    message = TextField()
    user = ForeignKeyField(User, backref='tweets')
```

While `DeferredForeignKey` is powerful, it can't resolve itself if deferred field is defined after the target class, and need to be resolved manually, which is really annoying to do so:

```python
class User(Model):
    username = CharField()
    favorite_tweet = DeferredForeignKey('Tweet', null=True)  # OK

class Tweet(Model):
    message = TextField()
    user = DeferredForeignKey(User, backref='tweets')

DeferredForeignKey.resolve(Tweet)  # Too annoying!
```

This can be solve easily by running this at the end of the file if every model in a single file, or after importing all the models into `__init__.py` :

```python
from peewee import DeferredForeignKey, Model

from .model import User, Tweet

for model in Model.__subclasses__():
    DeferredForeignKey.resolve(model)
```

`Model.__subclasses__()` returns a list of all subclasses of `Model`, which is useful to get all the models that are imported before the call. By doing this, DeferredForeignKey will resolve all unresolved fields that are stored in `DeferredForeignKey._unresolved`. This is a bit hacky, but from now on, you can use `DeferredForeignKey` instead of `ForeignKeyField` to avoid circular imports.

Although `ForeignKeyField` circular import error can be fixed by using `DeferredForeignKey`, it is still challenging when it comes to `ManyToManyField`, as `ManyToManyField` can only pass the model class, and the only thing that can be declared deferred is the through model, not the target model. To fix this, I made a modification of `ManyToManyField` that works just like the `DeferredForeignKey` by storing unresolved fields in `ManyToManyField._unresolved`:

```python
class ManyToManyField(peewee.MetaField):
    _unresolved = set()

    def __init__(self, rel_model_name, **kwargs):
        self.field_kwargs = kwargs
        self.rel_model_name = rel_model_name.lower()
        ManyToManyField._unresolved.add(self)

    __hash__ = object.__hash__

    def set_model(self, rel_model):
        field = peewee.ManyToManyField(rel_model, **self.field_kwargs)
        self.model._meta.add_field(self.name, field)

    @staticmethod
    def resolve(model_cls):
        unresolved = sorted(ManyToManyField._unresolved, key=operator.attrgetter('_order'))
        for dr in unresolved:
            if dr.rel_model_name == model_cls.__name__.lower():
                dr.set_model(model_cls)
                ManyToManyField._unresolved.discard(dr)

    def _create_through_model(self):
        """Use DeferredForeignKey to create the through model, as the rel_model is not resolved yet."""
        lhs = self.model.__name__.lower()
        rhs = self.rel_model_name.lower()
        tables = (lhs, rhs)

        class Meta:
            database = self.model._meta.database
            schema = self.model._meta.schema
            table_name = '%s_%s_through' % tables
            indexes = (
                ((lhs._meta.name, rhs._meta.name),
                 True),)

        params = {'on_delete': self._on_delete, 'on_update': self._on_update}
        attrs = {
            lhs._meta.name: DeferredForeignKey(lhs, **params),
            rhs._meta.name: DeferredForeignKey(rhs, **params),
            'Meta': Meta}

        klass_name = '%s%sThrough' % (lhs.__name__, rhs.__name__)
        return type(klass_name, (peewee.Model,), attrs)
```

The new `ManyToManyField` will store the relational model in `_unresolved` and resolve it when the target model is available by calling `ManyToManyField.resolve(target_model)`. Furthermore, if the through model is not specified, it will be created automatically, but with `DeferredForeignKey` instead of `ForeignKeyField` like the original implementation from Peewee, as the relational model is not resolved yet.

## `ManyToManyField` in autogenerated models from `playhouse.reflection.generate_models`

Peewee has a builtin extension called `playhouse` which is extremely handy with custom fields, model generator, etc. `playhouse.reflection.generate_models` is used to generate models from an existing database, and then return the models. For small projects and projects that have different codebases that using the same database but cannot use the same code for building database models (ie. my projects which has a bridge written in Python (using Peewee ORM) and the backend written in TypeScript (using TypeORM)). It also supports relationships, which is very useful as you now don't have to deal with handling the relationships by yourself.

But there are  the way `playhouse.reflection.generate_models` handling Many-to-Many relationships. Currently, the only relationships that are supported in `Peewee` are `OneToMany`, `ManyToOne`, and `OneToOne`, which are only requires foreign keys to handle relations between objects. But when it comes to `ManyToMany`, the madness begins. As `generate_models` only generates `ForeignKeyFIeld` to handle relations between fields and tables, when it meets a Many-to-Many table, which, of course, only contains the two foreign keys that related to the two tables of the relationship. This can cause many problems, as you now have to work on Many-to-Many relationships in a much more complex way, perform manual querying, etc. Luckily, we can use this workarround to create `ManyToManyField` in those models, which can solve all of our problems above.

```python
for name, model in models.items():
    attrs = model_attrs[name]
    if 'id' not in model._meta.fields:
        continue
    for fk in model._meta.backrefs.keys():
        if 'id' not in fk.model._meta.fields:
            right_model = [m for m in fk.model._meta.model_refs.keys() if m != model]
            if len(right_model) != 1:
                raise ValueError('the Many-to-many model %s contains more than 2 foreign keys' % fk.model)
            right_model = right_model[0]

            field = ManyToManyField(right_model, backref=pluralize(name), through_model=fk.model)

            attrs[pluralize(right_model._meta.name)] = field
        else:
            continue
    for attr, field in attrs.items():
        model._meta.add_field(attr, field)
```

This code is directly added to the end of `playhouse.reflection.Introspector` method `generate_models`, which is the one that we are actually using to generate models (`playhouse.reflection.generate_models` is a seperate function that is used to create a new `Introspector` object from the database, and then it generate the models for you). Let me explain the code a little bit:

1. First, we exclude all the models without the primary key field in it, and we may imply its name to be `id`. After excluding those tables, we want to work on the rest of the models. We search for all back-references in the table, and then if a we encounter a table with no primary key field in it, we move to the next step. Do this all over again until we move through all models and all of its back-references.
2. After you successfully found the table, it is now time to perform some magic. First, we have to search for the right-side model, which is the one related to the left-side table that we are working on. Most of auto-generated Many-to-Many tables from other ORMs contains only two foreign keys, one for the left-side and one for the right-side. We here imply that there should only exists only one model references that is different from our current left-side model.
3. After having the right-side model, we create a new field with the same name as the right model, but in plural form (you can find and use a `pluralize` function on your choice, we will not discuss about it here). This new `ManyToManyField` should contains both `backref` and `through_model` options to ultilize its power, as we all have those data available. Once finish, go back to step 2 if there are more models available, or else move to step 4.
4. After having all attributes stored, we add all the fields to the model `Meta`, using its method `add_field(attribute_name, field)`. And there you go! A new model class with proper `ManyToManyField` for you to work with your precious projects!

The code above might not work with your codebases, and you might need to modify it a little bit. But still, the concept is pretty nice, and I hope that this can save you a bunch of time.

## Ending

I hope these tricks will help you a lot with your code. If you have any questions, please feel free to contact me.

Happy coding!
