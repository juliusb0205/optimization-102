# Optimization 102


##Review of Indexing
* Just like an index at the start of a book.
* Make it easier to look for a specific item, because columns in question are catalogued.

```ruby
create table :fans do |t|
  t.string :name
  t.references :backstreet_boy, :index => true
end
```

##The Problem: N+1
```ruby
BackstreetBoy.all.each do |bboy|
   puts bboy.fans.first.name
end
```

```sql
SELECT * FROM backstreet_boys
SELECT name FROM fans WHERE backstreet_boy_id = 1 LIMIT 1
SELECT name FROM fans WHERE backstreet_boy_id = 2 LIMIT 1
SELECT name FROM fans WHERE backstreet_boy_id = 3 LIMIT 1
SELECT name FROM fans WHERE backstreet_boy_id = 4 LIMIT 1
SELECT name FROM fans WHERE backstreet_boy_id = 5 LIMIT 1
```
   
## The Solution: Eager Loading
### 1. `eager_load`
* `eager_load` creates one big query.
```ruby
BackstreetBoy.eager_load(:fans).each do |bboy|
   puts bboy.fans.first.name
end
```

```sql
SELECT "backstreet_boys"."id" AS t0_r0, "backstreet_boys"."name" AS t0_r1, "fans"."id" AS t1_r0, "fans"."name" AS t1_r1 LEFT OUTER JOIN "backstreet_boys_fans" ON "backstreet_boys_fans"."backstreet_boy_id" = "backstreet_boys"."id" LEFT OUTER JOIN "fans" ON "fans"."id" = "backstreet_boys_fans"."fan_id" WHERE "backstreet_boys"."id"
```

* This is good for queries that span multiple associations.

### 2. `preload`
* `preload` performs separate queries per association.
```ruby
BackstreetBoy.preload(:fans).each do |bboy|
   puts bboy.fans.first.name
end
```

```sql
SELECT  * FROM backstreet_boys WHERE "backstreet_boy_id" IN (1, 2, 3, 4, 5)
SELECT * FROM fans WHERE "fans"."id" IN (1, 2, 3, 4, 5, 6, 7, 8 ,9 ,10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20)
```

* This is good for queries that span only a few associations.

### 3. Eager Loading Gotchas

#### preload
* Using `preload` and subsequently chaining a condition afterwards will yield an error
```ruby
BackstreetBoy.preload(:fans).where(:fans => {:name => 'Jomelina'})
```
* Loading the association table separately doesn't `JOIN` the 2 tables together.

#### Solution?

* Use `eager_load` instead.

#### Notes
* `includes` decides when to use `eager_load` or `preload`.
* You may want to force `eager_load` for valid queries that handle several associations.

#### eager_load
* `eager_load`-ing/`includes`-ing than one HABTM association of the same model will cause weird things to happen to the query.

#### Solution/Workaround?
* Use a combination of `eager_load` and `preload`.

##5. Caching
### Low-level caching


* Each partial takes roughly 0.1ms to render. Fast? **I think not.**
* Imagine if you had over 9000 records.
* Remember to render by collection when possible:

```ruby
= render :partial => 'bboy_fan', :collection => bboy.fans
```

##5. Caching


4. Partials.
   a. speed of render
   b. use collection when able
5. Caching
   a. page caching
   b. low-level caching
   c. fragment caching
6. Wrap up
