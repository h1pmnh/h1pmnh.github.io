---
title: "Howto: Use Burp Hackvertor Plugin to Re-sign Requests" # Title of the blog post.
date: 2024-02-02T00:00:00-04:00 # Date of post creation.
description: "A how-to article describing the use of the Burp Hackvertor plugin to recalculate a signature header based on an entire request" # Description used for search engine.
featured: true # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: true # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - howto
  - burp
  - tutorial
  - hackvertor
  - web
---

## Summary

More and more in modern web applications, particularly sensitive applications such as financial apps, we see the introduction of signature headers which are used to provide some mechanism of tamper-proofing of the request from the client. These signatures can be problematic if using common tools such as Burp Suite or even automated tools such as `sqlmap`. In this article I describe a technique for recalculating these header values using the excellent Burp Suite plugin [Hackvertor](https://github.com/hackvertor/hackvertor) and custom tag definitions available in this plugin.

Please note, there are other viable techniques for doing this such as the use of [`mitmproxy`](https://mitmproxy.org/), and I don't claim this is the best technique for solving this problem, but it is *a* technique :smile:

## Describing the Problem

Typically a signed request will look something like this:

```http
POST /foo/bar?some=arg HTTP/1.1
Host: www.example.com
...
X-Signature-Header: <some_hex_value>
...

{"json":"attribute"}
```

In many cases, the value of the `X-Signature-Header` is calculated by the browser at the time the request is sent over the wire. Typically, this value is calculated using some hash-type or HMAC function, based on some subset of the contents of the request itself. I've run across multiple of these in the real world, and I've seen the following as any/all inputs to the hashing function:

 * HTTP method (e.g. `POST`)
 * HTTP url (e.g. `/foo/bar`)
 * HTTP query string (e.g. `some=arg`)
 * HTTP body (e.g. `{"json":"attribute"}`)
 * Other values such as a pre-shared key or API key

Correctly calculating this header after tampering requires understanding how the browser client is determining the value in the first place.

## Finding the Signature Calculation in Javascript Source

Because the header value is determined by the browser when it is making requests, it is almost always the case that the Javascript associated with the site will have the methodology for calculating the header. Usually once can simply use the "Search" function in the browser dev tools to find the header name (in our example, `X-Signature-Header`) in the Javascript, and trace the value of this header. If there is a sourcemap available, this is often even easier to determine, but even in obfuscated Javascript there are only a handful of libraries typically used such as [crypto-js](https://www.npmjs.com/package/crypto-js). This may take some practice but it's a good opportunity to learn to use the Developer Tools in your browser.

A slightly modified example from a rececnt test where the sourcemap was available:
```javascript
export const signRequest = (data: any): string => {
  return md5(JSON.stringify(data));
};
```

In this case, the header value is calculated using an MD5 hash of the `POST` body (the `POST` body is passed to this function).

## How to Use Custom Hackvertor Tags

If you're reading this, I assume you already know how to use Hackvertor. If not, I'd suggest getting familiar with the overall use of the tool first. By default for security reasons, Hackvertor disables the use of custom tags. You'll have to enable them by going to the "Hackvertor > Allow code execution tags" menu.

## Our Scenario

Let's assume the site we are tampering has the following signature mechanism:

 * Only `POST` requests are signed
 * `POST` requests are JSON
 * The value of the `X-Signature-Header` is calculated as the MD5 hash of the JSON body of the request

We want our custom Hackvertor tag to replace the `X-Signature-Header` if it is present, or add the header if it's not present.

## Creating our Custom Hackvertor Tag

In Burp Suite, click "Hackvertor > Create custom tag". Name the custom tag something and pick the Python language (we pick this because the built-in `hashlib` contains many common hashing functions). We'll paste the following tag body:

```python
import hashlib

def sign(request_bytes):
    # sign using the hash function as identified in the Javascript
    hash = hashlib.md5()
    hash.update(request_bytes)
    return hash.hexdigest()

full_request=input

# assume correct line endings per HTTP spec
line_endings = '\r\n'
# find the end of the headers block
end_of_headers = full_request.find(line_endings + line_endings)

# split the request by line endings into an array - NB: this require tweaking for binary protocols used in the body such as Protobuf
lines = full_request[0:end_of_headers+2].strip().split(line_endings)

# in case we need to parse out the method and URL for inclusion in the signature
if lines:
    first_line_words = lines[0].strip().split()
    if len(first_line_words) >= 2:
        method = first_line_words[0]
        url = first_line_words[1]
    else:
        method = None
        url = None
else:
    method = None
    url = None

body_index = -1
signature_index = -1
# find the location of the signature header if it exists, and the body
for i in range(0,len(lines)):
    if lines[i].startswith('X-Signature-Header'):
        signature_index = i
    if lines[i].strip() == '':
        body_index = i
        break

# get the body content
request_content = full_request[end_of_headers+4:]
# calculate the hash
signature = sign(request_content)

# update or insert the header into the array using the calculated value
if signature_index == -1:
    lines.insert(i-1,"X-Signature-Header: %s" % signature)
else:
    lines[signature_index] = "X-Signature-Header: %s" % signature

# reassemble the full request
output = line_endings.join(lines) + line_endings + line_endings + request_content
```

Hopefully the comments in the code are helpful, essentially we are:

 * Splitting the HTTP headers by line endings
 * Locating the signature header (if present) and body content
 * Signing the body content
 * Reassembling the entire HTTP request

Unlike a typical Hackvertor tag, which is usually used to wrap a specific piece of data e.g. a query string parameter, this tag is designed to wrap the _entire request_! This is to make using the tag much easier in practice and avoid the annoying copy/paste of parameters etc. I've found this style of Hackvertor custom tag to be very convenient to use and also to explain to triagers and customers.

Please save the tag and let's continue on to demonstrate its use in a request.

![Example Filled Custom Tag Form](/images/burp-hackvertor-signature/custom_tag_dialog.png)

**Note** It seems Hackvertor recently introduced a code length restriction? of 1337 bytes. To use this tag you may need to remove the comments.

## Using our Custom Hackvertor Tag

Open a new Repeater tab. Paste the sample request from the introduction:

```http
POST /foo/bar?some=arg HTTP/1.1
Host: www.example.com
X-Signature-Header: <some_hex_value>

{"json":"attribute"}
```

Next, let's insert our new tag.

 * Select the entire request
 * Right-click, select "Extensions > Hackvertor > Custom > <Your Tag>"
 * Send the request

![Right-click Menu Tree](/images/burp-hackvertor-signature/insert_custom_tag.png)

The request should look like this (please note the tag will differ slightly because of Hackvertor's use of custom tags):
```
<@__demo_signature_tag('5634400c8889932e588fc892746c69e3')>POST /foo/bar?some=arg HTTP/2
Host: www.example.com
X-Signature-Header: <some_hex_value>
Content-Length: 44

{"json":"attribute"}<@/__demo_signature_tag>
```

And if we look in Logger we should see our calculated `X-Signature-Header`:

```http
POST /foo/bar?some=arg HTTP/1.1
Host: www.example.com
X-Signature-Header: 068f44ed2f714f315dcbcb762470ee63
Content-Length: 20

{"json":"attribute"}
```

## Using Hackvertor Tags in Other Burp Functions

The awesome thing about Hackvertor is that it can be used in other Burp functions e.g. Scanner/Intruder, so you can construct a payload using these tags and then run an active scan. The Hackvertor custom tag will ensure the signature is updated with every payload Burp generates!

## Conclusion

If you find something wrong or just want to share similar experiences please feel free to reach out to me, I love hearing comments from folks!
