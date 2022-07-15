---
title: "Laravel Mocking Client"
date: 2022-07-15T23:00:09+02:00
draft: false
---
{{< arabic >}}
هنتكلم المقال عن الـ Mocking وازاي ممكن نستخدمه علشان منحتاجش نتكلم مع الـ  3rd party APIs في الـ development او الـ testing.

ليه محتاجين نعمل Mock ومنستخدمش الـ API ونبعت ريكوستات؟ فيه هنا اكتر من نقطه تمنعنا اننا نعمل كدا
اولاً مفيش اي سبب يخلينا نكلم الـ API واحنا بنعمل تست ف البروجكت او شغالين فـ الـ development
ان فيه APIs مش بتوفر اي testing sandbox علشان تجرب فيها.
ممكن بسهوله نوصل للحد الاقصي من الريكوستات اللي بنبعتها (hitting the ratelimit).
هنبقي بنبعت بينات وهميه و غير صحيحه بدون داعي.


لو اخدنا مثال علي كدا، تعالي نفترض ان عندي api بستخدمها علشان اعمل new customer. ف الـ endpoint اللي بنكلمها هي POST /customer/create ف الكود عندنا هيكون متقسم علي جزئين
.
-  Service Provider علشان نبعت الconfig params اللي ال Client بيحتاجها
{{< /arabic >}}

```php
<?php

class CustomerServiceProvider extends ServiceProvider
{
  public function register(){
            $this->app->bind(CustomrtClientInterface::class, function () {
            if (config('services.google.key')) {
                return new CustomerClient([
                        'base_uri' => 'https://www.googleapis.com/',
                        'headers' => [
                            'content-type' => 'application/json',
                            'Authorization' => 'Bearer ' . config('services.google.key'),
                        ]
                    ]
                );
            }
  }
}
```
{{< arabic >}}

 و الكود ف الكنترولر اللي هيستخدم الـclient ده علشان يبعت الـ new customer request.
{{< /arabic >}}

```php
<?php

class CustomerClient extends Client{
  public function createCustomer(Customer $customer){
    $this->post('customers/create',[
      "json" => [
        'id' => $customer->id,
        'name' => $customer->name,
      ]
    ]);
  }
}
```
{{< arabic >}}

الكود حالياً هيكلم الـ API سواء في الـ dev او الـ prod او الـ testing. 


الحلول المقترحة

لو فكرنا ف الحلول اللي ممكن ننفذها علشان منكلمش الـ API في الـ development ف عندنا

الحل الاول:
انك تعتمد علي env variable مثلا اسمه API_ENV ولو هو dev ف الكود فالـ client ميبعتش اي ريكوستات. وبالتالي التغيير في الكود هيبقي 


المشكله هنا ان كل method عندنا هتحتاج تستخدم ال condition ده. ودي مش افضل طريقه.

الحل التاني اننا نعمل Fake Client و نستخدمه فقط في الـ development والتغيير اللي هتحتاجه هيكون موجود فقط في الـ Service Provider 
{{< /arabic >}}

```php
<?php

use Illuminate\Support\ServiceProvider;

class CustomerServiceProvider extends ServiceProvider
{
    public function register()
    {

        $this->app->bind(CustomrtClientInterface::class, function () {
           if (App::environment(['local', 'staging'])){
                return new FakeCustomerClient();
            }
            return new CustomerClient([
                     'base_uri' => 'https://www.googleapis.com/',
                     'headers' => [
                         'content-type' => 'application/json',
                         'Authorization' => 'Bearer ' . config('services.google.key'),
        }
    }
}
```

Create customer request at CusmtomrClient 

```php 
<?php

class CustomerClient extends CustomrtClientInterface{
  
  public function createCustomer(Customer $customer){
    $this->post('customers/create',[
      "json" => [
        'id' => $customer->id,
        'name' => $customer->name,
      ]
    ]);
  }
}
```

{{< arabic >}}

بالشكل ده، في الـ development الـ Service Provider هيستخدم الـ Fake class وبالتالي مش هنكلم الـ API
{{< /arabic >}}

```php
<?php

class FakeCustomerClient extends CustomrtClientInterface{
  
  public function createCustomer(Customer $customer){
		return true;
  }
}
```
{{< arabic >}}

وبكدا نكون عرفنا ازاي نستخدم ال mocking. فالكود يكون أوضح واسهل ومباشر عشان نقدر نتفادي اننا نبعت داتا وهمية من اللوكل او التيست ل اي API احنا بنتعامل معاها . 
الmocking  بيتم استخدامها كمان ف كتابه ال testing بنعمل اوبجكت وهمي بيحكي انه كمل ال APIs 
عشان نقدر نختبر الكود من غير ما تكلم ال API
{{< /arabic >}}


