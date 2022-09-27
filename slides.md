---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://images.unsplash.com/photo-1568637836624-5bcbe4076ea6
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# use UnoCSS
css: unocss
---

# Enjoy the performance

Cajoling elephants and other high flying circus acts

<!--
I wanted to pull together a talk about what to look for while working in rails to keep our database performing.

We all know that rails is a truly incredible tool. In just a few lines of config you can setup a full fledged scalable full stack application that will provide real user value. Unfortunately, as spectacular as it's magic tricks can be, sometimes what goes on behind the scenes can be downright ugly. I am going to present a couple of case studies I found in our app that I hope will make it easier to spot performance issues before they happen.

Before we get started, I want to say: I am going to show some code from our application. When this code was written it was more important that it was readable and correct than performant. I think it did an excellent job of achieving that goal. Our product needs have evolved since then so the code needs to change with it.
-->

---
layout: image-right
image: https://cdn.shopify.com/s/files/1/0200/7616/articles/easy-card-tricks.progressive.jpg?v=1654125785
---

# Is this your card?

```ruby {all|2}
class MonthSummaryResource < ApplicationResource
  attribute :co2e_lbs :float

  stat co2e_kgs: [] do
    sum do |scope, _|
      scope.sum(:co2e_lbs) / LBS_PER_KG
    end
  end

  stat battery_based_kwh: [] do
    sum do |scope, _|
      scope.sum {|ms| 
        ms.energy_totals["battery_based_kwh"] 
      }
    end
  end
end
```

<!--
Okay, so lets jump in with both feet. Here we have a significantly paired down graffiti resource. 
For those of you that don't know we use a tool known as graffiti that makes it easy for us to present our json api in a very consistent well defined way without a whole bunch of boiler plate. I like to think of the resource as essentially the presenter layer of the application, because it takes the model data and forms it into the shape that the frontend expects. 

A very important implication of that fact is that any code we put in here, or that gets called from a resource, runs in real time within the context of a request cycle. In other words the user is waiting while this code runs. That puts it as essentially the most important code path to be performant.

**click**

The first line here is an attribute. We have probably all seen this and it's pretty simple understand what is going on. We just display whatever is in the database column coe_lbs there. I just wanted to point at this because this is the kind of thing rails is screaming fast at: just get me what's in the database.

-->

---
layout: image-right
image: https://cdn.shopify.com/s/files/1/0200/7616/articles/easy-card-tricks.progressive.jpg?v=1654125785
---

# Is this your card?

```ruby {4-8}
class MonthSummaryResource < ApplicationResource
  attribute :co2e_lbs :float

  stat co2e_kgs: [] do
    sum do |scope, _|
      scope.sum(:co2e_lbs) / LBS_PER_KG
    end
  end

  stat battery_based_kwh: [] do
    sum do |scope, _|
      scope.sum {|ms| 
        ms.energy_totals["battery_based_kwh"] 
      }
    end
  end
end
```
<div v-click>

  ```sql
    SELECT SUM(co2e_lbs) FROM month_summaries;
  ```
  
</div>



<!--
Here we have a stat. This is where graffiti get's interesting, and where whe start the real knife juggling. Ultimately what we are hoping to do is have this layer do as little as possible. We want postgres to hand us the data in as close to a complete from as we can so that this layer can focus on serialization and formatting. 

The goal of this particular query is to get the sum of all the carbon equivalent emissions within the scope. For now we have the scope wide open for simplicity. So let's take a look at the queries that get made when we execute this code. 

**click**

This seems reasonable. We are doing most of the work in postgres. Postgres will return a single number. We do need to do a unit conversion at the rails layer, but that is not so bad. 
-->
---
layout: image-right
image: https://cdn.shopify.com/s/files/1/0200/7616/articles/easy-card-tricks.progressive.jpg?v=1654125785
---

# Is this your card?

```ruby {9-15}
class MonthSummaryResource < ApplicationResource
  attribute :co2e_lbs :float

  stat co2e_kgs: [] do
    sum do |scope, _|
      scope.sum(:co2e_lbs) / LBS_PER_KG
    end
  end

  stat battery_based_kwh: [] do
    sum do |scope, _|
      scope.sum {|ms| 
        ms.energy_totals["battery_based_kwh"] 
      }
    end
  end
end
```

<div v-click>

  ```sql
    SELECT * FROM month_summaries;
  ```
</div>


<!--
Now let's take a look at this other stat. It's doing pretty much the same thing, so the sql query should look the same right? 

**click**

UHOH! There is what we were hoping to avoid. We are materializing every single month summary into memory and looping over them. The central issue here is that the field that we are trying to sum is not stored directly as a model attribute, so rails can't autogenerate the sql to some this directly. We are going to have to do it ourself. 
-->

---
layout: image-left
image: https://media.istockphoto.com/photos/playful-elephant-picture-id452404545?k=20&m=452404545&s=612x612&w=0&h=M3m9by2KbTNFVxAL0I18R5QGAbSPuTHtzWLbBw1SbD8=
---

# Elephants never forget

```ruby
scope.sum {|ms| 
  ms.energy_totals["battery_based_kwh"] 
}
```

```ruby{none|all|2-5|6|7}
scope
  .from("
      month_summaries, 
      json_each_text(month_summaries.energy_totals)
    ")
  .where("key = 'battery_based_kwh'")
  .sum("value::decimal")
```


<!--
Let's take a look at how we can write this query to be a little bit more performant. The top is what we had before. Now that we look at it closely it's pretty clear that this is doing a fair amount of logic in ruby.

** click **

The bottom does all logic in postgres which is much more performant. Now this talk is not about SQL so we aren't going to go super deep into what is going on in this specific query, but the key is that we are using the methods that rails gives us to form a performant sql query. Each method we call maps directly into something our orm knows how to include in a single call to the database.

** click **

from

** click **

where

** click **

sum; all sql methods
-->
---
layout: image-right
image: https://www.planetfashiontv.com/wp-content/uploads/2014/07/LE-cirque-du-soleil.jpg
---

# The sensational and shocking spectacle

```ruby 
stat amount_by_resource_name: [] do
  sum do |scope, _|
    scope
      .group_by{|ed| ed.ghg_emissions_factor.name}
      .map do |name, emission_days|
        [name, emission_days.sum(:amount)]
      end.to_h
    end
  end
end
```

<div v-click>

```sql
SELECT * FROM emission_days;
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...
.
.
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...
SELECT name FROM ghg_emission_factors where ...

```

</div>

<!-- 
Ok so let's take a look at another kind of performance issue. Here is a stat I found, and again I have removed some of the extra syntax, but this is pretty much the core. The goal here is to get emission volumes on a per ghg factor basis. So let's take a look at the queries made by this one

** click **

Oh boy. That's no good. This is what we are talking about when we talk about n+1 queries. The key here is that we are loading an association inside a loop. That means that we can't warn ruby that we are going to need multiple things out of the same table so it has to go back to the database over and over. While ruby is definitely slow, a round trip to the database is MUCH slower.
-->
---
layout: image-left
image: https://globalnews.ca/wp-content/uploads/2019/09/circus-edited.jpg?quality=85&strip=all
---

# Flying with the greatest of ease

```ruby{all|2|all}
scope
  .group_by{|ed| ed.ghg_emissions_factor.name}
  .map do |name, emission_days|
    [name, emission_days.sum(:amount)]
  end
end
```

```ruby {none|all}
scope
  .select("ghg_emissions_factors.name")
  .select("SUM(amount)")
  .joins(:ghg_emissions_factor)
  .group("ghg_emissions_factors.name")
```

<!--
Again we are going to take a look at how we can write this query to be a little bit more performant. The top is what we had before. You can see here that it is loading an associated model inside a loop.

** click **

Again the bottom does all logic in postgres which is much more performant. We are still using the methods that rails gives us to form a performant sql query. Again this allows the postgres client to run the query in a single call.
-->
---
layout: image-right
image: https://www.goldderby.com/wp-content/uploads/2022/03/ringmaster-costume-the-masked-singer.jpg
---

# 

```ruby 
attribute :cost_dollars, :big_decimal do
  convert_currency(
    @object.cost_major, 
    @object.currency.alpha_code
  )
end
```

<div v-click>

```sql
SELECT * FROM month_summaries;
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...
.
.
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...
SELECT alpha_code FROM currencies where ...

```

</div>

<!-- 
Ok so so far we have been focused on stat calls, but this mistake is pretty easy to make in other places as well. Let's take a look at this attribute. I think we can all see the issue here right? What's the query going to look like? 

** click **

yup, we have an n+1 here too. The solution to this one was a little more complicated, so I am not going to go down that well here, I just wanted to demonstrate that this is easy to do at any level.
-->