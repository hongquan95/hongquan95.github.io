---
layout: post
title:  "Laravel Queue"
date:   2019-05-03 22:00:00 +0700
categories: [laravel]
---

[# Introduction](#introduction)

[# Job](#job)

[# Dispatch Job](#dispatch-job)

[# Start Queue](#start-queue)

[# Supervisor](#supervisor)

# # Introduction

> Queue là cấu trúc hàng đợi theo cơ chế FIFO khác với stack(LIFO). Giúp trì hoãn các công việc tốn thời gian đến một thời điểm nào đó mới xử lý giúp cải thiện hiệu suất request tới website. Laravel cung cấp các queue sử dụng ở backend như Beanstalk, Amazon SQS, Redis hoặc qua database.

Cấu hình queue trong Laravel ở file config/queue.php. Ở đây ta sẽ cấu hình các connection cho mỗi queue driver sử dụng cho Laravel.
VD cấu hình queue cho database driver: 
```
'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'expire' => 60,
        ],
```

# Job
Mỗi job là mỗi công việc cần đẩy xuống queue để trì hoãn và thựchiện sau này.

Tạo job `php artisan make:job NameJob`

Ví dụ tạo Job tính tổng PaymentTotal

```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;

class PaymentTotal implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $payment;
    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct(Payment $payment)
    {

        $this->payment = $payment;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        $paymentsAmount = Payment::where('payment_id', '<', $this->payment->payment_id)->sum('amount');
        \Log::info('Payment to of payments_id less than ' . $this->payment->payment_id. ' is '. $paymentsAmount . ' USD');
    }
}
```
Mỗi Class Job đều implement `ShouldQueue`.

Trait `Dispatchable` sử dụng để đẩy job xuống queue.

Trait `InteractsWithQueue, Queueable` chứa các methods làm việc với job và queue.

Trait `SerializesModels` sử dụng cho việc type-hint Eloquent ở `contructor`.

`handle()`là method được xử lí trong hàng đợi, ta có thê type-hint ở đây. Laravel service-container sẽ tự động injects các dependences. Ta có thể quản lí việc inject bằng method `bindMethod` viết trong `service provider`.

# Dispatch Job
Đẩy job xuống queue để thực hiện.

Có thể delay, delete queue...

```php
    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        ...
        $payment = Payment::find($request->payment_id);
        PaymentTotal::dispatch($payment)->onQueue('payment');
        ...
    }
```
Cách gọi khác:
- Sử dụng trait Dispatchable
```php
$job = ((new PaymentTotal($payment))->onQueue('payment'));
$this->dispatch($job);
```
- Sử dụng helper:(Có thể truyền vào 1 Closure)
```php
dispatch((new PaymentTotal($payment))->onQueue('payment'))
```

# Start Queue
`php artisan queue:work --once --queue=payment --delay=0 --memory=128 --sleep=3 --tries=2` (thêm param & để chạy background)
`php artisan queue:listen` 
Mỗi khi thay đổi code ta phải restart lại queue để cập nhật code cho các Job.
Có thể sử dụng `queue:listen` (Giảm hiệu suất)


