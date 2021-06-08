---
title: "Creating a filesystem in the browser"
description: "In this post I'll show you how I implemented a filesystem in the browser and how you to can do it"
date: 2020-06-04
tags: [web,javascript]
draft: false
---

Recently I have been both creating and maintaining zesterer's programing language [tao](https://github.com/zesterer/tao) playground that you can check out at https://tao.jsbarretto.com/. One of the new features that is coming is a module system, the module system allows better code organization by splitting your code into virtual units called 🥁 modules. A single file is a module (some languages can have multiple modules per file) and you can get the public exports from other modules.

## The problem

Well that's cool but the playground only supports a file right now, heck even that isn't true it's just a single text editor from where the code is directly extracted. If you look for other languages playgrounds they normally don't have multiple files support you might say but hear me out I want it I have it. There's actually a `file and directory entries api` but it's non standard and only chromium based browsers seems to support it (you can check it here https://caniuse.com/#feat=filesystem) 😢.

## Alternatives

So the filesystem api is out and we need a way to store large-ish amounts of data in the browser and that needs to be accessible from javascript, and our options are `indexedDB` and ... wait what do you mean `indexedDB` is our only option? Well i guess `indexedDB` it is.

## indexedDB

`indexedDB` is as the name says a database that works with unique indexes compared with others web apis it's fairly low level but it can quickly and efficiently handle large amounts data, just remember that it's bound by the browser eviction criteria so don't store data that you can't risk to lose use a real backend for that instead.

## Opening the DB

Let's start coding now that we know what we'll be using. The first step to use the `indexedDB` api is opening a DB it's akin to opening a database connection to a real DB except that you actually create the DB if it doesn't exist. So first we want to get the `indexedDB` object but it might not be available in every object or use a prefix (all major browsers have a prefix-less object and some have with prefix, the prefix implementations might not be complete or don't work at all so for the sake of our mental sanity we'll use only prefix-less) so let's define a check

```javascript
if (!window.indexedDB) return; // 😢 No indexedDB for us
```

Finally we may open up our DB

```javascript
var db;
var request = indexedDB.open("TestDatabase", 1);
```

The star of the show here is the `open` method it takes a string and an integer as arguments (we can omit the integer if we don't need versioning), the string is the database name and the integer is the version of the database (It is a integer if you use a float it will be rounded and it make screw you over). We also define a variable that we will use later.

All operations on `indexedDB` is asynchronous (not your standard `Promise` though it's a `IDBRequest` 🤷‍♂️) and so we need to add two callbacks `onsuccess` and `onerror`.

```javascript
// In case something goes boom
request.onerror = function(event) {
  console.log(event); // Do proper error handling kids
};

// In case something goes 🥳
request.onsuccess = function(event) {
  // Assign to the variable we created earlier it will come in handy later
  db = event.target.result /* event.target.result is our database */;
};
```

So now we have our database but that isn't all we need a third callback `onupgradeneeded`, this callback is called both when we first create the DB and when we upgrade (i.e. up our version number, downgrading is a error) and it's also where we create our `createObjectStore` that also as the name says it will be where we store our objects hence our data, so lets go ahead and define it.

```javascript
request.onupgradeneeded = function(event) {
  // Store our db onsuccess will be called after
  // so no need to assign to our global variable
  let db = event.target.result;

  // Finally we create the store
  var objectStore = db.createObjectStore("files", { keyPath: "path" });
};
```

The important bit here is `createObjectStore` it takes a name and options (Note: when upgrading all stores will remain so you only need to create new ones or delete existing stores) the are only two fields `keyPath` and `autoIncrement`, if we use `keyPath` we need to define a index manually for each object (We want this in our store because we will use the path of the file to index into the db), `autoIncrement` works like `SERIAL` in sql increasing the index automatically. And thats it for the db setup.

## Add data

What good would our filesystem be if we couldn't add files? So let's define a function `createFile`

```javascript
function createFile(path) {
  // ...
}
```

Our function will take a fully qualified path from root (I didn't tell you but we'll cheat by not defining folders) next we will actually add to the db so inside the function lets do

```javascript
// We create a transaction that will work on our files object store
let filesObjectStore = db
  // We will write to the db so it has to be readwrite
  .transaction("files", "readwrite")
  // Get a reference to our object store
  .objectStore("files");

// Our "file" it's a js object with the path and empty data
let file = {
  // The path will be our index and we will use it for querying later
  path: path,
  // The data is a Blob we initialize it an empty data and set it's MIME type to text/plain
  data: new Blob([], { type: "text/plain" }),
};

// Finally add to our store the data
let add = filesObjectStore.add(file);

// Don't forget everything is asynchronous
add.onsuccess = function(event) {
  console.log("File added 😎");
};
```

To do any operation on the indexedDB we need to do what's called a transaction to create one we use the db method `transaction` it takes a array of strings or a string as it's first argument this is the list of object stores our transaction will span across and the smaller it's the faster it can be, next we define our mode this can either be `readonly` or `readwrite` the former is faster but only the latter can mutate data and as such doing a `readonly` transaction and mutating data will throw an error, the call will return a `IDBTransaction` object from where we can call `objectStore` to get our object store and proceed to call `add` with our file to finally add our file to the store but once again this is asynchronous so it will only be completed when `onsuccess` is called but you may notice a omission where is our `onerror` callback? The errors in `indexedDB` do what it's called bubbling that means that they will go up the error chain until they find a error handler in this case it is from `add` -> `indexedDB` -> `browser` this means that having a handler in `indexedDB` is enough and avoid defining error handlers everywhere. Our file structure is very simple as you can say we have the path that we use as index and our data that we store as a `Blob`, in theory you could use a string if you only stored text but by using `Blob`s we can also store binary data and adding download functionality becomes easier (That's beyond the scope of this post), you can also add whatever data you want such as creation time.

## Querying files

Next we will both get a single file and all files that we have stored.

### Single file

To get a file by path we define the function `getFile`

```javascript
function getFile(path) {
  // We create a transaction that will work on our files object store
  let filesObjectStore = db
    // We won't write to the db so we can use readonly
    .transaction("files", "readonly")
    // Get a reference to our object store
    .objectStore("files");

  // Finally get the file on the store
  let get = filesObjectStore.get(path);

  // Don't forget everything is asynchronous
  get.onsuccess = function(event) {
    // log our file
    console.log(event.target.result);

    // To actually read the file we use the FileReader interface
    const reader = new FileReader();

    // FileReader is asynchronous so we need to listen for the loadend event
    reader.addEventListener("loadend", (e) => {
      // log the text that the file had
      const text = e.srcElement.result;
      console.log(text);
    });

    // read our blob of data as a string
    reader.readAsText(event.target.result.blob);
  };
}
```

Most of the code to get a file by path is similar to the one to create a file the exceptions is that we set the mode to `readonly` because we won't mutate data and use the `get` method on the store with our path once again this is asynchronous so we add a `onsuccess` callback. Our callback is a little different this time we now extract our file using `event.target.result` and to read it we use the `FileReader` interface.

### All files

To get all files we define the function `getAllFiles`

```javascript
function getAllFiles() {
  // We create a transaction that will work on our files object store
  let filesObjectStore = db
    // We won't write to the db so we can use readonly
    .transaction("files", "readonly")
    // Get a reference to our object store
    .objectStore("files");

  // Finally get all the files on the store
  let getAll = filesObjectStore.getAll();

  // Don't forget everything is asynchronous
  getAll.onsuccess = function(event) {
    // log our files
    console.log(event.target.result);

    // To loop over our files we can't use foreach because of some shenanigans
    // instead we use for(const file in files)
    for (const file in event.target.result) {
      // ...
    }
  };
}
```

The code is all equal to getting a single file but instead of `get` we use `getAll` also we can't loop using `foreach` when we get our result because it triggers a `TypeError` even though `event.target.result` is an array.

## Updating files

So far we can create and read files but they are empty so it isn't of great value to us, to fix that let's update our file with some data for this we will define a function `updateFile` it uses a path and a string of data (you can change this)

```javascript
function getAllFiles(path, text) {
  // We create a transaction that will work on our files object store
  let filesObjectStore = db
    // We will write to the db so we must use readwrite
    .transaction("files", "readwrite")
    // Get a reference to our object store
    .objectStore("files");

  // To update our file we define one with the same path but with the new data
  let file = {
    path: path,
    // Our new data, text is wrapped in brackets because all Blobs
    // need to have it's part be a sequence
    data: new Blob([text], { type: "text/plain" }),
  };

  // Finally put the updated file on the store
  let put = filesObjectStore.put(file);

  // Don't forget everything is asynchronous
  put.onsuccess = function(event) {
    console.log("File updated 😎");
  };
}
```

The code to update is identical to that of creating a file except that this time we put some data in the file and use `put` instead of `add` this will overwrite the file with the same path with the new data instead of throwing an error

## Deleting files

We now have a complete file system but sometimes we also want to delete some files to keep disk usage low and to cleanup a little so for that we will define the function `deleteFile` that takes the file path

```javascript
function deleteFile(path) {
  // We create a transaction that will work on our files object store
  let filesObjectStore = db
    // We will write to the db so we must use readwrite
    .transaction("files", "readwrite")
    // Get a reference to our object store
    .objectStore("files");

  // Delete the file from the store
  let delete = filesObjectStore.delete(path);

  // Don't forget everything is asynchronous
  delete.onsuccess = function(event) {
    console.log("File deleted 😎");
  };
}
```

Once again this code looks familiar to that of creating a new file but this time we don't define our file and call the `delete` method with our path.

## Fin

That was a hell of a ride but now you have a file system running in the browser (it isn't really a file system but serves the same purpose), it doesn't handle things like name collisions and instead throws everything to the console but it should be enough to get you started if you want to learn more about `indexedDB` then take a look at the [mdn docs](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API), see you next time.
