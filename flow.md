`mermaid
flowchart TD
    %% ==== Entry point ==================================================
    Start([Start: python pdf_table_marker.py <PDF> [page]])

    %% ==== 1️⃣ Load PDF & render page =====================================
    subgraph LoadAndRender ["1️⃣ Load PDF & render page"]
        A1[Validate PDF path] --> A2[Open with pdfplumber]
        A2 --> A3{Is page_number valid?<br/>(1‑len(pages))}
        A3 -- Yes --> A4[Select page (0‑based index)]
        A3 -- No --> A5[Raise ValueError & exit]
        A4 --> A6[Render page to Pillow Image (150 dpi)]
        A6 --> A7[Return (pdfplumber.Page, PIL.Image)]
    end

    %% ==== 2️⃣ Interactive line‑drawing UI ================================
    subgraph UI ["2️⃣ Interactive line‑drawing UI"]
        B1[Create Matplotlib Figure/Axes] --> B2[Display image as background]
        B2 --> B3[Instantiate LineDrawer(ax, img_w, img_h)]
        B3 --> B4[Attach callbacks:\n• button_press → on_press\n• button_release → on_release\n• motion_notify → on_move\n• key_press → on_key]
        B4 --> B5[Show instructions overlay]
        B5 --> B6[User draws lines:\n • press ‘v’ → vertical mode\n • press ‘h’ → horizontal mode\n • click‑drag → temp line\n • release → commit line\n • press ‘c’ → clear all\n • press ‘q’ → quit]\n
        B6 --> B7[Lines stored in:\n • vert_lines (x‑coords)\n • horiz_lines (y‑coords)]
        B7 --> B8[Press <Enter> → close figure → exit UI loop]
    end

    %% ==== 3️⃣ Build grid from drawn separators ==========================
    subgraph BuildGrid ["3️⃣ Build grid from separators"]
        C1[Combine user lines with page borders (0 & width/height)] --> C2[Sort vertical & horizontal coordinates]
        C2 --> C3[Create cell boxes (Rectangles) from adjacent pairs]\n
        C3 --> C4[Store grid as list of cell bounds (x0,x1,y0,y1)]
    end

    %% ==== 4️⃣ Extract cell contents ======================================
    subgraph Extract ["4️⃣ Extract cell contents"]
        D1[Iterate over each cell] --> D2[Crop cell region from pdfplumber.Page using bounds]
        D2 --> D3[Use .extract_text() (or .within_bbox) to get string]
        D3 --> D4[Append text to row list]
        D4 --> D5[When row complete → append to table list]
    end

    %% ==== 5️⃣ Output JSON ================================================
    subgraph Output ["5️⃣ Output JSON"]
        E1[Convert table (list of rows) → Python dict] --> E2[json.dump(..., indent=2)]
        E2 --> E3[Print to stdout OR write to <pdf_name>_pageN.json]
    end

    %% ==== Error handling =================================================
    subgraph Errors ["Error handling"]
        Err1[FileNotFoundError (PDF missing)] --> End
        Err2[ValueError (invalid page)] --> End
    end

    %% ==== Flow connections =================================================
    Start --> LoadAndRender --> UI --> BuildGrid --> Extract --> Output --> End([End])

    %% ==== Early‑exit paths (quit / clear) =================================
    B6 -->|press ‘q’| End
    B6 -->|press ‘c’| B4   %% clears lines and returns to drawing mode