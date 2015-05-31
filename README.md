# imgix-blueprint

A blueprint for creating an imgix library in any language.

## About

This document is meant to serve as a resource for implementing native libraries for use with the [imgix URL API](https://www.imgix.com/docs/reference). Almost all libraries in their individual languages have a set of similar concerns: the library must be able to build imgix URLs, [secure imgix URLs](https://www.imgix.com/docs/tutorials/securing-images), and handle the corner cases that result.

This document will not cover framework-level integrations in different languages. This document is primarily concerned with building imgix API URLs reliably.

imgix cares very much about providing first-class, idiomatic support for as many languages as possible. If you create a library for a language that is not supported, please [get in touch](mailto:support@imgix.com) if you would like it to be supported officially.

## Naming

If it is idiomatic for the language, we recommend naming each library `"imgix-" + language_name`, e.g. `imgix-rb` or `imgix-php`.

## Hello, World!

The simplest transformation for a library should be able map a path to an imgix source.

Given the path:

```
/users/1.png
```

A given imgix library should be able to turn that into:

```
https://my-social-network.imgix.net/users/1.png
```

## Existing Libraries

Official libraries exist in the following languages:

- [Ruby](https://github.com/imgix/imgix-rb)
- [Python](https://github.com/imgix/imgix-python)
- [Java](https://github.com/imgix/imgix-java)
- [PHP](https://github.com/imgix/imgix-php)

Unofficial libraries are available in the following languages:

- [Objective-C/Swift](https://github.com/soffes/imgix-objc)

If you have an imgix library that you would like to see included, please [open a Pull Request](https://github.com/imgix/imgix-blueprint/pulls).

## Protocols

imgix recommends using the HTTP over TLS (https:) in all cases. All imgix sources are HTTPS-enabled and performant.

Given recommendations by Paul Irish and Ilya Grigorik, please do not use HTTP or protocol-relative URLs.

See:

- [The Protocol-relative URL](http://www.paulirish.com/2010/the-protocol-relative-url/), Paul Irish
- [Is TLS Fast Yet?](https://istlsfastyet.com/), Ilya Grigorik

## Web Proxy Sources

Web Proxy Sources are very powerful parts of the imgix URL API. While Amazon S3 and Web Folder sources all assume the same origin. Web Proxy Sources are able to proxy any publicly-accessible URL. Because of this, imgix requires that all Web Proxy URLs be signed.

This is also a point that can trip many library authors up. The URL-to-be-proxied is the path component of an imgix URL. While

```
https://my-social-network.imgix.net/http://avatars.com/john-smith.png
```

will work with imgix, imgix recommends that you **do not** merely append the origin URL like this. This method will begin causing issues if and when the origin URL has query parameters of its own.

Thus, imgix recommends URI-encoding the URL-to-be-proxied like so:

```
https://my-social-network.imgix.net/http%3A%2F%2Favatars.com%2Fjohn-smith.png
```

**Note**: Web Proxy URLs will also need to be signed. Please see the Securing URLs section below.

## URL parameters

The imgix URL API is a powerful way to manipulate images. The primary method of doing this is the [extensive set of URL parameters](https://www.imgix.com/docs/reference). These URL parameters are able to change the size, crop, file format, and much more.

These URL parameters should be appended to valid imgix URLs. For example, let's say we wanted to resize an image to be 400 pixels wide and 300 pixels tall, given this URL:

```
https://my-social-network.imgix.net/users/1.png
```

The library should then generate:

```
https://my-social-network.imgix.net/users/1.png?w=400&h=300
```

## Securing URLs

imgix recommends securing all URLs, although it is only required for Amazon S3 and Web Folder Sources. Securing URLs prevents others from using one of your Sources maliciously, say to use your imgix Source as a CDN for a separate site.

imgix URL signatures are represented by the special `s` parameter, and are a checksum of data pertaining to the URL itself and the imgix Source.

This parameter is generate as follows in Ruby:

```ruby
signature_base = token + path
signature_base += query if !query.nil?
Digest::MD5.hexdigest(signature_base)
```

Here are the following definitions of each variable in the above example:

- `token`: The Secure URL Token pertaining to the specific Source. It can be found in the [imgix web dashboard](https://webapp.imgix.com/source).
- `path`: The path of component of the final imgix URL including the leading slash, e.g. `/users/1.png` or `http%3A%2F%2Favatars.com%2Fjohn-smith.png`.
- `query`: The query string of the imgix URL parameters, leading with the `?`, e.g. `?w=400&h=300`.

