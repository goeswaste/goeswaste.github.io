---
title: Headless browser
---

The simplest reliable setup is **Playwright + headless Chromium** on your VPS. It loads the page like a browser, executes JavaScript, waits for the rendered content, and lets you extract HTML, text, screenshots, or API responses.

### 1. Install dependencies on Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y python3 python3-venv
```

Create an application directory:

```bash
mkdir -p ~/browser-fetcher
cd ~/browser-fetcher

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install playwright
python -m playwright install --with-deps chromium
```

Playwright requires a separate browser-install step; `--with-deps` installs the Linux libraries Chromium needs. <citation src="1,3"></citation>

### 2. Create a rendering script

Create `render.py`:

```python
import sys
from pathlib import Path
from playwright.sync_api import sync_playwright


def render_page(url: str):
    with sync_playwright() as p:
        browser = p.chromium.launch(
            headless=True,
            args=[
                "--no-sandbox",
                "--disable-dev-shm-usage",
            ],
        )

        context = browser.new_context(
            viewport={"width": 1440, "height": 900},
            java_script_enabled=True,
        )

        page = context.new_page()
        page.set_default_timeout(30_000)

        # Log browser requests and responses if needed
        page.on("request", lambda request:
                print(f">> {request.method} {request.url}", file=sys.stderr))

        page.on("response", lambda response:
                print(f"<< {response.status} {response.url}", file=sys.stderr))

        # Load the page
        page.goto(url, wait_until="domcontentloaded")

        # Prefer waiting for the actual content you need.
        # Replace this selector with one from the target page.
        # page.locator(".product-list").wait_for()

        # A short fallback wait for client-side rendering.
        page.wait_for_timeout(2_000)

        result = {
            "url": page.url,
            "title": page.title(),
            "html": page.content(),
            "text": page.locator("body").inner_text(),
        }

        Path("page.html").write_text(result["html"], encoding="utf-8")
        Path("page.txt").write_text(result["text"], encoding="utf-8")

        page.screenshot(path="page.png", full_page=True)

        browser.close()
        return result


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} https://example.com")
        sys.exit(1)

    output = render_page(sys.argv[1])
    print(f"Title: {output['title']}")
    print(f"Rendered HTML saved to page.html")
```

Run it:

```bash
source ~/browser-fetcher/.venv/bin/activate
cd ~/browser-fetcher

python render.py "https://example.com"
```

`page.content()` returns the DOM after JavaScript has run, unlike Python `requests`, which only retrieves the initial HTTP response. Playwright supports monitoring page requests and responses directly. <citation src="1,2"></citation>

### 3. Wait for the correct rendered element

Do not rely primarily on a fixed sleep. Wait for an element that indicates the data is available:

```python
page.goto(url, wait_until="domcontentloaded")

page.locator("[data-testid='results']").wait_for(
    state="visible",
    timeout=30_000
)

html = page.content()
```

Or:

```python
page.wait_for_selector("main article")
```

For pages that continuously poll, use a specific selector rather than `networkidle`, because such pages may never become completely idle. <citation src="1,4"></citation>

### 4. Extract structured data

For example:

```python
page.goto(url, wait_until="domcontentloaded")

page.locator(".product").first.wait_for()

products = page.locator(".product").evaluate_all("""
    elements => elements.map(element => ({
        name: element.querySelector(".name")?.innerText.trim(),
        price: element.querySelector(".price")?.innerText.trim(),
        url: element.querySelector("a")?.href
    }))
""")

print(products)
```

If the page is server-side rendered, you may be able to retrieve the data with ordinary HTTP:

```python
import requests

response = requests.get(
    "https://example.com",
    timeout=30,
    headers={"User-Agent": "Mozilla/5.0"},
)
html = response.text
```

Use Playwright when the content appears only after JavaScript execution, interaction, authentication, scrolling, or client-side API requests.

### 5. Capture the underlying API response

Often the best approach is to let the browser run, then capture the JSON request that supplies the page data:

```python
import json
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()

    with page.expect_response("**/api/products**") as response_info:
        page.goto("https://example.com/products")

    response = response_info.value
    data = response.json()

    print(json.dumps(data, indent=2))

    browser.close()
```

If an API request occurs after clicking a button:

```python
with page.expect_response("**/api/search**") as response_info:
    page.get_by_role("button", name="Search").click()

data = response_info.value.json()
```

Playwright supports waiting for a particular response using URL patterns or predicates. <citation src="2"></citation>

### 6. Run it as a service

For an internal HTTP endpoint, install Flask:

```bash
pip install flask
```

Create `app.py`:

```python
from flask import Flask, request, jsonify
from playwright.sync_api import sync_playwright

app = Flask(__name__)

playwright = sync_playwright().start()
browser = playwright.chromium.launch(
    headless=True,
    args=["--no-sandbox", "--disable-dev-shm-usage"],
)


@app.get("/render")
def render():
    url = request.args.get("url")

    if not url or not url.startswith(("http://", "https://")):
        return jsonify({"error": "A valid HTTP(S) URL is required"}), 400

    context = browser.new_context()
    page = context.new_page()

    try:
        page.goto(url, wait_until="domcontentloaded", timeout=30_000)
        page.wait_for_timeout(2_000)

        return jsonify({
            "url": page.url,
            "title": page.title(),
            "html": page.content(),
            "text": page.locator("body").inner_text(),
        })
    except Exception as error:
        return jsonify({"error": str(error)}), 500
    finally:
        context.close()


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8000)
```

Run it:

```bash
source ~/browser-fetcher/.venv/bin/activate
python app.py
```

Test locally on the VPS:

```bash
curl "http://127.0.0.1:8000/render?url=https://example.com"
```

For production, put Nginx or another reverse proxy in front of it and run the service with `systemd` or Docker.

Important operational precautions:

- Do not expose an unrestricted `/render?url=` endpoint publicly. It can be abused for SSRF attacks.
- Allow only `http` and `https` URLs.
- Block requests to `127.0.0.1`, private IP ranges, cloud metadata addresses, and internal hostnames.
- Add authentication, rate limits, URL length limits, and execution timeouts.
- Create a new browser context per request and close it afterward.
- Reuse one browser process, but do not share cookies or storage between unrelated users.
- Respect the target website’s terms, robots rules, authentication requirements, and rate limits.
- Do not attempt to bypass CAPTCHAs or access controls.

For a small VPS, one Chromium process with a few concurrent pages is usually a reasonable starting point. Limit concurrency because each page can consume substantial memory.
