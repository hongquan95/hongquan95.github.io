---
layout: post
title:  "Laravel Broadcasting"
date:   2019-05-15 22:00:00 +0700
categories: [laravel]
---

[# Introduction](#introduction)

[# Concept Overview](#concept-overview)

[# Defining Broadcast Events](#defining-broadcast-events)

[# Authorizing Channels](#authorizing-channels)

# Introduction
Hiện nay trong các ứng dụng hiện đại, Websocket được sử dụng để triển các ứng dụng real-timetime, cập nhật theo thời gian thực. Khi dữ liệu được update trên server thì, một message thường sẽ được gửi qua kết nối Websocket để có thể xử lí ở clientclient
> Laravel sử dụng Broadcast để hỗ trợ chúng ta triển khai các ứng dụng này. Giúp cho chúng ta dễ dàng `"broadcast"` event thông qua kết nối Websocket. Ngoài ra Broadcasting còn cho phép chia sẽ các event giữa server side và client chạy ứng dụng Javascript.
### Configuration
Cấu hình broadcasting trong `app\broadcasting.php`. Laravel hỗ trợ một số broadcast driver sau: [Pusher](https://pusher.com/), [Redis](https://laravel.com/docs/5.7/redis) và `log` driver. Ngoài ra ta có thể chọn driver null để disable broadcasting
#### Broadcast Service Provider
Trước khi broadcasting event, ta phải đăng kí `App\Providers\BroadcastServiceProvider`. Ta chỉ cần uncomment `providers` trong `config/app`. Lúc này ta có thể đăng kí xác thực broadcast và các callbacks.
#### CSRF Token
[Laravel Echo](https://laravel.com/docs/5.7/broadcasting#installing-laravel-echo) cần phải truy cập `CSRF token` nên ta cần phải thêm csrf cho thẻ `meta`:
```html
<meta name="csrf-token" content="{{ csrf_token() }}">
```

### Driver Prerequisites
##### Pusher
Nếu broadcasting event thông qua [Pusher](https://pusher.com/) ta phải cài Pusher bằng `composer`
```php
composer require pusher/pusher-php-server "~3.0"
```
Tiếp theo cấu hình thông tin Pusher trong file `config/broadcasting.php`.
```php
'pusher' => [
            'driver' => 'pusher',
            'key' => env('PUSHER_APP_KEY'),
            'secret' => env('PUSHER_APP_SECRET'),
            'app_id' => env('PUSHER_APP_ID'),
            'options' => [
                'cluster' => env('PUSHER_APP_CLUSTER'),
                'encrypted' => true,
            ],
        ],
```
Nếu sử dụng [Laravel Echo](https://laravel.com/docs/5.7/broadcasting#installing-laravel-echo) ta cần xác định `pusher` như là trình broadcaster khi khởi tạo Echo instance trong `resources/js/bootstrap.js`:
```js
import Echo from "laravel-echo";

window.Pusher = require('pusher-js');

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-pusher-key'
});
```
#### Redis
Nếu sử dụng Redis broadcaster. Ta phải cài đặt thư viện `Predis`
```php
composer required predis/predis
```
Redis broadcaster sẽ phát các messages sử dụng tính năng public/subcribe. Tuy nhiên, ta phải cần phải sử dụng thêm `Websocket server` để có thể nhận message từ Redis và phát các messages đó đến  các `Webscket channels`.
*Cơ chế hoạt động*:
Khi Redis broadcaster public 1 event, nó sẽ được public lên 1 channel, data sẽ được mã hoá bao gồm `event name`, `data payload` và `user` người tạo event socket ID
#### Socket.IO
Nếu ta sử dụng Redis broadcaster với Socket.IO server, ta cần phải include thư viện Socket.IO cho ứng dụng Javascript ở client-side. Ta cài đặt thông qua NPM:
```
npm install --save socket.io-client
```
Tương tự `Pusher` ta require `scoket.io` connecter vào Echo instance trong `resources/js/bootstrap.js`, và định nghĩa thêm tham số `host`.
Cuối cùng, ta khởi chạy server Socket.IO độc lập với Laravel vì Laravel không khởi chạy cùng với Socket.IO. Nếu muốn sử dụng chung server ta sử dụng [tlaverdure/laravel-echo-server](https://github.com/tlaverdure/laravel-echo-server) GitHub repository.
#### Queue Prerequisites
Trước khi broadcasting các events,ta cũng cần cấu hình để chạy `queue listener`. Các event broadcasting sẽ được đẩy vào queued và thực hiện, vậy nên thời gian response của ứng dụng không bị ảnh hưởng nghiêm trọng.

# Concept Overview
Laravel's event broadcasting cho phép broadcast các event thuộc phía server Laravel đến ứng dụng Javascript phía client-side thông qua Websocket. Hiện tại, Laravel vận chuyển bằng `Pusher` hoặc `Redis` driver. Các evenet phía server-side sẽ được xử lí một cách dễ dàng ở client-side thông qua thư viện Javascript [Laravel Echo](https://laravel.com/docs/5.7/broadcasting#installing-laravel-echo) 
Các event được broadcast trên các "channels", các channels có thể public hay private. Nếu là private user cần phải xác thực để có thể subcribe trên channel đó.
#### Using An Example Application
Tưởng tượng chúng ta có 1 website cho phép các users xem trạng thái shipping của mỗi oders. Giải sử ta có event `ShippingStatusUpdated` được kích hoạt mỗi khi status được update bới server.
```php
event(new ShippingStatusUpdated($update));
```
#### The `shouldBroadcast` Interface
Mỗi khi user xem orders, ta không muốn họ phải refresh lại trang để xem status của orders. Thay vào đó ta sẽ broadcast các status đến ứng dụng khi chúng updated. Vậy nên, ta cần chỉ định `ShippingStatusUpdated` với `ShouldBroadcast` interface. Laravel sẽ broadcast event mỗi khi event được kích hoạt:
```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class ShippingStatusUpdated implements ShouldBroadcast
{
    /**
     * Information about the shipping status update.
     *
     * @var string
     */
    public $update;
}
```
Vì implements `ShouldBroadcast` nên ta phải định nghĩa method `broadcastOn`. Method này dùng để phản hồi return về Channel mà event broadcast. Ta muốn chỉ user tạo order mới có khả năng xem trạng thái của order, vậy nên ta sẽ broadcast event trên 1 private channel:
```php
/**
 * Get the channels the event should broadcast on.
 *
 * @return array
 */
public function broadcastOn()
{
    return new PrivateChannel('order.'.$this->update->order_id);
}
```
#### Authorizing Channels
Ta nhớ rằng, user phải xác thực để có thể listen trên một private channel. Ta định nghĩa channel authorization trong `routes/channels.php` EX: ta cần xác thực bất kì user listen trên private `order.1` channel thực sự là người tạo ra order đó:
```php
Broadcast::channel('order.{orderId}', function ($user, $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```
method `channel` có 2 tham số: tên của channel và callback trả về `true` hay `false` trong trường hợp user xác thực để listen trên channel.
Callback sẽ nhận user xác thực như là tham số đầu vào.

#### Listening For Event Broadcasts
Về phía client-side. Ta lắng nghe các event bằng Laravel Echo. Ta sử dụng method `private` để subcribe `private channel`. Sau đó, ta có thể sử dụng method `listen` để lắng nghe `ShippingStatusUpdated` event. Mặc định, tất cả các `event's public` se được include trên broadcast event:
```php
Echo.private(`order.${orderId}`)
    .listen('ShippingStatusUpdated', (e) => {
        console.log(e.update);
    });
```
# Defining Broadcast Events
Ta phải implement `Illuminate\Contracts\Broadcasting\ShouldBroadcast`
và định nghĩa method `broadcastOn`.
`PrivateChannels` và `PresenceChannels` trình bày private channel yêu cầu [channel authorization](https://laravel.com/docs/5.7/broadcasting#authorizing-channels).
```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class ServerCreated implements ShouldBroadcast
{
    use SerializesModels;

    public $user;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(User $user)
    {
        $this->user = $user;
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('user.'.$this->user->id);
    }
}
```
Sau khi định nghĩ xong event, ta chỉ cần [kích hoạt event](https://laravel.com/docs/5.7/events) như thông thương. Một khi event được kích hoạt, một queued job sẽ tự động broadcast event trên broadcast driver.
### Broadcast Name
Mặc đinh, Laravel sẽ broadcast sử dụng tên class event. Tuy nhiên, ta cũng có thể customize lại thông qua method `broadcastAs`:
```php
/**
 * The event's broadcast name.
 *
 * @return string
 */
public function broadcastAs()
{
    return 'server.created';
}
```
Một khi đã customize broadcast name, ta phải đăng ký listener bắt đầu bằng dấu `.`. Nó sẽ giúp Echo không truy xuất sai đường dẫn.
```php
.listen('.server.created', function (e) {
    ....
});
```
### Broadcast Data
Khi một event được broadcast, tất cả các thuộc tính public sẽ được broadcast như là event's payload, ta có thể truy cập nó từ Javascript application. Ex:
```js
{
    "user": {
        "id": 1,
        "name": "Patrick Stewart"
        ...
    }
}
```
Nếu muốn customize payload ta sử dụng method `broadcastWith`:
```php
/**
 * Get the data to broadcast.
 *
 * @return array
 */
public function broadcastWith()
{
    return ['id' => $this->user->id];
}
```
### Broadcast Queue
Mặc định các event sẽ dùng queue default trong `queue.php`. Nếu muốn đổi tên queue ta sử dụng thuộc tính `$broadcastQueue` trong event class:
```php
/**
 * The name of the queue on which to place the event.
 *
 * @var string
 */
public $broadcastQueue = 'your-queue-name';
```
Nếu muốn dùng `sync queue` thì ta thay  interface `ShouldBroadcast` thành `ShouldBroadcastNow`.
# Authorizing Channels
Các private channel yêu cầu phải xác thực user hiện tại để có thể listen 1 private channel. Ta cần định nghĩa route để để có thể xác thực và subcribe các private channels.
### Defining Authorization Routes
Trong `BroadcastServiceProvider` sẽ có method `Broadcast::routes` dùng để đăng kí route `/broadcasting/auth`
#### Customizing The Authorization Endpoint
Ta customize lại route xác thực bằng param `authEndPoint`:
```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-pusher-key',
    authEndpoint: '/custom/endpoint/auth'
});
```
### Defining Authorization Callbacks
Khi có route xác thực broadcast, ta xử lí logic trong `routes/channels.php` bằng cách sử dụng method `Broadcast::channel` để đăng kí channel authorization callbacks:
```php
Broadcast::channel('order.{orderId}', function ($user, $orderId) {
    return $user->id === Order::findOrNew($orderId)->user_id;
});
```
Method `channel` nhận 2 tham số đó là tên của channel và một callback trả về `true` hoặc `false` trong trường hợp user xác thực thành công hay không?
`{orderId}` như là một kí tự đại diện cho tên của channel
#### Authorization Callback Model Binding
Giống như các HTTP route, ta cũng có thể model binding trong một route channel.
```php
use App\Order;

Broadcast::channel('order.{order}', function ($user, Order $order) {
    return $user->id === $order->user_id;
});
```
### Defining Channel Classes
Nếu có nhiều channel thì thay vì dùng Closure ta nên định nghĩa các channel thành các Channel Class
```php
php artisan make:channel OrderChannel
```
Sau khi có channel class ta đăng kí nó trong file `routes/channels.php`:
```php
use App\Broadcasting\OrderChannel;

Broadcast::channel('order.{order}', OrderChannel::class);
```
Trong channel class sẽ có method `join` dùng để xác thực user truy cập vào một channel:
```php
<?php

namespace App\Broadcasting;

use App\User;
use App\Order;

class OrderChannel
{
    /**
     * Create a new channel instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Authenticate the user's access to the channel.
     *
     * @param  \App\User  $user
     * @param  \App\Order  $order
     * @return array|bool
     */
    public function join(User $user, Order $order)
    {
        return $user->id === $order->user_id;
    }
}
```
# Broadcasting Events
```php
event(new ShippingStatusUpdated($updated)
```
Phát event và đẩy xuống để thực hiện trong queue.
### Only To Others
Thay vì sử dụng `event` ta sử dụng:
```php
 broadcast(new ShippingStatusUpdated($update))->toOthers();
```
*Ý nghĩa:*
Tránh việc duplicate code...
#### Configuration
Khi khởi tạo Laravel Echo, một SocketID được gán vào connection. Nếu sử dụng [Vue](https://vuejs.org/) và [Axios](https://github.com/mzabriskie/axios), socketID sẽ được tự động đính kèm vào header. Sau đó khi ta gọi `toOthers`, Laravel sẽ lấy socketID và nói cho broadcaster không broadcast tới các connection có socketID vừa được nhận.
Nếu không dùng Vue và Axios, ta phải cấu hình trong Javascript
```js
var socketID = Echo.socketID();
```
# Receiving Broadcasts
### Installing Laravel Echo
Laravel Echo là thư viện Javascript thực hiện công việc subcribe channels và listen các events được broadcast bởi Laravel. Ta có thể cài đặt nó thông qua NPM, ta cũng phải cài thêm `pusher-js` nếu sử dụng `Pusher broadcaster`:
```js
npm install --save laravel-echo pusher-js
```
Sau khi cài xong Echo, ta có thể tạo mới 1 instance Echo trong JS. Ta có sẵn example trong `resources/js/bootstrap.js`:
**Lưu ý:** Chạy `npm run watch-poll` để compile lại file js. Nên viết code trong file `resources/js/app.js` để dễ chỉnh sửa.
```js

import Echo from "laravel-echo"

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-pusher-key'
});
```
#### Using An Existing Client Instance
Nếu đã có sẵn `Pusher` hoặc `Socket.io` instance, ta truyền nó vào các option:
```js
const client = require('pusher-js');

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-pusher-key',
    client: client
});
```
### Listening For Events
Sau khi khởi tạo xong Echo, ta có thể bắt đầu việc listen event broadcasts. Đầu tiên gọi hàm `channel` để nhận instance `channel`, sau đó gọi method `listen` để lắng nghe một event cụ thể:
```js
Echo.channel('orders')
    .listen('OrderShipped', (e) => {
        console.log(e.order.name);
    });
```
Nếu listen private channel ta dùng cú pháp:
```js
Echo.private('orders')
    .listen(...)
    .listen(...)
    .listen(...);
```
### Leaving A Channel
```js
Echo.leaveChannel('orders');
// Echo.leave('orders');
```
### Namespaces
Namespace mặc định `App\Events`
Nếu muốn gọi namespace khác ta khai báo:
```js
window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-pusher-key',
    namespace: 'App.Other.Namespace'
});
```
hoặc  với prefix `.`
```js
Echo.channel('orders')
    .listen('.Namespace\\Event\\Class', (e) => {
        //
    });
```
# Presence Channels
Presence channels xây dựng trên private channels, để sử dụng các tính năng của nhưng user subcribed channel. Ví dụ như notify tới các users khi một user nào đó đang xem trên cùng một page.
### Authorizing Presence Channels
Presence channels cũng tượng tự như private channels, ta cũng phải xác thực cho nó. Khác biệt ở đây là trong callback của presence channels, ta sẽ không return `true` nếu user đã xác thực join vào channel, mà thay vào đó, ta return mảng `user`
Mảng nảy sẽ được cung cấp tới listener ở ứng dụng Javascript. Nếu 1 user xác thực không thành công join vào 1 presence channel, ta return `null` hoặc `false`:
```php
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```
### Join Presence Channels
Để join vào 1 presence channel, ta dùng method `join` của Echo. Nó sẽ trả về `PresenceChannel` để có thể dùng một số method sau:
```js
Echo.join(`chat.${roomId}`)
    .here((users) => {
        //
    })
    .joining((user) => {
        console.log(user.name);
    })
    .leaving((user) => {
        console.log(user.name);
    });
```
`here` callback: thực hiện ngay sau khi join channel thành công, và sẽ nhận về mảng users subcribed channel.
`joinning`: sẽ thực thi khi một user join vào channel.
`leaving`: thực thi khi một user rời channel.
### Broadcasting To Presence Channels
Presence channels nhận một event giống như private hay public channels. Ví dụ như một chatroom, chúng ta muốn broadcast `NewMessage` events tới room presence channel. Để làm điều này. Ta return `PresenceChannel` trong method `broadcastOn`:
```php
**
 * Get the channels the event should broadcast on.
 *
 * @return Channel|array
 */
public function broadcastOn()
{
    return new PresenceChannel('room.'.$this->message->room_id);
}
```
broadcast event, dùng `toOthers` để gửi broadcast tới các users loiaj trừ user hiện tại:
```php
broadcast(new NewMessage($message));

broadcast(new NewMessage($message))->toOthers();
```
Lúc này ta listen để join event ở Echo:
```js
Echo.join(`chat.${roomId}`)
    .here(...)
    .joining(...)
    .leaving(...)
    .listen('NewMessage', (e) => {
        //
    });
```
# Client Events
>Khi dùng `Pusher` ta phải enable `Client Events` option trong `App Settings` trong [application dashboard](https://dashboard.pusher.com/) để có thể gửi events tới client.
Đôi khi, ta mong muốn broadcasst event tới các client đã connected mà không tác động tới Laravel. Ví dụ như thông báo `is typing`, nới mà ta muốn thông báo với các users là có một user đang `type messages` trên màn hình. Ta sử dụng method `whisper` của Echo
```js
Echo.private('chat')
    .whisper('typing', {
        name: this.user.name
    });
```
Để listen client events, ta có thể dùng method `listenForWhisper`:
```js
Echo.private('chat')
    .listenForWhisper('typing', (e) => {
        console.log(e.name);
    });
```
# Notifications
Nếu sử dụng event broadcasting với [notifications](https://laravel.com/docs/5.7/notifications), ứng dụng Javscript phải nhận được notifications khi chúng diễn ra mà không phải refresh lại page. Trước tiên ta nghiên cứu [broadcast notification channel](https://laravel.com/docs/5.7/notifications#broadcast-notifications)
Một khi đã cấu hình notification sử dụng `broadcast channel`, ta có thể listen broadcast events sử dụng hàm `notification` của Echo. Lưu ý là tên của channel phải đúng với tên class nhận notifications:
```js
Echo.private(`App.User.${userId}`)
    .notification((notification) => {
        console.log(notification.type);
    });
```
Trong ví dụ trên, tất các notifications được gửi tới `App\User` thông của `broadcast channel` sẽ được nhận về thông qua callback. Một channel authorization callback dùng cho `App.User.{id}` channel được included trong `BroadcastServiceProvider` của Laravel



