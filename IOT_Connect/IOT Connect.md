


## Objective

The goal of this lab is to exploit a vulnerable Broadcast Receiver in the **IOT Connect** Android application. By abusing the receiver, we can activate the **Master Switch**, allowing us to control all connected devices—a capability that should not be available to a guest user.

---

### Step 1: Explore the Application

Extract the APK from the provided ZIP archive and install it on the Android device.

After launching the application, click on **Master Switch**. Since our objective is to enable this switch, we first observe how the application behaves.

When entering any three-digit number, the application displays the following message:

> **"Sorry, the master switch can't be controlled by guests."**

As shown below, this message becomes our starting point for the analysis.
![](res/pic1)


---

### Step 2: Static Analysis with JADX

Open the APK in JADX and search for the following string:

> **"Sorry, the master switch can't be controlled by guests."**

The search leads us to a method inside the `MasterSwitchActivity` class .
 ![[Pasted image 20260728014557.png|800]]

After reviewing the method, we notice that once all validation checks pass, the application creates an `Intent` with the following properties:

- **Action:** `MASTER_ON`
- **Extra:** `key` (the PIN entered by the user)

The next question is:

> **Which component receives this broadcast?**

---

### Step 3: Locate the Broadcast Receiver

Open the `AndroidManifest.xml` file and search for `<receiver>` entries.

We find a Broadcast Receiverthat:

- Listens for the `MASTER_ON` action.
- Is marked as **exported**, meaning any application installed on the device can send broadcasts to it.

At first glance, it appears that simply sending a broadcast with the correct action should trigger the receiver. However, after testing this approach, nothing happens.

This indicates that the receiver performs additional validation before executing its privileged functionality.

---

### Step 4: Analyze `onReceive()`

Search for the `onReceive()` method.

Although JADX returns several results, only one belongs to the application's package .

![[Pasted image 20260728020932.png|800]]

Inside the method, we find two important validation checks:

1. Verify that the received intent action is `MASTER_ON`.
2. Validate the supplied key using the `check_key()` method.

The first requirement is straightforward—we simply need to send a broadcast with the correct action.

The second requirement is determining the correct key accepted by `check_key()`.

---

### Step 5: Recover the Correct Key

![[Pasted image 20260728021254.png|800]]

Analysis of the `check_key()` method shows that it decrypts a fixed Base64-encoded string using AES. The encryption key is generated directly from the integer supplied by the user.

Since the input is limited to three digits, I asked chat-gpt to write simple Python brute-force script to test every possible value from `000` to `999`.

``` python
from base64 import b64decode
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import unpad

ciphertext = b64decode("OSnaALIWUkpOziVAMycaZQ==")

for key in range(1000):
    key_bytes = str(key).encode().ljust(16, b"\x00")

    try:
        cipher = AES.new(key_bytes, AES.MODE_ECB)
        plaintext = unpad(cipher.decrypt(ciphertext), 16).decode()

        if plaintext == "master_on":
            print("Correct key:", key)
            break
    except Exception:
        pass
```

Running the script reveals that the correct key is: `345`

---

### Step 6: Send the Broadcast

Now that we know both the required action and the correct key, we can send the broadcast directly using ADB:

```
adb shell am broadcast -a MASTER_ON --ei key 345
```

After executing the command, the application displays the following message:

> **"All devices are turned on."**

![[Pasted image 20260728031542.png|800]]

This confirms that we successfully exploited the exported Broadcast Receiver by sending an unauthorized broadcast that activated the Master Switch.

---

# Impact

Successful exploitation allows an attacker to:

- Bypass the application's intended access control mechanism.
- Trigger privileged functionality reserved for authorized users.
- Enable the Master Switch and control all connected IoT devices.
- Send malicious broadcasts from any third-party application or through ADB without requiring user interaction.
- Potentially compromise the availability and integrity of connected IoT devices if similar insecure broadcast receivers exist in production applications.


---

# Mitigation

To prevent this vulnerability:

- Mark the Broadcast Receiver as `android:exported="false"` unless external access is explicitly required.
- If the receiver must remain exported, protect it with a custom permission so that only trusted applications can communicate with it.
- Avoid relying on hardcoded secrets or client-side validation for authorization.
- Perform authorization and authentication checks on the receiving side before executing privileged actions.
- Avoid using predictable or easily brute-forced secrets such as short numeric keys.

---

# Conclusion

This vulnerability exists because the application exposes an exported Broadcast Receiver without enforcing proper authorization checks. Although the receiver validates a key before executing privileged functionality, the key can be recovered through static analysis and offline brute forcing.

Once the correct key is identified, any application—or an attacker using ADB—can send the expected broadcast and trigger privileged functionality without interacting with the application's user interface. This demonstrates the importance of properly securing exported Android components and enforcing server-side or receiver-side authorization rather than relying on client-side secrets.








---




# Alternative Solution Using Frida

Instead of recovering the correct key, this lab can also be solved by modifying the application's behavior at runtime using **Frida**.

After completing the same analysis described in **Steps 1–2**, we know that the application only sends the privileged broadcast after two checks succeed:

- The current user must **not** be a guest.
- The supplied PIN must pass `Checker.check_key()`.

Rather than bypassing these checks externally, we can hook the relevant methods and force them to return values that satisfy the application's logic.

---

##  Hook the Security Checks

Create the following Frida script:

```js
Java.perform(function () {

    var User = Java.use("com.mobilehackinglab.iotconnect.ROOM.User");

    User.isGuest.implementation = function () {
        return 3;
    };

    var Checker = Java.use("Checker");

    Checker.check_key.implementation = function (key) {
        console.log("[+] check_key(" + key + ") -> true");
        return true;
    };

});
```
`you can get the code easly from jadx with right click on the method then chose copy as frida snipped`
This script modifies two methods:

- `User.isGuest()` always returns a non-guest value.
- `Checker.check_key()` always returns `true`, regardless of the supplied PIN.

As a result, the application's authorization logic is completely bypassed.

---

## Step 2: Inject the Script

Start the application with Frida:

```shell
frida -U -l file-name.js "IOT connect"
```

then, open the application and enter **any three-digit PIN**.

Because both security checks have been bypassed, the application executes its normal code path and sends the internal `MASTER_ON` broadcast.

The result is identical to the first solution:

> **"All devices are turned on."**
