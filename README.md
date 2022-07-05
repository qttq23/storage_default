this includes:
- [1. Google Cloud Storage (recommended)](#1-google-cloud-storage-recommended)
- [2. Access control with IAM vs ACL](#2-access-control-with-iam-vs-acl)
- [3. Firebase Storage (not recommended)](#3-firebase-storage-not-recommended)
- [4. best practices](#4-best-practices)
- [5. other strategies](#5-other-strategies)

# 1. Google Cloud Storage (recommended)
- used to store static files such as images, videos, zip.
- each project has 1 or more `buckets`. Each bucket has 1 or more `objects` (files). Each file has the binary data and the `metadata` (such as Content-Type, Content-Disposition).

- GGCloudStorage uses `IAM and ACL` to manage access to buckets and objects.
- GG cloud storage only allows access through `Oauth2 token`. You get `oauth2 token` (usually from `service account`. from end-user is possible but not recommended, see why below) then pass that token along with request to GGcloudStorage. CloudStorage then checks if you present in the list of `IAM or ACL` to accept your request. 

- Due to it, GGCloudStorage doesnot have the separate libraries for admin. Every GCStorage libraries is called client libraries. it just cares about your `Oauth2 token`.
- for server, GGCloudStorage has library for `Nodejs` (also C++, Java, Python,.. but ironically, not for mobile and web). Nodejs GGCStorage library handles the complex process of getting oauth2 access token for you. so using Nodejs GGCStorage library in your server is recommended.
- for Clients in Mobile and Web and even Desktop, you can use the `Rest API` which is recommended.

(
quickstart:
https://cloud.google.com/storage/docs/discover-object-storage-console

access control:
https://cloud.google.com/storage/docs/access-control

authentication before using API or library:
https://cloud.google.com/storage/docs/authentication

nodejs client lib:
https://googleapis.dev/nodejs/storage/latest/

rest API:
https://cloud.google.com/storage/docs/xml-api/get-object
)

# 2. Access control with IAM vs ACL
- `IAM` is a system to manage user/account permission to access Google Cloud API/resources.
- `IAM` contains a list of pairs `<principle, role>`. `Principle` is a google user-account or a service account. `Role` is a set of permissions.
- Google Cloud Storage uses IAM to give client access to objects. It inherits entries from IAM and it can add more entries for its own.
for example: in IAM, you set an entry that says 'service account 1' has 'storage all permissions'. Then in each bucket's IAM, the 'service account 1' will present there and have all permissions to all of your buckets. later, in bucketA, if you set 'abc@gmail.com' has 'storage all permissions', the 'abc@gmail.com' will has all permissions to bucketA. other buckets will not be affected.

- `ACL` is a fined-grain way to restrict access to individual objects. It has the same concept with IAM: principle has some specific permissions (read/write/delete) to an object.

In practise, you should enable `ACL` to use both of `IAM and ACL`. for IAM, you should only allow the `service account` to have all permissions to storage. for ACL, you make objects public by allowing `alluser` to have `read` permission (get or download file). for ACL, you make object private by disallowing `allUsers` to have `read` permission. Regardless of public or private objects, `service account` can always access to them.  

Luckily, the Nodejs client library handles these ACL tasks to make your files public or private. (it requires ACL to be enabled first)

(
https://cloud.google.com/storage/docs/access-control/iam  
https://cloud.google.com/storage/docs/access-control/lists  

make file private:  
https://googleapis.dev/nodejs/storage/latest/File.html#makePrivate
)

# 3. Firebase Storage (not recommended)
- is a subset of Google Cloud Storage. It also uses GCStorage to create and manage buckets too.
- Can be used in conjunction with `Firebase Authentication`. Once client is authenticated by Firebase Auth, Firebase Storage can use `Rules` to detect who client is and furthermore look at client's `custom claims` to know the client's permission such as admin or normal user. this means you don't need additonal ApiServer to authenticate the requests to objects.

why not recommended?
- Must use `Firebase Products` which supports only Web and Mobile (not Desktop). 
- Firebase Storage will create its own bucket in Google cloud Storage. it cannot use the same bucket that you've created in google cloud storage.
- Rules in Firebase Storage can limit what you can check compared to your own ApiServer.
- Firebase Storage doesnot have alternatives in other cloud platform such as Azure or Amazon. This means Firebase is not a generic solution and you will get confused when switching to other cloud platforms.

(https://firebase.google.com/docs/storage)

# 4. best practices:

Usually, simple project should only have 1 `bucket` with the following structure:  
bucket/userA/file1.jpg  
bucket/userB/abc.txt  

You should enable `ACL` to make individual object private or public. (storage library for nodejs requires this)   
You should allow only the `service account` to have access to buckets and objects. Other end-user accounts should not present in the access list of any buckets.  
Doing so to minimize the potential of leaking data because of mis-config your buckets, objects.

When user want to download/upload file, your ApiServer generate a `Signed Url`. client will use this url to download/upload file.
The signed url has time expiration. ApiServer can require the client to provide exact some custom headers when client uses signed url. (but usually this is not neccessary)

Client (web, mobile, desktop) should use `XML Rest Api` to make requests to `Signed urls` to download/upload file.
Server (nodejs) should use Nodejs GCStorage client library to manage buckets and files and generate `signed urls`.

(
https://cloud.google.com/storage/docs/access-control/signing-urls-with-helpers

https://googleapis.dev/nodejs/storage/latest/File.html#getSignedUrl
)

## list files
client sends request to ApiServer.  
apiserver interacts with GCStorage to get client's files and return client list of files.   
(https://googleapis.dev/nodejs/storage/latest/Bucket.html#getFiles)

## delete file
client sends request to ApiServer.  
apiserver interacts with GCStorage to delete client's file.    
(https://googleapis.dev/nodejs/storage/latest/File.html#delete)

## download file
client requests to download a certain file.  
apiServer generate a `Signed Url` for downloading that file and return that signed url to client.  
client uses signed url to start download file.  

Note:   
in case of native clients (mobile, desktop), client has to use `XML Rest Api` to download file. client should `GET` the `signed url` and receive response's data partially in body. Each time data comes, client write those data to local file. repeat until response complete. (only 1 request and 1 response)

In case of web browser, client just needs to open that `signed url` in a new tab, the browser will prompt the SaveDialog to user to save file to and handle download process for you.

(
https://cloud.google.com/storage/docs/access-control/signing-urls-with-helpers#download-object

overview:
https://cloud.google.com/storage/docs/downloading-objects#download-object-xml
https://cloud.google.com/storage/docs/xml-api/get-object-download
)

## upload file
client request to upload a certain file.  
apiServer generate a `Signed Url` for uploading that file and return that signed url to client. This should be the `resumable upload`.  
client uses signed url to start uploading file.  
The `resumable upload` is recommended. In summary, the client will has to init a `POST` request to get destination URL. then client will `PUT` data partially to that destinationURL. Repeat `PUT` until GCStorage responds `OK`. (1 POST request, multiple PUT requests).

Both native client and web client should use this method. (GCStorage not recommend the `Multipart-form-data` method).  
Navive client will involve reading data from file (eg: c++, FILE or iostream) and send/receive HTTP requests (eg: c++, cpp-httplib).  
Web client will involve reading data from file (file input tag and File.slide() method) and send/receive HTTP requests (XMLHttpRequest).  

(
note: the `action` param should be 'resumable':  
https://cloud.google.com/storage/docs/access-control/signing-urls-with-helpers#upload-object  

overview:
https://cloud.google.com/storage/docs/performing-resumable-uploads#chunked-upload

https://cloud.google.com/storage/docs/xml-api/post-object-resumable  
https://cloud.google.com/storage/docs/xml-api/put-object-upload  
)

# 5. other strategies:
## signed urls (as describe in `best practice`)
- api server (nodejs) holds the Service Account and generates Signed Url which is short-lived and secured url. (accessing SignedUrls requires some headers)
- client (desktop, web,..) requests to api server to get signedUrl for Upload/Download/Delete an object in GC storage.

-> popular method. allow api server controls every actions of users.  
-> drawback: user usually has to make at least 2 requests to download/upload/delete a file. also increase number of tasks to api server.  
-> recommended.  

## stream upload/download
- api server (nodejs) holds the Service Account and forward data from/to user to/from GC strorage.
- upload: users uploads file to api server and api server forwards (streams) received data to GC storage. api server don't save data to local file.
- download: api server forwards (streams) received data from GC storage to the response to user. api server don't save data to local file.

-> allow api server controls every actions of users. user only makes 1 request for downloading or uploading.  
-> drawback: very heavy burden to api server. because it has to receive and forward the data that it do not use.  
-> not recommended.  

## firebase storage
- used in conjunction with Firebase Authentication.
- user needs to signin to Firebase authentication first.
- sigined user accesses GC storage directly.
- firebase Storage has Security Rules that can be used to investigate Custom Claims of signed-in user (set by Firebase Authentication).

-> access GC storage directly without backend server.  
-> drawbacks: Firebase only supports Web & Mobile. Impossible for desktop to use Firebase services.  

(GC storage also allows authentication from individual google end-user accounts (IAM). but that only works for Google account, not work for Github/Facebook accounts.
so not recommended)
