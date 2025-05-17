src/
└── main/
    ├── java/
    │   └── com/example/csvmerge/
    │       ├── controller/
    │       │   └── CsvMergeController.java
    │       ├── model/
    │       │   └── CsvRow.java
    │       ├── service/
    │       │   └── CsvMergeService.java
    │       └── CsvMergeApplication.java
    └── resources/
        └── application.properties

🟩 1. pom.xml (Add OpenCSV)
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.8</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
🟩 2. CsvMergeApplication.java
package com.example.csvmerge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CsvMergeApplication {
    public static void main(String[] args) {
        SpringApplication.run(CsvMergeApplication.class, args);
    }
}
🟦 3. model/CsvRow.java
package com.example.csvmerge.model;

import java.util.HashMap;
import java.util.Map;

public class CsvRow {
    private Map<String, String> fields = new HashMap<>();

    public void setField(String key, String value) {
        fields.put(key, value);
    }

    public String getField(String key) {
        return fields.getOrDefault(key, "");
    }

    public Map<String, String> getAllFields() {
        return fields;
    }
}
🟨 4. service/CsvMergeService.java
package com.example.csvmerge.service;

import com.example.csvmerge.model.CsvRow;
import com.opencsv.CSVReader;
import com.opencsv.CSVWriter;
import org.springframework.stereotype.Service;

import java.io.*;
import java.util.*;

@Service
public class CsvMergeService {

    private final String initialMarginPath = "/path/to/initial_margin.csv";
    private final String kondorPath = "/path/to/kondor.csv";
    private final String outputPath = "merged_output.csv";

    public File mergeCsvFiles() throws Exception {
        List<CsvRow> mergedRows = new ArrayList<>();
        Set<String> allHeaders = new LinkedHashSet<>();

        // Load initial margin
        try (CSVReader reader = new CSVReader(new FileReader(initialMarginPath))) {
            String[] headers = reader.readNext();

            allHeaders.addAll(Arrays.asList(headers));
            allHeaders.add("type nature");
            allHeaders.add("site code");

            String[] row;
            while ((row = reader.readNext()) != null) {
                CsvRow csvRow = new CsvRow();
                for (int i = 0; i < headers.length; i++) {
                    csvRow.setField(headers[i], row[i]);
                }
                csvRow.setField("type nature", "OP");
                csvRow.setField("site code", "3428");
                mergedRows.add(csvRow);
            }
        }

        // Load kondor
        try (CSVReader reader = new CSVReader(new FileReader(kondorPath))) {
            String[] headers = reader.readNext();
            allHeaders.addAll(Arrays.asList(headers));
            allHeaders.add("base currency");
            allHeaders.add("call amount");
            allHeaders.add("rate");
            allHeaders.add("site code");

            String[] row;
            while ((row = reader.readNext()) != null) {
                CsvRow csvRow = new CsvRow();
                for (int i = 0; i < headers.length && i < row.length; i++) {
                    csvRow.setField(headers[i], row[i]);
                }

                // Special columns from kondor:
                csvRow.setField("type nature", csvRow.getField("type nature"));
                csvRow.setField("base currency", row.length > 22 ? row[22] : "");
                csvRow.setField("call amount", row.length > 21 ? row[21] : "");
                csvRow.setField("rate", row.length > 80 ? row[80] : "");
                csvRow.setField("site code", csvRow.getField("site code"));

                mergedRows.add(csvRow);
            }
        }

        // Write merged file
        List<String> headersList = new ArrayList<>(allHeaders);

        try (CSVWriter writer = new CSVWriter(new FileWriter(outputPath))) {
            writer.writeNext(headersList.toArray(new String[0]));

            for (CsvRow row : mergedRows) {
                String[] values = new String[headersList.size()];
                for (int i = 0; i < headersList.size(); i++) {
                    values[i] = row.getField(headersList.get(i));
                }
                writer.writeNext(values);
            }
        }

        return new File(outputPath);
    }
}
🟥 5. controller/CsvMergeController.java
package com.example.csvmerge.controller;

import com.example.csvmerge.service.CsvMergeService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.InputStreamResource;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.File;
import java.io.FileInputStream;

@RestController
public class CsvMergeController {

    @Autowired
    private CsvMergeService mergeService;

    @GetMapping("/download-csv")
    public ResponseEntity<Resource> downloadMergedCsv() throws Exception {
        File mergedFile = mergeService.mergeCsvFiles();

        InputStreamResource resource = new InputStreamResource(new FileInputStream(mergedFile));

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment;filename=" + mergedFile.getName())
                .contentType(MediaType.parseMediaType("text/csv"))
                .contentLength(mergedFile.length())
                .body(resource);
    }
}
http://localhost:8080/download-csv
------------------------------------------
 Create Model Class CsvRow.java
This will represent one merged CSV row as a Map of column name -> value.

java
Copy
Edit
package com.example.csvmerge.model;

import java.util.LinkedHashMap;
import java.util.Map;

public class CsvRow {
    private Map<String, String> columns = new LinkedHashMap<>();

    public CsvRow() {}

    public void setColumn(String key, String value) {
        columns.put(key, value);
    }

    public String getColumn(String key) {
        return columns.get(key);
    }

    public Map<String, String> getColumns() {
        return columns;
    }
}
Step 2: Service Class CsvMergeService.java
It reads the two CSV files, merges rows with your logic, and returns the merged CSV content as a byte array (for download).

java
Copy
Edit
package com.example.csvmerge.service;

import com.example.csvmerge.model.CsvRow;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVPrinter;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.*;

@Service
public class CsvMergeService {

    public byte[] mergeCsvFiles(MultipartFile initialMarginFile, MultipartFile kondorFile) throws IOException {
        // Parse initial margin CSV
        Reader initReader = new InputStreamReader(initialMarginFile.getInputStream(), StandardCharsets.UTF_8);
        CSVParser initParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(initReader);

        // Parse kondor CSV
        Reader kondorReader = new InputStreamReader(kondorFile.getInputStream(), StandardCharsets.UTF_8);
        CSVParser kondorParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(kondorReader);

        // Headers from initial margin and kondor
        Set<String> initHeaders = initParser.getHeaderMap().keySet();
        List<String> kondorHeadersList = new ArrayList<>(kondorParser.getHeaderMap().keySet());

        // We'll create a merged header list with unique headers:
        // Start with all initial margin headers
        LinkedHashSet<String> mergedHeaders = new LinkedHashSet<>(initHeaders);

        // Add kondor headers except those merged/mapped specially (type nature, site code, base currency, call amount, rate)
        // Kondor has 'type nature' and 'site code' columns - add if not in mergedHeaders
        // base currency, call amount, rate are mapped to initial margin header names

        // Add 'type nature' header if not present (initial margin has none)
        mergedHeaders.add("type nature");
        // Add 'site code' header if not present
        mergedHeaders.add("site code");

        // base currency, call amount, rate already in initial margin, so don't add kondor's column names

        // Add kondor headers except the special mapped ones and those already in initial margin headers
        for (String kHeader : kondorHeadersList) {
            // skip kondor cols: type nature, site code, base currency (col 22), call amount (col 21), rate (col 80)
            if (kHeader.equalsIgnoreCase("type nature") || kHeader.equalsIgnoreCase("site code")) {
                // already handled
                continue;
            }
            // We have to skip kondor col 22 (base currency), 21 (call amount), 80 (rate) based on index, not header name
            // So handle them by index later, so here add only those columns that are not mapped
            // We'll add kondor columns except those mapped by index to initial margin columns

            // Find kondor col index to skip mapped cols by index
            int kIndex = kondorParser.getHeaderMap().get(kHeader);

            if (kIndex == 20 || kIndex == 21 || kIndex == 79) { // zero-based indexes for call amount(21), base currency(22), rate(80)
                // skip mapped cols
                continue;
            }
            // if mergedHeaders already contains this header, skip it (to avoid duplicates)
            if (!mergedHeaders.contains(kHeader)) {
                mergedHeaders.add(kHeader);
            }
        }

        List<CsvRow> mergedRows = new ArrayList<>();

        // Process initial margin rows
        for (CSVRecord initRecord : initParser) {
            CsvRow row = new CsvRow();
            // copy all initial margin columns
            for (String h : initHeaders) {
                row.setColumn(h, initRecord.get(h));
            }
            // add type nature = "type nature-OP"
            row.setColumn("type nature", "type nature-OP");

            // add site code = 3428
            row.setColumn("site code", "3428");

            // call amount, base currency, rate are present in initial margin already - no change

            // For columns present only in kondor but not in initial margin or mergedHeaders - set null
            for (String h : mergedHeaders) {
                if (!row.getColumns().containsKey(h)) {
                    row.setColumn(h, "");
                }
            }

            mergedRows.add(row);
        }

        // Process kondor rows
        for (CSVRecord kondorRecord : kondorParser) {
            CsvRow row = new CsvRow();

            // For columns from initial margin: base currency, call amount, rate
            // base currency in initial margin - use kondor col 22 (index 21)
            String baseCurrency = kondorRecord.get(kondorParser.getHeaderMap().entrySet().stream()
                    .filter(e -> e.getValue() == 21)
                    .findFirst().get().getKey());
            row.setColumn("base currency", baseCurrency);

            // call amount initial margin col - kondor col 21 (index 20)
            String callAmount = kondorRecord.get(kondorParser.getHeaderMap().entrySet().stream()
                    .filter(e -> e.getValue() == 20)
                    .findFirst().get().getKey());
            row.setColumn("call amount", callAmount);

            // rate initial margin col - kondor col 80 (index 79)
            String rate = kondorRecord.get(kondorParser.getHeaderMap().entrySet().stream()
                    .filter(e -> e.getValue() == 79)
                    .findFirst().get().getKey());
            row.setColumn("rate", rate);

            // type nature from kondor (present)
            String typeNature = kondorRecord.get("type nature");
            row.setColumn("type nature", typeNature);

            // site code from kondor
            String siteCode = kondorRecord.get("site code");
            row.setColumn("site code", siteCode);

            // Now add other columns from kondor (excluding mapped cols 21,22,80, type nature, site code)
            for (String kHeader : kondorHeadersList) {
                if (kHeader.equalsIgnoreCase("type nature") || kHeader.equalsIgnoreCase("site code")) continue;
                int kIndex = kondorParser.getHeaderMap().get(kHeader);
                if (kIndex == 20 || kIndex == 21 || kIndex == 79) continue;

                // if initial margin has this column, but kondor also has? unlikely, but just put kondor's value here
                row.setColumn(kHeader, kondorRecord.get(kHeader));
            }

            // For columns present in initial margin but missing in kondor, set null or empty
            for (String h : mergedHeaders) {
                if (!row.getColumns().containsKey(h)) {
                    row.setColumn(h, "");
                }
            }

            mergedRows.add(row);
        }

        // Write merged CSV to byte array
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        OutputStreamWriter osw = new OutputStreamWriter(baos, StandardCharsets.UTF_8);

        CSVPrinter csvPrinter = new CSVPrinter(osw, CSVFormat.DEFAULT.withHeader(mergedHeaders.toArray(new String[0])));

        for (CsvRow r : mergedRows) {
            List<String> rowValues = new ArrayList<>();
            for (String h : mergedHeaders) {
                String val = r.getColumn(h);
                rowValues.add(val == null ? "" : val);
            }
            csvPrinter.printRecord(rowValues);
        }
        csvPrinter.flush();

        return baos.toByteArray();
    }
}
Step 3: Controller Class CsvMergeController.java
This will expose an endpoint to upload the two CSV files and return the merged CSV file for download.

java
Copy
Edit
package com.example.csvmerge.controller;

import com.example.csvmerge.service.CsvMergeService;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("/api/csv")
public class CsvMergeController {

    private final CsvMergeService csvMergeService;

    public CsvMergeController(CsvMergeService csvMergeService) {
        this.csvMergeService = csvMergeService;
    }

    @PostMapping(value = "/merge", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<byte[]> mergeCsv(
            @RequestParam("initialMargin") MultipartFile initialMarginFile,
            @RequestParam("kondor") MultipartFile kondorFile) {
        try {
            byte[] mergedCsvBytes = csvMergeService.mergeCsvFiles(initialMarginFile, kondorFile);

            return ResponseEntity.ok()
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"merged.csv\"")
                    .contentType(MediaType.parseMediaType("text/csv"))
                    .body(mergedCsvBytes);

        } catch (Exception e) {
            e.printStackTrace();
            return ResponseEntity.status(500).body(("Error merging CSV files: " + e.getMessage()).getBytes());
        }
    }
}
Step 4: Main Application CsvMergeApplication.java
java
Copy
Edit
package com.example.csvmerge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CsvMergeApplication {

    public static void main(String[] args) {
        SpringApplication.run(CsvMergeApplication.class, args);
    }
}
Step 5: Maven Dependencies
Add Apache Commons CSV to your pom.xml for CSV parsing:

xml
Copy
Edit
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-csv</artifactId>
    <version>1.10.0</version>
</dependency>
Step 6: How to Run & Use
Build and run the Spring Boot app.

Use a tool like Postman or curl to POST the two CSV files (initialMargin and kondor) to

bash
Copy
Edit
POST http://localhost:8080/api/csv/merge
Content-Type: multipart/form-data
Form Data:
  initialMargin: (select initial margin CSV file)
  kondor: (select kondor CSV file)
The response will be the merged CSV file merged.csv downloadable directly.

If you want, I can also help you with a simple HTML upload form to test the app from browser.

If you want me to generate a full working zip project, just say!







You said:
yes


ChatGPT said:
Got it! I'll provide you a step-by-step working Spring Boot application code structure that:

Reads two CSV files locally (paths given in code).

Merges them based on your rules.

Creates a new CSV file with merged data.

Provides a REST API endpoint to download the merged CSV file.

Assumptions
You have two CSV files:

initial_margin.csv (1595 rows + header)

kondor.csv (458 rows + header)

Both have headers, but some columns only exist in one file.

You'll place files in a known folder (e.g., src/main/resources/files/).

Step 1: Project Setup
Create this folder structure:

swift
Copy
Edit
src/main/java/com/example/csvmerge/
    CsvMergeApplication.java
    model/
        CsvRow.java
    controller/
        CsvMergeController.java
    service/
        CsvMergeService.java
src/main/resources/files/
    initial_margin.csv
    kondor.csv
Step 2: Dependencies in pom.xml
Make sure you have dependencies for Spring Web and Apache Commons CSV for CSV reading/writing.

xml
Copy
Edit
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-csv</artifactId>
        <version>1.10.0</version>
    </dependency>
</dependencies>
Step 3: Code files
CsvMergeApplication.java
java
Copy
Edit
package com.example.csvmerge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CsvMergeApplication {

    public static void main(String[] args) {
        SpringApplication.run(CsvMergeApplication.class, args);
    }
}
model/CsvRow.java
We won't create a strict POJO for rows since columns differ and are many. We'll use a Map<String,String> to hold column-value pairs.

java
Copy
Edit
package com.example.csvmerge.model;

import java.util.HashMap;
import java.util.Map;

public class CsvRow {
    private Map<String, String> columns = new HashMap<>();

    public Map<String, String> getColumns() {
        return columns;
    }

    public void setColumns(Map<String, String> columns) {
        this.columns = columns;
    }

    public String getValue(String column) {
        return columns.get(column);
    }

    public void setValue(String column, String value) {
        columns.put(column, value);
    }
}
service/CsvMergeService.java
This will do the CSV reading, merging and writing.

java
Copy
Edit
package com.example.csvmerge.service;

import com.example.csvmerge.model.CsvRow;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVPrinter;
import org.apache.commons.csv.CSVRecord;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.*;

@Service
public class CsvMergeService {

    private final String initialMarginPath = "src/main/resources/files/initial_margin.csv";
    private final String kondorPath = "src/main/resources/files/kondor.csv";
    private final String mergedCsvPath = "src/main/resources/files/merged.csv";

    public String mergeCsvFiles() throws IOException {

        // Read initial margin CSV
        List<CsvRow> initialRows = readCsv(initialMarginPath);

        // Read kondor CSV
        List<CsvRow> kondorRows = readCsv(kondorPath);

        // Extract headers
        Set<String> allHeaders = new LinkedHashSet<>();

        // Headers from initial margin
        Set<String> initialHeaders = initialRows.isEmpty() ? Collections.emptySet() : initialRows.get(0).getColumns().keySet();
        allHeaders.addAll(initialHeaders);

        // Headers from kondor
        Set<String> kondorHeaders = kondorRows.isEmpty() ? Collections.emptySet() : kondorRows.get(0).getColumns().keySet();

        // We know that kondor has columns like:
        // "Type Nature" present only in kondor
        // base currency at column 22 (index 21)
        // call amount at column 21 (index 20)
        // rate at column 80 (index 79)
        // site code present only in kondor

        // Normalize headers as per your requirement:
        // We will create common headers:
        // type nature, base currency, site code, call amount, rate, plus other columns.

        // Remove redundant headers from kondor and replace with mapped headers
        // The CSV records have header names from the CSV files.
        // We have to identify the exact header names from kondor for those columns.

        // Let's get kondor headers list as List to get by index
        List<String> kondorHeaderList = new ArrayList<>(kondorHeaders);

        // Identify key kondor headers from index:
        // base currency: col 22 = index 21
        String kondorBaseCurrencyCol = kondorHeaderList.size() > 21 ? kondorHeaderList.get(21) : null;
        // call amount: col 21 = index 20
        String kondorCallAmountCol = kondorHeaderList.size() > 20 ? kondorHeaderList.get(20) : null;
        // rate: col 80 = index 79
        String kondorRateCol = kondorHeaderList.size() > 79 ? kondorHeaderList.get(79) : null;

        // type nature column is present in kondor but not in initial margin
        String kondorTypeNatureCol = "Type Nature";
        // site code column in kondor
        String kondorSiteCodeCol = "Site Code";

        // Remove these specific columns from kondorHeaders to avoid duplication
        // We'll add single columns in final headers
        allHeaders.remove(kondorTypeNatureCol);
        allHeaders.remove(kondorSiteCodeCol);
        allHeaders.remove(kondorBaseCurrencyCol);
        allHeaders.remove(kondorCallAmountCol);
        allHeaders.remove(kondorRateCol);

        // Add unified headers
        allHeaders.add("Type Nature");
        allHeaders.add("Site Code");
        allHeaders.add("Base Currency");
        allHeaders.add("Call Amount");
        allHeaders.add("Rate");

        // Prepare merged rows
        List<Map<String, String>> mergedRows = new ArrayList<>();

        // Add initial margin rows with modifications
        for (CsvRow row : initialRows) {
            Map<String, String> map = new HashMap<>();

            // Copy all initial columns
            for (String h : initialHeaders) {
                map.put(h, row.getValue(h));
            }

            // type nature - as per your instruction: type nature-OP for all initial margin rows
            map.put("Type Nature", "type nature-OP");

            // site code = 3428
            map.put("Site Code", "3428");

            // base currency - from initial margin base currency col
            String baseCurrencyVal = row.getValue("Base Currency");
            map.put("Base Currency", baseCurrencyVal != null ? baseCurrencyVal : "");

            // call amount
            String callAmountVal = row.getValue("Call Amount");
            map.put("Call Amount", callAmountVal != null ? callAmountVal : "");

            // rate
            String rateVal = row.getValue("Rate");
            map.put("Rate", rateVal != null ? rateVal : "");

            // For other columns that exist only in kondor but not initial margin, put null
            for (String h : allHeaders) {
                if (!map.containsKey(h)) {
                    map.put(h, null);
                }
            }

            mergedRows.add(map);
        }

        // Add kondor rows with modifications
        for (CsvRow row : kondorRows) {
            Map<String, String> map = new HashMap<>();

            // Copy all kondor columns except those handled differently
            for (String h : kondorHeaders) {
                if (h.equals(kondorTypeNatureCol) || h.equals(kondorSiteCodeCol)
                        || h.equals(kondorBaseCurrencyCol) || h.equals(kondorCallAmountCol)
                        || h.equals(kondorRateCol)) {
                    continue; // skip these, will map manually below
                }
                map.put(h, row.getValue(h));
            }

            // type nature from kondor column
            map.put("Type Nature", row.getValue(kondorTypeNatureCol));

            // site code from kondor column
            map.put("Site Code", row.getValue(kondorSiteCodeCol));

            // base currency from kondor col 22 (index 21)
            map.put("Base Currency", row.getValue(kondorBaseCurrencyCol));

            // call amount from kondor col 21 (index 20)
            map.put("Call Amount", row.getValue(kondorCallAmountCol));

            // rate from kondor col 80 (index 79)
            map.put("Rate", row.getValue(kondorRateCol));

            // For other columns that exist only in initial margin but not kondor, put null
            for (String h : allHeaders) {
                if (!map.containsKey(h)) {
                    map.put(h, null);
                }
            }

            mergedRows.add(map);
        }

        // Write merged CSV
        writeMergedCsv(new ArrayList<>(allHeaders), mergedRows, mergedCsvPath);

        return mergedCsvPath;
    }

    private List<CsvRow> readCsv(String filePath) throws IOException {
        List<CsvRow> rows = new ArrayList<>();

        try (Reader reader = Files.newBufferedReader(Paths.get(filePath));
             CSVParser csvParser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader)) {

            for (CSVRecord csvRecord : csvParser) {
                CsvRow row = new CsvRow();

                for (String header : csvParser.getHeaderMap().keySet()) {
                    row.setValue(header, csvRecord.get(header));
                }

                rows.add(row);
            }
        }
        return rows;
    }

    private void writeMergedCsv(List<String> headers, List<Map<String, String>> rows, String outputPath) throws IOException {
        try (BufferedWriter writer = Files.newBufferedWriter(Paths.get(outputPath));
             CSVPrinter csvPrinter = new CSVPrinter(writer, CSVFormat.DEFAULT.withHeader(headers.toArray(new String[0])))) {

            for (Map<String, String> row : rows) {
                List<String> rowData = new ArrayList<>();
                for (String header : headers) {
                    rowData.add(row.get(header));
                }
                csvPrinter.printRecord(rowData);
            }

            csvPrinter.flush();
        }
    }
}
controller/CsvMergeController.java
Controller exposes an endpoint /download to trigger merge and download file.

java
Copy
Edit
package com.example.csvmerge.controller;

import com.example.csvmerge.service.CsvMergeService;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.*;

@RestController
public class CsvMergeController {

    private final CsvMergeService csvMergeService;

    public CsvMergeController(CsvMergeService csvMergeService) {
        this.csvMergeService = csvMergeService;
    }

    @GetMapping("/download")
    public ResponseEntity<InputStreamResource> downloadMergedCsv() throws IOException {

        // Merge CSV files and get path of merged file
        String mergedCsvPath = csvMergeService.mergeCsvFiles();

        File file = new File(mergedCsvPath);

        InputStreamResource resource = new InputStreamResource(new FileInputStream(file));

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment;filename=merged.csv")
                .contentType(MediaType.parseMediaType("text/csv"))
                .contentLength(file.length())
                .body(resource);
    }
}

http://localhost:8080/merge?initialMarginPath=/home/user/csv/initial_margin.csv&kondorPath=/home/user/csv/kondor.csv
