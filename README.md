# Teamwerx Staging Auto-Deploy for WordPress

![Staging Deployment](https://github.com/lewismorgan/Teamwerx/workflows/Staging%20Deployment/badge.svg)

1. Create a local site on [Local by Flywheel](https://localwp.com/)
2. Make sure the following secrets are entered in the repository settings:
   - FTP_STAGE_SERVER=<ftp://url:port/>
   - FTP_STAGE_USER=ftp_username
   - FTP_STAGE_PASSWORD=ftp_password
3. Clone the repository, making sure `/app` folder is the root and the `/public` folder exists under it
4. Delete all tables and run the backup.sql script on the database that was created by [Local](https://localwp.com/)
5. Stop the site from running, and rerun it. You should get a message to fix URL's within the [Local](https://localwp.com/) application, accept the fix otherwise the images will not load properly.
6. Perform changes on the `develop` branch
7. Once you're done with the changes, merge `develop` into `stable`
8. Wait for the `Staging Deployment` GH Action to finish
9. Verify the changes are visible on the staging site

---

- [Teamwerx Staging Auto-Deploy for WordPress](#teamwerx-staging-auto-deploy-for-wordpress)
  - [Setup](#setup)
    - [1. Creating the Initial Repository using a wp-content folder](#1-creating-the-initial-repository-using-a-wp-content-folder)
    - [2. GitHub Deployment Workflow](#2-github-deployment-workflow)
      - [I. Create .git-ftp.log on FTP Server](#i-create-git-ftplog-on-ftp-server)
      - [II. Create Deployment Action](#ii-create-deployment-action)
      - [III. Add FTP Server Secrets to GitHub](#iii-add-ftp-server-secrets-to-github)
    - [3. Cloning Repository after Initial Setup](#3-cloning-repository-after-initial-setup)
  - [Post-Setup](#post-setup)
    - [Restore localhost database to the Teamwerx Backup Database](#restore-localhost-database-to-the-teamwerx-backup-database)
      - [Option A: Site Shell](#option-a-site-shell)
      - [Option B: MySQLWorkbench](#option-b-mysqlworkbench)
    - [Fix Site Links](#fix-site-links)
  - [Development Workflow](#development-workflow)
  - [Misc Info](#misc-info)
    - [Site Domains Mode](#site-domains-mode)
    - [ftp-deploy Action](#ftp-deploy-action)
    - [Backup Database](#backup-database)

---

## Setup

**Before beginning,** create a SQL Database backup of the WordPress site that you want to add to Git. See the [backup.sql](backup.sql) file for an example script. The guide assumes this will be called `backup.sql` located at the root of the repository.

The steps will cover the process that is required to take the contents of a WordPress `wp-content` folder and create a Git repository similar to this one from scratch, _with_ deployment to staging. These are the steps that were used to create the initial commits for the repository.

Once the repository has been setup on [GitHub](https://github.com/), you can instead follow the [cloning steps](#3-cloning-repository-after-initial-setup) to create a new development environment.

---

### 1. Creating the Initial Repository using a wp-content folder

Create a new directory named _teamwerx_.

```bash
mkdir teamwerx
```

Install [Local by Flywheel](https://localwp.com/), and navigate to Preferences->Advanced. Change the _Router Mode_ to _Site Domains_.

!['Site Domain Mode'](/.github/img/1.png)

Create a new site called _Teamwerx_ in [Local](https://localwp.com/). Change the site directory to the location of the `teamwerx/` folder. Use _Preferred_ for **Choose your Environment**. Within **Setup WordPress**, enter in information you will remember. Click on **Add Site**.

!['Initial Teamwerx Site'](/.github/img/2.png)

The file structure within `teamwerx/` should resemble the same as below using `ls -a`:

- app/
- conf/
- log/

The file structure within `teamwerx/app/` should resemble the same as below using `ls -a`:

- public/

Before continuing, ensure that you can [visit the site](https://teamwerx.local).

!['Local Default Site'](/.github/img/3.png)

Create a new git repository within `~/Local Sites/teamwerx/app`. The actual git repository will end up residing in this directory, `~/Local Sites/teamwerx/app`.

```bash
git init
```

**Note**: Users will need to create an initial site in [Local by Flywheel](https://localwp.com/) before cloning the repository. See [Cloning Repository after Initial Setup](#3-cloning-repository-after-initial-setup) for a guide on how to setup a site via cloning the repository from GitHub.

Create the repository on [GitHub](https://github.com/). The below command is using the [GitHub CLI](https://cli.github.com/manual/gh_repo_create) within `teamwerx/app`.

```bash
gh repo create teamwerx-app --private
```

Create a new file named `.gitignore` with the contents of [this repository's .gitignore](/.gitignore) within `~/Local Sites/teamwerx/app` directory.

**Note**: I used [jq](https://stedolan.github.io/jq/) and [GitHub CLI](https://cli.github.com/) to download the file with an Authentication Token from a private repository.

```bash
curl $(gh api repos/lewismorgan/Teamwerx/contents/.gitignore | jq '. | .download_url' | sed 's/"//g') --output .gitignore
git add .gitignore
```

Create the initial commit. You can squash everything later once all the files are in to reduce the size of the git deltas.

```bash
git add public/
git commit -m 'Initial commit'
```

Results of `ls -a` within `teamwerx/app` should now look like this:

- .git/
- public/
- .gitignore

Once you've confirmed the repository has been created and the file heiarchy is the same as above for `~/Local Sites/teamwerx/app`, push the changes. If you used the [GitHub CLI](https://cli.github.com/), then a new remote to the repository should be added automatically called `origin`. You just need to now set the `master` branch to track it before pushing.

```bash
git push --set-upstream origin master
```

Download the contents of the `ftp://wp-content/` folder for the site you want to add to Git via FTP into a seperate location from where the Git repository resides. For this repository, the files were downloaded via FTP into a seperate folder and shared on OneDrive. You should now have 2 different folders:

- `ftp://wp-content/` copied into a local backup folder: `/ftp-wp-content`
- `~/Local Sites/teamwerx/app/public/wp-content/` folder

Delete all the contents of the `~/Local Sites/teamwerx/app/public/wp-content/` folder and replace it with the contents of `/ftp-wp-content` folder.

The below bash code uses the _Teamwerx_ site that had it's contents FTP downloaded and then shared through OneDrive. Contents are then synced from the OneDrive folder to `~/Local Sites/teamwerx/app/public/wp-content`.
Environment variable `$FTP_BACKUP_FOLDER` refers to the OneDrive folder `/ftp-wp-content` and the `cwd` is `~/Local Sites/teamwerx/app/public`.

```bash
rm -rf wp-content
git commit -am 'Remove wp-content'
rsync -a $FTP_BACKUP_FOLDER/wp-content ~/'Local Sites'/teamwerx/app/public
git add wp-content/
git commit -m 'Add wp-content files from current FTP server'
git status
```

Review the status and make sure there are no changes to the repository. Push once everything looks good.

```bash
git push
gh repo view -w
```

Add a copy of the SQL backup, `backup.sql`, for the entirety of the WordPress site and add it to the repository. Commit and push to the repository. The `cwd` is `~/Local Sites/teamwerx/app`.

```bash
rsync $FTP_BACKUP_FOLDER/backup.sql ~/'Local Sites'/teamwerx/app
git add backup.sql
git commit -m 'Add backup.sql'
git push
gh repo view -w
```

Squash the commits to a single initial commit to help reduce the size of the repository on a new branch called `develop`

```bash
git rebase -i
git checkout -b develop
git push --set-upstream origin develop
```

Restore the SQL backup following the [Restore localhost database to the Teamwerx Backup Database](#restore-localhost-database-to-the-teamwerx-backup-database) process.

---

### 2. GitHub Deployment Workflow

After the repository has been setup, you need to now create the workflow for pushing to FTP when changes are made on a branch (staging in this case).

In order for the action to work correctly, you need to tell it what commit the latest FTP deploy occured with using a `.git-ftp.log` file. See more information in [ftp-deploy action](#ftp-deploy-action) and how to setup one up under [Create .git-ftp.log on FTP Server](#create-git-ftplog-on-ftp-server).

#### I. Create .git-ftp.log on FTP Server

Performing this task will make it so once the workflow runs, it will not do a full deployment on the first run. It will only upload changed files from the last deployed commit SHA.

Install Cyberduck and establish a connection via FTP to the server. Make sure you can view hidden files.

Once connected, create a new file called `.git-ftp.log` with the **long SHA** of the _latest unpushed commit_ on the repository.
Your repository should already match the contents of the FTP servers `wp-content/` folder if you haven't made any content changes since creating the repository. If not, then you will need to upload all the changed files before continuing. Any changes prior to the commit entered in the file will not be recognized. See [ftp-deploy Action](#ftp-deploy-action) for more information.

```bash
git rev-parse HEAD > .git-ftp.log
```

Upload the created `.git-ftp.log` file that contains the commit SHA to the FTP server in the root `/` directory.

<!-- !['Cyberduck .git-ftp.log upload'](/.github/img/14.png) -->

#### II. Create Deployment Action

Create a new directory called `.github/workflows`

```bash
mkdir -p .github/workflows
```

Copy and paste the code in [.github/workflows/staging.yaml](/.github/workflows/staging.yaml) to `/.github/workflows/staging.yaml`.

**Note**: I used [jq](https://stedolan.github.io/jq/) and [GitHub CLI](https://cli.github.com/) to download the file with an Authentication Token from a private repository.

```bash
curl $(gh api repos/lewismorgan/Teamwerx/contents/.github/workflows/staging.yaml | jq '. | .download_url' | sed 's/"//g') --output .github/workflows/staging.yaml
git add .github/workflows/staging.yaml
```

**_Note:_** Fetch depth is set to 10 because it automatically will only fetch the most recent commit. When [GitHub](https://github.com/) pulls the repository, it will not contain older history past **10 commits**. If you push more changes to the `staging` branch than 10 commits, the action will do a full deploy because it will act like the commit does not exist. Fix this by changing the `fetch-depth` to a number larger than the commit history since the last deployment. See [ftp-deploy Action](#ftp-deploy-action) for more info.

Uncomment the line that contains the `git-ftp-args` parameter to enable a dry run. Dry run will make it so that no files are actually uploaded onto the FTP server, it'll only log the files that _could_ be uploaded.

```yml
---
- name: FTP Deploy
  uses: SamKirkland/FTP-Deploy-Action@3.1.1
  with:
    ftp-server: ${{ secrets.FTP_STAGE_SERVER }}
    ftp-username: ${{ secrets.FTP_STAGE_USER }}
    ftp-password: ${{ secrets.FTP_STAGE_PASSWORD }}
    local-dir: public
    git-ftp-args: --dry-run
```

Commit and push

```bash
git add .github/workflows/staging.yaml
git commit -m 'Add staging workflow'
git push
```

Create a new branch called `staging` and push `upstream`.

```bash
git checkout -b staging
git push --set-upstream origin staging
```

View the [GitHub](https://github.com/) repository's `Actions` page. The workflow `Staging Deployment` should now be running and fail after `git push` until the secrets have been entered.

[Add in the ftp server secrets](#iii-add-ftp-server-secrets-to-github) to fix `Staging Deployment` and re-run the workflow.

#### III. Add FTP Server Secrets to GitHub

The `Staging Deployment` workflow that was created has defined 3 secret variables that will need to be added to the repository in order for the workflow to work properly. Secrets can not be viewed once entered, even by the administrator.

Enter the values for the following secrets within the [GitHub](https://github.com/) repositories `Settings -> Secrets` page:

- FTP_STAGE_SERVER=<ftp://stage.server:port>
- FTP_STAGE_USER=ftp_stage_username
- FTP_STAGE_PASSWORD=ftp_stage_password

!['GH Actions FTP Secrets'](/.github/img/15.png)

Rerun the workflow by selecting `Re-run all jobs` options on the failed [GitHub Actions workflow page](https://github.com/lewismorgan/teamwerx/actions).

!['GH Actions Rerun Jobs'](/.github/img/16.png)

Wait for the action to finish running. It should show a green checkmark now. If it does not, re-check the secrets that were entered. Within the build log for the `FTP Deploy` job step, it should say that no files were changed but there is a different commit ID. This just means that files outside the `/public` folder changed.

<!-- !['GH Actions FTP Deploy Job'](/.github/img/17.png) -->

Run through the development workflow again making changes to the sites header following the [Development workflow](#development-workflow) guide.

Once you verified that the correct files are being uploaded throughout the [development workflow](#development-workflow) process, comment out the following line in `.github/workflows/staging.yml` to do an actual deployment to the FTP server:

```yaml
git-ftp-args: --dry-run
```

Check the FTP server's `.git-ftp.log` file to verify that the commit SHA is the latest commit pushed to the `staging` branch after commenting out the `git-ftp-args` dry run argument `--dry-run`.

---

### 3. Cloning Repository after Initial Setup

Install [Local by Flywheel](https://localwp.com/), and navigate to `Preferences -> Advanced`. Change the _Router Mode_ to _Site Domains_.

!['Site Domain Mode'](/.github/img/1.png)

Create a new site called _Teamwerx_ in [Local](https://localwp.com/), leave the defaults for all the steps.

!['New Teamwerx Site'](/.github/img/2.png)

Make the `~/Local Sites/teamwerx/app` folder the `cwd`

```bash
cd ~/'Local Sites'/teamwerx/app
```

Clone the repository into the `cwd`, `~/Local Sites/teamwerx/app`

```bash
gh repo clone lewismorgan/Teamwerx
```

A new folder will be created in `~/Local Sites/teamwerx/app` with the name of the repository, Teamwerx.
_Copy the contents of the cloned Teamwerx folder_ from the git clone, `~/Local Sites/teamwerx/app/Teamwerx`, into the `cwd`, `~/Local Sites/teamwerx/app`

```bash
rsync -a Teamwerx/ ~/'Local Sites'/teamwerx/app
```

Make sure the folder `~/Local Sites/teamwerx/app` now contains the content of the cloned repository using `ls -a`:

- .git/
- .github/
- bin/
- public/
- Teamwerx/

Delete the git cloned `~/Local Sites/teamwerx/app/Teamwerx/` folder once the contents of the folder have been copied into `~/Local Sites/teamwerx/app`

```bash
rm -rf Teamwerx
```

Verify that git is now working within `~/Local Sites/teamwerx/app` and shows no changes to commit, it might take a while the first time to refresh the index.

```bash
git status
```

Navigate to [teamwerx.local](http://teamwerx.local/) in your web browser. The default theme should be shown.

!['Local Site without DB install'](/.github/img/3.png)

Restore the database for the created site using the `backup.sql` script. Follow the [Restore localhost database to the Teamwerx Backup Database](#restore-localhost-database-to-the-teamwerx-backup-database) guide on the exact steps to accomplish this.
Once the database has been restored, the development environment has been setup.

## Post-Setup

### Restore localhost database to the Teamwerx Backup Database

Ensure that the local Teamwerx site is running and open the Site Shell within [Local](https://localwp.com/)

!['Site Shell'](/.github/img/4.png)

Create a root user that can access 127.0.0.1 with full permissions within the Site Shell. **_You must do this step within the site shell._**

```bash
mysql -e "CREATE USER 'root'@'127.0.0.1' IDENTIFIED BY 'root'; GRANT ALL ON *.* TO 'root'@'127.0.0.1';"
```

Determine the port that the created database is located on to access via MySQLWorkbench or other tool

```bash
mysql -e "SHOW VARIABLES WHERE Variable_name='port'";
```

!['Teamwerx local database port'](/.github/img/5.png)

Connect to the Teamwerx localhost database to drop the tables and restore `backup.sql` either through the [Site Shell (Option A)](#option-a-site-shell) or [MySQLWorkbench (Option B)](#option-b-mysqlworkbench).

---

#### Option A: Site Shell

Open the _Teamwerx Site Shell_ from [Local by Flywheel](https://localwp.com/).

!['Site Shell'](/.github/img/4.png)

Enter the `mysql` prompt with the following command, typing in the password for `root` when prompted

```bash
mysql -u root -p
```

Drop all the wordpress tables from the `local` schema within the `mysql` prompt

```sql
DROP TABLE `local`.`wp_commentmeta`, `local`.`wp_comments`, `local`.`wp_links`, `local`.`wp_options`, `local`.`wp_postmeta`, `local`.`wp_posts`, `local`.`wp_termmeta`, `local`.`wp_term_relationships`, `local`.`wp_terms`, `local`.`wp_term_taxonomy`, `local`.`wp_usermeta`, `local`.`wp_users`;
```

Enter `quit` to exit the prompt

Restore the database from the backup

```bash
mysql -u root -p local < ../backup.sql
```

Continue to [Fix Site Links](#fix-site-links) section

---

#### Option B: MySQLWorkbench

Install [MySQLWorkbench](https://www.mysql.com/products/workbench/) and connect to the database.

Enter the below values and test connection.

- Hostname: 127.0.0.1
- Port: Found from `mysql -e "SHOW VARIABLES WHERE Variable_name='port'";`
- Username: root
- Password: root

!['MySQLWorkbench Settings'](/.github/img/6.png)

Select all of the tables for the `local` database schema and drop them.

!['MySQLWorkbench Drop Tables'](/.github/img/7.png)

File->Run SQL Script... and select `backup.sql` located at `~/Local Sites/teamwerx/app/backup.sql`. Choose `local` as the default schema. Press **Run**.

!['MySQLWorkbench Run SQL Script'](/.github/img/8.png)

A window will open, showing the following output:

```Preparing...
Importing backup.sql...
Finished executing script
Operation completed successfully
```

Close the window and continue to the [Fix Site Links](#fix-site-links) section. Once the links have been fixed, you should be able to view the site properly and the setup is complete.

---

### Fix Site Links

Stop the site and rerun through [Local by Flywheel](https://localwp.com/) after restoring the database to refresh the webserver.

A warning will come up saying that the URL settings do not match. Press _Fix It_.

!['Fix Site Links'](/.github/img/9.png)

Wait for [Local](https://localwp.com/) to reload the site. Press visit site to view the site. It should now be themed.

!['Teamwerx Local Site'](/.github/img/10.png)

## Development Workflow

Once the repsoitory has been setup on [GitHub](https://github.com/) following [the initial repository setup steps](#1-creating-the-initial-repository-using-a-wp-content-folder), you can additionally clone the repository following the [repository cloning steps](#3-cloning-repository-after-initial-setup) to create a new development environment.

Checkout the `develop` branch

```bash
git checkout develop
```

Make changes to the site, such as adding in a header. Once you have changes commit them and push to the repository.

```bash
git commit -am 'Fixed site header'
git push
```

Create a Pull Request to merge `develop` into `staging`

!['Pull Request'](/.github/img/11.png)

**Note**: Files are uploaded based on the last Commit SHA that is referenced inside `.git-ftp.log` located on the FTP server. Only the **10** most recent commits are fetched by the action, otherwise a full FTP deploy will occur. See [FTP Deploy Info](#ftp-deploy-action).

Approve the Pull Request and visit the [Actions](https://github.com/lewismorgan/Teamwerx/actions). I reccomend protecting both the `develop` branch and the `staging` branch so GitHub stops telling you that the branches can be deleted after merging. Merging a PR will automatically delete a branch if you're using the `gh` CLI to merge.

![Deployment In Progress](/.github/img/12.png)

Wait for the [Staging Deployment](https://github.com/lewismorgan/Teamwerx/actions?query=workflow%3A%22Staging+Deployment%22) action to finish. Verify that only the modified files were uploaded _if dry run is disabled_. Otherwise, check to make sure that only the changed files _would of been uploaded_ via the `FTP Deploy` section in the Action's log.

<!-- ![Deploy Complete](/.github/img/13.png) -->

Visit the published staging site to view the staged changes once the deployment has completed.

Lastly, it might be helpful to rebase `develop` on top of `staging` if there are no more added commits to `develop` since the merge into `staging` for a cleaner tree with merge commits.

![Commit Tree](/.github/img/18.png)

## Misc Info

### Site Domains Mode

In order to access the admin panel, you need to enable _HTTPS_ since _localhost:port_ does not support HTTPS. Fix this by enabling **_Site Domains_** mode within Local.

Navigate to `Preferences -> Advanced`. Change the _Router Mode_ to _Site Domains_.

!['Site Domain Mode'](/.github/img/1.png)

### ftp-deploy Action

Deployment works by using the [ftp-deploy](https://github.com/marketplace/actions/ftp-deploy) action. The action works by reading the commit sha from a file located on the FTP server called `.git-ftp.log`.

If `.git-ftp.log` does not exist on the server, then the action will deploy all of the files via FTP. Only the files changed since that stored commit SHA will be uploaded via FTP.

If the `fetch-depth` option for [actions/checkout@v2](https://github.com/marketplace/actions/checkout) is less than the number of commits `staging` is behind on, then a full FTP deploy will occur.
Mitigate this by changing fetch-depth to a number higher than the commits `staging` is behind on.

### Backup Database

Create a backup the database by running the following command in a shell

`mysqldump --skip-add-drop-table $DB_NAME > backup.sql`
