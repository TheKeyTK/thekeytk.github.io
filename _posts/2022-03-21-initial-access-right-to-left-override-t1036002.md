---
title: Initial Access -  Right-To-Left Override [T1036.002]
layout: post
---

You have probably heard that Microsoft will soon disable macros in the documents that are coming from the internet ([https://docs.microsoft.com/en-us/deployoffice/security/internet-macros-blocked](https://docs.microsoft.com/en-us/deployoffice/security/internet-macros-blocked){:target="blank"}). It got me thinking, what are the other alternatives to gain initial access to the target's environment, what kind of payloads can we deliver?

While browsing Mitre's att&ck I have come across a pretty old technique, known as the [right-to-left override attack (rtlo)](https://attack.mitre.org/techniques/T1036/002/){:target="blank"}. If you have never heard of it, I will try to explain it briefly.
Some languages write from left to right side, like English and German for example. Some languages write from right to left side, like Arabic and Hebrew for example.
By default, Windows OS displays letters from left to right side but since it also supports other languages it supports writing from right to left side. There is a unicode character that flips the text to the right-to-left side (`\u202E`) and you can combine it with the already existing text.
Basically, you can reverse text in file names and that way hide the true file extension.

##### *Note: In my examples Windows OS is set up to show the file extensions. By default file extensions are hidden. I believe there is no point in performing this type of extension spoofing if the target Windows OS is hiding file extensions.*
<br/>
I have written a simple [tool](https://github.com/ExAndroidDev/rtlo-attack){:target="blank"} in c# that spoofs the extension of a desired executable as well as changes the icon to anything you specify.

### How to use the tool?
The tool accepts two arguments, the first one is the input executable and the second is the icon. Since I don't know how to easily explain how to use my tool I will try to explain it through examples.

Firstly you need to rename your executable to suits the required format before passing it into the tool. 
	Make sure that the extension you would like to spoof is the last part before the true extension. For example if you want to make a file look like "`.png`" make sure your executable name is something like "`simple.exemple`**.png.**`exe`".
Also, make sure that the true extension, let's say it is ".exe" or ".scr", is present in the filename, whether in its true form or reversed ("`exe.`" or "`rcs.`"), like "`simple`**.exe**`mple.png.exe`". (More details on reversed extension below.)

Once you rename your executable to match the required format pass it to the tool as a first argument along with the desired icon. <br/>
``` text
./rtlo-attack.exe simple.exemple.png.exe png.ico
```

![example-1](/assets/example-1.png){: .center-image }

The tool works by taking the file extension, like in the previous example "`.exe`" and finding the match inside the executable's name. After the match is found, it adds rtlo and ltro characters to hide the actual extension and preserve the ordinary-looking name. If you are interested in more details take a look at the [source code](https://github.com/ExAndroidDev/rtlo-attack/blob/main/Program.cs#L47){:target="blank"}.


### Additional obfuscation
As I mentioned a few paragraphs above, you can also reverse the true file extension to obfuscate it even more. 
Let's take a look at the next example. I am going to use the"`.scr`" extension, since reversing "`.exe`" doesn't look much different.
##### *Note: To create an ".scr" executable just change the file extension from ".exe" to ".scr" and the executable will still run.*
<br/>
So let's rename an executable to meet filename format requirements before we pass it to the tool. Rename the executable to something like "`My-pictu`**rcs.**`png.scr`" and pass it to the tool. Notice how we specified the true file extension but in reverse ("`rcs.`") in the file name. After the tool has done its magic the resulting executable name will look like this "`My-picturcs.png`". 

![example-2](/assets/example-2.png){: .center-image }
<br/>

The other interesting way of obfuscation is changing extension casing. Since Windows OS doesn't care about the lower or upper case, you can have an extension such as "`.exE`".
An example executable can look like this "`My`**.exE**`nvironment.png.exE`" and the output would end up looking like this: "`My.exEnvironment.png`".

![example-3](/assets/example-3.png){: .center-image }
<br/>

### Name ideas:

| Original name | Spoofed name | 
| -------- | -------- |
| Company.Executive.Summary.txt.Exe     | Company.Executive.Summary.txt     |
|SOMETHINGEXe.png.eXE | SOMETHINGEXe.png |





<br/>

###  What about anti-virus?
You might have noticed that some AVs, like Microsoft's Defender, for example, flag a file with rtlo character in its name. 
Fortunately, someone has already researched that part. Thanks to the [http://blog.sevagas.com/?Bypass-Defender-and-other-thoughts-on-Unicode-RTLO-attacks](http://blog.sevagas.com/?Bypass-Defender-and-other-thoughts-on-Unicode-RTLO-attacks){:target="blank"}, he concluded that AV might flag the file if it contains a rtlo character and ends with a valid extension. I will summarize it below but I encourage you to read the article.

For example, if you spoof the file name to look like "`smth.executive.summary.txt`" it will be flagged by AV, but if the name is "`smth.executive.summary.blah`" it will not.
If only there is a way to make an extension look legit but is not actually valid...

Of course there is a way. Since we are already playing with different languages why wouldn't we use letters from different alphabets.<br/> For example payload "`smth.executive.summary.txt`" is flagged by the AV, but "`smth.executive.summary.tхt`" is not since the "`x`" (`\u0445`) is not an actual Latin letter "`x`" (`\u0078`) but an Cyrillic version of letter "`h`".  Simply copy/paste the unicode character from any unicode table website and paste it into the file name and it will do the trick.


##### *Note: There are a variety of executable file extensions, but keep in mind that not all of them can have a custom icon. For example, you can't change the icon of the "`.bat`" or "`.com`" file types.*

### But how do we deliver it?
One option would be to make your target download ZIP archive or ISO/VHD image. ISO image is automatically mounted to the Windows OS by simply double-clicking it and since it is a different system file format it would bypass the so-called mark-of-the-web (MOTW). But that is the content for the other post.


The end!
