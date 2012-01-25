[![Build Status](https://secure.travis-ci.org/jweir/ScaleDown.png)](http://travis-ci.org/jweir/ScaleDown)

The image scaling server used by [Fame Driver](http://famedriver.com).
It supports dynamic sizes, image cropping and color conversion (CMYK to RGB).
It only supports images on the same machine (this is not a distributed scaler).

ScaleDown
==========
So you want to scale an image?
------------------------------

You have an image on `example.com`. It has a public path of `/images/john/picture.png`.
You want it scaled to fit in a 400x400 pixel box.  ScaleDown is running on the subdomain `images`.

    http://server/images/john/scaled/400x400/picture.png?HMAC_SIGNATURE

The scaled file is saved in a public path identical to the request. It will be statically served on the next request.

Geometry, Labels and Cropping
==============================

There are two methods of requesting a scaled image: geometry or label.

Geometry
--------

A geometry defines a box the image must fit within: `WIDTHxHEIGHT`.

```sh
# scale to a 300 pixel width box by 900 pixel high box
http://server/images/john/scaled/300x900/picture.png?HMAC_SIGNATURE
```

Either, but not both, dimensions may use the keyword `auto`. This will scale the image to fit the defined dimension.

```sh
# scale to a 500 pixel wide box, of any height
http://server/images/john/scaled/500xauto/picture.png?HMAC_SIGNATURE
```

When using a geometry an HMAC is required (see below).
    
Labels
------

A label is nothing more than a predefined geometry. It does not require an HMAC signature.

```sh
# A label called thumbnail has been defined as 100x100
http://server/images/john/scaled/thumbnail/picture.png
```

*Labels generate the same file as geometry request. A symbolic link is used to statically server it.*

Crop
----

To crop an image include the `-crop` option.  The image will be scaled and cropped to fit the geometry. This is a simple crop positioned on the center and top of the image.

Both geometry and labels accept the `-crop` option.

```sh
http://server/images/john/scaled/thumbnail-crop/picture.png
http://server/images/john/scaled/100x100-crop/picture.png
```

Info
----
There is a very simple `/info` function for getting image dimensions. It just returns a string with the WIDTHxHEIGHT of the original image.

    http://server/images/logo.png/info


HMAC & Geometry
==============

An HMAC signature is required to prevent a DOS attack. This ensures that your URL was generated by a trusted source.

HMAC requires a shared key between the application generating the URL and the ScaleDown server.

Ruby HMAC URL Generator
------------------

```ruby
require 'ruby-hmac'

def signed_image_url(absolute_path, filename, geometry)
  shared_secret = "secret"
  hmac = HMAC::SHA1.new(shared_secret).update([absolute_path, 'scaled', geometry, filename].join("/")).to_s[0...8]
  "http://server#{[absolute_path, 'scaled', geometry, CGI.escape(filename)].join("/")}?#{hmac}"
end
```

Node.js HMAC URL Generator
---------------------

```javascript
// Uses the Node.js crypot library
var crypto = require('crypto')

function hmac(string){
  var shared_secret = "secret"
  return crypto.createHmac('sha1',shared_secret).update(string).digest('hex').substr(0,8)
}

function signed_image_url(absolute_path, filename, geometry){
  signature = hmac( [ absolute_path, '/scaled', "/" + geometry, "/", filename].join("") )
  return [global.$FD.assetHost,  absolute_path, '/scaled', "/", geometry, "/",escape(filename)].join("") + "?"+ signature
}
```

URL Schema
==========

The schema is

    http://:host/:path_to_file/scaled/:target/:filename?:hmac_signature

    :host           the address running ScaleDown
    :path_to_file   the public path of the original file
    `scaled`        keyword
    :target         a label or width x height. An optional `-crop` can accompanie either
    :filename       the filename of the original image to scale
    :hmac_signature security measure to validate the request(not required for labels)


Supported Image Formats
=======================

ScaleDown will handle any image that Image Magick can process.  But there are some rules:

* PNGs remain PNGs
* All other images are converted to JPG and issued a 301 redirect

Installation & Configuration
==============================

    gem install scale_down

Create a Rackup file (config.ru). 

See https://github.com/jweir/ScaleDown/tree/master/config.sample.ru for all options

```ruby
require 'rubygems'
require 'scale_down'

ScaleDown.tap do |config|
  # Shared secret for generating the HMAC signature
  config.hmac_key    = "secret"

  # Path to the public directory
  config.public_folder = File.expand_path(File.dirname(__FILE__))+"/public"
end

run ScaleDown::Controller
```

Configure Nginx, Apache, etc to run the server.

Known Issues
===========

Pluses in filenames will not route properly.  Filenames need to have these removed or replaced.

Dependencies
============

* Sinatra
* RMagick
* Ruby-HMAC

RMagick can be a bit tricky to install, these links http://www.google.com/search?q=install+rmagick might help.

LICENSE
=======

(The MIT License)

Copyright © 2011 John Weir & Fame Driver LLC

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the ‘Software’), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED ‘AS IS’, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
