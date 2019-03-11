# JekyllMail #

JekyllMail enables you to post to your [Jekyll](https://github.com/mojombo/jekyll)
or [Octopress](http://octopress.org/) powered blog by email.

## How it Works ##
Once configured (see below) JekyllMail will log into a POP3 account, check for messages with a pre-defined secret in the subject line, convert them into appropriately named files, and save them in your `_posts` directory. Images will be extracted and saved in a date specific directory under your Jekyll images directory.

Please note. JekyllMail assumes that the address it is checking  is *exclusively* for its use and will only be used to post emails  to a single blog. JekyllMail does support multiple blogs but you  will need a seperate e-mail account for each one.


## Usage ##
The magic is all in the subject line. In order to differentiate your email from the spam that's almost guaranteed to find your account eventually suck in the appropriate metadata A subject line for JekyllMail has two parts the title (of your post) and the metadata which will go into the YAML frontmatter Jekyll needs. The metadata is a series of key value pairs separated by slashes. One of those key value pairs *must* be "secret" and the secret listed in your configuration. Note that the keys must contain no spaces and be immediately followed by a colon.

	<subject> || key: value / key: value / key: value, value, value
An example:

	My Awesome Post || secret: more-1337 / tags: awesome, excellent, spectacular

Your secret should be short, easy to remember, easy to type, and very unlikely to show up in an e-mail from another human or spammer.

Your e-mail can be formatted in Markdown, Textile, or HTML.

### Subject Metadata ###
Metadata is separated from your subject by a double pipe. There are a handful of keys that JekyllMail is specifically looking for in the subject.
**All of these are optional except "secret"**:

* published: defaults to true. Set this to "false" to prevent the post from being published.
* markup: can be: `html`, `markdown`, or `textile`
* tags: expects a comma separated list of tags for your post
* slug: the "slug" for the file-name. E.g. yyyy-mm-dd-*slug*.extension

If you don't provide a slug JekyllMail will just convert your title to a slug

### Images ###
Image attachments will be extracted by JekyllMail and placed in dated directory
that corresponds with the date of the posting.

For example If you attached flag.jpg to a post sent on July 4th 2012 it would be
stored in `<images_dir>/2012/07/04/flag.jpg`


JekyllMail will look for the image tags in your document that reference the image
filename and update them to point to the correct published file path. For example
it will convert `![alt text](flag.jpg)` in a Markdown document to
`![alt text](http://example.com/path/to/images/dir/2012/07/04/flag.jpg)`.
Textile and HTML posts are also supported.

In practice this simply means that if you insert a `![alt text](flag.jpg)`
tag and attach an image named `flag.jpg` to the same email everything will
show up as expected in your post even though JekyllMail has moved that image
off to a dated subdirectory (just like the post's url).

## Installation ##
Clone this git repo on your server, cd into the resulting directory, and
run `bundle install` to make sure all the required gems are present.

Required Gems:

* bundler
* jekyll
* nokogiri
* mail

After editing the `_config.yml` file (in JekyllMail) you'll need to wire things up to rebuild after each commit. Instructions are below.

## Configuration ##
JekyllMail is configured via a \_config.yml file in its root directory.
Within this are a couple global settings and a series of "blog" stanzas one for each blog you'll have it checking mail for.

A config file for a single blog will look something like this:

```yaml
---
debug: false
blogs:
- name: my blog name
  active: true
  markup: markdown
  jekyll_blog_dir: /Users/masukomi/workspace/jekyllmail_test_site
  images_dir_under_jekyll: assets/img
  posts_dir_under_jekyll: _posts
  images_dir_under_site_url: /assets/img

  origin_repo_branch: master

  local_repo: /Users/masukomi/workspace/jekyllmail_test_site
  origin_repo: /Users/masukomi/workspace/jekyllmail_test_site
  pop_server: mail.example.com
  pop_user: jekyllmail@example.com
  pop_password: a_really_good_password_goes_here
  secret: jekyllmail
  markup: markdown
  site_url: http://blog.example.com
  commit_after_save: true
  delete_after_run: true

```

### Configuration Notes ###

`jekyll_blog_dir` is the absolute path to your Jekyll install on the same box as the `jekyllmail.rb` script.

`images_dir_under_jekyll` defaults to `assets/img` in current Jekyll deploys.  It represents the directory under your Jekyll root where images are stored.

`post_dir_under_jekyll` defaults to `_posts` in current Jekyll deploys.

`images_dir_under_site_url` is `/assets/img` with a default Jekyll configuration.  It means that you would serve an image from `https://<my_site>/assets/img/foo.jpg`

`origin_repo_branch` JekyllMail can push changes from your local
Jekyll install to a remote repo. It assumes that remote repo will be named `origin` but you can configure what branch it will push to.

`local_repo` JekyllMail assumes that your local Jekyll install is a git repository. if `origin_repo` is different it will try and push to it.

The `secret` is a short piece of text that must appear in the subject of
each email. This is used to filter out the spam and will never be posted.

`local_repo` is where JekyllMail will store your files during its run. This will be automatically configured to push to `origin_repo` if `commit_after_save` and `origin_repo` are specified. Any new posts and images to the `local_repo` will be pushed to `origin_repo`, and then deleted after the run is complete.

`delete_after_run` tells JekyllMail to delete the emails after successfully processing them. You probably want this set to true unless you're debugging and want to reprocess the same email(s) repeatedly.


Please note that paths must *not* end with a slash.  Your `pop_user` doesn't have to be an e-mail address. It might just be "jekyllmail", or whatever username you've chosen for the e-mail account.  It all depends on how your server is configured. It's probably best to use something other than "jekyllmail" though.

## Wiring Things up ##
You need to schedule a cronjob to run regularly to kick of JekyllMail and check for new e-mails.

To kick of JekyllMail you'll want a script that looks something like this.
You can use the `run_jekyllmail.sh` file that comes with JekyllMail as
a template.

```bash
	#!/bin/sh
	cd /full/path/to/jekyllmail
	bundle exec ruby jekyllmail.rb
```


Save the file anywhere that isn't served up to the public, make it executable, and add a new line to your [crontab](http://crontab.org/) to run it every five minutes or so. This is an example crontab line to do this

```
4,9,14,19,24,29,34,39,44,49,54,59    *    *    *    * /home/my_username/jekyllmail_repo/run_jekyllmail.sh
```

When JekyllMail finds something it will save the files in the appropriate locations and commit them to the appropriate git repo. When it does we can leverage git's `hooks/post-commit` to regenerate the HTML without needing to unnecessarily rebuild it on a regular interval.

The `hooks/post-commit` file in your repo should look something like this.  Don't forget to make it executable. You can use the `build_site.sh` file that comes with JekyllMail as a templote.

```bash
#!/bin/sh
cd /full/path/to/blog
jekyll build
cp -r _site /full/path/to/the/directory/your/site/is/hosted/from
```

Depending on your server's ruby / gem configuration you may have to add some additional info to the top of those scripts ( just below the `#!/bin/sh` ). On a system with a locally installed RVM and gems directory the top of your script might look something like this:

```bash
#!/bin/sh
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm" # Load RVM into a shell session *as a function*
GEM_PATH=$GEM_PATH:/home/my_username/.gems
PATH=$PATH:/home/my_username/.gems/bin
```

## Warning ##
At the end of every run JekyllMail *deletes every e-mail* in the account.
This is for two reasons:

1. We don't want to have to maintain a list of what e-mails we've already ingested and posted
2. Once an e-mail's been ingested we don't need it
3. There are probably 400 spam e-mails in the account that should be deleted anyway.
4. less e-mail in the box means faster runs

Ok, four reasons.

If you want to disable this set the `delete_after_run` configuration setting to false.

## Developers ##
If you set the `debug` option at the top if the configuration file to `true` it will cause a bunch of debug statements to be printed during the run, and logged to the log file.  It will also prevent it from deleting the e-mails at the end. It's much easier to work on JekyllMail when you don't have to keep sending it new e-mails.

Have fun, and remember to send in pull-requests. :)

### Known Issues ###
Check out the [Issues page](https://github.com/masukomi/JekyllMail/issues) on
Github for the current list of known issues (if any).

## Credit where credit is due ##
JekyllMail was based on a [post & gist](http://tedkulp.com/2011/05/18/send-email-to-jekyll/)  by [Ted Kulp](http://tedkulp.com/), but has come a long way since then.

## License ##
JekyllMail is distributed under the [MIT License](http://www.opensource.org/licenses/mit-license.php).


