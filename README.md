# Tram - File upload server

Adjust logic by passing options to the constructor.

See [Options](#options) for an in depth description.

## Renaming
By default, existing files are not overwritten but applied a renaming scheme until a collision is avoided.

Filenames are stripped of leading and trailing space, and trailing dot.
This is also applied everytime the file is renamed.

### `rename` default: `null`
Providing `"hex"` will create a random hexadecimal filename.

- **Sanitizing** is when the filename should be kept but translations must be made, 
  or when the file extension may need to be changed.
  
  [See more](lib/Scheme/Rename/Sanitize.php)
  
  #### File collision
  Sanitizing imply that for the same arguments, the same result is expected and 
  because of that, collisions can not be prevented.
  
  Therefore [`default_rename`](#default_rename-stringcallableschemerenameprototype-default-append-int) is applied instead if a collision is detected.

- **Renaming** is when the file extension should be kept but the filename does not matter 
  or must follow a certain pattern.
  
  [See more](lib/Scheme/Rename/Unique.php)
  
  #### File collision
  All schemes **other than** [Scheme\Rename\Pure](lib/Scheme/Rename/Pure.php) are expected to give different results 
  for the same arguments, and must eventually solve a collision when applied consecutively. 

If the file already exists and no scheme is provided, [`default_rename`](#default_rename-stringcallableschemerenameprototype-default-append-int) will be applied.

[Available schemes](#values)

## Validating
Filenames are stripped of leading and trailing space, and trailing dot.

Filenames are scanned for filesystem reserved characters, control characters and dot paths.

If the filename fails these tests or are stripped to the empty string, the file is **excluded**.

This security check is applied before any renaming.

### `validate` default: `null`
Providing `"image"` will only accept image types.

[Available schemes](#values-1)

## Filesize
Filesize is limited by [`max_filesize`](#max_filesize-intfloat-default-null) in addition to [`validate`](#validate-stringcallableschemevalidateprototype-default-null).

While the limit is optional, PHP applies its own limitations directed in `php.ini`.

[See more](#max_filesize-intfloat-default-null)

## Saving
Only successful files are saved, but all are available for inspection.

## Options

### `max_filesize`: `int|float` default: `null`
Max filesize in megabytes.

PHP limits filesize with the [`upload_max_filesize`](https://www.php.net/manual/en/ini.core.php#ini.upload-max-filesize) and [`post_max_size`](https://www.php.net/manual/en/ini.core.php#ini.post-max-size) directives in `php.ini`.

If this setting exceed one of those, the same error will be returned, but no transfer will have happened.

### `overwrite`: `bool` default: `false`
Whether or not to overwrite files.

### `rename`: `string|callable|Scheme\Rename\Prototype` default: `null`
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

Available classes are `Unique` and `Sanitize`. [Scheme\Rename\Unique](lib/Scheme/Rename/Unique.php) is used if omitted.

#### Values

- `"time"`:
  `"20220129T140829Z0100.txt"`

- `"hex[:length]"`:
  `"6e154dfc.txt"`

- `"int[:length]"`:
  `"8737.txt"`

- `"Sanitize=clean"`
  
  Replaces space with dash and truncates excessive `.` (dot), `-` (slash) and `_` (underscore).

- `"Sanitize=proper"`
  
  Removes a set of characters unsuitable or with special meaning in URLs.

- `"Sanitize=prude"`
  
  Removes all but `A-Z`, `0-9`, `_` (underscore), `-` (dash) and `.` (dot).

- `"Sanitize=strip:pattern"`
  
  Removes by pattern where pattern is a complete regex pattern: `Sanitize=strip:/[!?:,]/`

- `callable`
  ```php
  fn(File $file): string
  ```
  Function is bound to [Scheme\Rename\Unique](lib/Scheme/Rename/Unique.php) and able to use its public methods.

  Functions must return the complete filename **including** file extension but **_excluding_** path.

  ***Important: The function will be invoked with the same argument until the returned filename does not exist or `overwrite` is `false`, this may result in an infinite loop.***
  ```php
  fn($file) => strtoupper($this->hex(file: $file, length: 4))
  ```

- `Scheme\Rename\Prototype`
  
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
  
  Applies [`default_rename`](#default_rename-stringcallableschemerenameprototype-default-append-int) if file exists and [`overwrite`](#overwrite-bool-default-false) is `false`.

[How to implement a custom class](lib/Scheme/Rename/Prototype.php)

### `validate`: `string|callable|Scheme\Validate\Prototype` default: `null`
How files are validated.

Strings must adhere to syntax:

> `[class=]method[:argument,...][|another_method...]`

> `"whitelist:image/jpeg,text/plain"`

Provide a subclass by prepending it to the method:
> `"ScriptValidate=php"`

The only available class is [Scheme\Validate\Common](lib/Scheme/Validate/Common.php) which is used if omitted.

Filesize is automatically limited to [`max_filesize`](#max_filesize-intfloat-default-null).

Filenames resembling malicious inputs are automatically invalidated.

Multiple methods in the same class may be specified:
> `"whitelist:text/plain|strict"`

#### Values

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
  Function is bound to [Scheme\Validate\Common](lib/Scheme/Validate/Common.php) and able to use its public methods.
  ```php
  fn($file) => preg_match("/^[A-Za-z0-9._-]+$/", $file->filename) === 1
  ```

- `Scheme\Validate\Prototype`
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

### `default_rename`: `string|callable|Scheme\Rename\Prototype` default: `"append int"`
How files are renamed if the file already exists, [`overwrite`](#overwrite-bool-default-false) is `false` and
there is no [`rename`](#rename-stringcallableschemerenameprototype-default-null) provided or the provided one is a [Scheme\Rename\Pure](lib/Scheme/Rename/Pure.php).

[See `rename` for more](#rename-stringcallableschemerenameprototype-default-null)

### `mode_directory`: `octal` default: `0755`
Permission mode for created directories.

### `mode_file`: `octal` default: `0644`
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

Files can be saved to multiple directories by calling [`Route::transit()`](lib/Route.php) again.
```php
$files = $route->transit("/avatars")->transit("/backup")->tally();
```

The [`Route`](lib/Route.php) object returned from [`Track::stop()`](lib/Track.php) is **JSON** encodable.
