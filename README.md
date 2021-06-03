# wagtail-faq

* [Wagtail Templates](#wagtail-templates)
* [Improve Wagtail Behavior](#improve-wagtail-behaviour)
* [Wagtail Admin Customization](#wagtail-admin-customization)
* [Rich Text Editor](#rich-text-editor)
* [Pages](#pages)
* [Images](#images)
* [Documents](#documents)
* [Sorting](#sorting)
* [Searching](#searching)
* [Wagtail API](#wagtail-api)
* [ModelAdmin](#modeladmin)
* [Various Questions](#various-questions)


Wagtail Templates
-----------------

### How to properly get the URL of a page

Use `{% pageurl page %}` where `page` must be a proper Wagtail page (or else an exception will be thrown) or `{% slugurl slug %}` where `slug` is a string; `slugurl` will return `None` if the slug is not found. Also, please be extra careful with this because if multiple pages exist with the same slug, the page chosen is undetermined!

### Can I retrieve the type of a page in my templates?

There are various ways to do that but the simplest one seems to be using the content type of that page. Something like this: `{{ page.content_type.model }}`. You could also use `{{ page.content_type.app_label }}` to also retrieve the app label of that page. Finally, if you want a friendly representation, you can use `{{ page.get_verbose_name }}`.

### How to display breadcrumbs for my pages?

You can create a template snippet like this one:

```html
{% load wagtailcore_tags %}

{% if page.get_ancestors|length > 1 %}
<ul class="breadcrumb">
    {% for ancestor_page in page.get_ancestors %}
        {% if not ancestor_page.is_root %}
            {% if ancestor_page.depth > 2 %}
                <li class="breadcrumb-item"><a href="{% pageurl ancestor_page %}" title="{{ ancestor_page.title }}">{{ ancestor_page.title|truncatewords:4 }}</a></li>
            {% endif %}
        {% endif %}
    {% endfor %}
    <li class="breadcrumb-item">{{ page.title|truncatewords:4 }}</li>
</ul>
{% endif %}
```

Use the `{% include %}` tag to include that template wherever you wish to display the breadcrumbs. Please notice that I display only pages that have a `depth > 2` because of how my Wagtail site works; just use the proper depth for your own case. Also, because some pages may have long titles, I'm using `truncatewords` to properly cut-off long titles.

Improve Wagtail Behaviour
-------------------------

### Wagtail throws a server error when an image/document/other thing that is used in a Page using `PROTECTED` Foreign Key

Here's the relevant issue: https://github.com/wagtail/wagtail/issues/1602. Since this is very difficult to fix in Wagtail, just add the following middleware to your list of middleware classes to display a proper error message instead of the 500 server error:

```python
class HandleProtectionErrorMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def process_exception(self, request, exception):
        from django.db.models.deletion import ProtectedError
        from django.contrib import messages
        from django.http import HttpResponseRedirect

        if isinstance(exception, ProtectedError):
            messages.error(
                request,
                "The object you are trying to delete is used somewhere. Please remove any usages and try again!.",
            )
            return HttpResponseRedirect(request.path)

        return None

    def __call__(self, request):
        response = self.get_response(request)
        return response
```        


### I want my slugs to be properly transliterated to ASCII so instead of `/δοκιμή/` I want to see `/dokime/` as a slug

Try this code:

```python
from django.utils.html import format_html
from wagtail.core import hooks


@hooks.register('insert_editor_js')
def editor_js():
    return  r"""<script>

function cleanForSlug(val, useURLify) {
    let cleaned = URLify(val, 255);
    if (cleaned) {
        return cleaned;
    }
    return '';
}

    </script>"""
``` 

*Since Wagtail 2.12* you can also use this must simpler code for your hook:

```
@hooks.register('insert_editor_js')
def editor_js():
return  r"""<script>
window.unicodeSlugsEnabled = false
    </script>"""
```



Wagtail Admin customization
---------------------------

### I want to add some custom css to my Wagtail admin to fix things that are displayed broken

You can use `insert_global_admin_css`. For example, try something like this:

```python
@hooks.register("insert_global_admin_css", order=100)
def global_admin_css():
    return """
        <style>
            .condensed-inline-panel__card-header>h2 {
                font-size: 0.8em;
                white-space: normal !important;
            }
        </style>
        """
```

### How could I run custom clean checks when saving a Page through the Wagtail Admin?

You can add a custom wagtail form that inherits from `WagtailAdminPageForm` and overrides its `clean()` method. Something like this:

```python
from wagtail.admin.forms.pages import WagtailAdminPageForm

class CustomPageForm(WagtailAdminPageForm):
    def clean(self):
        cleaned_data = super().clean()
        if cleaned_data['value'] == 42:
	    self.add_error(
		"value", "Value cannot be 42!"
	    )

        return cleaned_data
```

Then use `CustomPageForm` for your page's form using the `base_form_class` attribute, i.e:

```python
class CustomPage(Page):
    base_form_class = IntPageForm
```

### How could I validate the data of inline panels?

If your page has a `Parental` relation with a model and render the field through an `InlinePanel` (check this for more info https://docs.wagtail.io/en/v2.0/reference/pages/panels.html#inline-panels) then your form will render the inline panel as a formset. To validate a field in that formset you can use the following clean method in your custom form:

```python
    def clean(self):
        cleaned_data = super().clean()
	# You can run checks for the main page here
        for form in self.formsets["custompage_related_documents"].forms: # this is the same as the related_name parameter of your ParentalKey
            if form.is_valid():
                cleaned_form_data = form.clean()
                doc = cleaned_form_data["document"]
                if doc and doc.collection.name != "INTERNAL DOCUMENTS":
                    form.add_error(
                        "document", "Only internal documents are allowed!"
                    )

        return cleaned_data
```

Rich Text Editor
----------------

### How can I add anchor links? 

Use this: https://github.com/thibaudcolas/wagtail_draftail_experiments/tree/master/wagtail_draftail_anchors


### How can I open external links to a new window?

Something like this should work:

```python
from django.utils.html import escape
from wagtail.core import hooks
from wagtail.core.rich_text import LinkHandler

class NewWindowExternalLinkHandler(LinkHandler):
    # This specifies to do this override for external links only.
    # Other identifiers are available for other types of links.
    identifier = "external"

    @classmethod
    def expand_db_attributes(cls, attrs):
        href = attrs["href"]
        # Let's add the target attr, and also rel="noopener" + noreferrer fallback.
        # See https://github.com/whatwg/html/issues/4078.
        return '<a href="%s" target="_blank" rel="noopener noreferrer">' % escape(href)


@hooks.register("register_rich_text_features")
def register_external_link(features):
    features.register_link_type(NewWindowExternalLinkHandler)
```

### How can I open document links to an external window?

This is useful for PDF files because chrome opens them in the same window and users are complaining that are navigated away from the site! Just use the following snippet:

```python
from wagtail.documents.rich_text import DocumentLinkHandler
from django.core.exceptions import ObjectDoesNotExist


class CustomDocumentLinkHandler(DocumentLinkHandler):
    @classmethod
    def expand_db_attributes(cls, attrs):
        try:
            doc = cls.get_instance(attrs)
            return '<a target="_blank" href="%s">' % escape(doc.url)
        except (ObjectDoesNotExist, KeyError):
            return "<a>"


@hooks.register("register_rich_text_features")
def register_link_handler(features):

    features.register_link_type(CustomDocumentLinkHandler)
```

**Important:** Please notice that if the application which registers this snippet is *before* the `wagtail.documents` application in your `settings.INSTALLED_APPS` the `wagtail.documents` will override your handler with the default one. So you need to put your app *below* `wagtail.documents` so your own handler is used instead.


### How to show-hide icons in the Wagtail rich-text editor and change their ordering
```python
WAGTAILADMIN_RICH_TEXT_EDITORS = {
    "default": {
        "WIDGET": "wagtail.admin.rich_text.DraftailRichTextArea",
        "OPTIONS": {
            "features": [
                "bold",
                "italic",
                "h3",
                "h4",
                "ol",
                "ul",
                "link",
                "document-link",
                "image",
                # "anchor",
                "embed",
            ]
        },
    }
}
```



Pages
-----

### How can I create a page programatically?

One of the most common questions for Wagtail :)
Copying directly from https://stackoverflow.com/a/43041179/119071:

```python
page = SomePageType(title="My new page", body="<p>Hello world</p>")  # adjust fields to match your page type
parent_page.add_child(instance=page)
```

### How can I get the children and grand-children of a page without doing lots of queries?

The naive approach is to get the children of the page using `page.get_children()` and then get each child's children. This leads to N+1 queries which is a no-no. 

There's a better way! You can use something like:

```python
page.get_descendants().filter(depth__lte=page.depth+2)
```

The children of the `page` have a `depth==page.depth+1` and the grandchildren have a `depth==page.depth+2`.

Or change 2 to n to get the descendants of the page that are up to n levels below the page.

### How are the wagtail Pages stored in the database?

Wagtail uses a tree-like structure to store its pages. The library doing the heavy work is called django-treebeard (https://github.com/django-treebeard/django-treebeard/). It uses an intuitive approach to store the tree: Each `Page` has a path field that denotes its position in the tree and is actually a varchar. Each child in each level in the tree gets a value from `0001` to `ZZZZ` (i.e after 9 the letters A-Z are counted) and for each level a new quadruple is added. So root trees will have a path like `0001` or `BCD3`. First level pages will have paths like 
`00012233`, `00010001` or `BCD30002` (notice that the first two are children of `0001` root node while the last one is child of `BCD3` root node). 

One implication of this is that there can be a finite number of children for each node, equal to 10 numbers + 26 letters = 36 combinations ^ 4 (places) = 1679616 - 1 (because the first value is `0001` not `0000`) 1679615 children. Another implication is that because the `path` is a `varchar(255)` there can be up to 255/4 = 63 levels for each tree! Notice that this kind of path has also sorting between children built-in; you'll know that `0003322B` will be after `0003322A` (and both will be children of `0003`).

Now, for each of your models that overrides `Page` to create a specific page type (i.e `HomePage`, `StandardPage`) there will be two rows in the database in two different tables. One in the `wagtail_page` models that will have the tree related staff I mentioned above and another in the corresponding table for your page type (i.e `home_homepage`) that will have the custom fields of your page type and a foreign key (which also is used as a primary key) to the corresponding page. This is why you need to use `specific()` (https://docs.wagtail.io/en/v2.11.3/reference/pages/queryset_reference.html#wagtail.core.query.PageQuerySet.specific) when querying `Page` to retrieve the custom fields for each page of your queryset result.

Images
------

### How can I quickly add multiple related images to a page? 

Use this: https://github.com/spapas/wagtail-multi-upload

### I don't want my editors to upload small images for some pages!

Sometimes the editors don't care (or don't even know) about image sizes, and they will upload an image with a 200px width as the central photo of a new article; they may even not care when they see the big pixelized artefacts this will generate! The canonical way to fix this is to add a `Form` for your `Page`. To do this, first create a form class with a clean method like this:

```python
from wagtail.admin.forms import WagtailAdminPageForm

class CustomPageForm(WagtailAdminPageForm):
    def clean(self):
        cleaned_data = super().clean()
        image = cleaned_data.get('image')
        if bi and bi.width < 1200:
            form.add_error("image", "Error! image is too small - width must be > 1200px!")
        
        return cleaned_data
```

Then, to use this form (and its clean method) just add the following attribute to your Page model: `base_form_class = CustomPageForm`. Then when your editors submit small images they will see an error for that field!

### What if my images are in an InlinePanel ?

See the corresponding answer for [Wagtail Admin Customizing](#wagtail-admin-customization).


### Ok fine but I don't want my editors to be able to select small images!!

Continuing from the previous FAQ, you can do some acrobatics to *filter* small images from the image chooser your editors will see. This needs a lot of acrobatics though thus I'd recommend using the canonical way mentioned above. But since I researched it here goes nothing:

1. This works with Wagtail 2.11. I haven't tested it with other Wagtail versions
2. Start by putting the following `AdminImageChooserEx` class somewhere:

```python
from wagtail.images.widgets import AdminImageChooser

class AdminImageChooserEx(AdminImageChooser):
    def __init__(self, *args, **kwargs):
        min_width = kwargs.pop("min_width")
        super().__init__(**kwargs)
        self.min_width = min_width
        self.image_model = get_image_model()

    def render_html(self, name, value, attrs):
        instance, value = self.get_instance_and_id(self.image_model, value)
        original_field_html = super(AdminChooser, self).render_html(name, value, attrs)

        return render_to_string(
            "wagtailimages/widgets/image_chooser_ex.html",
            {
                "widget": self,
                "original_field_html": original_field_html,
                "attrs": attrs,
                "value": value,
                "image": instance,
                "min_width": getattr(self, "min_width", 10),
            },
        )

```

This class expects to be called with a `min_width` argument; it will then pass it to the context of a template named `wagtailimages/widgets/image_chooser_ex.html`. Notice the trickery with the `super(AdminChooser, self).render_html(...)` (somebody would expect either `super().render_html()` - py3 style or even `super(AdminImageChooser, self).render_html(...)` - py2 style); this line is **correct**; we need to use our *grandparent's* `render_html`, not our parent's (which is the usual thing) .

3. The `AdminImageChooserEx` class needs an `image_chooser_ex.html` template. So create a directory named `templates\wagtailimages\widgets` in your app and add the following to it

```
{% extends "wagtailimages/widgets/image_chooser.html" %}

{% block chooser_attributes %}data-chooser-url="{% url "wagtailimages:chooser" %}?min_width={{ min_width }}"{% endblock %}
```

It just overrides the `image_chooser.html` template to pass the min_width option to the `data-chooser-url` attribute along with the image chooser URL.

4. Use a hook to filter the images of the chooser by their width:

```python
from wagtail.core import hooks

@hooks.register("construct_image_chooser_queryset")
def show_images_with_width(images, request):
    min_width = request.GET.get("min_width")
    if min_width:
        images = images.filter(width__gte=min_width)

    return images
```

5. Add a panel that would *actually* set that width:

```python
from wagtail.images.edit_handlers import ImageChooserPanel

class ImageExChooserPanel(ImageChooserPanel):
    min_width = 2000
    object_type_name = "image"

    def widget_overrides(self):
        return {self.field_name: AdminImageChooserEx(min_width=self.min_width)}
```

The above only allows images with a width of more than 2000 pixel. You need to a *different* class for each width you need (just override `ImageExChooserPanel` setting a different `min_width`)

6. Finally *use* that `ImageExChooserPanel`  in your page: 

```python
Page.content_panels + [
	# ...
        ImageExChooserPanel("image",),
    ]
```

7. Profit!

(I guess that some people would ask why I didn't pass the `min_width` parameter to `ImageExChooserPanel` and I needed to construct a different class for each `min_width`, i.e call it like `ImageExChooserPanel("image", min_width=2000)`. Unfortunately, because of things I can't understand these parameters are *lost* and the `ImageExChooserPanel` was called without the `min_width`. So you need to set it on the class for it to work).

### Is it possible to have protected/private images?

No, it isn't. By default, all images are served directly from your HTTP server and you can't have any permissions check for them. You can have private documents though, so you could theoretically upload your private/protected images as documents.

Documents
---------

### Can I have protected/private documents?

Yes, you definitely can by adding the documents to a specific collection and allowing only particular users to view the documents of that collection. However, there are some caveats that depend on the way you have configured your wagtail document serving method *and* your media storage backend. The documentation for that is here: https://docs.wagtail.io/en/stable/reference/settings.html#documents but I'll give you some quick tips on the next two answers for how to configure things for two following scenarios when you have your documents stored locally on your server (if you're using S3 or similar things then you're on your own). 

Wagtail by default (if you save your files locally at least) will stream the files through your python workers; i.e. these processes will be tied to serving the file for as much time as the file downloading takes. If you have four workers (which is a common amount) and have four people with slow connections downloading a large file at the same time then your site *will not work anymore*! This is a huge problem, and you *need* to fix it before going to production. Just to make things even more crystal: If your Wagtail site has documents you *cannot* use the default settings!

### How can I configure my document serving if I don't have private documents?

This is easy; just use the following setting: `WAGTAILDOCS_SERVE_METHOD = direct`. This will configure wagtail so when you use `{{ document.url }}` it will output the path of your file inside your `MEDIA_URL`; so your media will be served directly from your web server just like your static files. Of course you need to have properly configured your django site and your web server to serve the media files (by serving `MEDIA_ROOT` etc).

Please notice that if you do this you can't use collections to set permissions on your docs since everything will be server through your web server.

### How can I configure my document serving if I do have private documents?

First of all, by default Wagtail uses the `WAGTAILDOCS_SERVE_METHOD = serve_view` setting. This means that when you use `{{ document.url }}` it will output the name of a view that would serve the document. This view does the permissions checks for the collection and then serves the document. What is important to do here is to *not* serve the document through your python worker but use your web server (i.e. Nginx) for that. This is a common problem in the Django world (have permissions on media files) and is solved using `django-sendfile` (https://github.com/johnsensible/django-sendfile). With a few words as possible, using this mechanism you tell Nginx to "hide" your protected documents folder and *only* serve it if he gets a proper response from your application. I.e you request the document serving view and if the permissions pass the `django-sendfile` will return a response telling Nginx to serve a file. Nginx will see that response and instead of returning it to the request it will actually return the file. If you want to learn more take a look at these two SO questions  https://stackoverflow.com/questions/7296642/django-understanding-x-sendfile and https://stackoverflow.com/questions/28166784/restricting-access-to-private-file-downloads-in-django.

Ok, now how to properly configure Wagtail for this. First of all, add the following settings to your settings (`MEDIA_*` should be there but anyway):

```python
WAGTAILDOCS_SERVE_METHOD = "serve_view" # We talked about this
SENDFILE_BACKEND = "sendfile.backends.nginx" # If you are using nginx; there is support for other web sevrers
MEDIA_URL = "/media/"
MEDIA_ROOT = "/home/serafeim/hcgwagtail/media"
SENDFILE_ROOT = "/home/serafeim/hcgwagtail/media/documents"
SENDFILE_URL = "/media/documents/"
```

The above tells Django that the docs should be server through nginx and where the documents will be. Finally, add the following two entries in your Nginx configuration:

```python
    location /media/documents/ {
        internal;
        alias /home/serafeim/hcgwagtail/hcgwagtail/media/documents/;
    }

    location /media/ {
        alias /home/serafeim/hcgwagtail/hcgwagtail/media/;
    }
```

The first one tells Nginx that the files in `/media/documents` will be served through the sendfile mechanism I described before; the second one is the common media serving directive. Notice that the first one will match first so documents won't be served directly; please make sure that this really is the case by trying to get an uploaded document directly by its URL (i.e. `/media/documents/...`).

### Can I search my documents with their id?

Each wagtail document has a unique id which is visible in its public link together with the filename, i.e each document will be served through a link like the following: 
https://example.com/documents/<document_id>/<filename>. Sometimes your editors may need to search for a document using that particular id (because they see it in the page but they haven't added a proper title so as the document to be indexed). To resolve that you can add the `id` field of the document to the search index.

If you are using a custom document model then adding the `id` field to the search index should be as trivial as something like:

```python
from wagtail.documents.models import AbstractDocument
from wagtail.search.index import SearchField	
	
search_fields = AbstractDocument.search_fields + [
	index.SearchField("id"),
]
```

However if you are *not* using a custom document model then you can resort to some monkey patching: Add the following snippet to the models.py of an application that is loaded *before* the `wagtail.documents` application (as determined by the order of the apps in the `INSTALLED_APPS` setting):
	
```python	
from wagtail.documents.models import AbstractDocument
from wagtail.search.index import SearchField

AbstractDocument.search_fields += [SearchField("id")]
```
	
The above will override the `search_fields` of the  `AbstractDocument` model that `Document` inherits from thus it will use the `id` in the search fields.
	
Don't forget to re-index your models by running `python manange.py update_index`.
	

Sorting
-------

### How can I add a default order for the pages displayed in a `PageChooserPanel`

Use something like this:
```python
from wagtail.core import hooks

@hooks.register("construct_page_chooser_queryset")
def fix_page_sorting(pages, request):
    pages = pages.order_by("-latest_revision_created_at")
    return pages
```

### How can I order a page queryset using Wagtail's admin sorting field (the one with the 6 dots)?

Use `queryset.order_by('path')`

Searching
---------

### How will searching work if I have private pages?

By default search results *will* contain all pages even if the current user doesn't have permissions to view them. So he'll see the page but when he clicks on it he'll need to login (or get a permission error if he doesn't have permission). To display only the pages that are public (i.e visible to everybody) you can use `public()` https://docs.wagtail.io/en/v2.11.1/reference/pages/queryset_reference.html#wagtail.core.query.PageQuerySet.public, for example `Page.objects.live().public().search("Hello world!")` - notice that `live()` is also needed to skip non-published pages.

### Can I display private pages in results if the user has permissions to view them?

Let's suppose you've got a Page queryset with all the results of a search. You can use the following snippet to remove pages that the current user doesn't have permission to view:

```python
from wagtail.wagtailcore.models import PageViewRestriction
def exclude_invisible_pages(request, pages):
    # Get list of pages that are restricted to this user
    restricted_pages = [
        restriction.page
        for restriction in PageViewRestriction.objects.all().select_related('page')
        if not restriction.accept_request(request)
    ]
    # Exclude the restricted pages and their descendants from the queryset
    for restricted_page in restricted_pages:
        pages = pages.not_descendant_of(restricted_page, inclusive=True)
    return pages
```

Wagtail API
-----------

### Why the Wagtail API does not return private pages?

By default the Wagtail API will display only public pages. You can change it by overriding the `get_base_queryset` method of `PagesAPIViewSet`. So, in your api.py file do something like:

```python
from wagtail.api.v2.views import PagesAPIViewSet
from wagtail.api.v2.router import WagtailAPIRouter
from wagtail.core.models import Page, Site

# Create the router. "wagtailapi" is the URL namespace
api_router = WagtailAPIRouter("wagtailapi")

class AllPagesAPIViewSet(PagesAPIViewSet):
    def get_base_queryset(self):

        # Get live pages
        queryset = Page.objects.all().live()

        # Filter by site
        site = Site.find_for_request(self.request)
        if site:
            base_queryset = queryset
            queryset = base_queryset.descendant_of(site.root_page, inclusive=True)
	                
	    # If internationalisation is enabled, include pages from other language trees
            if getattr(settings, 'WAGTAIL_I18N_ENABLED', False):
                for translation in site.root_page.get_translations():
                    queryset |= base_queryset.descendant_of(translation, inclusive=True)

        else:
            # No sites configured
            queryset = queryset.none()

        return queryset


api_router.register_endpoint("pages", AllPagesAPIViewSet)
```

Of course you can do whatever else tricks you want there to enable your API only for specific pages.



ModelAdmin
----------

### Can I display an image to one of the columns of my ModelAdmin listing?

Yes! It's actually very easy. Add a custom function to your model that returns a "safe" text containing an `<img>` element with the image you want to display as src. You should use the rendition API to return a proper thumbnail fit for your column. Something like this method for example: 

```
from django.utils.html import format_html


class ModelWithImage(models.Model):
    image = models.ForeignKey(CustomImage, on_delete=models.CASCADE)
    
    def show_image(self):
        return format_html(
            "<img style='max-width: 200px;' src='{0}'>".format(
                self.image.get_rendition("width-200|jpegquality-60").url
            )
        )
```

Now you can add this show_image method to the `list_display` attribute of your `ModelAdmin` and profit!


Various Questions
-----------------


### How can I check if a user can publish pages?

```python
    from wagtail.core.models import UserPagePermissionsProxy
    
    if not UserPagePermissionsProxy(request.user).can_publish_pages():
        messages.add_message(request, messages.ERROR, "No access!")
        return HttpResponseRedirect(reverse("wagtailadmin_home"))
```        


### Let's suppose I've added an image or a link to a rich-text field, what happens when that image or link are deleted/moved?

The correct thing: They're referenced by ID in the rich text data, so they'll continue to work after moving or renaming. If they're deleted completely, that will end up as a broken link in the text.

### What to use for syndication (RSS)?

Just use Django's syndication framework: https://docs.djangoproject.com/en/3.0/ref/contrib/syndication/

### What to use for the sitemap?

Here you go: https://docs.wagtail.io/en/v2.8/reference/contrib/sitemaps.html

### How can I display HTML to my wagtail forms? 

Use the `WAGTAILFORMS_HELP_TEXT_ALLOW_HTML` setting - see https://docs.wagtail.io/en/stable/releases/2.9.3.html
