---
title: pasteHandler
date: 2026-09-10 14:27:12
---

```
<?php
/*
 * reader.php
 *
 * Single-file AI article reader.
 *
 * Requirements:
 *   PHP 8+
 *   cURL extension
 *   DOM extension
 *
 * No Composer.
 *
 * Features:
 *   - URL input
 *   - Reader-mode article extraction
 *   - Lazy-loaded image support
 *   - Images remain in their original positions
 *   - Gemini AI translation
 *   - Language selector
 *   - Dark mode
 *   - Server-side Gemini API key
 *
 * IMPORTANT:
 *   This does not bypass paywalls, authentication, subscriptions,
 *   robots restrictions, or other access controls.
 */


/* ============================================================
 * CONFIGURATION
 * ============================================================
 */

const GEMINI_MODEL = 'gemini-3.8-flash';

$geminiKey = getenv('GEMINI_API_KEY');


/* Languages shown in the dropdown. */
$languages = [
    'English',
    'Simplified Chinese',
    'Traditional Chinese',
    'Japanese',
    'Korean',
    'French',
    'German',
    'Spanish',
    'Italian',
    'Portuguese',
    'Dutch',
    'Russian',
    'Arabic',
    'Hindi',
    'Thai',
    'Vietnamese'
];


/* ============================================================
 * REQUEST STATE
 * ============================================================
 */

$url = trim($_POST['url'] ?? '');

$targetLanguage =
    trim($_POST['language'] ?? 'English');

$title = '';
$author = '';
$content = '';

$error = '';

$translated = false;


/* Validate selected language. */
if (!in_array($targetLanguage, $languages, true)) {
    $targetLanguage = 'English';
}


/* ============================================================
 * MAIN REQUEST
 * ============================================================
 */

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    if (!$geminiKey) {

        $error =
            'GEMINI_API_KEY is not configured on the server.';

    } elseif (!filter_var($url, FILTER_VALIDATE_URL)) {

        $error =
            'Please enter a valid URL.';

    } else {

        $scheme =
            strtolower(
                parse_url($url, PHP_URL_SCHEME)
            );

        if (!in_array($scheme, ['http', 'https'], true)) {

            $error =
                'Only HTTP and HTTPS URLs are supported.';

        } else {

            try {

                /*
                 * Download webpage.
                 */
                [$html, $finalUrl] =
                    fetchPage($url);


                /*
                 * Extract readable article.
                 */
                [
                    $title,
                    $author,
                    $content
                ] = extractArticle(
                    $html,
                    $finalUrl
                );


                if (!$content) {
                    throw new Exception(
                        'No readable article content was found.'
                    );
                }


                /*
                 * Translate the article if requested.
                 *
                 * We translate text blocks inside the DOM.
                 * Images and their positions are never sent
                 * to Gemini and are therefore never modified.
                 */
                if (
                    $targetLanguage !== 'English'
                    ||
                    shouldTranslateEnglish()
                ) {

                    $content =
                        translateArticle(
                            $content,
                            $targetLanguage,
                            $geminiKey
                        );

                    $translated = true;
                }

            } catch (Throwable $e) {

                $error = $e->getMessage();
            }
        }
    }
}


/* ============================================================
 * FETCH PAGE
 * ============================================================
 */

function fetchPage(string $url): array
{
    $ch = curl_init($url);

    curl_setopt_array($ch, [

        CURLOPT_RETURNTRANSFER => true,

        CURLOPT_FOLLOWLOCATION => true,

        CURLOPT_MAXREDIRS => 5,

        CURLOPT_CONNECTTIMEOUT => 10,

        CURLOPT_TIMEOUT => 25,

        CURLOPT_ENCODING => '',

        CURLOPT_USERAGENT =>
            'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ' .
            'AppleWebKit/605.1.15 (KHTML, like Gecko) ' .
            'Version/17.0 Safari/605.1.15',

        CURLOPT_HTTPHEADER => [
            'Accept: text/html,application/xhtml+xml',
            'Accept-Language: en-US,en;q=0.9'
        ]
    ]);

    $html = curl_exec($ch);

    if ($html === false) {

        $message = curl_error($ch);

        curl_close($ch);

        throw new Exception(
            'Could not retrieve the page: ' . $message
        );
    }

    $status =
        curl_getinfo(
            $ch,
            CURLINFO_HTTP_CODE
        );

    $contentType =
        curl_getinfo(
            $ch,
            CURLINFO_CONTENT_TYPE
        );

    $finalUrl =
        curl_getinfo(
            $ch,
            CURLINFO_EFFECTIVE_URL
        );

    curl_close($ch);


    if ($status >= 400) {

        throw new Exception(
            "The website returned HTTP $status."
        );
    }


    if (
        $contentType
        &&
        stripos(
            $contentType,
            'text/html'
        ) === false
    ) {

        throw new Exception(
            'The URL does not contain an HTML page.'
        );
    }


    /*
     * Don't allow enormous responses.
     */
    if (strlen($html) > 15 * 1024 * 1024) {

        throw new Exception(
            'The downloaded page is too large.'
        );
    }


    return [
        $html,
        $finalUrl ?: $url
    ];
}


/* ============================================================
 * ARTICLE EXTRACTION
 * ============================================================
 */

function extractArticle(
    string $html,
    string $baseUrl
): array {

    libxml_use_internal_errors(true);

    $dom = new DOMDocument();

    $dom->loadHTML(
        '<?xml encoding="UTF-8">' . $html,
        LIBXML_NOERROR |
        LIBXML_NOWARNING
    );

    libxml_clear_errors();

    $xpath = new DOMXPath($dom);


    /*
     * Remove elements that are almost never part of
     * article content.
     */
    $removeQueries = [

        '//script',

        '//style',

        '//noscript',

        '//iframe',

        '//object',

        '//embed',

        '//video',

        '//audio',

        '//canvas',

        '//form',

        '//input',

        '//button',

        '//select',

        '//textarea',

        '//nav',

        '//footer',

        '//aside',

        '//dialog',

        '//svg',

        '//template'
    ];


    foreach ($removeQueries as $query) {

        foreach (
            $xpath->query($query)
            as $node
        ) {

            if ($node->parentNode) {

                $node->parentNode
                     ->removeChild($node);
            }
        }
    }


    /*
     * Remove obvious advertising / social / navigation
     * containers.
     */
    $badWords = [

        'advert',
        'advertisement',
        'ad-container',
        'ad-wrapper',
        'banner',

        'cookie',
        'consent',

        'newsletter',

        'subscribe',
        'subscription',

        'social',
        'share',
        'sharing',

        'related',
        'recommend',
        'recommended',

        'comments',
        'comment',

        'sidebar',
        'breadcrumb',

        'popup',
        'modal',
        'overlay',

        'promo',
        'promotion',

        'sponsor',
        'sponsored',

        'outbrain',
        'taboola',
        'disqus'
    ];


    foreach (
        $xpath->query(
            '//*[@id or @class]'
        )
        as $node
    ) {

        $value =
            strtolower(
                $node->getAttribute('id')
                . ' '
                .
                $node->getAttribute('class')
            );


        foreach ($badWords as $word) {

            if (
                preg_match(
                    '/(^|[\s_-])'
                    . preg_quote($word, '/')
                    . '($|[\s_-])/i',
                    $value
                )
            ) {

                if ($node->parentNode) {

                    $node->parentNode
                         ->removeChild($node);
                }

                break;
            }
        }
    }


    /*
     * Title.
     */
    $title = '';

    $titleNode =
        $xpath->query('//title')
              ->item(0);

    if ($titleNode) {

        $title =
            cleanText(
                $titleNode->textContent
            );
    }


    /*
     * Prefer OpenGraph title.
     */
    $ogTitle =
        $xpath->query(
            '//meta[@property="og:title"]/@content'
        )->item(0);

    if (
        $ogTitle
        &&
        trim($ogTitle->nodeValue)
    ) {

        $title =
            cleanText(
                $ogTitle->nodeValue
            );
    }


    /*
     * Author.
     */
    $author = '';

    $authorQueries = [

        '//meta[@name="author"]/@content',

        '//meta[@property="article:author"]/@content',

        '//*[@rel="author"]',

        '//*[contains(
            translate(@class,
            "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
            "abcdefghijklmnopqrstuvwxyz"),
            "author"
        )]'
    ];


    foreach ($authorQueries as $query) {

        $node =
            $xpath->query($query)
                  ->item(0);

        if ($node) {

            $author =
                cleanText(
                    $node->nodeValue
                    ??
                    $node->textContent
                );

            if ($author) {
                break;
            }
        }
    }


    /*
     * Find article candidate.
     */
    $candidates = [];


    $queries = [

        '//article',

        '//*[@role="main"]',

        '//*[contains(
            translate(@class,
            "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
            "abcdefghijklmnopqrstuvwxyz"),
            "article"
        )]',

        '//*[contains(
            translate(@class,
            "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
            "abcdefghijklmnopqrstuvwxyz"),
            "post-content"
        )]',

        '//*[contains(
            translate(@class,
            "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
            "abcdefghijklmnopqrstuvwxyz"),
            "entry-content"
        )]',

        '//*[contains(
            translate(@class,
            "ABCDEFGHIJKLMNOPQRSTUVWXYZ",
            "abcdefghijklmnopqrstuvwxyz"),
            "story"
        )]',

        '//main',

        '//body'
    ];


    foreach ($queries as $query) {

        foreach (
            $xpath->query($query)
            as $node
        ) {

            $text =
                cleanText(
                    $node->textContent
                );

            $paragraphs =
                $xpath->query(
                    './/p',
                    $node
                )->length;

            $images =
                $xpath->query(
                    './/img',
                    $node
                )->length;


            $score =
                strlen($text)
                +
                ($paragraphs * 250)
                +
                ($images * 100);


            if (strlen($text) < 300) {
                $score -= 1000;
            }


            $candidates[] = [

                'node' => $node,

                'score' => $score
            ];
        }
    }


    if (!$candidates) {

        return [
            $title,
            $author,
            ''
        ];
    }


    usort(
        $candidates,
        fn($a, $b) =>
            $b['score'] <=> $a['score']
    );


    $article =
        $candidates[0]['node'];


    /*
     * Remove hidden content.
     */
    foreach (
        $xpath->query(
            './/*[@hidden or @aria-hidden="true"]',
            $article
        )
        as $node
    ) {

        if ($node->parentNode) {

            $node->parentNode
                 ->removeChild($node);
        }
    }


    /*
     * Process images.
     */
    foreach (
        $xpath->query(
            './/img',
            $article
        )
        as $img
    ) {

        $src = '';


        /*
         * Common lazy-loading attributes.
         */
        $attributes = [

            'src',

            'data-src',

            'data-original',

            'data-lazy-src',

            'data-url',

            'data-image'
        ];


        foreach ($attributes as $attribute) {

            if (
                $img->hasAttribute($attribute)
            ) {

                $candidate =
                    trim(
                        $img->getAttribute(
                            $attribute
                        )
                    );


                if ($candidate) {

                    $src = $candidate;

                    break;
                }
            }
        }


        /*
         * Lazy srcset.
         */
        if (
            !$src
            &&
            $img->hasAttribute(
                'data-srcset'
            )
        ) {

            $src =
                extractFirstSrcsetUrl(
                    $img->getAttribute(
                        'data-srcset'
                    )
                );
        }


        if (
            !$src
            &&
            $img->hasAttribute('srcset')
        ) {

            $src =
                extractFirstSrcsetUrl(
                    $img->getAttribute('srcset')
                );
        }


        if (!$src) {

            if ($img->parentNode) {

                $img->parentNode
                    ->removeChild($img);
            }

            continue;
        }


        $src =
            absoluteUrl(
                $src,
                $baseUrl
            );


        if (!$src) {

            if ($img->parentNode) {

                $img->parentNode
                    ->removeChild($img);
            }

            continue;
        }


        $img->setAttribute(
            'src',
            $src
        );


        if (
            !$img->hasAttribute('alt')
        ) {

            $img->setAttribute(
                'alt',
                ''
            );
        }


        /*
         * Remove lazy-loading attributes.
         */
        $removeAttributes = [

            'srcset',
            'sizes',

            'data-src',
            'data-srcset',
            'data-original',
            'data-lazy-src',
            'data-url',
            'data-image',

            'loading',

            'onclick',
            'onload'
        ];


        foreach (
            $removeAttributes
            as $attribute
        ) {

            $img->removeAttribute(
                $attribute
            );
        }
    }


    /*
     * Clean attributes.
     */
    $allowedAttributes = [

        'href',
        'src',
        'alt',
        'title',
        'width',
        'height'
    ];


    foreach (
        $xpath->query(
            './/*',
            $article
        )
        as $node
    ) {

        if (!$node->hasAttributes()) {
            continue;
        }


        $attributes = [];


        foreach ($node->attributes as $attribute) {

            $attributes[] =
                $attribute->name;
        }


        foreach ($attributes as $attribute) {

            if (
                !in_array(
                    strtolower($attribute),
                    $allowedAttributes,
                    true
                )
            ) {

                $node->removeAttribute(
                    $attribute
                );
            }
        }
    }


    /*
     * Make links absolute.
     */
    foreach (
        $xpath->query(
            './/a',
            $article
        )
        as $link
    ) {

        $href =
            trim(
                $link->getAttribute('href')
            );


        if (!$href) {
            continue;
        }


        $absolute =
            absoluteUrl(
                $href,
                $baseUrl
            );


        if ($absolute) {

            $link->setAttribute(
                'href',
                $absolute
            );

            $link->setAttribute(
                'target',
                '_blank'
            );

            $link->setAttribute(
                'rel',
                'noopener noreferrer'
            );
        }
    }


    /*
     * Return article HTML.
     */
    $content = '';

    foreach (
        $article->childNodes
        as $child
    ) {

        $content .=
            $dom->saveHTML($child);
    }


    return [
        $title,
        $author,
        $content
    ];
}


/* ============================================================
 * GEMINI TRANSLATION
 * ============================================================
 */

function translateArticle(
    string $html,
    string $targetLanguage,
    string $apiKey
): string {

    libxml_use_internal_errors(true);

    $dom = new DOMDocument();

    $dom->loadHTML(
        '<?xml encoding="UTF-8">' . $html,
        LIBXML_NOERROR |
        LIBXML_NOWARNING
    );

    libxml_clear_errors();

    $xpath = new DOMXPath($dom);


    /*
     * These are the elements whose textual contents
     * should be translated.
     *
     * Images are deliberately NOT in this list.
     */
    $query = implode('|', [

        './/p',

        './/h1',
        './/h2',
        './/h3',
        './/h4',

        './/li',

        './/blockquote',

        './/figcaption',

        './/dt',
        './/dd',

        './/th',
        './/td'
    ]);


    $elements =
        $xpath->query(
            $query,
            $dom
        );


    /*
     * Build translation blocks.
     *
     * Each block receives an ID.
     */
    $blocks = [];

    $counter = 1;


    foreach ($elements as $element) {

        /*
         * Don't translate elements containing
         * nested translatable elements.
         *
         * This prevents a paragraph containing
         * a nested <li>, etc. from being duplicated.
         */
        $nested =
            $xpath->query(
                './/p | .//h1 | .//h2 | .//h3 |
                 .//h4 | .//li | .//blockquote |
                 .//figcaption | .//dt | .//dd |
                 .//th | .//td',
                $element
            );


        /*
         * If this element contains another candidate,
         * skip the parent.
         */
        if ($nested->length > 0) {
            continue;
        }


        $text =
            trim(
                $element->textContent
            );


        if (!$text) {
            continue;
        }


        /*
         * Avoid translating extremely short UI-like text.
         */
        if (
            mb_strlen($text, 'UTF-8') < 2
        ) {
            continue;
        }


        $id =
            'text_' .
            str_pad(
                (string)$counter,
                5,
                '0',
                STR_PAD_LEFT
            );


        $blocks[$id] = [

            'element' => $element,

            'text' => $text
        ];


        $counter++;
    }


    /*
     * Process in batches.
     *
     * 25 blocks per request is a reasonable balance
     * between API efficiency and response reliability.
     */
    $chunks =
        array_chunk(
            $blocks,
            25,
            true
        );


    foreach ($chunks as $chunk) {

        $translations =
            translateBatch(
                $chunk,
                $targetLanguage,
                $apiKey
            );


        foreach (
            $chunk
            as $id => $block
        ) {

            if (
                !isset(
                    $translations[$id]
                )
            ) {
                continue;
            }


            $translation =
                trim(
                    (string)$translations[$id]
                );


            if (!$translation) {
                continue;
            }


            $element =
                $block['element'];


            /*
             * Replace the text INSIDE the element.
             *
             * The element itself remains in exactly
             * the same position in the DOM.
             *
             * Images outside this text element therefore
             * remain completely untouched.
             */
            while (
                $element->firstChild
            ) {

                $element->removeChild(
                    $element->firstChild
                );
            }


            $element->appendChild(
                $dom->createTextNode(
                    $translation
                )
            );
        }
    }


    /*
     * Return translated HTML.
     */
    $result = '';

    $body =
        $dom->getElementsByTagName(
            'body'
        )->item(0);


    if ($body) {

        foreach (
            $body->childNodes
            as $child
        ) {

            $result .=
                $dom->saveHTML($child);
        }

    } else {

        $result =
            $dom->saveHTML();
    }


    return $result;
}


/* ============================================================
 * GEMINI BATCH REQUEST
 * ============================================================
 */

function translateBatch(
    array $blocks,
    string $targetLanguage,
    string $apiKey
): array {

    $items = [];


    foreach (
        $blocks
        as $id => $block
    ) {

        $items[] = [

            'id' => $id,

            'text' => $block['text']
        ];
    }


    $jsonInput =
        json_encode(
            $items,
            JSON_UNESCAPED_UNICODE |
            JSON_UNESCAPED_SLASHES
        );


    /*
     * Ask Gemini to return ONLY a JSON object:
     *
     * {
     *   "text_00001": "...",
     *   "text_00002": "..."
     * }
     */
    $prompt = <<<PROMPT
You are translating an article for a web reader.

Translate every item into {$targetLanguage}.

Rules:
1. Preserve the exact meaning.
2. Do not summarize.
3. Do not add explanations.
4. Do not omit information.
5. Preserve names, numbers, dates, URLs, and technical terms appropriately.
6. Return ONLY valid JSON.
7. The JSON keys must be exactly the same IDs supplied below.
8. The value for each ID must contain only its translated text.

Input:

{$jsonInput}
PROMPT;


    $payload = [

        'contents' => [

            [

                'role' => 'user',

                'parts' => [

                    [
                        'text' => $prompt
                    ]
                ]
            ]
        ],

        'generationConfig' => [

            'responseMimeType' =>
                'application/json',

            'temperature' => 0.2
        ]
    ];


    $endpoint =
        'https://generativelanguage.googleapis.com/' .
        'v1beta/models/' .
        GEMINI_MODEL .
        ':generateContent';


    $ch =
        curl_init($endpoint);


    curl_setopt_array($ch, [

        CURLOPT_RETURNTRANSFER => true,

        CURLOPT_POST => true,

        CURLOPT_CONNECTTIMEOUT => 10,

        CURLOPT_TIMEOUT => 90,

        CURLOPT_HTTPHEADER => [

            'Content-Type: application/json',

            'x-goog-api-key: ' . $apiKey
        ],

        CURLOPT_POSTFIELDS =>
            json_encode(
                $payload,
                JSON_UNESCAPED_UNICODE
            )
    ]);


    $response =
        curl_exec($ch);


    if ($response === false) {

        $message =
            curl_error($ch);

        curl_close($ch);

        throw new Exception(
            'Gemini request failed: ' .
            $message
        );
    }


    $httpCode =
        curl_getinfo(
            $ch,
            CURLINFO_HTTP_CODE
        );


    curl_close($ch);


    if ($httpCode >= 400) {

        $decoded =
            json_decode(
                $response,
                true
            );


        $message =
            $decoded['error']['message']
            ??
            'Unknown Gemini API error.';


        throw new Exception(
            'Gemini API error: ' .
            $message
        );
    }


    $responseJson =
        json_decode(
            $response,
            true
        );


    $text =
        $responseJson
        ['candidates'][0]
        ['content']['parts'][0]
        ['text']
        ?? '';


    if (!$text) {

        throw new Exception(
            'Gemini returned an empty translation.'
        );
    }


    /*
     * Decode Gemini's JSON response.
     */
    $translations =
        json_decode(
            trim($text),
            true
        );


    /*
     * Fallback for accidental Markdown fences.
     */
    if (!is_array($translations)) {

        $text =
            preg_replace(
                '/^```(?:json)?\s*/i',
                '',
                trim($text)
            );

        $text =
            preg_replace(
                '/\s*```$/',
                '',
                $text
            );


        $translations =
            json_decode(
                $text,
                true
            );
    }


    if (!is_array($translations)) {

        throw new Exception(
            'Gemini returned invalid translation JSON.'
        );
    }


    return $translations;
}


/* ============================================================
 * URL HELPERS
 * ============================================================
 */

function absoluteUrl(
    string $url,
    string $base
): ?string {

    $url =
        trim($url);


    if (!$url) {
        return null;
    }


    /*
     * Never allow executable/data URLs.
     */
    if (
        preg_match(
            '#^(javascript|data|blob):#i',
            $url
        )
    ) {

        return null;
    }


    if (
        preg_match(
            '#^https?://#i',
            $url
        )
    ) {

        return $url;
    }


    $baseParts =
        parse_url($base);


    if (!$baseParts) {
        return null;
    }


    $scheme =
        $baseParts['scheme']
        ?? 'https';


    $host =
        $baseParts['host']
        ?? '';


    if (
        str_starts_with(
            $url,
            '//'
        )
    ) {

        return $scheme . ':' . $url;
    }


    if (
        str_starts_with(
            $url,
            '/'
        )
    ) {

        return
            $scheme .
            '://' .
            $host .
            $url;
    }


    $basePath =
        $baseParts['path']
        ?? '/';


    $directory =
        rtrim(
            str_replace(
                '\\',
                '/',
                dirname($basePath)
            ),
            '/'
        );


    $full =
        $directory .
        '/' .
        $url;


    $segments = [];


    foreach (
        explode(
            '/',
            $full
        )
        as $segment
    ) {

        if (
            $segment === ''
            ||
            $segment === '.'
        ) {
            continue;
        }


        if ($segment === '..') {

            array_pop($segments);

        } else {

            $segments[] =
                $segment;
        }
    }


    return
        $scheme .
        '://' .
        $host .
        '/' .
        implode(
            '/',
            $segments
        );
}


function extractFirstSrcsetUrl(
    string $srcset
): string {

    $items =
        preg_split(
            '/\s*,\s*/',
            trim($srcset)
        );


    if (!$items) {
        return '';
    }


    $first =
        trim($items[0]);


    return
        preg_split(
            '/\s+/',
            $first
        )[0]
        ?? '';
}


function cleanText(
    string $text
): string {

    return trim(
        preg_replace(
            '/\s+/u',
            ' ',
            html_entity_decode(
                $text,
                ENT_QUOTES |
                ENT_HTML5,
                'UTF-8'
            )
        )
    );
}


/*
 * Currently kept as a separate function so the translation
 * behavior can easily be changed later.
 */
function shouldTranslateEnglish(): bool
{
    return false;
}

?>
<!DOCTYPE html>

<html lang="en">

<head>

<meta charset="UTF-8">

<meta
    name="viewport"
    content="width=device-width, initial-scale=1.0"
>

<title>
<?= htmlspecialchars(
    $title ?: 'Reader',
    ENT_QUOTES,
    'UTF-8'
) ?>
</title>


<script>

/*
 * Restore dark mode before the page paints.
 */
(function () {

    if (
        localStorage.getItem(
            'reader-theme'
        ) === 'dark'
    ) {

        document.documentElement
            .classList
            .add('dark');
    }

})();

</script>


<style>

:root {

    color-scheme: light;

    --bg: #f5f5f7;

    --paper: #ffffff;

    --text: #1d1d1f;

    --muted: #6e6e73;

    --border: #d2d2d7;

    --input: rgba(255,255,255,.94);

    --button: #007aff;

    --button-text: white;

    --quote: #f2f2f7;
}


html.dark {

    color-scheme: dark;

    --bg: #111111;

    --paper: #1c1c1e;

    --text: #f5f5f7;

    --muted: #a1a1a6;

    --border: #3a3a3c;

    --input: #2c2c2e;

    --button: #0a84ff;

    --button-text: white;

    --quote: #2c2c2e;
}


* {
    box-sizing: border-box;
}


html {
    background: var(--bg);
}


body {

    margin: 0;

    background: var(--bg);

    color: var(--text);

    font-family:
        -apple-system,
        BlinkMacSystemFont,
        "SF Pro Text",
        "Helvetica Neue",
        Arial,
        sans-serif;

    transition:
        background .2s ease,
        color .2s ease;
}


/* ============================================================
   TOOLBAR
   ============================================================
 */

.toolbar {

    position: sticky;

    top: 0;

    z-index: 1000;

    padding: 11px 14px;

    background:
        color-mix(
            in srgb,
            var(--bg) 88%,
            transparent
        );

    backdrop-filter: blur(20px);

    -webkit-backdrop-filter: blur(20px);

    border-bottom:
        1px solid
        color-mix(
            in srgb,
            var(--border) 55%,
            transparent
        );
}


.toolbar-inner {

    max-width: 1000px;

    margin: auto;

    display: flex;

    gap: 8px;
}


.url-form {

    display: flex;

    flex: 1;

    gap: 8px;

    min-width: 0;
}


.url-input {

    flex: 1;

    min-width: 0;

    height: 43px;

    padding: 0 14px;

    border:
        1px solid
        var(--border);

    border-radius: 12px;

    background: var(--input);

    color: var(--text);

    outline: none;

    font-size: 15px;
}


.url-input::placeholder {
    color: var(--muted);
}


.url-input:focus {

    border-color:
        var(--button);

    box-shadow:
        0 0 0 3px
        color-mix(
            in srgb,
            var(--button) 18%,
            transparent
        );
}


.language {

    height: 43px;

    max-width: 170px;

    padding: 0 11px;

    border:
        1px solid
        var(--border);

    border-radius: 12px;

    background: var(--input);

    color: var(--text);

    font-size: 14px;

    outline: none;
}


.read-button {

    height: 43px;

    padding: 0 17px;

    border: 0;

    border-radius: 12px;

    background: var(--button);

    color: var(--button-text);

    font-weight: 600;

    cursor: pointer;
}


.theme-button {

    width: 43px;

    height: 43px;

    flex: 0 0 43px;

    border: 0;

    border-radius: 12px;

    background: var(--quote);

    color: var(--text);

    font-size: 19px;

    cursor: pointer;
}


/* ============================================================
   READER
   ============================================================
 */

.reader {

    width:
        min(
            calc(100% - 32px),
            820px
        );

    margin:
        46px auto
        100px;

    padding:
        clamp(32px, 6vw, 70px)
        clamp(24px, 7vw, 76px);

    background: var(--paper);

    border-radius: 20px;

    box-shadow:
        0 15px 50px
        rgba(0,0,0,.07);
}


.article-header {
    margin-bottom: 42px;
}


.article-title {

    margin: 0 0 17px;

    font-family:
        Georgia,
        "Times New Roman",
        serif;

    font-size:
        clamp(34px, 5vw, 54px);

    line-height: 1.08;

    letter-spacing: -1.5px;
}


.article-meta {

    color: var(--muted);

    font-size: 15px;
}


/* ============================================================
   ARTICLE
   ============================================================
 */

.article {

    font-family:
        Georgia,
        "Times New Roman",
        serif;

    font-size: 20px;

    line-height: 1.72;
}


.article p {
    margin: 0 0 1.35em;
}


.article h1,
.article h2,
.article h3,
.article h4 {

    font-family:
        -apple-system,
        BlinkMacSystemFont,
        "SF Pro Display",
        "Helvetica Neue",
        Arial,
        sans-serif;

    line-height: 1.2;

    letter-spacing: -.5px;

    margin-top: 1.7em;

    margin-bottom: .7em;
}


.article h1 {
    font-size: 36px;
}


.article h2 {
    font-size: 30px;
}


.article h3 {
    font-size: 24px;
}


.article h4 {
    font-size: 21px;
}


.article a {
    color: var(--button);
}


.article blockquote {

    margin: 30px 0;

    padding:
        5px 0 5px 23px;

    border-left:
        4px solid
        var(--border);

    color: var(--muted);

    font-style: italic;
}


.article ul,
.article ol {

    padding-left: 1.4em;

    margin-bottom: 1.4em;
}


.article li {
    margin-bottom: .5em;
}


/* ============================================================
   IMAGES
   ============================================================
 */

.article img {

    display: block;

    width: auto;

    max-width: 100%;

    height: auto;

    margin:
        34px auto;

    border-radius: 10px;

    background: var(--quote);
}


.article figure {
    margin: 36px 0;
}


.article figure img {
    margin: 0 auto;
}


.article figcaption {

    margin-top: 9px;

    color: var(--muted);

    font-family:
        -apple-system,
        BlinkMacSystemFont,
        "Helvetica Neue",
        Arial,
        sans-serif;

    font-size: 14px;

    line-height: 1.45;

    text-align: center;
}


/* ============================================================
   ERROR
   ============================================================
 */

.error {

    width:
        min(
            calc(100% - 32px),
            820px
        );

    margin: 35px auto;

    padding: 17px 19px;

    border-radius: 13px;

    background: #351717;

    color: #ffb4b4;

    font-size: 15px;
}


/* ============================================================
   EMPTY
   ============================================================
 */

.empty {

    width:
        min(
            calc(100% - 32px),
            650px
        );

    margin: 110px auto;

    text-align: center;

    color: var(--muted);
}


.empty-icon {

    font-size: 45px;

    margin-bottom: 14px;
}


.empty h1 {

    margin: 0 0 8px;

    color: var(--text);

    font-size: 27px;
}


/* ============================================================
   MOBILE
   ============================================================
 */

@media (max-width: 700px) {

    .toolbar-inner {
        flex-wrap: wrap;
    }


    .url-form {
        width: 100%;
    }


    .language {

        flex: 1;

        max-width: none;
    }


    .reader {

        width: 100%;

        margin: 0;

        padding:
            35px 22px
            70px;

        border-radius: 0;

        box-shadow: none;
    }


    .article {

        font-size: 19px;

        line-height: 1.7;
    }


    .article img {

        width:
            calc(100% + 20px);

        max-width:
            calc(100% + 20px);

        margin-left: -10px;

        border-radius: 7px;
    }
}

</style>

</head>


<body>


<header class="toolbar">

    <div class="toolbar-inner">

        <form
            method="POST"
            class="url-form"
        >

            <input
                class="url-input"
                type="url"
                name="url"
                value="<?= htmlspecialchars(
                    $url,
                    ENT_QUOTES,
                    'UTF-8'
                ) ?>"
                placeholder="Paste an article URL…"
                autocomplete="url"
                required
            >


            <select
                class="language"
                name="language"
                aria-label="Translation language"
            >

                <?php foreach (
                    $languages
                    as $language
                ): ?>

                    <option
                        value="<?= htmlspecialchars(
                            $language,
                            ENT_QUOTES,
                            'UTF-8'
                        ) ?>"
                        <?= $language ===
                            $targetLanguage
                            ? 'selected'
                            : '' ?>
                    >
                        <?= htmlspecialchars(
                            $language,
                            ENT_QUOTES,
                            'UTF-8'
                        ) ?>
                    </option>

                <?php endforeach; ?>

            </select>


            <button
                class="read-button"
                type="submit"
            >
                Read
            </button>

        </form>


        <button
            class="theme-button"
            id="themeButton"
            type="button"
            title="Toggle dark mode"
            aria-label="Toggle dark mode"
        >
            ◐
        </button>

    </div>

</header>


<?php if ($error): ?>

    <div class="error">

        <?= htmlspecialchars(
            $error,
            ENT_QUOTES,
            'UTF-8'
        ) ?>

    </div>


<?php elseif ($content): ?>

    <main class="reader">

        <header class="article-header">

            <?php if ($title): ?>

                <h1 class="article-title">

                    <?= htmlspecialchars(
                        $title,
                        ENT_QUOTES,
                        'UTF-8'
                    ) ?>

                </h1>

            <?php endif; ?>


            <?php if ($author): ?>

                <div class="article-meta">

                    <?= htmlspecialchars(
                        $author,
                        ENT_QUOTES,
                        'UTF-8'
                    ) ?>

                </div>

            <?php endif; ?>

        </header>


        <article class="article">

            <?= $content ?>

        </article>

    </main>


<?php else: ?>

    <section class="empty">

        <div class="empty-icon">
            📖
        </div>


        <h1>
            Reader
        </h1>


        <p>
            Paste an article URL above, choose a language,
            and press Read.
        </p>

    </section>

<?php endif; ?>


<script>

/*
 * Dark-mode switch.
 */
const themeButton =
    document.getElementById(
        'themeButton'
    );


function updateThemeIcon() {

    const dark =
        document.documentElement
            .classList
            .contains('dark');


    themeButton.textContent =
        dark ? '☀' : '◐';
}


themeButton.addEventListener(
    'click',
    function () {

        const dark =
            document.documentElement
                .classList
                .toggle('dark');


        localStorage.setItem(
            'reader-theme',
            dark ? 'dark' : 'light'
        );


        updateThemeIcon();
    }
);


updateThemeIcon();

</script>


</body>

</html>
```