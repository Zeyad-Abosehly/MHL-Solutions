
### Objective

- Successfully execute remote code through the exploitation of a vulnerability in a third-party library.

# Static Analysis

#### AndroidManifest.xml

The manifest reveals only one exported activity, `MainActivity`, which declares intent filters for `VIEW` actions with schemes `file`, `http`, `https`, and MIME type `application/yaml`.  

![[Pasted image 20260829113456.png|800]]


The application requests the following permissions:

- INTERNET
- READ_EXTERNAL_STORAGE`
- WRITE_EXTERNAL_STORAGE
- MANAGE_EXTERNAL_STORAGE

![[Pasted image 20260829113226.png|800]]

#### MainActivity Code Flow

- `onCreate()` sets up the UI, configures `EditText`, and calls `setButtonListeners()` and `handleIntent()`.    

![[Pasted image 20260829114151.png|800]]
- `handleIntent()` retrieves the incoming `Intent`, checks for `ACTION_VIEW`, and if a `Uri` is present, calls `CopyUtil.copyFileFromUri(data)` to copy the file locally, then passes the resulting `Uri` to `loadYaml()`.
![[Pasted image 20260829114045.png|800]]

this method do something interesting 
![[Pasted image 20260829114436.png|800]]

```java
Object deserializedData = yaml.load(inputStream);  
String serializedData = yaml.dump(deserializedData);
```

- These two lines were the first I'd seen, and after some research, I found that they represent a known security vulnerability called **Insecure Sequence Decryption** . The use of `new Yaml(options)` without a `SafeConstructor` (or a restricted `Constructor`) and directly loading attacker-controlled YAML is a textbook example of CVE-2022-1471.

> **CVE-2022-1471**: SnakeYAML’s default `Constructor` does not restrict types that can be instantiated during deserialization. An attacker can supply malicious YAML to achieve remote code execution. [Read more](https://infosecwriteups.com/%EF%B8%8F-inside-the-160-comment-fight-to-fix-snakeyamls-rce-default-1a20c5ca4d4c)


# exploitation 

- **Open the app** on the emulator. You will see a text area and two buttons (Save and Load).
- **Write the malicious YAML payload** in the text area:

```yaml
!!java.io.FileOutputStream ["/data/data/com.mobilehackinglab.configeditor/files/pwn.txt"]
```
This payload forces SnakeYAML to instantiate `java.io.FileOutputStream` with a path inside the app’s private storage.

![[Screenshot_1787996709.png|200]]

- tap the **Load** button. Select the previously saved file. The app reads the file and passes its URI to `loadYaml()`.
- **Observe the result**: The app attempts to deserialize the YAML. The `FileOutputStream` constructor runs, creating the file. Then `yaml.dump()` serializes the instantiated object and displays it in the text area. You will this message :

![[Screenshot_1787996731.png|200]]

`The `fd` (file descriptor) value `58` indicates the file was opened (or created) – it’s a real file descriptor assigned by the OS.`

- **Verify the file existence** using `adb`:
![[Pasted image 20260829125011.png|800]]

The file `pwn.txt` is present in the app’s internal storage.

# Impact

Successful exploitation of this vulnerability enables an attacker to instantiate arbitrary classes, which can lead to remote code execution. Possible consequences include:

- Running commands with the app’s privileges.
- Reading or modifying private application data.
- Abusing storage permissions to access user files.

--- 
# Mitigation

1. **Upgrade SnakeYAML** to version 1.33 or later, which uses `SafeConstructor` by default.
2. **Explicitly use `SafeConstructor`**:

```java
Yaml yaml = new Yaml(new SafeConstructor(new LoaderOptions()));

```
3. **Avoid deserializing untrusted YAML** if the app only needs to display the raw content; instead, read the stream as plain text.    
4. **Restrict URI schemes** – the activity should only accept `content://` URIs from trusted providers, not arbitrary `file` or `http` links.
5. **Validate and sanitize** any YAML input, and consider using a whitelist of allowed classes if custom deserialization is required.


---

# Conclusion

The application is vulnerable to CVE-2022-1471, enabling arbitrary class instantiation and potentially remote code execution. The proof-of-concept successfully created a file in the app’s private storage using a malicious YAML payload delivered through the app’s own save/load functionality. Immediate remediation is recommended.