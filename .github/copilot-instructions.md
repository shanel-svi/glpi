## Deployment environment (this instance)

This is GLPI 11.0.6 running locally on macOS (Apple M3 Pro, ARM64) via Docker Compose.

**URLs**
- App: `https://itsm.shanel.com` (port 443) — also `http://itsm.shanel.com:8080` for test env
- Mailpit (catch-all SMTP): `http://itsm.shanel.com:8025`
- DbGate (DB browser): `http://itsm.shanel.com:9000`
- Webpack dev server: `http://itsm.shanel.com:9637`

**Key paths**
- Repo root: `/Users/shanelwijeratne/Documents/GLPI/glpi/glpi/`
- Custom ARM64 Dockerfile: `.docker/app/Dockerfile` (built from `php:8.4-apache`, not the upstream `ghcr.io` image which is x86_64 only)
- Apache/PHP config sources: `.docker/app/files/`
- TLS cert (mkcert, trusted by macOS Keychain): `.docker/certs/itsm.shanel.com.pem`
- PHP error log (host-mounted): `logs/php/php-error.log`
- GLPI internal log (plugin errors, SQL, warnings): `files/_log/php-errors.log`
- Xdebug log: `logs/php/xdebug.log`

**Daily workflow commands** (always run from repo root)
```bash
gmake up          # start all containers
gmake down        # stop containers (data preserved)
gmake build       # rebuild app image after Dockerfile/.ini changes
gmake cc          # clear GLPI cache
gmake bash        # shell into app container
gmake install     # full from-scratch setup (nukes and reinstalls DB)
```

**Docker Compose override** (`docker-compose.override.yaml`) disables `openldap` (ARM64-incompatible) via a profile, adds Xdebug env vars, mounts certs + log volumes, and binds all ports to `127.0.0.2`. Do not delete it.

## Git workflow

**All local changes must stay on the `local/infra-setup` branch.** Never commit directly to `11.0/bugfixes` or any upstream branch.

**Branch layout**
- `11.0/bugfixes` — tracks upstream GLPI. Pull upstream here only, no local commits.
- `local/infra-setup` — all infrastructure, config, and plugin customisations. Rebases onto `11.0/bugfixes` to pick up upstream changes.

**Updating from upstream**
```bash
git checkout 11.0/bugfixes
git pull
git checkout local/infra-setup
git rebase 11.0/bugfixes   # replay local commits on top of upstream; do NOT use merge
```

**Making changes**
1. Ensure you are on `local/infra-setup` before editing any file:
   ```bash
   git branch   # must show * local/infra-setup
   ```
2. Commit to `local/infra-setup` only.

**Files NOT in git** (must be recreated manually if lost):
- `docker-compose.override.yaml` — see "Docker Compose override" section above for full content
- `.docker/certs/` — regenerate with `mkcert itsm.shanel.com` from that directory

**Database** — MariaDB 11.8, credentials `glpi`/`glpi`, database `glpi`. Direct query:
```bash
docker compose exec db mariadb -uglpi -pglpi glpi -e "SELECT ..."
```
After any direct DB change run `gmake cc`.

**Xdebug** — active in `debug,develop` mode on port 9003. VS Code launch config in `.vscode/launch.json` ("Listen for Xdebug (GLPI Docker)"). Path mapping: `/var/www/glpi` → repo root.

**Installed plugins** — `oauthimap` (marketplace/oauthimap). OAuth/SMTP configured for Microsoft Entra (Azure), SMTP via `smtp.office365.com:587`.

## Plugin system

**Installation** — Plugins are installed via the GLPI Marketplace UI (Setup → Marketplace), which downloads them into `marketplace/<pluginname>/`. Activating a plugin from the UI calls `plugin_<name>_install()` in `hook.php`, which runs `Migration` against each `inc/*.class.php` that has a static `install()` method to create/alter DB tables.

**Filesystem layout** (see `marketplace/oauthimap/` as the reference):
```
marketplace/<pluginname>/
  setup.php          # Required: defines version constants, plugin_init_<name>(), plugin_version_<name>()
  hook.php           # Required: plugin_<name>_install(), plugin_<name>_uninstall()
  inc/               # Legacy class files — PluginNameClass pattern, loaded via autoload
  front/             # Legacy page scripts (avoid in new plugins; use controllers)
  ajax/              # AJAX endpoints
  templates/         # Twig templates
  locales/           # .po/.mo translation files
  vendor/            # Plugin-specific Composer dependencies
```

**Bootstrap sequence** — On every request GLPI calls:
1. `plugin_<name>_boot()` — stateless early init (e.g. registering stateless paths with `SessionManager`)
2. `plugin_init_<name>()` — registers hooks in the global `$PLUGIN_HOOKS` array (only runs when session is active)

**Hook registration** — All integration points are registered in `plugin_init_<name>()` via `$PLUGIN_HOOKS`:
```php
$PLUGIN_HOOKS['csrf_compliant']['myplugin'] = true;
$PLUGIN_HOOKS['config_page']['myplugin']    = 'front/config.php';
$PLUGIN_HOOKS['menu_toadd']['myplugin']     = ['config' => 'PluginMypluginFoo'];
$PLUGIN_HOOKS['post_item_form']['myplugin'] = [PluginMypluginHook::class, 'method'];
$PLUGIN_HOOKS['item_add']['myplugin']       = ['TargetClass' => [MyClass::class, 'method']];
// Other common hooks: pre_item_update, item_update, item_delete, secured_fields,
// mail_server_protocols, display_login, add_javascript, add_css
```
Guard all hook registration with `if (Plugin::isPluginActive('myplugin'))`.

**Class naming** — Legacy plugin classes follow `PluginNameClass` (e.g. `PluginOauthimapApplication`). New plugin code should use PSR-4 namespaces under `GlpiPlugin\Name\` (e.g. `GlpiPlugin\Oauthimap\MailCollectorFeature`) with Composer autoloading.

**DB migrations** — Use GLPI's `Migration` class inside `install()`/`uninstall()` static methods on each model class. Never write raw `CREATE TABLE` calls directly; use `$migration->addTable()`, `$migration->addField()`, `$migration->addKey()`, then call `$migration->executeMigration()` once in `hook.php`.

**Architecture overview**
GLPI 11.0 uses a Symfony kernel (`src/Glpi/Kernel.php`) with controllers in `src/Glpi/Controller/`. Legacy pages in `front/` are served via `LegacyFileLoadController`. New features must use controllers + Twig templates (`templates/`), not `front/` files. ORM is in `src/Glpi/DBAL/`. Assets/JS built via webpack (config in `webpack.config.js`), output to `public/build/`.

---
Follow GLPI’s latest coding standards and best practices: naming, indentation, comments, and PER Coding Style 3.0 compliance.
Use the GLPI framework whenever possible.
Use the snake_case variable naming convention.
Do not use deprecated code.
Do not use PHP features older than version 8.2.
Do not use GLPI code or APIs older than version 11.0.
Never create .md or .txt files to explain changes.
Never explain what you did.
Do not add unnecessary comments or TODO notes.
Follow the MVC pattern, routing, and controllers wherever possible.
Do not create /front/ files — always use controllers and routes if possible.
Never output raw HTML with echo; always use Twig templates.
Never execute raw SQL — always use GLPI’s ORM and database abstraction layer.
Do not ask clarification questions, except when a real choice between two technical solutions must be made.
Do not generate tests unless requested.
When generating code, always ensure it is secure and free from vulnerabilities.
When importing libraries or packages, prefer already imported ones; if using new ones, they must be compatible with GLPI GPLv3+ License.

## End-to-end tests (Playwright)

Before writing or modifying any e2e test, read existing tests in `tests/e2e/specs/` and page objects in `tests/e2e/pages/` to understand established patterns and conventions.

Update these instructions if you learn new information.

### Page Object Model

Always use the Page Object Model pattern. Locators and reusable interactions must live in page classes under `tests/e2e/pages/`, not in spec files.

- Every page class extends `GlpiPage` (defined in `tests/e2e/pages/GlpiPage.ts`), which provides helper methods such as `getButton()`, `getLink()`, `getCheckbox()`, `getTextbox()`, `getTab()`, `getRegion()`, `getHeading()`, `getRadio()`, `getSpinButton()`, `getDropdownByLabel()`, `getRichTextByLabel()`, and common actions like `doSetDropdownValue()`, `doAddNote()`, `doLogout()`, etc.
- Declare locators as public readonly properties initialized in the constructor.
- Expose user-facing actions as `async do*()` methods on the page object.
- Spec files should only contain test logic; all element access goes through page objects.

### Selectors

Use Playwright's semantic locators exclusively — prefer `getByRole()`, `getByLabel()`, `getByTestId()`, `getByText()`, `getByTitle()`, `getByAltText()`, and the helper methods inherited from `GlpiPage`.

- **Never use raw CSS or XPath selectors** (`page.locator(...)`) unless there is absolutely no semantic alternative. When a raw locator is unavoidable, add `// eslint-disable-next-line playwright/no-raw-locators` on the line above.
- Before adding a raw locator, first check if the element can be targeted using `data-testid`, `aria-label`, `title`, `alt`, or `role` attributes. When no semantic attribute exists, add a `data-testid` attribute to the element in the Twig template or source code.
- If a button or interactive element cannot be located by its accessible name (e.g., because an icon replaces the text), **add an `aria-label` attribute** to the element in the Twig template or source code so it can be targeted with `getByRole()`.
- Acceptable uses of raw locators (with eslint-disable):
  - Third-party library elements (Select2, Cytoscape, fileupload) that cannot receive `data-testid` attributes.
  - Hidden `<input>` elements used only for value verification (e.g., `input[name="field_options[...]"]`).
  - XPath ancestor traversals needed to find a parent container from a semantic child locator.
- Toast notification links: use `page.getByRole('alert').getByRole('link')` instead of raw `div.toast-container .toast-body a`.
- For file uploads, use `GlpiPage.doAddFileToUploadArea(file, parent)` instead of manually locating `input[type="file"]`.
- For file upload removal buttons, use `getByTitle('Delete')` (the jQuery fileupload plugin adds `title="Delete"`).

### Tab navigation

Use the `forcetab` URL parameter to navigate directly to a specific tab instead of clicking tab elements:

```typescript
await page.goto(`/front/computer.form.php?id=${id}&forcetab=ItemVirtualMachine$1`);
```

To find forcetab IDs, check `getTabNameForItem()` and `defineTabs()` in the relevant PHP class. The default form tab uses `ClassName$main`; numbered tabs follow the keys defined in `getTabNameForItem()` (e.g., `User$1`, `Glpi\Asset\AssetDefinition$2`).

Page objects should accept an optional `tab` parameter in navigation methods:

```typescript
public async goto(id: number, tab?: string): Promise<void> {
    let url = `/front/computer.form.php?id=${id}`;
    if (tab) {
        url += `&forcetab=${tab}`;
    }
    await this.page.goto(url);
}
```

Only click tabs directly when you need to set up `page.route()` intercepts before the tab content loads.

### Accessibility testing

Use `@axe-core/playwright` (`AxeBuilder`) for accessibility checks, scoped to the relevant page section:

```typescript
import AxeBuilder from '@axe-core/playwright';

const a11y_results = await new AxeBuilder({ page })
    .include('[data-testid="my-component"]')
    .analyze();
expect(a11y_results.violations).toEqual([]);
```

### Linting

All e2e test code must pass lint checks. Run the following command and fix any errors before considering the work done:

```sh
make lint-playwright lint-js
```

### Running tests

Always run the relevant tests to validate changes. Use the following command with the correct path to the spec file:

```sh
make playwright c=tests/e2e/specs/<path-to-spec-file>
```

Do not skip this step — tests must pass before the task is complete.

### Test fixtures and utilities

Use the custom GLPI fixture (`tests/e2e/fixtures/glpi_fixture.ts`) which provides `page`, `profile`, `entity`, `csrf`, `formImporter`, and `api` helpers. Use the `api` fixture to create test data instead of navigating the UI for setup.
