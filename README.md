rhoconnect-rb
===

A ruby library for the [Rhoconnect](http://rhomobile.com/products/rhosync) App Integration Server.

Using rhoconnect-rb, your application's model data will transparently synchronize with a mobile application built on the [Rhodes framework](http://rhomobile.com/products/rhodes), or any of the available [Rhoconnect clients](http://rhomobile.com/products/rhosync/).  This client includes built-in support for [ActiveRecord](http://ar.rubyonrails.org/) and [DataMapper](http://datamapper.org/) models.

## Getting started

Load the `rhoconnect-rb` library:

	require 'rhoconnect-rb'

Note, if you are using datamapper, install the `dm-serializer` library and require it in your application.  `rhoconnect-rb` depends on this utility to interact with Rhoconnect applications using JSON.
	
## Usage
Now include Rhoconnect::Resource in a model that you want to synchronize with your mobile application:

	class Product < ActiveRecord::Base
	  include Rhoconnect::Resource
	end
	
Or, if you are using DataMapper:

	class Product
	  include DataMapper::Resource
	  include Rhoconnect::Resource
	end

### Partitioning Datasets
	
Next, your models will need to declare a partition key for `rhoconnect-rb`.  This partition key is used by `rhoconnect-rb` to uniquely identify the model dataset when it is stored in a rhoconnect instance.  It is typically an attribute on the model or related model.  `rhoconnect-rb` supports two types of partitions:

* :app - No unique key will be used, a shared dataset is synchronized for all users.
* lambda { some lambda } - Execute a lambda which returns the unique key string.

For example, the `Product` model above might have a `belongs_to :user` relationship.  This provides us a simple way to organize the `Product` dataset for rhoconnect by reusing this relationship.  The partition identifying a username would be declared as:

	class Product < ActiveRecord::Base
	  include Rhoconnect::Resource
	  
	  belongs_to :user
	
	  def partition 
		lambda { self.user.username }
	  end
	end
	
Now all of the `Product` data synchronized by rhoconnect will organized by `self.user.username`.  Note: You can also used a fixed key if the dataset doesn't require a dynamic value:

	def partition
	  :app
	end
	
For more information about Rhoconnect partitions, please refer to the [Rhoconnect docs](http://docs.rhomobile.com/rhosync/source-adapters#data-partitioning).

### Querying Datasets

`rhoconnect-rb` installs a `/rhoconnect/query` route in your application which the Rhoconnect instance invokes to query the dataset for the dataset you want to synchronize.  This route is mapped to a `rhoconnect_query` method in your model.  This method should return a collection of objects:

	class Product < ActiveRecord::Base
	  include Rhoconnect::Resource
	  
	  belongs_to :user
	
	  def partition 
		lambda { self.user.username }
	  end
	
	  def self.rhoconnect_query(partition)
	    Product.where(:user_id => partition)
	  end
	end

In this example, `self.rhoconnect_query` returns a list of products where the partition string (provided by the rhoconnect instance) matches the `user_id` field in the products table.  

### Configuration and Authentication

Configure Rhoconnect in an initializer like `config/initializers/rhoconnect.rb` (for Rails), or directly in your application (i.e. Sinatra):

	# Setup the Rhoconnect uri and api token.
	# Use rhoconnect:get_token to get the token value.
	
	config.uri   = "http://myrhoconnect.com"
	config.token = "secrettoken"

Example: 

   	Rhoconnect.configure do |config|
      config.uri   = "http://myrhoconnect-server.com"
      config.token = "secrettoken"
	end
	
Example with authentication:

Rhoconnect installs a `/rhoconnect/authenticate` route into your application which will receive credentials from the client.  Add block which handles the credentials:

	Rhoconnect.configure do |config|
      config.uri   = "http://myrhoconnect-server.com"
      config.token = "secrettoken"
	  config.authenticate = lambda { |credentials| 
        User.authenticate(credentials[:login], credentials[:password]) 
	  }
	end

## Meta
Created and maintained by Lucas Campbell-Rossen, Vladimir Tarasov and Lars Burgess.

Released under the [MIT License](http://www.opensource.org/licenses/mit-license.php).