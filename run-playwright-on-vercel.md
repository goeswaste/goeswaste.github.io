---
title: Run Playwright on Vercel
date: 2026-11-14 11:19:17
---

### is Vercel a Node.js platform?

Yes, Vercel supports Node.js, but it is not exclusively a Node.js platform.

Vercel is a cloud platform for deploying and hosting web applications, particularly frontend applications and full-stack JavaScript frameworks.

How Vercel relates to Node.js
	•	Node.js runtime: Vercel supports Node.js for running server-side code.
	•	Serverless Functions: You can write backend APIs using Node.js and deploy them as serverless functions.
	•	Next.js: Vercel is the company behind Next.js, a React framework that uses Node.js during development and can run server-side code in production.
	•	Other runtimes: Vercel also supports Edge Runtime and other languages and frameworks.

Example architecture

If you’re building a full-stack application:

Component	Technology	Hosting
Frontend	React / Next.js	Vercel
Backend API	Node.js / Express	Vercel Functions or another server
Database	PostgreSQL / MongoDB	External database provider
Authentication	Auth.js / Clerk	Vercel or external provider

Is Vercel similar to a Node.js hosting platform?

Somewhat, but there is a distinction:
	•	Vercel: Optimized for frontend and full-stack application deployment.
	•	Render: Supports hosting persistent Node.js backend services.
	•	Railway: Supports Node.js applications, databases, and backend infrastructure.
	•	DigitalOcean: Provides more traditional server and infrastructure hosting.

In short: Vercel supports Node.js, but it’s better described as a frontend and full-stack cloud deployment platform rather than a dedicated Node.js hosting platform.

### Can I run a Playwright script on Vercel?

Yes! You can run Playwright on Vercel, but the best approach depends on what you’re trying to accomplish.

There are three main approaches.

1. Run Playwright inside a Vercel Function

You can create a Next.js API route that executes Playwright.

For example:

// app/api/scrape/route.ts

import { chromium } from "playwright";

export const runtime = "nodejs";

export async function GET() {
  const browser = await chromium.launch({
    headless: true,
  });

  try {
    const page = await browser.newPage();

    await page.goto("https://example.com");

    const title = await page.title();

    return Response.json({ title });
  } finally {
    await browser.close();
  }
}

Considerations:
	•	Browser binaries increase deployment bundle size.
	•	Native Linux dependencies must be available.
	•	Vercel Functions have execution time limits.
	•	Browser memory consumption can be significant.

You may need a compatible Chromium build, such as @sparticuz/chromium.

Vercel also documents larger function bundles for certain configurations.

2. Use a remote browser (Recommended)

Instead of running Chromium inside Vercel, run the browser externally and connect to it using Playwright.

Some options:

Provider	Purpose
Browserbase	Hosted browsers for automation
Browserless	Remote Chrome and Playwright
Kernel	Cloud browser infrastructure

Example:

import { chromium } from "playwright";

const browser = await chromium.connectOverCDP(
  process.env.BROWSER_WS_ENDPOINT!
);

const page = await browser.newPage();

await page.goto("https://example.com");

console.log(await page.title());

await browser.close();

This approach keeps your Vercel application lightweight.

Vercel also provides a [Kernel browser automation starter](Vercel Kernel Starter￼).

3. Run Playwright through GitHub Actions

If your objective is automated testing after deployment:
	1.	Deploy your application to Vercel.
	2.	Trigger GitHub Actions after deployment.
	3.	Install Playwright and Chromium in GitHub Actions.
	4.	Execute your tests against the Vercel deployment URL.

Vercel has an official guide for this approach: Vercel Playwright E2E Testing Guide￼.

⸻

My recommendation

It depends on your use case:

Your requirement	Recommended approach
Web scraping	Vercel + remote browser
Browser automation	Vercel + Browserbase/Browserless
E2E testing	GitHub Actions + Playwright
Screenshot generation	Vercel + remote browser
Long-running browser tasks	Dedicated browser infrastructure

One important question

What are you planning to use Playwright for?
	•	A. Web scraping (extracting data from websites)
	•	B. Browser automation (logging in, clicking buttons, filling forms)
	•	C. Automated testing of your Next.js application
	•	D. Running an AI agent that controls a browser

The architecture I would recommend is quite different depending on your answer.

### A

For web scraping using Playwright, you can use Vercel, but I’d recommend a slightly different architecture depending on the scale of your scraper.

Recommended architecture

           User
             |
             v
       Next.js App
          (Vercel)
             |
             v
       API Endpoint
       /api/scrape
             |
             v
      Playwright Script
             |
             v
       Remote Browser
     (Browserbase /
      Browserless)
             |
             v
       Target Website
             |
             v
       Extracted Data
             |
             v
        PostgreSQL

Why this architecture?
	1.	Vercel handles your frontend and API endpoints.
	2.	Playwright controls a remote Chromium browser.
	3.	The remote browser visits websites and extracts information.
	4.	Extracted data gets saved to PostgreSQL.
	5.	Your frontend displays the results.

Can you run Playwright directly on Vercel?

Yes. There are some recent improvements worth knowing:
	•	Vercel Functions support Node.js.
	•	Large Functions can support browser automation dependencies.
	•	With Fluid Compute, certain Pro and Enterprise functions can run for up to 30 minutes (extended duration is currently beta).  ￼

However, Vercel’s fair-use guidelines explicitly list scrapers as an example of workloads that aren’t permitted under its standard usage guidelines. Check the applicable terms before deploying a scraping workload there.  ￼

My suggested technology stack

Component	Technology
Frontend	Next.js
Application hosting	Vercel
Browser automation	Playwright
Browser infrastructure	Browserbase
Database	PostgreSQL
Background jobs	Inngest
Data validation	Zod

One important question

How will your scraper operate?

1. On-demand scraping: A user enters a URL, clicks Scrape, and gets the extracted data.

2. Scheduled scraping: Automatically scrape websites every 5 minutes, hourly, or daily.

3. Bulk scraping: Scrape hundreds or thousands of URLs in one execution.

4. Continuous scraping: Keep browser sessions running and continuously monitor websites for changes.

This distinction matters because I’d design the infrastructure differently for each scenario.