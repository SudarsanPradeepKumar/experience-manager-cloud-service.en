---
title: Adding an SSL Certificate - Managing SSL Certificates
description: Adding an SSL Certificate - Managing SSL Certificates
exl-id: 104b5119-4a8b-4c13-99c6-f866b3c173b2
---
# Adding an SSL Certificate {#adding-an-ssl-certificate}

>[!NOTE]
>AEM as a Cloud Service will only accept certificates that conform with OV (Organization Validation) or EV (Extended Validation) policy. DV (Domain Validation) policy will not be accepted. In addition, any certificate must be a X.509 TLS certificate from a trusted certification authority (CA) with a matching 2048-bit RSA private key. AEM as a Cloud Service will accept wildcard SSL certificates for a domain.

A Certificate takes a few days to provision and it is recommended that the certificate be provisioned even months in advance. Refer to [Getting an SSL Certificate](/help/implementing/cloud-manager/managing-ssl-certifications/get-ssl-certificate.md) for more details.

## Certificate Format {#certificate-format}

SSL files must be in PEM format in order to be installed on Cloud Manager. Common file extensions that are within the PEM format include `.pem,` .`crt`, `.cer`, and `.cert`. 

Follow the steps below to convert the format of your SSL files to PEM:

* Convert PFX to PEM

   `openssl pkcs12 -in certificate.pfx -out certificate.cer -nodes`

* Convert P7B to PEM

   `openssl pkcs7 -print_certs -in certificate.p7b -out certificate.cer`

* Convert DER to PEM

   `openssl x509 -inform der -in certificate.cer -out certificate.pem`

## Important Considerations {#important-considerations}

* A user must be in the Business Owner or Deployment Manager role in order to install an SSL certificate in Cloud Manager.

* At any given time, Cloud Manager will allow a maximum of 10 SSL certificates that can be associated with one or more environments across your Program, even if a certificate is expired. Cloud Manager UI will, however, allow up 50 SSL certificates to be installed in the program with this constraint. Typically a certificate can cover multiple domains (up to 100 SANs) so consider grouping multiple domains in the same certificate in order to stay within this limit.


## Adding a Certificate {#adding-a-cert}

Follow the steps below to add a certificate:

1. Login to Cloud Manager.
1. Navigate to **Environments** screen from **Overview** page.
1. Click on **SSL Certificates** from the left navigation menu. A table with details of any existing SSL certificates will be displayed on this screen.

   ![](/help/implementing/cloud-manager/assets/ssl/ssl-cert-1.png)

1. Click on **Add SSL Certificate** to open **Add SSL Certificate** dialog box.

     * Enter a name for your certificate in **Certificate Name**. This can be any name that helps you reference your certificate easily.
     * Paste the **Certificate**, **Private key** and **Certificate chain** into their respective fields. Use the paste icon to the right of the input box. 
     All three fields are not optional and must be included.

       ![](/help/implementing/cloud-manager/assets/ssl/ssl-cert-02.png)

  
       >[!NOTE]
       >Any errors detected will be displayed. You must address all errors before your certificate can be saved. Refer to the [Certificate Errors](#certificate-errors) to learn more about addressing common errors.

1. Click **Save** to submit your certificate. You will see it displayed as a new row in the table.

   ![](/help/implementing/cloud-manager/assets/ssl/ssl-cert-3.png)

## Certificate Errors {#certificate-errors}

### Certificate Policy {#certificate-policy}

If you see the error “Certificate policy must conform with EV or OV, and not DV policy.”, please check the policy of your certificate.

Normally certificate types are identified by the OID values embedded in policies. These OIDs are unique and hence converting a certificate to text form and searching for the OID will confirm the certificate as having a match.

You can view your certificate detail as follows.

```text
openssl x509 -in 9178c0f58cb8fccc.pem -text
certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            91:78:c0:f5:8c:b8:fc:cc
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, ST = Arizona, L = Scottsdale, O = "GoDaddy.com, Inc.", OU = http://certs.godaddy.com/repository/, CN = Go Daddy Secure Certificate Authority - G2
        Validity
            Not Before: Nov 10 22:55:36 2021 GMT
            Not After : Dec  6 15:35:06 2022 GMT
        Subject: C = US, ST = Colorado, L = Denver, O = Alexandra Alwin, CN = adobedigitalimpact.com
        Subject Public Key Info:
...
```

This tables provides identifying patterns.

|Pattern|Certificate Type|Acceptable|
|---|---|---|
|`2.23.140.1.2.1`|DV|No|
|`2.23.140.1.2.2`|OV|Yes|
|`2.23.140.1.2.3` and `TLS Web Server Authentication`|IV cert with permission to use for https|Yes|

`grep`ping for the patterns, you can confirm your certificate type.

```shell
# "EV Policy"
openssl x509 -in certificate.pem -text grep "Policy: 2.23.140.1.1" -B5

# "OV Policy"
openssl x509 -in certificate.pem -text grep "Policy: 2.23.140.1.2.2" -B5

# "DV Policy - Not Accepted"
openssl x509 -in certificate.pem -text grep "Policy: 2.23.140.1.2.1" -B5
```

### Correct Certificate Order {#correct-certificate-order}

The most common reason for a certificate deployment to fail is that the intermediate or chain certificates are not in the correct order. Specifically, intermediate certificate files must end with the root certificate or certificate most proximate to the root and be in a descending order from the `main/server` certificate to the root. 

You can determine the order of your intermediate files using the following command:

`openssl crl2pkcs7 -nocrl -certfile $CERT_FILE | openssl pkcs7 -print_certs -noout`

You can verify that the private key and `main/server` certificate match using the following commands:

`openssl x509 -noout -modulus -in certificate.pem | openssl md5`

`openssl rsa -noout -modulus -in ssl.key | openssl md5`

>[!NOTE]
>The output of these two commands must be exactly the same. If you cannot locate a matching private key to your `main/server` certificate, you will be required to re-key the certificate by generating a new CSR and/or requesting an updated certificate from your SSL vendor.

### Certificate Validity Dates {#certificate-validity-dates}

Cloud Manager expects the SSL certificate to be valid for at least 90 days into the future. You should check the validity of the certificate chain.
