# Subsystem Design Report

## Subsystem 1 (MFA)

---

## Subsystems 2 to 4 Overview

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
The tag must be near the patient when the patient is trying to upload the data up to the database. 

Interfaces:
1. HTTP POST method  
The enctype = “multipart/form-datavalue” is required for uploading files in forms.
1. [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to post the data to the url we specified, which will be the server code. 
1. FileUploadController for Java Spring Boot

The MFA tag will have to be nearby so that the web app can ensure that the patient is who he/she says he/she is (refer to Subsystem 1). 
After validating that the data belongs to the patient because of the tag, the web app will send the file to the server code. 

