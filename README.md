# imgix-blueprint

A blueprint for creating an imgix library in any language.


## About

This document is meant to serve as a resource for implementing native libraries for use with the [imgix URL API](https://www.imgix.com/docs/reference). Almost all libraries in their individual languages have a set of similar concerns: the library must be able to build imgix URLs, [secure imgix URLs](https://www.imgix.com/docs/tutorials/securing-images), and handle the corner cases that result.

This document will not cover framework-level integrations in different languages. This document is primarily concerned with building imgix API URLs reliably.

imgix cares very much about providing first-class, idiomatic support for as many languages as possible. If you create a library for a language that is not supported, please [get in touch](mailto:support@imgix.com) if you would like it to be supported officially.

## Existing Libraries

Official libraries exist in the following languages:

- [Java](https://github.com/imgix/imgix-java)
- [NodeJS](https://github.com/imgix/imgix-core-js)
- [PHP](https://github.com/imgix/imgix-php)
- [Python](https://github.com/imgix/imgix-python)
- [Ruby](https://github.com/imgix/imgix-rb)

Unofficial libraries are available in the following languages:

- [C#](https://github.com/raynjamin/Imgix-CSharp)
- [Go](https://github.com/parkr/imgix-go)
- [Objective-C](https://github.com/soffes/imgix-objc)
- [Swift](https://www.github.com/hodinkee/iris)
- [Elixir](https://github.com/ianwalter/imgex)

If you have an imgix library that you would like included here, please [open a Pull Request](https://github.com/imgix/imgix-blueprint/pulls).

There is a complete list of libraries (including framework-specific libraries) in [the imgix documentation](https://www.imgix.com/docs/libraries).


## Naming

If it is idiomatic for the language, we recommend naming each library `"imgix-" + language_name`, e.g. `imgix-rb` or `imgix-php`.


## Versioning

All imgix libraries must follow [Semantic Versioning](http://semver.org/).

## Hello, World!

The simplest transformation for a library should be able map a path to an imgix source.

Given the origin path:

```
/users/1.png
```

A given imgix library should be able to turn that into:

```
https://my-social-network.imgix.net/users/1.png
```


## Protocols

imgix recommends using the HTTP over TLS (https:) in all cases. All imgix sources are HTTPS-enabled.

Given recommendations by Paul Irish and Ilya Grigorik, please do not use HTTP or protocol-relative URLs.

See:

- [The Protocol-relative URL](http://www.paulirish.com/2010/the-protocol-relative-url/), Paul Irish
- [Is TLS Fast Yet?](https://istlsfastyet.com/), Ilya Grigorik


## Web Proxy Sources

Web Proxy Sources are very powerful parts of the imgix URL API. While Amazon S3 and Web Folder sources all assume the same origin, Web Proxy Sources are able to proxy any publicly-accessible URL. Because of this, imgix requires that all Web Proxy URLs be signed.

This is also a point that can trip many library authors up. The URL-to-be-proxied is the path component of an imgix URL. While

```
https://my-social-network.imgix.net/http://avatars.com/john-smith.png # Don't do this
```

will work with imgix, imgix recommends that you **do not** merely append the origin URL like this. This method will begin causing issues if and when the origin URL has query parameters of its own.

Thus, imgix recommends URI-encoding the URL-to-be-proxied like so:

```
https://my-social-network.imgix.net/http%3A%2F%2Favatars.com%2Fjohn-smith.png
```

**Note**: Web Proxy URLs will also need to be signed. Please see the [Securing URLs section below](#securing-urls).


## URL Parameters

The imgix URL API is a powerful way to manipulate images. The primary method of doing this is the [extensive set of URL parameters](https://www.imgix.com/docs/reference). These URL parameters are able to change the size, crop, file format, and much more.

These URL parameters should be appended to valid imgix URLs. For example, let's say we wanted to resize an image to be 400 pixels wide and 300 pixels tall, given this URL:

```
https://my-social-network.imgix.net/users/1.png
```

The library should then generate:

```
https://my-social-network.imgix.net/users/1.png?w=400&h=300
```

### Parameter Encoding

All parameters and their values should be URI encoded before generating the output URL. This helps avoid encoding errors, as well as potential XSS vulnerabilities when dealing with user-generated content. If the library is passed an image URL of `users/1.png` and a parameter of `hello world` with a value of `this/seems… pretty sketchy! 😁`, it should generate the following url:

```
https://my-social-network.imgix.net/users/1.png?hello%20world=this%2Fseems%E2%80%A6%20pretty%20sketchy!%20%F0%9F%98%81
```

If the library is passed any parameter ending in `64`, it should automatically encode that parameter's value as an RFC4648 "base64url" string as described in the [base64url encode problematic parameters](#base64url-encode-problematic-parameters) section of this document. For example, the same user image as above with the `txt64` parameter and value `this/seems… pretty sketchy! 😁` should result in the following URL:

```
https://my-social-network.imgix.net/users/1.png?txt64=dGhpcy9zZWVtc-KApiBwcmV0dHkgc2tldGNoeSEg8J-YgQ
```

If working in JavaScript, it's important to note that the built-in `atob` and `btoa` methods are not URL safe, and only support [Latin-1](https://en.wikipedia.org/wiki/Latin-1_Supplement_(Unicode_block)) characters. Because of these limitations, it may be useful to use a third-party tool for encoding and decoding Base64 in JavaScript, such as [js-base64's](https://github.com/dankogai/js-base64) `Base64.encodeURI` method.

All parameters and values passed to the library must be unencoded.


## Securing URLs

imgix recommends securing all URLs, although it is not required for Amazon S3 and Web Folder Sources. Securing URLs prevents others from using one of your Sources maliciously, say to use your imgix Source as a CDN for a separate site.

imgix URL signatures are represented by the special `s` parameter.  The value is an MD5 hash of the data pertaining to the URL itself and the imgix Source. Note that the hash value must be in lowercase hex. In a secured imgix URL, the `s` parameter **must** be the last parameter.

This parameter is generated as follows in Ruby:

```ruby
signature_base = token + path
signature_base += query if !query.nil?
Digest::MD5.hexdigest(signature_base)
```

Here are the following definitions of each variable in the above example:

- `token`: The alphanumeric Secure URL Token pertaining to the specific Source. It can be found in the [imgix web dashboard](https://webapp.imgix.com/source).
- `path`: The path of component of the final imgix URL including the leading slash, e.g. `/users/1.png` or `/http%3A%2F%2Favatars.com%2Fjohn-smith.png`.  Special characters in the path (for example UTF-8 encoded codepoints) must remain percent encoded.
- `query`: The query string of the imgix URL parameters, leading with the `?`, e.g. `?w=400&h=300`. If there are no query parameters, this should be left out of the signature base.

<a name="base64url-encode-problematic-parameters"></a>
## base64url encode problematic parameters

When dealing with complex inputs, encoding can be difficult to properly implement. To help with this, almost every parameter in imgix has a Base64 alias, which allows the values to be encoded using the "base64url" encoding with a URL and filename safe alphabet ([RFC 4648](https://en.wikipedia.org/wiki/Base64#RFC_4648)). These parameters are keyed by appending `64` to the end of the parameter name. Thus `txt` becomes `txt64`.

Please note that, per RFC 4648, "base64" and "base64url" encodings are not interchangeable.  Many libraries only implement "base64" encoding.  Encoding parameter values using "base64" encoding WILL cause your requests to fail ... be sure you check for and know the difference!

When writing libraries, the output of any value passed to a parameter ending in `64` must be "base64url" encoded, with any padding characters (equals signs and newlines [`=`, `\n`]) removed.

Here’s an example:

```
?txt=Hello,+World!
```

is the same as

```
?txt64=SGVsbG8sIFdvcmxkIQ
```

Both of these will overlay "Hello, World!" onto an image using the imgix [`txt` parameter](https://docs.imgix.com/apis/url/text/txt).

Let's see how that works in Ruby:

```ruby
client = Imgix::Client.new(host: 'static.imgix.net')
client.path('base.png').to_url(mark64: 'https://assets.imgix.net/logo.png')
# => "https://static.imgix.net/base.png?mark64=aHR0cHM6Ly9hc3NldHMuaW1naXgubmV0L2xvZ28ucG5n"
```

This pattern helps ensure that the users of your libraries never encounter encoding issues while generating imgix URLs.

When consuming a library, imgix recommends always using the Base64 variant of the following parameters:

- [`mark`](https://docs.imgix.com/apis/url/watermark/mark)
- [`blend`](https://docs.imgix.com/apis/url/blending/blend)
- [`txt`](https://docs.imgix.com/apis/url/text/txt)
- [`txtfont`](https://docs.imgix.com/apis/url/text/txtfont)

### Examples

The following are a few examples for securing URLs, which library authors should use as a spot check:

#### Simple Paths

Given:

- Path: `/users/1.png`
- Secure URL Token: `FOO123bar`
- No imgix parameters

The resulting signature should be:

```
6797c24146142d5b40bde3141fd3600c
```

This makes the final URL:

```
https://my-social-network.imgix.net/users/1.png?s=6797c24146142d5b40bde3141fd3600c
```

#### Fully-Qualified URLs

Given:

- Origin URL: `http://avatars.com/john-smith.png`
- Path: `/http%3A%2F%2Favatars.com%2Fjohn-smith.png`
- Secure URL Token: `FOO123bar`
- No imgix parameters

The resulting signature should be:

```
493a52f008c91416351f8b33d4883135
```

This makes the final URL:

```
https://my-social-network.imgix.net/http%3A%2F%2Favatars.com%2Fjohn-smith.png?s=493a52f008c91416351f8b33d4883135
```

#### Simple Paths with imgix URL API parameters

Given:

- Path: `/users/1.png`
- Secure URL Token: `FOO123bar`
- The following imgix parameters: `w=400` and `h=300`

The resulting signature should be:

```
c7b86f666a832434dd38577e38cf86d1
```

This makes the final URL:

```
https://my-social-network.imgix.net/users/1.png?w=400&h=300&s=c7b86f666a832434dd38577e38cf86d1
```

Or if your language likes to alphabetize the params:

```
https://my-social-network.imgix.net/users/1.png?h=300&w=400&s=1a4e48641614d1109c6a7af51be23d18
```

#### Fully-Qualified URLs with imgix URL API parameters

Given:

- Path: `/http%3A%2F%2Favatars.com%2Fjohn-smith.png`
- Secure URL Token: `FOO123bar`
- The following imgix parameters: `w=400` and `h=300`

The resulting signature should be:

```
61ea1cc7add87653bb0695fe25f2b534
```

This makes the final URL:

```
https://my-social-network.imgix.net/http%3A%2F%2Favatars.com%2Fjohn-smith.png?w=400&h=300&s=61ea1cc7add87653bb0695fe25f2b534
```

Or if your language likes to alphabetize parameters:

```
https://my-social-network.imgix.net/http%3A%2F%2Favatars.com%2Fjohn-smith.png?h=300&w=400&s=a201fe1a3caef4944dcb40f6ce99e746
```
