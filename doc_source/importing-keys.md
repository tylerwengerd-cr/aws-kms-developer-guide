# Importing key material in AWS KMS keys<a name="importing-keys"></a>

You can create a [AWS KMS keys](concepts.md#kms_keys) \(KMS key\) with key material that you supply\. 

A KMS key is a logical representation of an encryption key\. The metadata for a KMS key includes the ID of [key material](concepts.md#key-material) used to encrypt and decrypt data\. When you [create a KMS key](create-keys.md), by default, AWS KMS generates the key material for that KMS key\. But you can create a KMS key without key material and then import your own key material into that KMS key, a feature often known as "bring your own key" \(BYOK\)\.

![\[Image NOT FOUND\]](http://docs.aws.amazon.com/kms/latest/developerguide/images/import-key.png)

**Note**  
AWS KMS does not support decrypting any AWS KMS ciphertext outside of AWS KMS, even if the ciphertext was encrypted under a KMS key with imported key material\. AWS KMS does not publish the ciphertext format this task requires, and the format might change without notice\.

Imported key material is supported only for [symmetric encryption KMS keys](concepts.md#symmetric-cmks) in AWS KMS key stores, including [multi\-Region symmetric encryption KMS keys](multi-region-keys-import.md)\. It is not supported on [asymmetric KMS keys](symmetric-asymmetric.md#asymmetric-cmks), [HMAC KMS keys](hmac.md), or KMS keys in [custom key stores](custom-key-store-overview.md)\.

When you use imported key material, you remain responsible for the key material while allowing AWS KMS to use a copy of it\. You might choose to do this for one or more of the following reasons:
+ To prove that you generated the key material using a source of entropy that meets your requirements\.
+ To use key material from your own infrastructure with AWS services, and to use AWS KMS to manage the lifecycle of that key material within AWS\.
+ To set an expiration time for the key material in AWS and to [manually delete it](#importing-keys-delete-key-material), but to also make it available again in the future\. In contrast, [scheduling key deletion](deleting-keys.md#deleting-keys-how-it-works) requires a waiting period of 7 to 30 days, after which you cannot recover the deleted KMS key\.
+ To own the original copy of the key material, and to keep it outside of AWS for additional durability and disaster recovery during the complete lifecycle of the key material\.

You can monitor the use and management of a KMS key with imported key material\. AWS KMS records an entry in your AWS CloudTrail log when you [create the KMS key](ct-createkey.md), [download the public key and import token](ct-getparametersforimport.md), and [import the key material](ct-importkeymaterial.md)\. AWS KMS also records an entry when you [manually delete imported key material](#importing-keys-delete-key-material) or when AWS KMS [deletes expired key material](ct-deleteexpiredkeymaterial.md)\.

For information about important differences between KMS keys with imported key material and those with key material generated by AWS KMS, see [About imported key material](#importing-keys-considerations)\.

**Regions**

Imported key material is supported in all AWS Regions that AWS KMS supports\. The requirements for imported key material are different in China Regions\. For details, see [Importing key material step 3: Encrypt the key material](importing-keys-encrypt-key-material.md)

**Topics**
+ [About imported key material](#importing-keys-considerations)
+ [Permissions for importing key material](#importing-keys-permissions)
+ [How to import key material](#importing-keys-overview)
+ [How to reimport key material](#reimport-key-material)
+ [How to identify KMS keys with imported key material](#identify-imported-keys)
+ [How to create an expiration alarm](#imported-key-material-expiration-alarm)
+ [How to delete imported key material](#importing-keys-delete-key-material)

## About imported key material<a name="importing-keys-considerations"></a>

Before you decide to import key material into AWS KMS, you should understand the following characteristics of imported key material\.

**You generate the key material**  
You are responsible for generating the key material using a source of randomness that meets your security requirements\. The key material you import must be a 256\-bit symmetric encryption key, except in China Regions, where it must be a 128\-bit symmetric encryption key\.

**You can delete the key material**  
You can delete imported key material from a KMS key, immediately rendering the KMS key unusable\. Also, when you import key material into a KMS key, you can determine whether the key expires and [set its expiration date](importing-keys-create-cmk.md)\. When the expiration date arrives, AWS KMS [deletes the key material](#importing-keys-delete-key-material)\. Without key material, the KMS key cannot be used in any cryptographic operation\. To restore the key, you must reimport the same key material into the key\. 

**Can't change the key material**  
When you import key material into a KMS key, the KMS key is permanently associated with that key material\. You can [reimport the same key material](#reimport-key-material), but you cannot import different key material into that KMS key\. Also, you cannot [enable automatic key rotation](rotate-keys.md) for a KMS key with imported key material\. However, you can [manually rotate a KMS key](rotate-keys.md#rotate-keys-manually) with imported key material\. 

**Can't change the key material origin**  
KMS keys designed for imported key material have an [origin](concepts.md#key-origin) value of `EXTERNAL` that cannot be changed\. You cannot convert a KMS key for imported key material to use key material from any other source, including AWS KMS\.

**Can't decrypt with any other KMS key**  
When you encrypt data under a KMS key, the ciphertext is permanently associated with the KMS key and its key material\. It cannot be decrypted with any other KMS key, including a different KMS key with the same key material\. This is a security feature of KMS keys\.  
The only exception is [multi\-Region keys](multi-region-keys-overview.md), which are designed to be interoperable\. For details, see [Why aren't all KMS keys with imported key material interoperable?](multi-region-keys-import.md#mrk-import-why)\.

**No portability or escrow features**  
The symmetric ciphertexts that AWS KMS produces are not portable\. AWS KMS does not publish the symmetric ciphertext format that portability requires, and the format might change without notice\.   
+ AWS KMS cannot decrypt symmetric ciphertexts that you encrypt outside of AWS, even if you use key material that you have imported\. 
+ AWS KMS does not support decrypting any AWS KMS symmetric ciphertext outside of AWS KMS, even if the ciphertext was encrypted under a KMS key with imported key material\. 
Also, you cannot use any AWS tools, such as the [AWS Encryption SDK](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/) or [Amazon S3 client\-side encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingClientSideEncryption.html), to decrypt AWS KMS symmetric ciphertexts\.  
As a result, you cannot use keys with imported key material to support key escrow arrangements where an authorized third party with conditional access to key material can decrypt certain ciphertexts outside of AWS KMS\. To support key escrow, use the [AWS Encryption SDK](https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/java-example-code.html#java-example-multiple-providers) to encrypt your message under a key that is independent of AWS KMS\.

**You're responsible for availability and durability**  
You are responsible for the key material's overall availability and durability\. AWS KMS is designed to keep imported key material highly available\. But AWS KMS does not maintain the durability of imported key material at the same level as key material that AWS KMS generates\. To restore imported key material that has been deleted from a KMS key, you must retain a copy of the key material in a system that you control\. Then, you can reimport it into the KMS key\.  
This difference is meaningful in the following cases:  
+ When you [set an expiration time](importing-keys-import-key-material.md) for your imported key material, AWS KMS deletes the key material after it expires\. AWS KMS does not delete the KMS key or its metadata\. You can [create a Amazon CloudWatch alarm](#imported-key-material-expiration-alarm) that notifies you when imported key material is approaching its expiration date\.

  You cannot delete key material that AWS KMS generates for a KMS key and you cannot set AWS KMS key material to expire, although you can [rotate it](rotate-keys.md)\.
+ When you [manually delete imported key material](#importing-keys-delete-key-material), AWS KMS deletes the key material but does not delete the KMS key or its metadata\. In contrast, [scheduling key deletion](deleting-keys.md#deleting-keys-how-it-works) requires a waiting period of 7 to 30 days, after which AWS KMS permanently deletes the KMS key, its metadata, and its key material\.
+ In the unlikely event of certain region\-wide failures that affect AWS KMS \(such as a total loss of power\), AWS KMS cannot automatically restore your imported key material\. However, AWS KMS can restore the KMS key and the metadata\.

## Permissions for importing key material<a name="importing-keys-permissions"></a>

To create and manage KMS keys with imported key material, the user needs permission for the operations in this process\. You can provide the `kms:GetParametersForImport`, `kms:ImportKeyMaterial`, and `kms:DeleteImportedKeyMaterial` permissions in the key policy when you create the KMS key\. In the AWS KMS console, these permissions are added automatically for key administrators when you create a key with an **External** key material origin\.

To create KMS keys with imported key material, the principal needs the following permissions\.
+ [kms:CreateKey](customer-managed-policies.md#iam-policy-example-create-key) \(IAM policy\)
  + To limit this permission to KMS keys with imported key material, use the [kms:KeyOrigin](conditions-kms.md#conditions-kms-key-origin) policy condition with a value of `EXTERNAL`\.

    ```
    {
      "Sid": "CreateKMSKeysWithoutKeyMaterial",
      "Effect": "Allow",
      "Resource": "*",
      "Action": "kms:CreateKey",
      "Condition": {
        "StringEquals": {
          "kms:KeyOrigin": "EXTERNAL"
        }
      }
    }
    ```
+ [kms:GetParametersForImport](https://docs.aws.amazon.com/kms/latest/APIReference/API_GetParametersForImport.html) \(Key policy or IAM policy\)
  + To limit this permission to requests that use a particular wrapping algorithm and wrapping key spec, use the [kms:WrappingAlgorithm](conditions-kms.md#conditions-kms-wrapping-algorithm) and [kms:WrappingKeySpec](conditions-kms.md#conditions-kms-wrapping-key-spec) policy conditions\. 
+ [kms:ImportKeyMaterial](https://docs.aws.amazon.com/kms/latest/APIReference/API_ImportKeyMaterial.html) \(Key policy or IAM policy\)
  + To allow or prohibit key material that expires and control the expiration date, use the [kms:ExpirationModel](conditions-kms.md#conditions-kms-expiration-model) and [kms:ValidTo](conditions-kms.md#conditions-kms-valid-to) policy conditions\.

To reimport imported key material, the principal needs the [kms:GetParametersForImport](https://docs.aws.amazon.com/kms/latest/APIReference/API_GetParametersForImport.html) and [kms:ImportKeyMaterial](https://docs.aws.amazon.com/kms/latest/APIReference/API_ImportKeyMaterial.html) permissions\.

To delete imported key material, the principal needs [kms:DeleteImportedKeyMaterial](https://docs.aws.amazon.com/kms/latest/APIReference/API_DeleteImportedKeyMaterial.html) permission\.

For example, to give the example `KMSAdminRole` permission to manage all aspects of a KMS key with imported key material, include a key policy statement like the following one in the key policy of the KMS key\.

```
{
  "Sid": "Manage KMS keys with imported key material",
  "Effect": "Allow",
  "Resource": "*",
  "Principal": {
    "AWS": "arn:aws:iam::111122223333:role/KMSAdminRole"
  },
  "Action": [
    "kms:GetParametersForImport",
    "kms:ImportKeyMaterial",
    "kms:DeleteImportedKeyMaterial"
  ]  
}
```

## How to import key material<a name="importing-keys-overview"></a>

The following overview explains how to import your key material into AWS KMS\. For more details about each step in the process, see the corresponding topic\.

1. [Create a symmetric encryption KMS key with no key material](importing-keys-create-cmk.md) – The key spec must be `SYMMETRIC_DEFAULT` and the origin must be `EXTERNAL`\. A key origin of `EXTERNAL` indicates that the key is designed for imported key material and prevents AWS KMS from generating key material for the KMS key\. In a later step you will import your own key material into this KMS key\.

1. [Download the public key and import token](importing-keys-get-public-key-and-token.md) – After completing step 1, download a public key and an import token\. These items protect your key material while it's imported to AWS KMS\.

1. [Encrypt the key material](importing-keys-encrypt-key-material.md) – Use the public key that you downloaded in step 2 to encrypt the key material that you created on your own system\.

1. [Import the key material](importing-keys-import-key-material.md) – Upload the encrypted key material that you created in step 3 and the import token that you downloaded in step 2\.

   When the import operation completes successfully, the key state of the KMS key changes from `PendingImport` to `Enabled`\. You can now use the KMS key in cryptographic operations\.

AWS KMS records an entry in your AWS CloudTrail log when you [create the KMS key](ct-createkey.md), [download the public key and import token](ct-getparametersforimport.md), and [import the key material](ct-importkeymaterial.md)\. AWS KMS also records an entry when you delete imported key material or when AWS KMS [deletes expired key material](ct-deleteexpiredkeymaterial.md)\. 

## How to reimport key material<a name="reimport-key-material"></a>

If you manage a KMS key with imported key material, you might need to reimport the key material\. You might reimport key material to replace expired or deleted key material\. You might also reimport key material to change the expiration model or expiration date of the key material\.

You must reimport the same key material that was originally imported into the KMS key\. You cannot import different key material into a KMS key\. Also, AWS KMS cannot create key material for a KMS key that is created without key material\.

To reimport key material, use the same procedure that you used to [import the key material](#importing-keys-overview) the first time, with the following exceptions\.
+ Use an existing KMS key, instead of creating a new KMS key\. You can skip [Step 1](importing-keys-create-cmk.md) of the import procedure\.
+ If the KMS key has imported key material, you must [delete the existing imported key material](#importing-keys-delete-key-material) before you reimport the key material\.
+ When you reimport key material, you can change the expiration model and expiration date\. 

Each time you import key material to a KMS key, you need to [download and use a new wrapping key and import token](importing-keys-get-public-key-and-token.md) for the KMS key\. The wrapping procedure does not affect the content of the key material, so you can use different wrapping keys \(and different import tokens\) to import the same key material\.

## How to identify KMS keys with imported key material<a name="identify-imported-keys"></a>

When you create a KMS key with no key material, the value of the [`Origin`](concepts.md#key-origin) property of the KMS key is `EXTERNAL`, and it cannot be changed\. Unlike the [key state](key-state.md), the `Origin` value doesn't depend on the presence or absence of key material\. 

You can use the `EXTERNAL` origin value to identify KMS keys designed for imported key material\. You can find the key origin in the AWS KMS console or by using the [DescribeKey](https://docs.aws.amazon.com/kms/latest/APIReference/API_DescribeKey.html) operation\. You can also view the properties of the key material, such as whether and when it expires by using the console or the APIs\.

### To identify KMS keys with imported key material \(console\)<a name="identify-imported-keys-console"></a>

1. Open the AWS KMS console at [https://console\.aws\.amazon\.com/kms](https://console.aws.amazon.com/kms)\.

1. To change the AWS Region, use the Region selector in the upper\-right corner of the page\.

1. Use either of the following techniques to view the `Origin` property of your KMS keys\.
   + To add an **Origin** column to your KMS key table, in the upper right corner, choose the **Settings** icon\. Choose **Origin** and choose **Confirm**\. The **Origin** column makes it easy to identify KMS keys with an `EXTERNAL` origin property value\.
   + To find the value of the `Origin` property of a particular KMS key, choose the key ID or alias of the KMS key\. Then choose the **Cryptographic configuration** tab\. The tabs are below the **General configuration** section\.

1. To view detailed information about the key material, choose the **Key material** tab\. This tab appears on the detail page only for KMS keys with imported key material\.

### To identify KMS keys with imported key material \(AWS KMS API\)<a name="identify-imported-keys-api"></a>

Use the [DescribeKey](https://docs.aws.amazon.com/kms/latest/APIReference/API_DescribeKey.html) operation\. The response includes the `Origin` property of the KMS key, the expiration model, and the expiration date, as shown in the following example\.

```
$  aws kms describe-key --key-id 1234abcd-12ab-34cd-56ef-1234567890ab
{
    "KeyMetadata": {
        "KeyId": "1234abcd-12ab-34cd-56ef-1234567890ab",
        "Origin": "EXTERNAL",
        "ExpirationModel": "KEY_MATERIAL_EXPIRES"
        "ValidTo": 1568894400.0,
        "Arn": "arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab",
        "AWSAccountId": "111122223333",
        "CreationDate": 1568289600.0,
        "Enabled": false,
        "MultiRegion": false,
        "Description": "",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "PendingImport",
        "KeyManager": "CUSTOMER",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
        "EncryptionAlgorithms": [
            "SYMMETRIC_DEFAULT"
        ]
    }
}
```

## Creating a CloudWatch alarm for expiration of imported key material<a name="imported-key-material-expiration-alarm"></a>

You can create a CloudWatch alarm that notifies you when the imported key material in a KMS key is approaching its expiration time\. For example, the alarm can notify you when the time to expire is less than 30 days away\.

When you [import key material into a KMS key](#importing-keys), you can optionally specify a date and time when the key material expires\. When the key material expires, AWS KMS deletes the key material and the KMS key becomes unusable\. To use the KMS key again, you must [reimport the key material](#reimport-key-material)\. However, if you reimport the key material before it expires, you can avoid disrupting processes that use that KMS key\.

This alarm uses the [`SecondsUntilKeyMaterialExpires` metric](monitoring-cloudwatch.md#key-material-expiration-metric) that AWS KMS publishes to CloudWatch for KMS keys with imported key material that expires\. Each alarm uses this metric to monitor the imported key material for a particular KMS key\. You cannot create a single alarm for all KMS keys with expiring key material or an alarm for KMS keys that you might create in the future\.

**Requirements**

The following resources are required for a CloudWatch alarm that monitors the expiration of imported key material\.
+ A KMS key with imported key material that expires\. For help, see [How to identify KMS keys with imported key material](#identify-imported-keys)\. 
+ An Amazon SNS topic\. For details, see [Creating an Amazon SNS topic](https://docs.aws.amazon.com/sns/latest/dg/sns-create-topic.html) in the *Amazon CloudWatch User Guide*\.

**Create the alarm**

Follow the instructions in [Create a CloudWatch alarm based on a static threshold](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ConsoleAlarms.html) using the following required values\. For other fields, accept the default values and provide names as requested\.


| Field | Value | 
| --- | --- | 
| Select metric |  Choose **KMS**, then choose **Per\-Key Metrics**\. Choose the row with the KMS key and the `SecondsUntilKeyMaterialExpires` metric\. Then choose **Select metric**\. The **Metrics** list displays the `SecondsUntilKeyMaterialExpires` metric only for KMS keys with imported key material that expires\. If you don't have KMS keys with these properties in the account and Region, this list is empty\.  | 
| Statistic | Minimum | 
| Period | 1 minute | 
| Threshold type | Static | 
| Whenever \.\.\. | Whenever metric\-name is Greater than 1 | 

## Deleting imported key material<a name="importing-keys-delete-key-material"></a>

You can delete the imported key material from a KMS key at any time\. Also, when imported key material with an expiration date expires, AWS KMS deletes the key material\. In either case, AWS KMS deletes the key material immediately, the [key state](key-state.md) of the KMS key changes to *pending import*, and the KMS key can't be used in any cryptographic operations\. 

However, these actions do not delete the KMS key\. To use the KMS key again, you must [reimport the same key material](#reimport-key-material) into the KMS key\. In contrast, deleting a KMS key is irreversible\. If you [schedule key deletion](deleting-keys.md#deleting-keys-how-it-works) and the required waiting period expires, AWS KMS deletes the key material and all metadata associated with the KMS key\.

To delete key material, you can use the AWS Management Console or the AWS KMS API\. You can use the API directly by making HTTP requests, or by using an[AWS SDK](https://aws.amazon.com/tools/#sdk), the [AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/) \(AWS CLI\), or [AWS Tools for PowerShell](https://docs.aws.amazon.com/powershell/latest/userguide/)\.

AWS KMS records an entry in your AWS CloudTrail log when you delete imported key material and when [AWS KMS deletes expired key material](ct-deleteexpiredkeymaterial.md)\.

**Topics**
+ [How deleting key material affects AWS services](#importing-keys-delete-key-material-services)
+ [Delete key material \(console\)](#importing-keys-delete-key-material-console)
+ [Delete key material \(AWS KMS API\)](#importing-keys-delete-key-material-api)

### How deleting key material affects AWS services<a name="importing-keys-delete-key-material-services"></a>

When you delete key material, the KMS key with no key material becomes unusable right away \(subject to eventual consistency\)\. However, resources encrypted with [data keys](concepts.md#data-keys) protected by the KMS key are not affected until the the KMS key is used again, such as to decrypt the data key\. This issue affects AWS services, many of which use data keys to protect your resources\. For details, see [How unusable KMS keys affect data keys](concepts.md#unusable-kms-keys)\.

### Delete key material \(console\)<a name="importing-keys-delete-key-material-console"></a>

You can use the AWS Management Console to delete key material\.

1. Sign in to the AWS Management Console and open the AWS Key Management Service \(AWS KMS\) console at [https://console\.aws\.amazon\.com/kms](https://console.aws.amazon.com/kms)\.

1. To change the AWS Region, use the Region selector in the upper\-right corner of the page\.

1. In the navigation pane, choose **Customer managed keys**\.

1. Do one of the following:
   + Select the check box for a KMS key with imported key material\. Choose **Key actions**, **Delete key material**\.
   + Choose the alias or key ID of a KMS key with imported key material\. Choose the **Key material** tab and then choose **Delete key material**\.

1. Confirm that you want to delete the key material and then choose **Delete key material**\. The KMS key's status, which corresponds to its [key state](key-state.md), changes to **Pending import**\.

### Delete key material \(AWS KMS API\)<a name="importing-keys-delete-key-material-api"></a>

To use the [AWS KMS API](https://docs.aws.amazon.com/kms/latest/APIReference/) to delete key material, send a [DeleteImportedKeyMaterial](https://docs.aws.amazon.com/kms/latest/APIReference/API_DeleteImportedKeyMaterial.html) request\. The following example shows how to do this with the [AWS CLI](https://aws.amazon.com/cli/)\.

Replace `1234abcd-12ab-34cd-56ef-1234567890ab` with the key ID of the KMS key whose key material you want to delete\. You can use the KMS key's key ID or ARN but you cannot use an alias for this operation\.

```
$ aws kms delete-imported-key-material --key-id 1234abcd-12ab-34cd-56ef-1234567890ab
```