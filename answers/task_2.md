# Task 2: “Over quota” errors for Australian users

[< Back to solutions menu](../readme.md)

Users in Australia are complaining they still get some “over quota” errors from the API after their quota resets at the beginning of the month and after a few hours it resolves itself. A user provided the following access logs:

```
timestamp                 | url                        | response
2022-10-31T23:58:01+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-10-31T23:59:17+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-11-01T01:12:54+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-11-01T01:43:11+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-11-01T16:05:20+10:00 | https://acme.corp/api/g6az | { data: [{ ... }] }
2022-11-01T16:43:39+10:00 | https://acme.corp/api/g6az | { data: [{ ... }] }
```

## Solution

The User#count_hits method fetches hits since the beginning of the current month in the server's time zone. However, Australian users are ahead of UTC. So, even if it's a new month for them, the server might still consider it as the previous month. This is why they still get "over quota" errors.

The best solution at this case is set the user's timezone (If Available on model) to determine the beginning of the month.

IMPORTANT: In case the user's timezone is not available, we can use the request's timezone and in the last case both are unavailable then use the server's timezone.

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
    Rails.cache.increment("#{self.id}/hits_count")
  end

  def current_user_time(request_timezone: 'UTC')
    @current_user_time ||= Time.now.in_time_zone(self.timezone || request_timezone)
  end
end
```
