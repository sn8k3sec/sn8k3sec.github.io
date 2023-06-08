---
layout: default
title: An Intro to Azure Blob Storage and Abusing the Reader Role to Access Data
parent: Blogs
nav_order: 1
---
# An Intro to Azure Blob Storage and Abusing the Reader Role to Access Data

**By: Costa Papadatos**

### Update:

I have released a forked version of the popular BlobHunter tool that can use the "Reader" role to identify misconfigured containers! https://github.com/Sol1dSn8keS3cur1ty/BlobHunter-Lite

## Intro

In this post, I am hoping to achieve two things:

1. Give an introduction to Azure Storage Accounts, and more specifically, the Blob Storage service. I hope to describe how this service works and the somewhat confusing configuration settings that control anonymous access. 
2. Detail how you may be able to abuse the "Reader" role to read sensitive information stored within Blob Storage.

Before I dive into Storage Accounts and Blob Storage, let's dig into the Azure Resource Manager(AzureRM) RBAC role, Reader.

## The "Reader" Role

According to Microsoft, this role allows you to 

>"View all resources, but does not all you to make any changes"

That is simple enough; it's a read-only role that let's you, well, read things.

While this role allows you see Azure resources, it usually doesn't let you read the data inside of them. 

An example would be: You can see that an Azure Storage Account exists, but you can't read the contents in it's containers. Or can you ;) 

This is because AzureRM roles seperate it's permissions into "Control-Plane" and "Data-Plane" operations. 

An easy way to think of this is: 

* **Control-plane** operations describe what the role can do to an Azure resource
* **Data-plane** operations describe what you can do to *the data* inside an Azure resource. 

Let's take a look at the role definition for Reader.

```
{
  "assignableScopes": [
    "/"
  ],
  "description": "View all resources, but does not allow you to make any changes.",
  "id": "/subscriptions/{subscriptionId}/providers/Microsoft.Authorization/roleDefinitions/acdd72a7-3385-48ef-bd42-f606fba81ae7",
  "name": "acdd72a7-3385-48ef-bd42-f606fba81ae7",
  "permissions": [
    {
      "actions": [
        "*/read"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
  "roleName": "Reader",
  "roleType": "BuiltInRole",
  "type": "Microsoft.Authorization/roleDefinitions"
}
```

You can see under "actions", this role has "**\*/read**" permissions. The "actions" section of the above role definition is describing only the **Control-Plane** permission. This means, as it relates to what this role *can do* to Azure resources: It can only read them.

You can also see under "dataActions", nothing is listed. This is by design. As I described earlier, the Reader role can see Azure resources (Control-Plane=actions) but it can't see the data inside of those resources (Data-Plane=dataActions).

Now that we have a good understanding of what the "Reader" role can do, let's take a look at the Storage Account service.

## Azure Storage Accounts

Azure storage accounts are Microsoft's equivalent of Amazon Web Services (AWS) Simple Storage Solution (S3).

According to Microsoft, 

>"*An Azure storage account contains all of your Azure Storage data objects, including blobs, file shares, queues, tables, and disks. The storage account provides a unique namespace for your Azure Storage data that's accessible from anywhere in the world over HTTP or HTTPS.**"

The Azure storage account service consists of several sub-services such as:

* Blob Storage 
* Files Service
* Table Service
* Queue Service

For the purposes of this blog I will only focus on "Blob Storage", as it is the most common and very similar to an S3 bucket.

### Blob Storage

The Blob Storage service consists of “containers” and "blobs". A good way to think of this is a "blob" is the file, and the "container" is the folder it is stored in. Blob storage is accessed through HTTP/HTTPS, and each blob within a container has a unique URL you can use to download the file.

Whenever you create a storage account, a DNS record is automatically created. For Azure Blob Storage, this DNS record uses the following format: **\<storage-account-name\>.blob.core.windows.net**. This format is publicly known, which makes DNS brute-forcing to identify valid storage account names possible. More on this topic later.

The storage account name must be unique as its DNS record is publicly facing. This means no two organizations can have the same storage account name. 

I went ahead and created a storage account for the purpose of this blog with the name "d3mostorageaccount".

![](/assets/images/Storage Account.png)

You can see below several DNS records are automatically created, including the one we are interested in: 

"blob": "<https://d3mostorageaccount.blob.core.windows.net>"

```
$ az storage account list
[
  {
    ...snip
    "primaryEndpoints": {
      "blob": "https://d3mostorageaccount.blob.core.windows.net/",
      "dfs": "https://d3mostorageaccount.dfs.core.windows.net/",
      "file": "https://d3mostorageaccount.file.core.windows.net/",
      "internetEndpoints": null,
      "microsoftEndpoints": null,
      "queue": "https://d3mostorageaccount.queue.core.windows.net/",
      "table": "https://d3mostorageaccount.table.core.windows.net/",
      "web": "https://d3mostorageaccount.z13.web.core.windows.net/"
    ...snip
    }
```

### Containers

In order to store data within blob storage, you must first create a "Container." An Azure container is a way to organize and manage blobs in Azure Storage. A container acts as a top-level folder for blobs, and can be thought of as a logical grouping for blobs. Each storage account can have multiple containers, and each container can have multiple blobs. Blobs in a container can be accessed via a URL that includes the container name, such as:

<https://[storage account name].blob.core.windows.net/[container name]/[blob name>. 

You can set permissions for the container and its blobs, and can use Azure's built-in tools to copy and move blobs within and between containers.

Container names do not have to be unique as they are tied to a unique storage account. 

I've created a new container called "democontainer".

```
$ az storage container list --account-name d3mostorageaccount
[
  {
    ...snip
    "name": "democontainer",

	...snip
      "publicAccess": "container"
    }
  }
]
```

You can see under the "d3mostorageaccount" we now have a container named "democontainer". You can also see it's public access level is set to "container". We will do a deep dive on the various public access levels later in this blog.

### Blobs

Now that we have a storage account and container created, we can finally upload files (blobs). A blob is a file of any type and size that can be stored as an object in Azure Storage. 

I've uploaded a simple text file to serve as an example.

```
$ az storage blob list --container-name democontainer --account-name d3mostorageaccount
[
  {
	...snip
    "name": "example.txt",
	...snip
]
```

### Full URL Structure

With a storage account, container, and blob created, let's review the full URL structure.

As previously mentioned, the DNS record for storage accounts uses the following format:
* \<storage-account-name\>.blob.core.windows.net

For our example, the DNS record for our storage account would be 
* d3mostorageaccount.blob.core.windows.net

The full URL structure of a blob uses the following format:
* \<storage-account-name\>.blob.core.windows.net\/\<container-name\>/\<blob-name/>

The full URL for our "Example.txt" file would be:
* https://d3mostorageaccount.blob.core.windows.net/democontainer/Example.txt

Let's try downloading this file. I will use "curl" for this, but you simply put the above URL in a web-browser if you prefer.

```
$ curl https://d3mostorageaccount.blob.core.windows.net/democontainer/example.txt

This is just an example!
```

As you can see, we are able to read the file!

But wait, shouldn't some form of authentication be required to do this?

Well, as we saw before, this container currently has it's "publicAccess" setting set to "container".

```
$ az storage container list --account-name d3mostorageaccount
[
  {
    ...snip
    "name": "democontainer",

	...snip
      "publicAccess": "container"
    }
  }
]
```

Let's go over the various public access settings that determine whether your blobs are available anonymously.

### Public Access Levels and Anonymous Access

Storage accounts have their own setting to determine whether or not containers within them can be accessed over the Internet. The "Blob Public Access" setting on a storage account has to be set to Enabled for your storage account to be exposed to the Internet.

You can see in for our example storage account,  "Blob Public Access" is enabled.

![](/assets/images/Blob Public Access Enabled.png)

**However, this setting doesn't actually determine whether or not your containers and blobs are available anonymously.**

Each container within a storage account has a seperate public access level setting that determines anonymous access.

There are three public access levels for containers:

* Container: Anonymous read access for containers and blobs.
* Blob: Anonymous read access for Blobs only
* Private: No anonymous access

These descriptions provided by Microsoft can be a little confusing. Hopefully I can make this a bit easier to understand.

Azure Blob Storage consists of containers (directories), which store blobs (files). Each container has it's own public access level that determines if you can access the blobs within them anonymously. The names of these public access levels are "Private, Blob, and Container". The confusion here is that the actual resources are called containers and blobs, but public access level settings are also called "Container" or "Blob".

Let's go over each of these public access levels.

**Container:** If a container has the public access level set to "Container", all blobs within the container can be accessed anonymously if the storage account name and container name are known. This is the most permissive and most dangerous public access level for a container.

**Blob:** If a container has the public access level set to "Blob”, the blobs within the container can be accessed anonymously if the storage account name, container name, and blob name are known. This is less dangerous as an attacker would have to successfully guess or brute-force all three pieces of information in order to download the blob.

**Private:** No anonymous access is allowed.

Remember, our "democontainer" currrently has it's public access level set to "Container".

```
az storage container list --account-name d3mostorageaccount
[
  {
    ...snip
    "name": "democontainer",

	...snip
      "publicAccess": "container"
    }
  }
]
```

With our current configuration, if an attacker successfully guessed or brute-forced the “d3mostorageaccount” storage account name, the attacker could begin brute-forcing for valid container names. If the attacker successfully guessed a container name called “democontainer”, they would be able to list/download every blob in that container and download the files anonymously.

To take advantage of this and list out all blobs in a container, use this URL structure. <https://storage-account-name.blob.core.windows.net/container-name/?restype=container&comp=list>

For our example, this URL would look like this: <https://d3mostorageaccount.blob.core.windows.net/democontainer/?restype=container&comp=list>

If you navigate to this URL in a web-browser, you would see the following.

![](/assets/images/Listing Blobs.png)

As you can see, it lists out all the blobs within the container and even gives us the URL to download the "Example.txt" blob!

![](/assets/images/Demo Blob.png)

This can be pretty dangerous if this container stored sensitive information. However, it's all contigent on being able to guess the storage account name and container name. There are several tools designed to do this and it's not as difficult as you may think. 

**Note:** The storage accounts “Blob Public Access” setting overrides individual container public access level settings. If a container has its Public Access Level set to “Container”, but the storage account has “Blob Public Access” disabled, the containers cannot be accessed anonymously over the Internet.

Many organizations will include their name or  other easily guessable key-word within the storage account name. A popular tool used to do this is "MicroBurst". <https://github.com/NetSPI/MicroBurst> 

This tool is designed to take a list of keywords and perform DNS bruteforcing in order to identify valid storage account names. If it finds a valid storage account, it will then begin trying to brute-force valid container names.

I won't go over the details on how to do this, but Karl Fosaaen over at NetSPI has a great blog post on this: <https://www.netspi.com/blog/technical/cloud-penetration-testing/anonymously-enumerating-azure-file-resources/>

That's a totally viable attack path, but what if the storage account name isn't easily guessable? Well, this is a situation where security by obscurity definitely helps. But what if we had "Reader" privileges on the Azure environment?

### Abusing the Reader Role to Access Blobs

Often times when performing an Azure penetration test, you will start off from a "compromise assumed" perspective. The organization you are pentesting may give you read-only access to their Azure environment by providing a user account with the "Reader" role assigned.

If you remember, **the Reader role can see Azure resources, but not the data within them**. While we don't have explicit permissions to read the data within a storage account, we can view the storage accounts name and the name of any containers within them. We can even read the public access level set on each container!

As we just went over, if a container has it's public access level set to "Container", all an attacker needs to know is the storage account and container name in order to anonymously access the blobs within the container. 

Let's go back to our example. I've created a new storage account with a name that would be very difficult for an attacker to guess or brute-force. 

![](/assets/images/New Storage Account.png)
I've also created a container within this storage account that would also be difficult to guess or brute-force and given it the Public access level of "Container", which we know is dangerous.

![](/assets/images/New Containers.png)

Using an account with the "Reader" role, if I try to access this container in the Azure Portal I get an authorization error.

![](/assets/images/Can't Access.png)

However, since I can see the storage account name and container name, I can access the blobs within this container fully anonymously!

![](/assets/images/Listing Blobs 2.png)
We can see there is a blob called "secret.txt". Using the provided URL, we can download this file completely anonymously.

![](/assets/images/Super Secret File.png)

So as you can see, while the Reader role didn't let us see the data within the storage account containers, it did let us see the storage account name, container name, and the public access level of the container. This is a good example of where the "Control-plane" permissions allowed us to overcome the limitations of the "Data-plane" permissions assigned to the Reader role. 

## Automation

A popular tool that effectively audits storage account and container public access levels is BlobHunter, https://github.com/cyberark/BlobHunter. However, this tool requires a principal with either Owner, Contributor or, Storage Account Contributor. I have created a fork of this awesome tool that will work with Reader privileges to successfully list out any publicly available containers. One caveat, it does not list all the files within those containers as the original tool does, but gives you a good place to start manually searching. This fork is available here: https://github.com/Sol1dSn8keS3cur1ty/BlobHunter-Lite

## Remediation

1. Always set your container's public access level to "Private" if the data within them contain sensitive information you don't want to expose to the Internet. 
2. As you saw, the names of the storage accounts and container names are kind of like secrets in their own right. I think this is a case where "security by obscurity" is helpful, so avoid using easy to guess names when naming these resources.

## Conclusion

I hope I delivered on the two goals I set out to achieve deciding to write this blog.

1. Give an introduction to Azure Storage Accounts, and more specifically, the Blob Storage service. 
2. Detail how you may be able to abuse the "Reader" role to read sensitive information stored within Blob Storage.

