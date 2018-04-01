# WebP on demand

This is a solution for serving auto-generated WebP images instead of JPEG/PNG to browsers that supports WebP. It works by `.htaccess` magic coupled with an image converter. Basically, JPEGs and PNGs are routed to the image converter, unless it has already converted the image. In that case, it is routed directly to the converted image.

The solution also works on images referenced in CSS.

The image converter is able to use several methods to convert the image (`imagick`, `gd`, directly calling `cwebp` binary or connecting to the 'EWWW Image Optimizer' cloud service). To learn more about its options, go to the [project on github](https://github.com/rosell-dk/webp-convert).

## Installation
1. Copy the `.htaccess` and the `webp-convert` directory into the root of your site. If you are using `git clone`, note that you can pass `--recursive` to the `clone` command in order fetch the `WebPConvert` [submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules) as well.
2. Make sure that the `www-data` user has permission to write in the relevant portion of your site.
3. If you are using *nginx* to serve JPEGs and PNGs, reconfigure it so Apache gets to serve these

## Configuration
Quality and location of the converted files are both customizable. 

For location, you have two options:

1. Putting the converted file in the same folder as the original. The file will get the same name as the original plus `.webp`. For example, `image.jpg` will be converted into `image.jpg.webp`.

2. Putting the converted file into a specific folder. The converted file will then be organized into the same structure as the original. For example, if you set the folder to be `webp-cache`, then `/images/2017/fun-at-the-hotel.jpg` will be converted into `/webp-cache/images/2017/fun-at-the-hotel.jpg`.

You choose between the two options by commenting/uncommenting the corresponding blocks in the `.htaccess`. Precise instructions are given in the there.

You can configure the image converter directly in the `.htaccess`. The options are described on the project page of [webp-convert](https://github.com/rosell-dk/webp-convert) (scroll down to the part that describes the script).

If you want to have a different quality on a certain image, you can append `&reconvert&quality=95` to the image URL. You can in fact override or change any converter option like this.

## Verifying that it works
Note that the solution does not change any HTML. In the HTML the image source is still set to its JPEG variant, eg `image.jpg`. On browsers that support WebP images, that request is routed to the WebP image. To verify that the solution is working, do the following:

- Open the page in Google Chrome
- Right-click the page and choose "Inspect"
- Click the "Network" tab
- Reload the page
- Find a `jpeg` or `png` image in the list. In the 'type' column, it should say `webp`

## Troubleshooting
By appending `?debug` to your image URL, you get a report from `WebPConvert` instead of the converted image. If there is no report, it means that the `.htaccess` is not working as intended.

- Perhaps there are other rules in your `.htaccess` that interfere with the rules?
- Perhaps you are using *nginx* to serve image files, which means Apache does not get the chance to process the request. You then need to reconfigure your server setup

If you *do* get a report, see what it says. As with `&reconvert`, you can set up `WebPConvert`'s options. For example, appending `&debug&preferred-converters=gd` will configure it to try the `gd` converter before the other converters.

## Limitations

* The solution does not work on Microsoft IIS server
* The solution requires PHP > 5.5.0 compiled with webp support

## FAQ

### How do I make this work with a CDN?
Chances are that the default setting of your CDN is not to forward any headers to your origin server. But the solution needs the "Accept" header, because this is where the information is whether the browser accepts webp images or not. You will therefore have to make sure to configure your CDN to forward the "Accept" header.

The .htaccess takes care of setting the "Vary" HTTP header to "Accept" when routing WebP images. When the CDN sees this, it knows that the response varies, depending on the "Accept" header. The CDN is thus instructed not to cache the response on URL only, but also on the "Accept" header. This means that it will store an image for every accept header it meets. Luckily, there are (not that many variants for images[https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation/List_of_default_Accept_values#Values_for_an_image], so it is not an issue.

## Detailed explanation of how it works

### With location set to same folder as original (option 1)
When the destination of the converted files is set to be the same as the originals, the .htaccess contains the following code (comments removed)

```
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /

  RewriteCond %{HTTP_ACCEPT} image/webp
  RewriteRule ^(.*)\.(jpe?g|png)$ $1.$2.webp [NC,T=image/webp,E=accept:1]

  RewriteCond %{QUERY_STRING} (^reconvert.*)|(^debug.*) [OR]
  RewriteCond %{DOCUMENT_ROOT}/$1.$2.webp !-f
  RewriteCond %{QUERY_STRING} (.*)
  RewriteRule ^(.*)\.(jpe?g|png)\.(webp)$ webp-convert/webp-convert.php?source=$1.$2&quality=85&preferred-converters=imagick,cwebp,gd&serve-image=yes&%1 [NC]

</IfModule>
<IfModule mod_headers.c>
    Header append Vary Accept env=REDIRECT_accept
</IfModule>
AddType image/webp .webp
```

Lets take it line by line, skipping the first couple of lines:

```RewriteCond %{HTTP_ACCEPT} image/webp```\
This condition makes sure that the following rule only applies when the client has sent a HTTP_ACCEPT header containing "image/webp". In other words: The next rule only activates if the browser accepts WebP images.

```RewriteRule ^(.*)\.(jpe?g|png)$ $1.$2.webp [NC,T=image/webp,E=accept:1]```\
This line rewrites any request that ends with ".jpg", ".jpeg" or ".png" (case insensitive). The target is set to the same as source, but with ".webp" appended to it. Also, MIME type of the response is set to "image/webp" (my tests shows that Apache gets the MIME type right without this, but better safe than sorry - it might be needed in other versions of Apache). The E flag part sets the environment variable "accept" to 1. This is used further down in the .htaccess to conditionally append a Vary header. So setting this variable means that the Vary header will be appended if the rule is triggered. The NC flag makes the match case insensitive. 

```RewriteCond %{QUERY_STRING} (^reconvert.*)|(^debug.*) [OR]```\
This line tests if the query string starts with "reconvert" or "debug". If it does, the rule that passes the request to the image converter will have green light to go. So appending "&reconvert" to an image url will force a new convertion even though the image is already converted. And appending "&debug" also bypasses the file check, and actually also results in a reconversion - but with text output instead of the image.

```RewriteCond %{DOCUMENT_ROOT}/$1.$2.webp !-f```\
This line adds the condition that the image is not already converted. The $1 and $2 refers to matches of the following rule. The condition will only match files ending with ".jpeg.webp", "jpg.webp" or "png.webp". As a webp is thus requested, it makes sense to have the rule apply even to browsers not accepting "image/webp". 

```RewriteCond %{QUERY_STRING} (.*)```\
This line enables us to use the query string in the following rule, as it will be available as "%1". The condition is always met.

```RewriteRule ^(.*)\.(jpe?g|png)\.(webp)$ webp-convert/webp-convert.php?source=$1.$2&quality=85&preferred-converters=imagick,cwebp,gd&serve-image=yes&%1 [NC]```\
This line rewrites any request that ends with ".jpg", ".jpeg" or ".png" to point to the image converter script. The php script get passed a "source" parameter, which is the file path of the source image. The script also accepts a destination root. It is not set here, which means the script will save the the file in the same folder as the source. The %1 is the original query string (it refers to the match of the preceding condition). This enables overiding the converter options set here. For example appending "&debug&preferred-converters=gd" can be used to test the gd converter. Or "&reconvert&quality=100" can be appended in order to reconvert the image using better quality.

```Header append Vary Accept env=REDIRECT_accept```\
This line appends a response header containing: "Vary: Accept", but only when the environment variable "accept" is set by the "REDIRECT" module.

### With location set to specific folder (option 2)
When the destination of the converted files is set to be a specific folder as the originals, the core functionality lies in the following code:

```
RewriteCond %{HTTP_ACCEPT} image/webp
RewriteCond %{QUERY_STRING} (^reconvert.*)|(^debug.*) [OR]
RewriteCond %{DOCUMENT_ROOT}/webp-cache/$1.$2.webp !-f
RewriteCond %{QUERY_STRING} (.*)
RewriteRule ^(.*)\.(jpe?g|png)$ webp-convert/webp-convert.php?source=$1.$2&quality=80&destination-root=webp-cache&preferred-converters=imagick,cwebp&serve-image=yes&%1 [NC,T=image/webp,E=accept:1]

RewriteCond %{HTTP_ACCEPT} image/webp
RewriteCond %{QUERY_STRING} !((^reconvert.*)|(^debug.*))
RewriteCond %{DOCUMENT_ROOT}/webp-cache/$1.$2.webp -f
RewriteRule ^(.*)\.(jpe?g|png)$ /webp-cache/$1.$2.webp [NC,T=image/webp,E=accept:1,QSD]
```

The code is divided in two blocks. The first block takes care of delegating the request to the image converter under a set of conditions. The other block takes care of rewriting the url to point to an existing image under another set of conditions. The two set of conditions are mutual exclusive.

You should know that in *mod_rewrite*, OR has higher precedence than AND [[ref](https://stackoverflow.com/questions/922399/how-to-use-and-or-for-rewritecond-on-apache)]. The first set of conditions thus reads:

If *Browser accepts webp images* AND (*Query string begins with "reconvert" or "debug"* OR *There is no existing converted image*) AND *Query string is empty or not*

The last condition is always true. It is there to make the query string available to the following rule.

The other set of conditions reads;
If *Browser accepts webp images* AND *Query string does not begin with "reconvert" or "debug"* AND *There is an existing converted image*

Otherwise it is the same kind of stuff that is going on as in option 1 - which is described in the preceding section. Oh, there is the QSD flag. It tells Apache to strip the query string.

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

## Roadmap

* Only serve webp when filesize is smaller than original (ie. the script can generate an (extra) file image.jpg.webp.too-big.txt when filesize is larger - the htaccess can test for its existence)
* Is there a trick to detect that the original has been updated?

## Related
* [My original post presenting the solution](https://www.bitwise-it.dk/blog/webp-on-demand)
* [Wordpress adaptation of the solution](https://github.com/rosell-dk/webp-express) - It's on github, but I have submitted it to Wordpress. Once it is hopefully approved, you will be able to install it directly from Wordpress. It is called "WebP Express"
* https://www.maxcdn.com/blog/how-to-reduce-image-size-with-webp-automagically/
