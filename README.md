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

. File Structure
Place your CSVs like this:

bash
Copy
Edit
C:/csvfiles/initial_margin.csv  
C:/csvfiles/kondor.csv  
2. CsvMergeController.java
java
Copy
Edit
package com.example.csvmerge.controller;

import com.example.csvmerge.service.CsvMergeService;
import org.springframework.core.io.InputStreamResource;
import org.springframework.core.io.Resource;
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

    @GetMapping("/download-merged")
    public ResponseEntity<Resource> downloadMergedFile() throws IOException {
        File initialFile = new File("C:/csvfiles/initial_margin.csv");
        File kondorFile = new File("C:/csvfiles/kondor.csv");

        File mergedFile = csvMergeService.mergeCsvFiles(initialFile, kondorFile);

        InputStreamResource resource = new InputStreamResource(new FileInputStream(mergedFile));

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=" + mergedFile.getName())
                .contentType(MediaType.parseMediaType("text/csv"))
                .body(resource);
    }
}
3. CsvMergeService.java
Make sure the merged file is saved under C:/csvfiles/merged_output.csv:

java
Copy
Edit
package com.example.csvmerge.service;

import com.example.csvmerge.model.CsvRow;
import org.apache.commons.csv.*;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

@Service
public class CsvMergeService {

    public File mergeCsvFiles(File initialMarginFile, File kondorFile) throws IOException {
        List<CsvRow> mergedRows = new ArrayList<>();

        // Read Initial Margin
        try (Reader reader = new InputStreamReader(new FileInputStream(initialMarginFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);

            for (CSVRecord record : parser) {
                CsvRow row = new CsvRow();
                row.setTypeNature("OP");
                row.setBaseCurrency(record.get("Base Currency"));
                row.setSiteCode("3428");
                row.setCallAmount(record.get("Call Amount"));
                row.setRate(record.get("Rate"));
                mergedRows.add(row);
            }
        }

        // Read Kondor
        try (Reader reader = new InputStreamReader(new FileInputStream(kondorFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);

            for (CSVRecord record : parser) {
                CsvRow row = new CsvRow();
                row.setTypeNature(record.get("Type Nature"));
                row.setBaseCurrency(record.get(21)); // Column 22
                row.setSiteCode(record.get("Site Code"));
                row.setCallAmount(record.get(20));   // Column 21
                row.setRate(record.get(79));         // Column 80
                mergedRows.add(row);
            }
        }

        // Write merged CSV
        File outputFile = new File("C:/csvfiles/merged_output.csv");
        try (Writer writer = new FileWriter(outputFile, StandardCharsets.UTF_8);
             CSVPrinter printer = new CSVPrinter(writer, CSVFormat.DEFAULT.builder()
                     .setHeader("Type Nature", "Base Currency", "Site Code", "Call Amount", "Rate")
                     .build())) {

            for (CsvRow row : mergedRows) {
                printer.printRecord(
                        row.getTypeNature(),
                        row.getBaseCurrency(),
                        row.getSiteCode(),
                        row.getCallAmount(),
                        row.getRate()
                );
            }
        }

        return outputFile;
    }
}
4. CsvRow.java (Model Class)
java
Copy
Edit
package com.example.csvmerge.model;

public class CsvRow {
    private String typeNature;
    private String baseCurrency;
    private String siteCode;
    private String callAmount;
    private String rate;

    // Getters and Setters
    public String getTypeNature() { return typeNature; }
    public void setTypeNature(String typeNature) { this.typeNature = typeNature; }

    public String getBaseCurrency() { return baseCurrency; }
    public void setBaseCurrency(String baseCurrency) { this.baseCurrency = baseCurrency; }

    public String getSiteCode() { return siteCode; }
    public void setSiteCode(String siteCode) { this.siteCode = siteCode; }

    public String getCallAmount() { return callAmount; }
    public void setCallAmount(String callAmount) { this.callAmount = callAmount; }

    public String getRate() { return rate; }
    public void setRate(String rate) { this.rate = rate; }
}
To Test:
Place your CSV files at C:/csvfiles/initial_margin.csv and C:/csvfiles/kondor.csv.

Run your Spring Boot application.

Open your browser and go to:

bash
Copy
Edit
http://localhost:8080/download-merged
It will trigger a download of merged_output.csv.


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

