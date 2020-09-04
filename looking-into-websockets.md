---
layout: default
---

## WebSockets with Rails

### What are WebSockets?

WebSocket is a protocol that enables bidirectional communication between the client and the server of a web application over a single long living TCP connection.

>The WebSocket protocol enables interaction between a web browser (or other client application) and a web server with lower overheads, facilitating real-time data transfer from and to the server.

>This is made possible by providing a standardized way for the server to send content to the client without being first requested by the client, and allowing messages to be passed back and forth while keeping the connection open. In this way, a two-way ongoing conversation can take place between the client and the server.

>The communications are done over TCP port number 80 (or 443 in the case of TLS-encrypted connections), which is of benefit for those environments which block non-web Internet connections using a firewall. Similar two-way browser-server communications have been achieved in non-standardized ways using stopgap technologies such as Comet.

Setting up the configuration for ActionCable with Redis, inside of config/cable.yml

```yaml
development:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: rails-chat-tutorial_development

test:
  adapter: async

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: rails-chat-tutorial_production
```

Set cookies when establishing a WebSocket connection, as we don't have access to the user session.

```ruby
Warden::Manager.after_set_user do |user,auth,opts|
  scope = opts[:scope]
  auth.cookies.signed["#{scope}.id"] = user.id
end

Warden::Manager.before_logout do |user, auth, opts|
  scope = opts[:scope]
  auth.cookies.signed["#{scope}.id"] = nil
end
```

Set identifier in main ```ApplicationCable::Connection``` class so that we may find the specific connection later on.

```ruby

module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      if verified_user = User.find_by(id: cookies.signed['user.id'])
        verified_user
      else
        reject_unauthorized_connection
      end
    end
  end
end
```

Inside of the find_verified_user method we access the cookie that we previously set in the warden hook.

The initial setup of the `RoomChannel` class.

> A channel encapsulates a logical unit of work, similar to what a controller does in a regular MVC setup.

```ruby
class RoomChannel < ApplicationCable::Channel
  def subscribed
    room = Room.find params[:room]
    stream_for room
  end
end
```

The subscribed method gets called once a subscription to the channel is established.

Everytime a room message is being created, we need to broadcast to the message's room stream.

`app/controllers/room_messages_controller.rb`

```ruby
def create
  @room_message = RoomMessage.create user: current_user,
                                     room: @room,
                                     message: params.dig(:room_message, :message)

  RoomChannel.broadcast_to @room, @room_message
end
```

We need to add some data to the room page in order to use them via JavaScript to subscribe to the appropriate stream.

`app/views/rooms/show.html.erb`

```html
<div class="chat" data-channel-subscribe="room" data-room-id="<%= @room.id %>">
```

And finally create a `room_channel` handler to subscribe and handle incoming channel data.

```js
$(function() {
  $('[data-channel-subscribe="room"]').each(function(index, element) {
    var $element = $(element),
      room_id = $element.data('room-id')
      messageTemplate = $('[data-role="message-template"]');

    $element.animate({ scrollTop: $element.prop("scrollHeight")}, 1000)

    App.cable.subscriptions.create(
      {
        channel: "RoomChannel",
        room: room_id
      },
      {
        received: function(data) {
          var content = messageTemplate.children().clone(true, true);
          content.find('[data-role="user-avatar"]').attr('src', data.user_avatar_url);
          content.find('[data-role="message-text"]').text(data.message);
          content.find('[data-role="message-date"]').text(data.updated_at);
          $element.append(content);
          $element.animate({ scrollTop: $element.prop("scrollHeight")}, 1000);
        }
      }
    );
  });
});

```

[back](./)
