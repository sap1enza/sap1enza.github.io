---
layout: post
title: ActiveRecord - Quick Tips
tags: [ruby, rails, activerecord]
image: '/images/posts/rails.jpg'
---

I will use the data from an dump with 100.000 registers

## pluck vs. map

Map is useful for arrays, and that is it. Rails select all columns to instantiate an object for each row

Pluck delegates that job to the SQL, which turns the process way much faster and effective.

```ruby
Benchmark.ms { Purchase.all.map(&:id) }
=> 23967.25999999944

Benchmark.ms { Purchase.pluck(:id) }
=> 425.65599999943515
```

If you already had an array with the objects, maybe it is better to use the map than touch the database again

## uniq

Order makes difference

```ruby
Benchmark.ms { Purchase.pluck(:id).uniq }
 => 1645.4709999998158

Benchmark.ms { Purchase.uniq.pluck(:id) }
 => 446.65100000020175
```

At the first case, the pluck returns an array with an id for each row from the Purchases table, the duplicated are then filtered using the `Array#uniq`

In the second the uniq uses the `ActiveRecord::QueryMethods#uniq` that filter directly in the SQL with DISTINCT


# find_each vs. find_in_batches vs. Each

For iterations over a small number of objects, using each solves it. But when the talk about thousands or millions of objects, using 
each becomes inviable taking in consideration that it loads all objects to just them iterate over it.
 
The solution is to “broke” that in blocks

```ruby
Purchase.where(confirmed: false).each do |purchase|
  purchase.confirm!
end


Purchase.where(confirmed: false).find_in_batches do |group|
  group.each { |purchase| purchase.confirm! }
end

Purchase.where(confirmed: false).find_each do |puchase| 
  purchase.confirm!
end
```

`each` will block the row UNTIL gets it done, it can be a trouble if we are talking about a big table or if the table is used in other scenarios as well

Using the `find_in_batches`, it will make the queries by blocks of 1000 ( default, but can be changed ), in other words it will no longer block the table to other queries

`find_each` is basically an shortcut for `find_in_batches` that dont use the scope of the blocks returned


## includes

Nice to avoid N+1 queries, since it also load the relation records 

```ruby
Purchase.includes(:person).map { |purchase| purchase.person.id }
```

## joins vs includes

Joins is a good one if you are just filtering through the results, not accessing the records from the relationship directly. ( It dont avoid N+1 queries )

```ruby
Purchase.joins(:person).where(:people => {confirmed: true}).map { |purchase| purchase.id }
```

Otherwise, if will access the associations directly its a good one to use includes since it avoids N+1 queries

```ruby
Purchase.includes(:person).map { |purchase| purchase.person.id }
```

## exists vs any

```ruby
Benchmark.ms { Purchase.any? }
 => 31.580000009853393

Benchmark.ms { Purchase.exists? }
 => 3.292000008514151
```

`any?`:  accepts a block and retrieves the records in the relation with it  ( calls #to_a, calls the block, hits it with `Enumerable#any?` ), without a block its the same as `!empty?` and count the records of the relation

As few examples earlier, the `exists?` delegate it to the SQL setting a `LIMIT 1 ` and becomes way faster.

# Group Counting

Interesting way to get the count based on attributes of the model.

```ruby
Purchase.group(:signed_contract).count
 => {false=>1527, true=>98484}
```