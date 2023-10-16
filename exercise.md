# Staff Engineer coding assignment

# Brief

Acme Corp. has an API service that allows registered users to fetch any information about the company and its wide array of products. It quickly became a very popular service and, to provide fair service to all users, a 10,000 API requests per-user monthly limit was added. Not long after the change, users started to report that their requests were very slow or not returning at all.

Acme decided to ask the community for help. A task list was provided with the top issues affecting their API. They've also released parts of the code base they've identified as being problematic.

## Code

```ruby
class User < ApplicationRecord
	has_many :hits

	def count_hits
		start = Time.now.beginning_of_month
		hits = hits.where('created_at > ?', start).count
		return hits
  end
end
```

```ruby
class ApplicationController < ActionController::API
	before_filter :user_quota

	def user_quota
		render json: { error: 'over quota' } if current_user.count_hits >= 10000
  end
end
```

```sql
Table "public.hits"
   Column   |            Type             | Collation | Nullable | Default
------------+-----------------------------+-----------+----------+---------
 id         | bigint                      |           | not null |
 user_id    | integer                     |           | not null |
 endpoint   | character varying           |           | not null |
 created_at | timestamp without time zone |           | not null |
Indexes:
    "hits_pkey" PRIMARY KEY, btree (id)
    "index_hits_on_user_id" btree (user_id)
```

# Tasks

## #1

Requests to Acme's API are timing out after 15 seconds. After some investigation in production, they've identified that the `User#count_hits` method's response time is around 500ms and is called 50 times per second (on average).

### Objective

Make changes to the code provided so that the API response time is back to acceptable levels and users no longer see timeouts. Feel free to use additional gems, tools, methods, or techniques.

## #2

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

### Objective

Describe the root cause of the issue and provide a fix for it.

## #3

Acme identified that some users have been able to make API requests over the monthly limit.

### Objective

Describe how can that be possible and provide a fix for it.

## #4

Acme's development team has reported working with the code base is difficult due to accumulated technical debt and bad coding practices. They've asked the community to help them refactor the code so it's clean, readable, maintainable, and well-tested.

### Objective

Make changes to the code so it matches Acme's new code quality standards.

### Solution

[View Solution](./readme.md)
