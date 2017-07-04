# Flexirest

[![Build Status](https://travis-ci.org/flexirest/flexirest.svg?branch=master)](https://travis-ci.org/flexirest/flexirest)
[![Coverage Status](https://coveralls.io/repos/github/flexirest/flexirest/badge.svg?branch=master)](https://coveralls.io/github/flexirest/flexirest?branch=master)
[![Code Climate](https://codeclimate.com/github/flexirest/flexirest.png)](https://codeclimate.com/github/flexirest/flexirest)
[![Gem Version](https://badge.fury.io/rb/flexirest.png)](http://badge.fury.io/rb/flexirest)
[![Average time to resolve an issue](http://isitmaintained.com/badge/resolution/flexirest/flexirest.svg)](http://isitmaintained.com/project/flexirest/flexirest "Average time to resolve an issue")
[![Percentage of issues still open](http://isitmaintained.com/badge/open/flexirest/flexirest.svg)](http://isitmaintained.com/project/flexirest/flexirest "Percentage of issues still open")

This gem is for accessing REST services in an ActiveRecord style.  ActiveResource already exists for this, but it doesn't work where the resource naming doesn't follow Rails conventions, it doesn't have in-built caching and it's not as flexible in general.

If you are a previous user of ActiveRestClient, there's some more information on [why I created this fork and how to upgrade](https://github.com/flexirest/flexirest/blob/master/Migrating-from-ActiveRestClient.md).

- [Installation](#installation)
- [Basic Usage](#usage)
  - [Create a new person](#create-a-new-person)
  - [Find a person](#find-a-person-not-needed-after-creating)
  - [Update a person](#update-a-person)
  - [Get all people](#get-all-people)
  - [Ruby on Rails Integration](#ruby-on-rails-integration)
- [Advanced Features](#advanced-features)
	- [Faraday Configuration](#faraday-configuration)
	- [Associations](#associations)
		- [Association Type 1 - Loading Other Classes](#association-type-1-loading-other-classes)
		- [Association Type 2 - Lazy Loading From Other URLs](#association-type-2-lazy-loading-from-other-urls)
		- [Association Type 3 - HAL Auto-loaded Resources](#association-type-3-hal-auto-loaded-resources)
		- [Association Type 4 - Nested Resources](#association-type-4-nested-resources)
    - [Association Type 5 - JSON API Auto-loaded Resources](#association-type-5-json-api-auto-loaded-resources)
    - [Combined Example](#combined-example)
	- [Caching](#caching)
	- [Using callbacks](#using-callbacks)
	- [Lazy Loading](#lazy-loading)
	- [Authentication](#authentication)
		- [Basic](#basic)
		- [Api-Auth](#api-auth)
	- [Body Types](#body-types)
	- [Parallel Requests](#parallel-requests)
	- [Faking Calls](#faking-calls)
	- [Per-request Timeouts](#per-request-timeouts)
	- [Per-request Params Encoding](#per-request-params-encoding)
	- [Automatic Conversion of Fields to Date/DateTime](#automatic-conversion-of-fields-to-datedatetime)
	- [Raw Requests](#raw-requests)
	- [Plain Requests](#plain-requests)
  - [JSON API](#json-api)
	- [Proxying APIs](#proxying-apis)
	- [Translating APIs](#translating-apis)
	- [Default Parameters](#default-parameters)
	- [Root element removal](#root-element-removal)
	- [Required Parameters](#required-parameters)
	- [Updating Only Changed/Dirty](#updating-only-changed-dirty)
	- [HTTP/Parse Error Handling](#httpparse-error-handling)
	- [Validation](#validation)
		- [Permitting nil values](#permitting-nil-values)
- [Debugging](#debugging)
- [XML Responses](#xml-responses)
- [Contributing](#contributing)


## Installation

Add this line to your application's Gemfile:

```ruby
gem 'flexirest'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install flexirest

## Usage

First you need to create your new model class:

```ruby
# config/environments/production.rb
Rails.application.configure do
  # ...
  config.api_server_url = "https://www.example.com/api/v1"
end

# app/models/person.rb
class Person < Flexirest::Base
  base_url Rails.application.config.api_server_url

  get :all, "/people"
  get :find, "/people/:id"
  put :save, "/people/:id"
  post :create, "/people"
end
```

Note I've specified the base_url in the class above.  This is useful where you want to be explicit or use different APIs for some classes and be explicit. If you have one server that's generally used, you can set it once with a simple line in a `config/initializer/{something}.rb` file:

```ruby
Flexirest::Base.base_url = "https://www.example.com/api/v1"
```

Any `base_url` settings in specific classes override this declared default. You can also assign an array of URLs to `base_url` and Flexirest will randomly pull one of the URLs for each request, giving you a very simplistic load balancing (it doesn't know about the health or load levels of the backends).

You can then use your new class like this:

```ruby
# Create a new person
@person = Person.create(
  first_name:"John"
  last_name:"Smith"
)

# Find a person (not needed after creating)
id = @person.id
@person = Person.find(id)

# Update a person
@person.last_name = "Jones"
@person.save

# Get all people
@people = Person.all
@people.each do |person|
  puts "Hi " + person.first_name
end
```

If an API returns an array of results and you have [will_paginate](https://rubygems.org/gems/will_paginate) installed then you can call the paginate method to return a particular page of the results (note: this doesn't reduce the load on the server, but it can help with pagination if you have a cached response).

```ruby
@people = Person.all
@people.paginate(page: 1, per_page: 10).each do |person|
  puts "You made the first page: " + person.first_name
end
```

Note, you can assign to any attribute, whether it exists or not before and read from any attribute (which will return nil if not found).  If you pass a string or a number to a method it will assume that it's for the "id" field.  Any other field values must be passed as a hash and you can't mix passing a string/number and a hash.

```ruby
@person = Person.find(1234)  # valid
@person = Person.find("1234")  # valid
@person = Person.find(:id => 1234)  # valid
@person = Person.find(:id => 1234, :name => "Billy")  # valid
@person = Person.find(1234, :name => "Billy")  # invalid
```

You can also call any mapped method as an instance variable which will pass the current attribute set in as parameters (either GET or POST depending on the mapped method type).  If the method returns a single instance it will assign the attributes of the calling object and return itself.  If the method returns a list of instances, it will only return the list. So, we could rewrite the create call above as:

```ruby
@person = Person.new
@person.first_name = "John"
@person.last_name  = "Smith"
@person.create
puts @person.id
```

The response of the #create call set the attributes at that point (any manually set attributes before that point are removed).

If you have attributes beginning with a number, Ruby doesn't like this.  So, you can use hash style notation to read/write the attributes:

```ruby
@tv = Tv.find(model:"UE55U8000") # { "properties" : {"3d" : false} }
puts @tv.properties["3d"]
@tv.properties["3d"] = true
```

If you want to debug the response, using inspect on the response object may well be useful.  However, if you want a simpler output, then you can call `#to_json` on the response object:

```ruby
@person = Person.find(email:"something@example.com")
puts @person.to_json
```

### Ruby on Rails Integration

A detailed guide, how to integrate Flexirest with a RESTful resources can be found in the [Ruby-on-Rails-Integration.md](https://github.com/flexirest/flexirest/blob/master/Ruby-on-Rails-Integration.md).

## Advanced Features

### Faraday Configuration

Flexirest uses Faraday to allow switching HTTP backends, the default is to just use Faraday's default. To change the used backend just set it in the class by setting `adapter` to a Faraday supported adapter symbol.

```ruby
Flexirest::Base.adapter = :net_http
# or ...
Flexirest::Base.adapter = :patron
```

In versions before 1.2.0 the adapter was hardcoded to `:patron`, so if you want to ensure it still uses Patron, you should set this setting.

If you want more control you can pass a **complete** configuration block ("complete" means that the block does not *override* [the default configuration](https://github.com/flexirest/flexirest/blob/5b1953d89e26c02ca74f74464ccb7cd4c9439dcc/lib/flexirest/configuration.rb#L184-L201), but rather *replaces* it). For available config variables look into the Faraday documentation.

```ruby
Flexirest::Base.faraday_config do |faraday|
  faraday.adapter(:net_http)
  faraday.options.timeout       = 10
  faraday.headers['User-Agent'] = "Flexirest/#{Flexirest::VERSION}"
end
```
### Associations

There are two types of association.  One assumes when you call a method you actually want it to call the method on a separate class (as that class has other methods that are useful).  The other is lazy loading related classes from a separate URL.

#### Association Type 1 - Loading Other Classes

If the call would return a single instance or a list of instances that should be considered another object, you can also specify this when mapping the method using the `:has_one` or `:has_many` options respectively.  It doesn't call anything on that object except for instantiate it, but it does let you have objects of a different class to the one you initially called.

```ruby
class Expense < Flexirest::Base
  def inc_vat
    ex_vat * 1.20
  end
end

class Address < Flexirest::Base
  def full_string
    return "#{self.street}, #{self.city}, #{self.region}, #{self.country}"
  end
end

class Person < Flexirest::Base
  get :find, "/people/:id", :has_many => {:expenses => Expense}, :has_one => {:address => Address}
end

@person = Person.find(1)
puts @person.expenses.reduce {|e| e.inc_vat}
puts @person.address.full_string
```

You can also use `has_one`/`has_many` on the class level to allow chaining of classes.  You can specify the class name or allow the system to automatically convert it to the singular class. For example:

```ruby
class Expense < Flexirest::Base
  def inc_vat
    ex_vat * 1.20
  end
end

class Address < Flexirest::Base
  def full_string
    return "#{self.street}, #{self.city}, #{self.region}, #{self.country}"
  end
end

class Person < Flexirest::Base
  has_one :addresses
  has_many :expenses, Expense
  get :find, "/people/:id"
end

class Company < Flexirest::Base
  has_many :people
  get :find, "/companies/:id"
end
```

Sometimes we want attributes to just return a simple Ruby Array. To achieve this we can add an `:array` option to the method. This is especially useful when the attribute contains an array of scalar values. If you don't specify the `:array` option Flexirest will return a `Flexirest::ResultIterator`. To illustrate the difference consider the following example:

```ruby
class Book < Flexirest::Base
  # :authors attribute ["Robert T. Kiyosaki", "Sharon L. Lechter C.P.A"]
  # :genres attribute ["self-help", "finance", "education"]
  get :find, "/books/:name", array: [:authors]
end
```

In the example above, the following results can be observed:

```ruby
@book = Book.find("rich-dad-poor-dad")
puts @book.authors
#=> ["Robert T. Kiyosaki", "Sharon L. Lechter C.P.A"]
puts @book.authors.class
#=> Array
puts @book.genres
#=> #<Flexirest::ResultIterator:0x007ff420fe7a88 @_status=nil, @_headers=nil, @items=["self-help", "finance", "education"]>
puts @books.genres.class
#=> Flexirest::ResultIterator
puts @books.genres.items
#=> ["self-help", "finance", "education"]
```

When the `:array` option includes an attribute, it is assumed the values were returned with the request, and they will not be lazily loaded. It is also assumed the attribute values do not map to a Flexirest resource.

#### Association Type 2 - Lazy Loading From Other URLs

When mapping the method, passing a list of attributes will cause any requests for those attributes to mapped to the URLs given in their responses.  The response for the attribute may be one of the following:

```ruby
"attribute" : "URL"
"attribute" : ["URL", "URL"]
"attribute" : { "url" : "URL"}
"attribute" : { "href" : "URL"}
"attribute" : { "something" : "URL"}
```

The difference between the last 3 examples is that a key of `url` or `href` signifies it's a single object that is lazy loaded from the value specified.  Any other keys assume that it's a nested set of URLs (like in the array situation, but accessible via the keys - e.g. object.attribute.something in the above example).

It is required that the URL is a complete URL including a protocol starting with "http".  To configure this use code like:

```ruby
class Person < Flexirest::Base
  get :find, "/people/:id", :lazy => [:orders, :refunds]
end
```

And use it like this:

```ruby
# Makes a call to /people/1
@person = Person.find(1)

# Makes a call to the first URL found in the "books":[...] array in the article response
# only makes the HTTP request when first used though
@person.books.first.name
```

#### Association Type 3 - HAL Auto-loaded Resources

You don't need to define lazy attributes if they are defined using [HAL](http://stateless.co/hal_specification.html) (with an optional embedded representation).  If your resource has an `_links` item (and optionally an `_embedded` item) then it will automatically treat the linked resources (with the `_embedded` cache) as if they were defined using `:lazy` as per type 2 above.

If you need to, you can access properties of the HAL association.  By default just using the HAL association gets the embedded resource (or requests the remote resource if not available in the `_embedded` list).

```ruby
@person = Person.find(1)
@person.students[0]._hal_attributes("title")
```

#### Association Type 4 - Nested Resources

It's common to have resources that are logically children of other resources. For example, suppose that your API includes these endpoints:

| HTTP Verb | Path                        |                                          |
|-----------|-----------------------------|------------------------------------------|
| POST      | /magazines/:magazine_id/ads | create a new ad belonging to a magazine  |
| GET       | /magazines/:magazine_id/ads | display a list of all ads for a magazine |

In these cases, your child class will contain the following:

```ruby
class Ad < Flexirest::Base
  post :create, "/magazines/:magazine_id/ads"
  get :all, "/magazines/:magazine_id/ads"
end
```

You can then access Ads by specifying their magazine IDs:

```ruby
Ad.all(magazine_id: 1)
Ad.create(magazine_id: 1, title: "My Add Title")
```

#### Association Type 5 - JSON API Auto-loaded Resources

If attributes are defined using [JSON API](http://jsonapi.org), you don't need to define lazy attributes. If your resource has a `links` object with a `related` item, it will automatically treat the linked resources as if they were defined using `:lazy`.

You need to activate JSON API by specifying the `json_api` proxy:

```ruby
class Article < Flextirest::Base
  proxy :json_api
end
```

If you want to embed linked resources directly in the response (i.e. request a JSON API compound document), use the `includes` class method. The linked resource is accessed in the same was as if it was lazily loaded, but without the extra request:

```ruby
# Makes a call to /articles with parameters: include=images
Article.includes(:images).all

# For nested resources, the include parameter becomes: include=images.tags,images.photographer
Article.includes(:images => [:tags, :photographer]).all
```

#### Combined Example

OK, so let's say you have an API for getting articles.  Each article has a property called `title` (which is a string) and a property `images` which includes a list of URIs.  Following this URI would take you to a image API that returns the image's `filename` and `filesize`.  We'll also assume this is a HAL compliant API. We would declare our two models (one for articles and one for images) like the following:

```ruby
class Article < Flexirest::Base
  get :find, '/articles/:id', has_many:{:images => Image} # ,lazy:[:images] isn't needed as we're using HAL
end

class Image < Flexirest::Base
  # You may have mappings here

  def nice_size
    "#{size/1024}KB"
  end
end
```

We assume the /articles/:id call returns something like the following:

```json
{
  "title": "Fly Fishing",
  "author": "J R Hartley",
  "images": [
    "http://api.example.com/images/1",
    "http://api.example.com/images/2"
  ]
}
```

We said above that the /images/:id call would return something like:

```json
{
  "filename": "http://cdn.example.com/images/foo.jpg",
  "filesize": 123456
}
```

When it comes time to use it, you would do something like this:

```ruby
@article = Article.find(1)
@article.images.is_a?(Flexirest::LazyAssociationLoader)
@article.images.size == 2
@article.images.each do |image|
  puts image.inspect
end
```

At this point, only the HTTP call to '/articles/1' has been made.  When you actually start using properties of the images list/image object then it makes a call to the URL given in the images list and you can use the properties as if it was a nested JSON object in the original response instead of just a URL:

```ruby
@image = @article.images.first
puts @image.filename
# => http://cdn.example.com/images/foo.jpg
puts @image.filesize
# => 123456
```

You can also treat `@image` looks like an Image class (and you should 100% treat it as one) it's technically a lazy loading proxy.  So, if you cache the views for your application should only make HTTP API requests when actually necessary.

```ruby
puts @image.nice_size
# => 121KB
```

### Caching

Expires and ETag based caching is enabled by default, but with a simple line in the application.rb/production.rb you can disable it:

```ruby
Flexirest::Base.perform_caching = false
```

or you can disable it per classes with:

```ruby
class Person < Flexirest::Base
  perform_caching false
end
```

If Rails is defined, it will default to using Rails.cache as the cache store, if not, you'll need to configure one with a `ActiveSupport::Cache::Store` compatible object using:

```ruby
Flexirest::Base.cache_store = Redis::Store.new("redis://localhost:6379/0/cache")
```

### Using callbacks

You can use callbacks to alter get/post parameters, the URL or set the post body (doing so overrides normal parameter insertion in to the body) before a request or to adjust the response after a request.  This can either be a block or a named method (like ActionController's `before_callback`/`before_action` methods).

The callback is passed the name of the method (e.g. `:save`) and an object (a request object for `before_request` and a response object for `after_request`). The request object has four public attributes `post_params` (a Hash of the POST parameters), `get_params` (a Hash of the GET parameters), `headers` and `url` (a `String` containing the full URL without GET parameters appended).

```ruby
require 'secure_random'

class Person < Flexirest::Base
  before_request do |name, request|
    if request.post? || name == :save
      id = request.post_params.delete(:id)
      request.get_params[:id] = id
    end
  end

  before_request :replace_token_in_url

  before_request :add_authentication_details

  before_request :replace_body

  before_request :override_default_content_type

  private

  def replace_token_in_url(name, request)
    request.url.gsub!("#token", SecureRandom.hex)
  end

  def add_authentication_details(name, request)
    request.headers["X-Custom-Authentication-Token"] = ENV["AUTH_TOKEN"]
  end

  def replace_body(name, request)
    if name == :create
      request.body = request.post_params.to_json
    end
  end

  def override_default_content_type(name, request)
    if name == :save
      request.headers["Content-Type"] = "application/json"
    end
  end
end
```

If you need to, you can create a custom parent class with a `before_request` callback and all children will inherit this callback.

```ruby
class MyProject::Base < Flexirest::Base
  before_request do |name, request|
    request.get_params[:api_key] = "1234567890-1234567890"
  end
end

class Person < MyProject::Base
  # No need to declare a before_request for :api_key, already defined by the parent
end
```

After callbacks work in exactly the same way:

```ruby
class Person < Flexirest::Base
  after_request :fix_empty_content

  private

  def fix_empty_content(name, response)
    if response.status == 204 && response.body.blank?
      response.body = '{"empty": true}'
    end
  end
end
```

**Note:** since v1.3.21 this isn't necessary, empty responses for 204 are accepted normally (the method returns `true`), but this is hear to show an example of an `after_request` callback.

### Lazy Loading

Flexirest supports lazy loading (delaying the actual API call until the response is actually used, so that views can be cached without still causing API calls).

**Note: Currently this isn't enabled by default, but this is likely to change in the future to make lazy loading the default.**

To enable it, simply call the lazy_load! method in your class definition:

```ruby
class Article < Flexirest::Base
  lazy_load!
end
```

If you have a ResultIterator that has multiple objects, each being lazy loaded or HAL linked resources that isn't loaded until it's used, you can actually parallelise the fetching of the items using code like this:

```ruby
items.parallelise(:id)

# or

items.parallelise do |item|
  item.id
end
```

This will return an array of the named method for each object or the response from the block and will have loaded the objects in to the resource.


### Authentication

#### Basic

You can authenticate with Basic authentication by putting the username and password in to the `base_url` or by setting them within the specific model:

```ruby
class Person < Flexirest::Base
  username 'api'
  password 'eb693ec-8252c-d6301-02fd0-d0fb7-c3485'

  # ...
end
```

#### Api-Auth

Using the [Api-Auth](https://github.com/mgomes/api_auth) integration it is very easy to sign requests. Include the Api-Auth gem in your Gemfile and in then add it to your application. Then simply configure Api-Auth one time in your app and all requests will be signed from then on.

```ruby
require 'api-auth'

@access_id = '123456'
@secret_key = 'abcdef'
Flexirest::Base.api_auth_credentials(@access_id, @secret_key)
```

You can also specify different credentials for different models just like configuring base_url
```ruby
class Person < Flexirest::Base
  api_auth_credentials('123456', 'abcdef')
end
```

For more information on how to generate an access id and secret key please read the [Api-Auth](https://github.com/mgomes/api_auth) documentation.

If you want to specify either the `:digest` or `:override_http_method` to ApiAuth, you can pass these in as options after the access ID and secret key, for example:

```ruby
class Person < Flexirest::Base
  api_auth_credentials('123456', 'abcdef', digest: "sha256")
end
```

### Body Types

By default Flexirest puts the body in to normal CGI parameters in K=V&K2=V2 format.  However, if you want to use JSON for your PUT/POST requests, you can use either (the other option, the default, is `:form_encoded`):

```ruby
class Person < Flexirest::Base
  request_body_type :json
  # ...
end
```

or

```ruby
Flexirest::Base.request_body_type = :json
```

This will also set the header `Content-Type` to `application/x-www-form-urlencoded` by default or `application/json; charset=utf-8` when `:json`. You can override this using the callback `before_request`.

If you have an API that is inconsistent in its body type requirements, you can also specify it on the individual method mapping:

```ruby
class Person < Flexirest::Base
  request_body_type :form_encoded # This is the default, but just for demo purposes

  get :all, '/people', request_body_type: :json
end
```

### Parallel Requests

Sometimes you know you will need to make a bunch of requests and you don't want to wait for one to finish to start the next. When using parallel requests there is the potential to finish many requests all at the same time taking only as long as the single longest request. To use parallel requests you will need to set Flexirest to use a Faraday adapter that supports parallel requests [(such as Typhoeus)](https://github.com/lostisland/faraday/wiki/Parallel-requests).
```ruby
# Set adapter to Typhoeus to use parallel requests
Flexirest::Base.adapter = :typhoeus
```

Now you just need to get ahold of the connection that is going to make the requests by specifying the same host that the models will be using. When inside the `in_parallel` block call request methods as usual and access the results after the `in_parallel` block ends.
```ruby
Flexirest::ConnectionManager.in_parallel('https://www.example.com') do
    @person = Person.find(1234)
    @employers = Employer.all

    puts @person #=> nil
    puts @employers #=> nil
end # The requests are all fired in parallel during this end statement

puts @person.name #=> "John"
puts @employers.size #=> 7
```

### Faking Calls

There are times when an API hasn't been developed yet, so you want to fake the API call response.  To do this, you can simply pass a `fake` option when mapping the call containing the response.

```ruby
class Person < Flexirest::Base
  get :all, '/people', fake: [{first_name:"Johnny"}, {first_name:"Bob"}]
end
```

You may want to run a proc when faking data (to put information from the parameters in to the response or return different responses depending on the parameters).  To do this just pass a proc to :fake:

```ruby
class Person < Flexirest::Base
  get :all, '/people', fake: ->(request) { {result: request.get_params[:id]} }
end
```

### Per-request Timeouts

There are times when an API is generally quick, but one call is very intensive.  You don't want to set a global timeout in the Faraday configuration block, you just want to increase the timeout for this single call. To do this, you can simply pass a `timeout` option when mapping the call containing the response (in seconds).

```ruby
class Person < Flexirest::Base
  get :all, '/people', timeout: 5
end
```

### Per-request Params Encoding

When url-encoding get parameters, Rudy adds brackets([]) by default to any parameters in an Array. For example, if you tried to pass these parameters:

```ruby
Person.all(param: [1, 2, 3])
```

Ruby would encode the url as

```
?param[]=1&param[]=2&param[]=3
```

If you prefer flattened notation instead, pass a `params_encoder` option of `:flat` when mapping the call. So this call:

```ruby
class Person < Flexirest::Base
  get :all, '/people', params_encoder: :flat
end
```

would output the following url

```
?param=1&param=2&param=3
```

### Automatic Conversion of Fields to Date/DateTime

By default Flexirest will attempt to convert all fields to a Date or DateTime object if it's a string and the value matches a pair of regular expressions. However, on large responses this can be computationally expensive.  You can disable this automatic conversion completely with:

```ruby
Flexirest::Base.disable_automatic_date_parsing = true
```

Additionally, you can specify when mapping the methods which fields should be parsed (so you can disable it in general, then apply it to particular known fields):

```ruby
class Person < Flexirest::Base
  get :all, '/people', parse_fields: [:created_at, :updated_at]
end
```

It is also possible to whitelist fields should be parsed in your models, which is useful if you are instantiating these objects directly. The specified fields also apply automatically to request mapping.

```ruby
class Person < Flexirest::Base
  parse_date :updated_at, :created_at
end

# to disable all mapping
class Disabled < Flexirest::Base
  parse_date :none
end
```

This system respects `disable_automatic_date_parsing`, and will default to mapping everything - unless a `parse_date` whitelist is specified, or automatic parsing is globally disabled.

### Raw Requests

Sometimes you have have a URL that you just want to force through, but have the response handled in the same way as normal objects or you want to have the callbacks run (say for authentication).  The easiest way to do that is to call `_request` on the class:

```ruby
class Person < Flexirest::Base
end

people = Person._request('http://api.example.com/v1/people') # Defaults to get with no parameters
# people is a normal Flexirest object, implementing iteration, HAL loading, etc.

Person._request('http://api.example.com/v1/people', :post, {id:1234,name:"John"}) # Post with parameters
```

If you want to use a lazy loaded request instead (so it will create an object that will only call the API if you use it), you can use `_lazy_request` instead of `_request`.  If you want you can create a construct that creates and object that lazy loads itself from a given method (rather than a URL):

```ruby
@person = Person._lazy_request(Person._request_for(:find, 1234))
```

This initially creates an Flexirest::Request object as if you'd called `Person.find(1234)` which is then passed in to the `_lazy_request` method to return an object that will call the request if any properties are actually used.  This may be useful at some point, but it's actually easier to just prefix the `find` method call with `lazy_` like:

```ruby
@person = Person.lazy_find(1234)
```

Doing this will try to find a literally mapped method called "lazy_find" and if it fails, it will try to use "find" but instantiate the object lazily.

### Plain Requests

If you are already using Flexirest but then want to simply call a normal URL and receive the resulting content as a string (i.e. not going through JSON parsing or instantiating in to an Flexirest::Base descendent) you can use code like this:

```ruby
class Person < Flexirest::Base
end

people = Person._plain_request('http://api.example.com/v1/people') # Defaults to get with no parameters
# people is a normal Flexirest object, implementing iteration, HAL loading, etc.

Person._plain_request('http://api.example.com/v1/people', :post, {id:1234,name:"John"}) # Post with parameters
```

The parameters are the same as for `_request`, but it does no parsing on the response

You can also bypass the response parsing using a mapped method like this:

```ruby
class Person < Flexirest::Base
  get :all, "/v1/people", plain: true
end
```

The response of a plain request (from either source) is a `Flexirest::PlainResponse` which acts like a string containing the response's body, but it also has a `_headers` method that returns the HTTP response headers and a `_status` method containing the response's HTTP method.

### JSON API

If you are working with a [JSON API](http://jsonapi.org), you need to activate JSON API by specifying the `json_api` proxy:

```ruby
class Article < Flextirest::Base
  proxy :json_api
end
```

This proxy translates requests according to the JSON API specifications, parses responses, and retrieves linked resources. It also adds the `Accept: application/vnd.api+json` header for all requests.

It supports lazy loading by default. Unless a compound document is returned from the connected JSON API service, it will make another request to the service for the specified linked resource.

To reduce the number of requests to the service, you can ask the service to include the linked resources in the response. Such responses are called "compound documents". To do this, use the `includes` method:

```ruby
# Makes a call to /articles with parameters: include=images
Article.includes(:images).all

# For nested resources, the include parameter becomes: include=images.tags,images.photographer
Article.includes(:images => [:tags, :photographer]).all
```

For post and patch requests, the proxy formats a JSON API complied request, and adds a `Content-Type: application/vnd.api+json` header. It guesses the `type` value in the resource object from the class name, but it can be set specifically with `alias_type`:

```ruby
class Photographer < Flexirest::Base
  proxy :json_api
  # Sets the type in the resource object to "people"
  alias_type :people

  patch :update, '/photographers/:id'
end
```

NB: Updating relationships is not yet supported.

### Proxying APIs

Sometimes you may be working with an old API that returns JSON in a less than ideal format or the URL or parameters required have changed.  In this case you can define a descendent of `Flexirest::ProxyBase`, pass it to your model as the proxy and have it rework URLs/parameters on the way out and the response on the way back in (already converted to a Ruby hash/array). By default any non-proxied URLs are just passed through to the underlying connection layer. For example:

```ruby
class ArticleProxy < Flexirest::ProxyBase
  get "/all" do
    url "/all_people" # Equiv to url.gsub!("/all", "/all_people") if you wanted to keep params
    response = passthrough
    translate(response) do |body|
      body["first_name"] = body.delete("fname")
      body
    end
  end
end

class Article < Flexirest::Base
  proxy ArticleProxy
  base_url "http://www.example.com"

  get :all, "/all", fake:"{\"name\":\"Billy\"}"
  get :list, "/list", fake:"[{\"name\":\"Billy\"}, {\"name\":\"John\"}]"
end

Article.all.first_name == "Billy"
```

This example does two things:

1. It rewrites the incoming URL for any requests matching "*/all*" to "/all_people"
2. It uses the `translate` method to move the "fname" attribute from the response body to be called "first_name".  The translate method must return the new object at the end (either the existing object alterered, or a new object to replace it with)

As the comment shows, you can use `url value` to set the request URL to a particular value, or you can call `gsub!` on the url to replace parts of it using more complicated regular expressions.

You can use the `get_params` or `post_params` methods within your proxy block to amend/create/delete items from those request parameters, like this:

```ruby
get "/list" do
  get_params["id"] = get_params.delete("identifier")
  passthrough
end
```

This example renames the get_parameter for the request from `identifier` to `id` (the same would have worked with post_params if it was a POST/PUT request).  The `passthrough` method will take care of automatically recombining them in to the URL or encoding them in to the body as appropriate.

If you want to manually set the body for the API yourself you can use the `body` method

```ruby
put "/update" do
  body "{\"id\":#{post_params["id"]}}"
  passthrough
end
```

This example takes the `post_params["id"]` and converts the body from being a normal form-encoded body in to being a JSON body.

The proxy block expects one of three things to be the return value of the block.

1. The first options is that the call to `passthrough` is the last thing and it calls down to the connection layer and returns the actual response from the server in to the "API->Object" mapping layer ready for use in your application
2. The second option is to save the response from `passthrough` and use `translate` on it to alter the structure.
3. The third option is to use `render` if you want to completely fake an API and return the JSON yourself

To completely fake the API, you can do the following.  Note, this is also achievable using the `fake` setting when mapping a method, however by doing it in a Proxy block means you can dynamically generate the JSON rather than just a hard coded string.

```ruby
put "/fake" do
  render "{\"id\":1234}"
end
```

### Translating APIs

**IMPORTANT: This functionality has been deprecated in favour of the "Proxying APIs" functionality above.  You should aim to remove this from your code as soon as possible.**

Sometimes you may be working with an API that returns JSON in a less than ideal format.  In this case you can define a barebones class and pass it to your model.  The Translator class must have class methods that are passed the JSON object and should return an object in the correct format.  It doesn't need to have a method unless it's going to translate that mapping though (so in the example below there's no list method). For example:

```ruby
class ArticleTranslator
  def self.all(object)
    ret = {}
    ret["first_name"] = object["name"]
    ret
  end
end

class Article < Flexirest::Base
  translator ArticleTranslator
  base_url "http://www.example.com"

  get :all, "/all", fake:"{\"name\":\"Billy\"}"
  get :list, "/list", fake:"[{\"name\":\"Billy\"}, {\"name\":\"John\"}]"
end

Article.all.first_name == "Billy"
```

### Default Parameters

If you want to specify default parameters you shouldn't use a path like:

```ruby
class Person < Flexirest::Base
  get :all, '/people?all=true' # THIS IS WRONG!!!
end
```

You should use a defaults option to specify the defaults, then they will be correctly overwritten when making the request

```ruby
class Person < Flexirest::Base
  get :all, '/people', :defaults => {:active => true}
end

@people = Person.all(active:false)
```

If you specify `defaults` as a `Proc` this will be executed with the set parameters (which you can change). For example to allow you to specify a reference (but the API wants it formated as "id-reference") you could use:

```ruby
class Person < Flexirest::Base
  get :all, "/", defaults: (Proc.new do |params|
    reference = params.delete(:reference) # Delete the old parameter
    {
      id: "id-#{reference}"
    } # The last thing is the hash of defaults
  end)
end
```

### Root element removal

If your JSON or XML object comes back with a root node and you'd like to ignore it, you can define the mapping as:

```ruby
class Feed < Flexirest::Base
  get :list, "/feed", ignore_root: "feed"
end
```

### Required Parameters

If you want to specify that certain parameters are required for a specific call, you can specify them like:

```ruby
class Person < Flexirest::Base
  get :all, '/people', :requires => [:active]
end

@people = Person.all # raises Flexirest::MissingParametersException
@people = Person.all(active:false)
```

### Updating Only Changed/Dirty

The most common RESTful usage of the PATCH http-method is to only send fields that have changed. The default action for all calls is to send all known object attributes for POST/PUT/PATCH calls, but this can be altered by setting the `only_changed` option on your call declaration.

```ruby
class Person < Flexirest::Base
  get :all, '/people'
  patch :update, '/people/:id', :only_changed => true # only send attributes that are changed/dirty
end

person = Person.all.first
person.first_name = 'Billy'
person.update # performs a PATCH request, sending only the now changed 'first_name' attribute
```

This functionality is per-call, and there is some additional flexibility to control which attributes are sent and when.

```ruby
class Person < Flexirest::Base
  get :all, '/people'
  patch :update_1, '/people/:id', :only_changed => true # only send attributes that are changed/dirty (all known attributes on this object are subject to evaluation)
  patch :update_2, "/people/:id", :only_changed => [:first_name, :last_name, :dob] # only send these listed attributes, and only if they are changed/dirty
  patch :update_3, "/people/:id", :only_changed => {first_name: true, last_name: true, dob: false} # include the listed attributes marked 'true' only when changed; attributes marked 'false' are always included (changed or not, and if not present will be sent as nil); unspecified attributes are never sent
end
```

#### Additional Notes:

* The above examples specifically showed PATCH methods, but this is also available for POST and PUT methods for flexibility purposes (even though they break typical REST methodology).
* This logic is currently evaluated before Required Parameters, so it is possible to ensure that requirements are met by some clever usage.
 - This means that if a method is `:requires => [:active], :only_changed => {active: false}` then `active` will always have a value and would always pass the `:requires` directive (so you need to be very careful because the answer may end up being `nil` if you didn't specifically set it).

### HTTP/Parse Error Handling

Sometimes the backend server may respond with a non-200/304 header, in which case the code will raise an `Flexirest::HTTPClientException` for 4xx errors or an `Flexirest::HTTPServerException` for 5xx errors.  These both have a `status` accessor and a `result` accessor (for getting access to the parsed body):

```ruby
begin
  Person.all
rescue Flexirest::HTTPClientException, Flexirest::HTTPServerException => e
  Rails.logger.error("API returned #{e.status} : #{e.result.message}")
end
```

If the response is unparsable (e.g. not in the desired content type), then it will raise an `Flexirest::ResponseParseException` which has a `status` accessor for the HTTP status code and a `body` accessor for the unparsed response body.

### Validation

You can create validations on your objects just like Rails' built in ActiveModel validations.  For example:

```ruby
class Person < Flexirest::Base
  validates :first_name, presence: true #ensures that the value is present and not blank
  validates :last_name, existence: true #ensures that the value is non-nil only
  validates :password, length: {within:6..12}, message: "Invalid password length, must be 6-12 characters"
  validates :post_code, length: {minimum:6, maximum:8}
  validates :salary, numericality: true, minimum: 20_000, maximum: 50_000
  validates :age, numericality: { minumum: 18, maximum: 65 }
  validates :suffix, inclusion: { in: %w{Dr. Mr. Mrs. Ms.}}

  validates :first_name do |object, name, value|
    object._errors[name] << "must be over 4 chars long" if value.length <= 4
  end

  get :index, '/'
end
```

Note: the block based validation is responsible for adding errors to `object._errors[name]` (and this will automatically be ready for `<<` inserting into).

Validations are run when calling `valid?` or when calling any API on an instance (and then only if it is `valid?` will the API go on to be called).

`full_error_messages` returns an array of attributes with their associated error messages, i.e. `["age must be at least 18"]`. Custom messages can be specified by passing a `:message` option to `validates`. This differs slightly from ActiveRecord in that it's an option to `validates` itself, not a part of a final hash of other options. This is because the author doesn't like the ActiveRecord format (but will accept pull requests that make both syntaxes valid). To make this clearer, an example may help:

```ruby
# ActiveRecord
validates :name, presence: { message: "must be given please" }

# Flexirest
validates :name, :presence, message: "must be given please"
```

#### Permitting nil values
The default behavior for `:length`, `:numericality` and `:inclusion` validators is to fail when a `nil` value is encountered. You can prevent `nil` attribute values from triggering validation errors for attributes that may permit `nil` by adding the `:allow_nil => true` option. Adding this option with a `true` value to `:length`, `:numericality` and `:inclusion` validators will permit `nil` values and not trigger errors. Some examples are:

```ruby
class Person < Flexirest::Base
  validates :first_name, presence: true
  validates :middle_name, length: { minimum: 2, maximum: 30 }, allow_nil: true
  validates :last_name, existence: true
  validates :nick_name, length: { minimum: 2, maximum: 30 }
  validates :alias, length: { minimum: 2, maximum: 30 }, allow_nil: false
  validates :password, length: { within: 6..12 }
  validates :post_code, length: { minimum: 6, maximum: 8 }
  validates :salary, numericality: true, minimum: 20_000, maximum: 50_000
  validates :age, numericality: { minimum: 18, maximum: 65 }
  validates :suffix, inclusion: { in: %w{Dr. Mr. Mrs. Ms.}}
  validates :golf_score, numericality: true, allow_nil: true
  validates :retirement_age, numericality: { minimum: 65 }, allow_nil: true
  validates :cars_owned, numericality: true
  validates :houses_owned, numericality: true, allow_nil: false
  validates :favorite_authors, inclusion: { in: ["George S. Klason", "Robert T. Kiyosaki", "Lee Child"] }, allow_nil: true
  validates :favorite_artists, inclusion: { in: ["Claude Monet", "Vincent Van Gogh", "Andy Warhol"] }
  validates :favorite_composers, inclusion: { in: ["Mozart", "Bach", "Pachelbel", "Beethoven"] }, allow_nil: false
end
```

In the example above, the following results would occur when calling `valid?` on an instance where all attributes have `nil` values:

- `:first_name` must be present
- `:last_name` must be not be nil
- `:nick_name` must be not be nil
- `:alias` must not be nil
- `:password` must not be nil
- `:post_code` must not be nil
- `:salary` must not be nil
- `:age` must not be nil
- `:suffix` must not be nil
- `:cars_owned` must not be nil
- `:houses_owned` must not be nil
- `:favorite_artists` must not be nil
- `:favorite_composers` must not be nil

The following attributes will pass validation since they explicitly `allow_nil`:

- `:middle_name`
- `:golf_score`
- `:retirement_age`
- `:favorite_authors`

### Debugging

You can turn on verbose debugging to see what is sent to the API server and what is returned in one of these two ways:

```ruby
class Article < Flexirest::Base
  verbose true
end

class Person < Flexirest::Base
  verbose!
end
```

By default verbose logging isn't enabled, so it's up to the developer to enable it (and remember to disable it afterwards).  It does use debug level logging, so it shouldn't fill up a correctly configured production server anyway.

If you prefer to record the output of an API call in a more automated fashion you can use a callback called `record_response` like this:

```ruby
class Article < Flexirest::Base
  record_response do |url, response|
    File.open(url.parameterize, "w") do |f|
      f << response.body
    end
  end
end
```

## XML Responses

Flexirest uses Crack to allow parsing of XML responses.  For example, given an XML response of (with a content type of `application/xml` or `text/xml`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">

  <title>Example Feed</title>
  <link href="http://example.org/"/>
  <updated>2003-12-13T18:30:02Z</updated>
  <author>
    <name>John Doe</name>
  </author>
  <id>urn:uuid:60a76c80-d399-11d9-b93C-0003939e0af6</id>

  <entry>
    <title>Atom-Powered Robots Run Amok</title>
    <link href="http://example.org/2003/12/13/atom03"/>
    <id>urn:uuid:1225c695-cfb8-4ebb-aaaa-80da344efa6a</id>
    <updated>2003-12-13T18:30:02Z</updated>
    <summary>Some text.</summary>
  </entry>

  <entry>
    <title>Something else cool happened</title>
    <link href="http://example.org/2015/08/11/andyjeffries"/>
    <id>urn:uuid:1225c695-cfb8-4ebb-aaaa-80da344efa6b</id>
    <updated>2015-08-11T18:30:02Z</updated>
    <summary>Some other text.</summary>
  </entry>

</feed>
```

You can use:

```ruby
class Feed < Flexirest::Base
  base_url "http://www.example.com/v1/"
  get :atom, "/atom"
end

@atom = Feed.atom

puts @atom.feed.title
puts @atom.feed.link.href
@atom.feed.entry.each do |entry|
  puts "#{entry.title} -> #{entry.link.href}"
end
```

For testing purposes, if you are using a `fake` content response when defining your endpoint, you should also provide `fake_content_type: "application/xml"` so that the parser knows to use XML parsing.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
