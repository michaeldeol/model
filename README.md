# Lotus::Model

A persistence framework for [Lotus](http://lotusrb.org).

It delivers a convenient public API to execute queries and commands against a database.
The architecture eases keeping the business logic (entities) separated from details such as persistence or validations.

It implements the following concepts:

  * [Entity](#entities) - An object defined by its identity.
  * [Repository](#repositories) - An object that mediates between the entities and the persistence layer.
  * [Data Mapper](#data-mapper) - A persistence mapper that keep entities independent from database details.
  * [Adapter](#adapter) – A database adapter.
  * [Query](#query) - An object that represents a database query.

Like all the other Lotus components, it can be used as a standalone framework or within a full Lotus application.

## Status

[![Gem Version](https://badge.fury.io/rb/lotus-model.svg)](http://badge.fury.io/rb/lotus-model)
[![Build Status](https://secure.travis-ci.org/lotus/model.svg?branch=master)](http://travis-ci.org/lotus/model?branch=master)
[![Coverage](https://img.shields.io/coveralls/lotus/model/master.svg)](https://coveralls.io/r/lotus/model)
[![Code Climate](https://img.shields.io/codeclimate/github/lotus/model.svg)](https://codeclimate.com/github/lotus/model)
[![Dependencies](https://gemnasium.com/lotus/model.svg)](https://gemnasium.com/lotus/model)
[![Inline docs](http://inch-ci.org/github/lotus/model.png)](http://inch-ci.org/github/lotus/model)

## Contact

* Home page: http://lotusrb.org
* Mailing List: http://lotusrb.org/mailing-list
* API Doc: http://rdoc.info/gems/lotus-model
* Bugs/Issues: https://github.com/lotus/model/issues
* Support: http://stackoverflow.com/questions/tagged/lotus-ruby
* Chat: https://gitter.im/lotus/chat

## Rubies

__Lotus::Model__ supports Ruby (MRI) 2+

## Installation

Add this line to your application's Gemfile:

    gem 'lotus-model'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install lotus-model

## Usage

### Entities

An object that is defined by its identity.

An entity is the core of an application, where the part of the domain logic is implemented.
It's a small, cohesive object that expresses coherent and meaningful behaviors.

It deals with one and only one responsibility that is pertinent to the
domain of the application, without caring about details such as persistence
or validations.

This simplicity of design allows developers to focus on behaviors, or
message passing if you will, which is the quintessence of Object Oriented Programming.

```ruby
require 'lotus/model'

class Person
  include Lotus::Entity
  self.attributes = :name, :age
end
```

When a class includes `Lotus::Entity` it will receive the following interface:

  * `#id`
  * `#id=`
  * `#initialize(attributes = {})`

Also, the usage of `.attributes=` defines accessors for the given attribute names.

If we expand the code above in **pure Ruby**, it would be:

```ruby
class Person
  attr_accessor :id, :name, :age

  def initialize(attributes = {})
    @id, @name, @age = attributes.values_at(:id, :name, :age)
  end
end
```

Indeed, **Lotus::Model** ships `Entity` only for developers's convenience, but the
rest of the framework is able to accept any object that implements the interface above.

### Repositories

An object that mediates between entities and the persistence layer.
It offers a standardized API to query and execute commands on a database.

A repository is **storage independent**, all the queries and commands are
delegated to the current adapter.

This architecture has several advantages:

  * Applications depend on a standard API, instead of low level details
    (Dependency Inversion principle)

  * Applications depend on a stable API, that doesn't change if the
    storage changes

  * Developers can postpone storage decisions

  * Confines persistence logic at a low level

  * Multiple data sources can easily coexist in an application

When a class includes `Lotus::Repository`, it will receive the following interface:

  * `.persist(entity)` – Create or update an entity
  * `.create(entity)`  – Create a record for the given entity
  * `.update(entity)`  – Update the record corresponding to the given entity
  * `.delete(entity)`  – Delete the record corresponding to the given entity
  * `.all`   - Fetch all the entities from the collection
  * `.find`  - Fetch an entity from the collection by its ID
  * `.first` - Fetch the first entity from the collection
  * `.last`  - Fetch the last entity from the collection
  * `.clear` - Delete all the records from the collection
  * `.query` - Fabricates a query object

**A collection is a homogenous set of records.**
It corresponds to a table for a SQL database or to a MongoDB collection.

**All the queries are private**.
This decision forces developers to define intention revealing API, instead leak storage API details outside of a repository.

Look at the following code:

```ruby
ArticleRepository.where(author_id: 23).order(:published_at).limit(8)
```

This is **bad** for a variety of reasons:

  * The caller has an intimate knowledge of the internal mechanisms of the Repository.

  * The caller works on several levels of abstraction.

  * It doesn't express a clear intent, it's just a chain of methods.

  * The caller can't be easily tested in isolation.

  * If we change the storage, we are forced to change the code of the caller(s).

There is a better way:

```ruby
require 'lotus/model'

class ArticleRepository
  include Lotus::Repository

  def self.most_recent_by_author(author, limit = 8)
    query do
      where(author_id: author.id).
        order(:published_at)
    end.limit(limit)
  end
end
```

This is a **huge improvement**, because:

  * The caller doesn't know how the repository fetches the entities.

  * The caller works on a single level of abstraction. It doesn't even know about records, only works with entities.

  * It expresses a clear intent.

  * The caller can be easily tested in isolation. It's just a matter of stubbing this method.

  * If we change the storage, the callers aren't affected.

Here is an extended example of a repository that uses the SQL adapter.

```ruby
class ArticleRepository
  include Lotus::Repository

  def self.most_recent_by_author(author, limit = 8)
    query do
      where(author_id: author.id).
        desc(:id).
        limit(limit)
    end
  end

  def self.most_recent_published_by_author(author, limit = 8)
    most_recent_by_author(author, limit).published
  end

  def self.published
    query do
      where(published: true)
    end
  end

  def self.drafts
    exclude published
  end

  def self.rank
    published.desc(:comments_count)
  end

  def self.best_article_ever
    rank.limit(1)
  end

  def self.comments_average
    query.average(:comments_count)
  end
end
```

### Data Mapper

A persistence mapper that keeps entities independent from database details.
It is database independent, it can work with SQL, document, and even with key/value stores.

The role of a data mapper is to translate database columns into the corresponding attribute of an entity.

```ruby
require 'lotus/model'

mapper = Lotus::Model::Mapper.new do
  collection :users do
    entity User

    attribute :id,   Integer
    attribute :name, String
    attribute :age,  Integer
  end
end
```

For simplicity sake, imagine that the mapper above is used with a SQL database.
We use `#collection` to indicate the name of the table that we want to map, `#entity` to indicate the class that we want to associate.
In the end, each `#attribute` call, is to associate the column with a Ruby type.

For advanced mapping and legacy databases, please have a look at the API doc.

### Adapter

An adapter is a concrete implementation of persistence logic for a specific database.
**Lotus::Model** is shipped with two adapters:

  * SqlAdapter
  * MemoryAdapter

An adapter can be associated with one or multiple repositories.

```ruby
require 'pg'
require 'lotus/model'
require 'lotus/model/adapters/sql_adapter'

mapper = Lotus::Model::Mapper.new do
  # ...
end

adapter = Lotus::Model::Adapters::SqlAdapter.new(mapper, 'postgres://host:port/database')

PersonRepository.adapter  = adapter
ArticleRepository.adapter = adapter
```

In the example above, we reuse the adapter because the target tables (`people` and `articles`) are defined in the same database.
**As rule of thumb, one adapter instance per database.**

### Query

An object that implements an interface for quering the database.
This interface may vary, according to the adapter's specifications.
Think of an adapter for Redis, it will probably employ different strategies to filter records than an SQL query object.

### Conventions

  * A repository must be named after an entity, by appending `"Repository"` to the entity class name (eg. `Article` => `ArticleRepository`).

### Thread safety

**Lotus::Model**'s is thread safe during the runtime, but it isn't during the loading process.
The mapper compiles some code internally, be sure to safely load it before your application starts.

```ruby
Mutex.new.synchronize do
  mapper.load!
end
```

**This is not necessary, when Lotus::Model is used within a Lotus application.**

## Example

For a full working example, have a look at [EXAMPLE.md](https://github.com/lotus/model/blob/master/EXAMPLE.md).
Please remember that the setup code is only required for the standalone usage of **Lotus::Model**.
A **Lotus** application will handle that configurations for you.

## Versioning

__Lotus::Model__ uses [Semantic Versioning 2.0.0](http://semver.org)

## Contributing

1. Fork it ( https://github.com/lotus/model/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Copyright

Copyright 2014 Luca Guidi – Released under MIT License
