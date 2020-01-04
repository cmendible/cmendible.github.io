---
author: Carlos Mendible
categories:
- Azure
- dotNetCore
date: "2016-10-24T10:11:58Z"
description: Use .NET Core to Create Azure Blob Storage SAS Keys
image: /wp-content/uploads/2016/10/SAS.jpg
# tags: Blob BlobStorage SAS SharedAccessSignature
title: Use .NET Core to Create Azure Blob Storage SAS Keys
url: /2016/10/24/use-net-core-create-azure-blob-storage-sas-keys/
---
If you intend to use the <a href="https://www.nuget.org/packages/WindowsAzure.Storage/" target="_blank">WindowsAzure.Storage</a> library with .Net Core you'll hit a"compatibility wall" trying to make it work.

So how can we use .Net Core to Create Azure Blob Storage SAS Keys?

## 1. Add the System.Security.Cryptography.Algorithms dependency
---
Add the following dependency in your project.json file
    
``` json
    "System.Security.Cryptography.Algorithms": "4.2.0"
```
## 2. Create a StorageEntitySas class to hold the SAS key
---
The following class will hold de blob Uri and the SAS credentials.
    
``` csharp
    public struct StorageEntitySas
    {
        public string Credentials;
        public Uri BlobUri;
        public string Name;
    }
```

## 3. Create the following methods
---
I've created the GetSharedAccessSignature to return a write only SAS key for the blob. But you can easily change it to add more permissions.
    
``` csharp
        /// <summary>
        /// Creates a write only access SAS key for the given blob.
        /// </summary>
        /// 
        /// 
        /// 
        /// 
        /// 
        /// 
        /// <returns>SAS credentials for ther blob.</returns>    
        private StorageEntitySas GetSharedAccessSignature(
            string accountName,
            string blobContainer,
            string blobName,
            string key,
            DateTimeOffset sharedAccessStartTime,
            DateTimeOffset sharedAccessExpiryTime)
        {
            var canonicalNameFormat = $"/blob/{accountName}/{blobContainer}/{blobName}";
            var st = sharedAccessStartTime.UtcDateTime.ToString("yyyy-MM-ddTHH:mm:ssZ");
            var se = sharedAccessExpiryTime.UtcDateTime.ToString("yyyy-MM-ddTHH:mm:ssZ");
            var sasVersion = "2015-07-08";

            string stringToSign = string.Format("{0}\n{1}\n{2}\n{3}\n{4}\n{5}\n{6}\n{7}\n{8}\n{9}\n{10}\n{11}\n{12}", new object[]
            {
                "w", // Write only permission.
                st,
                se,
                canonicalNameFormat,
                string.Empty,
                string.Empty,
                string.Empty,
                sasVersion,
                string.Empty,
                string.Empty,
                string.Empty,
                string.Empty,
                string.Empty
            });

            var sas = GetHash(stringToSign, key);

            var credentials =
                $"?sv={sasVersion}&sr=b&sig={UrlEncoder.Default.Encode(sas)}&st={UrlEncoder.Default.Encode(st)}&se={UrlEncoder.Default.Encode(se)}&sp=w";

            return new StorageEntitySas
            {
                BlobUri = new Uri($"https://{accountName}.blob.core.windows.net/{blobContainer}/{blobName}"),
                Credentials = credentials,
                Name = blobName
            };
        }

        private string GetHash(string stringToSign, string key)
        {
            byte[] keyValue = Convert.FromBase64String(key);

            using (HMACSHA256 hmac = new HMACSHA256(keyValue))
            {
                return Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign)));
            }
        }
```

## 4. Call the GetSharedAccessSignature method.
---  

``` csharp
    var blobSas = GetSharedAccessSignature(accountName, blobContainer, blobName, storageKey, sharedAccessStartTime, sharedAccessExpiryTime);
```

Find a full working sample here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/cloud.design.patterns/valet.key/valet.key.server" target="_blank">https://github.com/cmendible/dotnetcore.samples/tree/master/cloud.design.patterns/valet.key/valet.key.server</a>

Hope it helps!!!