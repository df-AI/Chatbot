
# [GCP] Storing Image and Video Files in Cloud Storage

Overview
@(GCP Coursera)[node.js, GCP 프로젝트 생성, Cloud Datastore, Storing Image and Video Files]

In this lab, you will enhance the online Quiz application to work with images and videos. You will store files as objects in a Cloud Storage bucket.

The Quiz application user interface has been changed to include a file input in the add question form, and the handler has been modified to receive the image data in the server-side application.

You will add the code that stores the uploaded file into Cloud Storage, make the object publicly available, and then store the object URL along with other application data in Google Cloud Datastore.

----------

[TOC]

## Reviewing the Case Study Application

> In this section, you will access Cloud Shell, clone the git repository containing the Quiz application, and run the application.

### Clone source code in Cloud Shell

1. In the Cloud Console, click Activate Google Cloud Shell.
2. If a dialog box appears, click Start Cloud Shell.
3. Clone the repository for the class.
```powershell
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```
### Configure and run the case study application
1. To change the working directory, execute the following command: 
```powershell
cd ~/training-data-analyst/courses/developingapps/nodejs/cloudstorage/start
```

2. To configure the application, execute the following command:
```powershell
. prepare_environment.sh
```
> This script file
> Creates an App Engine application.
> Exports an environment variable, GCLOUD_PROJECT.
> Runs npm install.
> Creates entities in Cloud Datastore.
> Prints out the Google Cloud Platform Project ID.

3. To run the application, execute the following command:
```powershell
npm start
```

### Review the changes to the case study application
1. To view the application, click Web preview > Preview on port 8080.
2. Click the Create Question link in the toolbar.

> You should see a simple form that contains textboxes for the question and answers and radio buttons to select the correct answer.
> The form has a new file upload field that will be used to upload either image or video files.

## Examining the Case Study Application Code
### Employ the Cloud Shell Editor
1. In Cloud Shell, click Launch code editor.
2. Navigate to the /training-data-analyst/courses/developingapps/nodejs/cloudstorage/start folder using the file browser panel on the left side of the editor.

### Review the Express Web application 
1. Select the add.pug file in the .../server/web-app/views/questions folder.
> This file contains the pug template for the Create Question form.
> Notice how the form has been modified to use multipart/form-data as the enc-type, and there are two new form controls:
> A file upload control called image
> A hidden field called imageUrl

2. Select the questions.js file in the .../server/web-app folder.
> This file has been modified so that the POST handler chains together three distinct middleware calls:
> 1. Configures Multer to accept a single image file from a form field called image. Multer is a popular Express middleware package for handling file uploads. The Multer handler consumes the file multipart upload from the browser and holds the file data in memory.
> 2. Consumes the file processed by Multer. This handler is where you will write the Cloud Storage code to upload a file, make it publicly available, and store the public URL for the object.
> 3. Consumes the public URL to store the data into Cloud Datastore. You already wrote the code to store an entity in Cloud Datastore in the previous lab!

3. Select the .../server/gcp/cloudstorage.js file.
> This is the file where you will write code to save image file data into Cloud Storage.

## Creating a Cloud Storage Bucket
In this section, you will create a Cloud Storage bucket and export an environment variable that references it.
### Create a Cloud Storage bucket
1. Return to the Cloud Shell command line, and stop the application by pressing Ctrl+C.
2. To create a Cloud Storage bucket named <Project ID>-media, execute the following command:
```powershell
gsutil mb gs://$DEVSHELL_PROJECT_ID-media
```
> You can create a bucket using the gsutil mb command, passing through the name of the bucket as gs://BUCKET_NAME
> You can use $DEVSHELL_PROJECT_ID as the bucket name prefix followed by -media

3. To export the Cloud Storage bucket name as an environment variable named GCLOUD_BUCKET, execute the following command:
```powershell
export GCLOUD_BUCKET=$DEVSHELL_PROJECT_ID-media
```
> Recall that the application makes use of environment variables for configuration.
> This allows the development team to deploy the application into development, test, staging, and production just by changing these variables.

## Adding Objects to Cloud Storage

In this section, you will write code to save uploaded files into Cloud Storage.
> Important: Update code within the sections marked as follows:
> // TODO
> // END TODO
> To maximize your learning, review the code, inline comments, and related API documentation.

> You can view the API documentation for Cloud Storage at:
> https://googlecloudplatform.github.io/google-cloud-node/#/docs/storage

### Import and use the NodeJS Cloud Storage module
1. In the editor, move to the top of the ...gcp/cloudstorage.js file.
2. Load the '@google-cloud/storage module, and assign it to a constant named Storage.
3. Use the Storage(...) factory to construct a Cloud Storage client named storage.
4. Declare a string constant named GCLOUD_BUCKET, and initialize it with the name of the bucket you previously exported as an environment variable.
5. Declare a constant named bucket to reference the Cloud Storage [bucket][1].
**cloudstorage.js**
```javascript
// TODO: Load the module for Cloud Storage

const Storage = require('@google-cloud/storage');

// END TODO

// TODO: Create the storage client
// The Storage(...) factory function accepts an options
// object which is used to specify which project's Cloud
// Storage buckets should be used via the projectId
// property.
// The projectId is retrieved from the config module.
// This module retrieves the project ID from the
// GCLOUD_PROJECT environment variable.

const storage = Storage({
 projectId: config.get('GCLOUD_PROJECT')
});

// END TODO

// TODO: Get the GCLOUD_BUCKET environment variable
// Recall that earlier you exported the bucket name into an
// environment variable.
// The config module provides access to this environment
// variable so you can use it in code

const GCLOUD_BUCKET = config.get('GCLOUD_BUCKET');

// END TODO

// TODO: Get a reference to the Cloud Storage bucket

const bucket = storage.bucket(GCLOUD_BUCKET);

// END TODO
```

### Write code to send a file to Cloud Storage
1. In the sendUploadToGCS(req, res, next) handler, use the Cloud Storage client to upload a file to your Cloud Storage bucket and make it publicly available.
2. Get a reference to a Cloud Storage file object in the bucket.
3. Create a writeable stream to send data to the Cloud Storage object. Pass through the MIME type for the object.
4. Attach two event handlers to the stream: the first will handle errors, the second will be invoked when the data has finished uploading.
5. In the error event handler, invoke the next Express middleware.
6. In the finish event handler, make the file public, and set a new property on the Datastore entity that references the new public URL.
7. At the end of the sendUploadToGCS(req, res, next) handler, to upload the file's data into Cloud Storage, write the file to the stream.
**cloudstorage.js**
```javascript
function sendUploadToGCS (req, res, next) {

// The existing code in the handler checks to see if there
// is a file property on the HTTP request - if a file has
// been uploaded, then Multer will have created this
// property in the preceding middleware call.

  if (!req.file) {
    return next();
  }

// In addition, a unique object name, oname,  has been
// created based on the file's original name. It has a
// prefix generated using the current date and time.
// This should ensure that a new file upload won't
// overwrite an existing object in the bucket
  const oname = Date.now() + req.file.originalname;
  // TODO: Get a reference to the new object

  const file = bucket.file(oname);

  // END TODO

  // TODO: Create a stream to write the file into
  // Cloud Storage
// The uploaded file's MIME type can be retrieved using
// req.file.mimetype.
// Cloud Storage metadata can be used for many purposes,
// including establishing the type of an object.

  const stream = file.createWriteStream({
    metadata: {
      contentType: req.file.mimetype
    }
  });

  // END TODO

  // TODO: Attach two event handlers (1) error
  // Event handler if there's an error when uploading

  stream.on('error', (err) => {
    // TODO: If there's an error move to the next handler

    next(err);

    // END TODO
  });

  // END TODO

  // TODO: Attach two event handlers (2) finish
  // The upload completed successfully
  stream.on('finish', () => {

    // TODO: Make the object publicly accessible
    file.makePublic().then(() => {
      // TODO: Set a new property on the file for the
      // public URL for the object

// Cloud Storage public URLs are in the form:
// https://storage.googleapis.com/[BUCKET]/[OBJECT]
// Use an ECMAScript template literal (`https://...`)to
// populate the URL with appropriate values for the bucket
// ${GCLOUD_BUCKET} and object name ${oname}

      req.file.cloudStoragePublicUrl = `https://storage.googleapis.com/${GCLOUD_BUCKET}/${oname}`;

// END TODO

      // TODO: Invoke the next middleware handler

      next();

      // END TODO
    });
  });

  // END TODO

  // TODO: Upload the file's data into Cloud Storage

  stream.end(req.file.buffer);

  // END TODO
}
```

### Run the application and create a Cloud Storage object
1. Save the ...gcp/cloudstorage.js file, and then return to the Cloud Shell command.
2. Start the application by typing npm start
3. Download an image file to your local machine from https://upload.wikimedia.org/wikipedia/en/2/2e/Google_Cloud_Storage_logo.png
4. In Cloud Shell, click Web preview > Preview on port 8080 to preview the Quiz application.
![Alt text](./img/capture_2.png)

5. Click the Create Question link.
6. Complete the form with the following values, and then click Save.
### Table
| Form Field      |    Value | 
| :-------- | --------:|
| Author  | Your Name | 
| Quiz     |   Google Cloud Platform | 
| Title      |    Which product does this logo relate to? |
| Image      |    Upload the Google_Cloud_Storage_logo.png file you previously downloaded |
| Answer 1      |    App Engice |
| Answer 2      |    Cloud Storage (select the Answer 2 radio button!) |
| Answer 3      |    Compute Engine |
| Answer 4      |    Container Engine | 

7. Return to the Cloud Console and click on Navigation menu > Storage.
8. On the Storage > Browser page, click the correct bucket (named <Project ID>-media).
> You should see your new object named 
> '#UniqueNumber#Google_Cloud_Storage_logo.png.'
![Alt text](./img/capture_1.png)


### Run the client application and test the Cloud Storage public URL
1. Add /api/quizzes/gcp to the end of the application's URL.
> You should see that JSON data has been returned to the client corresponding to the Question you added in the web application.
> The imageUrl property should have a value corresponding to the object in Cloud Storage.

2. Return to the application home page and click the Take Test link.
3. Click GCP, and answer each question.
> When you get to the question you just added, you should see the image has been formatted inside the client-side web application!



[1]: https://googlecloudplatform.github.io/google-cloud-node/#/docs/storage/1.2.0/storage?method=bucket