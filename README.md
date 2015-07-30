# OpenPGP API library

The OpenPGP API provides methods to execute OpenPGP operations, such as sign, encrypt, decrypt, verify, and more without user interaction from background threads. This is done by connecting your client application to a remote service provided by [OpenKeychain](http://www.openkeychain.org) or other OpenPGP providers.

## License
While OpenKeychain itself is GPLv3+, the API library is licensed under Apache License v2.
Thus, you are allowed to also use it in closed source applications as long as you respect the [Apache License v2](https://github.com/open-keychain/openpgp-api/blob/master/LICENSE).

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

## Complete example
A complete working example is available in the [example project](https://github.com/open-keychain/openpgp-api/blob/master/example). The [``OpenPgpApiActivity.java``](https://github.com/open-keychain/openpgp-api/blob/master/example/src/main/java/org/openintents/openpgp/example/OpenPgpApiActivity.java) contains most relevant sourcecode.

## 1. Add the API library to your project

Add this to your build.gradle:

```
repositories {
    jcenter()
}

dependencies {
    compile 'org.sufficientlysecure:openpgp-api:7.0'
}
```

## 2. Understand the basic design of the OpenPGP API
The API is **not** designed around ``Intents`` which are started via ``startActivityForResult``. These Intent actions typically start an activity for user interaction, so they are not suitable for background tasks. Most API design decisions are explained at [the bottom of this wiki page](https://github.com/open-keychain/open-keychain/wiki/OpenPGP-API#internal-design-decisions).

We will go through the basic steps to understand how this API works, following this (greatly simplified) sequence diagram:
![](https://github.com/open-keychain/open-keychain/raw/master/Resources/docs/openpgp_api_1.jpg)

In this diagram the client app is depicted on the left side, the OpenPGP provider (in this case OpenKeychain) is depicted on the right.
The remote service is defined via the [AIDL](http://developer.android.com/guide/components/aidl.html) file [``IOpenPgpService``](https://github.com/open-keychain/openpgp-api/blob/master/openpgp-api/src/main/aidl/org/openintents/openpgp/IOpenPgpService.aidl).
It contains only one exposed method which can be invoked remotely:
```java
interface IOpenPgpService {
    Intent execute(in Intent data, in ParcelFileDescriptor input, in ParcelFileDescriptor output);
}
```
The interaction between the apps is done by binding from your client app to the remote service of OpenKeychain.
``OpenPgpServiceConnection`` is a helper class from the library to ease this step:
```java
OpenPgpServiceConnection mServiceConnection;

public void onCreate(Bundle savedInstance) {
    [...]
    mServiceConnection = new OpenPgpServiceConnection(this, "org.sufficientlysecure.keychain");
    mServiceConnection.bindToService();
}

public void onDestroy() {
    [...]
    if (mServiceConnection != null) {
        mServiceConnection.unbindFromService();
    }
}
```

Following the sequence diagram, these steps are executed:

1.  Define an ``Intent`` containing the actual PGP instructions which should be done, e.g.
    ```java
Intent data = new Intent();
data.setAction(OpenPgpApi.ACTION_ENCRYPT);
data.putExtra(OpenPgpApi.EXTRA_USER_IDS, new String[]{"dominik@dominikschuermann.de"});
data.putExtra(OpenPgpApi.EXTRA_REQUEST_ASCII_ARMOR, true);
    ```
    Define an ``InputStream`` currently holding the plaintext, and an ``OutputStream`` where you want the ciphertext to be written by OpenKeychain's remote service:
    ```java
InputStream is = new ByteArrayInputStream("Hello world!".getBytes("UTF-8"));
ByteArrayOutputStream os = new ByteArrayOutputStream();
    ```
    Using a helper class from the library, ``is`` and ``os`` are passed via ``ParcelFileDescriptors`` as ``input`` and ``output`` together with ``Intent data``, as depicted in the sequence diagram, from the client to the remote service.
    Programmatically, this can be done with:
    ```java
OpenPgpApi api = new OpenPgpApi(this, mServiceConnection.getService());
Intent result = api.executeApi(data, is, os);
    ```

2.  The PGP operation is executed by OpenKeychain and the produced ciphertext is written into ``os`` which can then be accessed by the client app.

3.  A result Intent is returned containing one of these result codes:
    * ``OpenPgpApi.RESULT_CODE_ERROR``
    * ``OpenPgpApi.RESULT_CODE_SUCCESS``
    * ``OpenPgpApi.RESULT_CODE_USER_INTERACTION_REQUIRED``

    If ``RESULT_CODE_USER_INTERACTION_REQUIRED`` is returned, an additional ``PendingIntent`` is returned to the client, which must be used to get user input required to process the request.
    A ``PendingIntent`` is executed with ``startIntentSenderForResult``, which starts a activity, originally belonging to OpenKeychain, on the [task stack](http://developer.android.com/guide/components/tasks-and-back-stack.html) of the client.
    Only if ``RESULT_CODE_SUCCESS`` is returned, ``os`` actually contains data.
    A nearly complete example looks like this:
    ```java
    switch (result.getIntExtra(OpenPgpApi.RESULT_CODE, OpenPgpApi.RESULT_CODE_ERROR)) {
        case OpenPgpApi.RESULT_CODE_SUCCESS: {
            try {
                Log.d(OpenPgpApi.TAG, "output: " + os.toString("UTF-8"));
            } catch (UnsupportedEncodingException e) {
                Log.e(Constants.TAG, "UnsupportedEncodingException", e);
            }

            if (result.hasExtra(OpenPgpApi.RESULT_SIGNATURE)) {
                OpenPgpSignatureResult sigResult
                        = result.getParcelableExtra(OpenPgpApi.RESULT_SIGNATURE);
                [...]
            }
            break;
        }
        case OpenPgpApi.RESULT_CODE_USER_INTERACTION_REQUIRED: {
            PendingIntent pi = result.getParcelableExtra(OpenPgpApi.RESULT_INTENT);
            try {
                startIntentSenderForResult(pi.getIntentSender(), 42, null, 0, 0, 0);
            } catch (IntentSender.SendIntentException e) {
                Log.e(Constants.TAG, "SendIntentException", e);
            }
            break;
        }
        case OpenPgpApi.RESULT_CODE_ERROR: {
            OpenPgpError error = result.getParcelableExtra(OpenPgpApi.RESULT_ERROR);
            [...]
            break;
        }
    }
    ```

4.  Results from a ``PendingIntent`` are returned in ``onActivityResult`` of the activity, which executed ``startIntentSenderForResult``.
    The returned ``Intent data`` in ``onActivityResult`` contains the original PGP operation definition and new values acquired from the user interaction.
    Thus, you can now execute the ``Intent`` again, like done in step 1.
    This time it should return with ``RESULT_CODE_SUCCESS`` because all required information has been obtained by the previous user interaction stored in this ``Intent``.
    ```java
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        [...]
        // try again after user interaction
        if (resultCode == RESULT_OK) {
            switch (requestCode) {
                case 42: {
                    encrypt(data); // defined like in step 1
                    break;
                }
            }
        }
    }
    ```


## 3. Tipps
*   ``api.executeApi(data, is, os);`` is a blocking call. If you want a convenient asynchronous call, use ``api.executeApiAsync(data, is, os, new MyCallback([... ]));``, where ``MyCallback`` is an private class implementing ``OpenPgpApi.IOpenPgpCallback``.
    See [``OpenPgpApiActivity.java``](https://github.com/open-keychain/openpgp-api/blob/master/example/src/main/java/org/openintents/openpgp/example/OpenPgpApiActivity.java) for an example.
*   Using

    ```java
    mServiceConnection = new OpenPgpServiceConnection(this, "org.sufficientlysecure.keychain");
    ```
    connects to OpenKeychain directly.
    If you want to let the user choose between OpenPGP providers, you can implement the [``OpenPgpAppPreference.java``](https://github.com/open-keychain/openpgp-api/tree/master/openpgp-api/src/main/java/org/openintents/openpgp/util/OpenPgpAppPreference.java) like done in the example app.

