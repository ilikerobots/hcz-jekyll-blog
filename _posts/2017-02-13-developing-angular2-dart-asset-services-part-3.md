---
layout: post
title:  "Developing Angular2 Dart Asset Services, Part 3"
date:   2017-02-13  02:00:00
categories: dart angular2 software 
---

_Continued from [Part 2]({% post_url 2017-01-11-developing-angular2-dart-asset-services-part-2 %})_

# Developing Angular2 Dart Asset Services, Part 3

This article continues an effort to externalize content assets for a fictitious app. In [Part 1]({% post_url 2016-12-26-developing-angular2-dart-asset-services %}) and [Part 2]({% post_url 2017-01-11-developing-angular2-dart-asset-services-part-2 %}), we developed an external HTML content repository and an Angular service to retrieve these assets.  But our [fictive scenario]({% post_url 2016-12-26-developing-angular2-dart-asset-services %}#the-fictive-scenario) requires that we include not only external HTML but photo assets as well.  In this article, we'll provide that support in a similar strategy as was done for HTML content.  

I'll demonstrate how, relying on Dart and specifically Dart Angular's strengths, we can easily automate much of the tedium of dealing with photo assets, including license compliance and converting raw images to production web assets.

# Photo Service Interface

Just as with our HTML content provider, our interface of a Photo provider is simple:
 
```dart
abstract class PhotoService {
  Future<Photo> getPhoto(PhotoId pId);
}
``` 

The `getPhoto` method expects a PhotoId, which is simple container:

```dart
enum PhotoCategory { general, section, splash }

class PhotoId {
  String id;
  PhotoCategory type;

  PhotoId(this.id, {this.type: PhotoCategory.general});
}
```
and returns a Future Photo:

```dart
class Photo {
  PhotoId id;
  Attribution attribution;
  String url;

  Photo(this.id, this.attribution, {this.url: null});
}
```

# Image License Compliance
 
We know our app will need to use external images, but we also understand that most images are provided under license which may come with obligations on our part.  Commonly authors must be attributed or licenses linked.  Our app must abide by these rules, and therefore the image asset provider must supply this information to our app.
 
Hence, the `Photo` class above contains an `Attribution`, which dictates how attribution should be handled:
 
```dart
  class Attribution {
    String author;
    String url;
    String attributionText;
    bool attributionRequired;
  
    Attribution(this.author, { this.url: null,
    this.attributionText: "", this.attributionRequired: false});
  }
```
 
Note that information about the specific license is not included.  Our app does not particularly care what the license is, only how any specific photo must be attributed.  Later in the article, we'll handle the translation of specific licenses into attribution details further down the stack (in fact, external to this app).
   

 
# Client-based Photo Service 

Our photo provider implementation needs to provide a Photo, meaning our implementation must determine the photo URL and attribution details.  Starting with the URL part, we implement a simple solution:

```dart
@Injectable()
class ClientPhotoService implements PhotoService {
  final Client _http;
  final String _photoUrlPrefix = 'http://localhost:8082/content/';

  ClientPhotoService();

  @override
  Future<Photo> getPhoto(PhotoId pId) async {
    final Attribution attrib = null; // TODO  
    return new Photo(pId, attrib, url: _getPhotoUrl(pId.type, pId.id));
  }
  
  String _getPhotoUrl(PhotoCategory type, String id) =>
      "${_photoUrlPrefix}images/${_categoryToPath(type)}/$id.jpg";

  String _categoryToPath(PhotoCategory cat) =>
      cat.toString().replaceFirst("PhotoCategory\.", "").toLowerCase();

}
```

Following the logic in `getPhoto`,  we are simply constructing a URL based on the provided photo identifier and category.  There's no need to physically fetch anything, since a web browser is very capable of fetching external image resources.  We simply provide a URL and rely on our asset repository to conform to this structure.

We will have to fetch the licensing information via network request, however.  We dictate a convention that our asset source include a JSON license file, with a `.license` extension, for each photo. The format of the JSON is proscribed per the following example:

```json
{ 
 "author_url":"http://www.openfotos.com/view/waiting-terminal-4747","author":"cam",
 "type":"PUBLIC_DOMAIN",
 "attribution_text":"Photo by cam",
 "attribution_required":false
}
```
With this convention defined, we update the `ClientPhotoService` implementation to retrieve this information and instantiate the correct attribution details:

```dart
@Injectable()
class ClientPhotoService implements PhotoService {
  final Client _http;
  final String _photoUrlPrefix = 'http://localhost:8082/content/';

  ClientPhotoService(this._http);

  @override
  Future<Photo> getPhoto(PhotoId pId) async {
    final Attribution attrib = await _getPhotoAttribution(pId.type, pId.id);
    return new Photo(pId, attrib, url: _getPhotoUrl(pId.type, pId.id));
  }

  Future<Attribution> _getPhotoAttribution(PhotoCategory type,
      String id) async {
    final Attribution attrib = new Attribution("Unknown");
    final String url = "${_photoUrlPrefix}images/${_categoryToPath(
        type)}/$id.jpg.license";
    try {
      final Response response = await _http.get(url);
      if (response.statusCode == 200) {
        final Map<String, dynamic> json = _extractData(response);
        if (json != null) {
          attrib.author = json['author'];
          attrib.url = json['author_url'];
          attrib.attributionText = json['attribution_text'];
          attrib.attributionRequired = json['attribution_required'];
        }
        return attrib;
      } else {
        _handleError(url, "Response ${response.statusCode}");
        return attrib;
      }
    } catch (e) {
      _handleError(url, e.runtimeType);
      return attrib;
    }
  }

  String _getPhotoUrl(PhotoCategory type, String id) =>
      "${_photoUrlPrefix}images/${_categoryToPath(type)}/$id.jpg";

  String _categoryToPath(PhotoCategory cat) =>
      cat.toString().replaceFirst("PhotoCategory\.", "").toLowerCase();

  dynamic _extractData(Response resp) =>
      JSON.decode(UTF8.decode(resp.bodyBytes));

  void _handleError(String url, dynamic e) {
    //TODO handle error
  }
}
```

The implementation of `_getPhotoAttribution` is similar to the `ClientContentService` implementation of `getContent` discussed in 
[Part 2]({% post_url 2017-01-11-developing-angular2-dart-asset-services-part-2 %}).  We fetch a remote file, in this case the JSON license file, using an injected Client.  Instead of sanitizing response as we did for HTML content, we simply decode the JSON and assign `Attribution` fields parameters from the decoded data.

Our app is now free to inject and use this photo asset provider.  Our app will decide exactly how to properly attribute photos based on the simple Attribution instructions provided from the service.  

# An External Photo Repository

While technically we've completed our obligations to our app, we don't actually have any photo assets to show or test with.  We can construct alternate providers for use in testing (there are [a few in the repo](https://github.com/ilikerobots/intallinn/tree/master/lib/service/photo)), but we can also make our own photo repo for use with `ClientPhotoService`.  However, resizing and cropping photos and hand-writing JSON certainly isn't our style, so what can an impatient Dart author do?

Recall in [Part 2]({% post_url 2017-01-11-developing-angular2-dart-asset-services-part-2 %}) we built a HTML content repo that would allow copywriters to author in Markdown.  Our content repo, with some custom transformers, would automatically convert these markdown files into production-ready HTML.  We'll use a similar approach here, allowing our photographers to simply store raw images and specify basic licensing info.  Our repo will take care of the work of transforming these into production-ready assets.

To start, we create a new repo and define our photo assets with all pertinent info in a yaml file `assets/image/section_photo.yaml`.  An excerpt:

```yaml
photos: 
 - kittens:
   in_filename: section/kitten_2017_02_23.png
   out_filename: kittens.jpg
   license: 
     type: GFDL
     author: Bart Dartner
     author_url: https://dartlang.org
 - doggo:
   in_filename: section/d.jpg
   out_filename: doggo.jpg
   license: 
     type: CC_BY_SA_30
     author: Dart Bartner
     author_url: https://google.com
```
Here we've described two input photo assets, along with our their desired output name and any pertinent licensing info.

We now define the licenses used above, including the necessary attribution requirements for each.  Example:

```dart
abstract class License {
  LicenseType get type;
  String getAttribution(String author);
  bool get attributionRequired => true;
}

class CcBySa30License extends License {
  LicenseType get type => LicenseType.LICENSE_CC_BY_SA_30;
  String getAttribution(String author) => "Photo by $author";
  bool get attributionRequired => true; 
}
```

There are additional licenses and some helpers, such as a `LicenseFactory`, that I omit here for clarity.  See [source](https://github.com/ilikerobots/intallinn_content/tree/sterile/lib/license) for more detail.

Finally, we create a Dart bin script to generate our production assets and license JSON from the above yaml.  Our script will loop through each entry in the yaml, for each performing cropping/scaling/resizing using the [ImageMagic convert binary](https://linux.die.net/man/1/imagemagick) and writing the JSON license file.  


```dart
// constants omitted for readability

main() async {
  String inPath = path.absolute(INPATH);
  String outPath = path.absolute(OUTPATH);

  Directory outDir = new Directory(outPath);
  if (!outDir.existsSync()) {
    await outDir.create(recursive: true);
  }

  String contents = await new File(YAML_PATH).readAsString();
  var yaml = loadYaml(contents);

  Directory tmpDir = await Directory.systemTemp.createTemp(tmpSubdirName);

  await Future.forEach(yaml['photos'], (Map f) async {
    print("Converting $INPATH${f['in_filename']} " +
        "to $OUTPATH${f['out_filename']}");

    String iFile = path.join(inPath, f['in_filename']);
    String oFile = path.join(outPath, f['out_filename']);
    String tmpBaseName = path.basenameWithoutExtension(iFile);
    String tmpFileName = "${tmpDir.path}/$tmpBaseName.cache.jpg";

    new File(iFile).copySync(tmpFileName); //copy to tmp location

    await doImageConvert(CONVERT_SCRIPT, ['-quality', '100'], tmpFileName);
    await doImageConvert(CONVERT_SCRIPT,
        ['-thumbnail', FINAL_GEOMETRY, '-quality', '$QUALITY'], tmpFileName);

    new File(tmpFileName).copySync(oFile); //copy to final location
    writeLicenseFile(f['license'], "$oFile.license");
  });
  tmpDir.delete(recursive: true);
}


Future<Null> doImageConvert(final String cmd, final List<String> args,
    final String filename) async {
  List<String> thisArgs = new List.from(args)
    ..add(filename)..add(filename);
  await exitOnFail(Process.run(cmd, thisArgs));
  return;
}

Future<Null> writeLicenseFile(Map<String, String> licenseAttribs,
    String fName) async {
  License l = new LicenseFactory().getLicenseFromString(licenseAttribs['type']);
  Map<String, dynamic> out = {}..addAll(licenseAttribs);
  out['attribution_text'] = l.getAttribution(licenseAttribs['author']);
  out['attribution_required'] = l.attributionRequired;
  await new File(fName).create()
    ..writeAsString(JSON.encode(out));
}

Future<Null> exitOnFail(Future<ProcessResult> resF) async {
  ProcessResult res = await resF;
  if (res.exitCode != 0) {
    stderr.writeln(res.stderr);
    exit(res.exitCode);
  }
  return;
}
```

The example above is fairly simple, but the script could be updated to apply effects, watermark images, modify image metadata, etc.

Now, if we place source images in `assets/photo/section` and update our `section_photos.yaml` accordingly, then when we run `pub bin/gen_section_photos` our processed images and JSON license info will be deposited into the `web/content/images/section` directory, just like our photo asset provider expects. 

We can then `pub serve` this repo for development use or, for production, simply copy our generated assets to a static web server.



# Conclusion

In these three articles, I've illustrated how relying on the strengths of Angular Dart and Dart in general can greatly boost productivity and flexibility.  

Dart and Angular Dart's features, including dependency injection, futures, transformers, and robust web-centric libraries make web development fun again.

# Source

These articles were based on work I did creating [inTallinn](https://intallinn.ee), a visitors' guide to my home city Tallinn, Estonia.  See the [full source code of inTallinn](https://github.com/ilikerobots/intallinn), which expands on the contents of these articles.




