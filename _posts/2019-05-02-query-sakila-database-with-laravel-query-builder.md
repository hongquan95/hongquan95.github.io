---
layout: post
title:  "Query Sakila Database with Laravel Query Builder"
date:   2019-05-02 22:00:00 +0700
categories: [laravel, database]
---

Document:
- [Sakila Database.](https://dev.mysql.com/doc/sakila/en/)
- [Download.](https://github.com/hongquan95/jOOQ/tree/master/jOOQ-examples/Sakila/mysql-sakila-db)
- [Questions](https://github.com/adesai25/MySQL-Exercises-with-Sakila-DB-)
- [Query Builder](https://laravel.com/docs/5.8/queries)

1. Select first_name full_name from actor
```php
DB::table('actor')->selectRaw("CONCAT(first_name,' ' ,last_name) full_name")->get()
```
2. Display the first and last name of each actor in a single column in upper case letters. Name the column `Actor Name`.
```php
DB::table('actor')
    ->selectRaw("CONCAT
        (CONCAT(SUBSTRING(first_name,1,1), LOWER(SUBSTRING(first_name, 2))),
        ' ',
        CONCAT(SUBSTRING(last_name,1,1), LOWER(SUBSTRING(last_name, 2) as 'Actor Name'")->get()
```

3. You need to find the ID number, first name, and last name of an actor, of whom you know only the first name, `Joe.` What is one query would you use to obtain this information?
```php
 DB::table('actor')
    ->select(['actor_id', 'first_name', 'last_name'])
    ->where('first_name', 'LIKE', 'JOE')
    ->get()
```

4. Find all actors whose last name contain the letters `GEN`:
```php
DB::table('actor')
    ->select(['actor_id', 'first_name', 'last_name'])
    ->where('last_name', 'LIKE', '%GEN%')
    ->get()
```

5. Find all actors whose last names contain the letters `LI`. This time, order the rows by last name and first name, in that order:
```php
DB::table('actor')
    ->select(['actor_id', 'first_name', 'last_name'])
    ->where('last_name', 'LIKE', '%LI%')
    ->orderBy('last_name', 'desc')
    ->orderBy('first_name', 'desc')
    ->get()
```

6. Using `IN`, display the `country_id` and `country` columns of the following countries: `Afghanistan`, `Bangladesh`, and `China`:
```php
DB::table('country')
    ->whereIn('country', ['Afghanistan', 'Bangladesh', 'China'])
    ->select('country_id', 'country')
    ->get()
```

7. List the last names of actors, as well as how many actors have that last name.
```php
DB::table('actor')
    ->select(['last_name', DB::raw("COUNT(last_name) as count")])
    ->groupBy('last_name')
    ->get()
```

8. List last names of actors and the number of actors who have that last name, but only for names that are shared by at least two actors
```php
DB::table('actor')
    ->select(['last_name', DB::raw("COUNT(last_name) as count")])
    ->groupBy('last_name')
    ->having('count', '>=', 2)
    ->get()
```

9. Use `JOIN` to display the `first` and `last names`, as well as the `address`, of each staff member. Use the tables `staff` and `address`:
```php
DB::table('staff')
    ->join('address', 'staff.address_id', '=', 'address.address_id')
    ->select(['first_name', 'last_name', 'address'])
    ->get()
```

10. Use `JOIN` to display the `total amount` rung up by each staff member in `August of 2005`. Use tables `staff` and `payment`.
```php
DB::table('staff')
    ->join('payment', 'staff.staff_id', '=', 'payment.staff_id')
    ->whereRaw("DATE_FORMAT(payment_date,'%Y-%m') = '2005-08'")
    ->select(['staff.*', DB::raw("SUM(amount) as total_amount")])
    ->groupBy('staff_id')
    ->get()
```

11. List each film and the number of actors who are listed for that film. Use tables film_actor and film. Use inner join.
```php
DB::table('film')
    ->join('film_actor', 'film.film_id', '=', 'film_actor.film_id')
    ->select(['film.*', DB::raw('COUNT(film_actor.film_id) count')])
    ->groupBy('film_actor.film_id')
    ->get()
```

12. How many copies of the film Hunchback Impossible exist in the inventory system?
```php
DB::table('inventory')
    ->join('film', 'inventory.film_id', '=', 'film.film_id')
    ->select(DB::raw("COUNT(inventory.film_id) count"))
    ->groupBy('inventory.film_id')
    ->where('film.title', '=', 'Hunchback Impossible')
    ->get()
```

13. Using the tables `payment` and `customer` and the `JOIN` command, list the total paid by each `customer`. List the customers alphabetically by last name:
```php
DB::table('customer')
    ->leftJoin('payment', 'customer.customer_id', '=', 'payment.customer_id')
    ->select(['customer.customer_id', DB::raw("CONCAT(first_name,' ',last_name) full_name"), DB::raw("SUM(payment.amount) as total_amount")])->groupBy('payment.customer_id')
    ->get()
```

14. The music of `Queen` and `Kris Kristofferson` have seen an unlikely resurgence. As an unintended consequence, films starting with the letters K and Q have also soared in popularity. Use subqueries to display the titles of movies starting with the letters K and Q whose language is English
```php
DB::table('film')
    ->select('title')
    ->where('title', 'LIKE', 'K%')
    ->orWhere('title', 'LIKE', 'Q%')
    ->whereRaw("language_id = (select language_id from language where name = 'English')")
    ->get()
```

15. Use subqueries to display all `actors` who appear in the film `Alone Trip`.
```php
DB::table('actor')
    ->join('film_actor', 'actor.actor_id', '=', 'film_actor.actor_id')
    ->select(DB::raw("CONCAT(first_name, ' ', last_name) full_name"))
    ->whereRaw("film_actor.film_id = (select film_id from film where title='Alone Trip')")
    ->get()
```

16. You want to run an email marketing campaign in Canada, for which you will need the `names` and `email addresses` of all Canadian customers. Use joins to retrieve this information..
```php
DB::table('customer')
    ->join('address', 'customer.address_id', '=', 'address.address_id')
    ->join('city', 'city.city_id', '=', 'address.city_id')
    ->join('country', 'country.country_id', '=', 'city.country_id')
    ->where('country.country', '=', 'Canada')
    ->select(DB::raw("CONCAT(first_name, ' ', last_name) full_name"), 'customer.email')
    ->get()
```

17. Sales have been lagging among young families, and you wish to target all family movies for a promotion. Identify all movies categorized as famiy films.
```php
DB::table('film')
    ->join('film_category', 'film.film_id', '=', 'film_category.film_id')
    ->join('category', 'category.category_id', '=', 'film_category.category_id')
    ->select(['film.title', 'category.name'])
    ->where('name', 'Family')
    ->get()
```

18. Sales have been lagging among young families, and you wish to target all family movies for a promotion. Identify all movies categorized as famiy films.
```php
DB::table('film')
    ->join('film_category', 'film.film_id', '=', 'film_category.film_id')
    ->join('category', 'category.category_id', '=', 'film_category.category_id')
    ->select(['film.title', 'category.name'])
    ->where('name', 'Family')
    ->get()
```

19. Display the most frequently rented movies in descending order.
```php
DB::table('film')
    ->join('inventory', 'film.film_id','=','inventory.film_id')
    ->join('rental', 'rental.inventory_id', '=', 'inventory.inventory_id')
    ->select(['title', DB::raw("COUNT('rental.rental_id') count")])
    ->groupBy('film.film_id')
    ->orderBy('count', 'desc')
    ->get()
```

20. Write a query to display how much business, in dollars, each store brought in.
```php
DB::table('payment')
    ->join('rental', 'payment.rental_id', '=', 'rental.rental_id')
    ->join('inventory', 'rental.inventory_id', '=', 'inventory.inventory_id')
    ->select(['inventory.store_id', DB::raw("CONCAT(SUM(payment.amount), '$') as total_amount")])
    ->groupBy('inventory.store_id')
    ->get()
```

21. Write a query to display for each store its store `ID`, `city`, and `country`.
```php
DB::table('store')
    ->join('address', 'store.address_id', '=', 'address.address_id')
    ->join('city', 'address.city_id', '=', 'city.city_id')
    ->join('country', 'city.country_id', 'country.country_id')
    ->select(['store.store_id', 'city.city', 'country.country'])
    ->get()
```

22. List the top five genres in gross revenue in descending order. (Hint: you may need to use the following tables: `category`, `film_category`, `inventory`, `payment`, and `rental`.)
```php
DB::table('payment')
    ->join('rental', 'payment.rental_id', '=', 'rental.rental_id')
    ->join('inventory', 'rental.inventory_id', '=', 'inventory.inventory_id')
    ->join('film_category', 'film_category.film_id', 'inventory.film_id')
    ->join('category', 'film_category.category_id', '=', 'category.category_id')
    ->select(['category.name',DB::raw("CONCAT(SUM(payment.amount), '$') as total_amount")])
    ->groupBy('film_category.category_id')
    ->orderBy('total_amount', 'desc')
    ->limit(5)
    ->get()
```
