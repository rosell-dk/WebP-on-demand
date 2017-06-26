# WebP on demand

This is a solution for serve autogenerated WebP images instead of jpeg/png to browsers that supports WebP. It works by .htaccess magic coupled with an image converter. Basically, jpegs and pngs are routed to the image converter, unless the image converter has already converted the image. In that case, it is routed directly to the converted image. 

The solution also works on images referenced in CSS.

## Installation
To use the solution on your website, simply copy the two files into your root folder. 

## Configuration
The quality and the location of the converted files is customizable. 

For location, you have three options:

1. Put the converted files in the same folder as the original. The file will get the same name as the original plus ".webp". Ie. "image.jpg" will be converted into "image.jpg.webp"

2. Put the converted files be into a specific folder. The converted files will then be organized into the same structure as the original. If you for example set the folder to be "webp-cache", then "/images/2017/fun-at-the-hotel.jpg" will be converted into "/webp-cache/images/2017/fun-at-the-hotel.jpg"

3. Do not to store the converted files at all. Each time an image is requested, a convertion will be made and served 

You choose between the three options by commenting/uncommenting blocks in the .htaccess. Precise instructions are given in the .htaccess.

## Verifying that it works
Note that the solution does not change any HTML. In the HTML the image src is still set to ie "example.jpg". On browsers that supports WebP images, that request is routed to the WebP image. To verify that the solution is working, do the following:

- Open the page in Google Chrome
- Right-click the page and choose "Inspect"
- Click the "Network" tab
- Reload the page
- Find a jpeg or png image in the list. In the "type" column, it should say "webp"


## Limitations

* The solution does not work on Microsoft IIS server
* The solution requires PHP > 5.5.0 compiled with webp support


## FAQ

### How do I make this work with a CDN?
Chances are that the default setting of your CDN is not to forward any headers to your origin server. But the solution needs the "Accept" header, because this is where the information is whether the browser accepts webp images or not. You will therefore have to make sure to configure your CDN to forward the "Accept" header.

The .htaccess takes care of setting the "Vary" HTTP header to "Accept" when routing WebP images. When the CDN sees this, it knows that the response varies, depending on the "Accept" header. The CDN is thus instructed not to cache the response on URL only, but also on the "Accept" header. This means that it will store an image for every accept header it meets. Luckily, there are (not that many variants for images[https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation/List_of_default_Accept_values#Values_for_an_image], so it is not an issue.

### Image converter does not work
The image converter is using the [imagewebp()](http://php.net/manual/en/function.imagewebp.php) function in PHP to create WebP images. The function is is available from PHP 5.5.0. However, it requires that PHP is compiled with WebP support, which unfortunately aren't the case on many webhosts (according to [this link](https://stackoverflow.com/questions/25248382/how-to-create-a-webp-image-in-php)). WebP generation in PHP 5.5 requires that php is configured with the "--with-vpx-dir" flag and in PHP 7.0, php has to be configured --with-webp-dir flag [source](http://il1.php.net/manual/en/image.installation.php).


## Roadmap

* Use cweb converter when imagewebp isn't available.
* Only serve webp when filesize is smaller than original (ie. the script can generate an (extra) file image.jpg.webp.too-big.txt when filesize is larger - the htaccess can test for its existence)
* Is there a trick to detect that the original has been updated?



## Detailed explanation of how it works


### With location set to same folder as original (option 1)
When the destination of the converted files is set to be the same as the originals, the .htaccess contains the following code (comments removed)

```
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteCond %{HTTP_ACCEPT} image/webp
  RewriteRule ^(.*)\.(jpe?g|png)$ $1.$2.webp [T=image/webp,E=accept:1]
  RewriteCond %{DOCUMENT_ROOT}/$1.$2.webp !-f
  RewriteRule ^(.*)\.(jpe?g|png)\.(webp)$ webp-convert/webp-convert.php?source=$1.$2&quality=80&preferred-converters=imagick,cwebp&serve-image
</IfModule>
<IfModule mod_headers.c>
    Header append Vary Accept env=REDIRECT_accept
</IfModule>
AddType image/webp .webp
```

Lets take it line by line, skipping the first couple of lines:

```RewriteCond %{HTTP_ACCEPT} image/webp```\
This condition makes sure that the following rule only applies when the client has sent a HTTP_ACCEPT header containing "image/webp". In other words: The next rule only activates if the browser accepts WebP images.

```RewriteRule ^(.*)\.(jpe?g|png)$ $1.$2.webp [T=image/webp,E=accept:1]```\
This line rewrites any request that ends with ".jpg", ".jpeg" or ".png". The target is set to the same as source, but with ".webp" appended to it. Also, MIME type of the response is set to "image/webp" (my tests shows that Apache gets the MIME type right without this, but better safe than sorry - it might be needed in other versions of Apache). The E flag part sets the environment variable "accept" to 1. This is used further down in the .htaccess to conditionally append a Vary header. So setting this variable means that the Vary header will be appended if the rule is triggered.

```RewriteCond %{DOCUMENT_ROOT}/$1.$2.webp !-f```\
This line is a new condition instructing that the following rule is only to be applied, if there is not already a converted file. Thus, there will be no more rules to be applied if the converted file exists. The $1 and $2 refers to matches of the following rule. The condition will only match files ending with ".jpeg.webp", "jpg.webp" or "png.webp". As a webp is thus requested, it makes sense to have the rule apply even to browsers not accepting "image/webp". 

```RewriteRule ^(.*)\.(jpe?g|png)\.(webp)$ webp-convert/webp-convert.php?source=$1.$2&quality=80&preferred-converters=imagick,cwebp&serve-image```\
The fourth line rewrites any request that ends with ".jpg", ".jpeg" or ".png" to point to the image converter script. The php script get passed a "source" parameter, which is the file path of the source image. The script also accepts a destination root. It is not set here, which means the script will save the the file in the same folder as the source.

```Header append Vary Accept env=REDIRECT_accept```\
This line appends a response header containing: "Vary: Accept", but only when the environment variable "accept" is set by the "REDIRECT" module.
 

### With location set to specific folder (option 2)
When the destination of the converted files is set to be a specific folder as the originals, the core functionality lies in the following code:

```
RewriteCond %{HTTP_ACCEPT} image/webp
RewriteCond %{DOCUMENT_ROOT}/web-cache/$1.$2.webp !-f
RewriteRule ^(.*)\.(jpe?g|png)$ webp-convert/webp-convert.php?source=$1.$2&quality=80&destination-root=webp-cache&preferred-converters=imagick,cwebp&serve-image [T=image/webp,E=accept:1]

RewriteCond %{HTTP_ACCEPT} image/webp
RewriteCond %{DOCUMENT_ROOT}/webp-cache/$1.$2.webp -f
RewriteRule ^(.*)\.(jpe?g|png)$ /webp-cache/$1.$2.webp [T=image/webp,E=accept:1]
```

The first line is a condition which makes sure that the following rule only applies when the client has send a HTTP_ACCEPT header containing "image/webp".

The second line and third line together takes care of rewriting requests that ends with ".jpg", ".jpeg" or ".png" to the converter script, when there is not already a converted file at the location. $1 matches the path of the original image. $2 matches the extension. The destination folder is set to "webp-cache" in this example. I have not set the rule to be last with the "L" directive, but this can be considered.

The fourth and fifth line together takes care of rewriting requests that points to already converted files, residing in the specific destination folder.

## A similar project
This project is very similar to [WebP realizer](https://github.com/rosell-dk/webp-realizer). *WebP realizer* assumes that the conditional part is in HTML, like this:
```
<picture>
  <source srcset="images/button.jpg.webp" type="image/webp" />
  <img src="images/button.jpg" />
</picture>
```
And then it automatically generates "image.jpg.webp" for you, the first time it is requested.\
Pros and cons:

- *WebP on demand* works on images referenced in CSS (*WebP realizer* does not)\
- *WebP on demand* requires no change in HTML (*WebP realizer* does)\
- *WebP realizer* works better with CDN's - CDN does not need to cache different versions of the same URL


## Related
* [My original post presenting the solution](https://www.bitwise-it.dk/blog/webp-on-demand)
* [Wordpress adaptation of the solution](https://github.com/rosell-dk/webp-express) - It's on github, but I have submitted it to Wordpress. Once it is hopefully approved, you will be able to install it directly from Wordpress. It is called "WebP Express"
* https://www.maxcdn.com/blog/how-to-reduce-image-size-with-webp-automagically/


