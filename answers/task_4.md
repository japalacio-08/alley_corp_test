# Task 4: Refactor for code quality

[< Back to solutions menu](../readme.md)

Acme's development team has reported working with the code base is difficult due to accumulated technical debt and bad coding practices. They've asked the community to help them refactor the code so it's clean, readable, maintainable, and well-tested.

## Solution

Separation of concerns, Single Responsibility Principle, DRY, etc.

Write unit tests to ensure code correctness

Use meaningful variable names

```Ruby
# Gemfile
gem 'redis-rails'
```

```Ruby
# app/models/user.rb
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

```Ruby
# app/services/user_quota_service.rb
class UserQuotaService
  attr_reader :user

  def initialize(user)
    @user = user
  end

  def over_quota?
    user.count_hits >= 10_000
  end
end
```

```Ruby
# app/exceptions/custom.rb
module Exceptions::Custom
  class OverQuota < StandardError; end
end
```

```Ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_user! # Devise validation for current_user
  before_action :user_quota

  rescue_from Exceptions::Custom::OverQuota, with: :over_quota_response

  # ...

  private

  def user_quota
    quota_service = UserQuotaService.new(current_user)
    raise Exceptions::Custom::OverQuota, 'over quota' if quota_service.over_quota?
  end

  def over_quota_response(exception)
    render json: { error: exception.message }, status: :unprocessable_entity
  end
end
```

## Tests | Requirements

* Add Gems to Gemfile

```Ruby
gem 'rspec-rails', '~> <version>'
gem 'factory_bot_rails'
```

* Install and config rspec

```Shell
rails generate rspec:install
```

* Create User Factory

```Ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    # Add attributes here, e.g. email, name, timezone, etc.
  end
end
```

## Tests | User Model

```Ruby
# spec/models/user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  describe 'after_create callback' do
    it 'records a hit when a user is created' do
      expect(Rails.cache).to receive(:increment).with(include('hits_count'))
      create(:user)
    end
  end

  describe '#count_hits' do
    let(:user) { create(:user) }
    let(:time) { Time.now }

    before do
      allow(Time).to receive(:now).and_return(time)
    end

    context 'when hits are not cached' do
      it 'counts the hits from the database' do
        hits_count = rand(1..10_000)
        allow(user.hits).to receive_message_chain(:where, :count).and_return(hits_count)
        expect(user.count_hits).to eq(hits_count)
      end
    end

    context 'when hits are cached' do
      let(:cached_hits_count) { rand(1..10_000) }

      before do
        Rails.cache.write("#{user.id}/hits_count", cached_hits_count)
      end

      it 'fetches the hits count from the cache' do
        expect(user.count_hits).to eq(cached_hits_count)
      end
    end

    context 'when considering timezones' do
      it 'takes the user timezone into account if available' do
        user_timezone = 'Asia/Tokyo'
        user.update(timezone: user_timezone)
        expect(Time).to receive(:now).and_return(time).in_time_zone(user_timezone)
        user.count_hits
      end

      it 'takes the request timezone into account if user timezone is not set' do
        request_timezone = 'Europe/London'
        expect(Time).to receive(:now).and_return(time).in_time_zone(request_timezone)
        user.count_hits(request_timezone: request_timezone)
      end
    end
  end

  describe '#record_hit' do
    let(:user) { create(:user) }

    it 'increments the hits count in the cache' do
      initial_hits = Rails.cache.read("user/#{user.id}/hits_count") || 0
      user.record_hit
      expect(Rails.cache.read("user/#{user.id}/hits_count")).to eq(initial_hits + 1)
    end
  end

  describe '#current_user_time' do
    let(:user) { create(:user) }
    let(:time) { Time.now }

    before do
      allow(Time).to receive(:now).and_return(time)
    end

    it 'returns the current time in user timezone if available' do
      user_timezone = 'Asia/Tokyo'
      user.update(timezone: user_timezone)
      expect(user.current_user_time).to eq(time.in_time_zone(user_timezone))
    end

    it 'returns the current time in request timezone if user timezone is not set' do
      request_timezone = 'Europe/London'
      expect(user.current_user_time(request_timezone: request_timezone)).to eq(time.in_time_zone(request_timezone))
    end
  end
end
```

## Tests | User Quota Service

```Ruby
# spec/services/user_quota_service_spec.rb
require 'rails_helper'

RSpec.describe UserQuotaService, type: :service do
  let(:user) { create(:user) }
  subject { UserQuotaService.new(user) }

  describe '#over_quota?' do
    context 'when user is under quota' do
      before do
        allow(user).to receive(:count_hits).and_return(5_000)
      end

      it 'returns false' do
        expect(subject.over_quota?).to be_falsey
      end
    end

    context 'when user is over quota' do
      before do
        allow(user).to receive(:count_hits).and_return(10_001)
      end

      it 'returns true' do
        expect(subject.over_quota?).to be_truthy
      end
    end
  end
end
```

## Tests | Application Controller

```Ruby
# spec/controllers/application_controller_spec.rb
require 'rails_helper'

RSpec.describe ApplicationController, type: :controller do
  controller do
    def index
      render json: { message: 'success' }
    end
  end

  describe 'Quota verification' do
    let(:user) { create(:user) }

    before do
      sign_in user # Devise helper for authentication
    end

    context 'when user is under quota' do
      before do
        allow_any_instance_of(User).to receive(:count_hits).and_return(5_000)
      end

      it 'renders a successful response' do
        get :index
        expect(response).to have_http_status(:ok)
      end
    end

    context 'when user is over quota' do
      before do
        allow_any_instance_of(User).to receive(:count_hits).and_return(10_001)
      end

      it 'renders an error response' do
        get :index
        parsed_response = JSON.parse(response.body)
        expect(parsed_response['error']).to eq('over quota')
        expect(response).to have_http_status(:unprocessable_entity)
      end
    end
  end
end

```
