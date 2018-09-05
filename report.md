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

The log in system will be protected Google reCAPTCHA to prevent brute force attacks to access into the accounts of administrators.

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

The minimum, average and maximum values of `Age` & `Reading` will be automatically generated. Furthermore, with each retrieval, the order of the data will be randomised to make it harder to re-identify each person through piecing different parts of the data.

Below is a sample of the original data:
![pre-anonymised data](https://github.com/IFS4205-2018-Sem1-Team1/design-report/raw/master/images/pre_anonymisation.png)

And the generated data:
![post-anonymised data](https://github.com/IFS4205-2018-Sem1-Team1/design-report/raw/master/images/post_anonymisation.png)

Notice that this data has 2-anonymity with respect to the attributes `Age`, `Gender`, `Location` and `Steps`, but not for the attribute `Disease`.

## Subsystem 4 (Secure Transfer)

## Subsystem 5 (Data Collection from Sensors)
Use Android's accelerometer to track movement activity and upload data to system.
