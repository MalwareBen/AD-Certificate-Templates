# AD-Certificate-Templates
Attack AD Certificate Templates and Mitigate


## Summary
- [Active Directory Certificate Service](https://github.com/MalwareBen/AD-Certificate-Templates#active-directory-certificate-services-ad-cs)
  - [Features](https://github.com/MalwareBen/AD-Certificate-Templates#features)  
- [AD CS Attack Vector](https://github.com/MalwareBen/AD-Certificate-Templates#ad-cs-attack-vector)
  - [The Weakness](https://github.com/MalwareBen/AD-Certificate-Templates#the-weakness)
- [The Attack](https://github.com/MalwareBen/AD-Certificate-Templates#attack)
  - [Enumerate User Groups](https://github.com/MalwareBen/AD-Certificate-Templates#enumerate-user-groups) 
  - [Enumerate Certificate Template](https://github.com/MalwareBen/AD-Certificate-Templates#enumerate-certificate-templates)
    - [Paramaters #1 - Relevant Permissions](https://github.com/MalwareBen/AD-Certificate-Templates#parameter-1---relevant-permissions)
    - [Parameters #2 - Client Authentication](https://github.com/MalwareBen/AD-Certificate-Templates#parameter-2---client-authentication)
    - [Parameters #3 - Subject Alternative Name](https://github.com/MalwareBen/AD-Certificate-Templates#parameter-3---subject-alternative-name-san)
- [Generate Malicious Templates](https://github.com/MalwareBen/AD-Certificate-Templates#generating-a-malicious-template)
- [Impersonate User](https://github.com/MalwareBen/AD-Certificate-Templates#impersonate-user-through-certificate)



## Active Directory Certificate Services (AD CS)

AD CS is Microsoft's Public Key Infrastructure implementation. Since Active Directory (AD) provides a level of trust in an organisation, it can be used as a Certificate Authority (CA) to prove and delegate trust.

### Features
AD CS can be used for:
- Creating and verifying digital signatures
- Client authentication
- Encrypting file system

![image](https://user-images.githubusercontent.com/88256245/168492603-af3e8b8b-2069-485c-baee-d57fcfcf99e0.png)

## AD CS Attack Vector
The most dangerous AD CS attack vector, is that certificates can survive credential rotation, thus providing persistent credential theft.

### The Weakness
Since AD CS is such a privileged function, it normally runs on selected domain controllers. Organisations tend to be too large to have an administrator create and distribute each certificate manually. This is where certificate templates come in.

Admininistrators of AD CS can create several templates that can allow any user with the relevant permissions to request a certificate themselves. These templates have parameters that say which user can request the certificate and what is required.

Combinations of these following parameters can be incredibly toxic and be abused for privilege escalation and persistent access:
- A template where we have the relevant permissions to request the certificate or where we have an account with those permissions
- A template that allows client authentication, meaning we can use it for Kerberos authentication
- A template that allow us to alter the subject alternative name (SAN)

### Attack
The following section will give explanation and commands to attack certificate templates on AD CS

### Enumerate User Groups
> net user ***username*** /domain > usergroups.txt

This command will lsit both Local and Global groups that user belongs to.

### Enumerate Certificate Templates
> certutil.exe -v -template > cert_templates.txt

Each template is denoted by Template[X] where X is the number. This can be used if you want to split the output from a single template. Now let's focus on the specific toxic parameter set that we are looking for in certificate templates.

### Parameter #1 - Relevant Permissions
We must have permissions to generate a certificate request in order for this exploit to work. Essentially, look for a template where our user has either the **Allow Enroll** or **Allow Full Control** permission.

Grep for all Allow Enroll keywords and review the output to see if any of the returned groups match groups that your user belongs to.
  
Two groups will be fairly common for certificates:
- Domain Users
- Domain Computers 

### Parameter #2 - Client Authentication
  The next step is to ensure that the certificate has the **Client Authentication** EKU. This EKU means that the certificate can be used for Kerberos authentication. There are other ways to exploit certificates.
  
  The certificate will be granted, given that we are the authenticated user on the machine requesting the certificate.
  
### Parameter #3 - Subject Alternative Name (SAN)
  Finally, veirfy that the template allows us, the certificate client, to specify the Subject Alternative Name (SAN). The SAN is usually something like the URL of the website that we are looking to encrypt. However, we have the ability to control the SAN, which leverages the certificate to generate a Kerberos Ticket for any AD account of our choosing.
  
  To find these templates, grep for the **CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT=1** property flag.
  
## Generating a malicious template
Windows Start Menu > mmc > File > Add/Remove Snap-in > Certificates > Add > Now expand the Certificates option > Right-click on Personal > Select All Tasks > Clieck on Request New Certificate > Select Next twice > Click the "More information is required to enroll this certificate".

  ![image](https://user-images.githubusercontent.com/88256245/168494586-3bfed49d-75f8-4931-abb6-f4a2c92b4b5b.png)
  
  Now add properties > Enroll > Now export the certificate > All Tasks > Export... > Export w/ Private Key > Select filename and export certificate
  
## Impersonate User Through Certificate
  Finally, impersonate a user to perform:
  - Use the certificate to request a Kerberos Ticket Granting Ticket (TGT)
  - Load the Kerberos TGT

  ### Rubeus
  > Rubeus.exe asktgt /user:***username*** /enctype:aes256 /certificate:***path to cert*** /password:***cert file pass*** /outfile:***name of file to write*** /domain:***domain name*** /dc:***IP***
  
  > Rubeus.exe changepw /ticket:***path to ticket*** /new:***new pass for user*** /dc:***domain*** /targetuser:***domain***\***username targeted***
  
  > runas /user:***domain***\***username*** cmd.exe
  
  
