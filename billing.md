# 라라벨 캐셔

- [소개](#introduction)
- [Stripe 설정하기](#stripe-configuration)
- [Braintree 설정하기](#braintree-configuration)
- [정기 구독 모델](#subscriptions)
    - [새로운 정기 구독 생성하기](#creating-subscriptions)
    - [정기 구독 상태 확인하기](#checking-subscription-status)
    - [정기 구독 유형 변경하기](#changing-plans)
    - [정기 구독 수량 변경하기](#subscription-quantity)
    - [정기 구독의 세금계산](#subscription-taxes)
    - [정기 구독 취소하기](#cancelling-subscriptions)
    - [정기 구독 재개](#resuming-subscriptions)
- [구독 트라이얼](#subscription-trials)
    - [신용카드 사전 등록](#with-credit-card-up-front)
    - [신용카드없이 사전 등록](#without-credit-card-up-front)
- [Stripe 후킹 처리하기](#handling-stripe-webhooks)
    - [실패한 정기 구독](#handling-failed-subscriptions)
    - [기타 웹 훅](#handling-other-webhooks)
- [Braintree 웹 훅 처리하기](#handling-braintree-webhooks)
    - [실패한 정기 구독](#handling-braintree-failed-subscriptions)
    - [기타 웹 훅](#handling-braintree-other-webhooks)
- [한번만 결제하기](#single-charges)
- [청구서](#invoices)
    - [청구서 PDF 로 생성하기](#generating-invoice-pdfs)

<a name="introduction"></a>
## 소개

라라벨 캐셔는 [Stripe](https://stripe.com)와 [Braintree](https://braintreepayments.com)에 의해서 제공되는 손쉽고 편리한 구독(정기 과금) 서비스를 위한 인터페이스를 제공합니다. 라라벨 캐셔는 여러분이 작성하는데 어려움을 겪는 구독을 위한 청구서에서 필요한 거의 모든 관용구문들을 다룹니다.  기본적인 구독 관리 외에도, 캐셔를 통해서 쿠폰 관리, 구독 변경, 구매 수량 변경, 취소 유예 기간, 그리고 청구서를 PDF로 생성할 수도 있습니다. 

<a name="stripe-configuration"></a>
## Stripe 설정하기

#### 컴포저

먼저 여러분의 `composer.json` Stripe 를 위한 파일에 캐셔 패키지를 추가하고 `composer update` 명령어를 실행 시켜야 합니다.

    "laravel/cashier": "~6.0"

#### 서비스 프로바이더

다음으로 `app` 설정 파일에 `Laravel\Cashier\CashierServiceProvider` [서비스 프로바이더](/docs/{{version}}/providers)를 등록해야 합니다. 

#### 데이터베이스 마이그레이션

캐셔를 바로 사용하기 전에, 데이터베이스를 준비해야 합니다. 몇개의 컬럼을 `users` 테이블에 추가하고, 사용자의 구독 정보를 저장할 새로운 `subscriptions` 테이블을 생성해야 합니다: 

    Schema::table('users', function ($table) {
        $table->string('stripe_id')->nullable();
        $table->string('card_brand')->nullable();
        $table->string('card_last_four')->nullable();
        $table->timestamp('trial_ends_at')->nullable();
    });

    Schema::create('subscriptions', function ($table) {
        $table->increments('id');
        $table->integer('user_id');
        $table->string('name');
        $table->string('stripe_id');
        $table->string('stripe_plan');
        $table->integer('quantity');
        $table->timestamp('trial_ends_at')->nullable();
        $table->timestamp('ends_at')->nullable();
        $table->timestamps();
    });

마이그레이션들이 생성되면 `migrate` 아티즌 명령어를 실행하십시오. 

#### 모델 설정

다음으로 `Billable` 트레이트-trait을 모델에 추가합니다. 

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }


#### 프로바이더 Key

다음으로 `services.php` 설정 파일에 Stripe 키를 지정해야 합니다: 

    'stripe' => [
        'model'  => App\User::class,
        'secret' => env('STRIPE_SECRET'),
    ],

<a name="braintree-configuration"></a>
## Braintree 설정하기

#### Braintree 주의사항

대부분의 동작들에서, Stripe 와 Braintree 의 캐셔 함수의 구현부분은 동일합니다. 두개의 서비스 모두 신용카드에 대한 구독 청구를 지원합니다. 다른 점은 Braintree 는 페이팔과 같은 결제도 지원한다는 것입니다. 하지만 Braintree 는 Stripe 에서 지원하는 몇가지 기능들은 누락되어 있습니다. 여러분은 Stripe 과 Braintree 둘중에 어느 서비스를 사용할지 결정해야 합니다:

- Braintree는 PayPal을 지원하지만 Stripe는 미지원.
- Braintree는 구독에 대해 `increment` 와`decrement` 메소드 지원하지 않습니다. 이것은 Braintree 의 제한 사항으로, Cashier 에서 제한하는 것은 아닙니다.
- Braintree는 퍼센트 기반 할인을 지원하지 않습니다. 이것은 Braintree 의 제한 사항으로, Cashier 에서 제한하는 것은 아닙니다.

#### Composer

먼저 여러분의 `composer.json` Braintree 를 위한 파일에 캐셔 패키지를 추가하고 `composer update` 명령어를 실행 시켜야 합니다.

    "laravel/cashier-braintree": "~1.0"

#### 서비스 프로바이더

다음으로 `app` 설정 파일에 `Laravel\Cashier\CashierServiceProvider` [서비스 프로바이더](/docs/{{version}}/providers)를 등록해야 합니다. 

#### Plan Credit Coupon

Braintree를 통해서 캐셔를 사용하기 전에 Braintree 설정 화면에서 `plan-credit` 할인을 정의할 필요가 있습니다. 이 할인은 월별 결제에서 년단위 결제로 변경하거나, 반대로 년단위 결제에서 월별 결제로 변경할 때 정확하게 비율을 배분하는데 사용될 것입니다. 이 할인율은 Braintree 설정 패널에서 여러분이 원하는 값으로 설정할 수 있으며 고객이 쿠폰을 적용한 경우 캐셔는 정의된 값을 덮어쓰게 될 것입니다.

#### 데이터베이스 마이그레이션

캐셔를 바로 사용하기 전에, 데이터베이스를 준비해야 합니다. 몇개의 컬럼을 `users` 테이블에 추가하고, 사용자의 구독 정보를 저장할 새로운 `subscriptions` 테이블을 생성해야 합니다: 

    Schema::table('users', function ($table) {
        $table->string('braintree_id')->nullable();
        $table->string('paypal_email')->nullable();
        $table->string('card_brand')->nullable();
        $table->string('card_last_four')->nullable();
        $table->timestamp('trial_ends_at')->nullable();
    });

    Schema::create('subscriptions', function ($table) {
        $table->increments('id');
        $table->integer('user_id');
        $table->string('name');
        $table->string('braintree_id');
        $table->string('braintree_plan');
        $table->integer('quantity');
        $table->timestamp('trial_ends_at')->nullable();
        $table->timestamp('ends_at')->nullable();
        $table->timestamps();
    });

마이그레이션들이 생성되면 `migrate` 아티즌 명령어를 실행하십시오. 

#### 모델 설정하기

다음으로 `Billable` 트레이트-trait을 모델에 추가합니다. 

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

#### 키 설정하기

다음으로 `services.php`에 다음의 옵션값들을 설정해야 합니다: 

    'braintree' => [
        'model'  => App\User::class,
        'environment' => env('BRAINTREE_ENV'),
        'merchant_id' => env('BRAINTREE_MERCHANT_ID'),
        'public_key' => env('BRAINTREE_PUBLIC_KEY'),
        'private_key' => env('BRAINTREE_PRIVATE_KEY'),
    ],

그리고 다음의 Braintree SDK 호출을 `AppServiceProvider` 서비스 프로바이더의 `boot` 메소드에 추가해야 합니다:  

    \Braintree_Configuration::environment(env('BRAINTREE_ENV'));
    \Braintree_Configuration::merchantId(env('BRAINTREE_MERCHANT_ID'));
    \Braintree_Configuration::publicKey(env('BRAINTREE_PUBLIC_KEY'));
    \Braintree_Configuration::privateKey(env('BRAINTREE_PRIVATE_KEY'));

<a name="subscriptions"></a>
## 정기 구독 모델

<a name="creating-subscriptions"></a>
### 새로운 정기 구독 생성하기

새로운 정기 구독을 생성하려면, 먼저 청구가 가능한 (일반적으로 `App\User`가 되는 ) 모델 인스턴스를 찾아야 합니다. 모델 인스턴스를 찾으면, 모델에 대한 정기 구독을 생성하기 위해 `newSubscription` 메소드를 사용할 수 있습니다. 

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')->create($creditCardToken);

`newSubscription` 메소드에 전달되는 첫번째 인자는 구독의 제목이 되어야 합니다. 어플리케이션이 단 하나의 구독모델을 제공한다면, `main` 또는 `primary` 와 같이 사용할 수 있습니다. 두번째 인자는 사용자가 구독하고자 하는 지정된 Stripe / Braintree plan입니다. 이 값은 Stripe 또는 Braintree 의 plan 식별자와 일치해야 합니다.

`create` 메소드는 자동으로 정기 구독을 생성할 것이며, 여러분의 database 에 고객의 ID와 관련된 결제정보를 업데이트 합니다. 

#### 사용자의 상세 정보 추가하기

만약 여러분이 추가적인 사용자 정보를 지정하고 싶다면 이러한 정보를 `create` 메소드의 두번째 인자로 전달하면 됩니다. 

    $user->newSubscription('main', 'monthly')->create($creditCardToken, [
        'email' => $email,
    ]);

추가적인 필드에 대한 Stripe 또는 Braintree의 지원정보를 확인하고자 한다면 Stripe의 [고객 생성에 관한 문서](https://stripe.com/docs/api#create_customer) 또는  [Braintree 매뉴얼](https://developers.braintreepayments.com/reference/request/customer/create/php)을 참고하십시오. 

#### 쿠폰

새로운 구독을 생성 할 때 쿠폰을 적용할 수 있게 하려면 `withCoupon` 메소드를 사용하면 됩니다. 

    $user->newSubscription('main', 'monthly')
         ->withCoupon('code')
         ->create($creditCardToken);

<a name="checking-subscription-status"></a>
### 정기 구독 가입 상태 확인하기

사용자가 정기 구독을 시작하고 난 뒤에, 이에 대한 정보를 확인하는 것은 메소드 하나로 손쉽게 확인이 가능합니다. 먼저 `subscribed` 메소드가 `true` 를 반환한다면 사용자의 정기구독 가입 상태는 무료 평가 기간을 포함하여, 활성화 되어 있다는 것을 의미합니다. 

    if ($user->subscribed('main')) {
        //
    }

`subscribed` 메소드는 라우트 미들웨어에 사용될수 있는 좋은 방법중 하나입니다: 사용자의 구독 상태에 따라서, 라우트 및 컨트롤러에 대한 액세스를 제한할 할 수 있습니다.

    public function handle($request, Closure $next)
    {
        if ($request->user() && ! $request->user()->subscribed('main')) {
            // This user is not a paying customer...
            return redirect('billing');
        }

        return $next($request);
    }

사용자가 현재 무료 평가 기간(trial) 을 통해서 이용중인지 아닌지 확인하고자 한다면, `onTrial` 메소드를 사용하면 됩니다. 이 메소드는 사용자들에게 그들이 현재 무료 평가 기간을 이용중이라는 정보를 표시하는데 유용하게 사용될 수 있습니다. 

    if ($user->subscription('main')->onTrial()) {
        //
    }

`subscribedToPlan` 메소드는 사용자가 주어진 Stripe / Braintree plan ID에 대당하는 구독 플랜을 이용하는지 확인할 수 있습니다. 다음 예제는 사용자가 `main` 구독을 `monthly` plan 으로 구독하고 있는지 확인하게 됩니다: 

    if ($user->onPlan('monthly')) {
        //
    }

#### 정기 구독 취소하기

사용자가 이전에 한번 구독 후, 취소 했는지를 확인 하기 위해서 `cancelled` 메소드를 사용할 수 있습니다.

    if ($user->subscription('main')->cancelled()) {
        //
    }

또한 사용자가 구독을 취소하고 있지만 아직 완전히 만료 되기 전의 "유예 기간" 중인지를 확인할 수 있습니다. 예를 들어, 사용자가 3 월 5 일에 구독을 취소하고 3 월 10 일에 만료 될 경우, 해당 사용자는 3 월 10 일까지가 "유예 기간”입니다. `subscribed` 메소드는 이 기간 동안 여전히 `ture` 를 반환하는 것에 유의하십시오. 

    if ($user->subscription('main')->onGracePeriod()) {
        //
    }

<a name="changing-plans"></a>
### 정기 구독 유형 변경하기

사용자가 어플리케이션을 구독한 뒤에, 구독 플랜을 변경하고자 하는 경우는 자주 있습니다. 사용자를 새로운 구독 플랜으로 변경하게 하려면 `swap` 메소드를 사용하면 됩니다. 예를 들어 다음처럼 사용자를 `premium` 구독 플랜으로 손쉽게 변경할 수 있습니다. 

    $user = App\User::find(1);

    $user->subscription('main')->swap('provider-plan-id');

사용자가 평가기간(trial)중이라면, 평가 기간은 정상적으로 유지됩니다. 또한 구독의 "수량"이 존재하는 경우에도 이 수량은 유지됩니다. 

    $user->subscription('main')->swap('provider-plan-id');

plan 을 변경하고자 하지만 트라이얼 기간은 넘겨버리고자 한다면 `skipTrial` 메소드를 사용하면 됩니다:

    $user->subscription('main')
            ->skipTrial()
            ->swap('provider-plan-id');

<a name="subscription-quantity"></a>
### 정기 구독 수량 변경하기

> **주의:** 구독 수량은 캐셔의 Stripe 에서만 지원이 됩니다. Braintree 는 Stripe 의 "수량"과 동일한 기능을 제공하지 않습니다.

때로는 구독은 "수량"에 영향을 받을 수 있습니다. 예를 들어 여러분의 어플리케이션에서 하나의 계정의 **사용자마다** 한달에 10달러를 부과하고 있다고 한다면, `incrementQuantity` 와 `decrementQuantity` 메소드를 사용하여 구독 수를 늘리거나 줄일 수 있습니다. 


    $user = User::find(1);

    $user->subscription('main')->incrementQuantity();

    // Add five to the subscription's current quantity...
    $user->subscription('main')->incrementQuantity(5);

    $user->subscription('main')->decrementQuantity();

    // Subtract five to the subscription's current quantity...
    $user->subscription('main')->decrementQuantity(5);

이 대신에, 특정 수량을 지정하고자 한다면, `updateQuantity` 메소드를 사용하면 됩니다.

    $user->subscription('main')->updateQuantity(10);

구독 수량에 대한 보다 자세한 내용은 [Stripe 문서](https://stripe.com/docs/guides/subscriptions#setting-quantities)를 참고하십시오.

<a name="subscription-taxes"></a>
### 정기구독의 세금계산

캐셔에서는, Stripe / Braintree에 손쉽게 "세율(tax_percent)'을 전달할 수 있습니다. 고객이 구매를 진행할 때 세금에 대한 퍼센트를 지정하려면 청구가 가능한 모델에 `taxPercentage` 메소드를 사용하여 구현 소수점 자릿수가 2 자리 이내에서 0에서 100 사이의 숫자를 반환하면 됩니다.

    public function taxPercentage() {
        return 20;
    }

이렇게 하면 모델별로 세율을 적용하여 다양한 국가와 해당 국가의 사용자를 기반으로 세율을 적용할 수 이 있습니다.

<a name="cancelling-subscriptions"></a>
### 정기구독 취소하기

정기구독을 취소하기 위해서는, 간단하게 사용자의 정기구독에 대해서 `cancle` 메소드를 호출하면 됩니다:

    $user->subscription('main')->cancel();

구독이 취소되면 캐셔는 자동으로 데이터베이스의 `ends_at` 컬럼을 설정합니다. 이 컬럼은 언제 `subscribed` 메소드가 `false`를 반환해야 하는지 알기 위해서 사용되어 집니다. 예를 들어 사용자가 구독을 3월 1일에 취소했지만, 정기 구독이 3월 5일에 종료하도록 예정되어 있다면 `subscribed` 메소드는 3월 5일까지 `true`를 반환할 것입니다. 

사용자가 구독을 취소했지만 아직 "유예 기간"이 남아 있는지 확인하고자 한다면, `onGracePeriod` 메소드를 사용하면 됩니다.

    if ($user->subscription('main')->onGracePeriod()) {
        //
    }

<a name="resuming-subscriptions"></a>
### 정기 구독 재개하기

만약 사용자가 취소한 구독을 재개하려면, `resume` 메소드를 사용하면 됩니다. 사용자는 구독을 재개하기 위해 **반드시** 자신의 유예 기간에 내에 있어야 합니다:

    $user->subscription('main')->resume();

만약 사용자가 구독을 취소하고 다음 정기 구독을 재개하는 경우 그 등록의 만료일이 되기 전까지는 비용이 바로 부과되지는 않습니다. 대신, 사용자의 정기 구독은 손쉽게 재활성화 되며, 원래의 주기에 따라 과금됩니다.

<a name="subscription-trials"></a>
## 구독 트라이얼

<a name="with-credit-card-up-front"></a>
### 신용카드 사전 등록

고객에게 트라이얼 기간을 제공하고, 결제 정보를 사전에 등록 해달라고 요청하고자 한다면, 구독을 생성할 때 `trialDays` 메소드를 해야합니다:

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')
                ->trialDays(10)
                ->create($creditCardToken);

이 메소드는 데이터베이스 안의 구독 레코드에 트라이얼 기간의 종료일을 설정하고, Stripe / Braintree 에게 이 기간이 지나기 전까지 고객에게 청구하지 않도록 지시합니다. 

> **주의:** 만약 고객이 구독의 트라이얼 기간이 끝나기 전까지 구독을 취소하지 않았다면, 트라이얼이 종료되는 대로 결제가 진행될것 입니다. 따라서 고객에게 트라이얼 종료일을 알려주어야만 합니다.

고객이 현재 트라잉얼 기간중인지 판단하려고 한다면, 사용자 인스턴스에서 `onTrial` 메소드를 사용하거나, 구독 인스턴스에서 `onTrial` 메소드를 사용하십시오. 다음의 두개의 예제는 모두 동일한 목적으로 구성되었습니다:   

    if ($user->onTrial('main')) {
        //
    }

    if ($user->subscription('main')->onTrial()) {
        //
    }

<a name="without-credit-card-up-front"></a>
### 신용카드없이 사전 등록

고객에게 신용카드에 대한 결제 정보의 사전등록 없이 트라이얼 기간을 부여하고자 한다면, 간단하게 사용자의 `trial_ends_at` 컬럼에 트리이얼 종료기간을 설정하면 됩니다. 예를 들면, 이는 일반적으로 사용자를 등록할 때 설정하게 됩니다.

    $user = User::create([
        // Populate other user properties...
        'trial_ends_at' => Carbon::now()->addDays(10),
    ]);

기존 구독에 연결되는 것이 아니기 때문에, Cashier에서는 이 타입의 트라이얼을 "포괄적 시험기간(generic trial)"이라고 지칭하고 있습니다. `User` 인스턴스의 `onTrial` 메소드는 만약 현재 시간이 아직 `trial_ends_at`을 넘기지 않은 경우에 `true`를 반환합니다:

    if ($user->onTrial()) {
        // User is within their trial period...
    }

또한 특별히 사용자가 현재 "포괄적" 시험기간 중이며 아직 실제 구독을 생성하지 않았는지 알고자 한다면 `onGenericTrial` 메소드를 사용할 수 있습니다.  

    if ($user->onGenericTrial()) {
        // User is within their "generic" trial period...
    }

사용자에게 실제 구독을 생성할 준비가 되면, 일반적으로 `newSubscription` 메소드를 사용할 수 있습니다:     

    $user = User::find(1);

    $user->newSubscription('main', 'monthly')->create($creditCardToken);

<a name="handling-stripe-webhooks"></a>
## Stripe 후킹 처리하기

<a name="handling-failed-subscriptions"></a>
### 실패한 정기 구독

만약 사용자의 신용카드가 만료 되었다면? 걱정하지 마십시오 - 캐셔는 사용자의 구독을 쉽게 취소할 수 있는 Webhook 컨트롤러를 포함하고 있습니다. 컨트롤러에 라우트를 지정하면 됩니다.

    Route::post(
        'stripe/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

이게 전부입니다! 실패한 결제는 컨트롤러에 의해 확인되어 처리될 것입니다. 컨트롤러는 Stripe 가 결제에 실패하였다고 결정되면(보통 3번의 결제 시도가 실패하면) 사용자의 구독을 취소시킬 것입니다. 잊지 마십시오: 여러분은 Stripe 설정 패널에서 웹 후킹 URI를 설정해야합니다.

Stripe의 웹 후킹은 라라벨의 [CSRF 확인](/docs/{{version}}/routing#csrf-protection)을 회피 할 필요가 있기 때문에 `VerifyCsrfToken` 미들웨어에서 제외될 URI 목록에 추가해 주거나 `web` 미들웨어 그룹의 외부에 있어야 합니다.

    protected $except = [
        'stripe/*',
    ];

<a name="handling-other-webhooks"></a>
### 기타 후킹

만약 추가적인 Stripe 웹 후크 이벤트가 있다면, 간단하게 Webhook 컨트롤러를 확장하면 됩니다. 확장하는 메소드의 이름은 캐셔가 요구하는 규칙에 따라야 하고, `handle` 로 시작하며 "카멜 케이스"의 Stripe 의 webhook 이름에 부합해야 합니다. 예를 들어, `invoice.payment_succeeded` webhook을 사용하고자 한다면 컨트롤러에는 `handleInvoicePaymentSucceeded` 메소드를 추가해야 합니다.

    <?php                                          

    namespace App\Http\Controllers;

    use Laravel\Cashier\Http\Controllers\WebhookController as BaseController;

    class WebhookController extends BaseController
    {
        /**
         * Handle a Stripe webhook.
         *
         * @param  array  $payload
         * @return Response
         */
        public function handleInvoicePaymentSucceeded($payload)
        {
            // Handle The Event
        }
    }

<a name="handling-braintree-webhooks"></a>
## Braintree 웹 훅 처리하기

<a name="handling-braintree-failed-subscriptions"></a>
### 실패한 정기 구독

고객의 신용카드가 만료되었다구요? 걱정하지 마십시오 - 캐셔는 이러한 경우를 대비하여 고객의 구독을 손쉽게 취소할 수 있는 웹훅 컨트롤러를 포함하고 있습니다. 간단히 컨트롤러에 라우트를 지정하기만 하면 됩니다: 

    Route::post(
        'braintree/webhook',
        '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook'
    );

이게 전부입니다! 실패한 결제는 컨트롤러에 의해 확인되어 처리될 것입니다. 컨트롤러는 사용자의 구독을 취소처리할 것입니다. 컨트롤러는 Braintree 가 결제에 실패하였다고 결정되면(보통 3번의 결제 시도가 실패하면) 사용자의 구독을 취소시킬 것입니다. 잊지 마십시오: 여러분은 Braintree 설정 패널에서 웹 후킹 URI를 설정해야합니다.

Braintree의 웹 후킹은 라라벨의 [CSRF 확인](/docs/{{version}}/routing#csrf-protection)을 회피 할 필요가 있기 때문에 `VerifyCsrfToken` 미들웨어에서 제외될 URI 목록에 추가해 주거나 `web` 미들웨어 그룹의 외부에 있어야 합니다.

    protected $except = [
        'braintree/*',
    ];

<a name="handling-braintree-other-webhooks"></a>
### 기타 후킹

만약 추가적인 Braintree 웹 후크 이벤트가 있다면, 간단하게 Webhook 컨트롤러를 확장하면 됩니다. 확장하는 메소드의 이름은 캐셔가 요구하는 규칙에 따라야 하고, `handle` 로 시작하며 "카멜 케이스"의 Braintree 의 webhook 컨벤션에 부합해야 합니다. 예를 들어, `dispute_opened` webhook을 사용하고자 한다면 컨트롤러에는 `handleDisputeOpened` 메소드를 추가해야 합니다.

    <?php

    namespace App\Http\Controllers;

    use Braintree\WebhookNotification;
    use Laravel\Cashier\Http\Controllers\WebhookController as BaseController;

    class WebhookController extends BaseController
    {
        /**
         * Handle a Braintree webhook.
         *
         * @param  WebhookNotification  $webhook
         * @return Response
         */
        public function handleDisputeOpened(WebhookNotification $notification)
        {
            // Handle The Event
        }
    }

<a name="single-charges"></a>
## 한번만 결제하기

### Simple Charge

> **주의:** Stripe 를 사용중일 떄, `charge` 메소드는 청구하려는 **어플리케이션에서 사용하는 통화의 가장 낮은 기준 금액**을 인수로 받습니다. 하지만 Braintree를 사용중이라면 `charge` 메소드에 전체 금액을 전달해야만 합니다.    

정기 구독하고 있는 고객의 신용 카드에 대해 "일회성 결제"를 하고 싶을 때에는 결제가 가능한 모델 인스턴스에 대해서 `charge` 메소드를 사용합니다. 

    // Stripe Accepts Charges In Cents...
    $user->charge(100);

    // Braintree Accepts Charges In Dollars...
    $user->charge(1);

`charge` 메소드는 두번째 인자로 Stripe / Braintree 청구에 사용할 수 있는 옵션에 대한 배열을 전달 받습니다.

    $user->charge(100, [
        'custom_option' => $value,
    ]);

`charge` 메소드는 결제가 실패했을 경우에 예외-exception을 발생시킵니다. 결제가 성공적으로 완료되었다면 메소드에서는 Stripe / Braintree 응답 객체가 반환됩니다. 

    try {
        $response = $user->charge(100);
    } catch (Exception $e) {
        //
    }

### 결제와 청구서

떄로는 한 번만 결제를 진행하면서 고객에게 PDF 영수증을 발행하고자 할 수도 있습니다. `invoiceFor` 메소드는 바로 이경우 사용합니다. 예를 들어, 고객에게 "한번의 요금"을 위해서 5달러를 청구해 보겠습니다:

    // Stripe Accepts Charges In Cents...
    $user->invoiceFor('One Time Fee', 500);

    // Braintree Accepts Charges In Dollars...
    $user->invoiceFor('One Time Fee', 5);

청구서는 즉시 고객의 신용카드를 통해서 결제될 것입니다. `invoiceFor` 메소드는 또한 Stripe / Braintree 결제 옵션을 지정하기 위한 배열을 세번째 인자로 전달 받을 수 있습니다:

    $user->invoiceFor('One Time Fee', 500, [
        'custom-option' => $value,
    ]);

> **주의:** `invoiceFor` 메소드는 결제 시도가 실패한 경우 이를 재시도하는 Stripe 청구서를 생성할 것입니다. 실패한 결제의 재시도를 위한 청구를 수행하지 않으려면, 결제 시도가 실패한 뒤에 Stripe API를 사용하여 청구서를 종료(close) 해야 합니다.  

<a name="invoices"></a>
## 청구서

`invoices` 메소드를 사용하여 청구가 가능한 모델의 청구서를 손쉽게 배열 형태로 조회 할 수 있습니다:

    $invoices = $user->invoices();

사용자의 청구서를 나열하는 경우, 청구서 관련 정보를 표시하기 위해서 청구서의 헬퍼 함수를 사용할 수 있습니다. 예를 들어 사용자가 쉽게 다운로드 할 수 있도록 테이블 내의 모든 청구서를 목록으로 보여주고자 할 수도 있습니다.

    <table>
        @foreach ($invoices as $invoice)
            <tr>
                <td>{{ $invoice->date()->toFormattedDateString() }}</td>
                <td>{{ $invoice->total() }}</td>
                <td><a href="/user/invoice/{{ $invoice->id }}">Download</a></td>
            </tr>
        @endforeach
    </table>

<a name="generating-invoice-pdfs"></a>
#### 청구서 PDF 로 생성하기

청구서를 PDF 로 생성하기 전에, `dompdf` PHP 라이브러리를 설치해야 합니다:

    composer require dompdf/dompdf

라우트 또는 컨트롤러 안에서 `downloadInvoice` 메소드를 사용하여 청구서를 PDF 로 생성하고 다운로드 할 수 있습니다. 이 메소드는 자동으로 브라우저서 다운로드 할 수 있는 HTTP 응답을 생성할 것입니다. 

    Route::get('user/invoice/{invoice}', function ($invoiceId) {
        return Auth::user()->downloadInvoice($invoiceId, [
            'vendor'  => 'Your Company',
            'product' => 'Your Product',
        ]);
    });
