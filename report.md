# Subsystem Design Report

## Subsystem 1 (MFA)

Multifactor authentication will apply to all users and will aim for a balance between confidentiality and usability.

### Services and tools to be used

MFA Tag Interfaces:
1. RFDuinoBLE API  
The RFduino device must include the #include <RFduinoBLE.h> command in order to make use of the RFduino BLE. 
1. Arduino Cryptography Libraries

Web App Interfaces:
1. GraphQL and PHP libraries for registration
1. Web Bluetooth API  
Access to Web Bluetooth API will be subject to permissions and it will only work in secure contexts. It only works with sites on HTTPS.  The connection to the device can only be called by a user action such as a click. 

Authentication protocol involved between the tag and the web app:  
Digital Signature

Algorithms to be used:
1. Ed25519 for digital signature - to be used for the tag from [Arduino Cryptography Libraries](https://rweather.github.io/arduinolibs/crypto.html) and for web app from [NaCL](https://github.com/dchest/tweetnacl-js#public-key-authenticated-encryption-box) and [Libsodium](https://github.com/jedisct1/libsodium.js)
1. SHA256 for cryptographic hashing

Rationale:  
A digital signature ensures authentication and non-repudiation. We cannot afford to use RSA on the rfduino board due to the lack of memory. In fact, ed25519 seems to have many [benefits](https://risan.io/upgrade-ssh-key-to-ed25519.html). 


##### Registration:
During the registration of accounts, the user selects the all the applicable roles (patient, therapist, researcher and/or admininstrator) and signs up with his NRIC number and password. The web app generates a unique salt for the user and adds it to his registered password. The web app generates a hash of the resultant value using SHA256 and stores both the user's hash and salt in the database of the web app.

After registration, administrators will be notified to approve of the registration and to add the user into the database. The 2FA tags will then be issued to the approved users. The private and public keys of each user are generated via the ED25519. Since the 2FA tags are issued by the administrators, the administrators could obtain the user's public key from the tag and store the web app's public key in the tag manually without the need for transmission of the keys. This eliminates the risk of malicious parties intercepting and giving incorrect public keys if the keys were to be transmitted instead.  

The 2FA tag contains:
1. The user's private key
1. The web app’s public key

The database of the web app contains: 
1. The user's NRIC number
1. The user's salt
1. The hash of the sum of the user's password and his salt 
1. The web app's private key 
1. The web app's public key 
1. The user's public key

##### After registration:
For 2FA Login into Web App: (both web app and 2FA tag need to be authenticated to be correct)
1. The user selects the role (which he had registered) to log in into and logs in with his log-in credentials: NRIC number and password.
1. The web app checks if the NRIC number is valid for the selected role.
1. If valid, the web app verifies the password by adding the salt associated with the user's NRIC number to the password, hashing the resultant value and comparing the computed hash with the one in the database associated with the user.
1. After successful verification, the web app requests to pair with a BLE device nearby (which is the user’s 2FA tag).  
1. To authenticate that the 2FA tag really belongs to the user logging in, we need to ensure that the 2FA tag has the private key of the patient. To do so, the web app first generates a random message and appends it to the number '1' (to distinguish from other request(s) such as the request for the validation of health records).
1. The web app then signs the pair with its private key using Ed25519, append the signature to the data prior to signing and sends over to the 2FA tag it has connected with.
1. Upon receiving the data with the signature, the 2FA tag decrypts the signature of the web app with the web app's public key using Ed25519. It compares the data with the decrypted signature to ensure integrity and authenticity of the data.
1. After successful verification, the 2FA tag hashes the random message generated using SHA256, signs the hash with the user's private key using Ed25519 and sends only the signature back to the web app.
1. Upon receiving the signature from the 2FA tag, the web app hashes random message it generated and decrypts the signature with the user's private key using Ed25519.
1. The web app compares the hash it obtained with the value it obtained from the decryption.
1. If both values match, the web app logs the user in. 


For validating health records: (both web app and tag need to be authenticated to be correct)
1. To implement the system of validation of patients' health records, patients have to sign their own health records. First, the web app pairs with a BLE device nearby (which is the patient’s tag) and hashes the health record to be signed using SHA256.
1. The web app appends the hash to the number '2' (to distinguish from other request(s) such as the request for the 2FA login) and signs it with its private key using Ed25519.
1. The web app appends the signature to the data ('2' and the hash value) and sends it over to the tag.
1. Upon receiving the data with the signature, the tag decrypts the signature of the web app with the web app's public key using Ed25519. It compares the data with the decrypted signature to ensure integrity and authenticity of the data.
1. After successful verification, the tag signs the hashed health record with the patient's private key using Ed25519 and sends only the signature back to the web app.
1. Upon receiving the signature from the tag, the web app decrypts the signature with the patient's public key using Ed25519 and compares the value obtained with the hash the web app generated in the first step.
1. If both match, the signature is authenticated and the web app stores the signature as an attribute of the health record in its database.

---

## Subsystems 2 to 4 Overview

Subsystems 2 to 4 will support the following functionalities with the following parameters:

1. Registration
    1. National Registration Identification Card (NRIC) number
    1. Name
    1. Email
    1. Phone
    1. Address
    1. Age
    1. Gender
    1. Role

1. Log In
    1. NRIC number
    1. Password
    1. Role

A sample POST query using GraphQL looks something like this:

```
{
  "query": "registration",
  "variables": { "ic": "someValue", ... }
}
```

The log in system will be protected with Google reCAPTCHA to prevent brute force attacks to access into the accounts of administrators.

### Security for Client & Server Communication

#### GraphQL

We will use GraphQL to facilitate communication between the Client & the Server. GraphQL is susceptible to broken Access Controls. GraphQL does not verify whether a user has the permissions to retrieve sensitive data such as the password of the user, etc. Therefore, we will be using Apache Shiro to perform user permissions authentication.

#### Apache Shiro
Apache Shiro will be used to perform user permissions **authentication** in the following steps:
1. Collect the subject’s principals and credentials
1. Submit the principals and credentials to an authentication system.
1. Perform either of the following: allow access or block access

#### JSON Web Token (JWT)
Once the user is authenticated, a JWT will be generated in the client side for **authorisation**. This JWT will be used along the channels between Client and Server, Server and Database. In addition, the JWT will be stored in a session storage under [HTML5 Web Storage](https://www.tutorialspoint.com/html5/html5_web_storage.htm). When the browser window is closed, the user will be automatically logged out. The JWT will be removed and becomes invalid.

If an incoming request contains no token, the request is denied from accessing any resources. If the request contains a token, the server side code will check if the information inside corresponds to an authorised user. If not, the request is denied. The JWT should be sent in an ‘Authorisation’ header using the ‘Bearer’ schema in the [OAuth protocol](https://help.salesforce.com/articleView?id=remoteaccess_oauth_jwt_flow.htm&type=5). Since a token, instead of a cookie, is sent in the ‘Authorisation header’, Cross-Origin Resource Sharing (CORS) will not be a potential area for exploit.

This is because unlike cookie-based authentication which is stateful, token-based authentication is stateless, hence the server does not keep a record of which users are logged in or which JWTs have been issued. Instead, every request to the server is accompanied by a token which the server uses to verify the authenticity of the request.

The JWT will be:
1. Signed with HMAC algorithm to prevent data tampering, thus preserving **integrity**
1. Sent via HTTPS to ensure **confidentiality** of the data in the token

In addition, using HTTPS as our only mode of transfer across channels will prevent any potential leaks from HTML5 Web Storage during transfers. It also serves as a more efficient method to ensure traffic is encrypted instead of having to deploy encryption algorithms when transferring over unsecured HTTP routes.

### Security for Server & Database Communication

We will be protecting our system by:
1. Using HTTPS to ensure confidentiality and integrity in data transfer.
1. Disallowing executables to be uploaded into the database. 
1. Sanitising user input. This helps to prevent SQL injections & XSS (especially if the input field is a custom type, such as JSON).
1. Using an anti-virus scanner to scan through image and video files that will be uploaded into the database.

---

## Subsystem 2 (Interface for Therapists & Patients)
This subsystem provides the web interface that will be used by Therapists, Patients and Administrators to access the Health Record System.

### Therapists Capabilities:
1. List all patients under their charge
1. Select and read patients' records only
1. Create new records
1. Edit their own created records
1. Print out reports

### Therapist's Interface
After logging in, therapists would be able to list all their patients under 'My Patient' tab.
![Therapist list all Patients](https://github.com/IFS4205-2018-Sem1-Team1/design-report/blob/subsystem-2/images/Therapist%20-%20List%20their%20patients.PNG?raw=true)

The therapist can then click on the patient's name to view the specific patient's records.
![therapist read Patient records](https://github.com/IFS4205-2018-Sem1-Team1/design-report/blob/subsystem-2/images/Therapist%20-%20Read%20Bob's%20Records.PNG?raw=true)

- On this page, the therapist would have an overview of all the records of that Patient that the therapist has been given permission to view.
- The therapist would only have read-only access to the records not owned by the therapist, and write access to records create by the therapist.
- 'Owner' tab indicates which therapist created that record.
- To retrieve the files to read the full details of the record, the therapist can click on the filename under the 'Document' tab.
- The therapist can also use this interface to 'Create' new records and notes, 'Edit' them, and 'Print' these records.

### Patients Capabilities:
1. Able to view but not change their medical records
1. Able to add/remove authorisation for therapist(s) to view any of their medical data
1. Able to add notes about their records
1. Print out reports

### Patient's Interface
After logging in, Patients would be able to view all their personal health records under 'My Records' tab.
![Patient personal records](https://github.com/IFS4205-2018-Sem1-Team1/design-report/blob/subsystem-2/images/Patient%20-%20Personal%20Records.PNG?raw=true)

- Patients would only have read-only access to these records.
- They can also view the detailed records by clicking on the filename under the 'Document' tab.
- Patients would be able to add records specific notes that their therapist would be able to read.
- 'Permission' tab indicates which therapist they have already allowed access to that specific record.
- Permission can be changed with the 'Add/Remove' button under the 'Change Permissions' tab.
- Patients would also have the ability to print their records

### Administrators Capabilities:
1. Add users to the system
1. Display logs of all transactions in the system

### Administrator's Interface
After logging in, Administrators would be able to add new users under the 'New Users' tab.
![Admin add New Users](https://github.com/IFS4205-2018-Sem1-Team1/design-report/blob/subsystem-2/images/Admin%20-%20Add%20New%20User.PNG?raw=true)

Administrators would also be able to generate monthly logs by choosing the right Year and Month under the 'Logs' tab.
![Admin get log](https://github.com/IFS4205-2018-Sem1-Team1/design-report/blob/subsystem-2/images/Admin%20-%20Get%20Log.PNG?raw=true)
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

### Overview

The other team will be given a special account (which is a type of user, such as `therapist` or `patient`) that can only perform one action: Upload their database to our database. 

### Interface

Upon logging into the special account, the other team will be redirected to a page whereby they can upload their database, image and video files separately. Our current database design is such that the image and video files are stored in a separate file system. The database stores the image and video file names. When a record is retrieved from the database, the server will also retrieve the corresponding image and video files based on their names.

The data can be in tab-separated value (tsv), with the following values:
1. Type
1. Subtype
1. Title
1. Date_time
1. Owner_ic
1. Signature
1. Content

Where the file names will be stored in `Content`.

### Security Issues

The upload stream will be restricted to the use of HTTPS so that traffic towards our database is encrypted and not susceptible to sniffing from an external party, thus preserving **confidentiality**. In addition, the data will be digitally signed using the HMAC algorithm embedded within HTTPS during upload. The digital signature can then be checked at the receiving end of the upload channel to detect whether the message has been deliberately modified, thus preserving **integrity**.

## Subsystem 5 (Data Collection from Sensors)
2 sets of upload data: 
1. Video
1. Phone accelerometer (Android's accelerometer)

Assumptions made:  
The tag must be near the patient when the patient is trying to upload the data up to the database. The MFA tag will have to be nearby so that the web app can ensure that the patient is who he says he is (refer to Subsystem 1).

Interfaces:
1. HTTP POST method  
The enctype = “multipart/form-datavalue” is required for uploading files in forms.
1. [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to post the data to the url we specified, which will be the server code. 
1. FileUploadController for Java Spring Boot

Uploading process:
1. The patient selects the type of record (Reading, Image, Time series data, Movie and Document) to upload on the web page and fills in the form provided by the web app.
1. For images and movies, only certain extension types are allowed. For instance, only mp4 and avi files are allowed for movies. The patient is then required by the web page to upload the image or movie.
1. The client does a check on the file type of the image or movie uploaded by checking the its extension. If valid, the uploading will be successful and the patient is allowed to click the submit button.
1. The patient clicks the submit button and is prompted by the web page to have his tag near the device with the web app.
1. When the patient brings his tag near the device with the web app, the tag will pair with the web app and sign the uploaded image or movie (refer to the part "For validating health records" of Subsystem 1).
1. After verifying that the image or movie had been signed by the patient, the form along with the signature will be submitted to the server through HTTPS (to ensure confidentiality of the health record).
1. The web server then checks the file type of the uploaded image or movie again and scans it for virus using an Anti-virus software.
1. If valid and virus-free, the web server stores the image or movie in its file system and stores the link to it in the file system as the "content" attribute of the health record and other details in the form (including the signature) in the database.
1. For records other than images and movies, the "content" field of the form is required to be filled in instead (no files will be uploaded). The "content" field will be signed by the tag in a similar way and will correspond directly to the "content" attribute of the health record after being transferred to the database.


