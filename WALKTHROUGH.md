# PDF AI Summarizer: Walkthrough

A guide for trainers and students who are new to this code.

## 1. What this app does

The app takes a PDF, pulls out its text, and asks Google Gemini to summarize it. Long documents are split into chunks. Each chunk gets its own summary, and then the chunk summaries are merged into one final summary. You pick the model and the summary style in the sidebar. You can also turn on a short structure analysis and a list of key quotes, and download the results.

## 2. Where things are

| File | What it is responsible for |
|---|---|
| `app.py` | Streamlit UI: sidebar settings, file upload, the pipeline run, results and downloads |
| `prompts.py` | Every LLM instruction: the system message, the summary styles and four prompt templates |
| `summarizer.py` | The Gemini calls: chunk summaries, combine, structure analysis and quotes |
| `pdf_processor.py` | Reading the PDF, cleaning text, counting tokens and splitting text into chunks |
| `utils.py` | Showing results, building the text and CSV exports, the time estimate and the API key check |
| `requirements.txt` | The Python packages the app needs |
| `.env.example` | A template for the settings file (`GEMINI_API_KEY`, `GEMINI_MODEL`) |
| `README.md` | Setup, run steps and troubleshooting |
| `.env` | Your own copy of the settings, with your real key. Created by you. Never share it. |

## 3. How a PDF flows through it

1. **Upload.** `app.py` calls `st.file_uploader(...)`. The file is a Streamlit upload object.
2. **Metadata.** `app.py` calls `PDFProcessor.get_pdf_metadata()`. It uses `pypdf` to read the page count, title, author and subject, and shows them in four info boxes. This step does not call the LLM.
3. **Process.** The user clicks "Process PDF". That runs `process_pdf()` in `app.py`.
4. **Extract.** `PDFProcessor.extract_text_from_pdf()` uses `PdfReader` and `page.extract_text()` on each page. Pages are joined with blank lines. A page that fails is skipped and logged. If the whole file cannot be read, it raises `RuntimeError`.
5. **Clean.** `PDFProcessor.clean_text()` removes control characters, squeezes extra spaces and tabs, and limits blank lines to one. Punctuation stays.
6. **Count tokens.** `PDFProcessor.count_tokens()` uses the tiktoken `cl100k_base` encoding. This is a rough count. Gemini uses its own tokenizer. The count is shown to the user for information.
7. **Chunk.** `PDFProcessor.chunk_text(clean_text, max_tokens)` splits the text at sentence ends (`.`, `!`, `?`). A sentence longer than `max_tokens` is cut by tokens. The pieces are then packed into chunks, with each chunk kept under `max_tokens`. If there is no text at all, it raises `ValueError`, and the app shows the message.
8. **Estimate.** `utils.estimate_processing_time()` gives a rough time for the number of chunks.
9. **Analysis (optional).** If "Show Document Analysis" is on, `PDFSummarizer.analyze_document_structure()` sends only the first 3000 characters to the model. It returns a dict with `status` set to `success` or `error`.
10. **Summarize chunks.** `PDFSummarizer.summarize_chunks()` calls `chunk_chain.batch(...)` with `max_concurrency` set to 4. Up to four chunks run at once. Each result is turned into a dict with `chunk_number`, `summary`, `ok` and the two lengths. A chunk that fails gets the text "Error processing chunk" and `ok` set to `False`.
11. **Combine.** `PDFSummarizer.combine_summaries()` drops the failed chunks. If none are left, it returns "No valid summaries were generated." If only one is left, it returns that summary and makes no LLM call. Otherwise it runs `combine_chain` with the chosen style. If that call fails, the code returns the section summaries joined together.
12. **Quotes (optional).** If "Extract Key Quotes" is on, `extract_key_quotes()` looks at the first 5000 characters. It strips list numbers, bullets and quote marks from each line and keeps up to 10 lines. If this step fails, the app shows a warning and carries on.
13. **Store.** `process_pdf()` saves everything in `st.session_state.results`.
14. **Display.** `display_results()` shows the analysis, the quotes, the summary (from `format_summary_display()` in `utils.py`) and a statistics table (from `create_summary_dataframe()`).
15. **Download.** Two `st.download_button`s export the text report (`export_summary_to_text()`) and the CSV of chunk stats. A download click reruns the script. Because the results are in session state, `display_results()` draws them again.

## 4. Prompt templates explained

### How `prompts.py` is built

`prompts.py` has three parts.

The system message is shared by every prompt:

```python
SYSTEM = (
    "You are a precise document summarizer. Use only information in the provided "
    "text; never invent facts, numbers or names. ..."
)
```

`STYLES` maps each summary type to its instruction text:

```python
STYLES: dict[str, str] = {
    "brief": "2-3 sentences focusing on the main points and key takeaways.",
    "detailed": ("A detailed summary. Include:\n" ...),
    "bullet_points": "Clear, organized bullet points. ...",
    "executive": ("An executive summary suitable for business leaders. Include:\n" ...),
}
```

The `_template` helper builds a `ChatPromptTemplate` with the shared system message and a human message:

```python
def _template(human: str) -> ChatPromptTemplate:
    return ChatPromptTemplate.from_messages([("system", SYSTEM), ("human", human)])
```

There are four templates. Each one is built with `_template`:

| Template | Input variables | What it does |
|---|---|---|
| `CHUNK_SUMMARY` | `{style}`, `{text}` | Summarizes one chunk in the chosen style |
| `COMBINE` | `{style}`, `{summaries}` | Merges the section summaries into one, and removes repeats |
| `ANALYSIS` | `{text}` | Describes the document type, topics, sections, audience and purpose |
| `QUOTES` | `{text}` | Asks for 5-10 verbatim quotes, one per line |

### How a chain is built in `summarizer.py`

`PDFSummarizer.__init__` creates one `ChatGoogleGenerativeAI` object. It then builds each chain in the same shape, `prompt | llm | parser`:

```python
parser = RunnableLambda(lambda m: str(m.text))
self.chunk_chain = CHUNK_SUMMARY | self.llm | parser
```

The prompt fills in the variables. The LLM call returns a message. The parser turns that message into a plain string. The parser is needed because Gemini 3 returns content as a list of blocks.

### Worked example: add a "tweet" style

To add a `tweet` style, make two changes.

1. In `prompts.py`, add one entry to `STYLES`:

   ```python
   "tweet": "A single tweet of at most 280 characters with the main takeaway.",
   ```

2. In `app.py`, add `"tweet"` to the list passed to `st.sidebar.selectbox("Summary Type", [...])`.

No other code needs to change. The styles are passed in as `{style}`, and `summary_type.replace('_', ' ').title()` works for any name. Watch out for one trap. `STYLES.get(summary_type, STYLES["detailed"])` falls back to "detailed" without an error. A typo in a style name gives you a detailed summary with no warning.

## 5. What changed from the old OpenAI version

Each point shows the old code first, then the current code.

- **Model:** Before: `ChatOpenAI(model="gpt-4o-mini")` with `OPENAI_API_KEY`. Now: `ChatGoogleGenerativeAI` with `GEMINI_API_KEY`. The default is `gemini-3.1-flash-lite`. The sidebar lists three models, and `GEMINI_MODEL` in `.env` sets the default.
- **Prompts:** Before: inline `PromptTemplate` strings inside `summarizer.py`, with no system message. Now: four `ChatPromptTemplate`s in `prompts.py`, each with a shared system message.
- **PDF library:** Before: PyPDF2. Now: `pypdf`.
- **Results and downloads:** Before: results were rendered inside `if st.button(...)`, so clicking a download button wiped them. Now: results are kept in `st.session_state.results`, so download clicks redraw them.
- **Chunk calls:** Before: sequential calls with `time.sleep(1)` between them. Now: `chunk_chain.batch(...)` with `max_concurrency` of 4.
- **Failure detection:** Before: `'Error' not in summary`, so a real summary containing the word "Error" was dropped. Now: each chunk result has an explicit `ok` flag.
- **Text cleaning:** Before: `clean_text` stripped apostrophes, `%`, `$` and quotes, and collapsed paragraphs. Now: only control characters and extra whitespace are removed. Apostrophes, `%` and `$` stay, and paragraph breaks are kept (blank lines are limited to one).
- **Time estimate:** Before: `num_chunks ** 2.5`. Now: `chunks * 3 / 4 + 5` seconds, which grows in a straight line.
- **Empty or scanned PDF:** Before: sent "." to the model. Now: `chunk_text` raises `ValueError` with a clear message, which the app shows.
- **Combine step (now only):** It uses the chosen style, and it is skipped when there is only one chunk.

## 6. Run it

Python 3.10 or newer is needed.

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

Get a key at https://aistudio.google.com/apikey. Open `.env` and set `GEMINI_API_KEY` to that key. Then start the app:

```bash
streamlit run app.py
```

The app stops and shows an error if the key is missing. It also says the key looks incomplete if it is shorter than 20 characters.

## 7. How to demo it in class

1. Upload a short report, two to five pages long. Point out the metadata boxes. Nothing has been sent to the model yet.
2. Keep the summary type on "detailed" and click Process PDF. Show the step messages, the chunk count and the time estimate.
3. Switch the summary type to "brief" and click Process again. Compare the two results.
4. Switch to "bullet_points" and compare it with "brief". Then open the expander with the individual chunk summaries.
5. Open `prompts.py` and change one line in `STYLES`, for example make "brief" ask for one sentence. Save the file, then restart Streamlit if the change does not show. Process again to see the new output.
6. Click "Download as Text". The results stay on the screen after the download, because they are in session state.

## 8. Common problems

- **Scanned PDF.** The metadata shows, but there is no text. The app shows "No extractable text: the PDF may be scanned images". Run OCR on the file first, then upload the new version.
- **429 rate limit.** Some chunks show "Error processing chunk" with the 429 message, and the stats show them as "Error". Wait a minute and try again. Or use a shorter document, or pick another model in the sidebar. If every chunk fails, the final summary reads "No valid summaries were generated."
- **"Model no longer available to new users" (or 404).** Set `GEMINI_MODEL` in `.env` to a model your key can use, then restart the app. You can also pick another model in the sidebar. If `GEMINI_MODEL` is not in the built-in list, the app adds it to the list.
- **Missing key.** The app shows "GEMINI_API_KEY not found!" and stops. Check that `.env` sits next to `app.py` and has a `GEMINI_API_KEY=` line. Restart the app after you change it.
