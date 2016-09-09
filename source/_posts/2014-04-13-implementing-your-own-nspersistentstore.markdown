---
layout: post
title: "Implementing your own NSPersistentStore"
date: 2014-04-13 13:18:00 +0200
comments: true
categories: opensource
published: false

---

## A quick overview

The first thing that might come to mind when reading the headline is: Why? There already are a couple of stores available for you to choose from. And of course, as long as these fit your needs, you definitely don't want to do the heavy lifting your self and implement your own store.

### In-Memory

The In-Memory store `NSInMemoryStoreType` is a store that, as the name suggests, stays completely in memory. It is never written to disk. If you or the system completely quits you app, all your data is gone. You typically would use this as a cache or store other data that can easily be reproduced.

### Binary

The binary store `NSBinaryStoreType` is a store that is written on disk in one binary file. It is loaded into memory at once and it is written to disk at once. There is no incremental loading at all. You typically want to use this kind of store only for very small data. The benefit is that accessing data from it is pretty fast, once the store has loaded all the data from the disk.

### SQLite

The `NSSQLiteStoreType` is what will match most of the database-like approaches. It stores its content into an sqlite database and can incrementally pull data out of it or push data into it. It is still pretty fast but you don't have to load all the data into memory. If you execute a fetch and the object is not already in the context that is executing the fetch or any parent context, it will load this particular object from disk without touching all the other objects that are stored in the sqlite database as well.

### XML

The XML store type `NSXMLStoreType` is not available on iOS. It uses an XML data structure to store its elements. It is completely loaded into memory, like the binary store type. The big difference here is, that you can open XML stores with any other tool that is capable of reading XML data.

## Your store

The XML store shows why you maybe want or need to implement your own store: If you have a given data format and want to operate on this data but don't want to miss the convenience of CoreData you can either choose to convert the data and store it into one of the given store types or you can write your own store and directly operate on your data. The last approach is very useful if there are other applications that need to use the data or if there is somebody who is constantly managing the data by hand, e.g. in a csv file. Instead of always converting between an existing CoreData store and the csv file, you can create a store that is capable of reading and writing this csv file directly.

### NSPersistentStoreCoordinator

When using CoreData, you don't use the NSPersistentStore your self. Instead, you talk to your NSManagedObjectContext which walks up the context context hierarchy - as long as there is one - and finally talks to the NSPersistentStoreCoordinator. The NSPersistentStoreCoordinator hands the objects to the NSPersistentStore. The coordinator however is capable of handling multiple stores and - as long as each entity type is only assigned to one store - automatically decides which object goes to which store. This behaviour makes room for a couple of other interesting use cases. One example: You are writing a twitter client and need to store a number of twitter timelines in your application. You save these posts in CoreData as it is really easy to set up a table view with CoreData and an NSFetchedResultsController. However, the credentials of the user accounts in your application should be stored in a secure way, so you put them in the KeyChain. You are always dealing with two data storing approaches. With your own custom store, you can create a User entity that holds the credentials for one user and then assign this entity to a store that writes all the informations into the keychain whereas all the other data is stored in an sqlite store type. If you choose this approach, CoreData handles this process completely transparent for the rest of your application. You create User objects and tweet objects in the same context, you save them and the User entity is automatically stored in the key chain.