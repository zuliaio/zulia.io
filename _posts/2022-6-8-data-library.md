---
layout: post
title: Data Library
description: Standalone library for reading and writing CSV, TSV, Excel, and JSON data
---

# Zulia Data Library

The `zulia-data` module is a standalone Java library for reading and writing CSV, TSV, Excel, and JSON data. It provides a unified API with format auto-detection, configurable type handling, and streaming support for large files.

## Gradle
```bash
repositories {
    mavenCentral()
    maven {
        url "https://maven.ascend-tech.us/repo/"
    }
}

dependencies {
    implementation 'io.zulia:zulia-data:5.1.2'
}
```

## Maven
```xml
<dependency>
    <groupId>io.zulia</groupId>
    <artifactId>zulia-data</artifactId>
    <version>5.1.2</version>
</dependency>
```

# Reading Data (Sources)

## Auto-Detect Spreadsheet Format

The `SpreadsheetSourceFactory` auto-detects the format from the file extension and creates the appropriate source.

```java
// Read with headers (CSV, TSV, or Excel based on extension)
try (SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromFileWithHeaders("/data/input.csv")) {
    for (SpreadsheetRecord record : source) {
        String name = record.getString("name");
        Integer age = record.getInt("age");
    }
}
```

### Header Options
```java
// Standard headers - allows duplicate and blank headers (renames duplicates with _2, _3 suffix)
SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromFileWithHeaders(filePath);

// Strict headers - no duplicate or blank headers allowed
SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromFileWithStrictHeaders(filePath);

// No headers - access by column index only
SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromFileWithoutHeaders(filePath);
```

### From Input Streams
```java
// From a DataInputStream
SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromStreamWithHeaders(dataInputStream);

// From a raw InputStream with metadata
DataStreamMeta meta = DataStreamMeta.fromFileName("data.xlsx");
SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromSingleUseStreamWithHeaders(inputStream, meta);
```

## Record Access

All record types support both name-based and index-based field access.

```java
for (SpreadsheetRecord record : source) {
    // By column name (requires headers)
    String title = record.getString("title");
    Integer year = record.getInt("pubYear");
    Double score = record.getDouble("score");
    Boolean active = record.getBoolean("active");
    Date created = record.getDate("created");

    // By column index (0-based)
    String first = record.getString(0);
    Integer second = record.getInt(1);

    // With default values
    int count = record.getInt("count", 0);

    // Lists from delimited cell values (default delimiter: ;)
    List<String> tags = record.getList("tags", String.class);
    List<Integer> ids = record.getList("ids", Integer.class);

    // Full row as string array
    String[] row = record.getRow();

    // Available headers
    SequencedSet<String> headers = record.getHeaders();
}
```

## CSV Source

```java
FileDataInputStream input = FileDataInputStream.from("/data/test.csv");
CSVSourceConfig config = CSVSourceConfig.from(input)
    .withHeaders()
    .withDelimiter(',')
    .withListDelimiter(';');

try (CSVSource source = CSVSource.withConfig(config)) {
    for (CSVRecord record : source) {
        String value = record.getString("column");
    }
}
```

### CSV Options

| Option | Default | Description |
|--------|---------|-------------|
| `withHeaders()` | no headers | Read first row as headers |
| `withStrictHeaders()` | no headers | Headers with no duplicates or blanks allowed |
| `withDelimiter(char)` | `,` | Field delimiter |
| `withListDelimiter(char)` | `;` | Delimiter for list values within a cell |
| `withBooleanParser(Function)` | true/t/yes/y/1 | Custom boolean parsing |
| `withDateParser(Function)` | ISO_DATE_TIME | Custom date parsing |

## TSV Source

```java
TSVSourceConfig config = TSVSourceConfig.from(input).withHeaders();
try (TSVSource source = TSVSource.withConfig(config)) {
    for (TSVRecord record : source) {
        String value = record.getString("column");
    }
}
```

Same options as CSV except the delimiter is fixed to tab.

## Excel Source

```java
FileDataInputStream input = FileDataInputStream.from("/data/test.xlsx");
ExcelSourceConfig config = ExcelSourceConfig.from(input).withHeaders();

try (ExcelSource source = ExcelSource.withConfig(config)) {
    for (ExcelRecord record : source) {
        String value = record.getString("column");

        // Access native POI objects if needed
        Row nativeRow = record.getNativeRow();
        Cell cell = record.getCell("column");
    }
}
```

### Excel Options

| Option | Default | Description |
|--------|---------|-------------|
| `withHeaders()` | no headers | Read first row as headers |
| `withStrictHeaders()` | no headers | Headers with no duplicates or blanks allowed |
| `withListDelimiter(char)` | `;` | Delimiter for list values within a cell |
| `withExcelCellHandler(handler)` | DefaultExcelCellHandler | Custom cell type conversion |
| `setOpenHandling(handling)` | FIRST_SHEET | FIRST_SHEET or ACTIVE_SHEET |

### Multiple Sheets
```java
ExcelSourceConfig config = ExcelSourceConfig.from(input).withHeaders();
try (ExcelSource source = ExcelSource.withConfig(config)) {
    // Read first sheet
    for (ExcelRecord record : source) { ... }

    // Switch to another sheet by index or name
    source.switchSheet(1);
    // or
    source.switchSheet("Sheet2");

    for (ExcelRecord record : source) { ... }

    // Query sheet info (these methods are on ExcelSource, not ExcelSourceConfig)
    int count = source.getNumberOfSheets();
    String name = source.getActiveSheetName();
}
```

## JSON Array Source

Reads a JSON array of objects `[{...}, {...}]`.

```java
JsonArraySourceConfig config = JsonArraySourceConfig.from(dataInputStream);
try (JsonArraySource source = JsonArraySource.withConfig(config)) {
    for (JsonSourceRecord record : source) {
        String name = record.getString("name");
        List<String> tags = record.getList("tags", String.class);
    }
}
```

## JSON Lines Source (JSONL)

Reads one JSON object per line.

```java
JsonLineSourceConfig config = JsonLineSourceConfig.from(dataInputStream);
try (JsonLineDataSource source = JsonLineDataSource.withConfig(config)) {
    for (JsonSourceRecord record : source) {
        String name = record.getString("name");
    }
}
```

### Error Handling
```java
// Log and skip bad lines instead of throwing
JsonLineSourceConfig config = JsonLineSourceConfig.from(dataInputStream)
    .withExceptionHandler(new LoggingJsonLineParseExceptionHandler(log));
```

# Writing Data (Targets)

## Auto-Detect Format

The `SpreadsheetTargetFactory` auto-detects the output format from the file extension.

```java
List<String> headers = List.of("name", "age", "active");

// Auto-detect format from extension (CSV, TSV, or XLSX)
try (SpreadsheetTarget<?, ?> target = SpreadsheetTargetFactory.fromFileWithHeaders("/data/out.csv", headers)) {
    target.appendValue("John");
    target.appendValue(30);
    target.appendValue(true);
    target.finishRow();

    target.appendValue("Jane");
    target.appendValue(25);
    target.appendValue(false);
    target.finishRow();
}
```

### Without Headers
```java
try (SpreadsheetTarget<?, ?> target = SpreadsheetTargetFactory.fromFile("/data/out.csv")) {
    target.writeRow("John", 30, true);
    target.writeRow("Jane", 25, false);
}
```

### Custom Configuration
```java
SpreadsheetTarget<?, ?> target = SpreadsheetTargetFactory.fromFileWithHeaders(
    "/data/out.csv", true, headers,
    config -> config.withListDelimiter('|')
);
```

## CSV Target

```java
CSVTargetConfig config = CSVTargetConfig.from(FileDataOutputStream.from("/data/out.csv", true))
    .withHeaders(List.of("name", "score", "date"))
    .withDelimiter(',')
    .withListDelimiter(';');

try (CSVTarget target = CSVTarget.withConfig(config)) {
    target.appendValue("John");
    target.appendValue(3.14);
    target.appendValue(new Date());
    target.finishRow();
}

// Or with defaults
try (CSVTarget target = CSVTarget.withDefaultsFromFile("/data/out.csv", true, headers)) {
    target.writeRow("John", 3.14, new Date());
}
```

### CSV Default Formatting
| Type | Default Format |
|------|---------------|
| Number (float/double) | 3 decimal places |
| Date | ISO_DATE_TIME |
| Boolean | "True" / "False" |
| Collection | Joined with list delimiter |
| Link | href value only |

### Custom Type Handlers
```java
CSVTargetConfig config = CSVTargetConfig.from(dataOutputStream)
    .withNumberTypeHandler((writer, value) -> writer.addField(String.format("%.1f", value.doubleValue())))
    .withDateTypeHandler((writer, value) -> writer.addField(new SimpleDateFormat("yyyy-MM-dd").format(value)))
    .withBooleanTypeHandler((writer, value) -> writer.addField(value ? "Y" : "N"));
```

## Excel Target

```java
ExcelTargetConfig config = ExcelTargetConfig.from(FileDataOutputStream.from("/data/out.xlsx", true))
    .withPrimarySheetName("Results")
    .withHeaders(List.of("Name", "Score", "Link"));

try (ExcelTarget target = ExcelTarget.withConfig(config)) {
    target.appendValue("John");
    target.appendValue(99.5);
    target.appendValue(new Link("https://example.com", "Example"));
    target.finishRow();
}
```

### Multiple Sheets
```java
try (ExcelTarget target = ExcelTarget.withConfig(config)) {
    // Write to first sheet (configured above)
    target.writeRow("John", 100);

    // Create additional sheets
    target.newSheet("Summary", List.of("Metric", "Value"));
    target.writeRow("Total", 500);
    target.writeRow("Average", 83.3);

    // Sheet without headers
    target.newSheet("Raw");
    target.writeRow("raw", "data");
}
```

### Excel Default Formatting
| Type | Default Format |
|------|---------------|
| String | Plain text (truncated at 32K chars) |
| Number (float/double) | 0.00 format |
| Date | m/d/yy |
| Boolean | Native boolean cell |
| Link | Blue underlined hyperlink |
| Header | Bold, centered, 10pt |
| Collection | Joined with list delimiter |

### Custom Cell Handlers
```java
ExcelTargetConfig config = ExcelTargetConfig.from(dataOutputStream)
    .withNumberTypeHandler((cellRef, value) -> {
        cellRef.cell().setCellValue(value.doubleValue());
        // Access WorkbookHelper for custom styles
        CellStyle style = cellRef.workbookHelper().createOrGetStyle("custom", s -> {
            s.setDataFormat(cellRef.workbookHelper().getWorkbook().createDataFormat().getFormat("#,##0.00"));
        });
        cellRef.cell().setCellStyle(style);
    });
```

### Red Bold Cell Handler
A built-in handler for writing red bold formatted cells, useful for emphasizing specific values:
```java
ExcelTargetConfig config = ExcelTargetConfig.from(dataOutputStream)
    .withRedBoldHandler(new RedBoldCellHandler());
```

### Merged Regions
Cells in an Excel sheet can be merged using `addMergedRegion`:
```java
try (ExcelTarget target = ExcelTarget.withConfig(config)) {
    target.writeRow("Merged Title");
    // merge row 0, columns 0 through 3
    target.addMergedRegion(0, 0, 0, 3);
    target.writeRow("A", "B", "C", "D");
}
```

## JSON Lines Target

```java
JsonLineDataTargetConfig config = JsonLineDataTargetConfig.from(dataOutputStream);
try (JsonLinesDataTarget target = new JsonLinesDataTarget(config)) {
    Document doc = new Document("name", "John").append("age", 30);
    target.writeRecord(doc);
}
```

# Compression

GZIP compression is handled automatically based on file extension.

```java
// Reading compressed files
SpreadsheetSource<?> source = SpreadsheetSourceFactory.fromFileWithHeaders("/data/input.csv.gz");

// Writing compressed files
SpreadsheetTarget<?, ?> target = SpreadsheetTargetFactory.fromFile("/data/output.csv.gz");
```

# Input/Output Streams

The library uses its own stream abstractions that carry metadata for format detection.

### Reading
```java
// From a file path
FileDataInputStream input = FileDataInputStream.from("/data/test.csv");

// From a raw InputStream (single use, no reset)
DataStreamMeta meta = DataStreamMeta.fromFileName("data.csv");
SingleUseDataInputStream input = SingleUseDataInputStream.from(inputStream, meta);

// Convenience overloads infer the metadata from a filename (and optional content type)
SingleUseDataInputStream fromName = SingleUseDataInputStream.from(inputStream, "data.csv");

// From an in-memory byte array
SingleUseDataInputStream fromBytes = SingleUseDataInputStream.from(bytes, "data.csv");
```

### Writing
```java
// To a file path
FileDataOutputStream output = FileDataOutputStream.from("/data/out.csv", true); // overwrite=true

// To a raw OutputStream
DataStreamMeta meta = DataStreamMeta.fromFileName("data.csv");
SingleUseDataOutputStream output = SingleUseDataOutputStream.from(outputStream, meta);
```
