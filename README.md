# Flutter_Phone_Number_Authentication


  firebase_auth:
  firebase_core:
  pinput:
  
  
  Authenticate with Firebase on Android using a Phone Number

You can use Firebase Authentication to sign in a user by sending an SMS message to the user's phone. The user signs in using a one-time code contained in the SMS message.

The easiest way to add phone number sign-in to your app is to use FirebaseUI, which includes a drop-in sign-in widget that implements sign-in flows for phone number sign-in, as well as password-based and federated sign-in. This document describes how to implement a phone number sign-in flow using the Firebase SDK.
Phone numbers that end users provide for authentication will be sent and stored by Google to improve our spam and abuse prevention across Google services, including but not limited to Firebase. Developers should ensure they have appropriate end-user consent prior to using the Firebase Authentication phone number sign-in service.
Before you begin
If you haven't already, add Firebase to your Android project.
In your module (app-level) Gradle file (usually <project>/<app-module>/build.gradle), add the dependency for the Firebase Authentication Android library. We recommend using the Firebase Android BoM to control library versioning.
Kotlin+KTX
Java

    dependencies {
        // Import the BoM for the Firebase platform
        implementation platform('com.google.firebase:firebase-bom:31.4.0')

        // Add the dependency for the Firebase Authentication library
        // When using the BoM, you don't specify versions in Firebase library dependencies
        implementation 'com.google.firebase:firebase-auth-ktx'
    }

    By using the Firebase Android BoM, your app will always use compatible versions of Firebase Android libraries.

    (Alternative) Add Firebase library dependencies without using the BoM
    If you haven't yet connected your app to your Firebase project, do so from the Firebase console.
    If you haven't already set your app's SHA-1 hash in the Firebase console, do so. See Authenticating Your Client for information about finding your app's SHA-1 hash.

Security concerns

Authentication using only a phone number, while convenient, is less secure than the other available methods, because possession of a phone number can be easily transferred between users. Also, on devices with multiple user profiles, any user that can receive SMS messages can sign in to an account using the device's phone number.

If you use phone number based sign-in in your app, you should offer it alongside more secure sign-in methods, and inform users of the security tradeoffs of using phone number sign-in.
Enable Phone Number sign-in for your Firebase project

To sign in users by SMS, you must first enable the Phone Number sign-in method for your Firebase project:

    In the Firebase console, open the Authentication section.
    On the Sign-in Method page, enable the Phone Number sign-in method.

Firebase's phone number sign-in request quota is high enough that most apps won't be affected. However, if you need to sign in a very high volume of users with phone authentication, you might need to upgrade your pricing plan. See the pricing page.
By enabling phone number authentication on Android, you agree to the Play Integrity terms and conditions.
Enable app verification

To use phone number authentication, Firebase must be able to verify that phone number sign-in requests are coming from your app. There are three ways Firebase Authentication accomplishes this:

    Play Integrity API: If a user has a device with Google Play services installed, and Firebase Authentication can verify the device as legitimate with the Play Integrity API, phone number sign-in can proceed. The Play Integrity API is enabled on a Google-owned project by Firebase Authentication, not on your project. This does not contribute to any Play Integrity API quotas on your project. Play Integrity Support is available with the Authentication SDK v21.2.0+ (Firebase BoM v31.4.0+).

    To use Play Integrity, if you haven't yet specified your app's SHA-256 fingerprint, do so from the Project settings of the Firebase console. Refer to Authenticating Your Client for details on how to get your app's SHA-256 fingerprint.
    SafetyNet: If a user has a device with Google Play services installed, and Firebase Authentication can verify the device as legitimate with Android SafetyNet, phone number sign-in can proceed.
    The SafetyNet Attestation API is deprecated and has been replaced by the Play Integrity API. To use the Play Integrity API, make sure that your app uses the Authentication SDK v21.2.0+ (Firebase BoM v31.4.0+). If you already enabled the SafetyNet API in your project, Firebase Authentication will use SafetyNet as a fallback to Play Integrity until SafetyNet is fully deprecated.
    SafetyNet has a default quota that is sufficient for most apps. See SafetyNet Quota Monitoring for more information.
    reCAPTCHA verification: In the event that Play Integrity or SafetyNet cannot be used, such as when a user has a device without Google Play services installed, Firebase Authentication uses a reCAPTCHA verification to complete the phone sign-in flow. The reCAPTCHA challenge can often be completed without the user having to solve anything. Note that this flow requires that a SHA-1 is associated with your application. This flow also requires your API Key to be unrestricted or allowlisted for PROJECT_ID.firebaseapp.com.
    The reCAPTCHA flow will only be triggered when Play Integrity and SafetyNet are unavailable. Nonetheless, you should ensure that both scenarios are working correctly.
    Starting in the Authentication SDK v21.2.0 (Firebase BoM v31.4.0), the activity parameter is optional. However, if the activity is not set and reCAPTCHA verification is attempted, a FirebaseAuthMissingActivityForRecaptchaException is thrown, which can be handled in the onVerificationFailed callback.

You can force the reCAPTCHA verification flow with forceRecaptchaFlowForTesting You can disable app verification (when using fictional phone numbers) using setAppVerificationDisabledForTesting.
Send a verification code to the user's phone

To initiate phone number sign-in, present the user an interface that prompts them to type their phone number. Legal requirements vary, but as a best practice and to set expectations for your users, you should inform them that if they use phone sign-in, they might receive an SMS message for verification and standard rates apply.

Then, pass their phone number to the PhoneAuthProvider.verifyPhoneNumber method to request that Firebase verify the user's phone number. For example:
Kotlin+KTX
Java

val options = PhoneAuthOptions.newBuilder(auth)
    .setPhoneNumber(phoneNumber)       // Phone number to verify
    .setTimeout(60L, TimeUnit.SECONDS) // Timeout and unit
    .setActivity(this)                 // Activity (for callback binding)
    .setCallbacks(callbacks)          // OnVerificationStateChangedCallbacks
    .build()
PhoneAuthProvider.verifyPhoneNumber(options)

Note: Depending on your billing plan, you might be limited to a daily quota of SMS messages sent. See Firebase Authentication Limits.

The verifyPhoneNumber method is reentrant: if you call it multiple times, such as in an activity's onStart method, the verifyPhoneNumber method will not send a second SMS unless the original request has timed out.

You can use this behavior to resume the phone number sign in process if your app closes before the user can sign in (for example, while the user is using their SMS app). After you call verifyPhoneNumber, set a flag that indicates verification is in progress. Then, save the flag in your Activity's onSaveInstanceState method and restore the flag in onRestoreInstanceState. Finally, in your Activity's onStart method, check if verification is already in progress, and if so, call verifyPhoneNumber again. Be sure to clear the flag when verification completes or fails (see Verification callbacks).

To easily handle screen rotation and other instances of Activity restarts, pass your Activity to the verifyPhoneNumber method. The callbacks will be auto-detached when the Activity stops, so you can freely write UI transition code in the callback methods.

The SMS message sent by Firebase can also be localized by specifying the auth language via the setLanguageCode method on your Auth instance.
Kotlin+KTX
Java

auth.setLanguageCode("fr")
// To apply the default app language instead of explicitly setting it.
// auth.useAppLanguage()

When you call PhoneAuthProvider.verifyPhoneNumber, you must also provide an instance of OnVerificationStateChangedCallbacks, which contains implementations of the callback functions that handle the results of the request. For example:
Kotlin+KTX
Java

callbacks = object : PhoneAuthProvider.OnVerificationStateChangedCallbacks() {

    override fun onVerificationCompleted(credential: PhoneAuthCredential) {
        // This callback will be invoked in two situations:
        // 1 - Instant verification. In some cases the phone number can be instantly
        //     verified without needing to send or enter a verification code.
        // 2 - Auto-retrieval. On some devices Google Play services can automatically
        //     detect the incoming verification SMS and perform verification without
        //     user action.
        Log.d(TAG, "onVerificationCompleted:$credential")
        signInWithPhoneAuthCredential(credential)
    }

    override fun onVerificationFailed(e: FirebaseException) {
        // This callback is invoked in an invalid request for verification is made,
        // for instance if the the phone number format is not valid.
        Log.w(TAG, "onVerificationFailed", e)

        if (e is FirebaseAuthInvalidCredentialsException) {
            // Invalid request
        } else if (e is FirebaseTooManyRequestsException) {
            // The SMS quota for the project has been exceeded
        } else if (e is FirebaseAuthMissingActivityForRecaptchaException) {
            // reCAPTCHA verification attempted with null Activity
        }

        // Show a message and update the UI
    }

    override fun onCodeSent(
        verificationId: String,
        token: PhoneAuthProvider.ForceResendingToken
    ) {
        // The SMS verification code has been sent to the provided phone number, we
        // now need to ask the user to enter the code and then construct a credential
        // by combining the code with a verification ID.
        Log.d(TAG, "onCodeSent:$verificationId")

        // Save verification ID and resending token so we can use them later
        storedVerificationId = verificationId
        resendToken = token
    }
}

Verification callbacks

In most apps, you implement the onVerificationCompleted, onVerificationFailed, and onCodeSent callbacks. You might also implement onCodeAutoRetrievalTimeOut, depending on your app's requirements.
onVerificationCompleted(PhoneAuthCredential)

This method is called in two situations:

    Instant verification: in some cases the phone number can be instantly verified without needing to send or enter a verification code.
    Auto-retrieval: on some devices, Google Play services can automatically detect the incoming verification SMS and perform verification without user action. (This capability might be unavailable with some carriers.) This uses the SMS Retriever API, which includes an 11 character hash at the end of the SMS message.

In either case, the user's phone number has been verified successfully, and you can use the PhoneAuthCredential object that's passed to the callback to sign in the user.

onVerificationFailed(FirebaseException)

This method is called in response to an invalid verification request, such as a request that specifies an invalid phone number or verification code.
onCodeSent(String verificationId, PhoneAuthProvider.ForceResendingToken)

Optional. This method is called after the verification code has been sent by SMS to the provided phone number.

When this method is called, most apps display a UI that prompts the user to type the verification code from the SMS message. (At the same time, auto-verification might be proceeding in the background.) Then, after the user types the verification code, you can use the verification code and the verification ID that was passed to the method to create a PhoneAuthCredential object, which you can in turn use to sign in the user. However, some apps might wait until onCodeAutoRetrievalTimeOut is called before displaying the verification code UI (not recommended).
onCodeAutoRetrievalTimeOut(String verificationId)

Optional. This method is called after the timeout duration specified to verifyPhoneNumber has passed without onVerificationCompleted triggering first. On devices without SIM cards, this method is called immediately because SMS auto-retrieval isn't possible.

Some apps block user input until the auto-verification period has timed out, and only then display a UI that prompts the user to type the verification code from the SMS message (not recommended).
Create a PhoneAuthCredential object

After the user enters the verification code that Firebase sent to the user's phone, create a PhoneAuthCredential object, using the verification code and the verification ID that was passed to the onCodeSent or onCodeAutoRetrievalTimeOut callback. (When onVerificationCompleted is called, you get a PhoneAuthCredential object directly, so you can skip this step.)

To create the PhoneAuthCredential object, call PhoneAuthProvider.getCredential:
Kotlin+KTX
Java

val credential = PhoneAuthProvider.getCredential(verificationId!!, code)

To prevent abuse, Firebase enforces a limit on the number of SMS messages that can be sent to a single phone number within a period of time. If you exceed this limit, phone number verification requests might be throttled. If you encounter this issue during development, use a different phone number for testing, or try the request again later.
Sign in the user

After you get a PhoneAuthCredential object, whether in the onVerificationCompleted callback or by calling PhoneAuthProvider.getCredential, complete the sign-in flow by passing the PhoneAuthCredential object to FirebaseAuth.signInWithCredential:
Kotlin+KTX
Java

private fun signInWithPhoneAuthCredential(credential: PhoneAuthCredential) {
    auth.signInWithCredential(credential)
            .addOnCompleteListener(this) { task ->
                if (task.isSuccessful) {
                    // Sign in success, update UI with the signed-in user's information
                    Log.d(TAG, "signInWithCredential:success")

                    val user = task.result?.user
                } else {
                    // Sign in failed, display a message and update the UI
                    Log.w(TAG, "signInWithCredential:failure", task.exception)
                    if (task.exception is FirebaseAuthInvalidCredentialsException) {
                        // The verification code entered was invalid
                    }
                    // Update UI
                }
            }
}

Test with fictional phone numbers

You can set up fictional phone numbers for development via the Firebase console. Testing with fictional phone numbers provides these benefits:

    Test phone number authentication without consuming your usage quota.
    Test phone number authentication without sending an actual SMS message.
    Run consecutive tests with the same phone number without getting throttled. This minimizes the risk of rejection during App store review process if the reviewer happens to use the same phone number for testing.
    Test readily in development environments without any additional effort, such as the ability to develop in an iOS simulator or an Android emulator without Google Play Services.
    Write integration tests without being blocked by security checks normally applied on real phone numbers in a production environment.

Fictional phone numbers must meet these requirements:

    Make sure you use phone numbers that are indeed fictional, and do not already exist. Firebase Authentication does not allow you to set existing phone numbers used by real users as test numbers. One option is to use 555 prefixed numbers as US test phone numbers, for example: +1 650-555-3434
    Phone numbers have to be correctly formatted for length and other constraints. They will still go through the same validation as a real user's phone number.
    You can add up to 10 phone numbers for development.
    Use test phone numbers/codes that are hard to guess and change those frequently.

Create fictional phone numbers and verification codes

    In the Firebase console, open the Authentication section.
    In the Sign in method tab, enable the Phone provider if you haven't already.
    Open the Phone numbers for testing accordion menu.
    Provide the phone number you want to test, for example: +1 650-555-3434.
    Provide the 6-digit verification code for that specific number, for example: 654321.
    Add the number. If there's a need, you can delete the phone number and its code by hovering over the corresponding row and clicking the trash icon.

Manual testing

You can directly start using a fictional phone number in your application. This allows you to perform manual testing during development stages without running into quota issues or throttling. You can also test directly from an iOS simulator or Android emulator without Google Play Services installed.

When you provide the fictional phone number and send the verification code, no actual SMS is sent. Instead, you need to provide the previously configured verification code to complete the sign in.

On sign-in completion, a Firebase user is created with that phone number. The user has the same behavior and properties as a real phone number user, and can access Realtime Database/Cloud Firestore and other services the same way. The ID token minted during this process has the same signature as a real phone number user.
Because the ID token for the fictional phone number has the same signature as a real phone number user, it is important to store these numbers securely and to continuously recycle them.

Another option is to set a test role via custom claims on these users to differentiate them as fake users if you want to further restrict access.

To manually trigger the reCAPTCHA flow for testing, use the forceRecaptchaFlowForTesting() method.

// Force reCAPTCHA flow
FirebaseAuth.getInstance().getFirebaseAuthSettings().forceRecaptchaFlowForTesting();

Integration testing

In addition to manual testing, Firebase Authentication provides APIs to help write integration tests for phone auth testing. These APIs disable app verification by disabling the reCAPTCHA requirement in web and silent push notifications in iOS. This makes automation testing possible in these flows and easier to implement. In addition, they help provide the ability to test instant verification flows on Android.
Make sure app verification is not disabled for production apps and that no fictional phone numbers are hardcoded in your production app.

On Android, call setAppVerificationDisabledForTesting() before the signInWithPhoneNumber call. This disables app verification automatically, allowing you to pass the phone number without manually solving it. Even though Play Integrity, SafetyNet, and reCAPTCHA are disabled, using a real phone number will still fail to complete sign in. Only fictional phone numbers can be used with this API.

// Turn off phone auth app verification.
FirebaseAuth.getInstance().getFirebaseAuthSettings()
   .setAppVerificationDisabledForTesting();

Calling verifyPhoneNumber with a fictional number triggers the onCodeSent callback, in which you'll need to provide the corresponding verification code. This allows testing in Android Emulators.
Java
Kotlin+KTX

String phoneNum = "+16505554567";
String testVerificationCode = "123456";

// Whenever verification is triggered with the whitelisted number,
// provided it is not set for auto-retrieval, onCodeSent will be triggered.
FirebaseAuth auth = FirebaseAuth.getInstance();
PhoneAuthOptions options = PhoneAuthOptions.newBuilder(auth)
        .setPhoneNumber(phoneNum)
        .setTimeout(60L, TimeUnit.SECONDS)
        .setActivity(this)
        .setCallbacks(new PhoneAuthProvider.OnVerificationStateChangedCallbacks() {
            @Override
            public void onCodeSent(@NonNull String verificationId,
                                   @NonNull PhoneAuthProvider.ForceResendingToken forceResendingToken) {
                // Save the verification id somewhere
                // ...

                // The corresponding whitelisted code above should be used to complete sign-in.
                MainActivity.this.enableUserManuallyInputCode();
            }

            @Override
            public void onVerificationCompleted(@NonNull PhoneAuthCredential phoneAuthCredential) {
                // Sign in with the credential
                // ...
            }

            @Override
            public void onVerificationFailed(@NonNull FirebaseException e) {
                // ...
            }
        })
        .build();
PhoneAuthProvider.verifyPhoneNumber(options);

Additionally, you can test auto-retrieval flows in Android by setting the fictional number and its corresponding verification code for auto-retrieval by calling setAutoRetrievedSmsCodeForPhoneNumber.

When verifyPhoneNumber is called, it triggers onVerificationCompleted with the PhoneAuthCredential directly. This works only with fictional phone numbers.

Make sure this is disabled and no fictional phone numbers are hardcoded in your app when publishing your application to the Google Play store.
Java
Kotlin+KTX

// The test phone number and code should be whitelisted in the console.
String phoneNumber = "+16505554567";
String smsCode = "123456";

FirebaseAuth firebaseAuth = FirebaseAuth.getInstance();
FirebaseAuthSettings firebaseAuthSettings = firebaseAuth.getFirebaseAuthSettings();

// Configure faking the auto-retrieval with the whitelisted numbers.
firebaseAuthSettings.setAutoRetrievedSmsCodeForPhoneNumber(phoneNumber, smsCode);

PhoneAuthOptions options = PhoneAuthOptions.newBuilder(firebaseAuth)
        .setPhoneNumber(phoneNumber)
        .setTimeout(60L, TimeUnit.SECONDS)
        .setActivity(this)
        .setCallbacks(new PhoneAuthProvider.OnVerificationStateChangedCallbacks() {
            @Override
            public void onVerificationCompleted(@NonNull PhoneAuthCredential credential) {
                // Instant verification is applied and a credential is directly returned.
                // ...
            }

            // ...
        })
        .build();
PhoneAuthProvider.verifyPhoneNumber(options);

Next steps

After a user signs in for the first time, a new user account is created and linked to the credentials—that is, the user name and password, phone number, or auth provider information—the user signed in with. This new account is stored as part of your Firebase project, and can be used to identify a user across every app in your project, regardless of how the user signs in.

    In your apps, you can get the user's basic profile information from the FirebaseUser object. See Manage Users.

    In your Firebase Realtime Database and Cloud Storage Security Rules, you can get the signed-in user's unique user ID from the auth variable, and use it to control what data a user can access.

You can allow users to sign in to your app using multiple authentication providers by linking auth provider credentials to an existing user account.

To sign out a user, call signOut:
Kotlin+KTX
Java

Firebase.auth.signOut()
Get Started with Firebase Authentication on Flutter

Connect your app to Firebase

Install and initialize the Firebase SDKs for Flutter if you haven't already done so.
Add Firebase Authentication to your app

From the root of your Flutter project, run the following command to install the plugin:

flutter pub add firebase_auth

Once complete, rebuild your Flutter application:

flutter run

Import the plugin in your Dart code:

    import 'package:firebase_auth/firebase_auth.dart';

To use an authentication provider, you need to enable it in the Firebase console. Go to the Sign-in Method page in the Firebase Authentication section to enable Email/Password sign-in and any other identity providers you want for your app.
(Optional) Prototype and test with Firebase Local Emulator Suite

Before talking about how your app authenticates users, let's introduce a set of tools you can use to prototype and test Authentication functionality: Firebase Local Emulator Suite. If you're deciding among authentication techniques and providers, trying out different data models with public and private data using Authentication and Firebase Security Rules, or prototyping sign-in UI designs, being able to work locally without deploying live services can be a great idea.

An Authentication emulator is part of the Local Emulator Suite, which enables your app to interact with emulated database content and config, as well as optionally your emulated project resources (functions, other databases, and security rules).

Using the Authentication emulator involves just a few steps:

    Adding a line of code to your app's test config to connect to the emulator.

    From the root of your local project directory, running firebase emulators:start.

    Using the Local Emulator Suite UI for interactive prototyping, or the Authentication emulator REST API for non-interactive testing.

    Call useAuthEmulator() to specify the emulator address and port:

    Future<void> main() async {
    WidgetsFlutterBinding.ensureInitialized();
    await Firebase.initializeApp();

    // Ideal time to initialize
    await FirebaseAuth.instance.useAuthEmulator('localhost', 9099);
    //...
    }

A detailed guide is available at Connect your app to the Authentication emulator. For more information, see the Local Emulator Suite introduction.

Now let's continue with how to authenticate users.
Check current auth state

Firebase Auth provides many methods and utilities for enabling you to integrate secure authentication into your new or existing Flutter application. In many cases, you will need to know about the authentication state of your user, such as whether they're logged in or logged out.

Firebase Auth enables you to subscribe in realtime to this state via a Stream. Once called, the stream provides an immediate event of the user's current authentication state, and then provides subsequent events whenever the authentication state changes.

There are three methods for listening to authentication state changes:
authStateChanges()

To subscribe to these changes, call the authStateChanges() method on your FirebaseAuth instance:

FirebaseAuth.instance
  .authStateChanges()
  .listen((User? user) {
    if (user == null) {
      print('User is currently signed out!');
    } else {
      print('User is signed in!');
    }
  });

Events are fired when the following occurs:

    Right after the listener has been registered.
    When a user is signed in.
    When the current user is signed out.

idTokenChanges()

To subscribe to these changes, call the idTokenChanges() method on your FirebaseAuth instance:

FirebaseAuth.instance
  .idTokenChanges()
  .listen((User? user) {
    if (user == null) {
      print('User is currently signed out!');
    } else {
      print('User is signed in!');
    }
  });

Events are fired when the following occurs:

    Right after the listener has been registered.
    When a user is signed in.
    When the current user is signed out.
    When there is a change in the current user's token.

If you set custom claims using the Firebase Admin SDK, you will only see this event fire when the following occurs:

    A user signs in or re-authenticates after the custom claims are modified. The ID token issued as a result will contain the latest claims.
    An existing user session gets its ID token refreshed after an older token expires.
    An ID token is force refreshed by calling FirebaseAuth.instance.currentUser.getIdTokenResult(true).

For further details, see Propagating custom claims to the client

userChanges()

To subscribe to these changes, call the userChanges() method on your FirebaseAuth instance:

FirebaseAuth.instance
  .userChanges()
  .listen((User? user) {
    if (user == null) {
      print('User is currently signed out!');
    } else {
      print('User is signed in!');
    }
  });

Events are fired when the following occurs:

    Right after the listener has been registered.
    When a user is signed in.
    When the current user is signed out.
    When there is a change in the current user's token.
    When the following methods provided by FirebaseAuth.instance.currentUser are called:
        reload()
        unlink()
        updateEmail()
        updatePassword()
        updatePhoneNumber()
        updateProfile()

idTokenChanges(), userChanges() & authStateChanges() will not fire if you update the User profile with the Firebase Admin SDK. You will have to force a reload using FirebaseAuth.instance.currentUser.reload() to retrieve the latest User profile.

idTokenChanges(), userChanges() & authStateChanges() will also not fire if you disable or delete the User with the Firebase Admin SDK or the Firebase console. You will have to force a reload using FirebaseAuth.instance.currentUser.reload(), which will cause a user-disabled or user-not-found exception that you can catch and handle in your app code.

Persisting authentication state

The Firebase SDKs for all platforms provide out of the box support for ensuring that your user's authentication state is persisted across app restarts or page reloads.

On native platforms such as Android & iOS, this behavior is not configurable and the user's authentication state will be persisted on device between app restarts. The user can clear the apps cached data using the device settings, which will wipe any existing state being stored.

On web platforms, the user's authentication state is stored in IndexedDB. You can change the persistence to store data in the local storage using Persistence.LOCAL. If required, you can change this default behavior to only persist authentication state for the current session, or not at all. To configure these settings, call the following method FirebaseAuth.instanceFor(app: Firebase.app(), persistence: Persistence.LOCAL);. You can still update the persistence for each Auth instance using setPersistence(Persistence.NONE).

// Disable persistence on web platforms. Must be called on initialization:
final auth = FirebaseAuth.instanceFor(app: Firebase.app(), persistence: Persistence.NONE);
// To change it after initialization, use `setPersistence()`:
await auth.setPersistence(Persistence.LOCAL);

Next Steps
Explore the guides on signing in and signing up users with the supported identity and authentication services.

Phone Authentication

Phone authentication allows users to sign in to Firebase using their phone as the authenticator. An SMS message is sent to the user (using the provided phone number) containing a unique code. Once the code has been authorized, the user is able to sign into Firebase.

    Phone numbers that end users provide for authentication will be sent and stored by Google to improve spam and abuse prevention across Google service, including to, but not limited to Firebase. Developers should ensure they have the appropriate end-user consent prior to using the Firebase Authentication phone number sign-in service.authentication

Firebase Phone Authentication is not supported in all countries. Please see their FAQs for more information.
Setup

Before starting with Phone Authentication, ensure you have followed these steps:

    Enable Phone as a Sign-In method in the Firebase console.
    Android: If you haven't already set your app's SHA-1 hash in the Firebase console, do so. See Authenticating Your Client for information about finding your app's SHA-1 hash.
    iOS: In Xcode, enable push notifications for your project & ensure your APNs authentication key is configured with Firebase Cloud Messaging (FCM). To view an in-depth explanation of this step, view the Firebase iOS Phone Auth documentation.
    Web: Ensure that you have added your applications domain on the Firebase console, under OAuth redirect domains.

Note; Phone number sign-in is only available for use on real devices and the web. To test your authentication flow on device emulators, please see Testing.
Usage

The Firebase Authentication SDK for Flutter provides two individual ways to sign a user in with their phone number. Native (e.g. Android & iOS) platforms provide different functionality to validating a phone number than the web, therefore two methods exist for each platform exclusively:

    Native Platform: verifyPhoneNumber.
    Web Platform: signInWithPhoneNumber.

Native: verifyPhoneNumber

On native platforms, the user's phone number must be first verified and then the user can either sign-in or link their account with a PhoneAuthCredential.

First you must prompt the user for their phone number. Once provided, call the verifyPhoneNumber() method:

await FirebaseAuth.instance.verifyPhoneNumber(
  phoneNumber: '+44 7123 123 456',
  verificationCompleted: (PhoneAuthCredential credential) {},
  verificationFailed: (FirebaseAuthException e) {},
  codeSent: (String verificationId, int? resendToken) {},
  codeAutoRetrievalTimeout: (String verificationId) {},
);

Note: Depending on your billing plan, you might be limited to a daily quota of SMS messages sent. See Firebase Auth Limits.

There are 4 separate callbacks that you must handle, each will determine how you update the application UI:

    verificationCompleted: Automatic handling of the SMS code on Android devices.
    verificationFailed: Handle failure events such as invalid phone numbers or whether the SMS quota has been exceeded.
    codeSent: Handle when a code has been sent to the device from Firebase, used to prompt users to enter the code.
    codeAutoRetrievalTimeout: Handle a timeout of when automatic SMS code handling fails.

verificationCompleted

This handler will only be called on Android devices which support automatic SMS code resolution.

When the SMS code is delivered to the device, Android will automatically verify the SMS code without requiring the user to manually input the code. If this event occurs, a PhoneAuthCredential is automatically provided which can be used to sign-in with or link the user's phone number.

FirebaseAuth auth = FirebaseAuth.instance;

await auth.verifyPhoneNumber(
  phoneNumber: '+44 7123 123 456',
  verificationCompleted: (PhoneAuthCredential credential) async {
    // ANDROID ONLY!

    // Sign the user in (or link) with the auto-generated credential
    await auth.signInWithCredential(credential);
  },
);

verificationFailed

If Firebase returns an error, for example for an incorrect phone number or if the SMS quota for the project has exceeded, a FirebaseAuthException will be sent to this handler. In this case, you would prompt your user something went wrong depending on the error code.

FirebaseAuth auth = FirebaseAuth.instance;

await auth.verifyPhoneNumber(
  phoneNumber: '+44 7123 123 456',
  verificationFailed: (FirebaseAuthException e) {
    if (e.code == 'invalid-phone-number') {
      print('The provided phone number is not valid.');
    }

    // Handle other errors
  },
);

codeSent

When Firebase sends an SMS code to the device, this handler is triggered with a verificationId and resendToken (A resendToken is only supported on Android devices, iOS devices will always return a null value).

Once triggered, it would be a good time to update your application UI to prompt the user to enter the SMS code they're expecting. Once the SMS code has been entered, you can combine the verification ID with the SMS code to create a new PhoneAuthCredential:

FirebaseAuth auth = FirebaseAuth.instance;

await auth.verifyPhoneNumber(
  phoneNumber: '+44 7123 123 456',
  codeSent: (String verificationId, int? resendToken) async {
    // Update the UI - wait for the user to enter the SMS code
    String smsCode = 'xxxx';

    // Create a PhoneAuthCredential with the code
    PhoneAuthCredential credential = PhoneAuthProvider.credential(verificationId: verificationId, smsCode: smsCode);

    // Sign the user in (or link) with the credential
    await auth.signInWithCredential(credential);
  },
);

By default, Firebase will not re-send a new SMS message if it has been recently sent. You can however override this behavior by re-calling the verifyPhoneNumber method with the resend token to the forceResendingToken argument. If successful, the SMS message will be resent.
codeAutoRetrievalTimeout

On Android devices which support automatic SMS code resolution, this handler will be called if the device has not automatically resolved an SMS message within a certain timeframe. Once the timeframe has passed, the device will no longer attempt to resolve any incoming messages.

By default, the device waits for 30 seconds however this can be customized with the timeout argument:

FirebaseAuth auth = FirebaseAuth.instance;

await auth.verifyPhoneNumber(
  phoneNumber: '+44 7123 123 456',
  timeout: const Duration(seconds: 60),
  codeAutoRetrievalTimeout: (String verificationId) {
    // Auto-resolution timed out...
  },
);

Web: signInWithPhoneNumber

On web platforms, users can sign-in by confirming they have access to a phone by entering the SMS code sent to the provided phone number. For added security and spam prevention, users are requested to prove they are human by completing a Google reCAPTCHA widget. Once confirmed, the SMS code will be sent.

The Firebase Authentication SDK for Flutter will manage the reCAPTCHA widget out of the box by default, however provides control over how it is displayed and configured if required. To get started, call the signInWithPhoneNumber method with the phone number.

FirebaseAuth auth = FirebaseAuth.instance;

// Wait for the user to complete the reCAPTCHA & for an SMS code to be sent.
ConfirmationResult confirmationResult = await auth.signInWithPhoneNumber('+44 7123 123 456');

Calling the method will first trigger the reCAPTCHA widget to display. The user must complete the test before an SMS code is sent. Once complete, you can then sign the user in by providing the SMS code to the confirm method on the resolved ConfirmationResult response:

UserCredential userCredential = await confirmationResult.confirm('123456');

Like other sign-in flows, a successful sign-in will trigger any authentication state listeners you have subscribed throughout your application.
reCAPTCHA Configuration

The reCAPTCHA widget is a fully managed flow which provides security to your web application.

The second argument of signInWithPhoneNumber accepts an optional RecaptchaVerifier instance which can be used to manage the widget. By default, the widget will render as an invisible widget when the sign-in flow is triggered. An "invisible" widget will appear as a full-page modal on-top of your application.

It is however possible to display an inline widget which the user has to explicitly press to verify themselves.

To add an inline widget, specify a DOM element ID to the container argument of the RecaptchaVerifier instance. The element must exist and be empty otherwise an error will be thrown. If no container argument is provided, the widget will be rendered as "invisible".

ConfirmationResult confirmationResult = await auth.signInWithPhoneNumber('+44 7123 123 456', RecaptchaVerifier(
  container: 'recaptcha',
  size: RecaptchaVerifierSize.compact,
  theme: RecaptchaVerifierTheme.dark,
));

You can optionally change the size and theme by customizing the size and theme arguments as shown above.

It is also possible to listen to events, such as whether the reCAPTCHA has been completed by the user, whether the reCAPTCHA has expired or an error was thrown:

RecaptchaVerifier(
  onSuccess: () => print('reCAPTCHA Completed!'),
  onError: (FirebaseAuthException error) => print(error),
  onExpired: () => print('reCAPTCHA Expired!'),
);

Testing

Firebase provides support for locally testing phone numbers:

    On the Firebase Console, select the "Phone" authentication provider and click on the "Phone numbers for testing" dropdown.
    Enter a new phone number (e.g. +44 7444 555666) and a test code (e.g. 123456).

If providing a test phone number to either the verifyPhoneNumber or signInWithPhoneNumber methods, no SMS will actually be sent. You can instead provide the test code directly to the PhoneAuthProvider or with signInWithPhoneNumbers confirmation result handler.
