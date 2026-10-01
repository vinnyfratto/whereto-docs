# Zoho Desk Help Center skin

The source for the custom look of support.wheretotrips.com. Zoho holds the live
copy; these files are the record, so edit here first and paste into Zoho.

Where it goes: Desk **Setup > Help Center > WhereTo Trips > Customization >
Themes > Elegant > Customize**.

| File | Zoho tab |
|---|---|
| `header.html` | HTML > Header |
| `footer.html` | HTML > Footer |
| `custom.css` | CSS |

Then **Save draft**, check it with **Preview**, and **Publish**.

Rules:
- Keep Zoho's `${...}` placeholders in the header (`${Home}`, `${MyRequests}`,
  `${KnowledgeBase}`, `${Community}`, `${SignInSignOut}`, `${UserPreference}`,
  `${Search}`, `${SearchHome}`). They render the portal's own nav and sign-in
  state. Keep the `navBar`, `menuIconContainer` and `menuBox` ids too: the
  phone menu script looks for them.
- Links to wheretotrips.com open in the same tab, so people can move between the
  site and the help center in one tab. The website's "Help Center" nav item does
  the same.
- Only the header and footer take HTML. Everything between them is Zoho's
  markup, styled from `custom.css` through its `Block__element` class names.
- Icons are the app's Solar bold-duotone set, applied as CSS masks over Zoho's
  sprite icons. Don't hand-edit the `ICONS:START`/`ICONS:END` block; change
  `build-icons.js` and run `node docs/operations/zoho-helpcenter/build-icons.js`.
  Knowledge Base category icons are matched by category name, so renaming or
  adding a category in Desk needs a new entry in `CATEGORIES` there.
- The CSS is too big to type into Zoho's editor comfortably: copy the whole
  file, click in the CSS editor, select all, paste.
- Fonts and logos load from wheretotrips.com (GitHub Pages sends
  `Access-Control-Allow-Origin: *`, so the font files work cross-origin).
