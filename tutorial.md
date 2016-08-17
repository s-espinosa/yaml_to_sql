# Moving from YAML to SQLite

## Overview
We're going to take TaskManager and convert our database storage from YAML::Store into SQLite3. We could use ActiveRecord, DataMapper, or Sequel, but that would be too easy. Before we use these tools that exist, we are going to build an ORM ourselves.

At a high level the steps will include the following:

1. Remove YAML::Store file
2. Create our SQLite database
3. Modify how we connect to the database
4. Modify our methods in TaskManager

Some questions for you to consider as you work through this tutorial:

* How do we know as we make changes if we've broken things?
* What do we need to do to accomplish this transformation?

## Remove YAML::Store

First things first. We're working without a net. Let's remove the task_manager and task_manager_test files from our db folder. These are the YAML files that we've been using to store data up to this point. Without these, we effectively don't have a database in our project.

## Create Our SQLite Database

### Getting Started

To get started with SQL in our project, add the SQLite gem to our Gemfile with the following line:

```
gem 'sqlite3'
```

Congrats! You can now use SQLite3 in your project.

### Migrations

That's great, but how do we actually put that gem to work? We need to create a database and add some tables. We could potentially access those things directly from our terminal, but that's no fun, and it makes it pretty difficult to work with other people. Instead, let's create some migrations.

Migrations allow you to evolve your database structure over time. Each migration includes instructions to make some change to the database (e.g. adding a table, adding columns to a table, dropping a table, etc.). One advantage to this approach is that it will allow you to transfer the application to different computers without transferring the whole database. This isn't a big problem for us now, but as your database grows it will be advantageous to be able to transfer the *instructions to create the database* instead of the database itself.

Once we start using an existing ORM (instead of creating our own), we'll need to be particular about where our migrations go and be mindful of how we name them and the content. For this tutorial, to the degree possible, we're going to work to replicate some of the naming conventions that we expect you to see in the future.

Let's start with a migration to create the database and an initial table. Create a directory for your migrations and then a file to hold your first migration:

```
$ mkdir db/migrations
$ touch db/migrations/001_create_tasks.rb
```

The 001 in the file name above helps us keep our migrations in order. If we create a table in one migration and then later in our project decide that we need to add a column to that table, the file names will keep us from trying to add a column to a table that doesn't yet exist.

Go ahead and add the code below to your newly created migration.

```
# db/migrations/001_create_tasks.rb

require 'sqlite3'

environments = ["test", "development"]

environments.each do |environment|
  database = SQLite3::Database.new("db/task_manager_#{environment}.db")
  database.execute("CREATE TABLE tasks (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          title VARCHAR(64),
          description VARCHAR(64)
        );"
  )
  puts "creating tasks table for #{environment}"
end
```

Let's go ahead and run the code in this file with:

```
$ ruby db/migrations/001_create_tasks.rb
```

Congrats! You have a database? Actually, you have two (one for test, and one for development). I mean, the command line told me so, right?

### Are you sure?

We have a migration. We ran the migration. How do we know if our database was created?

We can open our database using the sqlite3 gem with the following command:

```
$ sqlite3 db/task_manager_development.db
```

That will bring us into the command line shell for SQLite3. Let's see if we've successfully created the table we wanted to create. Run the following line of code in the shell:

```
sqlite> .schema
```

You should see a description of the table that you just created in your database. Ideally there is some reference to an id, title, and description included in the response you see. That should give you some comfort that your table has been created with the proper headers.

We can also use some SQL to see what's in the database. Go ahead and use the following code to select all rows from your tasks table.

```
sqlite> SELECT * FROM tasks;
```

That was a little bit anticlimactic. I expect that you didn't see any output. Take solace in the fact that if you didn't have a table in your database you would've seen an error.

To exit use `Ctrl-\`

We could add some data to our table from this interface, but that's not exactly what web development is all about. Plus it doesn't feel very efficient...

### Seed Data

For development purposes, sometimes you'll see what's called a seed file. This file adds data to your database so that you have existing data you can view/manipulate without having to enter data in your browser. Let's go ahead and create a seed file to populate our new database.

Go ahead and create a file `db/seeds.rb` and enter the following code into it.

```
# db/seeds.rb

require 'sqlite3'

database = SQLite3::Database.new("db/task_manager_development.db")

#Delete existing records from the tasks table before inserting these new records; we'll start from scratch.
database.execute("DELETE FROM tasks")

# Insert records.
database.execute("INSERT INTO tasks
                (title, description)
                VALUES
                  ('Go to the Gym', 'exercise is good for you'),
                  ('Make dinner', 'obviously'),
                  ('Pay the electric bill', 'it is due in 10 days'),
                  ('Book a flight home for vacation', 'End of August');"
                )

# verifying that our SQL INSERT statement worked
puts "It worked and:"
p database.execute("SELECT * FROM tasks;")
```

Go ahead and run this file with `ruby db/seeds.db`

You should see the output from the puts and p command at the end of that file showing some items that have been inserted into your database. Since we're actually using an SQL query in that last line we should feel pretty comfortable that everything has been entered into the database correctly. If you're curious, go ahead and run the commands from the end of the previous section to see how the output in the command line interface changes now that you have some data.

One quick note: you'll notice that we're again using #new to create a database object that we can use in our Ruby code, but at this point the database itself already exists. SQLite3::Database.new will first try to access the database passed to it as an argument, and then create a new database if no such database exists.

## Modify how we connect to the database

Now we've seen that we've successfully created a database. We know that we can add data to our database using our seed file. Let's link our app to the new database that we've created.

### Updating our database connection in our controller:

First, in the controller, let's go ahead and replace our current definition of task_manager with a new version that replaces the YAML references with references to our new SQL database using the code below.

In your `app/task_manager_app.rb` file:

```
# app/task_manager_app.rb
def task_manager
  if ENV['RACK_ENV'] == "test"
    database = SQLite3::Database.new('db/task_manager_test.db')
  else
    database = SQLite3::Database.new('db/task_manager_development.db')
  end
  database.results_as_hash = true
  TaskManager.new(database)
end
```

For more information on the #results_as_hash method take a look at the documentation [here](http://www.rubydoc.info/github/luislavena/sqlite3-ruby/SQLite3%2FDatabase%3Aresults_as_hash).

### Updating our database connection in tests:

Let's go ahead and add a similar change to our test_helper. Replace the task_manager definition in the `test/test_helper.rb` with the code below.

```
# test/test_helper.rb

def task_manager
  database = SQLite3::Database.new('db/task_manager_test.db')
  database.results_as_hash = true
  TaskManager.new(database)
end
```

## Modify our methods in TaskManager

Finally, let's go ahead and update our TaskManager model to use the SQL database that we're now passing it. The syntax will be a little bit different.

### Accessing our database in our TaskManager model

Previously, in our `app/models/task_manager.rb` we accessed our data in our YAML::Store file like so:

```
database.transaction do
    # modify yaml
end
```

Now, we'll access our data in our SQLite3 database with code similar to the following:

```
database.execute("STRING OF SQL HERE;")
```

Each of the following broken methods (or the helper methods that make them work) will need to be updated to work using the new syntax above:

* create
* find
* all
* update
* delete
* delete_all

Let's do the first one together. In your #all method you currently have the following:

```
raw_tasks.map { |data| Task.new(data) }
```

There's no reference here directly to the database, but if we dig a little deeper into that #raw_tasks method that's where the database magic is happening. Let's go ahead and change that #raw_tasks method to use our new SQL database. We want to return all the tasks just in a hash, so...

```
def self.raw_tasks
  database.execute("SELECT * FROM tasks")
end
```

That code should select all rows and all columns from our tasks table and return it to us as a hash, which is what our #all method expects.

Work on the remaining methods to see if you can get them to work (feel free to discuss with the people around you). We'll meet back up before the end of class to discuss any issues you might have found. If you finish updating all the methods before we come back together, go ahead and start working on your homework for this evening.

## Homework/Worktime

Transform Skill Inventory or Robot World to store your data in a SQLite3 database instead of YAML.

## Resources

### Code

* Code [before](https://github.com/turingschool-examples/task-manager/tree/feature-testing-implemented-1605) transformation (storing data w/ YAML)
* Code [after](https://github.com/turingschool-examples/task-manager/tree/yaml-to-sqlite3-1605) transformation (storing data w/ SQLite3)

### Other Sites

* [ORM Ruby Introduction](https://www.sitepoint.com/orm-ruby-introduction/)
* [ActiveRecord Documentation](http://guides.rubyonrails.org/active_record_basics.html)
* [DataMapper Documentation](http://datamapper.org/)
* [Sequel Documentation](http://sequel.jeremyevans.net/)
