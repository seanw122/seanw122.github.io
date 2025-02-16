---
layout: post
title:  "EBCDIC - Really??"
date:   2021-02-23 15:09:00 +0000
categories: Encoding, EBCDIC
---

Yup, EBCDIC.

I had to transform a file in EBCDIC to something legible on Windows and parsable with .NET. I received the files via a thumb drive. I copied the files to my laptop and started my investigation. I use [WinHex](https://www.x-ways.net/winhex/). It's not free but it's great.

I opened the file with WinHex and it shows the bytes and an ANSI/ASCII version. I quickly noticed the first two bytes converted in ASCII is `PK`. What does this tell you? PK means it's likely a compressed file. Sure was! A quick chat with the client indicated they use 7-Zip on the mainframe to compress the files. Not sure why the files I got were named with `txt` extension. (smh)

So I created a very quick method to convert the data. There's problem easier or built-in methods now in the latest .NET.
{% highlight c# %}
private static byte[] ConvertBytes(byte[] original, string encoding)
{
    var target = Encoding.ASCII;
    var ebcdic = Encoding.GetEncoding(encoding);
    return Encoding.Convert(ebcdic, target, original);
}
{% endhighlight %}
