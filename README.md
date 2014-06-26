# once-ler

once-ler supercharges your `let`s and `before`s with the performance
of `before(:all)`.

## Installation

```ruby
gem "once-ler"
```

## Basic usage

### before(:once) { ... }

Change a slow `before` to `before(:once)`

### let_once(...) { ... }

Change a slow `let` (or `let!`) to `let_once`

## Ambitious usage

Automatically speed up all `let`s/`before`s in an example group:

    describe "something" do
      onceler!
      ...
    end

Or even more ambitiously, apply it to all your specs:

    RSpec.configure do |c|
      c.onceler!
    end

## How does it work?

Any `before(:once)`/`let_once` blocks will run just once for the current
context/describe block, before any of its examples run. Any side effects
(ivars) and return values will be recorded and replayed before each spec
in the block runs. Database statements are replayed/rolled back via
nested transactions (savepoints).

This can give you a dramatic speedup, since you can minimize the number
of activerecord callbacks/inserts/updates.

## Caveats

* You need to use transactional fixtures.
* Your once'd blocks should have no side effects other than database
  statements, return values, and instance variables.
* Your return values and instance variables need to be able to handle a
  Marshal.dump/load round trip
* Your once'd blocks' behavior should not depend on side effects of other
  non-once'd blocks. For example:
  * a `before(:once)` block should not reference instance variables set by a
    `before`.
  * a `let_once` block should not call non-once'd `let`s or `subject`s.
* Because all `let_once`s will be recorded and replayed (even if not used
  in a particular example), you should ensure they don't conflict with
  each other (e.g. unique constraint violations, or one `let_once`
  mutating the return value of another).
* Some effort is made to preserve object identity, but just for instance
  variables and return values, e.g.:

  ```ruby
    let_once(:user) { User.new }
    let_once(:users) { [user] }

    before(:once) do
      @joe = user
      @also_joe = @joe
      @joe_ish = [@joe]
    end

    # within an example:
    # user == @joe             => true
    # user.equal? @joe         => true # yay
    # user == @also_joe        => true
    # user.equal? @also_joe    => true # yay
    # user == users[0]         => true
    # user.equal? users[0]     => false # d'oh
    # user == @joe_ish[0]      => true
    # user.equal? @joe_ish[0]  => false # d'oh
  ```
