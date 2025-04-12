---
title: Magical & in Ruby
date: 2025-04-12 20:58:14
tags: [Ruby]
categories: [Technical]
---

{% asset_img post6_title.png rl %}


What is `&` in Ruby...

<!-- more -->

Recently, I've been learning the Ruby programming language.  
With my previous experience using Python, I originally thought it would be easy to get started,   
but when actually using it, I still feel like it's kinda different from Python.

There are many things in Ruby that Python doesn't have,  
and these things often make it difficult for me to make the transition.

Although Ruby might be more aligned with the object-oriented style,  
my feeling is that it's also more abstract at the same time.

So this post is very short, mainly to record the `&` symbol that is often encountered when using methods in Ruby,  
which still makes me feel kinda `abstract` sometimes. 🥹


---

## TL;DR
Some notes about `&` usage when using methods in Ruby.

---


## A couple of different usage about `&`

### 1. 🤔

The first scenario is using `&` to defined parameter of a method.  
Specifically means that the **last parameter** of a method is defined with `&`.

```Ruby
def func(&p)
  puts "parameter class: #{p.class}"
  p.call
end
```

In this scenario, Ruby would expect there is a `block` coming after the method call.  
What it does is taking that block into the parameter list and converting it to a `proc` object 
then assign to the parameter.  

So We could examine like this:

```Ruby
func {puts "Hi, I'm proc."}

# output
# parameter class: Proc
# Hi, I'm proc
```

The general procedure
1. Ruby sees there is a parameter defined with `&` and expect a block.
2. While invoking the method , we put a block after it.
3. Ruby then takes the block, converting it to a Proc object and assigning it to the prarmeter.


### 2. 🥴

The second scenario is like the opposite side of the first scenario.  
While invoking a method with `&xxx` placing as the **last** argument like the below

```Ruby
def func
  yield
end

p = Proc.new {puts "Hi, I'm proc"}
func(&p)
```

Notice that there is no parameter defined of the method.  
We just called the method with the `&p` argument.

So Ruby will do the following 3 things:

1. Ruby checks if `p` has a `to_proc` method. If it does, it will convert it to a proc object.   
   (if it's already a proc, no conversion is needed).
2. After converting to a proc, it gets kicked out of the argument list and is then converted to a block.
3. Finally, it's used like a normal block.


So the above method called example is equivelant to this:  
```Ruby
func {puts "Hi, I'm proc"}
```

Another common way is to use a `symbol` to call, because symbols have a `to_proc` method.
```Ruby
def func(x)
  result = yield(x)
  puts result
end

func('wayne', &:upcase)
```

Ruby sees the `:upcase` symbol and invokes the `to_proc` method on it,  
and then convert it a block.
```Ruby
# &:upcase doing this action
symbol_p = :upcase.to_proc
# and it basically equals to => Proc.new {|x| x.upcase}
```

### 3. 🤯

The third scenario is bascially the different way of handling the first scenario.  
And this one is not very straightforward for me.

We could also invoke like this
```Ruby
def func(&p)
  puts "parameter class: #{p.class}"
  p.call
end

pr = Proc.new {puts "Hi, I'm another proc."}

func(&pr)
# output
# parameter class: Proc
# Hi, I'm another proc.
```

Here is the diagram describe the process

{% asset_img mermaid.png rl %}


----

That's all I wanna share in this post ~

See ya ~ 👋
