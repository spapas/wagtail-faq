# wagtail-faq

### How can I add anchor links? 

Use this: https://github.com/thibaudcolas/wagtail_draftail_experiments/tree/master/wagtail_draftail_anchors

### How can I quickly add multiple related images to a Page? 

Use this: https://github.com/spapas/wagtail-multi-upload

### How can I add a default order for the pages displayed in a `PageChooserPanel`

Use something like this:
```
from wagtail.core import hooks

@hooks.register("construct_page_chooser_queryset")
def fix_page_sorting(pages, request):
    pages = pages.order_by("-latest_revision_created_at")
    return pages
```

### How can I open external links to a new window?

Something like this should work:

```
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

### How can I check if a user can publish pages?

```
    from wagtail.core.models import UserPagePermissionsProxy
    
    if not UserPagePermissionsProxy(request.user).can_publish_pages():
        messages.add_message(request, messages.ERROR, "No access!")
        return HttpResponseRedirect(reverse("wagtailadmin_home"))
```        


### What to use for syndication (rss)?

Just use the django syndication framework: https://docs.djangoproject.com/en/3.0/ref/contrib/syndication/

### What to use for the sitemap?

Here you go: https://docs.wagtail.io/en/v2.8/reference/contrib/sitemaps.html


### Let's suppose i've added an image or a link to a richtext field. what happens when that image or link are deleted/moved ?

The correct thing: They're referenced by ID in the rich text data, so they'll continue to work after moving or renaming. If they're deleted completely, that will end up as a broken link in the text.

### Wagtail throws a server error when an image/document/other thing that is used in a Page using `PROTECTED` Foreign Key

Here's the relevant issue: https://github.com/wagtail/wagtail/issues/1602. Since this is very difficult to fix in wagtail, just add the following middleware to your list of middleware classes to display a proper error message instead of the 500 server error:

```

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

### How can I order a page queryset using the wagtal-admin sorting field (the one with the 6 dots)?

Use `queryset.order_by('path')`


### I want my slugs to be properly transliterated to ASCII so instead of `/δοκιμή/` I want to see `/dokime/` as a slug.

Try this code:

```
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


### I want to add some custom css to my wagtail admin to fix things that are are displayed broken

You can try something like this:

```
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

### How to show-hide icons in the wagtail richtext editor and change their ordering
```
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

### Can I retrieve the type of  a page in my templates?

There are various ways to do that but the simplest one seems to be using the content type of that page. Something like this: `{{ page.content_type.model }}`. You could also use `{{ page.content_type.app_label }}` to also retrieve the app label of that page. Finally, if you want a friendly representation you can use `{{ page.get_verbose_name }}`.

### I don't want my editors to upload small images for some Pages!

Sometimes the editors don't care (or don't even know) about image sizes and they will upload an image with a 200px width as the central photo of a new article; they may even not care when they see the big pixelized artifacts this will generate! The canonical way to fix this is to add a `Form` for your `Page`. To do this, first create a form class with a clean method like this:

```
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


### Ok fine but I don't want my editors to be able to select small images!!

Continuing from the previous FAQ, you can do some acrobatics to *filter* small images from the image chooser your editors will see. This needs a lot of acrobatics though thus I'd recommend to use the canonical way mentioned above. But since I researched it here goes nothing:

1. This works with Wagtai 2.11. I haven't tested it with other Wagtail versions
2. Start by putting the following `AdminImageChooserEx` class somewhere:

```
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

This class expects to be called with a `min_width` argument; it will then pass it to the context of a templated named `wagtailimages/widgets/image_chooser_ex.html` renders. Notice the trickery with the `super(AdminChooser, self).render_html(...)` (somebody would expect either `super().render_html()` - py3 style or even `super(AdminImageChooser, self).render_html(...)` - py2 style); this line is correct.

3. The `AdminImageChooserEx` class needs an `image_chooser_ex.html` template. So create a directory named `templates\wagtailimages\widgets` in your app and add the following to it

```
{% extends "wagtailimages/widgets/image_chooser.html" %}

{% block chooser_attributes %}data-chooser-url="{% url "wagtailimages:chooser" %}?min_width={{ min_width }}"{% endblock %}
```

It just overrides the `image_chooser.html` template to pass the min_width option to the `data-chooser-url` attribute along with the image chooser url.

4. Use a hook to filter the images of the chooser by their width:

```
from wagtail.core import hooks

@hooks.register("construct_image_chooser_queryset")
def show_images_with_width(images, request):
    min_width = request.GET.get("min_width")
    if min_width:
        images = images.filter(width__gte=min_width)

    return images
```

5. Add a panel that would *actually* set that width:

```
from wagtail.images.edit_handlers import ImageChooserPanel

class ImageExChooserPanel(ImageChooserPanel):
    min_width = 2000
    object_type_name = "image"

    def widget_overrides(self):
        return {self.field_name: AdminImageChooserEx(min_width=self.min_width)}
```

The above only allows images with a width of more than 2000 pixel. You need to a *different* class for each width you need (just override `ImageExChooserPanel` setting a different `min_width`)

6. Finally *use* that `ImageExChooserPanel`  in your Page: 

```
Page.content_panels + [
	# ...
        ImageExChooserPanel("image",),
    ]
```

7. Profit!

(I guess that some people would ask why I didn't pass the `min_width` parameter to `ImageExChooserPanel` and I needed to construct a different class for each `min_width`, i.e call it like `ImageExChooserPanel("image", min_width=2000)`. Unfortuantely, because of things I can't understand these parameters are *lost* and the `ImageExChooserPanel` was called without the `min_width`. So you need to set it on the class for it to work).
