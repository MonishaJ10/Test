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

// CsvMergeApplication.java (Main class) package com.example.csvmerge;

import org.springframework.boot.SpringApplication; import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication public class CsvMergeApplication { public static void main(String[] args) { SpringApplication.run(CsvMergeApplication.class, args); } }

// CsvMergeController.java package com.example.csvmerge.controller;

import com.example.csvmerge.service.CsvMergeService; import org.springframework.beans.factory.annotation.Autowired; import org.springframework.core.io.InputStreamResource; import org.springframework.http.; import org.springframework.web.bind.annotation.;

import java.io.*;

@RestController @RequestMapping("/api/csv") public class CsvMergeController {

@Autowired
private CsvMergeService csvMergeService;

@GetMapping("/merge")
public ResponseEntity<InputStreamResource> mergeAndDownloadCsv() {
    try {
        File mergedFile = csvMergeService.mergeCsvFiles();
        InputStreamResource resource = new InputStreamResource(new FileInputStream(mergedFile));

        HttpHeaders headers = new HttpHeaders();
        headers.add(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=merged_output.csv");

        return ResponseEntity.ok()
                .headers(headers)
                .contentLength(mergedFile.length())
                .contentType(MediaType.parseMediaType("text/csv"))
                .body(resource);

    } catch (Exception e) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    }
}

}

// CsvMergeService.java package com.example.csvmerge.service;

import org.apache.commons.csv.CSVFormat; import org.apache.commons.csv.CSVParser; import org.apache.commons.csv.CSVPrinter; import org.apache.commons.csv.CSVRecord; import org.springframework.stereotype.Service;

import java.io.; import java.nio.charset.StandardCharsets; import java.util.;

@Service public class CsvMergeService {

private final String initialMarginPath = "C:/Users/h59606/Downloads/Initial_Margin_E0D_20250307.csv";
private final String kondorPath = "C:/Users/h59606/Downloads/KONDORFX.csv";

public File mergeCsvFiles() throws IOException {
    List<Map<String, String>> records = new ArrayList<>();
    Set<String> headersSet = new LinkedHashSet<>();

    // Load Initial Margin CSV
    try (Reader reader = new InputStreamReader(new FileInputStream(initialMarginPath), StandardCharsets.UTF_8)) {
        CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
        List<String> headers = parser.getHeaderNames();
        headersSet.addAll(headers);
        for (CSVRecord record : parser) {
            Map<String, String> row = new LinkedHashMap<>();
            for (String header : headers) {
                row.put(header, record.get(header));
            }
            records.add(row);
        }
    }

    // Load KONDOR CSV (specific columns only)
    try (Reader reader = new InputStreamReader(new FileInputStream(kondorPath), StandardCharsets.UTF_8)) {
        CSVParser parser = CSVFormat.DEFAULT.parse(reader);
        for (CSVRecord record : parser) {
            // Skip if row is empty
            if (record.size() <= 80) continue;

            Map<String, String> row = new LinkedHashMap<>();
            row.put("Type", record.get(20));
            row.put("Nature", record.get(20));
            row.put("Site Code", record.get(20));
            row.put("Call Amount", record.get(21));
            row.put("Base Currency", record.get(22));
            row.put("Rate", record.get(80));

            headersSet.addAll(Arrays.asList("Type", "Nature", "Site Code", "Call Amount", "Base Currency", "Rate"));
            records.add(row);
        }
    }

    File outputFile = new File("merged_output.csv");

    try (BufferedWriter writer = new BufferedWriter(new FileWriter(outputFile));
         CSVPrinter csvPrinter = new CSVPrinter(writer, CSVFormat.DEFAULT.withHeader(headersSet.toArray(new String[0])))) {

        for (Map<String, String> row : records) {
            List<String> rowValues = new ArrayList<>();
            for (String header : headersSet) {
                rowValues.add(row.getOrDefault(header, ""));
            }
            csvPrinter.printRecord(rowValues);
        }
    }

    return outputFile;
}

}

