# TrafficJam

[![Build Status](https://travis-ci.org/coinbase/traffic_jam.svg?branch=master)](https://travis-ci.org/coinbase/traffic_jam)
[![Coverage Status](https://coveralls.io/repos/coinbase/traffic_jam/badge.svg?branch=master)](https://coveralls.io/r/coinbase/traffic_jam?branch=master)

This is a library for enforcing rate limits across concurrent processes. This can be used to cap the number of actions that may be performed in a certain period of time. More generally, this can be used to enforce a cap on an integer amount that can be incremented or decremented. A limit consists of an action name, a maximum amount, and a period of time in seconds.

Instead of guaranteeing that the number of actions will never exceed the cap the given timeframe, the approach we take is to use a continuously regenerating limit. The amount remaining will constantly increase at a rate of *max / period* until it hits the cap. If, for example, the limit is 60 per minute, a user could increment by 60 at once, then increment by 1 per second forever without hitting the cap. As a consequence, *this algorithm guarantees that the total amount incremented will be less than twice the limit in any given timeframe*.

## Usage

```ruby
require 'traffic_jam'

TrafficJam.configure do |config|
  config.redis = Redis.new(url: REDIS_URL)
end

limit = TrafficJam::Limit.new(
  :requests_per_user,
  user.id,
  max: 3,
  period: 1 # seconds
)
limit.increment      # => true
limit.increment(2)   # => true
limit.increment      # => false
limit.increment!     # => raises TrafficJam::LimitExceededError

sleep 1

limit.increment(2)   # => true
limit.exceeded?      # => false
limit.exceeded?(2)   # => true

limit.used           # => 2
limit.remaining      # => 1
```

## Configuration

TrafficJam configuration object can be accessed with `TrafficJam.config` or in a block like `TrafficJam.configure { |config| ... }`. Configuration options are:

**redis** (required): A Redis instance to store amounts used for each limit.

**key_prefix** (default: "traffic_jam"): The string prefixing all keys in Redis.

**logger** (optional): A logger used to log all exceeded rate limits.

### Registering limits

Fixed limits can be registered for a key if the cap does not change depending on the value. All instance methods are available on the class.

```ruby
TrafficJam.configure do |config|
  config.register(:requests_per_user, 3, 1)
end

limit = TrafficJam.limit(:requests_per_user, "user1")
limit.increment(2)  # => true

TrafficJam.increment(:requests_per_user, "user1", 1)  # => true
TrafficJam.used(:requests_per_user, "user1")          # => 3
```

## Changing cap for a limit

Given an instance of `TrafficJam::Limit` with a maximum cap and a period, the behavior is to increase the amount remaining at a rate of *max / period* since the last time `increment` was called for the given value. If the cap is defined on a per-value basis, it is good practice to call `increment(0)` if the limit changes.

For example:

```ruby
user.requests_per_hour = 10

limit = TrafficJam::Limit.new(
  :requests_per_user, user.id,
  max: user.requests_per_hour, period: 60 * 60
)
limit.increment(8)  # => true

sleep 60

limit.increment(0)

user.requests_per_hour = 20
limit = TrafficJam::Limit.new(
  :requests_per_user, user.id,
  max: user.requests_per_hour, period: 60 * 60
)
limit.increment(8)  # => true
```

## Running tests

The `REDIS_URI` environment variable can be set in tests, and defaults to `redis://localhost:6379`.

```
rake test
```

To run a performance/stress test, see the `test/stress.rb` script.
