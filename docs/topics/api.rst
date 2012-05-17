.. _api:

======================
Marketplace API
======================

Before you go any further *please note the following*:

* This is currently a document in progress, prior to the API's being written so
  don't expect this to work. Formal API documentation will go on to `MDN`_ when
  it's ready.
* There is a seperate AMO set of APIs. You can find documentation of those on
  `AMO api on MDN`_.

Overall notes
-------------

Authentication
==============

Currently only two legged OAuth authentication is supported. This is focused on
clients who would like to create multiple apps on the app store from an end
point.

To get started you will need to get an OAuth token created in the site for you.
For more information on creating an OAuth token, contact the `marketplace
team`_, letting them know which Marketplace user account you would like to use
for authentication. Changing this later will give problems accessing old data.

Once you've got your token, you will need to ensure that the OAuth token is
sent correctly in each request.

*TODO*: insert example and more notes on OAuth.

Errors
======

Marketplace will return errors as JSON with the appropriate status code.

Data errors
-----------

If there is an error in your data, a 400 status code will be returned. There
can be multiple errors per field. Example::

        {
          "error": {
            "manifest": ["This field is required."]
          }
        }

Other errors
------------

The appropriate HTTP status code will be returned. The body may contain JSON
and it may contain a useful error message. Example::

        {
          "error": "Invalid OAuthToken."
        }

Verbs
=====

This follows the order of the `django-piston`_ REST verbs, a PUT for an update and POST for create.

Response
========

All responses are in JSON.

Adding an app to the Marketplace
--------------------------------

Adding an app follows a few steps, roughly analgous to the flow in a browser.
Basically, validate your app, add it in, update it with the various data and
then request review.

Validate
--------

To validate an app::

        POST /api/apps/validate

Body data should contain the manifest in JSON::

        {"manifest": "http://test.app.com/manifest"}

Validations are done async on the marketplace. The call will return immediately
with a status of 202::

        {"id": "123"}

To see how it's doing, poll for a result::

        GET /api/apps/validation/123

This will return the status of the validation. Validation not processed yet::

        {"processed": false}

Validation processed and good::

        {"valid": true, "processed": true}

Validation processed and an error::

        {"valid": false,
         "processed": true,
         "validation": {
           "errors": 1, "messages": [{
             "tier": 1,
             "message": "Your manifest must be served with the HTTP header \"Content-Type: application/x-web-app-manifest+json\". We saw \"text/html; charset=utf-8\".",
             "type": "error"
           }],
         "success": false}}

Create
------

This requires a successfully validated manifest. To create an app with your
validated manifest::

        POST /api/apps/create

Body data should contain the manifest id from the validate call and other data
in JSON::

        {"manifest_id": "123"}

If the creation succeeded you'll get a 201 status back. This will return the id
of the app on the marketplace as a slug. The marketplace will complete some of
the data using ::

        {"slug": "your-test-app",
         "status": ...}

All future calls for the app should use the slug.

Fields:

* manifest_id (required): the id of the manifest returned from verfication.

All other fields are detailed in update.

Update
------

Updates an app::

        PUT /api/apps/<slug>

The body contains JSON for the data to be posted.

These are the fields for the creation and update of an app. These will be
populated from the manifest if specified in the manifest. Will return a 200
status if the app was successfully updated.

Fields:

* name (required): the title of the app. Maximum length 127 characters.
* summary (required): the summary of the app. Maximum length 255 characters.
* categories (required): a list of the categories, at least two of:
  'entertainment', 'finance', 'games', 'music', 'news', 'productivity',
  'social networking', 'travel'.
* description (optional): long description. Some HTML supported.
* privacy_policy (required): your privacy policy. Some HTML supported.
* homepage (optional): a URL to your apps homepage.
* support_url (optional): a URL to your support homepage.
* support_email (required): the email address for support.
* device_types (required): a list of the device types at least one of:
  'desktop', 'phone', 'tablet'.
* payment_type (required): only choice at this time is 'free'.

*TODO*: should screenshot re-ordering be added here.

Status
------

To view details of an app, including its review status::

        GET /api/apps/<slug>

Returns the status of the app::

        {"slug": "your-test-app",
         "name": "My cool app",
         "screenshots": [1 , 2, 3]
         ...}

Delete
------

Deletes an app::

        DELETE /api/apps/<slug>

The app will only be hard deleted if it is incomplete. Otherwise it will be
soft deleted. A soft deleted app will not appear publicly in any listings
pages, but it will remain so that receipts, purchasing and other components
work.

Screenshots or video
--------------------

These can be added as seperate API calls. There are limits in the marketplace
for what screenshots and videos can be accepted.

Create
------

Create a screenshot or video::

        PUT /api/apps/<slug>/screenshot

The body should contain the screenshot or video to be uploaded.

This will return a 201 if the screenshot or video is successfully created. If
not we'll return the reason for the error.

Returns the screenshot id::

        {"id": "12"}

Update
------

Update a screenshot or video::

        POST /api/apps/<slug>/screenshot/<id>

This will return a 200 if the screenshot or video is succesfully updated.

Delete
------

Delete a screenshot of video::

        DELETE /api/apps/<slug>/screenshot/<id>

This will return a 200 if the screenshot has been deleted.


.. _`MDN`: https://developer.mozilla.org
.. _`marketplace team`: marketplace-team@mozilla.org
.. _`django-piston`: https://bitbucket.org/jespern/django-piston/wiki/Documentation
.. _`AMO api on MDN`: https://developer.mozilla.org/en/addons.mozilla.org_%28AMO%29_API_Developers%27_Guide