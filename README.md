# Tram - File upload server
Adjust logic by passing [options](#options) to the constructor.

## Renaming
Filenames are stripped of leading and trailing space, and trailing dot.

By default, existing files are not overwritten but applied a renaming scheme instead.

**[rename](#rename)** default: `null`

Providing `"Random=hex"` will create a random hexadecimal filename.

Built in classes to cover most requirements:
- **[Random](lib/Scheme/Rename/Random.php)**
  
  Generate unique names and keep the file extension.
  
- **[Sanitize](lib/Scheme/Rename/Sanitize.php) implements [Pure](lib/Scheme/Rename/Pure.php)**
  
  Replace or remove characters or strings, may also change file extension.
  
- **[Info](lib/Scheme/Rename/Info.php) implements [Pure](lib/Scheme/Rename/Pure.php)**
  
  Rename file according to characteristics of it.

If the file already exists and no scheme is provided, [default_rename](#default_rename) will be applied.

[Available schemes](#schemes)

## Validating
Filenames are stripped of leading and trailing space, and trailing dot.

Filenames are scanned for filesystem reserved characters, control characters and dot paths.

If the filename fails these tests or are stripped to the empty string, the file is **excluded**.

This security check is applied before any renaming.

**[validate](#validate)** default: `null`

Providing `"Common=image"` will only accept image types.

Built in classes to cover most requirements:
- **[Common](lib/Scheme/Validate/Common.php)**
  
  Validate mimetype, filename and characteristics of files.

### Multiple methods
When supplying multiple methods there are two distinct ways to validate.

- **Every** method must succeed for the file to be valid.
- Only **some** methods must succeed for the file to be valid.

The `approach` mode supports **`Every`** and **`Some`**.

**`Every`** is used if omitted.

[Available schemes](#schemes-1)

## Filesize
Filesize is limited by [max_filesize](#max_filesize) in addition to [validate](#validate).

While the limit is optional, PHP applies its own limitations directed in `php.ini`.

[See more](#max_filesize)

## Saving
Only successful files are saved, but all are available for inspection.

## Option syntax description
Options, if strings, are parsed according to a domain specific syntax:
> `[mode[":" argument["," argument]...]] class "=" method[":" argument["," argument]...]["|" method]...`

## Options
### max_filesize
`int|float|string|callable|Scheme\Validate\Validate` default: `null`

Max filesize in megabytes.

PHP limits filesize with the [upload_max_filesize](https://www.php.net/manual/en/ini.core.php#ini.upload-max-filesize) and [post_max_size](https://www.php.net/manual/en/ini.core.php#ini.post-max-size) directives in `php.ini`.

If this setting exceed one of those, the same error will be returned, but no transfer will have happened.

This setting is just as versatile as [validate](#validate) but also supports numbers as 
a shorthand for `"Common=filesize:limit"`.

[How to implement a custom scheme](#custom-scheme)

### overwrite
`bool` default: `false`
Whether or not to overwrite files.

### rename
`string|callable|Scheme\Rename\Rename` default: `null`

How files are renamed.

No built in renaming scheme take multiple extensions into account, like `tar.gz`, and will fail when 
providing new filenames.

Strings must adhere to syntax:
> `["append"[:separator]] class=method[:arguments]...`
[Syntax description](#option-syntax-description)

> `"Random=hex:12"`

Prepend with `"append"` to keep the filename:
> `"append Random=int"`: `"foobar-2344.txt"`

Change separator by specifying it after append:
> `"append:# Random=int"`: `"foobar#2344.txt"`

Multiple methods in the same class may be specified:
> `"Sanitize=clean|proper"`

#### Schemes
- **[Random](lib/Scheme/Rename/Random.php)**
  - `"Random=hex[:length]"`:
    `"6e154dfc.txt"`
  - `"Random=int[:length]"`:
    `"8737.txt"`
- **[Info](lib/Scheme/Rename/Info.php)**
  - `"Info=time"`:
    `"20220129T140829Z0100.txt"`
- **[Sanitize](lib/Scheme/Rename/Sanitize.php)**
  - `"Sanitize=clean"`
  
    Replaces space with dash and truncates excessive `.` (dot), `-` (slash) and `_` (underscore).

  - `"Sanitize=proper"`
  
    Removes a set of characters unsuitable or with special meaning in URLs.

  - `"Sanitize=prude"`
  
    Removes all but `A-Z`, `0-9`, `_` (underscore), `-` (dash) and `.` (dot).

- `callable`
  ```php
  fn(File $file): string
  ```
  - Functions must return the complete filename **including** file extension but **_excluding_** path.

  - ***Functions will be invoked with the same argument until the returned filename does not exist or [overwrite](#overwrite) is `false`, this may result in an infinite loop.***
  
  - Functions may bind to a scheme to use its public methods:
    ```php
    (fn($file) => 
      strtoupper($this->hex(file: $file, length: 4))
    )->bindTo(new Scheme\Rename\Random);
    ```

- **[Scheme\Rename\Rename](lib/Scheme/Rename/Rename.php)**
  
  - Same as `"append Random=hex:12"`:
    ```php
    new Scheme\Rename\Random(["hex" => [12]], ["append" => []]);
    ```
    
  - Same as `"Random=hex"`:
    ```php
    (new Scheme\Rename\Random)->hex(...);
    [new Scheme\Rename\Random, "hex"];
    ```

- `null`
  
  Applies [default_rename](#default_rename) if file exists and [overwrite](#overwrite) is `false`.

[How to implement a custom scheme](#custom-scheme)

### validate
`string|callable|Scheme\Validate\Validate` default: `null`

How files are validated.

Strings must adhere to syntax:

> `["Every"|"Some"] class=method[:arguments]...`
[Syntax description](#option-syntax-description)

> `"Common=whitelist:image/jpeg,text/plain"`

Prepend with the preferred algorithm:
> `"Every Common=image|blacklist:image/x-icon"`: Files must be an image **and** not an ICO image.

> `"Some Common=image|whitelist:text/plain"`: Files must be **either** an image or text.

`"Every"` is used if omitted.

Filesize is automatically limited to [max_filesize](#max_filesize).

Filenames resembling malicious inputs are automatically invalidated.

Multiple methods in the same class may be specified:
> `"Some Common=image|whitelist:text/plain"`

#### Schemes
- `"Common=image[:mimetypes]"`
  
  Reliably validate files as images, optionally a subset of mimetypes.

- `"Common=whitelist:mimetypes"`
- `"Common=blacklist:mimetypes"`
- `"Common=only:characters"`
- `"Common=none:characters"`

- `callable`
  ```php
  fn(File $file): bool
  ```
  - Functions may bind to a scheme to use its public methods:
    ```php
    (fn($file) => 
      $this->image($file) && 
      preg_match("/^[A-Za-z0-9._-]+$/", $file->filename) === 1
    )->bindTo(new Scheme\Validate\Common);
    ```

- **[Scheme\Validate\Validate](lib/Scheme/Validate/Validate.php)**
  - Same as `"Common=whitelist:image/jpeg,text/plain"`:
    ```php
    new Scheme\Validate\Common(["whitelist" => ["image/jpeg", "text/plain"]]);
    ```
    
  - Same as `"Common=image"`:
    ```php
    (new Scheme\Validate\Common)->image(...);
    [new Scheme\Validate\Common, "image"];
    ```

- `null`
  
  No validation is applied.

[How to implement a custom scheme](#custom-scheme)

### default_rename
`string|callable|Scheme\Rename\Rename` default: `"append Random=int"`

How files are renamed if the file already exists, [overwrite](#overwrite) is `false` and
there is no [rename](#rename) scheme provided or the provided one is a **[`Pure`](lib/Scheme/Rename/Pure.php)** scheme.

***The provided scheme must eventually solve a file collision if applied again and again with the same arguments.***

[See rename for more](#rename)

### mode_directory
`octal` default: `0755`

Permission mode for created directories.

### mode_file
`octal` default: `0644`

Permission mode for saved files.

## Usage
```php
$track = new Tram\Track(["max_filesize" => 4]);
$route = $track->stop("profile_images")->transit("/avatars");

foreach ($route as $file) {
 if ($file->ok()) {
     print "{$file->name} was successfully uploaded!\n";
 } else {
     print "{$file->name} failed to upload: ".implode("\n", $file->events())."\n";
 }
}
```

Files can be saved to multiple directories by calling **[`transit()`](lib/Route.php)** again.
```php
$files = $route->transit("/avatars")->transit("/backup")->tally();
```

The **[Route](lib/Route.php)** object returned from **[`stop()`](lib/Track.php)** is **JSON** encodable.

## Custom scheme
### Renaming scheme
#### Translation and Generation
Renaming schemes take one of two forms:
- **[Translate](lib/Scheme/Rename/Translate.php)**
  - All methods are applied in sequence, each receiving the result of the previous method.
  - Methods return the complete filename and may change extension.
- **[Generate](lib/Scheme/Rename/Generate.php)**
  - Only the first method is applied.
  - Methods return the filename **without** extension and may **not** change extension.

On top of that, a scheme may signal that it is unable to solve file collisions:
- **[Pure](lib/Scheme/Rename/Pure.php)**
  - Methods are only applied once.
  - [default_rename](#default_rename) is applied in case of a file collision.

### Example #1
Let's implement a renaming scheme that names photographs according to their metadata.

We'll want names to contain the date the photo was taken, the width and height of the photo and the camera model taken with.

Because our scheme will return the same result for identical arguments, we'll want a **[Pure](lib/Scheme/Rename/Pure.php)** scheme so that collisions are handled by [default_rename](#default_rename) instead.

A suitable class is **[Info](lib/Scheme/Rename/Info.php)**, let's add a method to it:
```php
namespace Tram\Scheme\Rename;
use Tram\File;

class Info extends Generate implements Pure {
  /* ... */
  public function DSC(File $file): string {}
}
```

We can just as well create a whole new class, extend a suitable class, or even use a `callable`.
Having our class implement **[Pure](lib/Scheme/Rename/Pure.php)** is a simple way not to worry about file collision prevention.

For our implementation, we'll want it to handle non-images and images without metadata as well.
Let's write a fail-safe approach:
```php
public function DSC(File $file): string {
  static $exif;
  $exif ??= function_exists("exif_read_data");  /* cache lookup */

  $data = getimagesize($file->path);

  /* not an image */
  if (!$data or $data[2] < 1) {
    return date("Y-m-d_H-i-s", filemtime($file->path)?? time());
  }

  /* getimagesize() gives us the width and height */
  $resolution = "@{$data[0]}x{$data[1]}";
  /* if possible, read metadata */
  $meta = $exif? exif_read_data($file->path, "IFD0", true) : [];
  /* try different metadatas, resort to the file time */
  $time = $meta["EXIF"]["DateTimeOriginal"]?? 
          $meta["IFD0"]["DateTime"]??
          date("Y:m:d H:i:s", $meta["FILE"]["FileDateTime"]?? filemtime($file->path)?? time());
  $time = strtr($time, ": ", "-_");  /* rewrite the date to the same format as for non-images */
  $model = isset($meta["IFD0"]["Model"])? "_".str_replace(" ", "", $meta["IFD0"]["Model"]) : "";
  
  /* The parent class Generate provides the file extension */
  return $time.$resolution.$model;  /* 2022-02-01_14-52-38@6016x4000_NIKOND3200 */
}
```

Provide it as an option:
```php
new Tram\Track([
  "rename" => "Info=DSC"
]);
```

### Example #2
Let's limit filesize differently depending on the type of file:
```php
namespace Tram\Scheme\Validate;
use Tram\File;

class SelectiveFilesize extends Validate {
  /** Limits specified per mimetype, in megabytes */
  private const limits = [
    [
      "mimetypes" => ["image/"],
      "limit" => 3
    ],
    [
      "mimetypes" => ["text/plain"],
      "limit" => 1
    ],
    [
      "mimetypes" => ["application/msword", "application/vnd.openxmlformats-officedocument.wordprocessingml.document", "application/vnd.oasis.opendocument.text"],
      "limit" => 2
    ]
  ];

  /** The maximum limit regardless of mimetype */
  private const max_limit = 5;

  public function limit(File $file): bool {
    /* convert bytes to megabytes */
    $size = $file->size/1e6;
    $limit = $this->mimetype_to_limit($file->mimetype)?? self::max_limit;

    return $size <= $limit;
  }

  /** Searches for the limit on the mimetype */
  private function mimetype_to_limit(string $mimetype): ?float {
    foreach (self::limits as ["mimetypes" => $mimetypes, "limit" => $limit]) {
      foreach ($mimetypes as $current) {
        if (str_starts_with($mimetype, $current)) {
          return $limit;
        }
      }
    }
    return null;
  }
}
```

Now, the max filesize is selective:
```php
new Tram\Track([
  "max_filesize" => "SelectiveFilesize=limit"
]);
```
