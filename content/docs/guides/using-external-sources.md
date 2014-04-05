---
title: "Using external sources"
---

One of nanoc’s strengths is the ability to pull in data from external systems. In this guide, we’ll show how to do this, and what you can achieve with it.

## Reading from a database

Assume you work for a company that has a SQL database with HR information, such as an employee directory, and you want to extract that information onto a nice-looking web site. The schema could look like this (in SQLite3):

	#!sql
	CREATE TABLE employees (
	    id         INTEGER PRIMARY KEY NOT NULL,
	    first_name TEXT NOT NULL,
	    last_name  TEXT NOT NULL,
	    photo_url  TEXT NOT NULL
	)

The way to pull in this data would be to generate items for every employee. Such an item would not have any content (except perhaps the employee’s bio), but it would store its details as attributes.

TIP: It helps to not think of nanoc items as just pages or assets. They can represent much more varied data, such as recipes, reviews, teams, people (with a team identifier), projects (with a team identifier), etc.

For demonstration purposes, I recommend using [Sequel](http://sequel.jeremyevans.net/) in combination with [SQLite3](https://sqlite.org/). To install the dependencies:

<pre>
<span class="prompt">%</span> <kbd>gem install sequel</kbd>
<span class="prompt">%</span> <kbd>gem install sqlite3</kbd>
</pre>

Create a sample database with sample data:

	#!ruby
	require 'sequel'

	DB = Sequel.sqlite('test.db')

	sql = <<EOS
	CREATE TABLE employees (
	    id         INTEGER PRIMARY KEY NOT NULL,
	    first_name TEXT NOT NULL,
	    last_name  TEXT NOT NULL,
	    photo_url  TEXT NOT NULL
	)
	EOS

	DB[:employees].insert(
	    id: 1,
	    first_name: 'Denis',
	    last_name: 'Defreyne',
	    photo_url: 'http://employees.test/photos/1.png'
	)

Create the file <span class="filename">lib/data_sources/employee_db</span> and put in the following:

	#!ruby
	class HRDataSource < ::Nanoc::DataSource
	end

A data source is responsible for loading data, and is represented by the `Nanoc::DataSource` class. Each data source has a unique identifier, which is a Ruby symbol. Add the line `identifier :hr` inside the class:

	#!ruby
	class HRDataSource < ::Nanoc::DataSource

	  identifier :hr

	end

Data sources have an `#up` method, which should be overriden to perform actions to establish a connection with the remote data source. In this example, implement it so that it connects to the database:

	#!ruby
	class HRDataSource < ::Nanoc::DataSource

	  identifier :hr

	  def up
	    @db = Sequel.sqlite('test.db')
	  end

	end

Then we can generate items:

	#!ruby
	class HRDataSource < ::Nanoc::DataSource

	  identifier :hr

	  def up
	    @db = Sequel.sqlite('test.db')
	  end

	  def items
	    @db[:employees].map do |employee|
	      Nanoc::Item.new(
	        '',
	        employee,
	        "/employees/#{employee[:id]}/"
	      )
	    end
	  end

	end

Configure the data source in <span class="filename">nanoc.yaml</span>:

	#!yaml
	data_sources:
	  -
	    type:         hr
	    items_root:   /external/hr

The `type` is the same as the identifier of the data source, and `items_root` is a string that will be prefixed to all item identifiers coming from that data source. For example, employees will have identifiers like `"/external/hr/employees/1/"`.

Create the file <span class="filename">lib/helpers.rb</span> and put in one function that will make it easier to find employees:

	#!ruby
	def sorted_employees
	  employees = @items.select do |i|
	    i.identifier.start_with?('/external/hr/employees/')
	  end
	  employees.sort_by do |e|
	    [ e[:last_name], e[:first_name] ]
	  end
	end

Create the file <span class="filename">content/employees.erb</span> and put in code that finds all employees and prints them:

	#!html
	<h1>Employees</h1>

	<table>
	  <thead>
	    <tr>
	      <th>Photo</th>
	      <th>First name</th>
	      <th>Last name</th>
	    </tr>
	  </thead>
	  <tbody>
	    <% sorted_employees.each do |e| %>
	      <tr>
	        <td><%= e[:photo_url] %></td>
	        <td><%= e[:first_name] %></td>
	        <td><%= e[:last_name] %></td>
	      </tr>
	    <% end %>
	  </tbody>
	</table>

Compile, and you will see an employee directory!

## Reading from a web API

TODO: Write me. Maybe use a conference planning system as an example. Or recipes, reviews, …
