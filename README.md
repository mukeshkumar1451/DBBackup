package com.cognizant.collector.DatabaseCollector.Initializer;

import com.cognizant.collector.DatabaseCollector.service.DataTransferService;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;

@Component
public class ApplicationInitializer implements CommandLineRunner {

    private final DataTransferService dataTransferService;
    private final ApplicationContext applicationContext;

    public ApplicationInitializer(DataTransferService dataTransferService, ApplicationContext applicationContext) {
        this.dataTransferService = dataTransferService;
        this.applicationContext = applicationContext;
    }

    @Override
    public void run(String... args) throws Exception {
        System.out.println("Application started. Beginning data transfer...");

        // Perform the data transfer
        dataTransferService.transferAllCollections();

        System.out.println("Data transfer completed.");

        // Exit the application
        exitApplication();
    }

    private void exitApplication() {
        // Exit the application gracefully
        ((org.springframework.context.ConfigurableApplicationContext) applicationContext).close();
        System.exit(0);
    }
}




