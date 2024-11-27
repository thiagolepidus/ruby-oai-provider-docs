# OAI::Provider

Open Archives Initiative - Protocol for Metadata Harvesting see
<http://www.openarchives.org/>

## Features
* Easily setup a simple repository
* Simple integration with ActiveRecord
* Dublin Core metadata format included
* Easily add addition metadata formats
* Adaptable to any data source
* Simple resumption token support

## Usage

To create a functional provider either subclass {OAI::Provider::Base},
or reconfigure the defaults.

### Sub classing a provider

```ruby
 class MyProvider < Oai::Provider
   repository_name 'My little OAI provider'
   repository_url  'http://localhost/provider'
   record_prefix 'oai:localhost'
   admin_email 'root@localhost'   # String or Array
   source_model MyModel.new       # Subclass of OAI::Provider::Model
 end
```

### Configuring the default provider

```ruby
 class Oai::Provider::Base
   repository_name 'My little OAI Provider'
   repository_url 'http://localhost/provider'
   record_prefix 'oai:localhost'
   admin_email 'root@localhost'
   # record_prefix will be automatically prepended to sample_id, so in this
   # case it becomes: oai:localhost:13900
   sample_id '13900'
   source_model MyModel.new
 end
```

The provider does allow a URL to be passed in at request processing time
in case the repository URL cannot be determined ahead of time.

## Integrating with frameworks

### Camping

In the Models module of your camping application post model definition:

```ruby
  class CampingProvider < OAI::Provider::Base
    repository_name 'Camping Test OAI Repository'
    source_model ActiveRecordWrapper.new(YOUR_ACTIVE_RECORD_MODEL)
  end
```

In the Controllers module:

```ruby
  class Oai
    def get
      @headers['Content-Type'] = 'text/xml'
      provider = Models::CampingProvider.new
      provider.process_request(@input.merge(:url => "http:"+URL(Oai).to_s))
    end
  end
```

The provider will be available at "/oai"

### Rails

At the bottom of environment.rb create a OAI Provider:

```ruby
  # forgive the standard blog example.

  require 'oai'
  class BlogProvider < OAI::Provider::Base
    repository_name 'My little OAI Provider'
    repository_url 'http://localhost:3000/provider'
    record_prefix 'oai:blog'
    admin_email 'root@localhost'
    source_model OAI::Provider::ActiveRecordWrapper.new(Post)
    sample_id '13900' # record prefix used, so becomes oai:blog:13900
  end
```

Create a custom controller:

```ruby
  class OaiController < ApplicationController
    def index
      provider = BlogProvider.new
      response =  provider.process_request(oai_params.to_h)
      render :body => response, :content_type => 'text/xml'
    end

    private

    def oai_params
      params.permit(:verb, :identifier, :metadataPrefix, :set, :from, :until, :resumptionToken)
    end
  end
```

And route to it in your `config/routes.rb` file:

```ruby
   match 'oai', to: "oai#index", via: [:get, :post]
```

Special thanks to Jose Hales-Garcia for this solution.

### Leverage the Provider instance

The traditional implementation of the OAI::Provider would pass the OAI::Provider
class to the different resposnes. This made it hard to inject context into a
common provider. Consider that we might have different request headers that
change the scope of the OAI::Provider queries.

```ruby
class InstanceProvider
  def initialize(options = {})
    super({ :provider_context => :instance_based })
    @controller = options.fetch(:controller)
  end
  attr_reader :controller
end

class OaiController < ApplicationController
  def index
    provider = InstanceProvider.new({ :controller => self })
    request_body =  provider.process_request(oai_params.to_h)
    render :body => request_body, :content_type => 'text/xml'
  end
```

In the above example, the underlying response object will now receive an
instance of the InstanceProvider. Without the `super({ :provider_context => :instance_based })`
the response objects would have received the class InstanceProvider as the
given provider.

## Supporting custom metadata formats

See {OAI::MetadataFormat} for details.

## ActiveRecord Integration

ActiveRecord integration is provided by the `ActiveRecordWrapper` class.
It takes one required paramater, the class name of the AR class to wrap,
and optional hash of options.

As of `oai` gem 1.0.0, Rails 5.2.x and Rails 6.0.x are supported.
Please check the .travis.yml file at root of repo to see what versions of ruby/rails
are being tested, in case this is out of date.

Valid options include:

* `timestamp_field` - Specifies the model field/method to use as the update
  filter.  Defaults to `updated_at`.
* `identifier_field` -- specifies the model field/method to use to get value to use
   as oai identifier (method return value should not include prefix)
* `limit` -           Maximum number of records to return in each page/set.
  Defaults to 100, set to `nil` for all records in one page. Otherwise
  the wrapper will paginate the result via resumption tokens.
  _Caution:  specifying too large a limit will adversely affect performance._

Mapping from a ActiveRecord object to a specific metadata format follows
this set of rules:

1. Does `Model#to_{metadata_prefix}` exist?  If so just return the result.
2. Does the model provide a map via `Model.map_{metadata_prefix}`?  If so
   use the map to generate the xml document.
3. Loop thru the fields of the metadata format and check to see if the
   model responds to either the plural, or singular of the field.

For maximum control of the xml metadata generated, it's usually best to
provide a `to_{metadata_prefix}` in the model.  If using Builder be sure
not to include any `instruct!` in the xml object.

### Explicit creation example

```ruby
 class Post < ActiveRecord::Base
   def to_oai_dc
     xml = Builder::XmlMarkup.new
     xml.tag!("oai_dc:dc",
       'xmlns:oai_dc' => "http://www.openarchives.org/OAI/2.0/oai_dc/",
       'xmlns:dc' => "http://purl.org/dc/elements/1.1/",
       'xmlns:xsi' => "http://www.w3.org/2001/XMLSchema-instance",
       'xsi:schemaLocation' =>
         %{http://www.openarchives.org/OAI/2.0/oai_dc/
           http://www.openarchives.org/OAI/2.0/oai_dc.xsd}) do
         xml.tag!('oai_dc:title', title)
         xml.tag!('oai_dc:description', text)
         xml.tag!('oai_dc:creator', user)
         tags.each do |tag|
           xml.tag!('oai_dc:subject', tag)
         end
     end
     xml.target!
   end
 end
```

### Mapping Example

```ruby
 # Extremely contrived mapping
 class Post < ActiveRecord::Base
   def self.map_oai_dc
     {:subject => :tags,
      :description => :text,
      :creator => :user,
      :contibutor => :comments}
   end
 end
```
### Scopes for restrictions or eager-loading

Instead of passing in a Model class to OAI::Provider::ActiveRecordWrapper, you can actually
pass in any scope (or ActiveRecord::Relation). This means you can use it for restrictions:

    OAI::Provider::ActiveRecordWrapper.new(Post.where(published: true))

Or eager-loading an association you will need to create serialization, to avoid n+1 query
performance problems:

    OAI::Provider::ActiveRecordWrapper.new(Post.includes(:categories))

Or both of those in combination, or anything else that returns an ActiveRecord::Relation,
including using custom scopes, etc.

### Sets?

There is some code written to support oai-pmh "sets" in the ActiveRecord::Wrapper, but
it's somewhat inflexible, and not well-documented, and as I write this I don't understand
it enough to say more. See https://github.com/code4lib/ruby-oai/issues/67
