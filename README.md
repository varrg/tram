# Tram - File upload server

Adjust logic by passing [options](#options) to the constructor.

## Renaming
Filenames are stripped of leading and trailing space, and trailing dot.
This is also applied everytime the file is renamed.

By default, existing files are not overwritten but applied a renaming scheme until a collision is avoided.

**[rename](#rename)** default: `null`

Providing `"hex"` will create a random hexadecimal filename.

Built in classes to cover most requirements:
- **[Unique](lib/Scheme/Rename/Unique.php)**
  
  Random names, keep the file extension.
- **[Sanitize](lib/Scheme/Rename/Sanitize.php) implements [Pure](lib/Scheme/Rename/Pure.php)**
  
  Replace or remove characters or strings, may also change file extension.
- **[Meta](lib/Scheme/Rename/Meta.php) implements [Pure](lib/Scheme/Rename/Pure.php)**
  
  Rename file according to characteristics of it.

### File collision
  
By default, the renaming scheme will be applied again and again with the same arguments and 
methods must ensure a collision is eventually solved.

All schemes implementing **[Pure](lib/Scheme/Rename/Pure.php)**, on the other hand, are only 
applied once and for any file collision the [default_rename](#default_rename) is applied instead.

If the file already exists and no scheme is provided, [default_rename](#default_rename) will be applied.

[Available schemes](#renaming-schemes)

## Validating
Filenames are stripped of leading and trailing space, and trailing dot.

Filenames are scanned for filesystem reserved characters, control characters and dot paths.

If the filename fails these tests or are stripped to the empty string, the file is **excluded**.

This security check is applied before any renaming.

**[validate](#validate)** default: `null`

Providing `"image"` will only accept image types.

[Available schemes](#validating-schemes)

## Filesize
Filesize is limited by [max_filesize](#max_filesize) in addition to [validate](#validate).

While the limit is optional, PHP applies its own limitations directed in `php.ini`.

[See more](#max_filesize)

## Saving
Only successful files are saved, but all are available for inspection.

## Options

### max_filesize
`int|float|string|callable|Scheme\Validate\Prototype` default: `null`

Max filesize in megabytes.

PHP limits filesize with the [upload_max_filesize](https://www.php.net/manual/en/ini.core.php#ini.upload-max-filesize) and [post_max_size](https://www.php.net/manual/en/ini.core.php#ini.post-max-size) directives in `php.ini`.

If this setting exceed one of those, the same error will be returned, but no transfer will have happened.

This setting is just as versatile as [validate](#validate) but also supports numbers as 
a shorthand for `"filesize:limit"`.

[How to implement a custom class](#filesize-validating-scheme)

### overwrite
`bool` default: `false`
Whether or not to overwrite files.

### rename
`string|callable|Scheme\Rename\Prototype` default: `null`

How files are renamed.

No built in renaming scheme take multiple extensions into account, like `tar.gz`, and will fail when 
providing new filenames.

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

Available classes are **[Unique](lib/Scheme/Rename/Unique.php)**, **[Sanitize](lib/Scheme/Rename/Sanitize.php)** and **[Meta](lib/Scheme/Rename/Meta.php)**.

**[Unique](lib/Scheme/Rename/Unique.php)** is used if omitted.

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

- `callable`
  ```php
  fn(File $file): string
  ```
  Function is bound to **[Unique](lib/Scheme/Rename/Unique.php)** and able to use its public methods.

  Functions must return the complete filename **including** file extension but **_excluding_** path.

  ***Important: The function will be invoked with the same argument until the returned filename does not exist or [overwrite](#overwrite) is `false`, this may result in an infinite loop.***
  ```php
  fn($file) => strtoupper($this->hex(file: $file, length: 4))
  ```

- **[Scheme\Rename\Prototype](lib/Scheme/Rename/Prototype.php)**
  
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
  
  Applies [default_rename](#default_rename) if file exists and [overwrite](#overwrite) is `false`.

[How to implement a custom class](#renaming-scheme)

### validate
`string|callable|Scheme\Validate\Prototype` default: `null`

How files are validated.

Strings must adhere to syntax:

> `[mode[:argument,...]] [class=]method[:argument,...][|another_method...]`

> `"whitelist:image/jpeg,text/plain"`

Provide a subclass by prepending it to the method:
> `"Script=php"`

Available classes are **[Common](lib/Scheme/Validate/Common.php)** and the abstract class **[Some](lib/Scheme/Validate/Some.php)**.

**[Some](lib/Scheme/Validate/Some.php)** can be extended to allow methods provided to fail as long as at least one succeed.

**[Common](lib/Scheme/Validate/Common.php)** is used if omitted.

Filesize is automatically limited to [max_filesize](#max_filesize).

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
  Function is bound to **[Common](lib/Scheme/Validate/Common.php)** and able to use its public methods.
  ```php
  fn($file) => preg_match("/^[A-Za-z0-9._-]+$/", $file->filename) === 1
  ```

- **[Scheme\Validate\Prototype](lib/Scheme/Validate/Prototype.php)**
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

[How to implement a custom class](#validating-scheme)

### default_rename
`string|callable|Scheme\Rename\Prototype` default: `"append int"`

How files are renamed if the file already exists, [overwrite](#overwrite) is `false` and
there is no [rename](#rename) scheme provided or the provided one is a **[`Pure`](lib/Scheme/Rename/Pure.php)** scheme.

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
Let's implement a renaming scheme that names photographs according to their metadata.

We'll want names to contain the date the photo was taken, the width and height of the photo and the camera model taken with.

Because our scheme will return the same result for identical arguments, we'll want a **[Pure](lib/Scheme/Rename/Pure.php)** scheme so that collisions are handled by [default_rename](#default_rename) instead.

A suitable class is **[Meta](lib/Scheme/Rename/Meta.php)**, let's add a method to it:
```php
namespace Tram\Scheme\Rename;
use Tram\File;

class Meta extends Unique implements Pure {
  /* ... */
  public function DSC(File $file): string {}
}
```

We can just as well create a whole new class, extend a suitable class or the **[base class](lib/Scheme/Rename/Prototype)**, or even use a `callable`.
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
  
  /* The parent class Unique provides the file extension */
  return $time.$resolution.$model;  /* 2022-02-01_14-52-38@6016x4000_NIKOND3200 */
}
```

Provide it as an option:
```php
new Tram\Track([
  "rename" => "Meta=DSC"
]);
```

### Validating scheme
**[Common](lib/Scheme/Validate/Common.php)** succeeds only if all methods succeed.

Let's implement a validating scheme that toggles between the default and succeeding if **at least** one method succeed.

We'll inherit from **[Common](lib/Scheme/Validate/Common)** so we can use its methods:
```php
namespace Tram\Scheme\Validate;

class SomeOrEvery extends Common {}
```

We need to be able to provide the algorithm, let's implement a `mode`:
```php
protected string $algorithm = "every";

public function initialize(): void {
  /* the name of the mode is the only key and its arguments is the value */
  $algorithm = array_key_first($this->mode);
  if ($algorithm) {
    $this->algorithm = strtolower($algorithm);
  }
}
```

Now, we can specify algorithm by prepending the mode, and because we inherit from **[Common](lib/Scheme/Validate/Common)**, we can reference its methods:
```php
new Tram\Track([
  "validate" => "Some SomeOrEvery=image|whitelist:text/plain"
]);
```

Let's implement logic on the algorithm, the supplied methods are applied in **[`apply()`](lib/Scheme/Validate/Prototype.php)**:
```php
public function apply(\Tram\File $file): bool {
  foreach ($this->methods as $method => $args) {
    if ($this->{$method}($file, ...$args)) {
      /* at least one method succeeded, for Some, this is sufficient */
      if ($this->algorithm == "some") {
        return true;
      }
    } elseif ($this->algorithm == "every") {
      /* at least one method failed, for Every, this is sufficient */
      return false;
    }
  }

  /* for Every, reaching here means all methods succeeded,
  for Some, reaching here means no method succeeded */
  return $this->algorithm == "every";
}

/** Whitelists extensions, when mimetype is just not enough */
public function extension(\Tram\File $file, string ...$whitelist): bool {
  return in_array($file->extension, $whitelist);
}
```

Now, our scheme is flexible:
```php
new Tram\Track([
  "validate" => "Every SomeOrEvery=whitelist:text/plain,application/msword|extension:doc"
]);
```

### Filesize validating scheme
Let's limit filesize differently depending on the type of file:
```php
namespace Tram\Scheme\Validate;
use Tram\File;

class SelectiveFilesize extends Common {
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
