# Subsystem Design Report

## Subsystem 1 (MFA)

MFA Tag Interfaces:
1. RFDuinoBLE API  
The RFduino device must include the #include <RFduinoBLE.h> command in order to make use of the RFduino BLE. 
1. Arduino Cryptography Libraries

Web App Interface:
1. Web Bluetooth API  
Access to Web Bluetooth API will be subject to permissions and it will only work in secure contexts. It only works with sites on HTTPS.  The connection to the device can only be called by a user action such as a click. 

Communication protocol involved between the tag and the web app:  
Digital Signature

Algorithms to be used:
1. Ed25519 for digital signature - to be used for the tag from [Arduino Cryptography Libraries](https://rweather.github.io/arduinolibs/crypto.html) and for web app from [NaCL](https://github.com/dchest/tweetnacl-js#public-key-authenticated-encryption-box) and [Libsodium](https://github.com/jedisct1/libsodium.js)
1. SHA256 for hashing

Rationale:  
A digital signature ensures authentication and non-repudiation. We cannot afford to use RSA on the rfduino board due to the lack of RAM. In fact, ed25519 seems to have many [benefits](https://risan.io/upgrade-ssh-key-to-ed25519.html). 


##### Assumptions to be made before any communication is made:
Private and public keys that are stored initially are generated via the ED25519. Since the 2FA tags are issued by the us, we could obtain the patient's public key from the tag and store the web app's public key in the tag manually without the need for transmission of the keys. This eliminates the risk of malicious parties intercepting and giving incorrect public keys if the keys were to be transmitted instead. 

Before the communication begins:  

The 2FA tag contains:
1. The patient's private key
1. The web app’s public key

The database of the web app contains: 
1. The web app's private key 
1. The web app's public key 
1. The patient's public key


For 2FA Login into Web App: (both web app and 2FA tag need to be authenticated to be correct)
1. User logs in with the correct log-in credentials: username and password. 
1. After that, the web app will request to pair with a BLE device nearby (which is the user’s 2FA tag).  
1. To authenticate that the 2FA tag really belongs to the user logging in, we need to ensure that the 2FA tag has the private key of the user. To do so, the web app first generates a random message and appends it to the number '1' (to distinguish from other request(s) such as the request for the validation of health records).
1. The web app then signs the pair with its private key using Ed25519, append the signature to the data prior to signing and sends over to the 2FA tag it has connected with.
1. Upon receiving the data with the signature, the 2FA tag decrypts the signature of the web app with the web app's public key using Ed25519. It compares the data with the decrypted signature to ensure integrity and authenticity of the data.
1. After successful verification, the 2FA tag hashes the random message generated using SHA256, signs the hash with the patient's private key using Ed25519 and sends only the signature back to the web app.
1. Upon receiving the signature from the 2FA tag, the web app hashes random message it generated and decrypts the signature with the patient's private key using Ed25519.
1. The web app compares the hash it obtained with the value it obtained from the decryption.
1. If both values match, the web app logs the patient in. 


For validating data: (both web app and 2FA tag need to be authenticated to be correct)
1. The web app and MFA tag acquire a secret shared key via the ECDH using the same curve - curve25519. 
1. The web app will create a digital signature for the hash of the data, together with the hash of the data, encrypt it via AES using the shared key and send it to the MFA tag. 
1. The MFA tag checks if after decrypting it with the web app’s public key, the hash of the data match. 
1. After authenticating the web app, the MFA tag will use its private key to sign a hash of the data, encrypt it and send to the web app.
1. The web app will ensure that the MFA tag is authentic and then store the data in the file server. 
---

Subsystems 2 to 4 will support the following functionalities with the following parameters:

1. Registration
    1. National Registration Identification Card (NRIC) number
    1. Name
    1. Email
    1. Phone
    1. Address
    1. Age
    1. Role

1. Log In
    1. NRIC number
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
