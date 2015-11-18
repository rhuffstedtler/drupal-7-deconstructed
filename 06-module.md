# Module Support

The major difference between Drupal and most other CMSes is its modular
construction. Almost every aspect of the system is exposed in such a
way that developers can extend it or override by implementing
particular hooks in their own modules. They can also create hooks in
their module so that other developers can further extend the system.

Aside from extensibility, a second advantage is that it is possible to
disable unneeded modules to reduce the amount of code that is executed,
which improves Drupal's performance and reduces its exposure to attacks.

All of the functions that define the module subsystem live in
[module.inc](https://api.drupal.org/api/drupal/includes!module.inc/7).
You may remember from chapter 2 that this file is first included during the DRUPAL\_BOOTSTRAP\_VARIABLES
phase of the bootstrap process.

## The underlying database
All of the module information is stored in the system table.
```
+----------------+--------------+------+-----+---------+-------+
| Field          | Type         | Null | Key | Default | Extra |
+----------------+--------------+------+-----+---------+-------+
| filename       | varchar(255) | NO   | PRI |         |       |
| name           | varchar(255) | NO   |     |         |       |
| type           | varchar(255) | NO   | MUL |         |       |
| owner          | varchar(255) | NO   |     |         |       |
| status         | int(11)      | NO   |     | 0       |       |
| throttle       | tinyint(4)   | NO   |     | 0       |       |
| bootstrap      | int(11)      | NO   |     | 0       |       |
| schema_version | smallint(6)  | NO   |     | -1      |       |
| weight         | int(11)      | NO   |     | 0       |       |
| info           | text         | YES  |     | NULL    |       |
+----------------+--------------+------+-----+---------+-------+
```


## How modules get registered
Before we can use the functionality provided by a module, we must first
download it, install it, and enable it. The act of downloading the
module, whether one uses drush dl, FTPs the files, or uses the Update
module (don't do that), is simply the act of placing the files in a
directory within your Drupal installation where Drupal will know to scan
for the code. Prior to this point, all of the functionality of Drupal we
have looked at has been in non-optional include files. However, even
within core there is functionality provided by 40 separate modules, many
of which can be disabled.

*Need to look at the minimal install profile to figure out what the
absolute bare minimum is*

Installing a module is the act of enabling it for the first time. During
install, the system may need to create database tables for the module or
run other install hooks. When a module is enabled, we check to see if it
has dependencies which must be met, install its schema and update the
cache, invoke the install hooks (if this is our first installation or a
reinstallation) and then invoke hook\_modules\_installed() and
hook\_modules\_enabled().

Let's look at the code:
```
function module_enable($module_list, $enable_dependencies = TRUE) {
  if ($enable_dependencies) {
    // Get all module data so we can find dependencies and sort.
    $module_data = system_rebuild_module_data();
    // Create an associative array with weights as values.
    $module_list = array_flip(array_values($module_list));
```
$enable_dependencies is true by default. Most of the situations in which
it is overridden are in tests or when running other installs which can
be trusted to manage their own dependencies.

[system_rebuild_module_data()](https://api.drupal.org/api/drupal/modules!system!system.module/function/system_rebuild_module_data/7) is itself implemented in a module
(system.module to be precise). It's a short function, so let's look at it in its entirety:
```
function system_rebuild_module_data() {
  $modules_cache = &drupal_static(__FUNCTION__);
  // Only rebuild once per request. $modules and $modules_cache cannot be
  // combined into one variable, because the $modules_cache variable is reset by
  // reference from system_list_reset() during the rebuild.
  if (!isset($modules_cache)) {
    $modules = _system_rebuild_module_data();
    ksort($modules);
    system_get_files_database($modules, 'module');
    system_update_files_database($modules, 'module');
    $modules = _module_build_dependencies($modules);
    $modules_cache = $modules;
  }
  return $modules_cache;
}
```
Rebuilding the cache is a relatively expensive operation, so we first
check to see if we already have it cached. If not, we call
_system_rebuild_module_data() which does all the heavy lifting.

It's too long to reproduce, but the general gist of what it does is:
1. Use [drupal_system_listing()](https://api.drupal.org/api/drupal/includes!common.inc/function/drupal_system_listing/7)
   to scan the modules directories under the current installation profile
   and under sites/all to find every file that ends with a ".module" 
   extension.
2. Add the installation profile to list of modules and guarantee that
   its hooks are executed after all other modules by setting the weight
   to 1000.
3. Provide a reasonable set of defaults for .info parameters that might
   not be provided by individual modules.
4. If the directory it is in also has a .info file, call
   drupal_parse_info_file() on it. That function is basically just a
   cache wrapper around drupal_parse_info_format(), which is largely
   an ugly regular expression that turns a .info file into a nested
   array. For details on the .info format itself, see [Writing module
   .info files (Drupal 7.x)](https://www.drupal.org/node/542202). There
   is a test immediately thereafter to skip the module if it did not
   have a .info file.
5. Merge in the defaults from step 3 above.
6. Prefix any stylesheets and javascript with the module path.
7. Invoke hook_system_info_alter(). This is worth a whole subchapter on
   its own, because hooks are how Drupal does its module magic.

## Hooks
Hooks are the heart of the module system. A hook is just a PHP function
that follows a naming convention of <module name>_<hook name>. The meat
of most modules is a set of hook implementations and various 
supporting functions (which are not required to follow any particular
naming convention).

In order to generate the list of modules that implement particular
hooks, we call (module_implements())[https://api.drupal.org/api/drupal/includes!module.inc/function/module_implements/7]
with the name of the hook. As you might imagine, scanning all of the PHP
files in every module is an expensive operation, so this is typically
maintained in cache. As an aside, that's why you have to clear cache
when you write a new hook implementation in one of your modules.


Depending on whom you ask, there are either two or three types of hooks:
1. alter hooks
2. intercepting hooks
3. info hooks

###Alter hooks
Alter hooks allow you to change the contents of an object. These often
contain the word "alter" in the name. Under the hood, alter hooks are
calls to the drupal_alter() function.

Considering how important it is, drupal_alter() is a surprisingly small
function, if you exclude the comment lines. As an aside, the comments in
this function are exceptionally good, and serve as a great example of
the kind of commenting that you should be doing as a developer.

drupal_alter() takes two required arguments:
$type - the kind of data element to be altered ("form",
"field_display", "token_info", etc.). This can be an array, as in the
case of forms which allow you to implement hook_form_alter() to alter all
forms or hook_form_FORM_ID_alter() to alter a specific form.

$data - An appropriate data type to be altered in the implementation of
the hook. Exactly what this is depends on the type of thing being
altered.

Because drupal_alter is frequently called, it uses the advanced
drupal_static() pattern, as described in the
[performance](18-performance.md) chapter. drupal_alter() may take either
a single type of alterable data or an array of alterable types. The
former is, by far, the more common use case.

We'll describe that case first, and after you understand the function
talk about how it differs when an array is passed.

Assuming that we don't already have the functions cached, we build the
hook name from the type and the string "_alter" (e.g., "form_alter"). 



###Intercepting hooks
Intercepting hooks allow modules to respond to an event (such as a
  node being saved).

###Info hooks
Some developers make the further distinction of info hooks which allow
  a developer to retrieve information about an object, e.g.
  hook_block_info(). These hooks often have "info" in the name.



 

## The module lifecycle

Modules are loaded by the [module\_load\_all](module_load_all) function.
This function takes a single boolean parameter, $bootstrap. If it is
true, only the bootstrap modules are loaded (those modules that
implement hook\_boot, hook\_exit, hook\_watchdog, or
hook\_language\_init). During DRUPAL\_BOOTSTRAP\_VARIABLES, it is called
with a value of true.

The rest of the modules are not loaded until DRUPAL\_BOOTSTRAP\_FULL.

The function itself is very simple:
```
function module_load_all($bootstrap = FALSE) {
  static $has_run = FALSE;

  if (isset($bootstrap)) {
    foreach (module_list(TRUE, $bootstrap) as $module) {
      drupal_load('module', $module);
    }
    // $has_run will be TRUE if $bootstrap is FALSE.
    $has_run = !$bootstrap;
  }
  return $has_run;
}
```
The isset test on bootstrap is because the theme include specifically
calls module_load_all with NULL. *Need to figure out why that is*

module_list returns the list of enabled modules. If bootstrap is TRUE,
it will only return the bootstrap modules.

module_list() is a fairly thin wrapper around [system_list()](https://api.drupal.org/api/drupal/includes!module.inc/function/system_list/7). system_list() can return either:
1. All enabled modules
2. All bootstrap modules
3. All themes

The value for the cache bucket keyed by this function's name is an
array. There are separate sub-lists of the cache for each of the
scenarios listed above.

## Responding to hooks

## Exposing hooks of our own

## Module updates

## Disabling a module

## Uninstalling a module

## Odds and ends to work in somewhere
- One option in the .info file that you may not be familiar with is
  *required*. If set to TRUE, the module cannot be disabled. This should
  only be set in core modules. At the time of this writing, the
  following modules are always required: field_sql_storage, field, text,
  filter, node, system, and user.
