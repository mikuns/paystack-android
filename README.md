[![API](https://img.shields.io/badge/API-16%2B-blue.svg?style=plastic)](https://android-arsenal.com/api?level=16)




## Paystack Android

This is a library for easy integration of [Paystack](https://paystack.com) with your Android application. Use this
library in your Android app so we shoulder the burden of PCI compliance by helping you
avoid the need to send card data directly to your server. Instead, this library sends credit
card data directly to our servers.

## Summarized flow

1 Time to pay (user has provided card details on your app)

2 OPTION 1: Backend starts transaction (recommended)

a. App prompts backend to initialize a transaction, backend returns `access_code`. Backend will use this endpoint: https://developers.paystack.co/reference#initialize-a-transaction

b. Provide `access_code` and card details to our SDK's `chargeCard` function via `Charge` object

2 OPTION 2: App starts transaction

a. Provide transaction parameters and card details to our SDK's `chargeCard` function via `Charge` object

3 SDK will prompt user for PIN, OTP or Bank authentication as required

4 Once successful, we will send event to your webhook url and call onSuccess callback

## Requirements
- Android SDKv16 (Android 4.1 "Jelly Bean") - This is the first SDK version that includes
`TLSv1.2` which is required by our servers. Native app support for user devices older than
API 16 will not be available.

## Installation

### Android Studio (using Gradle)
You do not need to clone this repository or download the files. Just add the following lines to your app's `build.gradle`:

```gradle
dependencies {
    compile 'co.paystack.android:paystack:3.0.9'
}
```

### Eclipse
To use this library with Eclipse, you need to:

1. Clone the repository.
2. Import the **paystack** folder into your [Eclipse](http://help.eclipse.org/juno/topic/org.eclipse.platform.doc.user/tasks/tasks-importproject.htm) project
3. In your project settings, add the **paystack** project under the Libraries section of the Android category.

## Usage

### 0. Prepare for use

To prepare for use, you must ensure that your app has internet permissions by making sure the `uses-permission` line below is present in the AndroidManifest.xml.

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### 1. initializeSdk

To use the Paystack Android SDK, you need to first initialize it using the `PaystackSdk` class.

```java
public class App extends Application{
    @Override
    public void onCreate() {
        super.onCreate();

        PaystackSdk.initialize(getApplicationContext());
    }
}
```

Make sure to call this method in the `onCreate` method of your Fragment or Activity or Application.

### 2. Set your Public Key

Before you can charge a card with the `PaystackSdk` class, you need to set your public key. The library provides two approaches,

#### a. Add the following lines to the `<application></application>` tag of your AndroidManifest.xml

```xml
<meta-data
    android:name="co.paystack.android.PublicKey"
    android:value="your public key obtained from: https://dashboard.paystack.co/#/settings/developer"/>
```

#### b. Set the public key by code

This can be done anytime in your code. Just be sure to initialize before calling `chargeCard`.

```java
class Bootstrap {
    public static void setPaystackKey(String publicKey) {
        PaystackSdk.setPublicKey(publicKey);
    }
}
```
### 3. Collect and validate card details

At this time, we expect you to provide fields on your activity that collect the card details. Our `Card` class allows you collect and verify these. The library provides validation methods to validate the fields of the card.

#### card.validNumber
This method helps to perform a check if the card number is valid.

#### card.validCVC
Method that checks if the card security code is valid.

#### card.validExpiryDate
Method checks if the expiry date (combination of year and month) is valid.

#### card.isValid
Method to check if the card is valid. Always do this check, before charging the card.

#### card.getType
This method returns an estimate of the string representation of the card type.


### 4. chargeCard
Charging with the PaystackSdk is quite straightforward.

#### Parameters for the chargeCard function
- **Activity** - The first argument to the `PaystackSdk.chargeCard` is the calling Activity object. Always
give an Activity that will stay open till the end of the transaction. The currently
open Activity is just fine.


- **Charge** - This object allows you provide information about the transaction to be made. Before calling 
`chargeCard`, you should do a `charge.setCard(card)`. The charge can then be used in either of 2 ways
    - **Resume an initialized transaction**: If employing this flow, you would send all required parameters 
    for the transaction from your backend to the Paystack API via the `transaction/initialize` call - 
    documented [here](https://developers.paystack.co/reference#initialize-a-transaction).. The 
    response of the call includes an `access_code`. This can be used to charge the card by doing 
    `charge.setAccessCode({value from backend})`. Once an access code is set, the only other parameter
    relevant to the transaction is the card. Others will be ignored.
    - **Initiate a fresh transaction on Paystack**: Using the functions: `setCurrency`, `setPlan`,
     `setSubaccount`, `setTransactionCharge`, `setAmount`, `setEmail`, `setReference`, `setBearer`,
     `putMetadata`, `putCustomField`, you can set up a fresh transaction direct from the SDK. 
     Documentation for these parameters are same as for `transaction/initialize`.


- **Transaction Callback** - When an error occurs or transaction concludes successfully, we will call 
the methods available in the callback you provided. 
    - `OnSuccess` will be called once the charge succeeds.
    - `beforeValidate` is called every time the SDK needs to request user input. This function currently only allows the app know that the SDK is requesting further user input. 
    - `OnError` is called if an error occurred during processing. Some Exception types that you should watch include
        - *ExpiredAccessCodeException*: This would be thrown if the access code has already been used to attempt a charge.
        - *ChargeException*: This would be thrown if the charge failed. It would hold the message from the server.


```java
public class MainActivity extends AppCompatActivity {
  
  // This is the subroutine you will call after creating the charge
  // adding a card and setting the access_code
   public void performCharge(){
       PaystackSdk.chargeCard(MainActivity.this, charge, new Paystack.TransactionCallback() {
           @Override
           public void onSuccess(Transaction transaction) {
               // This is called only after transaction is deemed successful.
               // Retrieve the transaction, and send its reference to your server
               // for verification.
           }

           @Override
           public void beforeValidate(Transaction transaction) {
               // This is called only before requesting OTP.
               // Save reference so you may send to server. If
               // error occurs with OTP, you should still verify on server.
           }

           @Override
           public void onError(Throwable error, Transaction transaction) {
             //handle error here
           }

       });
   }
}
```

Note that once `chargeCard` is called, depending on settings agreed with Paystack's Customer Success team, the SDK _may_ prompt the user to provide their PIN, an OTP or conclude Bank Authentication. These 
are currently being managed entirely by the SDK. Your app will only be notified via the `beforeValidate` function of the
callback when OTP or Bank Authentication is about to start.


### 5. Verifying the transaction
Send the reference to your backend and verify by calling our REST API. An authorization will be returned which
will let you know if its code is reusable. You can learn more about our verify call
 [here](https://developers.paystack.co/reference#verifying-transactions).
 
Below is a sample authorization object returned along with the transaction details:
 ```json
    {
      "status": true,
      "message": "Verification successful",
      "data": {
        "amount": 10000,
        "currency": "NGN",
        "transaction_date": "2017-04-06T21:28:41.000Z",
        "status": "success",
        "reference": "d68rbovh4a",
        "domain": "live",
        "metadata": {
          "custom_fields": [
            {
              "display_name": "Started From",
              "variable_name": "started_from",
              "value": "sample charge card backend"
            },
            {
              "display_name": "Requested by",
              "variable_name": "requested_by",
              "value": "some person"
            },
            {
              "display_name": "Server",
              "variable_name": "server",
              "value": "some.herokuapp.com"
            }
          ]
        },
        "gateway_response": "Approved",
        "message": "Approved",
        "channel": "card",
        "ip_address": "41.31.21.11",
        "log": null,
        "fees": 150,
        "authorization": {
          "authorization_code": "AUTH_blahblah",
          "bin": "412345",
          "last4": "6789",
          "exp_month": "10",
          "exp_year": "2345",
          "channel": "card",
          "card_type": "mastercard debit",
          "bank": "Some Bank",
          "country_code": "NG",
          "brand": "mastercard",
          "reusable": true,
          "signature": "SIG_IJOJidkpd0293undjd"
        },
        "customer": {
          "id": 22421,
          "first_name": "Guava",
          "last_name": "Juice",
          "email": "guava@juice.me",
          "customer_code": "CUS_6t6che6w8hmt",
          "phone": "",
          "metadata": {},
          "risk_action": "default"
        },
        "plan": null
      }
    }
  ```
 To reuse the authorization gotten from charging this customer in future, you need to do 2 tests:
 1. In the sample JSON above, you can conclude that the transaction was successful because `data.status`="success".
 This means the authorization is active.
 2. Confirm that the authorization is reusable by checking `data.authorization.reusable` which is true in this case.
 Once both pass, you can save the authorization code against the customer's email.

### 6. Charging a card authorization from your server in future
To charge an authorization saved from concluding chargeCard, you need its authorization code and the custmer's email. The `charge_authorization` endpoint is documented [here](https://developers.paystack.co/docs/charging-returning-customers).

## Testing your implementation
You can (and should) test your implementation of the Paystack Android library in your Android app. You need the details of an
actual debit/credit card to do this, so we provide ##_test cards_## for your use instead of using your own debit/credit cards. 
You may find test cards on [this Paystack documentation page](https://developers.paystack.co/docs/test-cards).

To try out the OTP flow, we have provided a test "verve" card:

```
50606 66666 66666 6666
CVV: 123
PIN: 1234
TOKEN: 123456
```

Remember to use all test cards only with test keys. Also note that all bank issued cards will be declined in test mode.

## Building the example project

1. Clone the repository.
2. Import the project either using Android Studio or Eclipse
3. Deploy a sample backend from [PaystackJS-Sample-Backend (PHP)](https://github.com/PaystackHQ/PaystackJS-Sample-code) or [Sample charge card backend (NodeJS heroku single click deploy)](https://github.com/PaystackHQ/sample-charge-card-backend)
4. Copy the endpoints from the deployed backend to your `MainActivity.java` file. In 
the case of `verify`, only copy up to the `/` before the `:`
5. Add your public key to your `MainActivity.java` file
    - Note that the public key must match the secret key,
    else none of the transactions will be attempted
6. Build and run the project on your device or emulator

## FAQs

### Is authorization_code (https://developers.paystack.co/reference#charging-returning-customers) the same as the access_code)?

No

### Where do I get an access_code?

Initialize a transaction : https://developers.paystack.co/reference#initialize-a-transaction 

### Where do I get an authorization_code?

Verify a successful transaction : https://developers.paystack.co/reference#verify-transaction 

### If I’m trying to use the Android SDK to charge someone who we’ve previously charged, can I use the authorization_code?

You don't need the SDK to charge an authorization code. It doesn't even know of its existence. Rather, use our charge endpoint: https://developers.paystack.co/reference#charge-authorization 

## Security

If you discover any security related issues, please email support@paystack.com instead of using the issue tracker.

## Contact

For more enquiries and technical questions regarding the Android PaystackSdk, please post on 
our issue tracker: [https://github.com/PaystackHQ/paystack-android/issues](https://github.com/PaystackHQ/paystack-android/issues).

## Change log

Please see [CHANGELOG](CHANGELOG.md) for more information what has changed recently.

