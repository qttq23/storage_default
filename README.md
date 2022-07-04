this includes:
- 


# Google Cloud Storage (recommended)
- used to store static files such as images, videos, zip.
- each project has 1 or more `buckets`. Each bucket has 1 or more `objects` (files). Each file has the binary data and the metadata (such as Content-Type, Content-Disposition).
- GGCloudStorage uses `IAM and ACL` to manage access to buckets and objects.

- GG cloud storage only allows access through Oauth2 token. You get oauth2 token (usually from service account. from end-user is possible but not recommended, see why below) then pass that token along with request to GGcloudStorage. CloudStorage then checks if you present in the list of IAM or ACL to accept your request. 
- Due to it, GGCloudStorage doesnot have the separate libraries for client or admin. it just cares about your Oauth2 token.

- for server, GGCloudStorage has library for Nodejs (also C++, Java, Python,.. but ironically, not for mobile and web). Nodejs GGCStorage library handles the complex process of getting oauth2 access token for you. so using Nodejs GGCStorage library in your server is recommended.
- for Clients in Mobile and Web and even Desktop, you can use the Rest API which is recommended.

# authentication with IAM vs ACL
- IAM is a system to manage user/account permission to access Google Cloud API/resources.
- IAM contains a list of pairs <principle, role>. Principle is a google user-account or a service account. Role is a set of permissions. (see references here..)
- Google Cloud Storage uses IAM to give client access to objects. It inherits entries from IAM and it can add more entries for its own.
for example: in IAM, you set an entry that says 'service account 1' has 'storage all permissions'. Then in each bucket's IAM, the 'service account 1' will present there and have all permissions to all of your buckets. later, in bucketA, if you set 'abc@gmail.com' has 'storage all permissions', the 'abc@gmail.com' will has all permissions to bucketA. other buckets will not be affected.

- ACL is a fined-grain way to restrict access to individual objects. It has the same concept with IAM: principle has some specific permissions (read/write/delete) to an object.

In practise, you should enable ACL to use both of IAM and ACL. for IAM, you should only allow the `service account` to have all permissions to storage. for ACL, you make objects public by allowing `alluser` to have `read` permission (get or download file). for ACL, you make object private by disallowing `allUsers` to have `read` permission. Regardless of public or private objects, `service account` can always access to them.  

Luckily, the Nodejs client library handles these ACL tasks to make your files public or private. ( it requires ACL to be enabled first)

# Firebase Storage (not recommended)
- is a subset of Google Cloud Storage.
- Can be used with Firebase Authentication. Once client is authenticated by Firebase Auth, Firebase Storage can use Rules to detect who client is and furthermore look at client's custom claims to know the client's permission such as admin or normal user. this means you don't need additonal ApiServer to authenticate the requests to objects.

why not recommended?
- Must use Firebase Products which supports only Web and Mobile (not Desktop). 
- Firebase Storage will create new seperate bucket in Google cloud Storage. it cannot use the same bucket in google cloud storage.
- Rules in Firebase Storage can limit what you can check compared to your own ApiServer.
- Firebase Storage doesnot have alternatives in other cloud platform such as Azure or Amazon.

# best practices:

Usually, simple project should only have 1 bucket with the following structure:  
bucket/userA/file1.jpg  
bucket/userB/abc.txt  

You should enable ACL to make individual object private or public. (storage library for nodejs requires this) 
You should allow only the service account to access to buckets and objects. Other end-user accounts should not present in the access list of any buckets.
Doing so to minimize the potential of leaking data because of mis-config your buckets, objects.

???...

## list files

## delete file

## download file

## upload file

