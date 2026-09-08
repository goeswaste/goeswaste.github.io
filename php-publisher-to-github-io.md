---
title: PHP Publisher to Github.io
date: 2026-09-08 10:54:53
---

Here's a completely redesigned PHP with a modern, clean, and fully responsive interface:

```php
<?php
// ============ CONFIG ============
define('GITHUB_TOKEN', 'your_personal_access_token_here');
define('GITHUB_USERNAME', 'your_username');
define('GITHUB_REPO', 'your_username.github.io');
define('GITHUB_BRANCH', 'main');

error_reporting(E_ALL);
ini_set('display_errors', 0);

$message = '';
$message_type = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $pageName = trim($_POST['page_name'] ?? '');
    $markdownContent = trim($_POST['post_content'] ?? '');
    
    if (empty($pageName) || empty($markdownContent)) {
        $message = 'Page name and content are required.';
        $message_type = 'error';
    } else {
        $filename = preg_replace('/[^a-zA-Z0-9-_]/', '-', strtolower($pageName)) . '.md';
        
        $fullMarkdown = "---
title: " . $pageName . "
date: " . date('Y-m-d H:i:s') . "
---

" . $markdownContent;
        
        $result = pushToGitHub($filename, $fullMarkdown, $pageName);
        
        if ($result['success']) {
            $pageUrl = str_replace('.md', '', $filename);
            $message = 'Post published successfully! View at: https://' . GITHUB_USERNAME . '.github.io/' . $pageUrl;
            $message_type = 'success';
        } else {
            $message = $result['error'];
            $message_type = 'error';
        }
    }
}

function pushToGitHub($filename, $content, $pageName) {
    if (GITHUB_TOKEN === 'your_personal_access_token_here') {
        return array(
            'success' => false,
            'error' => 'GitHub token not configured.',
        );
    }
    
    $url = "https://api.github.com/repos/" . GITHUB_USERNAME . "/" . GITHUB_REPO . "/contents/" . $filename;
    
    $data = array(
        'message' => 'Add post: ' . $pageName,
        'content' => base64_encode($content),
        'branch' => GITHUB_BRANCH
    );
    
    $ch = curl_init();
    curl_setopt_array($ch, array(
        CURLOPT_URL => $url,
        CURLOPT_CUSTOMREQUEST => 'PUT',
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_HTTPHEADER => array(
            'Authorization: token ' . GITHUB_TOKEN,
            'User-Agent: PHP-GitHub-Publisher',
            'Content-Type: application/json',
            'Accept: application/vnd.github.v3+json'
        ),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 10,
        CURLOPT_SSL_VERIFYPEER => true
    ));
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $curlError = curl_error($ch);
    curl_close($ch);
    
    if ($curlError) {
        return array('success' => false, 'error' => 'Connection error: ' . $curlError);
    }
    
    $responseData = json_decode($response, true);
    
    if ($httpCode === 401) {
        return array('success' => false, 'error' => 'Invalid token. Check your GitHub credentials.');
    }
    
    if ($httpCode === 404) {
        return array('success' => false, 'error' => 'Repository not found. Check username and repo name.');
    }
    
    if ($httpCode === 422) {
        return array('success' => false, 'error' => 'Invalid request.');
    }
    
    if ($httpCode >= 200 && $httpCode < 300) {
        return array('success' => true, 'error' => '');
    }
    
    return array('success' => false, 'error' => 'HTTP ' . $httpCode . ' error.');
}
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Publish to GitHub Pages</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        :root {
            --primary: #3b82f6;
            --primary-dark: #1e40af;
            --success: #10b981;
            --error: #ef4444;
            --warning: #f59e0b;
            --gray-50: #f9fafb;
            --gray-100: #f3f4f6;
            --gray-200: #e5e7eb;
            --gray-300: #d1d5db;
            --gray-400: #9ca3af;
            --gray-500: #6b7280;
            --gray-600: #4b5563;
            --gray-700: #374151;
            --gray-900: #111827;
        }

        html {
            scroll-behavior: smooth;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 20px;
            color: var(--gray-700);
        }

        .wrapper {
            width: 100%;
            max-width: 600px;
        }

        .card {
            background: white;
            border-radius: 12px;
            box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
            overflow: hidden;
        }

        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            padding: 32px 24px;
            text-align: center;
            color: white;
        }

        .header h1 {
            font-size: 28px;
            font-weight: 700;
            margin-bottom: 8px;
        }

        .header p {
            font-size: 14px;
            opacity: 0.9;
        }

        .content {
            padding: 32px 24px;
        }

        .alert {
            padding: 16px;
            border-radius: 8px;
            margin-bottom: 24px;
            font-size: 14px;
            display: flex;
            align-items: flex-start;
            gap: 12px;
            animation: slideIn 0.3s ease;
        }

        @keyframes slideIn {
            from {
                opacity: 0;
                transform: translateY(-10px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }

        .alert.success {
            background-color: #ecfdf5;
            color: #065f46;
            border: 1px solid #a7f3d0;
        }

        .alert.error {
            background-color: #fef2f2;
            color: #7f1d1d;
            border: 1px solid #fecaca;
        }

        .alert-icon {
            flex-shrink: 0;
            font-size: 18px;
            margin-top: 1px;
        }

        .form-group {
            margin-bottom: 20px;
        }

        .form-group:last-of-type {
            margin-bottom: 0;
        }

        label {
            display: block;
            font-size: 14px;
            font-weight: 600;
            color: var(--gray-700);
            margin-bottom: 8px;
        }

        input[type="text"],
        textarea {
            width: 100%;
            padding: 12px 14px;
            border: 1.5px solid var(--gray-200);
            border-radius: 8px;
            font-size: 14px;
            font-family: inherit;
            background: white;
            transition: border-color 0.2s, box-shadow 0.2s;
        }

        input[type="text"]:focus,
        textarea:focus {
            outline: none;
            border-color: var(--primary);
            box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }

        input[type="text"]::placeholder,
        textarea::placeholder {
            color: var(--gray-400);
        }

        textarea {
            resize: vertical;
            min-height: 280px;
            font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Courier New', monospace;
            line-height: 1.6;
        }

        .char-count {
            font-size: 12px;
            color: var(--gray-400);
            margin-top: 4px;
            text-align: right;
        }

        button {
            width: 100%;
            padding: 14px;
            background: linear-gradient(135deg, var(--primary) 0%, var(--primary-dark) 100%);
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 15px;
            font-weight: 600;
            cursor: pointer;
            transition: transform 0.2s, box-shadow 0.2s;
            margin-top: 24px;
        }

        button:hover {
            transform: translateY(-2px);
            box-shadow: 0 10px 15px -3px rgba(59, 130, 246, 0.3);
        }

        button:active {
            transform: translateY(0);
        }

        .divider {
            height: 1px;
            background: var(--gray-100);
            margin: 32px 0;
        }

        .info-section {
            display: none;
        }

        .info-section.show {
            display: block;
            animation: fadeIn 0.3s ease;
        }

        @keyframes fadeIn {
            from {
                opacity: 0;
            }
            to {
                opacity: 1;
            }
        }

        .info-title {
            font-size: 13px;
            font-weight: 700;
            color: var(--gray-600);
            text-transform: uppercase;
            letter-spacing: 0.5px;
            margin-bottom: 12px;
        }

        .info-list {
            list-style: none;
            font-size: 13px;
            color: var(--gray-600);
            line-height: 1.8;
        }

        .info-list li {
            padding: 6px 0;
        }

        .info-list li:before {
            content: "→ ";
            color: var(--primary);
            font-weight: 700;
            margin-right: 6px;
        }

        .code-block {
            background: var(--gray-50);
            border: 1px solid var(--gray-200);
            border-radius: 6px;
            padding: 12px;
            font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Courier New', monospace;
            font-size: 12px;
            color: var(--gray-700);
            margin-top: 12px;
            overflow-x: auto;
            white-space: pre-wrap;
            word-break: break-word;
        }

        .toggle-btn {
            background: none;
            border: none;
            color: var(--primary);
            font-size: 13px;
            font-weight: 600;
            cursor: pointer;
            padding: 0;
            margin-top: 12px;
            text-decoration: underline;
            width: auto;
            transition: color 0.2s;
        }

        .toggle-btn:hover {
            color: var(--primary-dark);
            transform: none;
        }

        @media (max-width: 480px) {
            .header {
                padding: 24px 16px;
            }

            .header h1 {
                font-size: 24px;
            }

            .content {
                padding: 24px 16px;
            }

            textarea {
                min-height: 200px;
                font-size: 16px;
            }

            button {
                padding: 12px;
                font-size: 14px;
                margin-top: 20px;
            }
        }
    </style>
</head>
<body>
    <div class="wrapper">
        <div class="card">
            <div class="header">
                <h1>✨ Publish Post</h1>
                <p>Create and publish markdown posts to GitHub Pages</p>
            </div>

            <div class="content">
                <?php if ($message): ?>
                    <div class="alert <?php echo $message_type; ?>">
                        <div class="alert-icon">
                            <?php echo $message_type === 'success' ? '✓' : '!'; ?>
                        </div>
                        <div><?php echo htmlspecialchars($message); ?></div>
                    </div>
                <?php endif; ?>

                <form method="POST">
                    <div class="form-group">
                        <label for="page_name">Post Title</label>
                        <input 
                            type="text" 
                            id="page_name" 
                            name="page_name" 
                            placeholder="e.g., My First Blog Post" 
                            required
                            autofocus
                        >
                    </div>

                    <div class="form-group">
                        <label for="post_content">Content (Markdown)</label>
                        <textarea 
                            id="post_content" 
                            name="post_content" 
                            placeholder="Write your post in markdown...

# Heading
## Subheading

Regular paragraph text.

- Bullet point
- Another point

**Bold** and *italic* text" 
                            required
                        ></textarea>
                        <div class="char-count">
                            <span id="char-count">0</span> characters
                        </div>
                    </div>

                    <button type="submit">Publish Post</button>
                </form>

                <div class="divider"></div>

                <button class="toggle-btn" onclick="toggleInfo()">Need help? Show markdown guide</button>

                <div class="info-section" id="info-section">
                    <div class="info-title">Markdown Syntax</div>
                    <div class="code-block"># Heading
## Subheading
### Small heading

Regular paragraph.

**Bold** and *italic* text

- Bullet point
- Another point

1. Numbered item
2. Second item

[Link](https://example.com)

`inline code`

```
code block
```

> Quote</div>
                </div>
            </div>
        </div>
    </div>

    <script>
        function toggleInfo() {
            const section = document.getElementById('info-section');
            section.classList.toggle('show');
        }

        // Character counter
        document.getElementById('post_content').addEventListener('input', function() {
            document.getElementById('char-count').textContent = this.value.length;
        });
    </script>
</body>
</html>
```

## Design Features:

### **Modern & Clean**
- **Gradient header** with clear branding
- **Minimal color palette** (blue/purple, gray tones)
- **Ample whitespace** for breathability
- **Smooth animations** for messages

### **Responsive Design**
- **Mobile-first approach** – works perfectly on phones
- **Tablet-friendly** – optimized spacing
- **Desktop optimized** – max-width container
- **Touch-friendly** buttons and inputs

### **User Experience**
- **Auto-focus** on title input when page loads
- **Character counter** for content
- **Collapsible markdown guide** (saves space)
- **Alert animations** for success/error messages
- **Clear placeholder text** with examples
- **Visual feedback** on hover/focus states

### **Accessibility**
- Semantic HTML
- Good color contrast
- Proper label associations
- Keyboard navigation support

Try it out! The interface is **much simpler** while being **fully responsive** across all devices. 📱💻