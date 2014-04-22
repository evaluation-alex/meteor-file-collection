# fileCollection

`fileCollection` is a [Meteor.js](https://www.meteor.com/) smart [package](https://atmospherejs.com/package/collectionFS) that cleanly and simply extends Meteor's `Collection` metaphor to dealing with collections of files. If you know how to use Meteor's collections, you already know 90% of what you need to know to begin working with `fileCollection`.

```js
files = new fileCollection('myFiles');
```

Under the hood, file data is stored within the Meteor MongoDB instance using a Mongo technology called [gridFS](http://docs.mongodb.org/manual/reference/gridfs/). Your fileCollections and the underlying gridFS collection remain perfectly in sync because they *are* the same collection, and `fileCollection` is safe for concurrent read/write access to files via (automatic file locking)[https://github.com/vsivsi/gridfs-locks]. `fileCollection` also provides a simple way to enable secure HTTP REST (GET, POST, PUT, DELETE) intefaces to your files, and additionally supports robust and resumable file uploads using the excellent [Resumable.js](http://www.resumablejs.com/) library.

My goal in writing this package was to stay true to the spirit of Meteor and build something that is straightforward and just works. If you're searching for ways to deal with files on Meteor, you've probably also encountered [collectionFS](https://atmospherejs.com/package/collectionFS). If not, you should definitely check it out. It's a great library written by great people, and I even helped out with a rewrite of their [gridFS support](https://atmospherejs.com/package/cfs-gridfs).

Here's the difference in a nutshell: collectionFS is a Ferrari, and fileCollection is a Fiat. They do approximately the same thing, using some of the same technologies, but have different design philosophies. `fileCollection` is much simpler and less flexible; but if it does what you need you'll probably find it has a lot fewer moving parts and may be quite a bit more efficient. With that said, if you need all of the bells and whistles collectionFS is probably for you.

Enough words, time for code... The block below implements a server fileCollection with userId owner secured HTTP file upload and download, and sets the client up to provide drag and drop chunked file uploads. The only this missing is the UI templates and a some helper functions. See the `sampleApp` subdirectory for a complete working verison written in [CoffeeScript](http://coffeescript.org/).

```js

// Create a collection, and enable file upload and download using HTTP

files = new fileCollection('myFiles',
  { resumable: true,
    http: [
      { method: 'get',
        path: '/:md5',
        lookup: function (params, query) {
          return { md5: params.md5 }
      }
    ]
  }
);

if (Meteor.isServer) {

  // Only publish files owned by this userId, and ignore
  // file chunks being used by Resumable.js for current uploads

  Meteor.publish('myData',
    function () {
      files.find({ 'metadata._Resumable': { $exists: false },
                   'metadata.owner': this.userId })
    }
  );

  // Allow rules for security. Should look familiar!
  // Without these, no file writes would be allowed
  files.allow({
    remove: function (userId, file) {
      // Only owners can delete
      if (userId !== file.metadata.owner) {
        return false;
      } else {
        return true;
      }
    },
    // Client file document updates are all denied, implement Methods for that
    // This rule secures the HTTP REST interfaces' PUT/POST
    update: function (userId, file, fields) {
      // Only owners can upload file data
      if (userId !== file.metadata.owner) {
        return false;
      } else {
        return true;
      }
    },
    insert: function (userId, file) {
      // Assign the proper owner when a file is created
      file.metadata = file.metadata or {};
      file.metadata.owner = userId;
      return true;
    }
  });
}

if (Meteor.isClient) {
  Meteor.subscribe('myData');

  Meteor.startup(function() {
    // This assigns a file upload drop zone to some DOM node
    files.resumable.assignDrop($(".fileDrop"));

    // When a file is added via drag and drop
    myData.resumable.on('fileAdded', function (file) {
      // Create a new file in the file collection to upload
      files.insert({
        _id: file.uniqueIdentifier,  // This is the ID resumable will use
        filename: file.fileName,
        contentType: file.file.type
        },
        function (err, _id) {
          if (err) {
            return console.error("File creation failed!", err);
          }
          // Once the file exists on the server, start uploading
          myData.resumable.upload();
        });
    });
  });
}
```

### Installation

I've only tested with Meteor v0.8, it might run on Meteor v0.7 as well, but buyer beware!

Requires [meteorite](https://atmospherejs.com/docs/installing). To add to your project, run:

    mrt add fileCollection

The package exposes a global object `fileCollection` on both client and server.

To run tests (using Meteor tiny-test) run from within the `package` subdir:

    meteor test-packages ./fileCollection/

## Use

Before going any further, it will pay to take a minute to familiarize yourself with the MongoDB gridFS `files` [data model](http://docs.mongodb.org/manual/reference/gridfs/#the-files-collection). This is the schema used by `fileCollection` because fileCollection *is* gridFS.

Now, there are a couple of things to know about the file schema:

1.    Some of those attributes belong to you. Do whatever you want with them.
2.    Some of those attributes belong to gridFS, and you could **lose data** if you try too hard to mess with them.
3.    `_id`, `length`, `chunkSize`, `uploadDate` and `md5` should be considered read-only. Changing these directly is a bad idea.
4.    You can do whatever you want with `filename`, `contentType`, `aliases` and `metadata`. Go to town.
5.    `contentType` should probably be a valid [MIME Type](https://en.wikipedia.org/wiki/MIME_type)
6.    `filename` is *not* guaranteed unique. So `_id` is a better bet.

Sound complicated? It's really not, and `fileCollection` is here to help. First off, when you create a new file, you use `file.insert(...)`, and just populate the attributes you care about and fileCollection does the rest. You are guaranteed to get a valid gridFS file, even if you do this: `id = file.insert();`  Likewise, when you run `update` on the server, it tries hard to check that you aren't clobbering one of the "read-only" attributes with your modifier. And clients aren't allowed to directly `update` at all, although you can selectively give them that power via `Meteor.methods()`.

### Security

You may have noticed that the gridFS file schema says nothing about file ownership. That's your job. If you sqint at the code block up above, you will see a bare bones `Meteor.userId` based ownership scheme implemented with the attribute `file.metadata.owner`. Obviously, allow/deny rules are needed to enforce and defend that attribute, and fileCollection implements those in *almost* the same way that ordinary Meteor collections do. Here's how they're a little different:

*    A file is always created as a valid zero-length gridFS file using `insert` on the client/server.
*    The `update` allow/deny rules secure writing data to an inserted file from outside via HTTP. This means that an HTTP POST/PUT can not create a new file all by itself, it has to have been inserted first.
*    `remove` works just as you would expect, and it also secures the HTTP DELETE method.
*    HTTP REST interface calls can be authenticated to a Meteor userId using a currently valid authentication token.

### Limits

There are essentially no hard limits on the number or size of files other than what your hardware will support. Because fileCollection uses robust multiple reader / exclusive writer locking on top of gridFS, essentially any number of readers and writers of shared files may coexist with no risk of data loss.

## API

The `fileCollection` API is essentially a twist on the Meteor Collection API, with almost all of the same methods and a few file specific ones mixed in. The big loser is `upsert()`, it's gone. If you try to call it, you'll get an error. Ditto for `update()` on the client side.

### new fileCollection()

### file.find()

### file.findOne()

### file.insert()

### file.update()

### file.remove()

### file.allow()

### file.deny()

### file.upsertStream()

### file.findOneStream()

### file.importFile()

### file.exportFile()