# HyperDart

## Motivation

Hyperdart is a Dart implementation of core concepts taken from Rails and Hypertack. Hyperdart does not seek to  re-invent the wheel, but seeks to use as much of the incredible functionality provided by Dart, Flutter and other Dart packages, but augment that stack with some of the missing magical elements any Rails or Hyperstack developers love and use.

Arguably, Dart is the most Isomorphic language yet there is no clear Isomorphic Dart approach. Dart code  can run anywhere, but there is a lack of a clear approach as to how to handle core Isomorphic concepts such as :

+ Isomorphic Models - same models being shared across all implementation
+ Database schema versioning - very complicated topic when every client has a copy of a schema-less database
+ Sharing Models between languages in a language agnostic way
+ State management - Hyperstack's approach to state management is far better (INHO) that any approach in the React world. Its easy to understand and implementand vastly less complicated than cumbersome concepts like Bloc or Redux. 

## Scope

As stated earlier, Hyperdart does not want to re-invent any wheels, so where good options exist, this project will reference them and seek to be a best-practice guide. Where there is no alternative this project will provide one.

### User Interface 
for Mobile most certainly use [Flutter](https://flutter.dev/). It is an outstanding framework with a vibrant and engaged community. For Web development, there are two viable options - either [Flutter Web](https://flutter.dev/web) which is still in beta but being actively developed or one of the two React wrapper projects which are similar to Hyperstack. [React Dart](https://pub.dev/packages/react) for an experience which is as close to React as possible or [OverReact](https://pub.dev/packages/over_react) which is based on React Dart but adds additional functionality but does requires code generation.

### State Management
Flutter itself is not prescriptive as to where the business logic should be kept and the tempation is therefore to overload the Widgets. This is a common problem in reactive style programming. There are highly opinionated approaches like [Bloc](https://pub.dev/packages/flutter_bloc) or [Redux](https://pub.dev/packages/flutter_redux), both of which have steep learning curves and vast amounts of builderplate code. My personal favourate approach is [GetX](https://pub.dev/packages/get) which is a rapidly evolving library which combines navigation, dependency injection and state management.

### Isomorphic Models

In th Dart/Flutter world, Models are simply classes that hold data, constructed as streams are de-serialized. In the Rails/Hyperstack world, Models are active objects which wrap the database. I assure you, there is nothing simplier than `model.save()` and in the abasense of it your code becomes infinitely more complex and your life less enjoyable. 

HyperDart implements HyperModel - an ORM wrapper base class which extends any model to provide the following methods:

+ A simple constructor `HyperModel(Map data, String path`)
+ Saving and creating `Future<void> save({String collection})`
+ Database deletion `Future<void> delete()` 
+ Getting a collection off this object `CollectionReference get collection` - this powerfully allows HyperModels to be nested

HyperModel is in active development, and will eventually be published on `pub.dev` and in this repo. Until then, here is the implementation of the base class:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:cloud_firestore/cloud_firestore.dart' as fs;

class HyperModel {
  String _uid;
  String _path;
  Map _internalData;

  String _thisCollectionName;
  static fs.DocumentReference get _rootDocumentRef => HyperModel._db.document(config["rootPath"]);
  static Firestore _db = config["firestoreInstance"];

  /// The config map must contain the rootPath and the firestore instance
  /// These static values must be set once for the application (in main.dart) before this model can be used
  /// Example:
  /// `HyperModel.config = {"rootPath": "/projects/${Constants.projectID}", "firestoreInstance": Firestore.instance};`
  static Map config;

  /// Any class can inherit from HyperModel but the following values are needed for operation
  /// ----- required:
  /// `Map data` - any map of data elements which are accessible as attributes
  /// ----- optional:
  /// `String uid` - unique ID for this object (will be used to name the object in the database)
  /// `String path` - if this object already exists in a database, this must the the url to this object
  /// NB: The path is only needed if the object is already saved
  /// `String collectionName` - the name of the collection where this object will be saved
  /// NB: The collectionName is only needed if the object has not been saved
  /// `bool ignoreRootPath` allows you to bypass the pre-set root path and access collections from the FS root
  HyperModel(Map data, {String uid, String path, String collectionName, bool ignoreRootPath: false}) {
    assert(config.isNotEmpty);
    if (ignoreRootPath == false) assert(config["rootPath"] != null, "must set HyperModel config rootPath");
    assert(config['firestoreInstance'] != null, "must set HyperModel config firestoreInstance");

    // Must provide either path (if saved) or collectionName (to be able to save)
    if (path == null) assert(collectionName != null, "If this object is unsaved then a collectionName is needed");
    if (collectionName == null) assert(path != null, "There must be a path if there is no collectionName");

    _uid = uid;
    _internalData = data;

    _thisCollectionName = collectionName;
    _path = path;
  }

  /// `attributes` will give you a Map of the internal data of the model
  /// you can read and write to this Map
  /// for example `model.attributes['firstName'] = 'Sally'` will update/create an attribute
  /// to get an attribute, use a defined getter on the child object `print(model.firstname)` (if one is defined)
  /// otherwise use the getter `print(model.attributes['firstName'])`
  ///
  /// Updating attributes:
  /// when you update an attribute there is no automatic save, you will need to call `model.save()` tp save
  /// To save and update an attribute, use `model.saveAttribute('firstname', 'Sally')` which will update and save
  Map get attributes => _internalData;
  get uid => _uid;

  String toString() {
    return _internalData.toString();
  }

  /// This instance method will return a collection off this object
  /// for example `humans/123/onboarding` where this collection would be humans, the uid is 123
  /// and `collectionName` would be onboarding
  ///
  /// The collection method is useful to get a stream to a document or a collection
  /// For example:
  ///   static Stream<ContentPackModel> stream(String packName) {
  ///     return HyperModel.projectRootDocument
  ///         .collection("content-packs")
  ///         .document(packName)
  ///         .snapshots()
  ///         .map((snap) => ContentPackModel(
  ///               uid: snap.documentID,
  ///               path: snap.reference.path,
  ///               data: snap.data,
  ///             ));
  ///   }
  CollectionReference collection(String collectionName) {
    assert(_thisCollectionName != null, "collectionRef must be set to use .collection");
    // return HyperModel.documentRef(_collectionName, uid).collection(collectionName);
    return _rootDocumentRef.collection(_thisCollectionName).document(uid).collection(collectionName);
  }

  /// Creates (saves) an instance of this object in its collection
  /// This method is similar to save vut unlike save, the _path does not yet have to be set
  /// `String uid` to save the document with a specific
  Future<void> save({String uid}) async {
    fs.DocumentReference document;

    if (_path == null) {
      assert(_thisCollectionName != null, "collectionRef must be set to save");
      // this is a new unsaved model
      // use the uid of one is passed in
      if (uid != null) _uid = uid;
      if (_uid != null) {
        document = _rootDocumentRef.collection(_thisCollectionName).document(_uid);
      } else {
        document = _rootDocumentRef.collection(_thisCollectionName).document();
        _uid = document.documentID;
      }
      _path = document.path;
    } else {
      // this is an existing saved model, and this is a re-save, so just get the document
      document = _db.document(_path);
    }

    return document.setData(Map<String, dynamic>.from(_internalData), merge: true); // <- Flutter
  }

  /// returns the root document for this project - all other collections must be under here for this project
  /// for example the projectRootDocument would be `\projects\sandbox` and any subsequent collections would hang off that
  static DocumentReference get projectRootDocument {
    assert(_rootDocumentRef != null);
    return _rootDocumentRef;
  }

  /// returns the root document instance
  static Firestore get fsRoot {
    return _db;
  }

  /// Saves an individual attribute. If this object is being streamed this update will be pushed to the stream
  Future<void> saveAttribute(String attribute, dynamic value) async {
    assert(_path != null, "path must be set to saveAttribute");
    _internalData[attribute] = value;
    // store.doc(_path).update(data: {field: value}); <- JS
    return _db.document(_path).updateData({attribute: value}); // <- Flutter
  }

  /// Deletes an individual attribute from memory and also from disk
  Future<void> deleteAttribute(String attribute) {
    assert(_path != null, "path must be set to deleteField");
    // store.doc(_path).update(data: {attribute: fs.FieldValue.delete()}); <- JS
    _internalData.remove(attribute);
    return _db.document(_path).updateData({attribute: fs.FieldValue.delete()}); // <- Flutter
  }

  /// Deletes this object from the database, but leaves it in memory
  Future<void> delete() {
    assert(_path != null, "path must be set to delete");
    // store.doc(_path).delete(); <- JS
    return _db.document(_path).delete(); // <-  Flutter
  }
}

```

If anyone would like to participate in this project please reach out to me on Slack. Happy Darting! 
