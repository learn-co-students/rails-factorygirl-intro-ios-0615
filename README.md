

# FactoryGirl Introduction

FactoryGirl is a DSL for creating fixtures in your RSpec tests. Why is FactoryGirl a good gem to utilize? The major reason is test maintenance. Over the course of a Rails project, your assumptions about your models is likely to change over time. Let's take an example. What if we had to create normal fixtures/objects in our tests like below?

```ruby
before(:each) do
  product = Product.create(name: "Nerf Gun", type: "weapon", age: "13 and above")
end
```

Do you notice anything that would be hard to maintain? What if we changed the `type` attribute to `classification` attribute? If we had to go into every test suite and change that attribute, that would be a massive pain. And that's only for one model fixture/object. You can imagine how time consuming that would be if you had to fix this all across your test suites. It would be nice if we could do something like this:

```ruby
before(:each) do
  product = create(:product)
end
```

This is essentially what FactoryGirl does.

## Setting Up RSpec/FactoryGirl

Let's go ahead and include `rspec-rails`, `factory_girl_rails`, and `database_cleaner` in our Gemfile.

```ruby
# Gemfile

group :test, :development do
  gem 'rspec-rails'
  gem 'factory_girl_rails'
  gem 'database_cleaner'
end
```

If you put the `factory_girl_rails` gem only in the `:test` group and not the `:development` group as well, then the factories will not automatically be generated when a model spec is generated.

After we've run `bundle install`, lets go ahead and generate the RSpec resources. Run `rails g rspec:install`. You'll see the following output:

```
      create  .rspec
      create  spec
      create  spec/spec_helper.rb
      create  spec/rails_helper.rb
```

Then the next step is to add some configurations for FactoryGirl and DatabaseCleaner into our `rails_helper.rb` file. 

```ruby
# spec/rails_helper.rb

ENV['RAILS_ENV'] ||= 'test'
require 'spec_helper'
require File.expand_path('../../config/environment', __FILE__)
require 'rspec/rails'

ActiveRecord::Migration.maintain_test_schema!

RSpec.configure do |config|
  # Remove this line if you're not using ActiveRecord or ActiveRecord fixtures
  config.fixture_path = "#{::Rails.root}/spec/fixtures"

  config.use_transactional_fixtures = false

  config.include FactoryGirl::Syntax::Methods

  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.start
  end

  config.before(:each, :js => true) do
    DatabaseCleaner.strategy = :truncation
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end

  config.infer_spec_type_from_file_location!
end

```

The `FactoryGirl::Syntax::Methods` configuration allows you to just write `create(:factory)` instead of `FactoryGirl.create(:factory)` inside of a spec. 

The `DatabaseCleaner` configuration ensures that you're cleaning out examples made for each test, so that you don't risk contaminating other fixtures created for other tests. This ensures that there is a clean database slate for each test that is run in a suite.

Now let's go ahead and generate our RSpec tests for our Post and Comment models. Run the following commands in your Terminal: `rails g rspec:model Post` and `rails g rspec:model Comment`. You'll see the following output:

```
rails g rspec:model Post
      create  spec/models/post_spec.rb
      invoke  factory_girl
      create    spec/factories/posts.rb

rails g rspec:model Comment
      create  spec/models/comment_spec.rb
      invoke  factory_girl
      create    spec/factories/comments.rb
```

Now we've got our test environment set up.

## Writing Tests

Let's go ahead and start writing out tests for our Post model.

```ruby
# spec/models/post_spec.rb

describe Post, type: :model do
  it 'has a valid factory' do
    expect(build(:post)).to be_valid
  end

  describe 'model validations' do
  end

  describe 'model attributes' do
  end

end
```

Now I've set up a basic spec for our Post model. My first test, `has a valid factory`, is just checking to make sure that our Post factory gets built properly. This test is going to fail - we need to build out the specifications for our Post factory first.

```ruby
# spec/factories/posts.rb

FactoryGirl.define do
  factory :post do
    name { "Name of Post" }
    content { "Content of Post" }
  end
end

```

Post has two attributes: `name` and `content`. In the `spec/factories/posts.rb` file, we're defining a new factory. Inside of our `factory(:post)` block, we're setting the attributes equal to their respective values. In this case, our `name` attribute is equal to "Name of Post", and our `content` attribute is equal to "Content of Post". Now if we run the spec again for our Post model, the first test will pass.

Let's add the rest of the tests for our Post model:

```ruby
describe Post, type: :model do
  ...

  describe 'model validations' do
    it 'is valid with a name and content' do
      post = build(:post)
      expect(post).to be_valid
    end

    it 'is invalid without a name' do
      post = build(:post, name: nil)
      expect(post).to_not be_valid
    end

    it 'is invalid without content' do
      post = build(:post, content: nil)
      expect(post).to_not be_valid
    end
  end

  describe 'model attributes' do
    let(:post) { create(:post) }

    it 'has a #name attribute' do
      expect(post.name).to eq("Delivering Goods")
    end

    it 'has a #content attribute' do
      expect(post.content).to eq("In order to deliver goods, you should...")
    end
  end

end
```

Notice how I've defined the `post` fixture throughout the Post model tests. In some of the tests, I've built the fixtures just as they're defined in the factories. In the `model validations` section, I've overridden the default value of the `content` attribute in the factory, and assigned a value of `nil`. 

That's it for the Post model.

Now let's move on to the associations test for the Comment  model. We've made an association in our application, which is that a Post has many comments. How can we create a fixture for that association? Normally, that would involve creating two objects separately, and setting the `post` method of the comment object equal to the newly created post object. In FactoryGirl, you can assemble the associations so that whenever a Comment object is built in a spec, it automatically builds an associated Post object.

Let's look at the specs for the Comment model:

```ruby
describe Comment, type: :model do
  ...

  describe 'model attributes' do
    let(:comment) { create(:comment) }

    it 'has a #title attribute' do
    end

    it 'has a title equivalent to the associated post #name attribute' do
    end

    it 'has a #body attribute' do
    end
  end
end
```

Now let's set up the factory:

```ruby
FactoryGirl.define do
  factory :comment do
    post
    title { post.name }
    body { "The delivery man broke my jar of cookies." }
  end
end
```

Whenever a Comment factory is invoked in a spec, then it's going to automatically create a new Post factory, assuming that a factory for a Post already exists.

Let's build out the rest of the specs:

```ruby
describe Comment, type: :model do
  it 'has a valid factory' do
    expect(build(:comment)).to be_valid
  end

  describe 'model attributes' do
    let(:comment) { create(:comment) }

    it 'has a #title attribute' do
      expect(comment.title).to eq("Delivering Goods")
    end

    it 'has a title equivalent to the associated post #name attribute' do
      expect(comment.title).to eq(comment.post.name)
    end

    it 'has a #body attribute' do
      expect(comment.body).to eq("The delivery man broke my jar of cookies.")
    end
  end
end
```

Pretty nifty resource, eh?

## Resources

* [Github](http://github.com/) - [Getting Started with FactoryGirl](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md)
* [Slideshare](http://www.slideshare.net) - [Not So Brief Intro to FactoryGirl](http://www.slideshare.net/gabevanslv/factory-girl-15924188)
