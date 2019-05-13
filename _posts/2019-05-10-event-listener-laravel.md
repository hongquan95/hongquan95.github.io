---
layout: post
title:  "Event Listener Laravel"
date:   2019-05-10 22:00:00 +0700
categories: [laravel]
---

[# Introduction](#introduction)

[# Register Event-Listener](#register-event-listener)

[# Define Event](#define-event)

[# Define Listener](#define-listener)

[# Queued Event Listeners](#queued-event-listener)

[# Event Subscribers](#event-subscribers)

# Introduction
Laravel event cung cấp một observer cho phép lắng nghe và theo dõi nhiều event xảy ra trong ứng dụng.


# Register Event Listener
Event thường được lưu trong `app\Events` còn Lisener lưu trong `app\Listener`.
Để tạo event hoặc listener ta sư dụng các lệnh sau:
```php
   php artisan make:event NewEvent
   php artisan make:listener NewListener
```
Hoặc cấu hình trong `EventServiceProvider` rồi chạy lệnh `php artisan event:generate` để tự động tạo các class.
Thông thường Event hay được đăng kí trong `EventServiceProvider`, tuy nhiên ta cũng có thể đăng kí event sử dụng `Closure` trong method `boot` của `EventServiceProvider`:
```php
/**
 * Register any other events for your application.
 *
 * @return void
 */
public function boot()
{
    parent::boot();

    Event::listen('event.name', function ($foo, $bar) {
        //
    });
}
```
#### Wildcard Event Listeners
Ta sử dụng kí tự đại diện `*` khi muốn đăng kí nhiều `event`
```php
Event::listen('event.*', function ($eventName, array $data) {
    //
});
```

# Define Event
`Example:`
```php
<?php

namespace App\Events;

use App\Payment;
use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class PaymentCreateEvent
{
    use Dispatchable, InteractsWithSockets, SerializesModels;


    public $payment;
    public $type;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(Payment $payment, string $type)
    {
        $this->payment = $payment;
        $this->type = $type;
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('channel-name');
    }
}
```
`Dispatchable` giúp cho event để có thể sử dụng queue
`SerializesModels` Sử dụng để tuần sự hoá các Eloquent mà ta inject vào constructor.

# Define Listener
Khi Event được kích hoạt sẽ có các listener lắng nghe nó và xử lí trong method `handle`handle. Trong method `handle` ta có thể biểu diễn các hành động cần thiết để phản hồi event.
```php
<?php

namespace App\Listeners;

use App\Payment;
use Illuminate\Bus\Queueable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

class PaymentListener implements ShouldQueue
{
    use InteractsWithQueue;
    /**
     * The name of the connection the job should be sent to.
     *
     * @var string|null
     */
    public $connection = 'database';

    /**
     * The name of the queue the job should be sent to.
     *
     * @var string|null
     */
    public $queue = 'listeners';

    /**
     * The name of the queue the job should be sent to.
     *
     * @var string|null
     */
    public $delay = '5';

    /**
     * Create the event listener.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Handle the event.
     *
     * @param  object  $event
     * @return void
     */
    public function handle($event)
    {
        $id = $event->payment->payment_id;
        $totalAmount = Payment::where('payment_id', '<=', $id)->sum('amount');
        \Log::info('==============='. strtoupper($event->type). '==================');
        \Log::info('Payment to of payments_id less than ' . $id. ' is: '. $totalAmount . ' USD');
    }
}
```
Ta thấy class `PaymentListener` có một số attributes đặc biệt dùng để tương tác với queue của Laravel như `$connecttion`, `$queue`, `$delay`
Trong method `handle` ta có thể inject event mà lisener đã đăng kí lắng nghe từ trước đó.

# Queued Event Listener
Ta sử dụng listener như một job và đẩy xuống queue thực hiện để thực hiện các công việc tốn thời gian như gửi email hay gọi các API bên thứ 3
Để sử dụng được queue thì `Listener` phải implemenet `ShouldQueue`
Đến lúc này mỗi khi ta bắn event, thì các listener lắng nghe event đó sẽ được đẩy xuống queue và thực hiệnhiện

# Event Subscribers
## Writing Event Subscribers
Ta có 1 event có thể được nhiều listener lắng nghe, ngược lại một listener có thể lắng nghe nhiều event, ta gọi là event subscribers.
```php
<?php

namespace App\Listeners;

class UserEventSubscriber
{
    /**
     * Handle user login events.
     */
    public function onUserLogin($event) {}

    /**
     * Handle user logout events.
     */
    public function onUserLogout($event) {}

    /**
     * Register the listeners for the subscriber.
     *
     * @param  \Illuminate\Events\Dispatcher  $events
     */
    public function subscribe($events)
    {
        $events->listen(
            'Illuminate\Auth\Events\Login',
            'App\Listeners\UserEventSubscriber@onUserLogin'
        );

        $events->listen(
            'Illuminate\Auth\Events\Logout',
            'App\Listeners\UserEventSubscriber@onUserLogout'
        );
    }
}
```
Ta thấy trong `Example` `UserEventSubscriber` sẽ lắng nghe 2 event đó là event `Login` và event `Logout` ta sẽ định nghĩa các event mà listener cần subscriber và các method tương ứng  mà ta cần handle trong method `subscribe` trong class `UserEventSubscriber`

#### Registering Event Subscribers
Sau khi có 1 class SubscriberSubscriber ta đăng kí nó trong `EventServiceProvider` bằng cách add `UserEventSubscriber` vào mảng `$subscribe`
```php
<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event listener mappings for the application.
     *
     * @var array
     */
    protected $listen = [
        //
    ];

    /**
     * The subscriber classes to register.
     *
     * @var array
     */
    protected $subscribe = [
        'App\Listeners\UserEventSubscriber',
    ];
}
```
Document:
[Laravel Event](https://laravel.com/docs/5.7/events)
