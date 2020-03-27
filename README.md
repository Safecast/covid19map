# covid19map

This repository manages the deployment of Safecast's [Ushahidi](https://www.ushahidi.com/) instance for COVID-19 related data, located at [https://covid19map.safecast.org](https://covid19map.safecast.org/).

## Deployment

* Check out this GitHub repository.
* Download Ushahidi release 4.3.0 from [the official release page](https://github.com/ushahidi/platform-release/releases). Under "Assets," find [`ushahidi-platform-release-v4.3.0.tar.gz`](https://github.com/ushahidi/platform-release/releases/download/v4.3.0/ushahidi-platform-release-v4.3.0.tar.gz).
* Unpack this tarball in a temporary directory somewhere.
* Copy all of the contents of `ushahidi-platform-release-v4.3.0/html/` directly into the Git repository's root directory (the directory that this file is in). **Take special care not to miss dotfiles, particularly `ushahidi-platform-release-v4.3.0/html/.htaccess`**.
* Edit the file `app.a2616d37de2f1f42dbe1.css`, inserting these lines just before the end (`sourceMappingURL` is currently still the final line, not sure if it can be moved or not):
```
.markdown-white-link a{ color:#FFFFFF;text-decoration:none;border-bottom:1px dotted #FFFFFF !important; }
.markdown-white-link a:hover{ color:#FFFFFF;border-bottom:1px solid #FFFFFF !important;}
```
Some text fields in Ushahidi accept Markdown. When creating a link in one of these fields, we surround it with `<span class="markdown-white-link">` so that the link remains readable against the site's dark background.
* Deploy the new version with `eb deploy`

## Installation from scratch

There are several one-time tasks that need to be performed before a new installation.

* Create the database, EFS mount, and DNS entries: these are handled in the [Safecast infrastructure repository](https://github.com/Safecast/infrastructure/) via Terraform.
* Create the Elastic Beanstalk environment (use the latest version of PHP) and set the following environment values:
```
MAIL_USERNAME: <generate this in AWS SES>
MAIL_PASSWORD: <generate this in AWS SES>
MAIL_HOST: email-smtp.us-west-2.amazonaws.com
DATABASE_HOST: covid19map.prd.safecast.cc
DATABASE_PASSWORD: <set this using the RDS control panel>
BACKEND_URL: https://covid19map.safecast.org
MAIL_NAME: SafecastCOVID-19Map
APP_KEY: <generate this using the instructions below -- this encrypts user cookies, therefore needs to be consistent across instances>
```
* Use this command to generate the `APP_KEY` value -- this wil ensure the proper length and reasonable randomness: `cat /dev/urandom | LC_ALL=C tr -cd 'A-Za-z0-9_\!\@\#\$\%\^\&\*\(\)-+=' | fold -w 32 | head -1`
* Start up an environment and deploy the app.
* `cd` to `/var/www/html/platform` and, as root, execute:
```
php artisan passport:keys
chown -R webapp:webapp storage/passport
chmod 770 storage/passport
chmod 660 storage/passport/*
```
* At this point, you can log in to the instance with username `admin` and password `admin`. You will be immediately prompted to add an email address and change the password.
* After you have uploaded a logo, you may need to manually change its URL to HTTPS, for example:
```
update config set config_value = "\"https:\\/\\/covid19map.safecast.org\\/storage\\/5\\/e\\/5e6f3c067d3b3-safecast-square-color.png\"" where id = 8;
```

## TODO

Most items have been entered as issues in this GitHub project.

### Forking Ushahidi

Right now, we are making only minimal changes to Ushahidi's release package. If we want to fork the codebase and make more substantial changes, we will need to:

* Fork the upstream projects (backend in PHP; frontend client in JavaScript/AngularJS)
* Create CircleCI builds for these
* Come up with a plan to synchronize with upstream

Ushahidi publishes new releases fairly regularly, so if we have general improvements or patches, it is probably to our benefit to encourage people to contribute directly to them; we can instead work on getting into a more regular update cadence on our own instance.

It's also important to note that Ushahidi is AGPL licensed, meaning that we need to publish any change we make to the code (which of course is no problem for us) -- but, specifically, we can't just hack substantial changes on the server without source-controlling them or publishing them on GitHub; that would not be in keeping with the license.
