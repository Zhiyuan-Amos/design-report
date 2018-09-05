# Subsystem Design Report

## Subsystem 1 (MFA)

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

As described above, Apache Shiro will be used to perform user permissions authentication in the following steps:
1.	Collect the subject’s principals and credentials
2.	Submit the principals and credentials to an authentication system.
3.	Allow access, retry authentication, or block access

The authentication system is represented in Shiro by security-specific decentralised autonomous organisations (DAO), that are referred to as [Realms](https://shiro.apache.org/realm.html). A Realm is a component that can access application-specific security data such as users, roles, and permissions. The Realm translates this application-specific data into a format that Shiro understands so Shiro can in turn provide a single easy-to-understand Subject programming API.

Once the user is authenticated, a JWT will be generated for authorisation. This JWT will be used along the channels between Client and Server, Server and Database. In addition, the JWT will be stored in a session storage under HTML5 Web Storage. When the browser window is closed, the user will be automatically logged out. The JWT will be removed and becomes invalid.
If an incoming request contains no token, the request is denied from accessing any resources. If the request contains a token, the code will check if it is valid and if any of the checks fails, the request is denied. The JWT should be sent in an ‘Authorisation’ header using the ‘Bearer’ schema from JWT library. Since the token is sent in the ‘Authorisation header’, Cross-Origin Resource Sharing (CORS) will not be an issue to exploit as cookies are not used.

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
