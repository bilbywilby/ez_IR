
---

### **Stage 1: Multi-Vector Aggregation & Scraping**
1. **Target Identification**: Data is gathered from both community databases (such as RemoteCentral, ManualsLib, and OneForAll) and official OEM/manufacturer portals (such as GE/Jasco, RCA, and Samsung/Sony/LG) [2, 3].
2. **Resilient Web Scraping**: To prevent Web Application Firewall (WAF) blocks or Cloudflare 403 errors caused by basic text-dumping tools, scraping is handled via a Python wrapper script (`robust_gather_ir.py`) [2, 3]. This script utilizes:
   * **Rotated User-Agent headers** and session persistence [2, 3].
   * **Polite rate-limiting jitter** (random delays between requests) to prevent IP flagging [3].
   * **DOM text extraction** using BeautifulSoup [2, 3].
3. **Raw Telemetry Storage**: Extracted raw page contents are dumped into the `/remote_intelligence/raw/` directory [1, 3]. 
4. **Initial Candidate Filtering**: Basic regex pattern matching (e.g., `\b\d{3,5}\b`) isolates potential 3-, 4-, and 5-digit numerical code candidates from raw text blocks [3].

---

### **Stage 2: Contextual Proximity Parsing & Structuring**
Once raw dumps are collected, a parsing script (`parse_ir_intelligence_v2.py` or `parse_ir_intelligence_v3.py`) cleans and transforms unorganized text into structured data [4-6]:

1. **Brand Matching & Contextual Windowing**: 
   * The parser scans raw text files against an expanded list of known manufacturers (`KNOWN_BRANDS`) covering modern brands and legacy OEMs (e.g., Samsung, RCA, Sony, GE, Panasonic, Zenith) [6, 7].
   * When a brand keyword is found, the parser isolates a **1,000-character sliding window** (500 characters preceding and following the match) to analyze surrounding context without cross-brand code contamination [4-6, 8].
2. **Manual Overrides Layer**: 
   * Before applying regex patterns, the parser checks a manual rules file (`remote_intelligence/data/overrides.json` or `manual_overrides.json`) to handle irregular, chassis-based, or non-standard revision strings (e.g., `REV-X`, `Chassis-BN41`) that evade standard parsing [6, 7].
3. **Hardware Revision Signature Extraction**: 
   * Specialized regular expressions (`REVISION_PATTERNS`) detect specific hardware revision identifiers within the contextual window, including GE/Jasco `CL[9-12]` versions, standard `Revision`/`Rev`/`Version` numbers, and Voxx/RCA alphanumeric tags (such as `R271U1`) [4-6, 8]. If no revision is matched, it defaults to a `Generic` profile [4, 6, 13].
4. **Hierarchical Bucketing & Deduplication**: 
   * All extracted codes within the window are categorized into **3-digit**, **4-digit**, and **5-digit** arrays [4-6, 8].
   * The code arrays are deduplicated, sorted, and written atomically into a normalized JSON database at `remote_intelligence/data/codes_map.json` using the schema hierarchy: **`Brand` → `Remote_Revision` → `Code_Length`** [4-6, 8].

---

### **Stage 3: Offline Access & Directory Hierarchy**
1. **Directory Structure**:
   * `remote_intelligence/raw/`: Raw text dumps from target web sources [1, 6].
   * `remote_intelligence/data/`: `codes_map.json` and manual override files [1, 6].
   * `scripts/`: Modular deployment scripts and tools [1, 6].
2. **Rapid Lookup Interface**:
   * An interactive Terminal User Interface module (`scripts/portal_tui.py`) loads `codes_map.json` to allow offline querying by brand, revision, or code length [1, 14-16].
   * Integrated `termux-api` support allows users to copy filtered code arrays directly to the system clipboard [1, 14-16].
