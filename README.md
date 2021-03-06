# CentralLogger

Log to a central MongoDB from Rails apps.

## Usage

1. If using Bundler, add the following to your Gemfile then refresh your dependencies by executing "bundle install":

        gem "central_logger"

1. If you're just using gem:

        gem install central_logger

1. Add the following line to your ApplicationController:

        include CentralLogger::Filter

1. If using Rails 3, SKIP this step.  Otherwise, add the following to config/environment.rb:

       require 'central_logger'
       CentralLogger::Initializer.initialize_deprecated_logger(config)

1. Add mongo settings to database.yml for each environment in which you want to use the Central Logger for logging. The values below are defaults:

        development:
          adapter: mysql
          database: my_app_development
          user: root
          mongo:
            database: my_app               # required
            capsize: <%= 10.megabytes %>   # default: 250MB for production; 100MB otherwise
            host: localhost                # default: localhost
            port: 27017                    # default: 27017

  With that in place, a new MongoDB document (record) will be created for each request and,
  by default will record the following information: Runtime, IP Address, Request Time, Controller,
  Action, Params, Application Name and All messages sent to the logger. The structure of the Mongo document looks like this:

        {
          'action'           : action_name,
          'application_name' : action_name,
          'controller'       : controller_name,
          'ip'               : ip_address,
          'messages'         : {
                                 'info'  : [ ],
                                 'debug' : [ ],
                                 'error' : [ ],
                                 'warn'  : [ ],
                                 'fatal' : [ ]
                               },
          'params'           : { },
          'path'             : path,
          'request_time'     : date_of_request,
          'runtime'          : elapsed_execution_time_in_seconds,
          'url'              : full_url
        }

  Beyond that, if you want to add extra information to the base of the document
  (let's say something like user_guid on every request that it's available),
  you can just call the Rails.logger.add_metadata method on your logger like so
  (for example from a before_filter):

        # make sure we're using the CentralLogger in this environment
        if Rails.logger.respond_to?(:add_metadata)
          Rails.logger.add_metadata(:user_guid =&gt; @user_guid)
        end

## Examples

And now, for a couple quick examples on getting ahold of this log data...
First, here's how to get a handle on the MongoDB from within a Rails console:

    >> db = Rails.logger.mongo_connection
    => #&lt;Mongo::DB:0x102f19ac0 @slave_ok=nil, @name="my_app" ... &gt;

    >> collection = db[Rails.logger.mongo_collection_name]
    => #&lt;Mongo::Collection:0x1031b3ee8 @name="development_log" ... &gt;

Once you've got the collection, you can find all requests for a specific user (with guid):

    >> cursor = collection.find(:user_guid => '12355')
    => #&lt;Mongo::Cursor:0x1031a3e30 ... &gt;
    >> cursor.count
    => 5

Find all requests that took more that one second to complete:

    >> collection.find({:runtime => {'$gt' => 1000}}).count
    => 3

Find all order#show requests with a particular order id (id=order_id):

    >> collection.find({"controller" => "order", "action"=> "show", "params.id" => order_id})

Find all requests with an exception that contains "RoutingError" in the message or stack trace:

    >> collection.find({"messages.error" => /RoutingError/})

Copyright (c) 2009 Phil Burrows, released under the MIT license
