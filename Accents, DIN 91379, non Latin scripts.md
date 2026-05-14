# Accents, DIN 91379, complex scripts, glyph layout

## Correct positioning of accents
To process text containing letters composed of multiple Unicode glyphs e.g. letters with accents,
it is necessary to compute the correct positioning of the glyphs and code this positions into the resulting
PDF file. For complex scripts glyph substitution and reordering is necessary.

OpenPDF can process such texts starting with release 1.3.24.
This page describes the usage for release 3.0.4 or newer with `GlyphLayoutManager`.

For release 3.0.3 using (the now deprecated) `LayoutProcessor` see [Accents, DIN 91379, non Latin scripts (2026-04-02)](https://github.com/LibrePDF/OpenPDF/wiki/Accents,-DIN-91379,-non-Latin-scripts/5654d29adfebf5100edabdde2d2792b74fa66fbe),
for older releases see [Accents, DIN 91379, non Latin scripts (2025-06-06)](https://github.com/LibrePDF/OpenPDF/wiki/Accents,-DIN-91379,-non-Latin-scripts/5d4f967472cc1459cb484adfa86e4ad1db19b8cc).

Internally OpenPDF uses `Java2D` builtin routines for glyph layout, reordering and substitution.
Since `Java 9` these routines rely on the `HarfBuzz` shaping library.

## DIN 91379
We tested this approach with letters conforming to "DIN 91379: Characters and defined character sequences in Unicode for the electronic processing of names and data exchange in Europe, with CD-ROM" (and the predecessor DIN SPEC 91379) which describes a subset of Unicode consisting mainly of Latin letters and diacritic signs. This standard is mandatory for the data exchange of the German administration with citizens and businesses since Nov. 2024.

## Complex scripts
The processing of text in other languages, bidirectional and complex scripts using this approach is possible, you are invited
to try it and share the results.

## Multithreading
`GlyphLayoutManager` is enabled per `Document`, and has no static state, so it is designed to work in a
multithreading environment processing multiple documents in separate threads.

## Usage

### 1. Step: Provide an OpenType font
Provide an OpenType font containing the necessary characters and positioning information, see below
for some open source fonts.
If no OpenType font is provided, GlyphLayoutManager will throw an exception.
```java
	
	import org.openpdf.text.pdf.GlyphLayoutManager;
...
        float fontSize = 12.0f;
        GlyphLayoutManager glyphLayoutManager  = new GlyphLayoutManager();
        // The  OpenType fonts loaded with glyphLayoutManager.loadFont() are
        // available for glyph layout. Only these fonts can be used.
        String fontDir = "org/openpdf/examples/fonts/";
        Font sans = glyphLayoutManager.loadFont(fontDir + "noto/NotoSans-Regular.ttf", fontSize);
```
You can also load the font from an input source. You have to supply a name for loading the font that ends
with ".ttf" or ".otf".
```java
	
	import org.openpdf.text.pdf.GlyphLayoutManager;
...
        float fontSize = 12.0f;
        GlyphLayoutManager glyphLayoutManager  = new GlyphLayoutManager();
        // Provide the input source
        inputSource = ... ;
        Font sans = glyphLayoutManager.loadFont("NotoSans-Regular.ttf", inputSource, fontSize);
```
If an error occurs while loading a font, a `GlyphLayoutFontManager.FontLoadException` is thrown.

### 2. Step: Enable advanced glyph layout
You enable advanced glyph layout by registering the glyphLayoutManager with the Document.
```java
        try (Document document = new Document().setGlyphLayoutManager(glyphLayoutManager)) {
        	
        // proceed as usual
        }
```        
You can also use the following form:
```
        try (Document document = new Document()) {
            document.setGlyphLayoutManager(glyphLayoutManager);
            PdfWriter writer = PdfWriter.getInstance(document, Files.newOutputStream(Paths.get(fileName)));
            
            document.open();
            document.add(new Chunk("A̋ C̀ C̄ C̆ C̈ C̕ C̣ C̦ C̨̆ D̂ F̀ F̄ G̀ H̄ H̦ H̱ J́ J̌ K̀ K̂ K̄ K̇ K̕ K̛ K̦ K͟H K͟h", serif));
            // ...
        }
```

### Set Options
#### Default Options
Optionally you can set the default `GlyphLayoutManager` font options before loading the fonts.
These options are used for all fonts loaded with this `GlyphLayoutManager`.

```java
        GlyphLayoutManager glyphLayoutManager  =
                new GlyphLayoutManager().setDefaultFontOptions(new FontOptions().setKerningOn().setLigaturesOn());
```        

#### Options per font
If you want to use different options, you can set the font options per font while loading the font.
```java
        Font serifKerning = glyphLayoutManager.loadFont(fontDir + "noto/NotoSerif-Regular.ttf", 
                fontSize, new FontOptions().setKerningOn());
        Font serifLigatures = glyphLayoutManager.loadFont(fontDir + "noto/NotoSerif-Regular.ttf", 
                fontSize, new FontOptions().setLigaturesOn());
        Font serifKerningLigatures = glyphLayoutManager.loadFont(fontDir + "noto/NotoSerif-Regular.ttf", 
                fontSize, new FontOptions().setKerningOn().setLigaturesOn());
```

## Exceptions
### Checked Exceptions
`GlyphLayoutManager.loadFont` throws `GlyphLayoutFontManager.FontLoadException` if the font can not be loaded
or it is not an OpenType font.

### Unchecked Exceptions

The constructor of `GlyphLayoutManager` throws an `IllegalStateException`
if `LayoutProcessor` is enabled. Don't use the deprecated `LayoutProcessor`!

If a font is used that has not been loaded `GlyphLayoutManager.loadFont` an `UnsupportedOperationException`
is thrown. All fonts have to be loaded with `GlyphLayoutManager.loadFont`.

## LayoutProcessor
`LayoutProcessor` is the predecessor of `GlyphLayoutManager` and is deprecated now, use `GlyphLayoutManager`.

## FopGlyphProcessor

`GlyphLayoutManager` and [`FopGlyphProcessor`](https://github.com/LibrePDF/OpenPDF/wiki/Multi-byte-character-language-support-with-TTF-fonts) can't be used together, you have to decide for one of them.
If you use `GlyphLayoutProcessor`, `FopGlyphProcessor` is switched off using `document.setGlyphSubstitutionEnabled(false)`.
This call disables `FopGlyphProcessor`, and its functionality like glyph substitution and more will be provided by `GlyphLayoutManager`.

## Usage for HTML files
In addition to the default process for OpenPDF-html you have to create a `GlyphLayoutManager`,
load the fonts with `GlyphLayoutManager` and register the `GlyphLayoutManager` with the `ITextRenderer`.

```java
     public void test() throws Exception {
        var htmlFilename = "GlyphLayoutHtmlExample.html";
        var inputStream = this.getClass().getResourceAsStream(htmlFilename);
        var documentBuilder = DocumentBuilderFactory.newInstance().newDocumentBuilder();
        var document = documentBuilder.parse(inputStream);

        var glyphLayoutManager = new GlyphLayoutManager();
        var fontResolver = new ITextFontResolver();

        loadFont(glyphLayoutManager, fontResolver, "Arimo-Regular.ttf", "fonts/arimo/Arimo-Regular.ttf");
        loadFont(glyphLayoutManager, fontResolver, "Arimo-Bold.ttf", "fonts/arimo/Arimo-Bold.ttf");

        var pdfFilename = "GlyphLayoutHtmlExample.pdf";
        try (var outputStream = new FileOutputStream(pdfFilename)) {
            var renderer = new ITextRenderer(fontResolver);
            renderer.setDocument(document);
            renderer.setGlyphLayoutManager(glyphLayoutManager);
            renderer.layout();
            renderer.createPDF(outputStream);
        }
        System.out.println("PDF created: " + pdfFilename);
    }

    private void loadFont(GlyphLayoutManager glyphLayoutManager, ITextFontResolver fontResolver,
            String fontName, String fontResourcePath)
            throws IOException, GlyphLayoutFontManager.FontLoadException {
        var fontUrl = this.getClass().getResource(fontResourcePath);
        Objects.requireNonNull(fontUrl, "Font not found: " + fontResourcePath);
        var fontStream = fontUrl.openStream();
        var font = glyphLayoutManager.loadFont(fontName, fontStream, 12.0f);
        fontStream.close();
        fontResolver.addFont(font.getBaseFont(), fontUrl.getFile(), null);
    }
```

## Examples
### Producing a document
This example shows the correct rendering for all letters from [DIN 91379](https://en.wikipedia.org/wiki/DIN_91379).

Code: [GlyphLayoutDin91379.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutDin91379.java)

Result: [GlyphLayoutDin91379.pdf](https://github.com/user-attachments/files/26326914/GlyphLayoutDin91379.pdf)


### Processing a form
[GlyphLayoutFormDin91379.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutFormDin91379.java)

[GlyphLayoutFormDin91379.pdf](https://github.com/user-attachments/files/26327010/GlyphLayoutFormDin91379.pdf)



### Producing a document with bidirectional text
Java's <code>Bidi</code>-class is used to deduce the text direction for each chunk of text,
it should not be necessary to specify the text direction per font explicitly.

[GlyphLayoutBidi.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutBidi.java)

[GlyphLayoutBidi.pdf](https://github.com/user-attachments/files/26327014/GlyphLayoutBidi.pdf)


[GlyphLayoutBidiRotated.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutBidiRotated.java)

[GlyphLayoutBidiRotated.pdf](https://github.com/user-attachments/files/26327016/GlyphLayoutBidiRotated.pdf)


### Specify direction per font
It is possible to set the direction per font, but this should not be necessary.

[GlyphLayoutBidiPerFont.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutBidiPerFont.java)

[GlyphLayoutBidiPerFont.pdf](https://github.com/user-attachments/files/26326910/GlyphLayoutBidiPerFont.pdf)


### Load the font from an input stream
You can load the font from an input stream.

[GlyphLayoutInputStream.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutInputStream.java)

[GlyphLayoutInputStream.pdf](https://github.com/user-attachments/files/26327018/GlyphLayoutInputStream.pdf)


### Specify kerning and ligatures per document
Optionally you can specify kerning and ligatures per document.

[GlyphLayoutKernLiga.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutKernLiga.java)

[ GlyphLayoutKernLiga.pdf](https://github.com/user-attachments/files/26327020/GlyphLayoutKernLiga.pdf)


### Specify kerning and ligatures per font
Optionally you can specify kerning and ligatures per font.

[GlyphLayoutKernLigaPerFont.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutKernLigaPerFont.java)

[GlyphLayoutKernLigaPerFont.pdf](https://github.com/user-attachments/files/26327021/GlyphLayoutKernLigaPerFont.pdf)



### Use GlyphLayoutManager for Letters from the Unicode Supplementary Multilingual Plane
Show letters and symbols from the Unicode Supplementary Multilingual Plane,

[GlyphLayoutSMP.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutSMP.java)

[GlyphLayoutSMP.pdf](https://github.com/user-attachments/files/26327026/GlyphLayoutSMP.pdf)

### Use GlyphLayoutManager with an image

[GlyphLayoutWithImage.java](https://github.com/LibrePDF/OpenPDF/tree/master/pdf-toolbox/src/test/java/org/openpdf/examples/glyphlayout/GlyphLayoutWithImage.java)

[GlyphLayoutWithImage.pdf](https://github.com/user-attachments/files/26327030/GlyphLayoutWithImage.pdf)

### Use GlyphLayoutManager with an HTML file

[GlyphLayoutHtmlTest.java](https://github.com/vk-github18/OpenPDF-vk/blob/test-2026-05-14/openpdf-html/src/test/java/org/openpdf/pdf/GlyphLayoutHtmlExample.java)

[GlyphLayoutHtmlTest.html](https://github.com/vk-github18/OpenPDF-vk/blob/test-2026-05-14/openpdf-html/src/test/resources/org/openpdf/pdf/GlyphLayoutHtmlExample.html)

[GlyphLayoutHtmlExample.pdf](https://github.com/user-attachments/files/27773303/GlyphLayoutHtmlExample.pdf)


## Open source OpenType fonts
1. [Google Noto fonts.](https://www.google.com/get/noto/), [Latin, Greek, Cyrillic at GitHub](https://github.com/notofonts/latin-greek-cyrillic)
2. [Arimo](https://github.com/googlefonts/Arimo)
3. [Sudo coding font](https://www.kutilek.de/sudo-font/)

## References
1. [DIN 91379 (English Wikipedia)](https://en.wikipedia.org/wiki/DIN_91379)
2. [DIN 91379 (German Wikipedia)](https://de.wikipedia.org/wiki/DIN_91379)
3. [DIN 91379 Characters and Sequences (GitHub)](https://github.com/String-Latin/DIN-91379-Characters-and-Sequences)
4. [DIN 91379:2022-08: Characters and defined character sequences in Unicode for the electronic processing of names and data exchange in Europe, with CD-ROM](https://www.beuth.de/de/norm/din-91379/353496133) (access chargeable)
5. [Decision of IT Planungsrat 2022/51](https://www.it-planungsrat.de/beschluss/beschluss-2022-51) (in German)
6. [HarfBuzz text shaping library](https://harfbuzz.github.io/)
7. [HarfRust, HarfBuzz port to Rust](https://github.com/harfbuzz/harfrust)
