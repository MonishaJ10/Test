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
__________________________________________________________________________________________________________________________________
✅ Update CsvMergeApplication.java to run on startup:

package com.example.csvmerge.service;

import org.apache.commons.csv.*;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.*;

@Service
public class CsvMergeService {

    public File mergeCsvFiles(File initialMarginFile, File kondorFile) throws IOException {
        List<Map<String, String>> mergedData = new ArrayList<>();
        Set<String> headerSet = new LinkedHashSet<>();

        // Process Initial Margin File
        try (Reader reader = new InputStreamReader(new FileInputStream(initialMarginFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT
                    .builder()
                    .setHeader()
                    .setSkipHeaderRecord(true)
                    .build()
                    .parse(reader);

            for (CSVRecord record : parser) {
                Map<String, String> row = new LinkedHashMap<>();
                row.put("Type Nature", "OP");
                row.put("Site Code", "3428");
                row.put("Base Currency", record.get("Base Currency"));
                row.put("Call Amount", record.get("Call Amount"));
                row.put("Rate", record.get("Rate"));

                headerSet.addAll(row.keySet());
                mergedData.add(row);
            }
        }

        // Process Kondor File
        try (Reader reader = new InputStreamReader(new FileInputStream(kondorFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT
                    .builder()
                    .setHeader()
                    .setSkipHeaderRecord(true)
                    .build()
                    .parse(reader);

            Map<String, Integer> headerMap = parser.getHeaderMap();

            for (CSVRecord record : parser) {
                Map<String, String> row = new LinkedHashMap<>();

                for (String header : headerMap.keySet()) {
                    String value = record.get(header);

                    // Map specific column names (strings) to expected output headers
                    if (header.equals("21")) {
                        row.put("Call Amount", value);
                        headerSet.add("Call Amount");
                    } else if (header.equals("22")) {
                        row.put("Base Currency", value);
                        headerSet.add("Base Currency");
                    } else if (header.equals("80")) {
                        row.put("Rate", value);
                        headerSet.add("Rate");
                    } else {
                        row.put(header, value);
                        headerSet.add(header);
                    }
                }

                mergedData.add(row);
            }
        }

        // Write to merged output CSV
        File outputFile = File.createTempFile("merged_output", ".csv");
        try (Writer writer = new FileWriter(outputFile, StandardCharsets.UTF_8);
             CSVPrinter printer = new CSVPrinter(writer,
                     CSVFormat.DEFAULT
                             .builder()
                             .setHeader(headerSet.toArray(new String[0]))
                             .build())) {

            for (Map<String, String> row : mergedData) {
                List<String> rowValues = new ArrayList<>();
                for (String header : headerSet) {
                    rowValues.add(row.getOrDefault(header, ""));
                }
                printer.printRecord(rowValues);
            }
        }

        return outputFile;
    }
}


csvrow
package com.example.csvmerge.model;

import java.util.HashMap;
import java.util.Map;

public class CsvRow {
    private final Map<String, String> fields = new HashMap<>();

    public void set(String columnName, String value) {
        fields.put(columnName, value);
    }

    public String get(String columnName) {
        return fields.getOrDefault(columnName, "");
    }

    public Map<String, String> getAllFields() {
        return fields;
    }

    @Override
    public String toString() {
        return fields.toString();
    }
}
_______________________________________________________________________________________________________________________________________________

Final Goal (Updated Requirements)
The final merged CSV must:

1. Include:
✅ All columns from Initial Margin file

✅ All columns from Kondor file

✅ Columns Rate, Call Amount, and Base Currency should:

Appear only once

Merge values from:

Initial Margin headers

Kondor column 80 (Rate), 21 (Call Amount), and 22 (Base Currency)

2. Avoid:
❌ Repeating Call Amount, Base Currency, and Rate under different column names

❌ Using technical column indexes as headers

3. Also:
Type Nature = "OP" for Initial Margin rows (since it’s missing there)

Site Code = "3428" for Initial Margin rows (since it’s missing there)

4. Access:
Access the file via http://localhost:8080/download-merged

Automatically download the file in the browser

Save it in the same path as the input files

✅ Directory Structure

src/main/java/com/example/csvmerge/
├── CsvMergeApplication.java
├── controller/
│   └── CsvMergeController.java
├── service/
│   └── CsvMergeService.java
└── model/
    └── CsvRow.java (not required anymore)
✅ Updated Code
CsvMergeApplication.java

package com.example.csvmerge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CsvMergeApplication {
    public static void main(String[] args) {
        SpringApplication.run(CsvMergeApplication.class, args);
    }
}
controller/CsvMergeController.java

package com.example.csvmerge.controller;

import com.example.csvmerge.service.CsvMergeService;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.File;
import java.io.FileInputStream;
import java.io.OutputStream;

@RestController
public class CsvMergeController {

    @Autowired
    private CsvMergeService csvMergeService;

    @GetMapping("/download-merged")
    public void downloadMergedCsv(HttpServletResponse response) throws Exception {
        File mergedFile = csvMergeService.mergeCsvFiles(
                new File("C:/your/path/initial_margin.csv"),
                new File("C:/your/path/kondor.csv")
        );

        response.setContentType("text/csv");
        response.setHeader("Content-Disposition", "attachment; filename=merged_output.csv");
        try (FileInputStream input = new FileInputStream(mergedFile);
             OutputStream out = response.getOutputStream()) {
            input.transferTo(out);
        }
    }
}
service/CsvMergeService.java

package com.example.csvmerge.service;

import org.apache.commons.csv.*;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.*;

@Service
public class CsvMergeService {

    public File mergeCsvFiles(File initialMarginFile, File kondorFile) throws IOException {
        List<Map<String, String>> mergedRecords = new ArrayList<>();
        Set<String> headerSet = new LinkedHashSet<>();

        // --- Process Initial Margin ---
        try (Reader reader = new InputStreamReader(new FileInputStream(initialMarginFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<String> headers = new ArrayList<>(parser.getHeaderMap().keySet());
            headerSet.addAll(headers);
            headerSet.add("Type Nature");
            headerSet.add("Site Code");

            for (CSVRecord record : parser) {
                Map<String, String> row = new LinkedHashMap<>();
                for (String header : headers) {
                    row.put(header, record.get(header));
                }
                row.put("Type Nature", "OP");
                row.put("Site Code", "3428");
                mergedRecords.add(row);
            }
        }

        // --- Process Kondor ---
        try (Reader reader = new InputStreamReader(new FileInputStream(kondorFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<String> kondorHeaders = new ArrayList<>(parser.getHeaderMap().keySet());
            headerSet.addAll(kondorHeaders);

            // Remove original indexed columns if present
            headerSet.remove("21");
            headerSet.remove("22");
            headerSet.remove("80");

            headerSet.add("Call Amount");   // from col 21
            headerSet.add("Base Currency"); // from col 22
            headerSet.add("Rate");          // from col 80

            for (CSVRecord record : parser) {
                Map<String, String> row = new LinkedHashMap<>();
                for (String header : kondorHeaders) {
                    row.put(header, record.get(header));
                }

                // Replace/override if these fields already exist
                row.put("Call Amount", record.get(20));     // col index 21
                row.put("Base Currency", record.get(21));   // col index 22
                row.put("Rate", record.get(79));            // col index 80
                mergedRecords.add(row);
            }
        }

        // --- Write to Output File ---
        File outputFile = new File("C:/your/path/merged_output.csv"); // Save in same path

        try (Writer writer = new OutputStreamWriter(new FileOutputStream(outputFile), StandardCharsets.UTF_8);
             CSVPrinter printer = new CSVPrinter(writer, CSVFormat.DEFAULT
                     .builder()
                     .setHeader(headerSet.toArray(new String[0]))
                     .build())) {

            for (Map<String, String> row : mergedRecords) {
                List<String> rowValues = new ArrayList<>();
                for (String header : headerSet) {
                    rowValues.add(row.getOrDefault(header, ""));
                }
                printer.printRecord(rowValues);
            }
        }

        return outputFile;
    }
}
✅ What to Do
🔧 Step 1: Replace paths

new File("C:/your/path/initial_margin.csv")
new File("C:/your/path/kondor.csv")
with the actual local paths of your files.

🚀 Step 2: Run the Spring Boot app
Use your IDE (like IntelliJ or VSCode) or run:

bash
Copy
Edit
./mvnw spring-boot:run
🌐 Step 3: Open browser
Go to:
http://localhost:8080/download-merged
The merged CSV will automatically be downloaded.
