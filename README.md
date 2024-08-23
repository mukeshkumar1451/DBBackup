package com.cognizant.collector.DatabaseCollector.service;

import com.cognizant.collector.DatabaseCollector.beans.RTS.Defect;
import com.cognizant.collector.DatabaseCollector.beans.RTS.TestCase;
import com.cognizant.collector.DatabaseCollector.beans.RTS.Userstories;
import com.cognizant.collector.DatabaseCollector.Scheduler.SchedulerInfo;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Service;

import java.lang.reflect.Field;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class DataTransferService {

    @Autowired
    @Qualifier("sourceMongoTemplate")
    private MongoTemplate sourceMongoTemplate;

    @Autowired
    @Qualifier("targetMongoTemplate")
    private MongoTemplate targetMongoTemplate;

    private static final int BATCH_SIZE = 2000;
    private static final DateTimeFormatter TARGET_DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
    private static final DateTimeFormatter SOURCE_DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'");
    private static final DateTimeFormatter SCHEDULER_DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'");

    public void transferAllCollections(String jiraProjectKey) {
        if (jiraProjectKey == null || jiraProjectKey.trim().isEmpty()) {
            throw new IllegalArgumentException("jiraProjectKey must not be null or empty");
        }

        ensureDatabaseAndCollections();

        transferCollection("Userstories", Userstories.class, jiraProjectKey);
        transferCollection("Defect", Defect.class, jiraProjectKey);
        transferCollection("testcase", TestCase.class, jiraProjectKey);

        // Update SchedulerInfo separately
        updateSchedulerInfo(jiraProjectKey);

        // Optionally exit after data transfer
        System.exit(0);
    }

    private void ensureDatabaseAndCollections() {
        Set<String> collectionNamesSet = new HashSet<>(targetMongoTemplate.getCollectionNames());
        List<String> collectionNames = new ArrayList<>(collectionNamesSet);
        List<String> requiredCollections = Arrays.asList("Userstories", "Defect", "testcase", "SchedulerInfo");

        for (String collectionName : requiredCollections) {
            if (!collectionNames.contains(collectionName)) {
                targetMongoTemplate.createCollection(collectionName);
                System.out.println("Created collection: " + collectionName);
            }
        }
    }

    private <T> void transferCollection(String collectionName, Class<T> pojoClass, String jiraProjectKey) {
        int page = 0;
        List<T> batch;

        System.out.println("Fetching data from collection " + collectionName + "...");

        do {
            System.out.println("Fetching page " + page + " from collection " + collectionName + "...");
            batch = fetchDataBatch(collectionName, pojoClass, page, BATCH_SIZE, jiraProjectKey);

            if (batch != null && !batch.isEmpty()) {
                printData(batch);
                System.out.println("Inserting data into collection " + collectionName + "...");
                insertData(collectionName, pojoClass, batch, jiraProjectKey);
                System.out.println("Page " + page + " processed successfully.");
                page++;
            } else {
                System.out.println("No more data to fetch for collection " + collectionName + ".");
            }
        } while (batch != null && !batch.isEmpty());
    }

    private <T> List<T> fetchDataBatch(String collectionName, Class<T> pojoClass, int page, int batchSize, String jiraProjectKey) {
        Query query = new Query()
                .with(PageRequest.of(page, batchSize))
                .addCriteria(Criteria.where("jiraProjectKey").is(jiraProjectKey));
        return sourceMongoTemplate.find(query, pojoClass, collectionName);
    }

    private <T> void printData(List<T> data) {
        if (data != null && !data.isEmpty()) {
            System.out.println("Printing " + data.size() + " documents:");
            data.forEach(item -> System.out.println(item.toString()));
        } else {
            System.out.println("No documents to print.");
        }
    }

    private <T> void insertData(String collectionName, Class<T> pojoClass, List<T> data, String jiraProjectKey) {
        if (data != null && !data.isEmpty()) {
            for (T item : data) {
                try {
                    // Modify the item to ensure `lastUpdated` is in the correct format
                    String lastUpdatedField = "lastUpdated";
                    Object lastUpdatedValue = getFieldValue(item, lastUpdatedField);

                    // Convert the lastUpdated value to target format
                    if (lastUpdatedValue instanceof String) {
                        String lastUpdatedString = (String) lastUpdatedValue;
                        LocalDateTime localDateTime = LocalDateTime.parse(lastUpdatedString, SOURCE_DATE_FORMATTER);
                        String formattedLastUpdated = localDateTime.format(TARGET_DATE_FORMATTER);
                        setFieldValue(item, lastUpdatedField, formattedLastUpdated);
                    }

                    // Insert new document into target collection
                    targetMongoTemplate.insert(item, collectionName);
                    System.out.println("Document inserted into collection " + collectionName);

                } catch (Exception e) {
                    System.err.println("Error processing document: " + e.getMessage());
                }
            }

            System.out.println("Successfully processed " + data.size() + " documents in collection " + collectionName);
        }
    }

    private Object getFieldValue(Object item, String fieldName) throws NoSuchFieldException, IllegalAccessException {
        Field field = item.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(item);
    }

    private void setFieldValue(Object item, String fieldName, Object value) throws NoSuchFieldException, IllegalAccessException {
        Field field = item.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(item, value);
    }

    private void updateSchedulerInfo(String jiraProjectKey) {
        // Create a new SchedulerInfo document for the provided jiraProjectKey
        SchedulerInfo schedulerInfo = new SchedulerInfo();
        schedulerInfo.setJiraProjectKey(jiraProjectKey);

        // Convert current time to LocalDateTime and then format
        LocalDateTime now = LocalDateTime.now(ZoneId.of("UTC"));
        String formattedLastRun = now.format(SCHEDULER_DATE_FORMATTER);

        schedulerInfo.setLastRun(formattedLastRun);
        schedulerInfo.setStatus("Completed");

        // Insert the new document into the SchedulerInfo collection
        targetMongoTemplate.insert(schedulerInfo, "SchedulerInfo");
        System.out.println("SchedulerInfo document created for jiraProjectKey " + jiraProjectKey);
    }
}

