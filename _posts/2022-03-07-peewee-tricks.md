---
layout: post
title: "Tips and tricks for using Peewee ORM"
date: 2022-03-07
categories: Peewee
---

Peewee is a lightweight ORM for Python. Compared to SQLAlchemy, Peewee is much more simpler, and compared to Django ORM, it is much more explicit. Here are some Peewee tricks you can use to improve your code.

### Working with relationships

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

I hope these tricks will help you a lot with your code. If you have any questions, please feel free to contact me.

Happy coding!
