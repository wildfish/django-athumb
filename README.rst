django-athumb
=============

:Author: Greg Taylor
:License: BSD
:Status: Stable, but un-maintained (file an issue if you'd like to take over!)

Storing images and their thumbnails on S3 is a bit of a clumbsy endeavor with
Django. While this Django app may work with more typical storage backends, it
is intended to accept image uploads, thumbnail them, and upload the original
plus the thumbs to S3. You may then get to the thumbnails in your template
by doing something like::

    <img src="{% thumbnail some_obj.image '80x80' %}" />

This automatically assembles the remote S3 URL to retrieve the thumbnail from.
No error checking is done, and several assumptions are made for the sake of
speed.

Advantages of django-athumb
---------------------------

The primary advantage of django-athumb is that, unlike sorl and others,
thumbnails are generated at the time of user uploading the original image.
Instead of generating thumbs on-demand, or needing a caching layer, we generate
pre-defined thumbnails at time of upload. This leads to a few big benefits:

* We never check for the existence of a file, after the first save/upload. We
  assume it exists, skipping the need to hit caches or the disk. This makes
  us faster and more simple than some of the other alternatives.
* Since we define every possible thumbnail in advance via models.py, we have
  a defined set of possible values. Thumbnails sizes can also be named anything,
  whether it be '60x60_cropped' or 'medium_nocrop'.


Disadvantages of django-athumb
------------------------------

This wouldn't be a fair assessment without some of these:

* Some may prefer to define thumbnail sizes in templates. This is largely
  preference.
* We only support simple scaling/cropping/centering as thumbnail operations.
  sorl and others are going to give you a whole lot more gadgets out of the box.


License
-------

All code is under a BSD-style license, see LICENSE for details.

Source: http://github.com/duointeractive/django-athumb

Requirements
------------

* python >= 2.7
* django >= 1.6
* boto >= 3.0
* Pillow >= 2.5.0

Installation
------------

To install run::

    pip install django-athumb

or::

    easy_install django-athumb

Configuration
-------------

settings.py
^^^^^^^^^^^

Add to ``INSTALLED_APPS``::

    'athumb'

Add to ``TEMPLATE_CONTEXT_PROCESSORS`` in ``settings.py``::

    'django.core.context_processors.request'

If you want S3 storage as your default file back-end::

    # If you don't want this to be the global default, just make sure you
    # specify the S3BotoStorage_AllPublic backend on a per-field basis.
    DEFAULT_FILE_STORAGE = 'athumb.backends.s3boto.S3BotoStorage_AllPublic'

Then setup some values used by the backend::

    AWS_ACCESS_KEY_ID = 'YourS3AccessKeyHere'
    AWS_SECRET_ACCESS_KEY = 'YourS3SecretAccessKeyHere'
    AWS_STORAGE_BUCKET_NAME = 'OneOfYourBuckets'

If you would like to use a vanity domain instead of s3.amazonaws.com, you
first should configure it in amazon and then add this to settings::

    AWS_STORAGE_BUCKET_CNAME = 'static.yourdomain.com'

If you want a cache buster for your thumbnails (a string added to the end of
the image URL that causes browsers to re-fetch the image after changes), you
can set a value like this::

    MEDIA_CACHE_BUSTER = 'SomeValue'

You do not need to specify a cache buster.

If you aren't using the default S3 region, you can define it with the following
setting::

    AWS_REGION = 'us-east-1'

Using in models
---------------

After you have all of the above configured, you're ready to start using
athumb in your models. Here is an example model with a thumbnailing field.

::

    from django.db import models
    from athumb.fields import ImageWithThumbsField
    from athumb.backends.s3boto import S3BotoStorage_AllPublic

    # It is generally good to keep these stored in their own module, to allow
    # for other models.py modules to import the values. This assumes that more
    # than one model stores stuff in the same bucket.
    PUBLIC_MEDIA_BUCKET = S3BotoStorage_AllPublic(bucket='public-media')

    class YourModel(models.Model)
        image = ImageWithThumbsField(
            upload_to="store/product_images",
            thumbs=(
                ('50x50_cropped', {'size': (50, 50), 'crop': True}),
                ('60x60', {'size': (60, 60)}),
                ('80x1000', {'size': (80, 1000)}),
                ('front_page', {'size': (120, 1000)}),
                ('medium', {'size': (161, 1000)}),
                ('large', {'size': (200, 1000)}),
            ),
            blank=True, null=True,
            storage=PUBLIC_MEDIA_BUCKET)

A few things to note:

* The tuples in `thumbs` are in the format of `(name, options)`. The value
  for `name` can be whatever string you'd like. Notice that you can make the
  names dimensions, or something entirely different.
* The `storage` keyword is important, used for specifying the bucket for the
  field. If you don't specify `storage`, the default backend is used. As a
  shortcut, you could set `S3BotoStorage_AllPublic` as your default backend,
  and the `AWS_*` values would determine the default bucket.

Backends
^^^^^^^^

django-athumb comes with a simplified s3boto backend, modified from those found
in the django-storages project. For most cases, you'll want to use
``athumb.backends.s3boto.S3BotoStorage_AllPublic``, as it does not use HTTPS, and
is a good bit faster than ``S3BotoStorage`` because it makes some assumptions.

.. note:: This module is primarily aimed at storing and serving images to/from
    S3. I have not tested it at all with the standard Django Filesystem backend,
    though it *should* work.

Template Tags
-------------

When referring to media in HTML templates you can use custom template tags.
These tags can by accessed by loading the athumb template tag collection.

    {% load thumbnail %}

If you'd like to make the athumb tags global, you can add the following to
your master urls.py file:

    from django.template import add_to_builtins
    add_to_builtins('athumb.templatetags.thumbnail')

Some backends (S3) support https URLs when the requesting page is secure.
In order for the https to be detected, the request must be placed in the
template context with the key ``'request'``. This can be done automatically by adding
``'django.core.context_processors.request'`` to ``TEMPLATE_CONTEXT_PROCESSORS``
in settings.py

thumbnail
^^^^^^^^^

Returns the URL for the specified thumbnail size (as per the object's
models.py Model class)::

    {% thumbnail some_obj.image '50x50_cropped' %}

or, to save the value in a template context variable::

    {% thumbnail some_obj.image 'front_page' as 'some_var' %}

As long as you've got Django's request context processor in, the thumbnail tag
will detect when the current view is being served over SSL, and automatically
convert any http to https in the thumbnail URL. If you want to always force
SSL for a thumbnail, add it as an argument like this::

    {% thumbnail some_obj.image '60x60' force_ssl=True %}

To put the thumbnail URL on the context instead of just rendering
it, finish the tag with `as [context_var_name]`::

    {% thumbnail image '60x60' as 'thumb' %}
    <img src="{{ thumb }}" />


manage.py commands
------------------

athumb_regen_field
^^^^^^^^^^^^^^^^^^

    # ./manage.py athumb_regen_field <app.model> <field>

Re-generates thumbnails for all instances of the given model, for the given
field.


To-Do
-----

* See the issue tracker for a list of outstanding things needing doing.


Change Log
----------

2.4.1
=====

* Fixes to restore compatibility with migrations. (jneves)

2.4
===

* Changes to add Django 1.7 compatibility.

2.3
===

* Embed pial, eliminating the need to install it separately.

2.2
===

* Added support for different S3 regions via the AWS_REGION setting.
* Improved error handling.

2.1
===

* Make MEDIA_CACHE_BUSTER optional.
* Documented MEDIA_CACHE_BUSTER.

2.0
===

* Complete re-work of the way thumbnails are specified in models.py.
* Removal of the attribute-based image field size retrieval, since we no
  longer are just limited to dimensions.
* Further misc. improvements.

1.0
===

* Initial release.
