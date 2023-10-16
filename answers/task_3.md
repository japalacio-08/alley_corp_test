# Task 3: Users making API requests over the monthly limit

[< Back to solutions menu](../readme.md)

Acme identified that some users have been able to make API requests over the monthly limit.

## Solution

Potential issues probably are:

1. Multiple simultaneous requests => It can cause the hit count to be inaccurate => Answer: Atomic Operations Please check `User#record_hit` method (It was resolved previously).

2. If the cache get evicted or expires prepaturely, the hit count will be inaccurate => Answer: Set the cache expiration time to 1 month and use a persistent store like Redis with appropiate backup rules and eviction policies.

```Ruby
# Gemfile
gem 'redis-rails'
```

```Ruby
class User < ApplicationRecord
  has_many :hits
  after_create :record_hit

  def count_hits(request_timezone: 'UTC')
    Rails.cache.fetch("#{self.id}/hits_count", expires_in: 1.month) do
      start = current_user_time(request_timezone:).beginning_of_month
      hits.where('created_at > ?', start).count
    end
  end

  def record_hit
    Rails.cache.increment("user/#{self.id}/hits_count")
  end

  def current_user_time(request_timezone: 'UTC')
    @current_user_time ||= Time.now.in_time_zone(self.timezone || request_timezone)
  end
end
```
