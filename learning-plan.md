# 5-Step Learning Plan: Build a Text-Based CRUD Web App in Rust with SQLite

This guide provides a hands-on, learn-by-doing approach to building a text-based web application in Rust, featuring a CRUD (Create, Read, Update, Delete) interface using the Rocket framework and SQLite database. It includes practical exercises and detailed instructions for hosting on a Linux server. The plan assumes basic programming knowledge but minimal Rust experience and should take ~3-4 weeks to complete.

## Step 1: Master Rust Basics for Text Manipulation

**Goal**: Learn Rust’s core concepts (ownership, borrowing, structs, and error handling) through exercises focused on text processing, essential for a CRUD app.

**Lessons**:

- **Ownership and Borrowing**: Understand how Rust manages memory for strings.
- **Structs and Enums**: Define data structures for text notes.
- **Error Handling**: Use `Result` and `Option` for safe text operations.

**Exercises**:

1. Write a program to split a string into words and count them, exploring `String` vs `&str`:

   ```rust
   fn count_words(input: &str) -> usize {
       input.split_whitespace().count()
   }
   fn main() {
       let text = "Hello Rust CRUD App";
       println!("Word count: {}", count_words(text));
   }
   ```

   Run in the [Rust Playground](https://play.rust-lang.org).
2. Create a `Note` struct with `id: i32` and `content: String`. Write a function to truncate notes longer than 50 characters.
3. Parse a JSON-like string (e.g., `{"id":1,"content":"test"}`) into a `Note` struct using `serde_json`, handling errors with `Result`.

**Time**: 3-5 days, ~1 hour daily.

---

## Step 2: Set Up Your Rust Environment

**Goal**: Configure your development environment and use Cargo to manage dependencies for web and database development.

**Lessons**:

- Install Rust using `rustup` and learn Cargo commands (`cargo new`, `cargo run`).
- Add dependencies for web (Rocket) and database (SQLx) development.
- Set up VS Code with Rust Analyzer for code completion and debugging.

**Exercises**:

1. Create a new Cargo project: `cargo new crud-app`.
2. Add dependencies to `Cargo.toml` for Rocket, SQLx, and JSON serialization:

   ```toml
   [dependencies]
   rocket = { version = "0.5", features = ["json"] }
   sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio-native-tls"] }
   serde = { version = "1.0", features = ["derive"] }
   tokio = { version = "1.0", features = ["full"] }
   ```

3. Write a program that connects to an in-memory SQLite database and creates a `notes` table:

   ```rust
   use sqlx::{SqlitePool, Result};
   async fn setup_db() -> Result<SqlitePool> {
       let pool = SqlitePool::connect("sqlite::memory:").await?;
       sqlx::query("CREATE TABLE notes (id INTEGER PRIMARY KEY, content TEXT NOT NULL)")
           .execute(&pool)
           .await?;
       Ok(pool)
   }
   #[tokio::main]
   async fn main() -> Result<()> {
       let db = setup_db().await?;
       println!("Database setup complete!");
       Ok(())
   }
   ```

   Run with `cargo run` and verify it executes without errors.

**Time**: 2-3 days.

---

## Step 3: Build a CRUD Web Server with Rocket

**Goal**: Use Rocket to create a web server with CRUD endpoints for managing text notes, integrating with SQLite.

**Lessons**:

- Set up Rocket routes for GET, POST, PUT, and DELETE operations.
- Use SQLx to perform CRUD operations on the `notes` table.
- Handle JSON input/output with `serde`.

**Exercises**:

1. Create a Rocket server with a `/notes` GET endpoint to list all notes:

   ```rust
   #[macro_use] extern crate rocket;
   use rocket::serde::{json::Json, Serialize, Deserialize};
   use rocket::State;
   use sqlx::SqlitePool;

   #[derive(Serialize, Deserialize)]
   struct Note {
       id: i32,
       content: String,
   }

   #[get("/notes")]
   async fn list_notes(db: &State<SqlitePool>) -> Json<Vec<Note>> {
       let notes = sqlx::query_as!(Note, "SELECT id, content FROM notes")
           .fetch_all(db)
           .await
           .unwrap_or_default();
       Json(notes)
   }
   ```

2. Add CRUD endpoints: `/notes` (POST to create), `/notes/<id>` (GET to read, PUT to update, DELETE to delete). Example POST:

   ```rust
   #[post("/notes", data = "<note>")]
   async fn create_note(note: Json<Note>, db: &State<SqlitePool>) -> Result<Json<Note>, &'static str> {
       if note.content.is_empty() {
           return Err("Content cannot be empty");
       }
       sqlx::query!("INSERT INTO notes (id, content) VALUES (?, ?)", note.id, note.content)
           .execute(db)
           .await
           .map_err(|_| "Failed to create note")?;
       Ok(note)
   }
   ```

3. Set up the Rocket server with database connection:

   ```rust
   #[launch]
   async fn rocket() -> _ {
       let pool = setup_db().await.expect("Failed to setup database");
       rocket::build()
           .manage(pool)
           .mount("/", routes![list_notes, create_note])
   }
   ```

   Test with `curl http://localhost:8000/notes` and `curl -X POST -H "Content-Type: application/json" -d '{"id":1,"content":"My note"}' http://localhost:8000/notes`.

**Time**: 5-7 days. Test each endpoint thoroughly.

---

## Step 4: Create a Simple CRUD Web Page

**Goal**: Add a basic HTML frontend to interact with your CRUD API, served by Rocket, to create a complete text-based web app.

**Lessons**:

- Serve static HTML with Rocket’s `NamedFile`.
- Write a simple HTML form for CRUD operations, using JavaScript to call your API.
- Ensure the page displays notes and handles user input.

**Exercises**:

1. Add a route to serve an HTML page at `/`:

   ```rust
   use rocket::fs::NamedFile;
   use std::path::{Path, PathBuf};

   #[get("/")]
   async fn index() -> Option<NamedFile> {
       NamedFile::open(Path::new("static/index.html")).await.ok()
   }
   ```

   Update the `rocket` function to include this route: `.mount("/", routes![index, list_notes, create_note])`.
2. Create a `static/index.html` file with a form and table for CRUD operations:

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <title>Notes CRUD App</title>
       <style>
           body { font-family: Arial, sans-serif; margin: 20px; }
           table { border-collapse: collapse; width: 100%; }
           th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
           th { background-color: #f2f2f2; }
           .error { color: red; }
       </style>
   </head>
   <body>
       <h1>Notes CRUD App</h1>
       <form id="note-form">
           <input type="number" id="id" placeholder="Note ID" required>
           <input type="text" id="content" placeholder="Note Content" required>
           <button type="submit">Create Note</button>
       </form>
       <div id="error" class="error"></div>
       <h2>Notes</h2>
       <table id="notes-table">
           <thead>
               <tr>
                   <th>ID</th>
                   <th>Content</th>
                   <th>Actions</th>
               </tr>
           </thead>
           <tbody></tbody>
       </table>

       <script>
           async function fetchNotes() {
               const response = await fetch('/notes');
               const notes = await response.json();
               const tbody = document.querySelector('#notes-table tbody');
               tbody.innerHTML = '';
               notes.forEach(note => {
                   const row = document.createElement('tr');
                   row.innerHTML = `
                       <td>${note.id}</td>
                       <td>${note.content}</td>
                       <td>
                           <button onclick="updateNote(${note.id})">Update</button>
                           <button onclick="deleteNote(${note.id})">Delete</button>
                       </td>
                   `;
                   tbody.appendChild(row);
               });
           }

           document.getElementById('note-form').addEventListener('submit', async (e) => {
               e.preventDefault();
               const id = document.getElementById('id').value;
               const content = document.getElementById('content').value;
               const errorDiv = document.getElementById('error');
               try {
                   const response = await fetch('/notes', {
                       method: 'POST',
                       headers: { 'Content-Type': 'application/json' },
                       body: JSON.stringify({ id: parseInt(id), content })
                   });
                   if (!response.ok) throw new Error(await response.text());
                   document.getElementById('note-form').reset();
                   errorDiv.textContent = '';
                   fetchNotes();
               } catch (err) {
                   errorDiv.textContent = err.message;
               }
           });

           async function updateNote(id) {
               const content = prompt('Enter new content:');
               if (content) {
                   await fetch(`/notes/${id}`, {
                       method: 'PUT',
                       headers: { 'Content-Type': 'application/json' },
                       body: JSON.stringify({ id, content })
                   });
                   fetchNotes();
               }
           }

           async function deleteNote(id) {
               await fetch(`/notes/${id}`, { method: 'DELETE' });
               fetchNotes();
           }

           fetchNotes();
       </script>
   </body>
   </html>
   ```

3. Create a `static` directory in your project root, save `index.html` there, and test by visiting `http://localhost:8000`. Add PUT and DELETE endpoints:

   ```rust
   #[put("/notes/<id>", data = "<note>")]
   async fn update_note(id: i32, note: Json<Note>, db: &State<SqlitePool>) -> Result<Json<Note>, &'static str> {
       if note.content.is_empty() {
           return Err("Content cannot be empty");
       }
       sqlx::query!("UPDATE notes SET content = ? WHERE id = ?", note.content, id)
           .execute(db)
           .await
           .map_err(|_| "Failed to update note")?;
       Ok(note)
   }

   #[delete("/notes/<id>")]
   async fn delete_note(id: i32, db: &State<SqlitePool>) -> Result<(), &'static str> {
       sqlx::query!("DELETE FROM notes WHERE id = ?", id)
           .execute(db)
           .await
           .map_err(|_| "Failed to delete note")?;
       Ok(())
   }
   ```

   Update the `rocket` function: `.mount("/", routes![index, list_notes, create_note, update_note, delete_note])`.

**Time**: 7-10 days. Focus on integrating the frontend and backend.

---

## Step 5: Deploy to a Linux Server

**Goal**: Deploy your CRUD app to a Linux server (e.g., Ubuntu) and add a note search feature to enhance functionality.

**Lessons**:

- Package your Rust app as a binary for Linux.
- Configure a Linux server with a reverse proxy (Nginx) and SQLite persistence.
- Implement a text search feature for notes.

**Instructions for Hosting on Linux**:

1. **Set Up a Linux Server**:
   - Use a cloud provider like DigitalOcean, AWS EC2, or Linode to provision an Ubuntu 22.04 server.
   - SSH into the server: `ssh user@your-server-ip`.
   - Update the system: `sudo apt update && sudo apt upgrade -y`.

2. **Install Dependencies**:
   - Install Rust: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`, then `source $HOME/.cargo/env`.
   - Install SQLite and build tools: `sudo apt install sqlite3 libsqlite3-dev build-essential -y`.
   - Install Nginx: `sudo apt install nginx -y`.

3. **Deploy the App**:
   - Clone your project from GitHub: `git clone https://github.com/your-repo/crud-app.git`.
   - Navigate to the project directory and build the release binary: `cd crud-app && cargo build --release`.
   - Create an SQLite database file: `sqlite3 notes.db "CREATE TABLE notes (id INTEGER PRIMARY KEY, content TEXT NOT NULL)"`.
   - Update your Rocket app to use the file-based database by changing the connection string in `setup_db`:

     ```rust
     async fn setup_db() -> Result<SqlitePool> {
         let pool = SqlitePool::connect("sqlite:notes.db").await?;
         sqlx::query("CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, content TEXT NOT NULL)")
             .execute(&pool)
             .await?;
         Ok(pool)
     }
     ```

   - Copy the `static` directory with `index.html` to the server.
   - Run the app: `./target/release/crud-app`. Test locally with `curl http://localhost:8000`.

4. **Configure Nginx as a Reverse Proxy**:
   - Create an Nginx config file: `sudo nano /etc/nginx/sites-available/crud-app`.
   - Add:

     ```nginx
     server {
         listen 80;
         server_name your-domain-or-ip;
         location / {
             proxy_pass http://localhost:8000;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
         }
     }
     ```

   - Enable the config: `sudo ln -s /etc/nginx/sites-available/crud-app /etc/nginx/sites-enabled/`.
   - Test and restart Nginx: `sudo nginx -t && sudo systemctl restart nginx`.

5. **Run the App as a Service**:
   - Create a systemd service file: `sudo nano /etc/systemd/system/crud-app.service`.
   - Add:

     ```ini
     [Unit]
     Description=Rust CRUD App
     After=network.target

     [Service]
     User=your-username
     WorkingDirectory=/home/your-username/crud-app
     ExecStart=/home/your-username/crud-app/target/release/crud-app
     Restart=always

     [Install]
     WantedBy=multi-user.target
     ```

   - Enable and start the service: `sudo systemctl enable crud-app && sudo systemctl start crud-app`.

6. **Verify Deployment**:
   - Visit `http://your-domain-or-ip` in a browser to see the CRUD page.
   - Test CRUD operations using the HTML form.

**Exercise**:

- Add a `/search/<query>` endpoint to search notes by content:

  ```rust
  #[get("/search/<query>")]
  async fn search_notes(query: &str, db: &State<SqlitePool>) -> Json<Vec<Note>> {
      let notes = sqlx::query_as!(Note, "SELECT id, content FROM notes WHERE content LIKE ?", format!("%{}%", query))
          .fetch_all(db)
          .await
          .unwrap_or_default();
      Json(notes)
  }
  ```

  Update the `rocket` function: `.mount("/", routes![index, list_notes, create_note, update_note, delete_note, search_notes])`.
- Modify `index.html` to add a search form:

  ```html
  <form id="search-form">
      <input type="text" id="search-query" placeholder="Search notes">
      <button type="submit">Search</button>
  </form>
  ```

  Add JavaScript to handle search:

  ```javascript
  document.getElementById('search-form').addEventListener('submit', async (e) => {
      e.preventDefault();
      const query = document.getElementById('search-query').value;
      const response = await fetch(`/search/${encodeURIComponent(query)}`);
      const notes = await response.json();
      const tbody = document.querySelector('#notes-table tbody');
      tbody.innerHTML = '';
      notes.forEach(note => {
          const row = document.createElement('tr');
          row.innerHTML = `<td>${note.id}</td><td>${note.content}</td><td><button onclick="updateNote(${note.id})">Update</button><button onclick="deleteNote(${note.id})">Delete</button></td>`;
          tbody.appendChild(row);
      });
  });
  ```

  Test by searching for notes containing specific words.

**Time**: 7-10 days. Spend time on deployment and testing the search feature.

---

## Complete Code Example

Below is the full Rust code for the CRUD app (excluding `index.html`, included in Step 4):

```rust
#[macro_use] extern crate rocket;
use rocket::serde::{json::Json, Serialize, Deserialize};
use rocket::State;
use rocket::fs::NamedFile;
use sqlx::{SqlitePool, Result};
use std::path::{Path, PathBuf};

#[derive(Serialize, Deserialize)]
struct Note {
    id: i32,
    content: String,
}

async fn setup_db() -> Result<SqlitePool> {
    let pool = SqlitePool::connect("sqlite:notes.db").await?;
    sqlx::query("CREATE TABLE IF NOT EXISTS notes (id INTEGER PRIMARY KEY, content TEXT NOT NULL)")
        .execute(&pool)
        .await?;
    Ok(pool)
}

#[get("/")]
async fn index() -> Option<NamedFile> {
    NamedFile::open(Path::new("static/index.html")).await.ok()
}

#[get("/notes")]
async fn list_notes(db: &State<SqlitePool>) -> Json<Vec<Note>> {
    let notes = sqlx::query_as!(Note, "SELECT id, content FROM notes")
        .fetch_all(db)
        .await
        .unwrap_or_default();
    Json(notes)
}

#[post("/notes", data = "<note>")]
async fn create_note(note: Json<Note>, db: &State<SqlitePool>) -> Result<Json<Note>, &'static str> {
    if note.content.is_empty() {
        return Err("Content cannot be empty");
    }
    sqlx::query!("INSERT INTO notes (id, content) VALUES (?, ?)", note.id, note.content)
        .execute(db)
        .await
        .map_err(|_| "Failed to create note")?;
    Ok(note)
}

#[put("/notes/<id>", data = "<note>")]
async fn update_note(id: i32, note: Json<Note>, db: &State<SqlitePool>) -> Result<Json<Note>, &'static str> {
    if note.content.is_empty() {
        return Err("Content cannot be empty");
    }
    sqlx::query!("UPDATE notes SET content = ? WHERE id = ?", note.content, id)
        .execute(db)
        .await
        .map_err(|_| "Failed to update note")?;
    Ok(note)
}

#[delete("/notes/<id>")]
async fn delete_note(id: i32, db: &State<SqlitePool>) -> Result<(), &'static str> {
    sqlx::query!("DELETE FROM notes WHERE id = ?", id)
        .execute(db)
        .await
        .map_err(|_| "Failed to delete note")?;
    Ok(())
}

#[get("/search/<query>")]
async fn search_notes(query: &str, db: &State<SqlitePool>) -> Json<Vec<Note>> {
    let notes = sqlx::query_as!(Note, "SELECT id, content FROM notes WHERE content LIKE ?", format!("%{}%", query))
        .fetch_all(db)
        .await
        .unwrap_or_default();
    Json(notes)
}

#[launch]
async fn rocket() -> _ {
    let pool = setup_db().await.expect("Failed to setup database");
    rocket::build()
        .manage(pool)
        .mount("/", routes![index, list_notes, create_note, update_note, delete_note, search_notes])
}
```

---

## Tips for Success

- **Test Locally**: Use `curl` or Postman to test API endpoints (e.g., `curl -X POST -H "Content-Type: application/json" -d '{"id":1,"content":"Test note"}' http://localhost:8000/notes`).
- **Debug**: Add `env_logger` (`[dependencies] env_logger = "0.10"`) and initialize it in `main.rs` for debugging.
- **Security**: Validate inputs to prevent SQL injection (SQLx sanitizes queries, but always check user input).
- **Linux Deployment**: Ensure your server has at least 1GB RAM for Rust compilation.
- **Resources**: Refer to [Rocket’s documentation](https://rocket.rs), [SQLx’s GitHub](https://github.com/launchbadge/sqlx), and [DigitalOcean tutorials](https://www.digitalocean.com/community) for Linux setup.

By the end of this plan, you’ll have a fully functional CRUD web app with a simple HTML interface, deployed on a Linux server. Happy coding!
