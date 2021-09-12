# Mic (Forensics)
> My Epson InkJet printer is mysteriously printing blank pages. Is it trying to tell me something?

Viewing `scan.pdf`, we get 34 blank pages. Neat

Opening it in a text editor, there's a lot of random bytes (to be expected)
```
%PDF-1.3
%����
5 0 obj
<</BaseFont /Helvetica /Encoding /WinAnsiEncoding /Name /F1 /Subtype /Type1 /Type /Font >> 
endobj
6 0 obj
<</Type /FontDescriptor /FontName /BCDEEE+Calibri /Flags 32 /ItalicAngle 0 /Ascent 750 /Descent -250 /CapHeight 750 /AvgWidth 521 /MaxWidth 1743 /FontWeight 400 /XHeight 250 /StemV 52 /FontBBox [-503 -250 1240 750 ] /FontFile2 7 0 R >> 
endobj
7 0 obj
<</Filter /FlateDecode /Length1 81740 /Length 19389 >> stream
x��}|TU��9�N��$3�d�I&af�$� ����;�	5!���"FQ�(�^ѵ�X&j��bY��v]WW���*��s�9�������ϛ<�<�=��ޓ��cv|�Xm娊```
```

Using binwalk on the pdf, there is a ton of compressed data:

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PDF document, version: "1.3"
452           0x1C4           Zlib compressed data, default compression
20068         0x4E64          Zlib compressed data, default compression
316154        0x4D2FA         Zlib compressed data, default compression
335772        0x51F9C         Zlib compressed data, default compression
631222        0x9A1B6         Zlib compressed data, default compression
...
10634859      0xA2466B        Zlib compressed data, default compression
```

Looking at some of the compressed data, they appear to be mostly fonts, so that's not especially helpful. After some googling, I found [qpdf](http://qpdf.sourceforge.net/), a CLI tool that does PDF transformations.

Using this, we can decompress the pdf into something more human readable:
```
qpdf --qdf --object-streams=disable scan.pdf out.pdf
```

`out.pdf` looks like this

```
%PDF-1.3
%����
%QDF-1.0

%% Original object ID: 442 0
1 0 obj
<<
  /Pages 2 0 R
  /Type /Catalog
>>
endobj

%% Original object ID: 443 0
2 0 obj
<<
  /Count 34
  /Kids [
    3 0 R
    4 0 R
    5 0 R
    6 0 R
    ...
```

Still not great, but a lot better. After some headers/fonts, we get to the content of page 1:


```
%% Contents for page 1
%% Original object ID: 11 0
37 0 obj
<<
  /Length 38 0 R
>>
stream
1 0 0 1 0 0 cm
BT
/F1 12 Tf
14.4 TL
ET
255 255 0 RG
255 255 0 rg
n
0.288 839.0098 m
0.288 839.1688 0.15906 839.2978 0 839.2978 c
-0.15906 839.2978 -0.288 839.1688 -0.288 839.0098 c
-0.288 838.8507 -0.15906 838.7218 0 838.7218 c
0.15906 838.7218 0.288 838.8507 0.288 839.0098 c
f*
n
0.288 836.1298 m
0.288 836.2888 0.15906 836.4178 0 836.4178 c
-0.15906 836.4178 -0.288 836.2888 -0.288 836.1298 c
-0.288 835.9707 -0.15906 835.8418 0 835.8418 c
0.15906 835.8418 0.288 835.9707 0.288 836.1298 c
f*
```

After digging through the [PDF specifications](https://www.adobe.com/content/dam/acom/en/devnet/pdf/pdfs/PDF32000_2008.pdf) (page 133), we can see that the PDF is drawing very tiny curves in yellow all over the page. If we change the color from yellow to black, we get `mic_out_black.pdf`

After some more googling, we find these little dots are Machine Identification Codes (mic), something we were tipped off to from the start of this challenge. Printers often add these little yellow dots to the pages it prints as essentially metadata. The mic contains the date and time the pages was printed, the serial number of printer, and parody bits on the edges

Taking a look at the dots, the time fields stay the same, but the serial number changes every time. The serial number is read from top to bottom, and each number is read as two digets.

The first code is:
```
0000001 -> 01
0000010 -> 02
-> 0102, or "f" in ASCII
```

Reading the rest of the codes the same way, we get the flag: `flag{watchoutforthepoisonedcoffee}`
