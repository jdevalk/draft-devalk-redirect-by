# The Redirect-By HTTP Response Header Field

This is the working area for the individual Internet-Draft, "The Redirect-By HTTP Response Header Field".

- [Editor's Copy](https://jdevalk.github.io/draft-devalk-redirect-by/#go.draft-devalk-redirect-by.html)
- [Datatracker Page](https://datatracker.ietf.org/doc/draft-devalk-redirect-by) (after the first submission)
- [Individual Draft](https://datatracker.ietf.org/doc/html/draft-devalk-redirect-by)
- [Compare Editor's Copy to Individual Draft](https://jdevalk.github.io/draft-devalk-redirect-by/#go.draft-devalk-redirect-by.diff)

## Implementations

The field is already deployed at scale under the legacy name `X-Redirect-By`.
This table tracks who emits it, and the migration to the unprefixed `Redirect-By`
name this draft registers.

| Software                | Emits today     | `Redirect-By` status               | Source |
| ----------------------- | --------------- | ---------------------------------- | ------ |
| **WordPress** core      | `X-Redirect-By` (since 5.1) | Patch prepared — emit both in `wp_redirect()` | [`wp_redirect()`](https://github.com/WordPress/wordpress-develop/blob/trunk/src/wp-includes/pluggable.php) · [#42313](https://core.trac.wordpress.org/ticket/42313) |
| **ClassicPress**        | `X-Redirect-By` (via `wp_redirect()`) | Patch prepared — mirrors the WordPress change | [`pluggable.php`](https://github.com/ClassicPress/ClassicPress/blob/develop/src/wp-includes/pluggable.php) |
| **TYPO3**               | `X-Redirect-By` (redirect + shortcut middleware) | Change prepared — emit both | [RedirectHandler](https://github.com/TYPO3/typo3/blob/main/typo3/sysext/redirects/Classes/Http/Middleware/RedirectHandler.php) · [ShortcutAndMountPointRedirect](https://github.com/TYPO3/typo3/blob/main/typo3/sysext/frontend/Classes/Middleware/ShortcutAndMountPointRedirect.php) |
| **Moodle**              | `X-Redirect-By` (value `Moodle`; adds file:line under `DEBUG_DEVELOPER`) | Candidate | [`weblib.php` `redirect()`](https://github.com/moodle/moodle/blob/main/public/lib/weblib.php#L2240) |
| **Redirection** (WP plugin) | `X-Redirect-By: redirection`; its redirect-checker UI also *reads* the header | Patch prepared — emit both (checker-UI read is a noted follow-up) | [`actions/error.php`](https://github.com/johngodley/redirection/blob/trunk/actions/error.php) |
| **The SEO Framework** (WP plugin) | redirects, but no attribution value set | Patch prepared — pass `'The SEO Framework'` to `wp_safe_redirect()` | [`redirect.class.php`](https://github.com/sybrew/the-seo-framework/blob/master/inc/classes/front/redirect.class.php) |
| **Yoast SEO** (WP plugin) | `X-Redirect-By` (e.g. `Yoast SEO Premium`) | Candidate | — |
| **Winter CMS** (redirect plugin) | `X-Redirect-By: Winter.Redirect` | Patch prepared — emit both | [`RedirectManager.php`](https://github.com/wintercms/wn-redirect-plugin/blob/main/classes/RedirectManager.php#L181) |
| **Symfony** HttpFoundation | none          | Candidate — proposed as opt-in `RedirectResponse` support | [`RedirectResponse`](https://github.com/symfony/symfony/blob/8.2/src/Symfony/Component/HttpFoundation/RedirectResponse.php) |
| **Drupal**              | none (builds on Symfony HttpFoundation) | Candidate | — |
| **specification.website** | `Redirect-By`  | ✅ Shipping (reference implementation) | [`_middleware.ts`](https://github.com/jdevalk/specification.website/blob/main/functions/_middleware.ts) |

Contributions to move an implementation onto `Redirect-By` (keeping
`X-Redirect-By` during the transition) are welcome — open an issue or PR here to
update this table.

## Contributing

See the [guidelines for contributions](CONTRIBUTING.md).

Contributions can be made by creating pull requests. The GitHub interface supports creating pull requests using the Edit (✏) button.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

This repository is set up with [`i-d-template`](https://github.com/martinthomson/i-d-template),
carried as the `lib/` git submodule. After cloning, initialise it with:

```sh
$ git submodule update --init
```
