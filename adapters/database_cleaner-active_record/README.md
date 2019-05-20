# Database Cleaner for ActiveRecord

[![Build Status](https://travis-ci.org/DatabaseCleaner/database_cleaner-active_record.svg?branch=master)](https://travis-ci.org/DatabaseCleaner/database_cleaner-active_record)
[![Code Climate](https://codeclimate.com/github/DatabaseCleaner/database_cleaner-active_record/badges/gpa.svg)](https://codeclimate.com/github/DatabaseCleaner/database_cleaner-active_record)

Database Cleaner is a set of strategies for cleaning your database in Ruby.

The original use case was to ensure a clean state during tests.
Each strategy is a small amount of code but is code that is usually needed in any ruby app that is testing with a database.

## Gem Setup

```ruby
# Gemfile
group :test do
  gem 'database_cleaner-active_record'
end
```

## Supported Strategies

Here is an overview of the supported strategies:

<table>
  <tbody>
    <tr>
      <th>Truncation</th>
      <th>Transaction</th>
      <th>Deletion</th>
    </tr>
    <tr>
      <td> Yes</td>
      <td> <b>Yes</b></td>
      <td> Yes</td>
    </tr>
  </tbody>
</table>

(Default strategy is denoted in bold)

For support or to discuss development please use the [Google Group](http://groups.google.com/group/database_cleaner).

## What strategy is fastest?

For the SQL libraries the fastest option will be to use `:transaction` as transactions are simply rolled back. If you can use this strategy you should. However, if you wind up needing to use multiple database connections in your tests (i.e. your tests run in a different process than your application) then using this strategy becomes a bit more difficult. You can get around the problem a number of ways.

One common approach is to force all processes to use the same database connection ([common ActiveRecord hack](http://blog.plataformatec.com.br/2011/12/three-tips-to-improve-the-performance-of-your-test-suite/)) however this approach has been reported to result in non-deterministic failures.

Another approach is to have the transactions rolled back in the application's process and relax the isolation level of the database (so the tests can read the uncommitted transactions).

An easier, but slower, solution is to use the `:truncation` or `:deletion` strategy.

So what is fastest out of `:deletion` and `:truncation`? Well, it depends on your table structure and what percentage of tables you populate in an average test. The reasoning is out of the scope of this README but here is a [good SO answer on this topic for Postgres](http://stackoverflow.com/questions/11419536/postgresql-truncation-speed/11423886#11423886).

Some people report much faster speeds with `:deletion` while others say `:truncation` is faster for them. The best approach therefore is it try all options on your test suite and see what is faster.

If you are using ActiveRecord then take a look at the [additional options](#additional-activerecord-options-for-truncation) available for `:truncation`.

## How to use

```ruby
require 'database_cleaner/active_record'

DatabaseCleaner.strategy = :truncation

# then, whenever you need to clean the DB
DatabaseCleaner.clean
```

With the `:truncation` strategy you can also pass in options, for example:

```ruby
DatabaseCleaner.strategy = :truncation, only: %w[widgets dogs some_other_table]
```

```ruby
DatabaseCleaner.strategy = :truncation, except: %w[widgets]
```

(I should point out the truncation strategy will never truncate your schema_migrations table.)

Some strategies need to be started before tests are run (for example the `:transaction` strategy needs to know to open up a transaction). This can be accomplished by calling `DatabaseCleaner.start` at the beginning of the run, or by running the tests inside a block to `Database.cleaning`. So you would have:

```ruby
require 'database_cleaner/active_record'

DatabaseCleaner.strategy = :transaction

DatabaseCleaner.start # usually this is called in setup of a test

dirty_the_db

DatabaseCleaner.clean # cleanup of the test

# OR

DatabaseCleaner.cleaning do
  dirty_the_db
end
```

At times you may want to do a single clean with one strategy.

For example, you may want to start the process by truncating all the tables, but then use the faster transaction strategy the remaining time. To accomplish this you can say:

```ruby
require 'database_cleaner/active_record'

DatabaseCleaner.clean_with :truncation

DatabaseCleaner.strategy = :transaction

# then make the DatabaseCleaner.start and DatabaseCleaner.clean calls appropriately
```

### Additional ActiveRecord options for Truncation

The following options are available for ActiveRecord's `:truncation` strategy _only_ for MySQL and Postgres.

* `:pre_count` - When set to `true` this will check each table for existing rows before truncating it.  This can speed up test suites when many of the tables to be truncated are never populated. Defaults to `:false`. (Also, see the section on [What strategy is fastest?](#what-strategy-is-fastest))
* `:reset_ids` - This only matters when `:pre_count` is used, and it will make sure that a tables auto-incrementing id is reset even if there are no rows in the table (e.g. records were created in the test but also removed before DatabaseCleaner gets to it). Defaults to `true`.

The following option is available for ActiveRecord's `:truncation` and `:deletion` strategy for any DB.

* `:cache_tables` - When set to `true` the list of tables to truncate or delete from will only be read from the DB once, otherwise it will be read before each cleanup run. Set this to `false` if (1) you create and drop tables in your tests, or (2) you change Postgres schemas (`ActiveRecord::Base.connection.schema_search_path`) in your tests (for example, in a multitenancy setup with each tenant in a different Postgres schema). Defaults to `true`.


### RSpec Example

```ruby
RSpec.configure do |config|

  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end

end
```

### RSpec with Capybara Example

You'll typically discover a feature spec is incorrectly using transaction
instead of truncation strategy when the data created in the spec is not
visible in the app-under-test.

A frequently occurring example of this is when, after creating a user in a
spec, the spec mysteriously fails to login with the user. This happens because
the user is created inside of an uncommitted transaction on one database
connection, while the login attempt is made using a separate database
connection. This separate database connection cannot access the
uncommitted user data created over the first database connection due to
transaction isolation.

For feature specs using a Capybara driver for an external
JavaScript-capable browser (in practice this is all drivers except
`:rack_test`), the Rack app under test and the specs do not share a
database connection.

When a spec and app-under-test do not share a database connection,
you'll likely need to use the truncation strategy instead of the
transaction strategy.

See the suggested config below to temporarily enable truncation strategy
for affected feature specs only. This config continues to use transaction
strategy for all other specs.

It's also recommended to use `append_after` to ensure `DatabaseCleaner.clean`
runs *after* the after-test cleanup `capybara/rspec` installs.

```ruby
require 'capybara/rspec'

#...

RSpec.configure do |config|

  config.use_transactional_fixtures = false

  config.before(:suite) do
    if config.use_transactional_fixtures?
      raise(<<-MSG)
        Delete line `config.use_transactional_fixtures = true` from rails_helper.rb
        (or set it to false) to prevent uncommitted transactions being used in
        JavaScript-dependent specs.

        During testing, the app-under-test that the browser driver connects to
        uses a different database connection to the database connection used by
        the spec. The app's database connection would not be able to access
        uncommitted transaction data setup over the spec's database connection.
      MSG
    end
    DatabaseCleaner.clean_with(:truncation)
  end  

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each, type: :feature) do
    # :rack_test driver's Rack app under test shares database connection
    # with the specs, so continue to use transaction strategy for speed.
    driver_shares_db_connection_with_specs = Capybara.current_driver == :rack_test

    if !driver_shares_db_connection_with_specs
      # Driver is probably for an external browser with an app
      # under test that does *not* share a database connection with the
      # specs, so use truncation strategy.
      DatabaseCleaner.strategy = :truncation
    end
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.append_after(:each) do
    DatabaseCleaner.clean
  end

end
```

### Minitest Example

```ruby
DatabaseCleaner.strategy = :transaction

class Minitest::Spec
  before :each do
    DatabaseCleaner.start
  end

  after :each do
    DatabaseCleaner.clean
  end
end

# with the minitest-around gem, this may be used instead:
class Minitest::Spec
  around do |tests|
    DatabaseCleaner.cleaning(&tests)
  end
end
```

### Cucumber Example

If you're using Cucumber with Rails, just use the generator that ships with cucumber-rails, and that will create all the code you need to integrate DatabaseCleaner into your Rails project.

Otherwise, to add DatabaseCleaner to your project by hand, create a file `features/support/database_cleaner.rb` that looks like this:

```ruby
begin
  require 'database_cleaner/active_record'
  require 'database_cleaner/cucumber'

  DatabaseCleaner.strategy = :truncation
rescue NameError
  raise "You need to add database_cleaner to your Gemfile (in the :test group) if you wish to use it."
end

Around do |scenario, block|
  DatabaseCleaner.cleaning(&block)
end
```

This should cover the basics of tear down between scenarios and keeping your database clean.

### Configuration options

<table>
  <tbody>
    <tr>
      <th>ORM</th>
      <th>How to access</th>
      <th>Notes</th>
    </tr>
    <tr>
      <td> Active Record </td>
      <td> <code>DatabaseCleaner[:active_record]</code></td>
      <td> Connection specified as <code>:symbol</code> keys, loaded from <code>config/database.yml</code>. You may also pass in the ActiveRecord model under the <code>:model</code> key.</td>
    </tr>
  </tbody>
</table>

## Common Errors

#### DatabaseCleaner is trying to use the wrong ORM

DatabaseCleaner has an autodetect mechanism where if you do not explicitly define your ORM it will use the first ORM it can detect that is loaded.

Since ActiveRecord is the most common ORM used that is the first one checked for.

Sometimes other libraries (e.g. ActiveAdmin) will load other ORMs (e.g. ActiveRecord) even though you are using a different ORM.  This will result in DatabaseCleaner trying to use the wrong ORM (e.g. ActiveRecord) unless you explicitly define your ORM like so:

```ruby
# How to setup your ORM explicitly
DatabaseCleaner[:mongoid].strategy = :truncation
```

### STDERR is being flooded when using Postgres

If you are using Postgres and have foreign key constraints, the truncation strategy will cause a lot of extra noise to appear on STDERR (in the form of "NOTICE truncate cascades" messages).

To silence these warnings set the following log level in your `postgresql.conf` file:

```ruby
client_min_messages = warning
```

For ActiveRecord, you add the following parameter in your database.yml file:

<pre>
test:
  adapter: postgresql
  # ...
  min_messages: WARNING  
</pre>

## Debugging

In rare cases DatabaseCleaner will encounter errors that it will log.  By default it uses STDOUT set to the ERROR level but you can configure this to use whatever Logger you desire.

Here's an example of using the `Rails.logger` in `env.rb`:

```ruby
DatabaseCleaner.logger = Rails.logger
```

## COPYRIGHT

See [LICENSE] for details.