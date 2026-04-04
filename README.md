# ScribbleHub Novel To EPUB Automator

A sleek, headless automation engine wrapped in a beautifully styled frontend. Paste any **ScribbleHub** novel link and this tool will immediately extract the novel's core data, scrape every chapter, build a complete `.epub` file natively compatible with QuickNovel, and securely push the generated E-book straight to your designated Gmail inbox via SMTP.

## Features

- **Blazing Fast**: Directly polls WordPress AJAX core endpoints, instantly returning chapter logic without fully parsing heavy site over-head.
- **Auto-Repairing HTML Parsing**: Powered by `lxml`, aggressively fixing messy ScribbleHub un-closed text paragraphs, preventing data duplication.
- **Idempotent Resume**: Interrupted connection? Power tripped? The tool stores data by `Post ID` and caches existing sections using `skip_existing=True`. It only scrapes brand-new chapters, instantly piecing them into your updated EPUB!
- **QuickNovel Perfect Support**: Injects description data into standard metadata and specifically generates a **Preface Chapter** strictly to render plot hooks gracefully inside mobile readers. 
- **Dark-Mode Glass UI**: Modern gradient visuals bundled with a robust frontend that seamlessly polls updates natively from the background Python Thread.
- **E-Book Emailing**: Securely routes the finished file right into your Gmail library using SMTP App-Passwords.

## Installation & Setup

1. **Clone & Virtual Environment:**
   ```bash
   git clone <your-repo>
   cd scrapers/scribblehub
   
   python -m venv .venv
   
   # Windows:
   source .venv/Scripts/activate
   # Linux/macOS:
   source .venv/bin/activate
   ```

2. **Install Dependencies:**
   Ensure you have the required packages installed:
   ```bash
   pip install requests beautifulsoup4 cloudscraper ebooklib markdown fastapi "uvicorn[standard]" pydantic python-dotenv python-multipart lxml
   ```

3. **Configure Environment Secrets:**
   Create a `.env` file inside the `scrapers/scribblehub` directory and add your customized settings:
   ```env
   GMAIL_USER=your-email@gmail.com
   GMAIL_APP_PASSWORD=xxxx xxxx xxxx xxxx
   ```
   > You must generate an App Password from Google: *Manage Your Google Account -> Security -> 2-Step Verification -> App Passwords*.

## Running The Application

1. Spin up the FastAPI background server:
   ```bash
   uvicorn main:app --reload
   ```
2. Navigate your web browser to `http://localhost:8000`.
3. Paste a ScribbleHub novel URL and an email address in the fields.
4. Watch the progress bar orchestrate your local magic and check your email inbox!

## Security & Privacy
- Make sure to keep `.env` excluded from source control (It is specified thoroughly in the included `.gitignore`). All secrets are dynamically read via `os.getenv()`. No hardcoded strings exist inside the repository.
