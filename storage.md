# Storage

foobarjs provides a filesystem abstraction for managing file uploads and storage.

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

## Basic Usage

```js
import { Storage } from 'foobarjs/queue'  // Available from 'foobarjs'

// Write a file
await Storage.put('avatars/user1.jpg', fileContents)

// Read a file
const contents = await Storage.get('avatars/user1.jpg')

// Get file URL
const url = Storage.url('avatars/user1.jpg')
// → /uploads/avatars/user1.jpg

// Delete a file
await Storage.delete('avatars/user1.jpg')
```

## Using a Specific Disk

```js
const disk = Storage.disk('local')
await disk.put('file.txt', 'content')
```

## Custom Disks

Register custom storage disks:

```js
import { Storage } from 'foobarjs/storage'

class S3Disk {
  async put(path, contents) { /* upload to S3 */ }
  async get(path) { /* download from S3 */ }
  url(path) { /* return signed URL */ }
  async delete(path) { /* delete from S3 */ }
}

Storage.registerDisk('s3', new S3Disk())
```

## Image Manipulation

If the `sharp` library is available, you can manipulate images:

```js
import { Storage } from 'foobarjs/storage'

const thumbnail = await Storage.image('photos/photo.jpg')
  .resize(200, 200)
  .toBuffer()
```
