# Subsystem Design Report

## Subsystem 1 (MFA)

MFA Tag Interfaces:
1. RFDuino BLE API
The RFduino device must include the #include <RFduinoBLE.h> command in order to make use of the RFduino BLE. 
1. Arduino Cryptography Libraries

Web App Interface:
1. Web Bluetooth API 
Access to Web Bluetooth API will be subject to permissions and it will only work in secure contexts. It only works with sites on HTTPS.  The connection to the device can only be called by a  user action like a click for example. 

Communication protocol between the two devices:
Digital Signature + Diffie Hellman Key Exchange + Symmetric Encryption

Algorithms to be used:
1. Ed25519 for digital signature - to be used for the tag from [Arduino Cryptography Libraries](https://rweather.github.io/arduinolibs/crypto.html) and for web app from [NaCL](https://github.com/dchest/tweetnacl-js#public-key-authenticated-encryption-box) and [Libsodium](https://github.com/jedisct1/libsodium.js)
2. Curve25519 for getting shared key - to be used for web app from [here](https://github.com/indutny/elliptic)
3. AES256 for encryption 

Rationale: 
A digital signature ensures integrity, authentication and non-repudiation. We cannot afford to use RSA on the rfduino board due to the lack of RAM. In fact, ed25519 seems to have many [benefits](https://risan.io/upgrade-ssh-key-to-ed25519.html).  


##### Assumptions to be made before any communication is made:
Private and public keys that are stored initially are generated via the ED25519.

The database contains:
1. A private key linked to the user

The MFA tag contains:
1. A private key linked to the user
1. The web app’s public key

The web app contains: 
1. Its own private key 
2. Its own public key 


For 2FA Access into Web App:
1. User logs in with the correct log in credentials - username and password. 
1. After that, the web app will request to pair with a BLE device nearby - which is the user’s 2FA token.  
1. The web app would then attain the private key attributed to the user from the database and create a hash of it. 
1. The web app would then use its own private key to sign it via ed25519. (Digital signature)
1.  The web app and the MFA tag will then get a secret shared key using Diffie Helman Key Exchange. 
1. The web app will then use AES with the shared key to encrypt the digital signature and send it over. 
1. The MFA Tag would decrypt it.
1.  The MFA Tag would use the web app’s public key to decrypt to get the hash. It can then check if the hash of its private key is the same as the hash sent to it. If it is, then the web app is said to be authenticated because only an authentic web app would have the private key of the user. 
1. Next, to authenticate that the MFA tag really belongs to the user logging in, the MFA tag will use its private key to sign a hash of the web app’s public key. 
1. MFA tag would then use AES with the shared key to encrypt the digital signature and send it over. 
1. The web app would then decrypt and use the user’s public key to attain the hash value. It would then see if it matches the web app’s public key hash. If it does, it means that the MFA tag has the private key in it and belongs to the user. 
1. Thus, the web server can log the user in. 

For validating data:
1. The web app and MFA tag acquire a secret shared key via the ECDH using the same curve - curve25519. 
1. The eb app will create a digital signature for the hash of the data, together with the hash of the data, encrypt it via AES using the shared key and send it to the MFA tag. 
The MFA tag checks if after decrypting it with the web app’s public key, the hash of the data match. 
1. After authenticating the web app, the MFA tag will use its private key to sign a hash of the data, encrypt it and send to the web app.
1. The web app will ensure that the MFA tag is authentic and then store the data in the file server. 
---

Subsystems 2 to 4 will support the following functionalities with the following parameters:

1. Registration
    1. National Registration Identification Card (NRIC)
    1. Name
    1. Email
    1. Phone
    1. Address
    1. Age
    1. Role

1. Log In
    1. NRIC
    1. Password
    1. Role

A sample POST query looks something like this:

```
{
  "query": "registration",
  "variables": { "ic": "someValue", ... }
}
```

The log in system will be protected with Google reCAPTCHA to prevent brute force attacks to access into the accounts of administrators.

We will use GraphQL to facilitate communication between the Client & the Server. GraphQL is susceptible to:
1. SQL injections & XSS (especially if the input field is a custom type, such as JSON). Therefore, we will need to sanitise user input.
1. Broken Access Controls. GraphQL does not verify whether a user has the permissions to retrieve sensitive data such as the password of the user, etc. Therefore, we will be using Apache Shiro to perform user permissions authentication.

We will also be protecting our system by:
1. Using HTTPS to ensure confidentiality in data transfer between the Client & the Server.
1. Disallowing executables to be uploaded into the database. 
1. Using an anti-virus scanner to scan through image and video files that will be uploaded into the database.

---

## Subsystem 2 (Interface for Therapists & Patients)
Therapists:
1. Able to list patients under their charge
1. Select patients' records to view but not change
1. Create new records
1. Edit their own created records
1. Print out reports

Patients:
1. Able to view but not change their medical data
1. Able to add/remove authorisation for therapist(s) to view any of their medical data

In addition, admin users are able to:
1. Add users to the system
1. Display logs of all transactions in the system

## Subsystem 3 (Interface for Researchers & Anyone)
This subsystem will support the functionality of retrieving anonymous data (implemented through k-anonymity), which can be filtered by:
1. Location
1. Subtype
1. Age
1. Gender

A sample GET query looks something like this:

```
{
  anonymised_records( "location": "someValue", "subtype": "number of steps per day", ... ) {
    location
    subtype
    gender
    location
    reading
    disease
  }
}
```

The minimum, average and maximum values of `Age` & `Reading` will be automatically generated. Furthermore, with each retrieval, the order of the data will be randomised to make it harder to re-identify each person through piecing different parts of the data.

Below is a sample of the original data:
![pre-anonymised data](https://github.com/IFS4205-2018-Sem1-Team1/design-report/raw/master/images/pre_anonymisation.png)

And the generated data:
![post-anonymised data](https://github.com/IFS4205-2018-Sem1-Team1/design-report/raw/master/images/post_anonymisation.png)

Notice that this data has 2-anonymity with respect to the attributes `Age`, `Gender`, `Location` and `Steps`, but not for the attribute `Disease`.

## Subsystem 4 (Secure Transfer)

## Subsystem 5 (Data Collection from Sensors)
Use Android's accelerometer to track movement activity and upload data to system.
