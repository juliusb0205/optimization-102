# Optimization 102


##Review
###DB Indexing
* Just like an index at the start of a book.
* Make it easier to look for a specific item, because columns in question are catalogued.

```ruby
create table :fans do |t|
  t.string :name
  t.references :backstreet_boy, :index => true
end
```

###Partials
* Remember to render by collection when possible:

```ruby
= render :partial => 'bboy_fan', :collection => bboy.fans

# renders _fan.haml
= render bboy.fans
```

##The Problem: N+1
```ruby
BackstreetBoy.all.each do |bboy|
   puts bboy.fans.first.name
end
```

```sql
SELECT * FROM backstreet_boys
SELECT * FROM fans WHERE backstreet_boy_id = 1 LIMIT 1
SELECT * FROM fans WHERE backstreet_boy_id = 2 LIMIT 1
SELECT * FROM fans WHERE backstreet_boy_id = 3 LIMIT 1
SELECT * FROM fans WHERE backstreet_boy_id = 4 LIMIT 1
SELECT * FROM fans WHERE backstreet_boy_id = 5 LIMIT 1
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

##4. Caching
on your <environment>.rb file, set:

```ruby
config.action_controller.perform_caching = true
# :file_store, :memory_store, :mem_cache_store
config.cache_store = :file_store, '/tmp/backstreet_cache'
```
### SQL caching
`@bboy = BackstreetBoy.all`

### Low-level caching
* Manually/Directly store data in the cache store
```ruby
Rails.cache.fetch('nick_carters_biggest_fan') { 'James' }
=> "James"
Rails.cache.fetch('nick_carters_biggest_fan')
=> "James"
```

### Fragment caching

* Each partial takes roughly 0.1ms to render. Fast? **I think not.**
* Imagine if you had over 9000 records!?.

```ruby
- cache "bboy_fans-#{bboy.id}" do
  = render :partial => 'bboy_fan', :collection => bboy.fans
```

Write:
```
Exist fragment? views/bboy_fans-2 (1.6ms)
Write fragment views/bboy_fans-2 (0.9ms)
```

Read:
```
Exist fragment? views/bboy_fans-2 (0.6ms)
Read fragment views/bboy_fans-2 (0.0ms)
```

* Expire fragments when changes occur:
`expire_fragment("bboy_fans-#{@fan.bboy.id}")`

### Russian doll caching
* Nested fragments

```ruby
- cache "bboy_fans-#{bboy.id}" do
  - bboy.fans.each do |fan|
    - cache "fan-#{fan.id}" do
      = render :partial => 'bboy_fan', :locals => {:fan => fan}
```

## Further reading:
@bramirez on using memcached as an alternative cache store:
* https://github.com/clinic-it/zen/blob/master/tech_sessions/mem_cached.md

@bramirez's intro to fragment caching:
* https://github.com/clinic-it/zen/blob/master/tech_sessions/FragmentCaching.md
