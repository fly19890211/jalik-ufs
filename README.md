# UploadFS

UploadFS is a Meteor package that aims to make file uploading easy, fast and configurable.
An important feature is the ability to **start, stop or even abort a transfer** when you want.
It currently only supports file system storage but it is possible to extend possibilities by creating new Stores yourself.

## Testing

You can test the package by downloading and running [UFS-Example](https://github.com/jalik/ufs-example) which is simple demo of UploadFS.

## Installation

To install the package, execute this command in the root of your project :
```
meteor add jalik:ufs
```

If later you want to remove the package :
```
meteor remove jalik:ufs
```

## Plugins

As the package is modular, you can add support for custom stores.

* [UploadFS.store.Local](https://github.com/jalik/jalik-ufs-local)
* [UploadFS.store.GridFS](https://github.com/jalik/jalik-ufs-gridfs)

## Introduction

In file uploading, you basically have a client and a server, I haven't change those things.
So on the client side, you create an uploader for each file transfer needed, while on the server side you only configure a store where the file will be saved.
I'll use the `UploadFS.store.Local` store in the following examples.

## Configuration

You can access and modify settings via `UploadFS.config`.

```js
// Activate simulation for slowing file reading
UploadFS.config.simulateReadDelay = 1000; // 1 sec

// Activate simulation for slowing file uploading
UploadFS.config.simulateUploadDelay = 1000; // 1 sec

// Activate simulation for slowing file writing
UploadFS.config.simulateWriteDelay = 2000; // 2 sec

// This path will be appended to the site URL, be sure to not put a "/" as first character
// for example, a PNG file with the _id 12345 in the "photos" store will be available via this URL :
// http://www.yourdomain.com/uploads/photos/12345.png
UploadFS.config.storesPath = 'uploads';

// Set the temporary directory where uploading files will be saved
// before sent to the store.
UploadFS.config.tmpDir = '/tmp/uploads';
```

## Creating a Store

**All stores must be available on the client and the server.**

A store is the place where your files are saved, it could be your local hard drive or a distant cloud hosting solution.
Let say you have a **photos** collection which is used to save the files info.

```js
Photos = new Mongo.Collection('photos');
```

What you need is to create the store that will will contains the data of the **Photos** collection.
The **name** of the store must be unique because it will be used by the uploader to send the file on the store.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos'
});
```

## Filtering uploads

You can pass an `UploadFS.Filter` to the store to define restrictions on file uploads.
Filter is tested before inserting a file in the collection and uses `Meteor.deny()`.
If the file does not match the filter, it won't be inserted and will not be uploaded.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Apply a filter to restrict file upload
    filter: new UploadFS.Filter({
        minSize: 1,
        maxSize: 1024 * 1000 // 1MB,
        contentTypes: ['image/*'],
        extensions: ['jpg', 'png']
    })
});
```

The filter can be a function in which you need to throw an Error to stop insertion.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Apply a filter to restrict file upload
    filter: function(userId, file) {
        if (!userId) {
            throw new Meteor.Error(403, 'Forbidden');
        }
    }
});
```

## Transforming files

If you need to modify the file before it is saved to the store, you have to use the **transform** option.
A common use is to resize/compress images to get lighter versions of the uploaded files.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Transform file when reading
    transformRead: function (from, to, fileId, file, request) {
        from.pipe(to); // this returns the raw data
    }
    // Transform file when writing
    transformWrite: function (from, to, fileId, file) {
        var im = Npm.require('imagemagick-stream');
        var resize = im().resize('800x600').quality(90);
        from.pipe(resize).pipe(to);
    }
});
```

## Copying files (since v0.3.6)

You can copy files to other stores, it could be for backup or just to have alternative versions of the same file (eg: thumbnails).

```js
Thumbnails = new Mongo.Collection('thumbnails');

PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    copyTo: [
        new UploadFS.store.Local({
            collection: Thumbnails,
            name: 'thumbnails',
            transformWrite: function(from, to, fileId, file) {
                var im = Npm.require('imagemagick-stream');
                var resize = im().resize('128x128').quality(80);
                from.pipe(resize).pipe(to);
            }
        })
    ]
});
```

The copies contains 2 fields that links to the original file, `originalId` and `originalStore`.
So if you want to display a thumbnail instead of the original file :

```html
<template name="photos">
    {{#each photos}}
        <img src="{{thumb.url}}">
    {{/each}}
</template>
```

```js
Template.photos.helpers({
    photos: function() {
        return Photos.find();
    },
    thumb: function() {
        return Thumbnails.findOne({originalId: this._id});
    }
});
```

## Setting permissions

If you don't want anyone to do anything, you must define permission rules.
By default, there is no restriction (except the filter) on insert, remove and update actions.

```js
Photos.allow({
    insert: function (userId, doc) {
        return userId;
    },
    update: function (userId, doc) {
        return userId === doc.userId;
    },
    remove: function (userId, doc) {
        return userId === doc.userId;
    }
});
```

## Filtering file access

When returning the file for a HTTP request, you can do some checks to decide whether or not the file should be sent to the client.
This is done by defining the `onRead()` method on the store.

**Note:** Since v0.3.5, every file has a token attribute when its transfer is complete, this token can be used as a password to access/display the file. Just be sure to not publish it if not needed. You can also change this token whenever you want making older links to be staled.

```html
{{#with image}}
<a href="{{url}}?token={{token}}">
    <img src="{{url}}?token={{token}}">
</a>
{{/with}}
```

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    onRead: function (fileId, file, request, response) {
        // Allow file access if not private or if token is correct
        if (file.isPublic || request.query.token === file.token) {
            return true;
        } else {
            response.writeHead(403);
            return false;
        }
    }
});
```

## Server events

Some events are triggered to allow you to do something at the right moment on server side.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Called when file has been uploaded
    onFinishUpload: function (file) {
        console.log(file.name + 'has been uploaded');
    }
});
```

## Handling errors

On server side, you can do something when there is a store IO error.

```js
PhotosStore = new UploadFS.store.Local({
    collection: Photos,
    name: 'photos',
    path: '/uploads/photos',
    // Called when a copy error happened
    onCopyError: function (err, fileId, file) {
        console.error('Cannot create copy ' + file.name);
    }
    // Called when a read error happened
    onReadError: function (err, fileId, file) {
        console.error('Cannot read ' + file.name);
    }
    // Called when a write error happened
    onWriteError: function (err, fileId, file) {
        console.error('Cannot write ' + file.name);
    }
});
```

## Writing files

Sometimes you could need to write to the store directly on the server without any client involved.

```js
// Insert the file in database
var fileId = store.create(file);

// Save the file to the store
store.write(readStream, fileId, function(err, file) {
    if (err) {
        console.error(err);
    }else {
        console.log('file saved to store');
    }
});
```

## Uploading files

When the store on the server is configured, you can upload files to it.

Here is the template to upload one or more files :

```html
<template name="upload">
    <input type="files" multiple>
</template>
```

And there the code to upload the selected files :

```js
Template.upload.events({
    'change input[type=files]': function (ev) {
        var self = this;
    
        // Here we get the ArrayBuffer for each file of the event
        UploadFS.readAsArrayBuffer(ev, function (data, file) {
            // Prepare the file to insert in database, note that we don't provide an URL,
            // it will be set automatically by the uploader when file transfer is complete.
            var photo = {
                name: file.name,
                size: file.size,
                type: file.type,
                customField: 1337
            };
    
            // Create a new Uploader for this file
            var upload = new UploadFS.Uploader({
                // This is where the uploader will save the file
                store: PhotosStore,
                // The size of each chunk sent to the server
                chunkSize: 1024 * 8,
                // This tells how many tries to do if an error occurs during upload
                maxTries: 5,
                // The file data
                data: data,
                // The document to save in the collection
                file: photo,
                // The error callback
                onError: function (err) {
                    console.error(err);
                },
                // The complete callback
                onComplete: function (file) {
                    console.log(file.name + ' has been uploaded');
                }
            });
            
            // Reactive method to get upload progress
            Tracker.autorun(function() {
                console.log((upload.getProgress() * 100) + '% completed');
            });
            
            // Reactive method to get upload status
            Tracker.autorun(function() {
                console.log('transfer ' + (upload.isUploading() ? 'started' : 'stopped'));
            });
            
            // Starts the upload
            upload.start();
            
            // Stops the upload
            upload.stop();
            
            // Abort the upload
            upload.abort();
        });
    }
});
```

## Displaying images

After that, if everything went good, you have you file saved to the store and in database.
You can get the file as usual and display it using the url attribute of the document.

Here is the template to display a list of photos :

```html
<template name="photos">
    <div>
        {{#each photos}}
            {{#if uploading}}
                <img src="/images/spinner.gif" title="{{name}}">
                <span>{{completed}}%</span>
            {{else}}
                <img src="{{url}}" title="{{name}}">
            {{/if}}
        {{/each}}
    </div>
</template>
```

And there the code to load the file :

```js
Template.photos.helpers({
    completed: function() {
        return Math.round(this.progress * 100);
    },
    photos: function() {
        return Photos.find();
    }
});
```

## Template helpers

Some helpers are available by default to help you work with files inside templates.

```html
{{#if isAudio}}
    <audio src="{{url}}" controls></audio>
{{/if}}
{{#if isImage}}
    <img src="{{url}}">
{{/if}}
{{#if isVideo}}
    <video src="{{url}}" controls></video>
{{/if}}
```
