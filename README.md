# django-distill

`django-distill` is a minimal configuration static site generator and publisher
for Django.

`django-distill` extends existing Django sites with the ability to export
fully functional static sites. It is suitable for sites such as blogs that have
a mostly static front end but you still want to use a CMS to manage the
content.

It plugs directly into the existing Django framework without the need to write
custom renderers or other more verbose code. You can also use existing fully
dynamic sites and just generate static pages for a small subsection of pages
rather than the entire site.

For static files on CDNs we use the following 'cache buster' library to allow
for fast static media updates when pushing changes:

https://github.com/mgrp/django-cachekiller

# Installation

Install from pip:

```bash
$ pip install django-distill
```

Add `django_distill` to your `INSTALLED_APPS` in your `settings.py`:

```python
INSTALLED_APPS += ('django_distill',)
```

That's it.

# Limitations

`django-distill` generates static pages and therefore only views which allow
`GET` requests that return an `HTTP 200` status code are supported.

It is assumed you are using URI parameters such as `/blog/123-abc` and not
querystring parameters such as `/blog?post_id=123&title=abc`. Querystring
parameters do not make sense for static page generation for obvious reasons.

Additionally With one-off static pages dynamic internationalisation won't work
so all files are generated using the `LANGUAGE_CODE` value in your
`settings.py`.

Static media files such as images and style sheets are copied from your static
media directory defined in `STATIC_ROOT`. This means that you will want to run
`./manage.py collectstatic` **before** you run `./manage.py distill-local`
if you have made changes to static media. `django-distill` doesn't chain this
request by design, however you can enable it with the `--collectstatic`
argument.

# Usage

Assuming you have an existing Django project, edit a `urls.py` to include the
`distill_url` function which replaces Django's standard `url` function and
supports the new keyword arguments `distill_func` and `distill_file`. The
`distill_func` argument should be provided with a function or callable class
that returns an iterable or None. The `distill_file` argument is entirely
optional and allows you to override the URL that would otherwise be generated
from the reverse of the URL regex. This allows you to rename URLs like `/` to
something more sensible for static files like `index.html`. An example distill
setup for a theoretical blogging app would be:

```python
# replaces the standard django.conf.urls.url, identical syntax
from django_distill import distill_url

# views and models from a theoretical blogging app
from blog.views import PostIndex, PostView, PostYear
from blog.models import Post

def get_index():
    # The index URI regex, ^$, contains no parameters, named or otherwise.
    # You can simply just return nothing here.
    return None

def get_all_blogposts():
    # This function needs to return an iterable of dictionaries. Dictionaries
    # are required as the URL this distill function is for has named parameters.
    # You can just export a small subset of values here if you wish to
    # limit what pages will be generated.
    for post in Post.objects.all():
        yield {'blog_id': post_id, 'blog_title': post.title}

def get_years():
    # You can also just return an iterable containing static strings if the
    # URL only has one argument and you are using positional URL parameters:
    return ('2014', '2015')
    # This is really just shorthand for (('2014',), ('2015',))

urlpatterns = (
    # e.g. / the blog index
    distill_url(r'^$',
                PostIndex.as_view(),
                name='blog-index',
                distill_func=get_index,
                # / is not a valid file name! override it to index.html
                distill_file='index.html'),
    # e.g. /post/123-some-post-title using named parameters
    distill_url(r'^post/(?P<blog_id>[\d]+)-(?P<blog_title>[\w]+)$',
                PostView.as_view(),
                name='blog-post',
                distill_func=get_all_blogposts),
    # e.g. /posts-by-year/2015 using positional parameters
    distill_url(r'^posts-by-year/([\d]{4}))$',
                PostYear.as_view(),
                name='blog-year',
                distill_func=get_years),
)
```

Your site will still function identically with the above changes. Internally
the `distill_func` and `distill_file` parameters are removed and the URL is
passed back to Django for normal processing. This has no runtime performance
impact as this happens only once upon starting the application.

# The `distill-local` command

Once you have wrapped the URLs you want to generate statically you can now
generate a complete functioning static site with:

```bash
$ ./manage.py distill-local [optional /path/to/export/directory]
```

Under the hood this simply iterates all URLs registered with `distill_url` and
generates the pages for them using parts of the Django testing framework to
spoof requests. Once the site pages have been rendered then files from the
`STATIC_ROOT` are copied over. Existing files with the same name are replaced in
the target directory and orphan files are deleted.

`distill-local` supports the following optional arguments:

`--collectstatic`: Automatically run `collectstatic` on your site before
rendering, this is just a shortcut to save you typing an extra command.

`--quiet`: Disable all output other than asking confirmation questions.

`--force`: Assume 'yes' to all confirmation questions.

# The `distill-publish` command

```bash
$ ./manage.py distill-publish [optional destination here]
```

If you have configured at least once publishing destination (see below) you can
use the `distill-publish` command to publish the site to a remote location.

This will perform a full synchronisation, removing any remote files that are no
longer present in the generated static site and uploading any new or changed
files. The site will be built into a temporary directory locally first when
publishing which is deleted once the site has been published. Each file will be
checked that it has been published correctly by requesting it via the
`PUBLIC_URL`.

`distill-publish` supports the following optional arguments:

`--collectstatic`: Automatically run `collectstatic` on your site before
rendering, this is just a shortcut to save you typing an extra command.

`--quiet`: Disable all output other than asking confirmation questions.

`--force`: Assume 'yes' to all confirmation questions.

**Note** that this means if you use `--force` and `--quiet` that the output
directory will have all files not part of the site export deleted without any
confirmation.

# The `distill-test-publish` command

```bash
$ ./manage.py distill-test-publish [optional destination here]
```

This will connect to your publishing target, authenticate to it, upload a
randomly named file, verify it exists on the `PUBLIC_URL` and then delete it
again. Use this to check your publishing settings are correct.

`distill-test-publish` has no arguments.

# Optional configuration settings

You can set the following optional `settings.py` variables:

**DISTILL_DIR**: string, default directory to export to:

```python
DISTILL_DIR = '/path/to/export/directory'
```

**DISTILL_PUBLISH**: dictionary, like Django's `settings.DATABASES`, supports
`default`:

```python
DISTILL_PUBLISH = {
    'default': {
        ... options ...
    },
    'some-other-target': {
        ... options ...
    },
}
```

# Publishing targets

You can automatically publish sites to various supported remote targets through
backends just like how you can use MySQL, SQLite, PostgreSQL etc. with
Django by changing the backend database engine. Currently the engines supported
by `django-distill` are:

**django_distill.backends.rackspace_files**: Publish to a Rackspace Cloud Files
  container. Requires the Python library `pyrax` (`$ pip install pyrax`). The
  container must already exist (use the Rackspace Cloud control panel). Options:

```python
'some-rackspace-container': {
    'ENGINE': 'django_distill.backends.rackspace_files',
    'PUBLIC_URL': 'http://.../',
    'USERNAME': '...',
    'API_KEY': '...',
    'REGION': '...',
    'CONTAINER': '...',
},
```

**django_distill.backends.amazon_s3**: Publish to an Amazon S3 bucket. Requires
  the Python library `boto` (`$ pip install boto`). The bucket must already
  exist (use the AWS control panel). Options:

```python
'some-s3-container': {
    'ENGINE': 'django_distill.backends.amazon_s3',
    'PUBLIC_URL': 'http://.../',
    'ACCESS_KEY_ID': '...',
    'SECRET_ACCESS_KEY': '...',
    'BUCKET': '...',
},
```

**django_distill.backends.google_storage**: Publish to a Google Cloud Storage
  bucket. Requires the Python libraries `google-api-python-client` and
  `google-cloud-storage`
  (`$ pip install google-api-python-client google-cloud-storage`). The bucket
  must already exist and be set up to host a public static website (use the
  Google Cloud control panel). Options:

```python
'some-google-storage-bucket': {
    'ENGINE': 'django_distill.backends.google_storage',
    'PUBLIC_URL': 'https://storage.googleapis.com/[bucket.name.here]/',
    'JSON_CREDENTIALS': '/path/to/some/credentials.json',
    'BUCKET': '[bucket.name.here]',
},
```

# Contributing

All properly formatted and sensible pull requests, issues and comments are
welcome.
