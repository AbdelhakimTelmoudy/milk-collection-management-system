# Milk Collection Management System

This is a Java Spring Boot application for managing milk collection data.

## Dependencies

- Java 17
- Spring Boot 3.3.0
- Apache POI 5.1.0 (for handling Excel files)
- OpenCSV 5.5.1 (for handling CSV files)


###  Export File CSV
```
@GetMapping("/exportMilkCollectionsCSV")
    public ResponseEntity<ByteArrayResource> exportMilkCollectionsCSV() {
        try {
            List<MilkCollection> milkCollectionList = milkCollectionService.getAllMilkCollections();
            StringBuilder csvData = new StringBuilder();
            csvData.append("Date,Quantity,Member ID\n");
            for (MilkCollection milkCollection : milkCollectionList) {
                csvData.append(String.format("%s,%s,%d\n",
                        milkCollection.getDate().toString(),
                        String.format(Locale.US, "%.2f", milkCollection.getQuantity()).replace(',', '.'),
                        milkCollection.getMember().getId()));
            }
            byte[] csvBytes = csvData.toString().getBytes(StandardCharsets.UTF_8);
            ByteArrayResource resource = new ByteArrayResource(csvBytes);
            DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MM-yyyy");
            LocalDate localDate = LocalDate.now();
            String formattedString = localDate.format(formatter);
            String fileName = "exportedMilkCollection-"+formattedString+".csv";
            return ResponseEntity.ok()
                    .contentType(MediaType.parseMediaType("text/csv"))
                    .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + fileName + "\"")
                    .body(resource);
        } catch (Exception e) {
            e.printStackTrace();
            return ResponseEntity.status(500).body(null);
        }
    }
```
###  Import File CSV
```
@PostMapping("/uploadMilkCollectionsCSV")
public ResponseEntity<String> uploadMilkCollectionsCSV(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("Please upload a CSV file.");
    }
    try (Reader reader = new InputStreamReader(file.getInputStream())) {
        CSVReader csvReader = new CSVReaderBuilder(reader).withSkipLines(1).build();
        List<MilkCollection> milkCollections = new ArrayList<>();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        String[] nextLine;
        while ((nextLine = csvReader.readNext()) != null) {
            try {
                String dateString = nextLine[0];
                String quantityString = nextLine[1];
                String memberIdString = nextLine[2];
                LocalDate date = LocalDate.parse(dateString, formatter);
                double quantity = Double.parseDouble(quantityString);
                Long memberId = Long.parseLong(memberIdString);
                Optional<Member> memberOpt = memberRepository.findById(memberId);
                if (memberOpt.isEmpty()) {
                    System.err.println("Error processing row: Member not found: " + memberId);
                    continue;
                }
                MilkCollection milkCollection = new MilkCollection();
                milkCollection.setDate(date);
                milkCollection.setQuantity(quantity);
                milkCollection.setMember(memberOpt.get());

                milkCollections.add(milkCollection);
            } catch (Exception e) {
                System.err.println("Error processing row: " + e.getMessage());
            }
        }
        milkCollectionRepository.saveAll(milkCollections);
        return ResponseEntity.ok("CSV file uploaded and data saved to database successfully.");
    } catch (Exception e) {
        e.printStackTrace();
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Failed to upload CSV file: " + e.getMessage());
    }
}
```
###  Import File Excel

```
@PostMapping("/uploadMilkCollectionsExcel")
public ResponseEntity<String> uploadMilkCollectionsExcel(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("Please upload an Excel file.");
    }
    try (Workbook workbook = WorkbookFactory.create(file.getInputStream())) {
        Sheet sheet = workbook.getSheetAt(0);
        Iterator<Row> rowIterator = sheet.iterator();
        if (rowIterator.hasNext()) {
            rowIterator.next();
        }
        List<MilkCollection> milkCollections = new ArrayList<>();
        while (rowIterator.hasNext()) {
            Row row = rowIterator.next();
            try {
                Cell dateCell = row.getCell(2);
                LocalDate date = null;

                if (dateCell != null) {
                    if (dateCell.getCellType() == CellType.NUMERIC) {
                        date = dateCell.getDateCellValue().toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
                    } else if (dateCell.getCellType() == CellType.STRING) {
                        String dateString = dateCell.getStringCellValue();
                        date = LocalDate.parse(dateString);
                    }
                }
                double quantity = row.getCell(1).getNumericCellValue();
                String memberName = row.getCell(0).getStringCellValue();
                MilkCollection milkCollection = new MilkCollection();
                milkCollection.setDate(date);
                milkCollection.setQuantity(quantity);
                milkCollection.setMember(memberRepository.findAllByNomIgnoreCase(memberName));
                milkCollections.add(milkCollection);
            } catch (Exception e) {
                System.err.println("Error processing row: " + e.getMessage());
            }
        }
        milkCollectionRepository.saveAll(milkCollections);
        return ResponseEntity.ok("Excel file uploaded and data saved to database successfully.");
    } catch (IOException e) {
        e.printStackTrace();
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Failed to upload Excel file: " + e.getMessage());
    }
}
```
###  Export File Excel

```
@GetMapping("/exportMilkCollectionsExcel")
public void exportMilkCollectionsExcel(HttpServletResponse response) throws IOException {
    response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    response.setHeader("Content-Disposition", "attachment; filename=milk_collections.xlsx");
    List<MilkCollection> milkCollections = milkCollectionRepository.findAll();
    Workbook workbook = new XSSFWorkbook();
    Sheet sheet = workbook.createSheet("Milk Collections");
    Row headerRow = sheet.createRow(0);
    headerRow.createCell(0).setCellValue("Member Name");
    headerRow.createCell(1).setCellValue("Quantity");
    headerRow.createCell(2).setCellValue("Date");
    int rowNum = 1;
    for (MilkCollection milkCollection : milkCollections) {
        Row row = sheet.createRow(rowNum++);
        row.createCell(2).setCellValue(milkCollection.getDate().toString());
        row.createCell(1).setCellValue(milkCollection.getQuantity());
        row.createCell(0).setCellValue(milkCollection.getMember().getNom());
    }
    workbook.write(response.getOutputStream());
    workbook.close();
}
```