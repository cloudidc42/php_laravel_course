# Part 039: Laravel Mail & Notifications

**ระดับ:** สูง / มืออาชีพ  
**เวลาเรียน:** 3-4 ชั่วโมง  
**ความต้องการก่อนเรียน:** Part 037 (Queues), Part 038 (Events)

---

## เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Mailable classes สำหรับส่ง email
- ออกแบบ email ด้วย Markdown
- จัดการ Attachments
- Queue email สำหรับส่ง background
- ส่ง Notifications ผ่าน Mail, SMS, Slack
- Workshop: Order confirmation email และ Invoice

---

## 1. ตั้งค่า Mail

### 1.1 .env Configuration

```env
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=your_username
MAIL_PASSWORD=your_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@yourapp.com
MAIL_FROM_NAME="Your App"

# สำหรับ production ใช้ Mailgun
# MAIL_MAILER=mailgun
# MAILGUN_DOMAIN=mg.yourapp.com
# MAILGUN_SECRET=key-xxxxx
```

### 1.2 Mail Drivers

```bash
# Mailtrap (development)
MAIL_MAILER=smtp
MAIL_HOST=sandbox.smtp.mailtrap.io
MAIL_PORT=2525

# Mailgun
MAIL_MAILER=mailgun
composer require symfony/mailgun-mailer symfony/http-client

# Amazon SES
MAIL_MAILER=ses
composer require aws/aws-sdk-php

# Log (เก็บใน storage/logs)
MAIL_MAILER=log
```

---

## 2. สร้าง Mailable Class

```bash
php artisan make:mail OrderConfirmation
```

```php
<?php
// app/Mail/OrderConfirmation.php

namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Address;
use Illuminate\Mail\Mailables\Attachment;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class OrderConfirmation extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    public function __construct(
        public readonly Order $order
    ) {}

    /**
     * Get the message envelope (Subject, From, To, CC, BCC, Reply-to)
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            from: new Address('orders@shop.com', 'Shop Orders'),
            replyTo: [
                new Address('support@shop.com', 'Customer Support'),
            ],
            subject: "Order Confirmation #{$this->order->order_number}",
            tags: ['order-confirmation'],
            metadata: [
                'order_id' => $this->order->id,
            ],
        );
    }

    /**
     * Get the message content definition.
     */
    public function content(): Content
    {
        return new Content(
            markdown: 'emails.orders.confirmation',
            with: [
                'orderUrl' => route('orders.show', $this->order),
                'supportEmail' => 'support@shop.com',
            ],
        );
    }

    /**
     * Get the attachments for the message.
     */
    public function attachments(): array
    {
        return [
            // Attach invoice PDF
            Attachment::fromPath(storage_path("invoices/invoice-{$this->order->id}.pdf"))
                ->as('invoice.pdf')
                ->withMime('application/pdf'),
        ];
    }
}
```

---

## 3. Markdown Email Templates

```bash
php artisan make:mail OrderConfirmation --markdown=emails.orders.confirmation
```

สร้าง view ที่ `resources/views/emails/orders/confirmation.blade.php`:

```blade
{{-- resources/views/emails/orders/confirmation.blade.php --}}

@component('mail::message')
# ขอบคุณสำหรับการสั่งซื้อ! 🎉

สวัสดีคุณ {{ $order->customer->name }},

เราได้รับคำสั่งซื้อของคุณแล้ว และกำลังดำเนินการ

## รายละเอียดคำสั่งซื้อ

**เลขที่ออร์เดอร์:** #{{ $order->order_number }}  
**วันที่สั่งซื้อ:** {{ $order->created_at->format('d/m/Y H:i') }}  
**สถานะ:** {{ $order->status_label }}

@component('mail::table')
| สินค้า | จำนวน | ราคา |
|:-------|------:|-----:|
@foreach ($order->items as $item)
| {{ $item->product->name }} | {{ $item->quantity }} | ฿{{ number_format($item->total, 2) }} |
@endforeach
| **รวมทั้งสิ้น** | | **฿{{ number_format($order->total, 2) }}** |
@endcomponent

## ที่อยู่จัดส่ง

{{ $order->shipping_address->name }}  
{{ $order->shipping_address->address }}  
{{ $order->shipping_address->city }}, {{ $order->shipping_address->province }}  
{{ $order->shipping_address->postal_code }}

@component('mail::button', ['url' => $orderUrl, 'color' => 'primary'])
ดูรายละเอียดออร์เดอร์
@endcomponent

หากมีคำถาม สามารถติดต่อเราได้ที่ {{ $supportEmail }}

ขอบคุณที่ไว้วางใจเรา  
**{{ config('app.name') }}**

@slot('subcopy')
หากปุ่มด้านบนไม่ทำงาน คัดลอก URL นี้:
[{{ $orderUrl }}]({{ $orderUrl }})
@endslot

@endcomponent
```

---

## 4. ส่ง Email

```php
use App\Mail\OrderConfirmation;
use Illuminate\Support\Facades\Mail;

// ส่งทันที
Mail::to($order->customer->email)
    ->send(new OrderConfirmation($order));

// ส่งพร้อม CC และ BCC
Mail::to($order->customer->email)
    ->cc('manager@shop.com')
    ->bcc('audit@shop.com')
    ->send(new OrderConfirmation($order));

// ส่งหลายคน
Mail::to([
    ['address' => 'user1@example.com', 'name' => 'User 1'],
    ['address' => 'user2@example.com', 'name' => 'User 2'],
])->send(new OrderConfirmation($order));

// Queue (ส่ง background)
Mail::to($order->customer->email)
    ->queue(new OrderConfirmation($order));

// Queue พร้อม delay
Mail::to($order->customer->email)
    ->later(now()->addMinutes(10), new OrderConfirmation($order));
```

---

## 5. Invoice Email พร้อม PDF

```php
<?php
// app/Mail/InvoiceMail.php

namespace App\Mail;

use App\Models\Invoice;
use App\Services\PdfService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Attachment;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class InvoiceMail extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    public function __construct(
        public readonly Invoice $invoice
    ) {}

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: "Invoice #{$this->invoice->invoice_number}",
        );
    }

    public function content(): Content
    {
        return new Content(
            markdown: 'emails.invoice',
        );
    }

    public function attachments(): array
    {
        // สร้าง PDF แนบไปกับ email
        $pdfContent = app(PdfService::class)->generateInvoice($this->invoice);
        
        return [
            Attachment::fromData(fn() => $pdfContent, "invoice-{$this->invoice->invoice_number}.pdf")
                ->withMime('application/pdf'),
        ];
    }
}
```

```php
<?php
// app/Services/PdfService.php

namespace App\Services;

use App\Models\Invoice;
use Barryvdh\DomPDF\Facade\Pdf;

class PdfService
{
    public function generateInvoice(Invoice $invoice): string
    {
        $pdf = Pdf::loadView('pdf.invoice', ['invoice' => $invoice]);
        
        // ตั้งค่า PDF
        $pdf->setPaper('a4', 'portrait');
        $pdf->setOptions([
            'dpi' => 150,
            'defaultFont' => 'sans-serif',
        ]);

        return $pdf->output();
    }
    
    public function saveInvoice(Invoice $invoice): string
    {
        $pdf = Pdf::loadView('pdf.invoice', ['invoice' => $invoice]);
        $filename = "invoice-{$invoice->invoice_number}.pdf";
        $path = storage_path("app/invoices/{$filename}");
        
        $pdf->save($path);
        
        return $path;
    }
}
```

---

## 6. Notifications

Notifications เป็น abstraction layer ที่ส่ง notification ผ่านช่องทางหลายๆ อย่าง

```bash
php artisan make:notification InvoicePaid
```

```php
<?php
// app/Notifications/InvoicePaid.php

namespace App\Notifications;

use App\Models\Invoice;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Messages\SlackMessage;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    public function __construct(
        public readonly Invoice $invoice
    ) {}

    /**
     * ระบุช่องทางที่จะส่ง
     */
    public function via(object $notifiable): array
    {
        $channels = ['mail', 'database'];
        
        // ส่ง SMS ถ้า user ตั้งค่าไว้
        if ($notifiable->phone && $notifiable->sms_notifications) {
            $channels[] = 'vonage'; // SMS
        }
        
        // ส่ง Slack ถ้าเป็น admin
        if ($notifiable->hasRole('admin')) {
            $channels[] = 'slack';
        }

        return $channels;
    }

    /**
     * Email notification
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject("Invoice #{$this->invoice->invoice_number} Paid")
            ->greeting("Hello {$notifiable->name}!")
            ->line("Your invoice #{$this->invoice->invoice_number} has been paid.")
            ->line("Amount: ฿" . number_format($this->invoice->amount, 2))
            ->action('View Invoice', route('invoices.show', $this->invoice))
            ->line('Thank you for your business!');
    }

    /**
     * Database notification (เก็บใน notifications table)
     */
    public function toDatabase(object $notifiable): array
    {
        return [
            'type' => 'invoice_paid',
            'invoice_id' => $this->invoice->id,
            'invoice_number' => $this->invoice->invoice_number,
            'amount' => $this->invoice->amount,
            'message' => "Invoice #{$this->invoice->invoice_number} has been paid",
        ];
    }

    /**
     * SMS notification (ต้องติดตั้ง vonage/client)
     */
    public function toVonage(object $notifiable): \Illuminate\Notifications\Messages\VonageMessage
    {
        return (new \Illuminate\Notifications\Messages\VonageMessage)
            ->content("Invoice #{$this->invoice->invoice_number} has been paid. Amount: ฿" . number_format($this->invoice->amount, 2));
    }

    /**
     * Slack notification
     */
    public function toSlack(object $notifiable): SlackMessage
    {
        return (new SlackMessage)
            ->success()
            ->content("Invoice Paid!")
            ->attachment(function ($attachment) {
                $attachment->title("Invoice #{$this->invoice->invoice_number}", route('invoices.show', $this->invoice))
                           ->fields([
                               'Customer' => $this->invoice->customer->name,
                               'Amount' => '฿' . number_format($this->invoice->amount, 2),
                               'Date' => now()->format('d/m/Y'),
                           ]);
            });
    }
}
```

### 6.1 ส่ง Notification

```php
use App\Notifications\InvoicePaid;

// ส่งให้ user
$user->notify(new InvoicePaid($invoice));

// ส่งแบบ on-demand (ไม่ต้องมี user)
\Notification::route('mail', 'admin@shop.com')
    ->route('slack', 'https://hooks.slack.com/services/xxx')
    ->notify(new InvoicePaid($invoice));

// ส่งหลายคนพร้อมกัน
\Notification::send(User::all(), new InvoicePaid($invoice));
```

### 6.2 Database Notifications

```bash
# สร้าง notifications table
php artisan notifications:table
php artisan migrate
```

```php
// Model ต้องใช้ Notifiable trait
class User extends Authenticatable
{
    use Notifiable;
}

// อ่าน notifications
$user->notifications;              // ทั้งหมด
$user->unreadNotifications;        // ที่ยังไม่ได้อ่าน
$user->readNotifications;          // ที่อ่านแล้ว

// Mark as read
$user->unreadNotifications->markAsRead();
$notification->markAsRead();

// ลบ
$user->notifications()->delete();
```

---

## 7. Workshop: Order Confirmation System

```php
<?php
// app/Http/Controllers/OrderController.php

namespace App\Http\Controllers;

use App\Events\OrderPlaced;
use App\Jobs\GenerateInvoicePdf;
use App\Mail\OrderConfirmation;
use App\Models\Order;
use App\Notifications\OrderStatusChanged;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;

class OrderController extends Controller
{
    public function store(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'items' => 'required|array|min:1',
            'items.*.product_id' => 'required|exists:products,id',
            'items.*.quantity' => 'required|integer|min:1',
            'shipping_address' => 'required|array',
        ]);

        $order = DB::transaction(function () use ($request) {
            // สร้าง order
            $order = Order::create([
                'user_id' => auth()->id(),
                'order_number' => Order::generateOrderNumber(),
                'status' => 'pending',
                'shipping_address' => $request->shipping_address,
            ]);

            // เพิ่ม items
            foreach ($request->items as $item) {
                $product = \App\Models\Product::findOrFail($item['product_id']);
                
                $order->items()->create([
                    'product_id' => $product->id,
                    'product_name' => $product->name,
                    'price' => $product->price,
                    'quantity' => $item['quantity'],
                    'total' => $product->price * $item['quantity'],
                ]);
            }

            // คำนวณยอดรวม
            $order->update(['total' => $order->items->sum('total')]);

            return $order;
        });

        // ส่ง confirmation email แบบ queue
        Mail::to(auth()->user()->email)
            ->queue(new OrderConfirmation($order->load(['items.product', 'customer'])));

        // Fire event
        OrderPlaced::dispatch($order);

        // Generate invoice ใน background
        GenerateInvoicePdf::dispatch($order);

        return response()->json([
            'order' => $order,
            'message' => 'Order placed successfully',
        ], 201);
    }

    public function updateStatus(Request $request, Order $order): \Illuminate\Http\JsonResponse
    {
        $request->validate([
            'status' => 'required|in:confirmed,processing,shipped,delivered,cancelled',
        ]);

        $oldStatus = $order->status;
        $order->update(['status' => $request->status]);

        // ส่ง notification
        $order->customer->notify(new OrderStatusChanged($order, $oldStatus));

        return response()->json(['message' => 'Status updated']);
    }
}
```

---

## Quiz

### คำถาม 1
`ShouldQueue` ใน Mailable class ทำอะไร?

**A)** ส่ง email ทันที  
**B)** ส่ง email ผ่าน queue ใน background  
**C)** บันทึก email ลง database  
**D)** ส่ง email ซ้ำอัตโนมัติเมื่อ fail  

**เฉลย: B** - implement `ShouldQueue` ทำให้ email ถูกส่งผ่าน queue ใน background

---

### คำถาม 2
Notification `via()` method ใช้ทำอะไร?

**A)** กำหนด subject ของ notification  
**B)** กำหนดช่องทาง (mail, SMS, slack, database) ที่จะส่ง notification  
**C)** กำหนดผู้รับ  
**D)** กำหนด template  

**เฉลย: B** - `via()` return array ของช่องทางที่จะส่ง เช่น `['mail', 'database', 'slack']`

---

### คำถาม 3
`Attachment::fromData()` ใช้เพื่ออะไร?

**A)** แนบไฟล์จาก disk  
**B)** แนบข้อมูลจาก string/binary โดยตรง (เช่น PDF ที่ generate ใน memory)  
**C)** แนบ URL  
**D)** แนบจาก database  

**เฉลย: B** - `fromData()` รับ closure ที่ return binary content ใช้เมื่อไม่ต้องการบันทึกไฟล์ก่อนแนบ

---

### คำถาม 4
Database notifications เก็บข้อมูลที่ไหน?

**A)** `logs` table  
**B)** `notifications` table  
**C)** `messages` table  
**D)** `events` table  

**เฉลย: B** - เก็บใน `notifications` table ซึ่งสร้างด้วย `php artisan notifications:table`

---

## สรุป

ใน Part นี้เราได้เรียนรู้:
- ✅ สร้าง Mailable classes และ Markdown email templates
- ✅ แนบไฟล์และ PDF กับ email
- ✅ Queue emails สำหรับ background sending
- ✅ Notifications หลายช่องทาง (Mail, SMS, Slack, Database)
- ✅ Workshop: Order Confirmation System ครบวงจร

---

## ลิงก์ไป Part ถัดไป

➡️ **[Part 040: Laravel Storage](part-040-laravel-storage.md)**  
เรียนรู้เรื่อง File Storage, Image Upload & Processing, Custom Filesystems
