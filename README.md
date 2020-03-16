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

