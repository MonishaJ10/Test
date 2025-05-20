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


http://localhost:8080/download-merged
The merged CSV will automatically be downloaded.


@GetMapping("/view-merged")
public void viewMergedCsv(HttpServletResponse response) throws Exception {
    File mergedFile = csvMergeService.mergeCsvFiles(
            new File("C:/your/path/initial_margin.csv"),
            new File("C:/your/path/kondor.csv")
    );

    response.setContentType("text/html;charset=UTF-8");

    try (BufferedReader reader = new BufferedReader(new FileReader(mergedFile));
         OutputStream out = response.getOutputStream()) {

        StringBuilder html = new StringBuilder();
        html.append("<html><head><title>Merged CSV</title>")
            .append("<style>table { border-collapse: collapse; width: 100%; }")
            .append("th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }</style>")
            .append("</head><body><h2>Merged CSV Data</h2><table>");

        String line;
        boolean isHeader = true;

        while ((line = reader.readLine()) != null) {
            html.append("<tr>");
            String[] cells = line.split(",", -1);
            for (String cell : cells) {
                if (isHeader) {
                    html.append("<th>").append(cell).append("</th>");
                } else {
                    html.append("<td>").append(cell).append("</td>");
                }
            }
            html.append("</tr>");
            isHeader = false;
        }

        html.append("</table></body></html>");
        out.write(html.toString().getBytes(StandardCharsets.UTF_8));
    }
}

CsvMergeservice.java

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

        // --- Read Initial Margin File ---
        try (Reader reader = new InputStreamReader(new FileInputStream(initialMarginFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<String> headers = new ArrayList<>(parser.getHeaderMap().keySet());

            headerSet.addAll(headers);
            headerSet.add("Type Nature");
            headerSet.add("Site Code");

            for (CSVRecord record : parser) {
                Map<String, String> row = new LinkedHashMap<>();
                for (String header : headers) {
                    if (header.equalsIgnoreCase("Call Amount") ||
                        header.equalsIgnoreCase("Base Currency") ||
                        header.equalsIgnoreCase("Rate")) {
                        continue;
                    }
                    row.put(header, record.get(header));
                }
                row.put("Call Amount", record.get("Call Amount"));
                row.put("Base Currency", record.get("Base Currency"));
                row.put("Rate", record.get("Rate"));
                row.put("Type Nature", "OP");
                row.put("Site Code", "3428");

                mergedRecords.add(row);
            }
        }

        // --- Read Kondor File ---
        try (Reader reader = new InputStreamReader(new FileInputStream(kondorFile), StandardCharsets.UTF_8)) {
            CSVParser parser = CSVFormat.DEFAULT.withFirstRecordAsHeader().parse(reader);
            List<String> kondorHeaders = new ArrayList<>(parser.getHeaderMap().keySet());

            // Add headers except 16, 17, 38 (merged into existing)
            for (int i = 0; i < kondorHeaders.size(); i++) {
                if (i != 16 && i != 17 && i != 38 &&
                    !kondorHeaders.get(i).equalsIgnoreCase("Call Amount") &&
                    !kondorHeaders.get(i).equalsIgnoreCase("Base Currency") &&
                    !kondorHeaders.get(i).equalsIgnoreCase("Rate")) {
                    headerSet.add(kondorHeaders.get(i));
                }
            }

            headerSet.add("Call Amount");
            headerSet.add("Base Currency");
            headerSet.add("Rate");

            for (CSVRecord record : parser) {
                Map<String, String> row = new LinkedHashMap<>();

                for (int i = 0; i < record.size(); i++) {
                    if (i == 16 || i == 17 || i == 38) {
                        continue;
                    }

                    String header = i < kondorHeaders.size() ? kondorHeaders.get(i) : "Column" + i;

                    if (!header.equalsIgnoreCase("Call Amount") &&
                        !header.equalsIgnoreCase("Base Currency") &&
                        !header.equalsIgnoreCase("Rate")) {
                        row.put(header, record.get(i));
                    }
                }

                // Merge values from indices into corresponding fields
                row.put("Call Amount", safeGet(record, 16));
                row.put("Base Currency", safeGet(record, 17));
                row.put("Rate", safeGet(record, 38));

                mergedRecords.add(row);
            }
        }

        // --- Write to Output File ---
        File outputFile = new File("C:/your/path/merged_output.csv");

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

    private String safeGet(CSVRecord record, int index) {
        try {
            return record.get(index);
        } catch (Exception e) {
            return "";
        }
    }
}
----------------------------------
DATA CONNECTION 
1. MongoSpringBootApp.java

Path: src/main/java/com/example/mongoproject/MongoSpringBootApp.java

package com.example.mongoproject;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MongoSpringBootApp {
    public static void main(String[] args) {
        SpringApplication.run(MongoSpringBootApp.class, args);
    }
}


---

2. Staralgonew.java

Path: src/main/java/com/example/mongoproject/model/Staralgonew.java

package com.example.mongoproject.model;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "Staralgonew")
public class Staralgonew {

    @Id
    private String id;

    private String dealId;
    private String tradeDate;
    private String valueDate;
    private String maturityDate;
    private String legalEntity;
    private String crdsPartyNo;
    private String portfolioCode;
    private String managingLocation;
    private String secondaryIssue;
    private String notional;
    private String isin;
    private String crdsPartyCode;

    // Getters and setters

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getDealId() { return dealId; }
    public void setDealId(String dealId) { this.dealId = dealId; }

    public String getTradeDate() { return tradeDate; }
    public void setTradeDate(String tradeDate) { this.tradeDate = tradeDate; }

    public String getValueDate() { return valueDate; }
    public void setValueDate(String valueDate) { this.valueDate = valueDate; }

    public String getMaturityDate() { return maturityDate; }
    public void setMaturityDate(String maturityDate) { this.maturityDate = maturityDate; }

    public String getLegalEntity() { return legalEntity; }
    public void setLegalEntity(String legalEntity) { this.legalEntity = legalEntity; }

    public String getCrdsPartyNo() { return crdsPartyNo; }
    public void setCrdsPartyNo(String crdsPartyNo) { this.crdsPartyNo = crdsPartyNo; }

    public String getPortfolioCode() { return portfolioCode; }
    public void setPortfolioCode(String portfolioCode) { this.portfolioCode = portfolioCode; }

    public String getManagingLocation() { return managingLocation; }
    public void setManagingLocation(String managingLocation) { this.managingLocation = managingLocation; }

    public String getSecondaryIssue() { return secondaryIssue; }
    public void setSecondaryIssue(String secondaryIssue) { this.secondaryIssue = secondaryIssue; }

    public String getNotional() { return notional; }
    public void setNotional(String notional) { this.notional = notional; }

    public String getIsin() { return isin; }
    public void setIsin(String isin) { this.isin = isin; }

    public String getCrdsPartyCode() { return crdsPartyCode; }
    public void setCrdsPartyCode(String crdsPartyCode) { this.crdsPartyCode = crdsPartyCode; }
}


---

3. StaralgonewRepository.java

Path: src/main/java/com/example/mongoproject/repository/StaralgonewRepository.java

package com.example.mongoproject.repository;

import com.example.mongoproject.model.Staralgonew;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface StaralgonewRepository extends MongoRepository<Staralgonew, String> {
}


---

4. StaralgonewController.java

Path: src/main/java/com/example/mongoproject/controller/StaralgonewController.java

package com.example.mongoproject.controller;

import com.example.mongoproject.model.Staralgonew;
import com.example.mongoproject.repository.StaralgonewRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
public class StaralgonewController {

    @Autowired
    private StaralgonewRepository repository;

    @GetMapping("/staralgonew")
    public List<Staralgonew> getAll() {
        return repository.findAll();
    }
}


---

5. application.properties

Path: src/main/resources/application.properties

spring.data.mongodb.uri=mongodb://eurvli125734.xmp.ne.intra:20100/RECONDEV
spring.data.mongodb.database=RECONDEV
server.port=8080


---

Now you can:

1. Open a terminal in the project directory.


2. Run with: ./mvnw spring-boot:run or mvn spring-boot:run.



Go to:
http://localhost:8080/staralgonew
to see the MongoDB data from the Staralgonew collection.

Let me know if you'd like me to send you this as a .zip file!

