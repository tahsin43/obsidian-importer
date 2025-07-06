# Apple Notes to Markdown Conversion Process

## **Overview: The Complete Pipeline**

The Apple Notes importer is a sophisticated reverse-engineering system that converts Apple's proprietary binary format into standard Markdown. Here's how it works:

## **1. Database Access & Setup**

### **Location Discovery**
```typescript
const NOTE_FOLDER_PATH = 'Library/Group Containers/group.com.apple.notes';
const NOTE_DB = 'NoteStore.sqlite';
```
- Finds Apple's hidden database at `~/Library/Group Containers/group.com.apple.notes/NoteStore.sqlite`
- Creates a temporary copy for safe access

### **Protobuf Decoder Setup**
```typescript
this.protobufRoot = Root.fromJSON(descriptor);
```
- Loads the reverse-engineered Protocol Buffer schema from `descriptor.ts`
- This is the "decoder ring" that understands Apple's binary format

## **2. Database Structure Analysis**

### **Entity Mapping**
```typescript
this.keys = Object.fromEntries(
    (await this.database.all`SELECT z_ent, z_name FROM z_primarykey`).map(k => [k.Z_NAME, k.Z_ENT])
);
```
- Maps Apple's internal entity types (ICNote, ICFolder, etc.) to database table IDs
- Understands the database schema structure

### **Hierarchy Resolution**
```typescript
// Process accounts, folders, then notes
for (let a of noteAccounts) await this.resolveAccount(a.Z_PK);
for (let f of noteFolders) await this.resolveFolder(f.Z_PK);
for (let n of notes) await this.resolveNote(n.Z_PK);
```
- Builds the folder structure first
- Handles multiple iCloud accounts
- Creates proper folder hierarchy in Obsidian

## **3. Note Content Extraction**

### **Database Query**
```typescript
const row = await this.database.get`
    SELECT
        nd.z_pk, hex(nd.zdata) as zhexdata, zcso.ztitle1, zfolder,
        zcreationdate1, zcreationdate2, zcreationdate3, zmodificationdate1, zispasswordprotected
    FROM zicnotedata AS nd, ziccloudsyncingobject AS zcso
    WHERE zcso.z_pk = nd.znote AND zcso.z_pk = ${id}
`;
```
- Extracts the note's metadata (title, dates, folder)
- Gets the hex-encoded binary data (`zhexdata`) containing the actual content

### **Protobuf Decoding**
```typescript
const converter = this.decodeData(row.zhexdata, NoteConverter);
```
- Converts hex string back to binary
- Decompresses the data (Apple uses gzip compression)
- Decodes using the protobuf schema to get structured data

## **4. Content Type Detection & Conversion**

The system uses different converters based on content type:

### **Basic Text Notes** (`NoteConverter`)
```typescript
export class NoteConverter extends ANConverter {
    async format(): Promise<string> {
        let fragments = this.parseTokens();
        // Process each text fragment with its formatting
    }
}
```

### **Tables** (`TableConverter`)
```typescript
export class TableConverter extends ANConverter {
    // Apple Notes uses CRDTs for real-time collaboration
    // Everything is stored as references with indirection
    async parse(): Promise<string[][] | null> {
        // Decode complex table structure
        // Convert to 2D array
    }
    
    async format(): Promise<string> {
        // Convert to Markdown table format
        // | Column 1 | Column 2 |
        // | -- | -- |
        // | Data | Data |
    }
}
```

### **Scans** (`ScanConverter`)
```typescript
export class ScanConverter extends ANConverter {
    async format(): Promise<string> {
        // Extract scan images
        // Convert to Markdown image links
        return `\n${links.join('\n')}\n`;
    }
}
```

## **5. Text Formatting Conversion**

### **Token Parsing**
```typescript
parseTokens(): ANFragmentPair[] {
    // Apple Notes stores text as "attribute runs" - chunks of text with formatting
    // Each run has: text content + formatting attributes (bold, italic, color, etc.)
    while (i < this.note.attributeRun.length) {
        // Merge consecutive runs with same formatting
        // Split on line boundaries for proper Markdown
    }
}
```

### **Formatting Translation**
```typescript
async formatAttr(attr: ANAttributeRun): Promise<string> {
    switch (attr.fontWeight) {
        case ANFontWeight.Bold:
            attr.fragment = `**${attr.fragment}**`;
        case ANFontWeight.Italic:
            attr.fragment = `*${attr.fragment}*`;
    }
    if (attr.strikethrough) attr.fragment = `~~${attr.fragment}~~`;
}
```

### **Complex Formatting**
```typescript
async formatHtmlAttr(attr: ANAttributeRun): Promise<string> {
    // Handle HTML-only features like underline, font size, color
    if (attr.underlined) attr.fragment = `<u>${attr.fragment}</u>`;
    if (attr.color) style += `color:${this.convertColor(attr.color)};`;
}
```

## **6. Attachment Processing**

### **Attachment Types**
```typescript
switch (uti) {
    case ANAttachment.ModifiedScan:
        sourcePath = path.join('FallbackPDFs', ...);
    case ANAttachment.Scan:
        sourcePath = path.join('Previews', ...);
    case ANAttachment.Drawing:
        sourcePath = path.join('FallbackImages', ...);
    default:
        sourcePath = path.join('Media', ...);
}
```

### **File Extraction**
```typescript
const binary = await this.getAttachmentSource(account, sourcePath);
const file = await this.vault.createBinary(attachmentPath, binary, {
    ctime: this.decodeTime(row.ZCREATIONDATE),
    mtime: this.decodeTime(row.ZMODIFICATIONDATE)
});
```

## **7. Markdown Generation**

### **Content Assembly**
```typescript
// Combine all converted content
let converted = '';
for (let j = 0; j < fragments.length; j++) {
    converted += this.formatMultiRun(attr);
    converted += await this.formatAttr(attr);
}
```

### **File Creation**
```typescript
this.vault.modify(file, await converter.format(), {
    ctime: this.decodeTime(row.ZCREATIONDATE3 || row.ZCREATIONDATE2 || row.ZCREATIONDATE1),
    mtime: this.decodeTime(row.ZMODIFICATIONDATE1)
});
```

## **8. Special Features**

### **Internal Links**
```typescript
async getInternalLink(uri: string): Promise<string> {
    // Convert Apple Notes internal links to Obsidian wiki-links
    const identifier = uri.match(NOTE_URI)![1];
    let file = await this.importer.resolveNote(row.Z_PK);
    return this.app.fileManager.generateMarkdownLink(file, this.importer.rootFolder.path);
}
```

### **Handwriting Recognition**
```typescript
if (this.importer.includeHandwriting && row.ZHANDWRITINGSUMMARY) {
    link = `\n> [!Handwriting]-\n> ${row.ZHANDWRITINGSUMMARY}${link}`;
}
```

## **The Complete Flow:**

1. **Access** Apple's hidden database
2. **Decode** binary protobuf data using reverse-engineered schema
3. **Parse** complex Apple Notes structures (text, tables, scans, attachments)
4. **Convert** Apple's formatting to Markdown/HTML equivalents
5. **Extract** attachments and media files
6. **Generate** standard Markdown files with proper metadata
7. **Create** Obsidian-compatible file structure

## **Key Innovations:**

- **Reverse Engineering**: Undocumented Apple format → Open Markdown
- **Protocol Buffer Decoding**: Binary data → Structured content
- **CRDT Table Handling**: Complex collaboration data → Simple tables
- **Multi-format Support**: Text, tables, scans, drawings, attachments
- **Format Preservation**: Bold, italic, colors, fonts, alignment
- **Link Conversion**: Internal Apple links → Obsidian wiki-links

## **Technical Challenges Solved:**

### **Binary Format Decoding**
- Apple uses Protocol Buffers for data serialization
- Data is compressed with gzip
- Requires reverse-engineered schema to decode

### **Complex Data Structures**
- Tables use CRDTs (Conflict-free Replicated Data Types) for collaboration
- Multiple layers of indirection and object references
- Complex attachment and media handling

### **Format Preservation**
- Apple's rich text format → Markdown/HTML hybrid
- Preserves styling, colors, fonts, alignment
- Handles special features like handwriting recognition

### **File System Integration**
- Extracts media files from Apple's proprietary storage
- Maintains folder hierarchy and metadata
- Creates Obsidian-compatible file structure

## **Performance Considerations:**

### **Current Implementation**
- Processes all notes on every import
- Expensive protobuf decoding for each note
- File existence checks after processing

### **Optimization Opportunities**
- Pre-filter notes before processing
- Cache decoded data for incremental imports
- Batch processing for better performance

This system essentially acts as a "digital translator" that understands Apple's proprietary language and converts it to the universal Markdown format that Obsidian can read. 