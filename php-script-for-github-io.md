---
title: PHP Script for Github.io
date: 2026-09-08 10:52:12
---

I can help you create a PHP webpage for this purpose. However, I should clarify a few important things first:

**Important limitations:**

1. **PHP doesn't run on github.io** — GitHub Pages only serves static HTML, CSS, and JavaScript. You can't run PHP on github.io directly.

2. **You'll need a different hosting solution** — To use PHP, you'd need to host it on a server that supports PHP (like a shared hosting provider, VPS, or your own server).

3. **GitHub Pages workflow** — To post content to github.io, you'd typically:
   - Have your PHP script on a separate server
   - Use the GitHub API to commit files to your repository
   - GitHub Pages then automatically publishes the static files

Here's a solution that combines both approaches:

## PHP Script (hosted on your PHP server)

```php
<?php
// config.php - Set your GitHub credentials
define('GITHUB_TOKEN', 'your_personal_access_token_here');
define('GITHUB_USERNAME', 'your_username');
define('GITHUB_REPO', 'your_username.github.io');
define('GITHUB_BRANCH', 'main');

// Handle form submission
$message = '';
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $pageName = $_POST['page_name'] ?? '';
    $postContent = $_POST['post_content'] ?? '';
    
    // Validate inputs
    if (empty($pageName) || empty($postContent)) {
        $message = '<div class="error">Page name and content are required.</div>';
    } else {
        // Create filename (sanitize page name)
        $filename = preg_replace('/[^a-zA-Z0-9-_]/', '-', strtolower($pageName)) . '.html';
        
        // Create HTML content
        $htmlContent = "<!DOCTYPE html>
<html>
<head>
    <meta charset='UTF-8'>
    <title>" . htmlspecialchars($pageName) . "</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; line-height: 1.6; }
        .container { max-width: 800px; margin: 0 auto; }
    </style>
</head>
<body>
    <div class='container'>
        <h1>" . htmlspecialchars($pageName) . "</h1>
        <div class='content'>
            " . nl2br(htmlspecialchars($postContent)) . "
        </div>
    </div>
</body>
</html>";
        
        // Push to GitHub
        if (pushToGitHub($filename, $htmlContent, $pageName)) {
            $message = '<div class="success">Page published successfully to github.io!</div>';
        } else {
            $message = '<div class="error">Failed to publish page. Check your credentials.</div>';
        }
    }
}

// Function to push content to GitHub
function pushToGitHub($filename, $content, $pageName) {
    $url = "https://api.github.com/repos/" . GITHUB_USERNAME . "/" . GITHUB_REPO . "/contents/" . $filename;
    
    $data = array(
        'message' => 'Add post: ' . $pageName,
        'content' => base64_encode($content),
        'branch' => GITHUB_BRANCH
    );
    
    $options = array(
        'http' => array(
            'header'  => array(
                "Authorization: token " . GITHUB_TOKEN,
                "User-Agent: PHP-GitHub-Publisher",
                "Content-Type: application/json"
            ),
            'method'  => 'PUT',
            'content' => json_encode($data),
            'timeout' => 10
        )
    );
    
    $context = stream_context_create($options);
    $response = @file_get_contents($url, false, $context);
    
    return $response !== false;
}
?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Publish to GitHub Pages</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 20px;
        }
        .container {
            background: white;
            padding: 40px;
            border-radius: 8px;
            box-shadow: 0 10px 25px rgba(0, 0, 0, 0.2);
            width: 100%;
            max-width: 600px;
        }
        h1 {
            color: #333;
            margin-bottom: 30px;
            text-align: center;
        }
        .form-group {
            margin-bottom: 20px;
        }
        label {
            display: block;
            margin-bottom: 8px;
            color: #555;
            font-weight: 500;
        }
        input[type="text"],
        textarea {
            width: 100%;
            padding: 12px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-family: inherit;
            font-size: 14px;
            transition: border-color 0.3s;
        }
        input[type="text"]:focus,
        textarea:focus {
            outline: none;
            border-color: #667eea;
            box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
        }
        textarea {
            resize: vertical;
            min-height: 200px;
        }
        button {
            width: 100%;
            padding: 12px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 4px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: transform 0.2s, box-shadow 0.2s;
        }
        button:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
        }
        button:active {
            transform: translateY(0);
        }
        .message {
            margin-bottom: 20px;
            padding: 15px;
            border-radius: 4px;
            text-align: center;
        }
        .success {
            background-color: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
        }
        .error {
            background-color: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📝 Publish to GitHub Pages</h1>
        
        <?php if ($message) echo $message; ?>
        
        <form method="POST">
            <div class="form-group">
                <label for="page_name">Page Name</label>
                <input type="text" id="page_name" name="page_name" placeholder="e.g., My First Post" required>
            </div>
            
            <div class="form-group">
                <label for="post_content">Content</label>
                <textarea id="post_content" name="post_content" placeholder="Enter your post content here..." required></textarea>
            </div>
            
            <button type="submit">Publish to GitHub</button>
        </form>
    </div>
</body>
</html>
```

## Setup Instructions

1. **Create a Personal Access Token:**
   - Go to GitHub → Settings → Developer settings → Personal access tokens
   - Create a token with `repo` scope
   - Copy the token and paste it in the PHP file's `GITHUB_TOKEN`

2. **Update the config variables** at the top of the PHP script with your GitHub username and repository name

3. **Host the PHP file** on a server that supports PHP

4. **Access it** through your browser and start publishing!

This approach automatically generates HTML files and commits them to your GitHub Pages repository, which then publishes them automatically.