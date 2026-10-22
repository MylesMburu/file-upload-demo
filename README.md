# File Upload Demo (Express + Multer)

A small Node.js server that accepts image uploads, saves them to disk, and gives back a URL you can open in the browser.

If you have never built file uploads before, this README is for you. It covers:

1. [What you need installed](#1-prerequisites)
2. [How to set up and run the project](#2-setup)
3. [How file uploads work on the web](#3-how-file-uploads-work-the-big-picture), before any code
4. [What Multer is and why we need it](#4-what-is-multer)
5. [A line-by-line walkthrough of the code](#5-code-walkthrough)
6. [How to test uploads](#6-testing-the-endpoints) with Postman, curl, and a plain HTML form
7. [What the errors mean](#7-errors-you-might-see-and-what-they-mean)
8. [Common mistakes](#8-common-mistakes-and-gotchas)
9. [Exercises to try](#9-exercises-to-try-next)

---

## 1. Prerequisites

| Tool | Why you need it | How to check |
|------|-----------------|--------------|
| **Node.js** (v18 or newer; v20+ recommended) | Runs the JavaScript server | `node -v` |
| **npm** (comes with Node) | Installs the packages | `npm -v` |
| **A code editor** (VS Code is a good choice) | To read and edit the code | — |
| **Postman** or **curl** (optional but helpful) | To send test uploads | `curl --version` |

If `node -v` gives an error, install Node from <https://nodejs.org> (choose the LTS version), then close and reopen your terminal.

---

## 2. Setup

### Step 1: Get the code

Clone the repository (or download and unzip it), then open a terminal **inside the project folder**:

```bash
git clone <repo-url>
cd file-upload-demo
```

### Step 2: Install dependencies

```bash
npm install
```

This reads `package.json` and downloads everything into a `node_modules/` folder. The packages are:

| Package | What it does |
|---------|--------------|
| `express` | The web framework: creates the server and routes |
| `multer` | Reads uploaded files out of incoming requests (the main topic of this repo) |
| `cors` | Lets a frontend on a different address (for example a React app on port 3000) call this server |
| `dotenv` | Loads settings such as `PORT` from a `.env` file |
| `nodemon` (dev only) | Restarts the server automatically every time you save a file |

### Step 3: Make sure the `uploads/` folder exists

Uploaded files are saved into a folder called `uploads/` in the project root.

> ⚠️ **Multer will not create this folder for you** (in the way this project is configured). If it is missing, every upload fails with an error like `ENOENT: no such file or directory`.

Check that it exists, and create it if it doesn't:

```bash
mkdir uploads
```

(If you get "already exists", that's fine.)

### Step 4 (optional): Create a `.env` file

The server runs on port **5000** by default. To use a different port, create a file named `.env` in the project root:

```env
PORT=4000
```

`require('dotenv').config()` in `server.js` reads this file and puts the value into `process.env.PORT`.

### Step 5: Start the server

```bash
npm run dev
```

You should see:

```
Server running on http://localhost:5000
```

Open <http://localhost:5000> in your browser. You should see:

```json
{ "message": "File upload server is running" }
```

🎉 The server is running. To stop it, press `Ctrl + C` in the terminal.

> 💡 Always start the server **from the project root folder**. The upload path `'uploads/'` is *relative*, so it is resolved from the folder you ran the command in. Start the server from somewhere else and Multer will look for an `uploads/` folder that isn't there.

---

## 3. How file uploads work (the big picture)

Before looking at Multer, it helps to know what actually gets sent over the network when you upload a file.

### Normal requests vs. file uploads

Most API requests you've written send **JSON**:

```
POST /api/users
Content-Type: application/json

{ "name": "Amina", "email": "amina@example.com" }
```

JSON is text. A file such as a photo is **binary data** (raw bytes), and JSON can't carry raw bytes well. Browsers therefore use a special request format for uploads, called **`multipart/form-data`**.

### What `multipart/form-data` looks like

"Multipart" means the request body is split into several **parts**, one per form field. A random string called the **boundary** separates the parts. Here is a simplified version of what the browser actually sends:

```
POST /api/upload/avatar
Content-Type: multipart/form-data; boundary=----XyZ123

------XyZ123
Content-Disposition: form-data; name="username"

amina
------XyZ123
Content-Disposition: form-data; name="avatar"; filename="me.png"
Content-Type: image/png

‰PNG....(thousands of raw binary bytes of the image)....
------XyZ123--
```

Notice that each part has:

- a **field name** (`name="avatar"`), which is the "key", like a key in JSON
- for files, the **original filename** (`filename="me.png"`) and a **type** (`Content-Type: image/png`)
- the actual **content** (text, or the file's bytes)

### The problem

Express's built-in body parsers, `express.json()` and `express.urlencoded()`, **don't understand `multipart/form-data`**. If you send a file to a plain Express route, `req.body` is empty and the file is nowhere to be found.

Something has to:

1. read the incoming stream of bytes,
2. split it on the boundary,
3. save the file parts somewhere (disk or memory),
4. put the text fields into `req.body`,
5. put information about the files into `req.file` / `req.files`.

**That's Multer's job.**

### The full journey of an upload in this project

```
 ┌──────────────┐  multipart/form-data   ┌──────────────────────────────────────────┐
 │   Browser /  │ ─────────────────────▶ │  Express server                          │
 │   Postman    │                        │                                          │
 └──────────────┘                        │  1. Route matches  POST /api/upload/avatar│
        ▲                                │  2. Multer middleware runs:               │
        │                                │       • fileFilter  → is it an image?     │
        │                                │       • limits      → is it ≤ 5 MB?       │
        │                                │       • storage     → save to uploads/    │
        │                                │       • sets req.file                     │
        │                                │  3. Your route handler runs:              │
        │       JSON with file URL       │       • builds a URL                      │
        └─────────────────────────────── │       • res.json({...})                   │
                                         └──────────────────────────────────────────┘

 Later: GET /uploads/avatar-123.png  →  express.static serves the file from disk
```

---

## 4. What is Multer?

[Multer](https://github.com/expressjs/multer) is **Express middleware** for handling `multipart/form-data`.

Quick refresher: **middleware** is a function that runs *before* your route handler. It can read or modify `req` and `res`, then call `next()` to pass control along. `express.json()` is middleware too.

When you write:

```js
router.post('/avatar', upload.single('avatar'), (req, res) => { ... });
//                     └────── middleware ─────┘  └─── your handler ───┘
```

Multer runs first, processes the file, and **only then** calls your handler. By the time your code runs, the file is already saved on disk and its details are in `req.file`.

### Multer's three main settings

When you create a Multer instance you give it an options object. This project uses three options:

| Option | Question it answers | In this project |
|--------|---------------------|-----------------|
| `storage` | **Where** do files go, and **what are they called**? | Save to disk in `uploads/` with a unique name |
| `fileFilter` | **Which** files are allowed? | Only JPEG, PNG, GIF, and WebP images |
| `limits` | **How big** (or how many) can they be? | Max 5 MB per file |

### Multer's upload methods

After you create an instance (`const upload = multer({...})`), you choose a method based on what your form sends:

| Method | Use it when | Result is in |
|--------|-------------|--------------|
| `upload.single('avatar')` | One file in a field called `avatar` | `req.file` (an object) |
| `upload.array('photos', 5)` | Up to 5 files, all in a field called `photos` | `req.files` (an array) |
| `upload.fields([{ name: 'avatar', maxCount: 1 }, { name: 'gallery', maxCount: 8 }])` | Several different file fields | `req.files.avatar`, `req.files.gallery` |
| `upload.none()` | A multipart form with text fields only, no files | `req.body` only |
| `upload.any()` | Any files in any field (use with care) | `req.files` |

This project uses `single` and `array`.

> 🔑 **The field name matters.** The string you pass (`'avatar'`, `'photos'`) **must exactly match** the field name the client sends. If the frontend sends the file as `image` but the server expects `avatar`, Multer rejects it with `LIMIT_UNEXPECTED_FILE`. This is the most common beginner bug.

---

## 5. Code walkthrough

### Project structure

```
file-upload-demo/
├── config/
│   └── upload.js      ← Multer configuration (storage, filter, limits)
├── routes/
│   └── upload.js      ← The upload endpoints (/avatar and /gallery)
├── uploads/           ← Uploaded files end up here (.gitkeep keeps the folder in Git)
├── .gitignore         ← Keeps node_modules, .env and uploaded files out of Git
├── server.js          ← App entry point: sets up Express, static files, error handling
├── package.json       ← Project info, scripts, and dependencies
└── .env               ← (optional, you create it) PORT setting
```

Why split the files? `config/upload.js` defines **how** uploads are handled, once. Any route file can then `require` it and reuse the same rules.

---

### `config/upload.js`: configuring Multer

#### Part 1: Storage (where and what name)

```js
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, 'uploads/');
  },
  filename: function (req, file, cb) {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    const ext = path.extname(file.originalname);
    cb(null, file.fieldname + '-' + uniqueSuffix + ext);
  }
});
```

`multer.diskStorage()` saves files to your hard drive. It takes two functions, and Multer calls both of them **once for every uploaded file**.

**`destination`** decides which folder the file goes into.
- `cb` stands for "callback". You call it to give your answer back to Multer.
- The first argument to `cb` is always an **error** (`null` means "no error"). The second is your answer.
- So `cb(null, 'uploads/')` means: "No error. Put it in `uploads/`."

**`filename`** decides what to call the saved file.
- **Why not keep the original name?** Two users could both upload `photo.jpg`, and the second file would overwrite the first. Original names can also contain spaces, odd characters, or even path tricks like `../../server.js`.
- `Date.now()` is the current time in milliseconds, e.g. `1792400693007`.
- `Math.round(Math.random() * 1E9)` is a random number up to 1 billion, e.g. `993722843`. It makes clashes practically impossible even if two uploads arrive in the same millisecond.
- `path.extname(file.originalname)` takes the extension from the original name, e.g. `.png`.
- `file.fieldname` is the form field name, e.g. `avatar`.
- **Result:** `avatar-1792400693007-993722843.png` (you can see a real one in the `uploads/` folder).

The `file` object Multer passes in looks like this at this stage:

```js
{
  fieldname: 'avatar',          // the form field name
  originalname: 'me.png',       // name on the user's computer
  encoding: '7bit',
  mimetype: 'image/png'         // the type the client says the file is
}
```

> 💡 **Alternative: `multer.memoryStorage()`** keeps the file in RAM as a `Buffer` (`req.file.buffer`) instead of writing it to disk. Use it when you're about to send the file straight to a cloud service (Cloudinary, AWS S3, etc.) and don't need a local copy.

#### Part 2: File filter (which files are allowed)

```js
const fileFilter = (req, file, cb) => {
  const allowedTypes = ['image/jpeg', 'image/jpg', 'image/png', 'image/gif', 'image/webp'];

  if (allowedTypes.includes(file.mimetype)) {
    cb(null, true);    // ✅ accept the file
  } else {
    cb(new Error('Invalid file type. Only JPEG, PNG, GIF, and WebP images are allowed.'), false); // ❌ reject
  }
};
```

- A **MIME type** is a standard label for a type of file: `image/png`, `application/pdf`, `text/plain`, and so on.
- `cb(null, true)` → accept the file.
- `cb(null, false)` → skip the file **silently** (no error; the file simply isn't saved).
- `cb(new Error(...))` → reject with an error. Express passes this error to the error handler in `server.js`, which returns a `400` response.

The filter runs **before** the file is saved, so rejected files never touch the disk.

> ⚠️ **Security note:** `file.mimetype` comes from the **client**. Someone could rename `virus.exe` to `cat.png` and send it with `Content-Type: image/png`. For a learning project this check is fine. In production you'd also inspect the file's actual bytes (the [`file-type`](https://www.npmjs.com/package/file-type) package can do this) and never execute or trust uploaded content.

#### Part 3: Limits, and putting it together

```js
const upload = multer({
  storage: storage,
  fileFilter: fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5 MB max
  }
});

module.exports = upload;
```

- File sizes are measured in **bytes**. 1 KB = 1024 bytes and 1 MB = 1024 × 1024 bytes, so `5 * 1024 * 1024` = 5 MB.
- If a file goes over the limit, Multer stops reading it and raises a `MulterError` with code `LIMIT_FILE_SIZE`.
- Without limits, anyone could upload a 50 GB file and fill up your server's disk. **Always set limits.**
- Other useful limits include `files` (max number of files) and `fields` (max number of text fields).

We export the configured `upload` object so routes can use it.

---

### `routes/upload.js`: the endpoints

#### Single file upload: `POST /api/upload/avatar`

```js
router.post('/avatar', upload.single('avatar'), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' });
  }

  const fileUrl = `${req.protocol}://${req.get('host')}/uploads/${req.file.filename}`;

  res.json({
    message: 'Avatar uploaded successfully',
    file: {
      originalName: req.file.originalname,
      filename: req.file.filename,
      size: req.file.size,
      mimetype: req.file.mimetype,
      url: fileUrl,
    }
  });
});
```

Step by step:

1. `upload.single('avatar')` tells Multer: "expect **one** file in a field named **`avatar`**."
2. When your handler runs, the file is **already saved**. `req.file` contains:

   ```js
   {
     fieldname: 'avatar',
     originalname: 'me.png',
     encoding: '7bit',
     mimetype: 'image/png',
     destination: 'uploads/',
     filename: 'avatar-1792400693007-993722843.png',   // the name we generated
     path: 'uploads/avatar-1792400693007-993722843.png',
     size: 59368                                      // in bytes
   }
   ```

3. `if (!req.file)`: if the client didn't send a file at all, Multer doesn't raise an error. `req.file` is just `undefined`, so we check for it ourselves.
4. We build a **public URL** for the file:
   - `req.protocol` → `http`
   - `req.get('host')` → `localhost:5000`
   - Result: `http://localhost:5000/uploads/avatar-1792400693007-993722843.png`
5. We send back JSON describing the upload. A real app would usually **save `fileUrl` (or `filename`) in a database**, for example on the user's profile, so you can show it later.

> 💡 Text fields sent in the same form (e.g. `username`) are available in `req.body.username`. Multer fills `req.body` too.

#### Multiple file upload: `POST /api/upload/gallery`

```js
router.post('/gallery', upload.array('photos', 5), (req, res) => {
  if (!req.files || req.files.length === 0) {
    return res.status(400).json({ error: 'No files uploaded' });
  }

  const files = req.files.map(file => ({
    originalName: file.originalname,
    filename: file.filename,
    size: file.size,
    url: `${req.protocol}://${req.get('host')}/uploads/${file.filename}`,
  }));

  res.json({
    message: `${files.length} file(s) uploaded successfully`,
    files: files,
  });
});
```

- `upload.array('photos', 5)` means "accept **up to 5** files, all sent under the field name **`photos`**."
- Notice it's **`req.files`** (plural, an array) here, not `req.file`.
- If the client sends a 6th file, Multer raises `LIMIT_UNEXPECTED_FILE`.
- `fileFilter` and the 5 MB limit apply to **each** file.
- We use `.map()` to turn each file object into a URL and some info.

---

### `server.js`: wiring it all together

```js
const express = require('express');
const cors = require('cors');
const path = require('path');
require('dotenv').config();      // load .env into process.env
const multer = require('multer');

const app = express();
const PORT = process.env.PORT || 5000;
```

Standard setup. `multer` is imported here only so the error handler can recognize `multer.MulterError`.

#### Global middleware, then the routes

```js
app.use(cors());
app.use(express.json());

const uploadRoutes = require('./routes/upload');
app.use('/api/upload', uploadRoutes);
```

**Order matters in Express.** Middleware only applies to routes registered *after* it. That's why `cors()` comes first. If the routes were mounted above it, a frontend on another port would get CORS errors when it tried to upload.

Every route in `routes/upload.js` gets the `/api/upload` prefix:
- `router.post('/avatar')` → `POST /api/upload/avatar`
- `router.post('/gallery')` → `POST /api/upload/gallery`

#### Serving uploaded files

```js
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));
```

Saving a file to disk doesn't make it reachable in a browser by itself. `express.static` makes it reachable: any request to `/uploads/<filename>` is answered by sending that file from the `uploads/` folder.

- `__dirname` is the folder `server.js` lives in, so `path.join(__dirname, 'uploads')` is the absolute path to the `uploads/` folder.
- This line is what makes the `url` in the upload response work.

#### The error handler

```js
app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError) {
    if (err.code === 'LIMIT_FILE_SIZE') {
      return res.status(400).json({ error: 'File too large. Maximum size is 5 MB.' });
    }
    if (err.code === 'LIMIT_UNEXPECTED_FILE') {
      return res.status(400).json({ error: 'Too many files or unexpected field name.' });
    }
    return res.status(400).json({ error: err.message });
  }

  if (err.message && err.message.includes('Invalid file type')) {
    return res.status(400).json({ error: err.message });
  }

  next(err);
});
```

- Express recognizes an **error-handling middleware** because it has **four** parameters: `(err, req, res, next)`.
- When Multer fails (file too big, wrong field, etc.), it calls `next(err)` and Express skips straight to this function.
- Without this handler, the user would get an ugly HTML error page with a `500` status. With it, they get a clear JSON message with a `400` status ("your request was wrong").
- `multer.MulterError` covers Multer's built-in errors (size, count, field name).
- The "Invalid file type" check catches **our own** error from `fileFilter`. That error is a plain `Error`, not a `MulterError`, so it needs its own check.
- Anything else is passed on with `next(err)` to Express's default error handler.
- **The error handler must be registered after the routes**, otherwise errors from those routes never reach it.

---

## 6. Testing the endpoints

Start the server first (`npm run dev`). The examples below assume port 5000.

### Option A: Postman (easiest)

**Single upload**
1. Create a new request: method **POST**, URL `http://localhost:5000/api/upload/avatar`
2. Open the **Body** tab and choose **form-data**
3. In the **Key** column type `avatar`, then hover over the key cell and change the dropdown from **Text** to **File**
4. In the **Value** column click **Select Files** and pick an image
5. Click **Send**

> ⚠️ **Don't set the `Content-Type` header yourself.** Postman (and browsers) set it automatically, *including the boundary*. If you set it manually, the boundary is missing and Multer can't read the body.

**Multiple upload**
1. **POST** `http://localhost:5000/api/upload/gallery`
2. Body → form-data → key `photos` (type **File**)
3. Select several images for that one key (or add several rows, all with the key `photos`)
4. Send

### Option B: curl (terminal)

Run these from the project folder, replacing `me.png` with the path to a real image:

```bash
# Single file
curl -F "avatar=@me.png" http://localhost:5000/api/upload/avatar

# Multiple files (same field name repeated)
curl -F "photos=@cat.jpg" -F "photos=@dog.png" http://localhost:5000/api/upload/gallery
```

- `-F` sends a `multipart/form-data` field
- The `@` tells curl "read this from a file" (without it, curl sends the literal text)

> 🪟 On Windows PowerShell, `curl` may be an alias for a different command. Use `curl.exe` instead, or run these in Git Bash.

### Option C: a plain HTML form

Create a file `test.html` anywhere and open it in your browser:

```html
<!DOCTYPE html>
<html>
  <body>
    <h2>Single avatar</h2>
    <form action="http://localhost:5000/api/upload/avatar" method="POST" enctype="multipart/form-data">
      <input type="file" name="avatar" accept="image/*" />
      <button type="submit">Upload</button>
    </form>

    <h2>Gallery (up to 5)</h2>
    <form action="http://localhost:5000/api/upload/gallery" method="POST" enctype="multipart/form-data">
      <input type="file" name="photos" accept="image/*" multiple />
      <button type="submit">Upload</button>
    </form>
  </body>
</html>
```

Three things to notice:
- `enctype="multipart/form-data"` is **required**. Without it, the browser only sends the file *name*, not its contents.
- The input's `name` (`avatar` / `photos`) matches what Multer expects.
- `multiple` lets the user select several files.

### Option D: from JavaScript (e.g. React) with `fetch`

```js
const input = document.querySelector('input[type="file"]');

const formData = new FormData();
formData.append('avatar', input.files[0]);   // key must match upload.single('avatar')

const res = await fetch('http://localhost:5000/api/upload/avatar', {
  method: 'POST',
  body: formData,          // ⚠️ do NOT set a Content-Type header; fetch does it for you
});
const data = await res.json();
console.log(data.file.url);
```

For multiple files:

```js
const formData = new FormData();
for (const file of input.files) {
  formData.append('photos', file);
}
```

### What a successful response looks like

```json
{
  "message": "Avatar uploaded successfully",
  "file": {
    "originalName": "me.png",
    "filename": "avatar-1792650633169-728005832.png",
    "size": 59368,
    "mimetype": "image/png",
    "url": "http://localhost:5000/uploads/avatar-1792650633169-728005832.png"
  }
}
```

Copy the `url` into your browser and you'll see your image. You'll also find the file in the `uploads/` folder.

---

## 7. Errors you might see (and what they mean)

| You did this | Status | Response | Why |
|--------------|--------|----------|-----|
| Uploaded a `.txt` / `.pdf` / non-image | 400 | `Invalid file type. Only JPEG, PNG, GIF, and WebP images are allowed.` | `fileFilter` rejected it |
| Uploaded an image larger than 5 MB | 400 | `File too large. Maximum size is 5 MB.` | `limits.fileSize` exceeded |
| Used the wrong field name (e.g. `photo` instead of `avatar`) | 400 | `Too many files or unexpected field name.` | Multer only accepts the field you named in `single()`/`array()` |
| Sent 6+ files to `/gallery` | 400 | `Too many files or unexpected field name.` | `upload.array('photos', 5)` allows a maximum of 5 |
| Sent the request with no file | 400 | `No file uploaded` | Our own `if (!req.file)` check |
| `uploads/` folder doesn't exist | 500 | `ENOENT: no such file or directory` | Create the folder (see [Setup step 3](#step-3-make-sure-the-uploads-folder-exists)) |

---

## 8. Common mistakes and gotchas

1. **Field name mismatch.** The name in your form or `FormData` must exactly match the one in `upload.single('...')` / `upload.array('...')`. It's case-sensitive.
2. **Forgetting `enctype="multipart/form-data"`** on HTML forms.
3. **Setting `Content-Type` manually** in `fetch`/axios/Postman. Let the client set it so the boundary is included.
4. **Looking for the file in `req.body`.** Files go in `req.file` (single) or `req.files` (array). Only text fields go in `req.body`.
5. **Mixing up `req.file` and `req.files`.** `single()` → `req.file`; `array()` / `fields()` → `req.files`.
6. **Missing `uploads/` folder**, or starting the server from the wrong directory.
7. **Error handler registered before the routes.** It has to come after them.
8. **Committing uploaded files to Git.** User uploads usually shouldn't be in your repo. This project's `.gitignore` already handles it:
   ```gitignore
   node_modules/
   .env
   uploads/*
   !uploads/.gitkeep
   ```
   The empty `uploads/.gitkeep` file is there so Git still tracks the (otherwise empty) folder. Git doesn't save empty folders.
9. **Storing files on the server in production.** Many hosts (Render, Railway, Heroku, Vercel) have *temporary* disks that are wiped on every deploy or restart. Real apps usually upload to cloud storage such as **Cloudinary** or **AWS S3** and save only the URL in the database.

---

## 9. Exercises to try next

Work through these to check you understand how the pieces fit together:

1. **Change the size limit** to 2 MB. Remember to update the error message in `server.js` too.
2. **Allow PDFs** on a new route `POST /api/upload/document` with its own Multer config that only accepts `application/pdf`.
3. **Use `upload.fields()`** to make a `POST /api/upload/profile` route that accepts one `avatar` *and* up to 3 `photos` in the same request.
4. **Accept text fields** with the file, e.g. a `caption`, and include `req.body.caption` in the response.
5. **Add a delete route** `DELETE /api/upload/:filename` that removes a file using `fs.unlink`. (Watch out: make sure no one can pass `../` to delete files outside `uploads/`!)
6. **Add a list route** `GET /api/upload` that returns every file in `uploads/` using `fs.readdir`.
7. **Create the folder automatically** at startup with `fs.mkdirSync('uploads', { recursive: true })`.
8. **Build a small frontend** with an image preview before upload (look up `URL.createObjectURL`).
9. **Switch to `memoryStorage()`** and upload the buffer to Cloudinary instead of saving to disk.

---

## Quick reference

| Endpoint | Method | Field name | Max files | Result in |
|----------|--------|-----------|-----------|-----------|
| `/` | GET | — | — | Health check |
| `/api/upload/avatar` | POST | `avatar` | 1 | `req.file` |
| `/api/upload/gallery` | POST | `photos` | 5 | `req.files` |
| `/uploads/<filename>` | GET | — | — | Serves the saved file |

**Allowed types:** JPEG, PNG, GIF, WebP  **Max size:** 5 MB per file

### Useful links

- Multer documentation: <https://github.com/expressjs/multer>
- Express static files: <https://expressjs.com/en/starter/static-files.html>
- MDN: Using FormData: <https://developer.mozilla.org/en-US/docs/Web/API/FormData/Using_FormData_Objects>
- MDN: MIME types: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types>
