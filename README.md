# WordPress.org Theme Deploy

> Deploy your theme to the WordPress.org repository using GitHub Actions.

This Action commits the contents of your Git tag to the WordPress.org theme repository using the same tag name. It can exclude files as defined in either `.distignore` or `.gitattributes`, and optionally generate a ZIP file for your releases.

## Quick Start

Add this workflow to `.github/workflows/deploy.yml` in your theme repository:

```yaml
name: Deploy to WordPress.org
on:
  release:
    types: [published]
jobs:
  deploy:
    name: Deploy on release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
    - uses: actions/checkout@v4
    - name: WordPress Theme Deploy
      id: deploy
      uses: kishanjasani/action-wordpress-theme-deploy@main
      with:
        generate-zip: true
      env:
        SVN_PASSWORD: ${{ secrets.WPORG_SVN_PASSWORD }}
        SVN_USERNAME: ${{ secrets.WPORG_SVN_USERNAME }}
    - name: Upload ZIP to release
      uses: softprops/action-gh-release@v2
      with:
        files: ${{ steps.deploy.outputs.zip-path }}
```

Then add your WordPress.org credentials as repository secrets (`WPORG_SVN_USERNAME` and `WPORG_SVN_PASSWORD`) and create a release on GitHub!

## Configuration

### Required secrets

* `SVN_USERNAME` - Your WordPress.org username
* `SVN_PASSWORD` - Your WordPress.org password

[Secrets are set in your repository settings](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets). They cannot be viewed once stored.

### Optional environment variables

* `SLUG` - Defaults to the repository name. This is customizable in case your WordPress repository has a different slug or is capitalized differently.
* `VERSION` - Defaults to the tag name. We do not recommend setting this except for testing purposes.
* `ASSETS_DIR` - Defaults to `.wordpress-org`. This is customizable for other locations of WordPress.org theme repository-specific assets.
* `BUILD_DIR` - Defaults to `false`. Set this flag to the directory where you build your theme files into, then the action will copy and deploy files from that directory. Both absolute and relative paths are supported. The relative path if provided will be concatenated with the repository root directory. All files and folders in the build directory will be deployed, `.distignore` or `.gitattributes` will be ignored.

### Inputs

* `generate-zip` - Defaults to `false`. Generate a ZIP file from the SVN `trunk` directory. Outputs a `zip-path` variable for use in further workflow steps.
* `dry-run` - Defaults to `false`. Set this to `true` if you want to skip the final Subversion commit step (e.g., to debug prior to a non-dry-run commit). `dry-run` - `true` Doesn't require SVN secret.

### Outputs

* `zip-path` - The path to the ZIP file generated if `generate-zip` is set to `true`. Fully qualified including the filename, intended for use in further workflow steps such as uploading release assets.

## Excluding files from deployment

If there are files or directories to be excluded from deployment, such as tests or editor config files, they can be specified in either a `.distignore` file or a `.gitattributes` file using the `export-ignore` directive. If a `.distignore` file is present, it will be used; if not, the Action will look for a `.gitattributes` file and barring that, will write a basic temporary `.gitattributes` into place before proceeding so that no Git/GitHub-specific files are included.

`.distignore` is useful particularly when there are built files that are in `.gitignore`, and is a file that is used in [WP-CLI](https://wp-cli.org/). For modern theme setups with a build step and no built files committed to the repository, this is the way forward.

### Sample baseline files

#### `.distignore`

**Notes:** `.distignore` is for files to be ignored **only**; it does not currently allow negation like `.gitignore`.

```
/.wordpress-org
/.git
/.github
/node_modules
/tests
/bin

.distignore
.gitignore
.gitattributes
package.json
package-lock.json
composer.json
composer.lock
phpunit.xml
webpack.config.js
```

#### `.gitattributes`

```gitattributes
# Directories
/.wordpress-org export-ignore
/.github export-ignore
/node_modules export-ignore
/tests export-ignore

# Files
/.gitattributes export-ignore
/.gitignore export-ignore
/package.json export-ignore
/package-lock.json export-ignore
/composer.json export-ignore
/composer.lock export-ignore
/phpunit.xml export-ignore
```

## Example Workflow Files

To get started, you will want to copy the contents of one of the examples below into `.github/workflows/deploy.yml` and push that to your repository. You are welcome to name the file something else, but it must be in that directory. The usage of `ubuntu-latest` is recommended for compatibility with required dependencies in this Action.

### Deploy on creating a GitHub release

This is the **recommended approach** for most themes. It deploys your theme to WordPress.org when you create a release on GitHub and automatically attaches the ZIP file to the release.

```yaml
name: Deploy to WordPress.org
on:
  release:
    types: [published]
jobs:
  deploy:
    name: Deploy on release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
    - uses: actions/checkout@v4
    - name: Build # Remove or modify this step as needed
      run: |
        npm install
        npm run build
    - name: WordPress Theme Deploy
      id: deploy
      uses: kishanjasani/action-wordpress-theme-deploy@main
      with:
        generate-zip: true
      env:
        SVN_PASSWORD: ${{ secrets.WPORG_SVN_PASSWORD }}
        SVN_USERNAME: ${{ secrets.WPORG_SVN_USERNAME }}
    - name: Upload ZIP to release
      uses: softprops/action-gh-release@v2
      with:
        files: ${{ steps.deploy.outputs.zip-path }}
```

**How to use:**
1. Create a new tag: `git tag 1.0.0 && git push origin 1.0.0`
2. Go to your GitHub repository and create a new release from that tag
3. The action will automatically deploy to WordPress.org and attach the ZIP file to your release

### Deploy on pushing a new tag

This approach deploys immediately when you push a tag to your repository.

```yaml
name: Deploy to WordPress.org
on:
  push:
    tags:
    - "*"
jobs:
  tag:
    name: New tag
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build # Remove or modify this step as needed
      run: |
        npm install
        npm run build
    - name: WordPress Theme Deploy
      uses: kishanjasani/action-wordpress-theme-deploy@main
      env:
        SVN_PASSWORD: ${{ secrets.WPORG_SVN_PASSWORD }}
        SVN_USERNAME: ${{ secrets.WPORG_SVN_USERNAME }}
        SLUG: my-theme-slug # optional, remove if GitHub repo name matches SVN slug, including capitalization
```

**How to use:**
```bash
git tag -a 1.0.0 -m "Version 1.0.0"
git push origin 1.0.0
```

### Deploy with ZIP generation and artifact upload

This example deploys to WordPress.org and saves the ZIP file as a GitHub artifact.

```yaml
name: Deploy to WordPress.org
on:
  push:
    tags:
    - "*"
jobs:
  tag:
    name: New tag
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: |
        npm install
        npm run build
    - name: WordPress Theme Deploy
      id: deploy
      uses: kishanjasani/action-wordpress-theme-deploy@main
      with:
        generate-zip: true
      env:
        SVN_PASSWORD: ${{ secrets.WPORG_SVN_PASSWORD }}
        SVN_USERNAME: ${{ secrets.WPORG_SVN_USERNAME }}
    - name: Upload release asset
      uses: actions/upload-artifact@v4
      with:
        name: theme-zip
        path: ${{ steps.deploy.outputs.zip-path }}
```

### Dry run for testing

Use this to test your workflow without actually committing to WordPress.org SVN.

```yaml
name: Test Deploy
on:
  push:
    branches:
    - develop
jobs:
  test:
    name: Test deployment
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: |
        npm install
        npm run build
    - name: WordPress Theme Deploy (Dry Run)
      uses: kishanjasani/action-wordpress-theme-deploy@main
      with:
        dry-run: true
        generate-zip: true
```

**Note:** Dry run doesn't require SVN credentials.

## Setup Instructions

### Step 1: Add WordPress.org SVN credentials

Add your WordPress.org credentials as secrets in your repository:

1. Go to your GitHub repository
2. Click **Settings** > **Secrets and variables** > **Actions**
3. Click **New repository secret**
4. Add two secrets:
   - Name: `WPORG_SVN_USERNAME`, Value: your WordPress.org username
   - Name: `WPORG_SVN_PASSWORD`, Value: your WordPress.org password

### Step 2: Add the workflow file

Create a file at `.github/workflows/deploy.yml` in your theme repository with one of the example workflows above.

### Step 3: Update your theme version

Before creating a release, make sure your theme's version in `style.css` matches the version you're releasing:

```css
/*
Theme Name: My Theme
Version: 1.0.0
*/
```

### Step 4: Create a release

Choose one of these methods:

**Method A: Using GitHub Releases (Recommended)**
1. Create and push a tag: `git tag 1.0.0 && git push origin 1.0.0`
2. Go to your repository on GitHub
3. Click **Releases** > **Create a new release**
4. Select your tag, add release notes, and publish
5. The action will automatically deploy and attach the ZIP file

**Method B: Push tag directly**
```bash
git tag -a 1.0.0 -m "Version 1.0.0"
git push origin 1.0.0
```

### Step 5: Verify deployment

1. Check the **Actions** tab in your GitHub repository to see the workflow status
2. Once completed, verify your theme on WordPress.org

## Important Notes

* Your theme **must already be approved** and exist in the WordPress.org theme directory
* The `SLUG` must match your theme's slug on WordPress.org exactly (if not specified, it uses your GitHub repository name)
* Tags should follow [semantic versioning](https://semver.org/) (e.g., 1.0.0, 1.0.1, 2.0.0)
* The version in your `style.css` **must match** the tag version you're releasing
* When using the `release` trigger, ensure you have `permissions: contents: write` set in your workflow

## Troubleshooting

### Build step needed?

If your theme requires compilation (Sass, JavaScript bundling, etc.), add a build step before the deploy step:

```yaml
- name: Build
  run: |
    npm install
    npm run build
```

If your theme doesn't need building, remove this step entirely.

### Using a different repository slug?

If your GitHub repository name doesn't match your WordPress.org theme slug, set the `SLUG` environment variable:

```yaml
env:
  SVN_PASSWORD: ${{ secrets.WPORG_SVN_PASSWORD }}
  SVN_USERNAME: ${{ secrets.WPORG_SVN_USERNAME }}
  SLUG: my-wordpress-theme
```

### Want to test without deploying?

Use the `dry-run` option to test your workflow:

```yaml
with:
  dry-run: true
```

This doesn't require SVN credentials and won't commit anything to WordPress.org.

## License

MIT License

## Credits

Based on the [WordPress Plugin Deploy Action](https://github.com/10up/action-wordpress-plugin-deploy) by 10up.
