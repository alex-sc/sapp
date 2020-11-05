# SAPP - Simply A PDF Document Parser

SAPP stands for Simply a PDF Parser and it makes what is name says: parsing PDF files. It also enables other cool features such as rebuilding documents (to make the content more clear or compact), adding images or digitally signing documents.

# 1. Why SAPP
I created SAPP because I wanted to programmatically sign documents, including **multiple signatures**.

I tried [tcpdf](https://tcpdf.org/) along with [FPDI](https://www.setasign.com/products/fpdi/downloads/), but the results are not those expected. When importing a document using FPDI, the result is a new document with the pages of the original one, not the original document. So if the original document was signed, those signatures were lost.

I read about [SetaPDF-Signer](https://www.setasign.com/products/setapdf-signer/details/), but it cannot be used for free. I also inspected some CLI tools such as [PortableSigner](https://github.com/pflaeging/PortableSigner2), but (1) it was Java based (and I really don't like java) and (2) it depends on iText and other libraries which don't seem to be free at this moment.

At the end I got the [PDF 1.7 definition](https://www.adobe.com/content/dam/acom/en/devnet/pdf/pdfs/PDF32000_2008.pdf), and I learned about incremental PDF documents and its utility to include multiple signature in documents.

# 2. Using SAPP

To test the examples, you can clone the repository and use [composer](https://getcomposer.org/):

```bash
$ git clone https://github.com/dealfonso/sapp
$ cd sapp
$ composer dump-autoload
$ php pdfrebuild.php examples/testdoc.pdf > testdoc-rebuilt.pdf
``` 

## 1.1. Examples

In the root folder you can find two simple examples: 

1. `pdfrebuild.php`: this example gets a PDF file, loads it and rebuilds it to make every PDF object to be in order, and also reducing the amount of text to define the document. 
1. `pdfsign.php`: this example gets a PDF file and digitally signs it using a pkcs12 (pfx) certificate.

### 2.1.1. Rebuild PDF files with `pdfrebuild.php`

Once cloned the repository and generated the autoload content, it is possible to run the example:

```bash
$ php pdfrebuild.php examples/testdoc.pdf > testdoc-rebuilt.pdf
```

The result is a more ordered PDF document which is (problably) smaller. (e.g. rebuilding examples/testdoc.pdf provides a 50961 bytes document while the original was 51269 bytes).

```
$ ls -l examples/testdoc.pdf
-rw-r--r--@ 1 calfonso  staff  51269  5 nov 14:01 examples/testdoc.pdf
$ ls -l testdoc-rebuilt.pdf
-rw-r--r--@ 1 calfonso  staff  50961  5 nov 14:22 testdoc-rebuilt.pdf
```

And the `xref` table looks significantly better ordered:
```
$ cat examples/testdoc.pdf
...
xref
0 39
0000000000 65535 f
0000049875 00000 n
0000002955 00000 n
0000005964 00000 n
0000000022 00000 n
0000002935 00000 n
0000003059 00000 n
0000005928 00000 n
...
$ cat testdoc-rebuilt.pdf
...
xref
0 8
0000000000 65535 f
0000000009 00000 n
0000000454 00000 n
0000000550 00000 n
0000000623 00000 n
0000003532 00000 n
0000003552 00000 n
0000003669 00000 n
...
```

**The code:**

```php
use ddn\sapp\PDFDoc;

require_once('vendor/autoload.php');

if ($argc !== 2)
    fwrite(STDERR, sprintf("usage: %s <filename>", $argv[0]));
else {
    if (!file_exists($argv[1]))
        fwrite(STDERR, "failed to open file " . $argv[1]);
    else {
        $obj = PDFDoc::from_string(file_get_contents($argv[1]));

        if ($obj === false)
            fwrite(STDERR, "failed to parse file " . $argv[1]);
        else
            echo $obj->to_pdf_file_s(true);
    }
}
```

### 2.1.2. Sign PDF files with `pdfsign.php`

To sign a PDF document, it is possible to use the script `pdfsign.php`:

```bash
$ php pdfsign.php examples/testdoc.pdf caralla.p12 > testdoc-signed.pdf
```

And now the document is signed. And if you wanted to add a second signature, it is as easy as signing the resulting document again:

```bash
$ php pdfsign.php testdoc-signed.pdf user.p12 > testdoc-resigned.pdf
```

**The code:**

```php
use ddn\sapp\PDFDoc;

require_once('vendor/autoload.php');

if ($argc !== 3)
    fwrite(STDERR, sprintf("usage: %s <filename> <certfile>", $argv[0]));
else {
    if (!file_exists($argv[1]))
        fwrite(STDERR, "failed to open file " . $argv[1]);
    else {
        // Silently prompt for the password
        fwrite(STDERR, "Password: ");
        system('stty -echo');
        $password = trim(fgets(STDIN));
        system('stty echo');
        fwrite(STDERR, "\n");

        $file_content = file_get_contents($argv[1]);
        $obj = PDFDoc::from_string($file_content);
        
        if ($obj === false)
            fwrite(STDERR, "failed to parse file " . $argv[1]);
        else {
            if ($obj->sign_document($argv[2], $password) === false)
                fwrite(STDERR, "could not sign the document");
            else
                echo $obj->to_pdf_file_s();
        }
    }
}
```

# 3. Limitations

At this time, the main limitations are:
- Not dealing with **non-zero generation** pdf objects: they are uncommon, but according to the definition of the PDF structure, they are possible. If you find one non-zero generation object (you get an exception or error in that sense), please send me the document and I'll try to support it.
- Not dealing with **encrypted documents**.
- Other limitations, for sure :)

# 4. Attributions

1. The mechanism for calculating the signature hash is heavily inspired in tcpdf.
1. Reading jpg and png files has been taken from fpdf.