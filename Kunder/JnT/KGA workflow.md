
## Introduction
Welcome to the file import conversion tool using ChatGPT! Your task is to process an uploaded Excel file based on a predefined workflow. **Use the GPT-4 model** for this task to ensure accurate and reliable data processing.

After uploading this `.md` file for reference, you will see a cheerful prompt asking for the `.pdf` file to process. The instructions below detail the steps ChatGPT will perform.

After completing the steps, provide the processed file to the user as a `.xlsx`.

# Workflow Documentation: Processing and Combining Data from PDF (Löne ID Tables)

## Purpose
This workflow details the steps required to process a PDF containing structured data for various Löne IDs. The goal is to extract and clean each dataset, combine them into a single structured dataset, and export the result as an Excel file.

## Steps

### Step 1: Read and Structure Data for Each Löne ID
1. **Import Necessary Libraries**:
   - Ensure that the `re` module is imported for text processing.

2. **Identify the Löne ID**:
   - Extract the Löne ID from the text in the PDF for each sheet using a regular expression: `Löne Id:\s*(\d+)`.

3. **Extract Table Data**:
   - Identify the table associated with the Löne ID by locating rows starting with "Dag".
   - Combine the first two rows of the table to create column headers.
   - Drop the first two rows after headers are created to retain only the data.

4. **Clean the Data**:
   - Exclude the "Totaler" row and all rows below it to remove irrelevant information.
   - **Remove rows where either "Avr In" or "Avr Ut" is empty**:
     - Filter rows to retain only those where both "Avr In" and "Avr Ut" have valid, non-empty values.

5. **Add Löne ID to Each Row**:
   - Add a column labeled "Löne ID" to every row, specifying the respective Löne ID.

### Step 2: Combine Multiple Löne IDs
1. **Ensure Consistent Headers**:
   - Ensure column headers are consistent across all Löne IDs before combining datasets.
   - If headers differ, standardize them or deduplicate conflicting column names.

2. **Append Datasets**:
   - Start with the first Löne ID dataset.
   - Append subsequent datasets row-wise, ensuring the "Löne ID" column explicitly labels each row.

3. **Save as Excel File**:
   - Export the combined dataset to an Excel file using pandas `to_excel` method.

### Step 3: Fill Missing Cells in the "Datum" Column
1. **Normalize and Handle Empty Cells**:
   - Ensure non-date or blank cells in the "Datum" column are treated as missing by applying:
     ```python
     dataframe['Datum'] = dataframe['Datum'].apply(
         lambda x: x if pd.notna(x) and str(x).strip() else None
     )
     ```

2. **Copy the "Datum" Value into Empty Cells from the Row Above**:
   - Apply forward-fill to fill empty "Datum" cells using the value from the row above:
     ```python
     dataframe['Datum'] = dataframe['Datum'].fillna(method='ffill')
     ```

### Step 4: Filter and Retain Only Necessary Columns
1. **Retain Specific Columns**:
   - Keep the following columns:
     - Datum
     - Avr In
     - In Typ
     - Avr Ut
     - Ut Typ
     - OB Enkel
     - OB Dubbel
     - ÖT1
     - ÖT2
     - Frånvaro
     - Löne ID
   - Remove all other columns from the dataset.

2. **Export Filtered Dataset**:
   - Save the filtered dataset to an Excel file.

## Python Code
```python
import pandas as pd
import pdfplumber
import re

def process_pdf_to_excel(pdf_path, output_path):
    combined_data = []

    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            text = page.extract_text()

            # Identify Löne ID
            löne_id_match = re.search(r"Löne Id:\s*(\d+)", text)
            if löne_id_match:
                löne_id = löne_id_match.group(1)

                # Extract table data
                tables = page.extract_tables()
                for table in tables:
                    df = pd.DataFrame(table)

                    # Validate and structure the table
                    if df.iloc[0, 0] == "Dag":
                        # Combine headers from the first two rows
                        headers = [
                            f"{df.iloc[0, i]} {df.iloc[1, i]}".strip()
                            for i in range(len(df.columns))
                        ]
                        df.columns = headers
                        df = df.iloc[2:].reset_index(drop=True)

                        # Exclude rows below "Totaler"
                        if "Totaler" in df.iloc[:, 0].values:
                            totaler_index = df[df.iloc[:, 0].str.contains("Totaler", na=False)].index[0]
                            df = df.iloc[:totaler_index]

                        # Add Löne ID column
                        df["Löne ID"] = löne_id

                        # Remove rows where "Avr In" or "Avr Ut" is empty
                        df = df[df["Avr In"].str.strip().astype(bool) & df["Avr Ut"].str.strip().astype(bool)]

                        # Append processed data
                        combined_data.append(df)

    # Combine datasets
    combined_dataset = pd.concat(combined_data, ignore_index=True)

    # Fill missing "Datum" values
    combined_dataset['Datum'] = combined_dataset['Datum'].apply(
        lambda x: x if pd.notna(x) and str(x).strip() else None
    )
    combined_dataset['Datum'] = combined_dataset['Datum'].fillna(method='ffill')

    # Retain necessary columns
    columns_to_keep = [
        "Datum", "Avr In", "In Typ", "Avr Ut", "Ut Typ",
        "OB Enkel", "OB Dubbel", "ÖT1", "ÖT2", "Frånvaro", "Löne ID"
    ]
    filtered_dataset = combined_dataset[columns_to_keep]

    # Export to Excel
    filtered_dataset.to_excel(output_path, index=False)
```

## Instructions to Run the Python Code
1. **Setup the Environment**:
   - Ensure Python is installed on your system.
   - Install necessary libraries using pip:
     ```
     pip install pandas pdfplumber openpyxl
     ```

2. **Prepare the Input PDF**:
   - Place the PDF file you want to process in the same directory as the Python script, or provide the full path to the file.

3. **Run the Python Script**:
   - Execute the script in your terminal or command prompt:
     ```
     python your_script_name.py
     ```
   - Replace `your_script_name.py` with the name of your Python script.

4. **Output**:
   - The script will generate an Excel file containing the processed dataset.
   - Check the output directory for the generated Excel file.

5. **Verify the Output**:
   - Open the Excel file and ensure all steps (removing rows, filling "Datum," retaining columns) have been applied correctly.
