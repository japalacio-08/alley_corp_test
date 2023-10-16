# Task 1: Improve API Response Time

[< Back to solutions menu](../readme.md)

Requests to Acme's API are timing out after 15 seconds. After some investigation in production, they've identified that the `User#count_hits` method's response time is around 500ms and is called 50 times per second (on average).

## Solution

We can use Rails caching to store the hits count for each user with an expiration time of 1 hour. This will reduce the number of database queries and improve the response time of the `User#count_hits` method.

```Ruby
# Gemfile
gem 'redis-rails'
```

```Ruby
class User < ApplicationRecord
  has_many :hits
  after_create :record_hit

  def count_hits
    Rails.cache.fetch("#{self.id}/hits_count", expires_in: 1.month) do
      start = Time.now.beginning_of_month
      hits.where('created_at > ?', start).count
    end
  end

  def record_hit
    Rails.cache.increment("#{self.id}/hits_count")
  end
end
```
