pubnubdemos-weather
===================
Getting started

Create new rails app:
rails new app_name

For working with pubnub add pubnub gem in Gemfile
gem 'pubnub'

for easy work with pubnub SDK create singletone class:

require 'pubnub'

class PubnubListener

  CONFIG = ::Psych.load_file "#{::Rails.root}/config/pubnub.yml"

  @pubnub = Pubnub.new(
    :subscribe_key => CONFIG['weather_key']
  )

  WEATHER_CHANNEL = 'pubnub-weather'

  class << self
    attr_reader :subscribed

    def subscribe channel = WEATHER_CHANNEL, &block
      @pubnub.subscribe :channel  => channel, &block
    end

    def unsubscribe channel = WEATHER_CHANNEL, &block
      @pubnub.unsubscribe :channel  => channel, &block
    end

  end

end



also create config file for pubnub 
./config/pubnub.yml
with next content:
weather_key: sub-c-b1cadece-f0fa-11e3-928e-02ee2ddab7fe

the next step will create Pubnub initializer
./config/initializers/pubnub.rb

PubnubListener.subscribe do |envelope|
  if envelope.message
    Notifier.notify PubnubListener::WEATHER_CHANNEL, envelope.message
  end
end


It will be run when app starts

The next step is configure push_server for this application.
I use prepared push_server powered by nodejs. 
Next what we need add push_server config and write javascript connecting script.
./config/push_server.yml
host: localhost
port: 8082

use this for testing localy.

Install and run push server
All of you need is download server, install dependencies and run server.
git clone git@github.com:Sungee-Technology/push-server.git
cd push-server
npm install
node server

Now this server listen port 8082(by default) and if we need push message to clients by channel we send http post request on this server:port and all users who subscribed on selected channel channel will receive this message.

For push server we need notifer class. 
./lib/notifier.rb
require 'httparty'

module Notifier

  class << self

    def notify channel, message
      request Hash[channel: channel, message: message].to_json
    end

    private

    CONFIG = ::Psych.load_file "#{::Rails.root}/config/push_server.yml"

    HOST_WITH_PORT = "http://#{CONFIG['host']}:#{CONFIG['port']}"

    REQUEST_TIMEOUT = 1

    def request_url
      "#{HOST_WITH_PORT}"
    end

    def request message
      ::HTTParty.post request_url, body: message, timeout: REQUEST_TIMEOUT
    rescue ::Exception => error
      logger = ::Rails.logger
      logger.error "Tried to send message to push server but: #{error.inspect}"
    end

  end

end

And add javascript connector
//= require EventSource

var setupPushServerConnection = function (namespace, onmessage, callback) {
  var connection

  if ('WebSocket' in window) {
    connection = new WebSocket('ws://<%= ::Psych.load_file("#{::Rails.root}/config/push_server.yml")['host'] %>:8081/')

    connection.addEventListener('open', function () {
      var data = {channel: namespace}

      connection.send(JSON.stringify(data))
    })

    connection.addEventListener('close', function () {
      setTimeout(function () {
        setupPushServerConnection(namespace, onmessage, callback)
      }, 1000)
    })

  } else
    connection = new EventSource(location.pathname + '/broadcast')

  connection.addEventListener('message', function (event) {
    var json = $.parseJSON(event.data)
    if (json)
      onmessage(json)
  })

  if (typeof callback === 'function')
    callback(connection)

}

Usage example:
 
Traffic.prototype.connectToServer = function() {
    var self = this
    setupPushServerConnection('pubnub-weather', function(json){
       //TODO: something with data
    })
}

Thats all next what we need is just run rails application and check results. 
