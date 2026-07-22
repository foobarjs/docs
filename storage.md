[← Back to docs](./README.md)

# Storage

foobarjs provides a filesystem abstraction for managing file uploads and
storage.

## Configuration

Configure storage in `config/storage.js`:

```js
export default {
  default: 'local',
  disks: {
    local: { driver: 'local', root: 'public/uploads' },
  },
}
```

The default `local` disk stores files under `public/uploads/` regardless of
the `root` value. To use a different filesystem path, register a custom
`LocalDisk` — see [Custom disks](#custom-disks) below.

## Basic usage

```js
import { Storage } from 'foobarjs/storage'

await Storage.put('avatars/user1.jpg', fileContents)

const contents = await Storage.get('avatars/user1.jpg')

const url = Storage.url('avatars/user1.jpg')
// → /avatars/user1.jpg  (paths are already relative to public/)

await Storage.delete('avatars/user1.jpg')
```

`Storage` is also re-exported from the flat `foobarjs` entry:
`import { Storage } from 'foobarjs'`.

## Using a specific disk

```js
const disk = Storage.disk('local')
await disk.put('file.txt', 'content')
```

`Storage.put()`, `.get()`, `.url()`, `.delete()` all default to the disk
named by `config.storage.default`.

## Custom disks

Register custom storage disks (e.g., S3, GCS, or a `LocalDisk` with a
non-default root):

```js
import { Storage, LocalDisk } from 'foobarjs/storage'

// Point local storage at a different filesystem path.
Storage.registerDisk('local', new LocalDisk('/var/data/uploads'))

// Or add a new S3 disk.
class S3Disk {
  async put(path, contents) { /* upload to S3 */ }
  async get(path) { /* download from S3 */ }
  url(path) { /* return signed URL */ }
  async delete(path) { /* delete from S3 */ }
}

Storage.registerDisk('s3', new S3Disk())
```

## Image manipulation

If the `sharp` library is available (it's an optional peer of `foobarjs`),
you can manipulate images:

```js
import { Storage } from 'foobarjs/storage'

await Storage.image('photos/photo.jpg')
  .resize(200, 200)
  .save('photos/thumbnail.jpg')
```

## See also

- [Conventions](./conventions.md#storage)
- [Configuration](./configuration.md)
