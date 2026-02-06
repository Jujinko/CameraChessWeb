# Deployment to Plesk (Admin Edition)

This guide describes how to deploy the Camera Chess web application to an existing sub-domain using Plesk Admin Edition.

## Prerequisites

*   **Node.js & npm**: Ensure you have Node.js installed on your local machine to build the project.
*   **Plesk Access**: You need access to the Plesk Admin Panel with permissions to manage the sub-domain's files.

## 1. Local Build

First, you need to generate the production build of the application on your local machine.

1.  Open your terminal and navigate to the project directory.
2.  Install dependencies (if you haven't already):
    ```bash
    npm install
    ```
3.  Run the build script:
    ```bash
    npm run build
    ```

    This command will run TypeScript checks, linting, and then build the project using Vite. The output files will be generated in the `dist` directory.

## 2. Deploy to Plesk

1.  **Log in to Plesk**.
2.  Navigate to **Websites & Domains**.
3.  Locate the sub-domain you want to deploy to (e.g., `chess.example.com`).
4.  Click on **File Manager** (or "Dashboard" then "File Manager").
5.  Navigate to the **Document Root** of your sub-domain.
    *   This is often a folder named after your sub-domain (e.g., `chess.example.com` or `httpdocs/subdomain`). Ensure you are in the correct directory that serves the public files.
6.  **Clear existing files**: If there are old files in this directory, select them and delete them. *Note: Be careful not to delete system protected files if any, though usually, the document root only contains web files.*
7.  **Upload Build Artifacts**:
    *   Click the **Upload** button (usually a "+" icon or "Upload File").
    *   Select all the **contents** inside your local `dist` folder (index.html, assets folder, etc.).
    *   Alternatively, you can zip the contents of the `dist` folder locally, upload the zip file, and then extract it in the File Manager.

## 3. Configure SPA Routing (.htaccess)

Since this is a Single Page Application (SPA) using React Router, you need to configure the web server to redirect all requests to `index.html`. This ensures that when a user refreshes a page like `/play`, the server doesn't look for a directory named `play` but serves the app instead.

### For Apache (Standard on Plesk)

1.  In the Plesk **File Manager**, inside your document root, look for a file named `.htaccess`.
2.  If it doesn't exist, create a new file and name it `.htaccess`.
3.  Edit the file and add the following configuration:

    ```apache
    <IfModule mod_rewrite.c>
      RewriteEngine On
      RewriteBase /
      RewriteRule ^index\.html$ - [L]
      RewriteCond %{REQUEST_FILENAME} !-f
      RewriteCond %{REQUEST_FILENAME} !-d
      RewriteRule . /index.html [L]
    </IfModule>
    ```
4.  Save the file.

### For Nginx (If used as standalone without Apache)

If your Plesk setup uses Nginx exclusively (without Apache proxying), you need to update the "Apache & nginx Settings":

1.  Go to **Websites & Domains** -> **[Your Sub-domain]**.
2.  Click on **Apache & nginx Settings**.
3.  In the **Additional nginx directives** field, add:

    ```nginx
    location / {
        try_files $uri $uri/ /index.html;
    }
    ```
4.  Click **OK** or **Apply**.

## 4. Verification

1.  Open your browser and navigate to your sub-domain URL.
2.  The application should load.
3.  Try navigating to a sub-route (if any) or refresh the page to ensure the SPA routing configuration works correctly.
