---
title: Node.js Playwright API
date: 2026-11-19 14:11:44
---

### how do I use Playwright on VPS to visit a website and relay respond to a PHP webpage?

You can achieve this by running Playwright with Node.js on your VPS and connecting it to your PHP website through an HTTP API.

This is actually a very good architecture because Playwright handles browser automation, while PHP handles your website’s frontend and backend.

I’ll show you a working example.

1. How the system works

The architecture looks like this:

        USER
         |
         v
    PHP WEBSITE
    (index.php)
         |
         | HTTP POST
         v
    NODE.JS API
    (port 3000)
         |
         v
      PLAYWRIGHT
         |
         v
    CHROMIUM BROWSER
         |
         v
    TARGET WEBSITE
         |
         v
    EXTRACT PAGE DATA
         |
         v
    NODE.JS API
         |
         | JSON RESPONSE
         v
    PHP WEBSITE
         |
         v
    DISPLAY RESULT

For example:
	1.	A visitor enters a website URL into your PHP webpage.
	2.	PHP sends the URL to your Node.js API.
	3.	Node.js launches Playwright.
	4.	Playwright visits the target website.
	5.	Playwright extracts the page title and content.
	6.	Node.js returns the results as JSON.
	7.	PHP displays the results to the visitor.

2. Install Playwright on your VPS

Assuming you’re using Ubuntu 22.04/24.04.

Install Node.js if you haven’t already.

sudo apt update
sudo apt install nodejs npm -y

Create a project:

mkdir playwright-service
cd playwright-service

Initialize Node.js:

npm init -y

Install Playwright and Express:

npm install playwright express

Install Chromium:

npx playwright install --with-deps chromium

Playwright officially supports running Chromium in headless mode on Linux VPS environments.  ￼

3. Create the Playwright API

Create:

server.js

Here’s a basic working implementation.

const express = require('express');
const { chromium } = require('playwright');

const app = express();

app.use(express.json());

app.post('/scrape', async (req, res) => {

    const { url } = req.body;

    if (!url) {
        return res.status(400).json({
            success: false,
            error: 'URL is required'
        });
    }

    let browser;

    try {

        browser = await chromium.launch({
            headless: true
        });

        const page = await browser.newPage();

        await page.goto(url, {
            waitUntil: 'domcontentloaded',
            timeout: 30000
        });

        // Extract website information
        const result = await page.evaluate(() => {

            return {
                title: document.title,
                url: window.location.href,
                content: document.body.innerText
            };

        });

        res.json({
            success: true,
            data: result
        });

    } catch (error) {

        res.status(500).json({
            success: false,
            error: error.message
        });

    } finally {

        if (browser) {
            await browser.close();
        }

    }

});

app.listen(3000, '127.0.0.1', () => {
    console.log('Playwright API running on port 3000');
});

Start the service:

node server.js

Your Playwright API is now running at:

http://127.0.0.1:3000

4. Connect your PHP webpage

Now create:

index.php

<?php

$result = null;
$error = null;

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $url = $_POST['url'] ?? '';

    $ch = curl_init('http://127.0.0.1:3000/scrape');

    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json'
        ],
        CURLOPT_POSTFIELDS => json_encode([
            'url' => $url
        ]),
        CURLOPT_TIMEOUT => 40
    ]);

    $response = curl_exec($ch);

    if ($response === false) {

        $error = curl_error($ch);

    } else {

        $result = json_decode($response, true);

    }

    curl_close($ch);
}

?>

<!DOCTYPE html>
<html>
<head>
    <title>Playwright PHP Example</title>
</head>

<body>

<h2>Website Browser Automation</h2>

<form method="POST">

    <input
        type="url"
        name="url"
        placeholder="https://example.com"
        required
        style="width:350px;padding:10px"
    >

    <button type="submit">
        Visit Website
    </button>

</form>

<?php if ($error): ?>

    <h3>Error:</h3>
    <?= htmlspecialchars($error) ?>

<?php endif; ?>

<?php if ($result): ?>

    <?php if ($result['success']): ?>

        <h3>Website Information</h3>

        <p>
            <strong>Title:</strong>
            <?= htmlspecialchars($result['data']['title']) ?>
        </p>

        <p>
            <strong>URL:</strong>
            <?= htmlspecialchars($result['data']['url']) ?>
        </p>

        <h3>Website Content</h3>

        <pre><?= htmlspecialchars($result['data']['content']) ?></pre>

    <?php else: ?>

        <p>Error: <?= htmlspecialchars($result['error']) ?></p>

    <?php endif; ?>

<?php endif; ?>

</body>
</html>

PHP uses cURL to communicate with Node.js and receive its JSON response.  ￼

5. Keep Playwright running in the background

You probably don’t want to manually execute node server.js every time.

Install PM2:

sudo npm install -g pm2

Start the service:

pm2 start server.js --name playwright-service

Enable automatic startup:

pm2 startup
pm2 save

Now your Playwright service will continue running in the background.

6. Important security considerations

Since PHP is accepting URLs from visitors, you should implement some protections before deploying this publicly.
	•	Restrict which domains Playwright can visit.
	•	Block localhost, private IP addresses and internal network addresses to prevent SSRF attacks.
	•	Validate redirect destinations too.
	•	Add API authentication between PHP and Node.js.
	•	Set browser timeouts and limit concurrent browser sessions.
	•	Run Playwright under a dedicated, unprivileged Linux user.

For production, also consider reusing browser instances instead of launching a new Chromium process for every request.

⸻

7. What else can you do with this architecture?

Once Playwright is connected to PHP, you can automate much more than simply retrieving website text.

Feature	Possible?
Visit any permitted website	Yes
Extract HTML	Yes
Extract website text	Yes
Take screenshots	Yes
Click buttons	Yes
Fill forms	Yes
Login to websites you are authorized to access	Yes
Extract dynamically loaded content	Yes
Monitor website changes	Yes
Return screenshots to PHP	Yes
Return JSON data to PHP	Yes
Run multiple browser sessions	Yes

Useful official documentation:
	•	Playwright Documentation￼
	•	Playwright Browser API￼
	•	Playwright Page API￼

⸻

One question about your intended setup

There are two ways you might want to build this:

A. PHP → Playwright → Website → PHP

PHP sends a URL, Playwright visits it, extracts information and returns the result to PHP.

B. PHP → Playwright → Website → Live browser interaction → PHP

PHP sends commands such as:
	•	Open website.
	•	Click a button.
	•	Enter text.
	•	Wait for a response.
	•	Extract the result.
	•	Send the result back to PHP.

Which one are you trying to build?

If you want option B, I can show you how to build a persistent Playwright browser service that PHP can control through API commands without launching a new browser for every action.

### build a persistent Playwright browser service without launching a new browser for every action, also try minimal the resources Playwright service take on VPS. 

The best approach is to build a persistent Playwright browser service where Chromium launches only once and stays running on your VPS.

PHP communicates with it through an HTTP API. Each PHP request sends an instruction to an existing browser session rather than launching another browser.

This significantly reduces CPU overhead and browser startup time.

1. Recommended architecture

I recommend this architecture:

             PHP WEBSITE
                  |
                  |
             HTTP API
                  |
                  v
       NODE.JS PLAYWRIGHT SERVICE
                  |
                  |
          SINGLE CHROMIUM
           (Always Running)
                  |
          +-------+-------+
          |       |       |
          v       v       v
       Session Session Session
          1       2       3
          |       |       |
          v       v       v
       Website Website Website
          A       B       C

The important distinction:
	•	One Chromium browser process.
	•	Multiple isolated browser contexts.
	•	Each context maintains its own cookies and login sessions.
	•	Each context has its own page.
	•	Browser remains running.
	•	PHP can send multiple commands to an existing session.

For resource efficiency, I recommend starting with:

Component	Configuration
Chromium processes	1
Browser instances	1
Browser contexts	Maximum 3
Pages per context	1
Headless	Yes
Browser lifetime	Persistent
Session timeout	15 minutes idle
Navigation timeout	30 seconds
Concurrent sessions	3

You can increase these limits later.

2. Install dependencies

On your Ubuntu VPS:

mkdir playwright-service
cd playwright-service

npm init -y

npm install express playwright dotenv

Install Chromium:

npx playwright install --with-deps chromium

3. Create the configuration

Create .env:

PORT=3000

API_KEY=change-this-to-a-long-random-secret

MAX_SESSIONS=3

SESSION_TIMEOUT=900000

NAVIGATION_TIMEOUT=30000

# Only permit websites you trust.
ALLOWED_DOMAINS=example.com,google.com

For production, use your actual permitted domains.

This allowlist is important because your PHP API must not become an unrestricted proxy into internal VPS services or private networks.

4. Create the persistent Playwright service

Create server.js.

This is the main component.

It launches Chromium once, maintains browser sessions, and accepts commands from PHP.

require('dotenv').config();

const express = require('express');
const { chromium } = require('playwright');
const crypto = require('crypto');

const app = express();

app.use(express.json({
    limit: '2mb'
}));

// Configuration
const PORT = process.env.PORT || 3000;

const API_KEY = process.env.API_KEY;

const MAX_SESSIONS = Number(
    process.env.MAX_SESSIONS || 3
);

const SESSION_TIMEOUT = Number(
    process.env.SESSION_TIMEOUT || 900000
);

const NAVIGATION_TIMEOUT = Number(
    process.env.NAVIGATION_TIMEOUT || 30000
);

const ALLOWED_DOMAINS = (
    process.env.ALLOWED_DOMAINS || ''
)
.split(',')
.map(x => x.trim().toLowerCase())
.filter(Boolean);

// Global browser
let browser = null;

// Session storage
const sessions = new Map();

// --------------------------------
// Authentication
// --------------------------------

app.use((req, res, next) => {

    if (req.path === '/health') {
        return next();
    }

    const key = req.headers['x-api-key'];

    if (!API_KEY || key !== API_KEY) {
        return res.status(401).json({
            success: false,
            error: 'Unauthorized'
        });
    }

    next();

});

// --------------------------------
// URL validation
// --------------------------------

function validateURL(value) {

    let url;

    try {
        url = new URL(value);
    } catch {
        throw new Error('Invalid URL');
    }

    if (!['http:', 'https:'].includes(url.protocol)) {
        throw new Error('Only HTTP and HTTPS are allowed');
    }

    if (url.username || url.password) {
        throw new Error('Embedded credentials are not allowed');
    }

    const hostname = url.hostname.toLowerCase();

    const allowed = ALLOWED_DOMAINS.some(domain =>
        hostname === domain ||
        hostname.endsWith('.' + domain)
    );

    if (!allowed) {
        throw new Error('Domain is not allowed');
    }

    return url.href;
}

// --------------------------------
// Start browser ONCE
// --------------------------------

async function startBrowser() {

    if (browser) {
        return;
    }

    console.log('Starting Chromium...');

    browser = await chromium.launch({

        headless: true,

        args: [
            '--disable-dev-shm-usage',
            '--disable-extensions',
            '--disable-background-networking',
            '--disable-sync',
            '--no-first-run'
        ]

    });

    browser.on('disconnected', () => {

        console.error('Chromium disconnected');

        browser = null;

        sessions.clear();

    });

    console.log('Chromium started');

}

// --------------------------------
// Create browser session
// --------------------------------

async function createSession() {

    if (!browser) {
        await startBrowser();
    }

    if (sessions.size >= MAX_SESSIONS) {
        throw new Error('Maximum sessions reached');
    }

    const id = crypto.randomUUID();

    const context = await browser.newContext({

        viewport: {
            width: 1280,
            height: 720
        },

        acceptDownloads: false

    });

    const page = await context.newPage();

    page.setDefaultTimeout(15000);

    page.setDefaultNavigationTimeout(
        NAVIGATION_TIMEOUT
    );

    const session = {

        id,
        context,
        page,
        lastActivity: Date.now(),

        // Serialize commands within each session.
        queue: Promise.resolve()

    };

    sessions.set(id, session);

    return session;

}

// --------------------------------
// Get existing session
// --------------------------------

function getSession(id) {

    const session = sessions.get(id);

    if (!session) {
        throw new Error('Session not found or expired');
    }

    session.lastActivity = Date.now();

    return session;

}

// --------------------------------
// Execute session commands in order
// --------------------------------

function execute(session, callback) {

    const task = session.queue.then(callback);

    session.queue = task.catch(() => {});

    return task;

}

// --------------------------------
// Create new session
// --------------------------------

app.post('/sessions', async (req, res) => {

    try {

        const session = await createSession();

        res.json({

            success: true,

            session_id: session.id

        });

    } catch (error) {

        res.status(500).json({

            success: false,

            error: error.message

        });

    }

});

// --------------------------------
// Execute browser commands
// --------------------------------

app.post('/sessions/:id/action', async (req, res) => {

    try {

        const session = getSession(req.params.id);

        const page = session.page;

        const { action, url, selector, value } = req.body;

        const result = await execute(session, async () => {

            switch (action) {

                // Visit website
                case 'open': {

                    const target = validateURL(url);

                    await page.goto(target, {
                        waitUntil: 'domcontentloaded'
                    });

                    // Check final destination too.
                    validateURL(page.url());

                    return {
                        url: page.url(),
                        title: await page.title()
                    };

                }

                // Click element
                case 'click': {

                    await page.locator(selector).click();

                    return {
                        clicked: selector
                    };

                }

                // Fill input
                case 'fill': {

                    await page.locator(selector).fill(value);

                    return {
                        filled: selector
                    };

                }

                // Extract text
                case 'text': {

                    return await page.locator(
                        selector || 'body'
                    ).innerText();

                }

                // Extract HTML
                case 'html': {

                    return await page.content();

                }

                // Screenshot
                case 'screenshot': {

                    const screenshot = await page.screenshot({
                        type: 'jpeg',
                        quality: 60
                    });

                    return screenshot.toString('base64');

                }

                // Current page information
                case 'status': {

                    return {

                        url: page.url(),

                        title: await page.title()

                    };

                }

                default:

                    throw new Error('Unknown action');

            }

        });

        res.json({

            success: true,

            result

        });

    } catch (error) {

        res.status(400).json({

            success: false,

            error: error.message

        });

    }

});

// --------------------------------
// Close session
// --------------------------------

app.delete('/sessions/:id', async (req, res) => {

    try {

        const session = sessions.get(req.params.id);

        if (!session) {
            return res.status(404).json({
                success: false,
                error: 'Session not found'
            });
        }

        sessions.delete(req.params.id);

        await session.context.close();

        res.json({
            success: true
        });

    } catch (error) {

        res.status(500).json({
            success: false,
            error: error.message
        });

    }

});

// --------------------------------
// Automatic idle session cleanup
// --------------------------------

setInterval(async () => {

    const now = Date.now();

    for (const [id, session] of sessions) {

        if (
            now - session.lastActivity >
            SESSION_TIMEOUT
        ) {

            console.log('Removing idle session:', id);

            sessions.delete(id);

            await session.context.close().catch(() => {});

        }

    }

}, 30000);

// --------------------------------
// Health check
// --------------------------------

app.get('/health', (req, res) => {

    res.json({

        status: 'running',

        browser: browser ? 'connected' : 'disconnected',

        sessions: sessions.size,

        max_sessions: MAX_SESSIONS

    });

});

// --------------------------------
// Start service
// --------------------------------

async function main() {

    await startBrowser();

    app.listen(PORT, '127.0.0.1', () => {

        console.log(
            `Playwright service running on ${PORT}`
        );

    });

}

main().catch(console.error);

Important production note: The domain allowlist above validates the initial and final page URL, but it is not a complete SSRF defense. Redirects, subresources, DNS rebinding and browser requests need additional protection. For an Internet-facing service, enforce outbound network restrictions at the VPS/container level and validate navigation and request destinations. Also add limits on request bodies, selector lengths and command execution time.

5. Start the persistent service

Install PM2:

sudo npm install -g pm2

Start:

pm2 start server.js --name playwright

Save:

pm2 save

Check status:

pm2 status

Check logs:

pm2 logs playwright

Your Chromium process now remains alive even when PHP is not making requests.

6. PHP integration

Now comes the interesting part.

PHP can create a browser session and then control that same session repeatedly.

Create browser.php:

<?php

$api = 'http://127.0.0.1:3000';

$apiKey = 'change-this-to-a-long-random-secret';

function browserRequest($endpoint, $method = 'GET', $data = null) {

    global $api, $apiKey;

    $ch = curl_init($api . $endpoint);

    $headers = [
        'X-API-Key: ' . $apiKey,
        'Content-Type: application/json'
    ];

    curl_setopt_array($ch, [

        CURLOPT_RETURNTRANSFER => true,

        CURLOPT_CUSTOMREQUEST => $method,

        CURLOPT_HTTPHEADER => $headers,

        CURLOPT_TIMEOUT => 45

    ]);

    if ($data !== null) {

        curl_setopt(
            $ch,
            CURLOPT_POSTFIELDS,
            json_encode($data)
        );

    }

    $response = curl_exec($ch);

    curl_close($ch);

    return json_decode($response, true);

}

A. Create browser session

<?php

$result = browserRequest(
    '/sessions',
    'POST',
    []
);

$sessionId = $result['session_id'] ?? null;

echo $sessionId;

Store $sessionId in your PHP session or database, associated with the appropriate user.

B. Open a website

$result = browserRequest(

    "/sessions/$sessionId/action",

    'POST',

    [
        'action' => 'open',
        'url' => 'https://example.com'
    ]

);

print_r($result);

C. Click a button

$result = browserRequest(

    "/sessions/$sessionId/action",

    'POST',

    [
        'action' => 'click',
        'selector' => 'button.submit'
    ]

);

print_r($result);

D. Fill an input

$result = browserRequest(

    "/sessions/$sessionId/action",

    'POST',

    [
        'action' => 'fill',
        'selector' => 'input[name="email"]',
        'value' => 'test@example.com'
    ]

);

E. Extract website text

$result = browserRequest(

    "/sessions/$sessionId/action",

    'POST',

    [
        'action' => 'text',
        'selector' => 'body'
    ]

);

echo $result['result'] ?? '';

F. Take a screenshot

$result = browserRequest(

    "/sessions/$sessionId/action",

    'POST',

    [
        'action' => 'screenshot'
    ]

);

$image = base64_decode($result['result']);

file_put_contents('screenshot.jpg', $image);

G. Close browser session

browserRequest(
    "/sessions/$sessionId",
    'DELETE'
);

Notice something important:

None of these commands launches another Chromium browser.

They all use the same browser process and their existing session.

⸻

7. How to minimize VPS resource consumption

This is probably the most important part of your project.

Chromium can consume considerable RAM, particularly when multiple websites are open.

Here are my recommendations.

Optimization	Benefit
One persistent Chromium	Avoid repeated startup overhead
Headless mode	No graphical desktop required
One page per session	Limits memory usage
Limit sessions to 3	Prevents uncontrolled RAM consumption
Close idle contexts	Reclaims memory
Block unnecessary images	Reduces bandwidth and memory
Block unnecessary fonts	Reduces resource usage
Disable video loading	Reduces bandwidth
Reuse browser contexts	Preserves login sessions
Limit concurrent actions	Prevents CPU spikes

Optional: Block unnecessary resources

Inside createSession(), after creating the page, you can add:

await page.route('**/*', async route => {

    const type = route.request().resourceType();

    if ([
        'image',
        'media',
        'font'
    ].includes(type)) {

        return route.abort();

    }

    return route.continue();

});

This can substantially reduce resource consumption on text-oriented websites.

However, some websites depend on images, fonts or media for their layout and functionality. Enable this only when appropriate.

Recommended VPS specifications

These are starting-point estimates, not guaranteed minimums.

VPS RAM	Suggested configuration
1 GB	1 session, very light websites
2 GB	1–2 sessions
4 GB	3–5 sessions
8 GB	5–10 sessions

Actual usage depends heavily on the websites being visited.

I would start with a 2 GB VPS and one or two concurrent sessions if resource consumption is your primary concern.

Monitor memory:

free -h

Monitor Chromium processes:

ps aux | grep chromium

Monitor Node.js:

pm2 monit


⸻

8. An additional improvement I’d recommend

There is one architectural improvement worth considering.

Instead of having PHP send individual commands directly and wait for every browser operation, you could introduce a browser task queue.

For example:

PHP
 |
 | Create task
 v
Node.js API
 |
 v
Task Queue
 |
 v
Persistent Playwright
 |
 +---- Open website
 |
 +---- Click button
 |
 +---- Fill form
 |
 +---- Extract response
 |
 v
Task Result
 |
 v
PHP retrieves result

This offers several advantages:
	•	PHP doesn’t need to maintain long-running HTTP connections.
	•	Browser actions can execute sequentially.
	•	Multiple PHP requests can be queued.
	•	You can implement task status tracking.
	•	You can retry failed operations.
	•	You can prevent multiple users from accidentally controlling the same browser session simultaneously.

For a production system, I’d prefer this approach over synchronous requests.

⸻

9. My recommended final configuration

For your VPS, I’d build toward this:

Component	Recommendation
Backend	Node.js
Browser automation	Playwright
Browser	Chromium
Browser processes	1
Browser contexts	2–3
PHP communication	HTTP JSON API
Session storage	Redis or database
Task management	Lightweight in-memory queue initially
Process management	PM2
Browser cleanup	Automatic idle timeout
Browser restart	Automatic service recovery
Resource optimization	Block unnecessary resources

One particularly important distinction: Keep the Chromium process persistent, but don’t keep every browser context alive forever. Reuse contexts while they’re active and close them after inactivity. This gives you the startup-speed advantage without allowing memory consumption to grow indefinitely.

One thing I’d like to establish before extending this

What are your VPS specifications?

For example:
	•	1 CPU / 2 GB RAM
	•	2 CPU / 4 GB RAM
	•	4 CPU / 8 GB RAM

That will help determine whether we should use one shared Chromium with multiple contexts or a more aggressively resource-limited single-session architecture.

### Can I use a different port? Also, can I skip PM2 part by manually start this Playwright service everytime before the PHP is visited?

Yes to both! You don’t need PM2, and you can use practically any available port.

1. Change the port

Open your .env file:

PORT=8088

You can use other ports too, such as 3001, 5000, 8080 or 9000, provided the port isn’t already occupied.

Your existing server.js already supports this:

const PORT = process.env.PORT || 3000;

Restart the service after changing the port:

node server.js

Your API will now listen on port 8088.

Important: Keep the service bound to 127.0.0.1 if PHP and Node.js are running on the same VPS. You don’t need to expose the API port publicly.

2. Skip PM2 and manually start Playwright

Absolutely.

Instead of:

pm2 start server.js --name playwright

Simply run:

node server.js

You should see:

Starting Chromium...
Chromium started
Playwright service running on 8088

Keep that terminal or SSH session open.

Your service will continue running until you stop it.

To stop it, press:

CTRL + C

3. Run it in the background (optional)

If you don’t want to keep your SSH terminal open, you can use:

nohup node server.js > playwright.log 2>&1 &

Check whether it’s running:

ps aux | grep server.js

View logs:

tail -f playwright.log

Stop it:

pkill -f "node server.js"

Be careful with pkill if you run multiple Node.js services; using the process ID is safer.

4. PHP configuration

Just update your PHP API address:

$api = 'http://127.0.0.1:8088';

Everything else remains unchanged.

5. Test your service

Open another SSH terminal and run:

curl http://127.0.0.1:8088/health

Expected response:

{
    "status": "running",
    "browser": "connected",
    "sessions": 0,
    "max_sessions": 3
}


⸻

My recommendation

For your intended setup, I’d actually keep things simple:
	1.	Manually start Node.js and Chromium when you need them.
	2.	Keep the service running while PHP uses it.
	3.	Stop the service when you’re finished.
	4.	Let Chromium launch only once per service startup.

You don’t need PM2, Redis, Docker, or any additional infrastructure to make this work. A simple Node.js process is sufficient.

### Expose this service since Playwight on VPS are different server as PHP is hosted.

Yes. Since Playwright and PHP are hosted on different servers, you’ll need to expose the Playwright API so your PHP server can communicate with it over the Internet.

I recommend exposing it through a public IP address with API-key authentication. For better security, use HTTPS.

1. Change the Playwright server binding

In server.js, find:

app.listen(PORT, '127.0.0.1', () => {
    console.log(`Playwright service running on ${PORT}`);
});

Change it to:

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Playwright service running on port ${PORT}`);
});

This allows connections from external servers.

2. Configure your port

In .env:

PORT=8088

API_KEY=YOUR_LONG_RANDOM_SECRET_KEY

MAX_SESSIONS=3

SESSION_TIMEOUT=900000

NAVIGATION_TIMEOUT=30000

ALLOWED_DOMAINS=example.com,google.com

Generate a secure API key:

openssl rand -hex 32

Use the generated value as your API_KEY.

3. Open the VPS firewall

If you’re using Ubuntu with UFW:

sudo ufw allow 8088/tcp

Check:

sudo ufw status

Also check your VPS provider’s firewall/security group and allow TCP port 8088.

Security recommendation: Ideally, restrict incoming connections to your PHP server’s public IP instead of allowing everyone.

For example:

sudo ufw allow from PHP_SERVER_IP to any port 8088 proto tcp

Replace PHP_SERVER_IP with your actual PHP server IP.

4. Start Playwright

node server.js

Your service is now accessible through:

http://YOUR_PLAYWRIGHT_VPS_IP:8088

Test it from your PHP server:

curl http://YOUR_PLAYWRIGHT_VPS_IP:8088/health

Expected:

{
    "status": "running",
    "browser": "connected",
    "sessions": 0,
    "max_sessions": 3
}

5. Update your PHP configuration

On your PHP hosting server, change:

$api = 'http://127.0.0.1:3000';

To:

$api = 'http://YOUR_PLAYWRIGHT_VPS_IP:8088';

Keep your API key identical:

$apiKey = 'YOUR_LONG_RANDOM_SECRET_KEY';

Your existing PHP functions don’t need any other changes.

6. Important: Use HTTPS for production

Since your API key and browser commands travel between two different servers, I strongly recommend HTTPS.

You have two options:

Method	Recommendation
Nginx reverse proxy + SSL certificate	Best for production
Direct public IP + HTTP	Only for temporary testing

You can use Nginx as a reverse proxy and obtain a free SSL certificate using Let’s Encrypt.

Your final architecture would look like this:

       PHP SERVER
           |
           |
       HTTPS API
           |
           v
   PLAYWRIGHT VPS
           |
       NGINX :443
           |
           v
     NODE.JS :8088
           |
           v
       CHROMIUM
           |
           v
     TARGET WEBSITE

Keep Node.js bound to 127.0.0.1 behind Nginx in this final configuration, and expose only HTTPS publicly.

⸻

One additional security recommendation

Because this service can browse websites on behalf of your PHP application, make sure you:
	•	Authenticate every API endpoint except health checks.
	•	Restrict allowed domains.
	•	Block private and internal IP destinations, including redirects and subresource requests.
	•	Restrict access to your PHP server’s IP.
	•	Never expose Chromium’s debugging port publicly.

My recommendation: Use port 8088 internally, Nginx on port 443 externally, and HTTPS for communication between your PHP and Playwright servers. This gives you a clean and reasonably secure setup.

### For
npm install express playwright dotenv

Do I have to install express and dotenv? What are they by the way?

No, you don’t strictly need both. They serve different purposes.

Here’s what each package does:

1. Express

Express is a lightweight web framework for Node.js.

In our Playwright service, it acts as an HTTP API server that allows your PHP server to communicate with Playwright.

For example:

app.post('/sessions', async (req, res) => {
    // Create Playwright browser session
});

Without Express, you’d need to build the HTTP server yourself using Node.js’s built-in http module.

My recommendation: Keep Express. It makes the API much easier to develop and maintain.

2. dotenv

dotenv loads configuration variables from a .env file.

For example, your .env:

PORT=8088
API_KEY=my-secret-key
MAX_SESSIONS=3

Then in server.js:

require('dotenv').config();

const PORT = process.env.PORT || 3000;
const API_KEY = process.env.API_KEY;

Without dotenv, you can simply hardcode the settings:

const PORT = 8088;
const API_KEY = 'my-secret-key';
const MAX_SESSIONS = 3;

Or use Linux environment variables.

My recommendation: Keep dotenv. It’s convenient and helps separate configuration from your application code. Remember to protect .env from unauthorized access.

3. Playwright

This is the actual browser automation framework.

It controls Chromium and allows you to:
	•	Visit websites.
	•	Click buttons.
	•	Fill forms.
	•	Extract website content.
	•	Take screenshots.
	•	Maintain browser sessions.
	•	Interact with JavaScript-heavy websites.

This is the only essential browser automation package in our setup.

⸻

My recommendation for your VPS

I’d keep all three:

npm install express playwright dotenv

They’re lightweight compared with Chromium, and each has a useful purpose.

Package	Required?	Purpose
Playwright	Yes	Browser automation
Express	No, but recommended	HTTP API server
dotenv	No	Load .env configuration

Bottom line: If you want a simple, maintainable PHP-to-Playwright API, installing all three is the easiest approach.