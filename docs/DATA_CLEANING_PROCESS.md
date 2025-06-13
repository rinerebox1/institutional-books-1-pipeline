# Data Cleaning and OCR Post-Processing

This document outlines the data cleaning and OCR post-processing steps applied to the digitized text. The primary goal is to improve the readability and accuracy of the text extracted via Optical Character Recognition (OCR).

The process involves a sophisticated pipeline that first classifies lines of OCR output using a machine learning model and then applies a set of heuristics to reconstruct the text.

## 1. ML-based Line Classification

At the core of the OCR post-processing is a machine learning model (`StaticModelForClassification`). This model is trained to classify each line of the raw OCR output into one of several predefined categories. This classification allows the system to apply context-specific rules for cleaning and formatting.

The model assigns one of the following target types to each text line:

*   `UNKNOWN`: The line type could not be confidently determined.
*   `NOISE_OR_BROKEN_TEXT`: The line appears to be noise (e.g., specks, smudges) or severely garbled text.
*   `PAGE_NUMBER`: The line is identified as a page number.
*   `RUNNING_HEAD`: The line is identified as a running head (text repeated at the top or bottom of pages, like a chapter title or book title).
*   `HEADING_OR_TITLE`: The line represents a heading (e.g., chapter title, section title) or a title.
*   `PARAGRAPH_CHUNK`: The line is part of a regular paragraph of text.
*   `PARAGRAPH_END`: The line is the end of a paragraph, often ending with punctuation.
*   `LOOSE_SENTENCE_OR_LIST_ITEM`: The line is a standalone sentence or an item in a list.
*   `SEPARATOR`: The line is a visual separator, often made of dashes or other special characters.

## 2. Heuristic-based Text Recomposition

Once each line is classified, the system uses a set of heuristics to recompose the text from these chunks. The function `convert_page_chunks_to_text` orchestrates this, applying specific processing logic based on the `target_type` of each chunk, as well as its preceding and succeeding chunks.

The general approach involves:
- Applying type-specific rules to format the current chunk's text and decide on spacing/line breaks around it.
- Attempting to remove hyphenation at the end of chunks if the next chunk starts with a lowercase letter.
- Performing a final cleanup of the assembled page text.

Below are the details for how each chunk type is typically handled:

### Chunk-Specific Processing Logic

*   **`UNKNOWN` (`process_unknown_chunk`):**
    *   The text of chunks classified as `UNKNOWN` is generally added to the output as is, followed by a whitespace. This is a fallback for lines where the model couldn't make a confident prediction.

*   **`NOISE_OR_BROKEN_TEXT` (`process_noise_or_broken_text_chunk`):**
    *   **Separator Conversion:** If the chunk's text consists of a short sequence of dashes (e.g., "-", "--") or a series of identical special characters (e.g., "***", "+++"), it is converted into a standardized separator: `\n\n---\n\n`.
    *   **Skipping:** Empty strings, strings containing only whitespace, or single-character lines are typically skipped and not added to the output.
    *   **Hyphenation Removal:** If the chunk contains two or more words, an attempt is made to remove hyphenation (see `remove_hyphenation_from_chunk` below).
    *   **Default:** If none of the above conditions are met, the text is added as is, followed by a whitespace (unless hyphenation was removed, in which case only the modified text is added).

*   **`PAGE_NUMBER` (`process_page_number_chunk`):**
    *   **Removal:** If the chunk is likely an actual page number (based on its position on the page – first 5 lines or last 10% of lines – and its format – a single "word"), it is removed from the output.
    *   **Reclassification:** If it's not removed, the chunk's `target_type` is changed to `UNKNOWN`, and its text is added as is, followed by a whitespace. This handles cases where page numbers might be embedded within other text or misclassified.

*   **`RUNNING_HEAD` (`process_running_head_chunk`):**
    *   **Removal:** If the chunk is likely an actual running head (appearing in the first 10 lines of the page, and the book is past page 10), it is removed from the output.
    *   **Reclassification:** Otherwise, its `target_type` is changed to `UNKNOWN`, and its text is added as is, followed by a whitespace.

*   **`HEADING_OR_TITLE` (`process_heading_or_title_chunk`):**
    *   **Hyphenation Removal:** An attempt is made to remove hyphenation.
    *   **Line Breaks:**
        *   A double line break (`\n\n`) is prepended if this is the first heading in a sequence (i.e., the previous chunk was not a `HEADING_OR_TITLE`).
        *   A double line break is appended if it's the last in a sequence or if the heading text ends with line-breaking punctuation (e.g., '.', '!', ';', ':', '?').
    *   **Spacing:** A whitespace is added at the end of the current text, unless hyphenation was removed.

*   **`PARAGRAPH_CHUNK` (`process_paragraph_chunk`):**
    *   **Hyphenation Removal:** An attempt is made to remove hyphenation.
    *   **Sentence Break Detection:** If the chunk contains more than one "word", includes line-breaking punctuation, and is not surrounded by other paragraph-like chunks, the system attempts to find the last such punctuation mark and insert a double line break after it. This helps to split run-on sentences that were part of a single OCR line.
    *   **New Paragraph Detection:** If the *previous* chunk ended with punctuation and the *current* chunk starts with a number or an uppercase letter, a double line break is prepended, potentially starting a new paragraph or list item.
    *   **Spacing:** Text is followed by a whitespace, unless hyphenation was removed.

*   **`PARAGRAPH_END` (`process_paragraph_end_chunk`):**
    *   **Sentence Break Insertion:** Similar to `PARAGRAPH_CHUNK`, if the line has multiple words and contains line-breaking punctuation, a double line break is inserted after the last such punctuation.
    *   **Final Line Break:** The text is always followed by a double line break (`\n\n`) to ensure paragraph separation.

*   **`LOOSE_SENTENCE_OR_LIST_ITEM` (`process_loose_sentence_or_list_item_chunk`):**
    *   **Hyphenation Removal:** An attempt is made to remove hyphenation.
    *   **Leading Line Break:** If the chunk has multiple words and starts with a digit or an uppercase letter, a double line break is prepended.
    *   **Trailing Line Break:** A double line break is appended if the *next* chunk is not a `LOOSE_SENTENCE_OR_LIST_ITEM` and starts with an uppercase letter (and no hyphenation was removed from the current chunk). This helps separate distinct list items or loose sentences.
    *   **Spacing:** Text is followed by a whitespace, unless hyphenation was removed.

*   **`SEPARATOR` (`process_separator_chunk`):**
    *   Non-empty separator chunks are replaced with a standardized separator: `\n\n---\n\n`.
    *   Empty separators are skipped.

### General Hyphenation Removal (`remove_hyphenation_from_chunk`)

A utility function `remove_hyphenation_from_chunk` is used by several of the processors above.
*   **Logic:** If the current chunk's text ends with any type of dash character (e.g., "-", "‐", "–") and the *next* chunk's text starts with a lowercase letter, the dash is removed from the end of the current chunk's text.
*   **Return Value:** The function returns `True` if hyphenation was removed, `False` otherwise. This allows the calling processor to avoid adding extra whitespace if a word was just joined.

### Final String Cleanup

After processing all chunks for a page, a final cleanup pass is performed on the resulting string:

*   **Excess Line Breaks:** Multiple consecutive line breaks (e.g., `\n\n\n`) are reduced to double line breaks (`\n\n`). Spacing around line breaks (e.g., `\n \n`) is also normalized.
*   **Hyphenation Remnants:** Patterns like `-\n\n` are replaced with a space, and various dash/separator patterns are normalized (e.g., `\n\n--\n` to `\n\n---\n\n`).
*   **Double Spaces:** Multiple spaces are collapsed into single spaces.
*   **Coarse Sentence Joining:** Some regex substitutions attempt to join parts of sentences that might have been incorrectly split by double line breaks, for example:
    *   `re.sub(r"([a-z])\n\n([a-z])", r"\1 \2", output)`
    *   `re.sub(r"([a-z] +)\n\n([a-z])", r"\1 \2", output)`
    *   `re.sub(r"(,+)\n\n([a-zA-Z0-9])", r"\1 \2", output)`
*   The text is stripped of leading/trailing whitespace.

## 3. Book-Level Post-Processing

After all pages in a book have undergone the chunk-based recomposition, a few additional book-level post-processing steps are applied. These aim to catch issues that are easier to identify by looking at the text of multiple pages together.

*   **Remove Remaining Running Heads (`remove_remaining_running_heads`):**
    *   **Identification:** This function first scans through all pages of the book. It counts occurrences of identical lines of text that appear within the first few lines (specifically, top 5 lines after splitting by `\n\n`) of each page.
    *   Lines that contain fewer than 3 "words" or occur less than 3 times across the entire book are excluded.
    *   Lines occurring more than 3 times are considered "likely running heads."
    *   **Removal:** The identified likely running heads are then removed from the beginning of each page's text (only the first occurrence if a page text starts with it). This step is generally skipped for the first 10 pages of a book.
    *   Extra line breaks resulting from the removal are cleaned up.

*   **Remove Remaining Hyphenations (`remove_remaining_hyphenations`):**
    *   This function applies a coarse, regex-based hyphenation removal to each page's text.
    *   The primary regex used is `re.sub(r"(?<![-\w])(\d+)[\-–—‒―]\s+(\w+)", r"\1\2", text)`.
    *   **Purpose:** This specific pattern targets hyphenated numbers followed by whitespace and then an alphanumeric word (e.g., "item 123- abc" becomes "item 123abc"). It seems designed to fix a particular OCR error pattern where numbers are incorrectly hyphenated before a word.

*   **Remove Remaining Page Numbers (`remove_remaining_page_numbers`):**
    *   This function attempts to remove page number patterns that might still be present at the very beginning of a page's text, especially after the initial 10 pages of a book.
    *   It uses regexes to find patterns within the first 10 characters of the page text:
        *   `re.search(r"[−–—-]\s*\d+\s*[−–—-]", text[0:10])`: Looks for numbers enclosed in dashes (e.g., "---123---").
        *   `re.search(r"[−–—-]\s*\d+", text[0:10])`: Looks for numbers preceded by dashes (e.g., "---123").
        *   `re.search(r"[0-9]+ ", text[0:10])`: Looks for numbers followed by a space (e.g., "123 ").
    *   If a pattern is found, it's removed (only the first match).
    *   The page text is then stripped of leading/trailing whitespace.
