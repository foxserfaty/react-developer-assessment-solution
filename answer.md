# Section 1
## Question 1.1

```javascript
app.post('/api/notes', (req, res) => {
  try {
    const note = Note.create({
      title: req.body.title,
      content: req.body.content,
    });
    res.status(200).json(note);
  } catch (error) {
    console.log(error);
  }
});
```

I found 3 bugs in this snippet:
**Bug 1: No async for handler/await for function call.**
- Line: `app.post(...` and `const note = Note.create({...`
- User may experience: The endpoint returns a Promise object instead of the note -> the client receive something like `{}` or broken data; The note is probably still saved
- Fix: Make the handler `async` and `await` the call.

**Bug 2: Wrong status code**

- Line: `res.status(200).json(note);`
- 200 is OK, while here it should be 201 Created as we're creating a new note. It's for better clarity.
- Fix: Change `200` to `201`

**Bug 3: No error response**
- Line: `console.log(error);`
- In case of error, this handler would not send any response to indicate that. The user would probably be wondering what happened and there would be nothing in the response.
- Fix: add: `res.status(500).json({ error: 'There is a problem creating this note' });`

- Final fix:
```javascript
app.post('/api/notes', async (req, res) => {
  try {
    const note = await Note.create({
      title: req.body.title,
      content: req.body.content,
    });
    res.status(201).json(note);
  } catch (error) {
    console.log(error);
    res.status(500).json({ error: 'There is a problem creating this note' });
  }
});
```

## Question 1.2

```javascript
const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  age: String,
  createdAt: String,
});
```

- `name` should have `required: true`.
- `email` should have `unique: true` and `required: true`.
- `age` should be type Number.
- `createdAt` should be Date, but there's a better option by using `timestamp`. AFAIK MongoDB has `timestamp` built in to handle `createdAt` and `updatedAt`

- Updated:
```javascript
const userSchema = new mongoose.Schema(
  {
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    age: { type: Number },
  },
  { timestamps: true } 
);
```

## Question 1.3

The notes collection has 500,000 documents. This endpoint takes 8 seconds to respond:

```javascript
app.get('/api/notes/search', async (req, res) => {
  const notes = await Note.find({});
  const results = notes.filter((n) => n.title.toLowerCase().includes(req.query.q.toLowerCase()));
  res.json(results);
});
```

1. Why slow? 
- This snippet uses `Note.find({})` which pull the entire DB out instead of searching with filter, and that's half a million entry. It's taking up a lot of memory.
- It then proceed to filter using JS. Node.js is not made for filtering 500k entries. Looping over that much data is definitely going to be slow.

2. How to make it faster:
- Put the filtering in the query, not the js.
- I'm not entirely sure about the syntax though, but I'd imagine it being something like:
```javascript
const results = await Note.find({
  title: { query: req.query.q, option: 'i' }
});
```
- I do know `option: 'i'` means case-insensitive, so that's there for good measure.

# Section 2
## Question 2.1

```jsx
function NoteList() {
  const [notes, setNotes] = useState();

  useEffect(async () => {
    const res = await fetch('/api/notes');
    const data = await res.json();
    setNotes(data);
  }, []);

  function handleDelete(id) {
    fetch(`/api/notes/${id}`, { method: 'DELETE' });
    setNotes(notes.filter((n) => n.id !== id));
  }

  return (
    <ul>
      {notes.map((note) => (
        <li>
          {note.title}
          <button onClick={handleDelete(note._id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

I found 4:

**Bug 1: useState() missing initial value []**
- Line: `const [notes, setNotes] = useState();`
- What happens: `notes` will be `undefined`. As soon as page start rendering `notes.map(...)`, page will crash with "Cannot read properties of undefined (reading 'map')".
- Fix: `const [notes, setNotes] = useState([]);`

**Bug 2: onClick missing arrow function**
- Line: `<button onClick={handleDelete(note._id)}>Delete</button>`
- What happens: The function inside onClick would get called while rendering, thus deleting all notes on render. I'm not really sure why but this had happened to be before.
- Fix: Wrap in arrow function: `onClick={() => handleDelete(note._id)}`

**Bug 3: Delete is not async**
- Line: `fetch(...)`... `setNotes(...)`.
- What happens: For the most part, if the BE works just fine there would be no complications. But as soon as there's a problem deleting the note, FE would've been updated nonetheless of the result. Then, when the user refresh the page, the "deleted" notes just reappear.
- Fix: async for `handleDelete`, await the fetch.

**Bug 4? (more of a warning): <li> items doesn't have a `key` prop**

- Line: all `<li>` in the map
- What happens: React throws warnings about missing keys, could lead to problems with rendering/updating DOM elements.
- Fix: add `key={node._id}` to `<li>`.

Final fix:
```jsx
function NoteList() {
  const [notes, setNotes] = useState([]);

  useEffect(async () => {
    const res = await fetch('/api/notes');
    const data = await res.json();
    setNotes(data);
  }, []);

  async function handleDelete(id) {
    await fetch(`/api/notes/${id}`, { method: 'DELETE' });
    setNotes(notes.filter((n) => n.id !== id));
  }

  return (
    <ul>
      {notes.map((note) => (
        <li key={note._id}>
          {note.title}
          <button onClick={() => handleDelete(note._id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

# Section 3
## Question 3.1 - CORS error

Simply put, the web browser enforces a rule called Same-Origin Policy, where a page from one origin is not allowed to access responses from a different origin, UNLESS explicitly stated.

For this situation: FE React app is on port 3000, while Express BE is on port 5000 -> browser sees 2 different origin -> blocks the response.

### Fix: 
- Add `cors` package to Express

```javascript
const cors = require('cors');
app.use(cors({ origin: 'http://localhost:3000' }));
```

OR

- Use a proxy in `package.json` so it would show the same origin:
```json
"proxy": "http://localhost:5000"
```

## Question 3.2

```javascript
// Version A
function getUser(id) {
  return fetch(`/api/users/${id}`).then((res) => res.json());
}

// Version B
async function getUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}
```

They are doing the same thing, both runs fetch promise and return a JSON. Main different is probably readability.

I'm personally more used to Version B (dare I say I've only used that method). I'll try to explain version B:
- It's a clear and regular synchronous code, just read from the top to bottom: 
  - `res` is going to be the response of the `fetch` from `/api/users/${id}`
  - After fetching, return the parsed JSON from `res` with `res.json()`

# Section 4
## Question 4.1

```javascript
function requestLogger(res, req, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const dur = date.now() - start;
    console.log(`${req.method} ${req.url} - ${dur}ms`);
  })

  next();
}
```

Place before route definitions so it can see every requests:
```javascript
app.use(requestLogger);
```

## Question 4.2

```javascript
function groupByTag(bookmarks) {
  return bookmarks.reduce((acc, bookmark) => { // `reduce` can be used to build result in 1 go
    const key = bookmark.tag || 'untagged'; // if no tag, use 'untagged'
    if (!acc[key]) {
      acc[key] = [];
    }
    acc[key].push(bookmark);
    return acc;
  }, {}); // {} is the initial value, a blank array
}
```


