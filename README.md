# WordPress.org Theme Deploy

> Deploy your theme to the WordPress.org repository using GitHub Actions.

This Action commits the contents of your Git tag to the WordPress.org theme repository using the same tag name. It can exclude files as defined in either `.distignore` or `.gitattributes`.

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

## Example Workflow File

To get started, you will want to copy the contents of the example below into `.github/workflows/deploy.yml` and push that to your repository. You are welcome to name the file something else, but it must be in that directory. The usage of `ubuntu-latest` is recommended for compatibility with required dependencies in this Action.

### Deploy on pushing a new tag

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
      uses: ./action-wordpress-theme-deploy
      env:
        SVN_PASSWORD: ${{ secrets.SVN_PASSWORD }}
        SVN_USERNAME: ${{ secrets.SVN_USERNAME }}
        SLUG: deltra # optional, remove if GitHub repo name matches SVN slug, including capitalization
```

### Deploy on pushing a new tag and create release with ZIP

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
      uses: ./action-wordpress-theme-deploy
      with:
        generate-zip: true
      env:
        SVN_PASSWORD: ${{ secrets.SVN_PASSWORD }}
        SVN_USERNAME: ${{ secrets.SVN_USERNAME }}
    - name: Upload release asset
      uses: actions/upload-artifact@v4
      with:
        name: theme-zip
        path: ${{ steps.deploy.outputs.zip-path }}
    - name: Create Release
      uses: ncipollo/release-action@v1
      with:
        artifacts: ${{ steps.deploy.outputs.zip-path }}
        token: ${{ secrets.GITHUB_TOKEN }}
```

## Setup Instructions

1. **Create the action directory** in your theme repository (if you want to use it locally) or reference it from a published repository
2. **Add the workflow file** to `.github/workflows/deploy.yml` in your theme repository
3. **Add WordPress.org SVN credentials** as secrets in your repository:
   - Go to your repository Settings > Secrets and variables > Actions
   - Add `SVN_USERNAME` (your WordPress.org username)
   - Add `SVN_PASSWORD` (your WordPress.org password)
4. **Create a tag** to trigger the deployment:
   ```bash
   git tag -a 1.0.0 -m "Version 1.0.0"
   git push origin 1.0.0
   ```

## Important Notes

* Your theme must already be approved and exist in the WordPress.org theme directory
* The `SLUG` must match your theme's slug on WordPress.org exactly
* Tags should follow semantic versioning (e.g., 1.0.0, 1.0.1, etc.)
* The version in your `style.css` should match the tag version

## License

MIT License

## Credits

Based on the [WordPress Plugin Deploy Action](https://github.com/10up/action-wordpress-plugin-deploy) by 10up.
