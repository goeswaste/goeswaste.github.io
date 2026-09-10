---
title: Gemini for pasteHandler
date: 2026-09-10 14:22:59
---

Yes — Google Gemini API is a good fit for this.

The PHP reader can send the extracted article text to Gemini, receive the translation, and then put the translated text back into the same DOM positions. Gemini’s API supports ordinary text generation through its API, and Google currently documents REST access as well as SDKs.  ￼

I would structure it like this:

URL
 ↓
Fetch webpage
 ↓
Extract article
 ↓
Parse article DOM
 ↓
┌─────────────────────────────────┐
│ <p>Text...</p>                   │ → Gemini → translated text
│ <figure><img ...></figure>      │ → untouched
│ <p>More text...</p>              │ → Gemini → translated text
│ <figure><img ...></figure>      │ → untouched
└─────────────────────────────────┘
 ↓
Render reader

The important part

Rather than giving Gemini the entire HTML, I’d give it only the text nodes. For example:
```
function translateWithGemini(string $text, string $targetLanguage): string
{
    $apiKey = getenv('GEMINI_API_KEY');

    $prompt = <<<PROMPT
Translate the following text into {$targetLanguage}.

Rules:
- Preserve the meaning and tone.
- Do not summarize.
- Do not add explanations.
- Return only the translation.

TEXT:
{$text}
PROMPT;

    $payload = [
        'contents' => [
            [
                'parts' => [
                    ['text' => $prompt]
                ]
            ]
        ]
    ];

    $ch = curl_init(
        'https://generativelanguage.googleapis.com/v1beta/' .
        'models/gemini-3.8-flash:generateContent'
    );

    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'x-goog-api-key: ' . $apiKey
        ],
        CURLOPT_POSTFIELDS => json_encode($payload),
        CURLOPT_TIMEOUT => 60
    ]);

    $response = curl_exec($ch);

    if ($response === false) {
        throw new Exception(curl_error($ch));
    }

    curl_close($ch);

    $json = json_decode($response, true);

    return trim(
        $json['candidates'][0]['content']['parts'][0]['text']
        ?? ''
    );
}
```
The REST generateContent endpoint accepts a contents/parts request and returns the generated text in the candidate content, which is exactly what we need here.  ￼

Then we can walk through the article:
```
$xpath = new DOMXPath($dom);

foreach ($xpath->query('//p | //h1 | //h2 | //h3 | //h4 | //li | //blockquote') as $element) {

    $text = trim($element->textContent);

    if ($text === '') {
        continue;
    }

    $translated = translateWithGemini(
        $text,
        'Chinese'
    );

    /*
     * Replace only the text.
     * Images elsewhere in the DOM remain untouched.
     */
    while ($element->firstChild) {
        $element->removeChild($element->firstChild);
    }

    $element->appendChild(
        $dom->createTextNode($translated)
    );
}
```
So:

<p>This is the first paragraph.</p>

<figure>
    <img src="photo.jpg">
    <figcaption>A beautiful photograph.</figcaption>
</figure>

<p>This is the second paragraph.</p>

becomes:

<p>这是第一段。</p>

<figure>
    <img src="photo.jpg">
    <figcaption>一张漂亮的照片。</figcaption>
</figure>

<p>这是第二段。</p>

The image hasn’t moved at all.

One thing I’d change from the previous version

I wouldn’t make one Gemini request for every sentence or text node. That’s inefficient.

I’d assign IDs to the translatable blocks:

text_001
text_002
text_003
...

send several blocks to Gemini in one request, and ask it to return JSON:

{
  "text_001": "翻译后的第一段",
  "text_002": "翻译后的第二段",
  "text_003": "翻译后的第三段"
}

Gemini supports structured output, so this can be made considerably more reliable than parsing arbitrary generated text.  ￼

That gives us a much better architecture:

                   ┌── paragraph 1 ──┐
                   ├── paragraph 2 ──┤
Article DOM ───────┼── paragraph 3 ──┼──→ Gemini
                   ├── paragraph 4 ──┤
                   └── paragraph 5 ──┘
                                      │
                                      ↓
                              Translated JSON
                                      │
                                      ↓
                              Put text back
                                      │
                                      ↓
                    ┌─────────────────────────┐
                    │ translated paragraph 1  │
                    │ image                   │
                    │ translated paragraph 2  │
                    │ image                   │
                    │ translated paragraph 3  │
                    └─────────────────────────┘

One other consideration: don’t put your Gemini API key in JavaScript/browser code. Keep it server-side in an environment variable such as GEMINI_API_KEY, and have PHP make the Gemini request.

If you want, I can turn the previous reader.php into the complete single-file version with Gemini translation + a language dropdown + dark mode + lazy images, with no Composer dependency.