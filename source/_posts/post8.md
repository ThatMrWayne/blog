---
title: Introduce meta-programming in Ruby
date: 2025-09-13 21:17:04
tags: [Ruby]
categories: [Technical]
---

{% asset_img post8_title.png rl %}

Brief introduction about meta-programming in Ruby.


<!-- more -->

Recently, I've read the [《Programming Ruby 3.3 - The Pragmatic Programmers' Guide》](https://www.amazon.com/Programming-Ruby-3-2-Pragmatic-Programmers/dp/1680509829) which talks about some basic and a little advanced concept of Ruby.

And one of the topics is `meta-programming` which I personally thought is the
core part of this book.

So I'm gonna share some of my thoughts and understanding on this topic.

In order to understand meta-programming,  
I'll share some fundamental topics first.

----

TL;DR
Meta-programming in Ruby.

----

> The so called meta-programming is about letting the code itself modify or define itself.  
> Sounds pretty weird huh?

----

## self !

The fundamental, the basic of meta-programming is all about `self`.

### What is the role of `self` in Ruby ?

1. instance variable :  
When it comes to finding instance variable like `@abc` ,  
the current `self` determines where to find the instance variable.  

2. method call  
- implicit call  
  using the current `self` as receiver of the method.
- explicit call  
  `self` will be pointed to the receiver object. 
  ```ruby
    arr = [1,2,3]
    arr.size # self is set to arr
  ```
-------

## Class definition and singleton method

### class definition

One important thing to notice is that the class definition itself is a piece of executable code.  

Inside class definition, the `self` will be set to the class object.

```ruby
class Person
  @name = 'John'

  def self.name
    @name
  end
end

Person.name
#=> "John"
```
In this class definition example,  
since the `self` is set to class object `Person`,  
the instance variable `@name` actually belongs to class `Person`.

### singleton method

Singleton method is the kind of method that belong to that specific object and the singleton method is defined inside singleton class.

- ways of defining singleton methods
```ruby
def obj.some_method
  ...
end

# or

class << obj
  def some_method
    ...
  end
end
```
- `class method` is actually just the `singleton method` of that class object.  
  > 💡 The class method is just `instance method` of that class object.

  ```ruby
    class Abc
      def self.method_name
        xxx
      end
    end
  ```

----
## Inheritance relationship

Before we get going, we need to know this important concept first:
> ❗ The relationship of   inheritance of classes is parallel to the relationship of  their singleton classes.

I'll use diagrams below to elaborate this messy sentence.

- In the simplest inheritance relationship between two classes,  
their singleton classes also have the same inheritance relationship.  

We could prove it:  
```shell
> class A
end

> class B < A
end

> B.singleton_class.superclass
=> #<Class:A>
```
  

{% asset_img inheritance.png rl %}


-  This is a more complete picture.
{% asset_img inheritance2.png rl %}

Believe me ,  
knowing this first would make it a lot easier to understand the whole concept of meta-programming.

----

## Usage of module

Three different usage of module.

### include
When a class includes a module ,  
the module (technically the copy of that module) will be inserted above the host class in the method lookup chain of the host class
```shell
  module B
  end

  class A
  include B
  end

  A.ancestors
  =>  
  [A,
  B,
  ActiveSupport::Dependencies::RequireDependency,
  Object,
  PP::ObjectMixin,
  ActiveSupport::ToJsonWithActiveSupportEncoder,
  ActiveSupport::Tryable,
  JSON::Ext::Generator::GeneratorMethods::Object,
  Kernel,
  BasicObject]
```

### prepend
Technically prepend is same as include,  
the result would be the module being prepended is inserted prior to the host class in the method lookup chain.  
```shell
  module B
  end

  class C
    prepend B
  end

  C.ancestors
  =>
  [B,
  C,
  ActiveSupport::Dependencies::RequireDependency,
  Object,
  PP::ObjectMixin,
  ActiveSupport::ToJsonWithActiveSupportEncoder,
  ActiveSupport::Tryable,
  JSON::Ext::Generator::GeneratorMethods::Object,
  Kernel,
  BasicObject]
```
👉 Notice that the host class C is the first in it's ancestors.

### extend

> ❗ When it comes to `extend module` , it has something to do with singleton class.

When a host class extend a module , what it does is actually including that module in the singlton class of the host class.  

So that's why when a class extends a module ,  
the class could use the method defined in the module as class methods.  
Since those methods in that module are now placed in the method lookup chain of that host class.
```shell
> module D
    def foo
      puts "foo"
    end
  end

> class E
    extend D
  end

> E.methods
=>  
[:foo,
 :yaml_tag,
 :descendants,
 :subclasses,
 :json_creatable?,
 :class_attribute,
 :allocate,
 ...
]
```
Notice that the method `foo` is now the instance method that class E could invoke.

-----

## class macros

>❗ The so-called `class macros` in Ruby are just class methods invoked in the class definition context, written in a DSL style to look like declarative keywords.

Some common examples in Ruby on Rails:

1. `attr_accessor`

The `attr_accessor` is actually a method defined in class `Module`.


```ruby
  class Example
    attr_accessor :abc
  end
```

2. `has_many`

The `has_many` is actually a method defined in the singleton class of `ActiveRecord::Base`

```ruby
  class Album < ActiveRecord::Base
    has_many :tracks
  end
```

I feel like the way of using these class methods are the spirit of `meta-programming`, which let code itself achieve defining extra settings during the class context definition.

For example, we could try implementing the same attr_accessor like this:
```ruby
  class Module
    def my_attr_accessor(attr_name)
      # getter
      define_method(attr_name) do
        instance_variable_get("@#{attr_name}")
      end

      # setter
      define_method("#{attr_name}=") do |value|
        instance_variable_set("@#{attr_name}", value)
      end
    end
  end

  class Person
    my_attr_accessor :age
  end

  person = Person.new
  person.age = 30
  person.age
  => 30
```
-----

## Changing `self` temporarily

Ruby gives us some ways to manipulate self temporarily.

- instance_eval (defined in class `BasicObject` )
- class_eval (defined in  class `Module`)

In the context of class:
- It means defining class method (in the singleton class of that class object)
```ruby
  some_class.instance_eval do
    def xxx
      ...
    end
  end
```

- It means defining methods as instance method inside the class itself.
```ruby
  some_class.class_eval do
    def xxx
      ...
    end
  end
```

-----

## Concern

Last but not least, the `concern` in Ruby on Rails is a good example of meta-programming.  

Let's take a brief look on how concern work in principle.

The result of using concern is we could add instance methods, class methods and also using DSL to the host class that include the module which extending `ActiveSupport::Concern`.

Code below is more like a pseudo code for demonstrating the concept.

```ruby
module ActiveSupport
  module Concern
    def self.extended(base)
      base.instance_variable_set(:@_included_blocks, [])
    end

    def included(base = nil, &block)
      if base.nil?
        # 如果是 Concern module 本身呼叫 `included do ... end`
        @_included_blocks << block
      else
        # 如果真的被 include 到 class 了
        super
        @_included_blocks.each { |blk| base.class_eval(&blk) }
      end
    end

    def append_features(base)
      super
      base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
    end
  end
end
```

```ruby
# app/models/concerns/trackable.rb
module Foo
  extend ActiveSupport::Concern

  # instance method
  def track!
    puts "[#{self.class.name}] tracking..."
  end

  # included block （延遲執行，等真的被 include 到 class 才跑）
  included do
    puts ">> Trackable has been included into #{self}"
    def new_method
      puts "new_method"
    end
  end

  # class methods
  module ClassMethods
    def tracking_enabled?
      true
    end
  end
end


class Bar
  include Foo
end
```

There is a module `Foo` which extends `ActiveSupport::Concern`, and the class `Bar` include that module `Foo`.

🍻 The whole process flow :

1. In the module `Foo` definition, since it extends `ActiveSupport::Concern`,   
so it gets all the class methods from `ActiveSupport::Concern`.  
When it extends `ActiveSupport::Concern`, the hook method `extended` is invoked with `ActiveSupport::Concern` as receiver and the argument base is passed as class `Foo`. Here the instance variable `@_included_blocks` is set to class `Foo`.

2. While executing to `included do ...` in module `Foo`, it will invoke the included method defined in `ActiveSupport::Concern`, which will append the passed block to the `@_included_blocks` array.

3. When the class `Bar` includes `Foo`
  - the method `append_features` will be invoked first on module `Foo` , which leads to extending the module `ClassMethods` on class `Bar`.

  - the hook method `included` will then be invoked again on module `Foo`, and here is the key point in this line:  
  `@_included_blocks.each { |blk| base.class_eval(&blk) }`  
  The blocks stored in `@_included_blocks` will be executed in the context of class `Bar`,  
  so here the method `new_method` will be added as instance method and if there are other DSL added in the block could also be executed here (like validation, before_action kind of thing).


----

The skill of meta-programming is still something I'm trying to master in ,  
In this post, I share some of the core concepts about it.

Thank y'all for reading,  
that's all I wanna share in this post ~

See ya ~ 👋
