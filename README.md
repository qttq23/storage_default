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
apiserver interacts with GCStorage to get client's files and return client list of files. (may contain the file's metadata)  
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
(optionally, client requests a `HEAD` to get the file's metadata.)  

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
`Navive client` will involve reading data from file (eg: c++, FILE or iostream) and send/receive HTTP requests (eg: c++, cpp-httplib).  
`Web client` will involve reading data from file (file input tag and File.slide() method) and send/receive HTTP requests (XMLHttpRequest). remember to config CORS (see below)  
(looks like the 'input file' only contains file information not the file content. so don't worry when select large files. file's content can be get using 'File.arrayBuffer()')

(
note: the `action` param should be 'resumable':  
https://cloud.google.com/storage/docs/access-control/signing-urls-with-helpers#upload-object  

overview:
https://cloud.google.com/storage/docs/performing-resumable-uploads#chunked-upload

https://cloud.google.com/storage/docs/xml-api/post-object-resumable  
https://cloud.google.com/storage/docs/xml-api/put-object-upload  
)

## set CORS
by default, google cloud storage don't have CORS configured. you have to config manually if you are accessing from web browser.
(
https://cloud.google.com/storage/docs/configuring-cors#gsutil_1

sample: view `bucket_cors_config.json` & `set_bucket_cors.bat` files
)

## others
performance consideration: 
imagine your client app should show 20 images (big thumbnail) each time users open app.
each time a user opens your app, client app has to request server to sign 20 signed urls.
this ridiculously wastes your server's time and resources.
instead, consider generating a long time expired url such as 7 days.
client requests signed urls once and store them to local storage.
every time client needs to get signed url, it should first check the local storage.
if the url is expired, client can detect error (for html img tag: onerror event) and re-request new url from server.


fine-grained permission:
imagine your app has requirements to allow a user to store for file privately and potentially share to their friends.
everything works just fine: you set share policy to your own file and api server remembers that policy. 
When your friend requests that file, api server allows and gives your friends signed url to that file.
but what if later, you want to disallow one of your friends out of your file share policy ?
your disallowed friend already has signed url that only expired after next 7 days.

according to google storage docs: "Anyone who knows the URL can access the resource until the expiration time for the URL is reached or the key used to sign the URL is rotated."
this means you has to wait for expiration time for your new policy to take effect.
or you have to renew your service account, which affects the whole bucket not only your file. so this also not practical.
(https://cloud.google.com/storage/docs/access-control/signed-urls#should-you-use)

Another workaround is that you change your file's name in google storage.  (I recommended)
you also have to update storage filename in database (eg: mysql) where you store file's info and permission/policy.
after setting new storage filename, the disallowed friend cannot use signed url to access to old filename. he also can't request api server to give him a new signed url before api server already remembers your new policy.
the allowed friends temporarily cannot use old url but they can request api server to give them new signed url.

the only drawback is that nodejs google-storage library cannot actually 'rename' a file, it internally 'copy' then 'delete' old file.
so if the file you want to rename is large, the action 'rename' will take long to complete.
(https://googleapis.dev/nodejs/storage/latest/File.html#rename)


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
