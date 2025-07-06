# Apple Notes Importer Codebase Analysis

## **My Question**
I asked you to go through the codebase and explain how Apple Notes extraction works exactly, and explain the codebase structure.

## **Summary of Our Discussion**

### **1. Apple Notes Extraction Process**

The Apple Notes importer is a sophisticated system that extracts notes from Apple's proprietary database format and converts them to Markdown. Here's how it works:

#### **Database Access**
- **Location**: `~/Library/Group Containers/group.com.apple.notes/NoteStore.sqlite`
- **Approach**: Creates temporary copy of database to avoid conflicts with running Apple Notes app
- **Platform**: Only works on macOS due to Apple's data location

#### **Data Structure**
- **Protobuf Format**: Notes stored as hex-encoded, gzip-compressed protobuf data
- **Complex Schema**: Uses Apple's internal data format with custom descriptors
- **Rich Content**: Handles text formatting, attachments, tables, drawings, scans

#### **Extraction Flow**
1. **Database Setup**: Copy Apple Notes database to temp location
2. **Account/Folder Resolution**: Map Apple Notes structure to Obsidian folders
3. **Note Processing**: Extract note content and metadata
4. **Content Conversion**: Convert Apple's rich text to Markdown
5. **Attachment Handling**: Extract images, drawings, scans, tables

### **2. SQLite Folder Purpose**

You asked about the `sqlite` folder, which contains a **custom SQLite wrapper library**.

#### **What it is**
A convenience wrapper around the system's `sqlite3` command-line tool that provides:
- **Template Literal SQL Queries**: Safe, readable SQL syntax
- **Automatic Parameter Escaping**: Prevents SQL injection
- **Promise-based API**: Modern async/await interface
- **Cross-platform Compatibility**: Works with different SQLite versions

#### **Why it exists**
Instead of using standard SQLite libraries, this wrapper:
- **Simplifies complex queries** needed for Apple Notes extraction
- **Prevents SQL injection** through automatic escaping
- **Makes code readable** with template literal syntax
- **Handles platform differences** across macOS versions

#### **Usage Examples**
```javascript
// Instead of raw SQL:
const query = "SELECT * FROM notes WHERE id = " + id; // UNSAFE

// Use the wrapper:
const result = await db.get`SELECT * FROM notes WHERE id = ${id}`; // SAFE
```

### **3. Codebase Structure**

#### **Main Files**
- `apple-notes.ts`: Main importer class and orchestration
- `descriptor.ts`: Apple's protobuf schema definitions
- `models.ts`: TypeScript interfaces and data types
- `convert-note.ts`: Note content conversion logic
- `convert-table.ts`: Table handling (complex CRDT structure)
- `convert-scan.ts`: Scan/drawing extraction

#### **SQLite Wrapper Files**
- `sqlite/index.js`: Main wrapper interface
- `sqlite/utils.js`: SQL query building and escaping
- `sqlite/fallback.js`: Compatibility for different SQLite versions

### **4. Key Technical Challenges Solved**

1. **Platform Limitations**: Only works on macOS
2. **Database Access**: Handles SQLite with WAL mode and file copying
3. **Protobuf Decoding**: Reverse-engineered Apple's internal data format
4. **Rich Text Conversion**: Complex mapping from attribute runs to Markdown
5. **CRDT Tables**: Handles Apple's complex table data structure
6. **Attachment Resolution**: Maps internal references to actual files
7. **Multi-account Support**: Handles multiple Apple ID accounts

### **5. Where SQLite Wrapper is Used**

The wrapper is used extensively throughout the codebase for:
- **Metadata Queries**: Getting entity types, accounts, folders
- **Note Content**: Extracting note data and metadata  
- **Attachment Resolution**: Finding file paths and metadata
- **Internal References**: Resolving links between notes and attachments
- **Special Content**: Handling tables, scans, drawings

**Total Usage**: Found in 3 main files with ~20+ database queries using the wrapper's template literal syntax.

## **Conclusion**

The Apple Notes importer is a remarkably sophisticated system that demonstrates deep reverse engineering of Apple's proprietary formats. The SQLite wrapper is a key component that makes the complex database operations safe, readable, and maintainable. The entire system handles Apple's complex internal data structures while providing a clean interface for users to import their notes into Obsidian. 