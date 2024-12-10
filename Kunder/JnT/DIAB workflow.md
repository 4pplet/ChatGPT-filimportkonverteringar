
# Workflow for Processing DIAB PDF with Multiple Sheets

## Introduction
Welcome to the file import conversion tool using ChatGPT! Your task is to process an uploaded PDF file based on a predefined workflow. **Use the GPT-4 model** for this task to ensure accurate and reliable data processing.

After uploading this `.md` file for reference, you will see a cheerful prompt asking for the `.pdf` file to process. The instructions below detail the steps ChatGPT will perform.

After completing the steps, provide the processed file to the user as a `.xlsx`.

The current implementation ignores rows of days with "Semester" in it. The keywords to ignore is defined in the bottom of the file.

## Instructions for ChatGPT:
This workflow is designed to process a DIAB PDF file containing multiple sheets, where each sheet represents a consultant's data. The goal is to extract valid rows, including `Date`, `In`, `Out`, and `ID`, from all sheets and combine them into a single dataset.

### Steps to Follow:
1. **Open the PDF File**:
   - Read the entire PDF file and process each page sequentially.

2. **Extract Consultant ID**:
   - Look for the line containing `"Id/Kortnr:"` on each page.
   - Use refined regex matching (`Id/Kortnr:\s*(\d+)`) to extract the consultant ID.

3. **Identify Year and Month**:
   - Detect the header containing the year (`yyyy`) and month (`mars`, etc.).
   - Map Swedish month names to their respective numeric representations.

4. **Identify Valid Rows**:
   - Detect rows starting with `"XX DAY hh:mm hh:mm"`.
     - **Example**: `"01 fre 13:53 22:04"`.
   - Construct the full date using the year (`yyyy`) and month (`mars`, etc.) from the header.
   - Extract:
     - `Date`: The full date constructed from the day, month, and year.
     - `In`: The first time in the row.
     - `Out`: The second time in the row.

5. **Skip Irrelevant Rows**:
   - Exclude rows without valid `In` and `Out` times.
   - Ignore footer rows like `"INGPOST"` or `"Tidsumma"`.
   - Introduce keyword filtering:
     - Filter out rows containing specific keywords, such as `"Semester"`, which are not valid for `In` or `Out` times.
     - Maintain a configurable list of keywords for future additions.

6. **Combine Data**:
   - Add all valid rows from each sheet into a single dataset.
   - Include a column for the consultant ID (`ID`).

7. **Save the Output**:
   - Save the combined dataset as an Excel file with the following columns:
     - `Date`, `In`, `Out`, `ID`.

---

## Python Code:

```python
import pdfplumber
import pandas as pd
import re

def process_diab_pdf_with_keyword_filtering(pdf_path, output_path, keywords_to_filter):
    swedish_months = {
        "januari": "01", "februari": "02", "mars": "03", "april": "04",
        "maj": "05", "juni": "06", "juli": "07", "augusti": "08",
        "september": "09", "oktober": "10", "november": "11", "december": "12"
    }
    all_data = []
    logs = []

    with pdfplumber.open(pdf_path) as pdf:
        for page_num, page in enumerate(pdf.pages, start=1):
            page_text = page.extract_text()
            lines = page_text.split('\n')

            # Extract Consultant ID
            consultant_id = None
            for line in lines:
                match = re.search(r"Id/Kortnr:\s*(\d+)", line)
                if match:
                    consultant_id = match.group(1).strip()
                    logs.append(f"Consultant ID found: {consultant_id}")
                    break

            if not consultant_id:
                logs.append(f"Consultant ID not found on page {page_num}. Skipping.")
                continue

            # Identify year and month
            year, month = None, None
            for line in lines:
                if any(m in line.lower() for m in swedish_months.keys()):
                    for m_name, m_number in swedish_months.items():
                        if m_name in line.lower():
                            month = m_number
                            break
                    year_match = re.search(r"\d{4}", line)
                    year = year_match.group(0) if year_match else None
                    logs.append(f"Year and month detected: {year}-{month}.")
                    break

            if not (year and month):
                logs.append(f"Year and month not found on page {page_num}. Skipping.")
                continue

            # Parse valid rows
            for line in lines:
                # Skip rows containing keywords to filter
                if any(keyword.lower() in line.lower() for keyword in keywords_to_filter):
                    logs.append(f"Skipping row due to keyword filter: {line.strip()}")
                    continue

                if re.match(r"^\s*\d{2}\s\w{3}", line):
                    parts = line.split()
                    times = [p for p in parts if re.match(r"^\d{2}:\d{2}$", p)]
                    if len(times) >= 2:
                        day = parts[0].zfill(2)
                        date = f"{year}-{month}-{day}"
                        in_time, out_time = times[0], times[1]
                        all_data.append({"Date": date, "In": in_time, "Out": out_time, "ID": consultant_id})
                        logs.append(f"Row added: {date}, {in_time}, {out_time}, {consultant_id}")
                    else:
                        logs.append(f"Skipped row without sufficient times: {line.strip()}")

    # Save to Excel
    df = pd.DataFrame(all_data)
    df.to_excel(output_path, index=False)
    return logs

# Example usage
pdf_path = "/path/to/DIAB.pdf"
output_path = "/path/to/Processed_DIAB_with_Keywords.xlsx"
keywords_to_filter = ["Semester"]
logs = process_diab_pdf_with_keyword_filtering(pdf_path, output_path, keywords_to_filter)
print("\n".join(logs))
```


### Keyword Filter Configuration

To filter out rows containing specific keywords (e.g., "Semester", "VAB"), a configurable keyword filter is included in the script.

#### Adding or Removing Keywords
- Open the Python script.
- Locate the `keywords_to_filter` variable.
- Update the list to include or exclude keywords as needed. For example:

```python
keywords_to_filter = ["Semester", "VAB", "Tidsumma", "INGPOST"]
```

- Save the changes and rerun the script.

#### Default Filtered Keywords
- `Semester`: Indicates vacation days.
- `VAB`: Stands for "Vård av barn" (care of children).
- `Tidsumma`: Represents summary rows, not relevant for processing.
- `INGPOST`: Represents header rows, not valid data.

This configuration ensures flexibility in managing the data extraction process based on specific project requirements.
