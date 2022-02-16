# Tram - File upload server

Adjust logic by passing [options](#options) to the constructor.

## Renaming
By default, existing files are not overwritten but applied a renaming scheme until a collision is avoided.

Filenames are stripped of leading and trailing space, and trailing dot.
This is also applied everytime the file is renamed.

**[`rename`](#rename)** default: `null`

Providing `"hex"` will create a random hexadecimal filename.

- **Sanitizing** is when the filename should be kept but translations must be made, 
  or when the file extension may need to be changed.
  
  [See more](lib/Scheme/Rename/Sanitize.php)
  
  **_File collision_**
  
  Sanitizing imply that for the same arguments, the same result is expected and 
  because of that, collisions can not be prevented.
  
  [`Sanitize`](lib/Scheme/Rename/Sanitize.php) is therefore a [`Pure`](lib/Scheme/Rename/Pure.php) scheme and 
  [`default_rename`](#default_rename) is applied instead if a collision is detected.

- **Renaming** is when the file extension should be kept but the filename does not matter 
  or must follow a certain pattern.
  
  [See more](lib/Scheme/Rename/Unique.php)
  
  **_File collision_**
  
  All schemes **other than** [`Pure`](lib/Scheme/Rename/Pure.php) ones are expected to give different results 
  for the same arguments, and must eventually solve a collision when applied consecutively. 

If the file already exists and no scheme is provided, [`default_rename`](#default_rename) will be applied.

[Available schemes](#renaming-schemes)

## Validating
Filenames are stripped of leading and trailing space, and trailing dot.

Filenames are scanned for filesystem reserved characters, control characters and dot paths.

If the filename fails these tests or are stripped to the empty string, the file is **excluded**.

This security check is applied before any renaming.

**[`validate`](#validate)** default: `null`

Providing `"image"` will only accept image types.

[Available schemes](#validating-schemes)

## Filesize
Filesize is limited by [`max_filesize`](#max_filesize) in addition to [`validate`](#validate).

While the limit is optional, PHP applies its own limitations directed in `php.ini`.

[See more](#max_filesize)

## Saving
Only successful files are saved, but all are available for inspection.

## Options

### `max_filesize`
`int|float|string|callable|Scheme\Validate\Prototype` default: `null`

Max filesize in megabytes.

PHP limits filesize with the [`upload_max_filesize`](https://www.php.net/manual/en/ini.core.php#ini.upload-max-filesize) and [`post_max_size`](https://www.php.net/manual/en/ini.core.php#ini.post-max-size) directives in `php.ini`.

If this setting exceed one of those, the same error will be returned, but no transfer will have happened.

This setting is just as versatile as [`validate`](#validate) but also supports numbers as 
a shorthand for `"filesize:limit"`.

Selective limiting can easily be achieved by supplying a method that restricts differently depending on filetype.

### `overwrite`
`bool` default: `false`
Whether or not to overwrite files.

### `rename`
`string|callable|Scheme\Rename\Prototype` default: `null`

How files are renamed.

All provided renaming schemes will fail for multiple file extensions, like `tar.gz`.

Strings must adhere to syntax:
> `["append"[:separator]] [class=]method[:argument,...][|another_method...]`

> `"hex:12"`

Prepend with `"append"` to keep the filename:
> `"append int"`: `"foobar-2344.txt"`

Change separator by specifying it after append:
> `"append:# int"`: `"foobar#2344.txt"`

Provide a subclass by prepending it to the method. Classes can not be mixed:
> `"Sanitize=clean"`

Multiple methods in the same class may be specified:
> `"Sanitize=clean|proper"`

Available classes are [`Unique`](lib/Scheme/Rename/Unique.php), [`Sanitize`](lib/Scheme/Rename/Sanitize.php) and [`Meta`](lib/Scheme/Rename/Meta.php).

[`Unique`](lib/Scheme/Rename/Unique.php) is used if omitted.

#### Renaming schemes

- `"hex[:length]"`:
  `"6e154dfc.txt"`

- `"int[:length]"`:
  `"8737.txt"`
  
- `"Meta=time"`:
  `"20220129T140829Z0100.txt"`

- `"Sanitize=clean"`
  
  Replaces space with dash and truncates excessive `.` (dot), `-` (slash) and `_` (underscore).

- `"Sanitize=proper"`
  
  Removes a set of characters unsuitable or with special meaning in URLs.

- `"Sanitize=prude"`
  
  Removes all but `A-Z`, `0-9`, `_` (underscore), `-` (dash) and `.` (dot).

- `"Sanitize=strip:pattern"`
  
  Removes by pattern where pattern is a complete regex pattern: `"Sanitize=strip:/[!?:,]/"`

- `callable`
  ```php
  fn(File $file): string
  ```
  Function is bound to [`Unique`](lib/Scheme/Rename/Unique.php) and able to use its public methods.

  Functions must return the complete filename **including** file extension but **_excluding_** path.

  ***Important: The function will be invoked with the same argument until the returned filename does not exist or [`overwrite`](#overwrite) is `false`, this may result in an infinite loop.***
  ```php
  fn($file) => strtoupper($this->hex(file: $file, length: 4))
  ```

- [`Scheme\Rename\Prototype`](lib/Scheme/Rename/Prototype.php)
  
  - Same as `"append hex:12"`:
    ```php
    new Scheme\Rename\Unique(["hex" => [12]], ["append" => []])
    ```
    
  - Same as `"hex"`:
    ```php
    (new Scheme\Rename\Unique)->hex(...)
    [new Scheme\Rename\Unique, "hex"]
    ```

- `null`
  
  Applies [`default_rename`](#default_rename) if file exists and [`overwrite`](#overwrite) is `false`.

[How to implement a custom class](lib/Scheme/Rename/Prototype.php)

### `validate`
`string|callable|Scheme\Validate\Prototype` default: `null`

How files are validated.

Strings must adhere to syntax:

> `[class=]method[:argument,...][|another_method...]`

> `"whitelist:image/jpeg,text/plain"`

Provide a subclass by prepending it to the method:
> `"ScriptValidate=php"`

The only available class is [`Common`](lib/Scheme/Validate/Common.php) which is used if omitted.

Filesize is automatically limited to [`max_filesize`](#max_filesize).

Filenames resembling malicious inputs are automatically invalidated.

Multiple methods in the same class may be specified:
> `"whitelist:text/plain|strict"`

#### Validating schemes

- `"image[:mimetypes]"`
  
  Reliably validate files as images, optionally a subset of mimetypes.

- `"whitelist:mimetypes"`
  
  Only allow the provided mimetypes.

- `"blacklist:mimetypes"`
  
  Reject the provided mimetypes.

- `"only:characters"`
  
  Only allow the provided characters in filenames.

- `"none:characters"`
  
  Reject the provided characters in filenames.

- `callable`
  ```php
  fn(File $file): bool
  ```
  Function is bound to [`Common`](lib/Scheme/Validate/Common.php) and able to use its public methods.
  ```php
  fn($file) => preg_match("/^[A-Za-z0-9._-]+$/", $file->filename) === 1
  ```

- [`Scheme\Validate\Prototype`](lib/Scheme/Validate/Prototype.php)
  - Same as `"whitelist:image/jpeg,text/plain"`:
    ```php
    new Scheme\Validate\Common(["whitelist" => ["image/jpeg", "text/plain"]])
    ```
    
  - Same as `"image"`:
    ```php
    (new Scheme\Validate\Common)->image(...)
    [new Scheme\Validate\Common, "image"]
    ```

- `null`
  
  No validation is applied.

[How to implement a custom class](lib/Scheme/Validate/Prototype.php)

### `default_rename`
`string|callable|Scheme\Rename\Prototype` default: `"append int"`

How files are renamed if the file already exists, [`overwrite`](#overwrite) is `false` and
there is no [`rename`](#rename) provided or the provided one is a [`Pure`](lib/Scheme/Rename/Pure.php) scheme.

[See `rename` for more](#rename)

### `mode_directory`
`octal` default: `0755`

Permission mode for created directories.

### `mode_file`
`octal` default: `0644`

Permission mode for saved files.

## Usage
```php
$track = new Track(["max_filesize" => 4]);
$route = $track->stop("profile_images")->transit("/avatars");

foreach ($route as $file) {
 if ($file->ok()) {
     print "{$file->name} was successfully uploaded!\n";
 } else {
     print "{$file->name} failed to upload: ".implode("\n", $file->events())."\n";
 }
}
```

Files can be saved to multiple directories by calling [`transit()`](lib/Route.php) again.
```php
$files = $route->transit("/avatars")->transit("/backup")->tally();
```

The [`Route`](lib/Route.php) object returned from [`stop()`](lib/Track.php) is **JSON** encodable.
