niooimport java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.IOException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class FileWriterProgram {
    private static final String FILE_PATH = "output.txt";
    private static final int INTERVAL = 10000; // 10 seconds

    public static void main(String[] args) {
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");
        
        while (true) {
            try (BufferedWriter writer = new BufferedWriter(new FileWriter(FILE_PATH, true))) {
                String currentTime = dtf.format(LocalDateTime.now());
                writer.write(currentTime);
                writer.newLine();
                System.out.println("Written to file: " + currentTime);
            } catch (IOException e) {
                e.printStackTrace();
            }
            
            try {
                Thread.sleep(INTERVAL);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
_----_---
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.nio.file.*;

public class FilePollerProgram {
    private static final String FILE_PATH = "output.txt";

    public static void main(String[] args) {
        Path path = Paths.get(FILE_PATH);
        try (BufferedReader reader = new BufferedReader(new FileReader(FILE_PATH))) {
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println("Initial file content: " + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        try (WatchService watchService = FileSystems.getDefault().newWatchService()) {
            path.getParent().register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);

            while (true) {
                WatchKey key = watchService.take();
                for (WatchEvent<?> event : key.pollEvents()) {
                    if (event.kind() == StandardWatchEventKinds.ENTRY_MODIFY &&
                            ((Path) event.context()).endsWith(FILE_PATH)) {
                        try (BufferedReader reader = new BufferedReader(new FileReader(FILE_PATH))) {
                            String line;
                            while ((line = reader.readLine()) != null) {
                                System.out.println("New update: " + line);
                            }
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
                key.reset();
            }
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
_------using polling
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.attribute.FileTime;
import java.io.IOException;
import java.time.Instant;

@SpringBootApplication
@EnableScheduling
public class FilePollingApplication implements CommandLineRunner {

    private final Path filePath = Paths.get("path/to/your/file.txt");
    private FileTime lastModifiedTime = FileTime.from(Instant.EPOCH);

    public static void main(String[] args) {
        SpringApplication.run(FilePollingApplication.class, args);
    }

    @Override
    public void run(String... args) {
        System.out.println("Application started. Polling for file updates...");
    }

    @Scheduled(fixedRate = 5000)  // Poll every 5 seconds
    public void pollFileUpdates() {
        try {
            FileTime currentModifiedTime = Files.getLastModifiedTime(filePath);
            if (currentModifiedTime.compareTo(lastModifiedTime) > 0) {
                System.out.println("File has been updated.");
                lastModifiedTime = currentModifiedTime;
                // Add your file processing logic here
            } else {
                System.out.println("No update detected.");
            }
        } catch (IOException e) {
            System.err.println("Error checking file update: " + e.getMessage());
        }
    }
}
------
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.kafka.core.KafkaTemplate;

import java.io.IOException;
import java.nio.file.*;

@SpringBootApplication
public class FileWatcherApplication implements CommandLineRunner {

    private final Path filePath = Paths.get("path/to/your/file.txt");
    private static final String TOPIC = "your-kafka-topic";

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public static void main(String[] args) {
        SpringApplication.run(FileWatcherApplication.class, args);
    }

    @Override
    public void run(String... args) {
        System.out.println("Application started. Watching for file updates...");

        try (WatchService watchService = FileSystems.getDefault().newWatchService()) {
            Path parentDir = filePath.getParent();
            parentDir.register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);

            while (true) {
                WatchKey key = watchService.take();

                for (WatchEvent<?> event : key.pollEvents()) {
                    WatchEvent.Kind<?> kind = event.kind();

                    if (kind == StandardWatchEventKinds.OVERFLOW) {
                        continue;
                    }

                    WatchEvent<Path> ev = (WatchEvent<Path>) event;
                    Path changed = ev.context();

                    if (changed.equals(filePath.getFileName())) {
                        System.out.println("File has been updated.");
                        // Add your file processing logic here
                        processFile(filePath);
                    }
                }

                boolean valid = key.reset();
                if (!valid) {
                    break;
                }
            }

        } catch (IOException | InterruptedException e) {
            System.err.println("Error watching file: " + e.getMessage());
        }
    }

    private void processFile(Path path) {
        try {
            // Read the file content or any other processing logic
            String fileContent = new String(Files.readAllBytes(path));
            
            // Send the file content to Kafka
            kafkaTemplate.send(TOPIC, fileContent);

            System.out.println("File content sent to Kafka: " + fileContent);
        } catch (IOException e) {
            System.err.println("Error processing file: " + e.getMessage());
        }
    }
}

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.kafka.core.KafkaTemplate;

import java.io.IOException;
import java.nio.file.*;
import java.util.Date;

@SpringBootApplication
public class FileWatcherApplication implements CommandLineRunner {

    private final Path filePath = Paths.get("path/to/your/file.txt");
    private static final String TOPIC = "your-kafka-topic";

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private FileUpdateRepository fileUpdateRepository;

    public static void main(String[] args) {
        SpringApplication.run(FileWatcherApplication.class, args);
    }

    @Override
    public void run(String... args) {
        System.out.println("Application started. Watching for file updates...");

        try (WatchService watchService = FileSystems.getDefault().newWatchService()) {
            Path parentDir = filePath.getParent();
            parentDir.register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);

            while (true) {
                WatchKey key = watchService.take();

                for (WatchEvent<?> event : key.pollEvents()) {
                    WatchEvent.Kind<?> kind = event.kind();

                    if (kind == StandardWatchEventKinds.OVERFLOW) {
                        continue;
                    }

                    WatchEvent<Path> ev = (WatchEvent<Path>) event;
                    Path changed = ev.context();

                    if (changed.equals(filePath.getFileName())) {
                        System.out.println("File has been updated.");
                        processFile(filePath);
                    }
                }

                boolean valid = key.reset();
                if (!valid) {
                    break;
                }
            }

        } catch (IOException | InterruptedException e) {
            System.err.println("Error watching file: " + e.getMessage());
        }
    }

    private void processFile(Path path) {
        try {
            // Read the file content
            String fileContent = new String(Files.readAllBytes(path));
            long timestamp = new Date().getTime();

            // Save the file update to MongoDB
            FileUpdate fileUpdate = new FileUpdate(fileContent, timestamp);
            fileUpdateRepository.save(fileUpdate);

            // Send the file content to Kafka
            kafkaTemplate.send(TOPIC, fileContent);

            System.out.println("File content sent to Kafka and stored in MongoDB: " + fileContent);
        } catch (IOException e) {
            System.err.println("Error processing file: " + e.getMessage());
        }
    }
}
import java.io.IOException;
import java.nio.file.*;
import java.util.List;

public class LogFileWatcher {

    private static final String LOG_FILE_PATH = "/path/to/your/logfile.log"; // Update this path to your log file

    public static void main(String[] args) {
        Path logFilePath = Paths.get(LOG_FILE_PATH);
        Path logDir = logFilePath.getParent();
        String logFileName = logFilePath.getFileName().toString();

        try (WatchService watchService = FileSystems.getDefault().newWatchService()) {
            logDir.register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);

            System.out.println("Watching log file for changes: " + LOG_FILE_PATH);

            while (true) {
                WatchKey key = watchService.take();

                for (WatchEvent<?> event : key.pollEvents()) {
                    WatchEvent.Kind<?> kind = event.kind();

                    if (kind == StandardWatchEventKinds.ENTRY_MODIFY) {
                        Path changed = (Path) event.context();
                        if (changed.endsWith(logFileName)) {
                            List<String> lines = Files.readAllLines(logFilePath);
                            if (!lines.isEmpty()) {
                                String lastLine = lines.get(lines.size() - 1);
                                System.out.println("New log entry: " + lastLine);
                            }
                        }
                    }
                }
                boolean valid = key.reset();
                if (!valid) {
                    break;
                }
            }
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
import java.io.FileWriter;
import java.io.IOException;
import java.security.SecureRandom;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class NonBlockingFileWriter {

    private static final String FILE_PATH = "output.txt";
    private static final SecureRandom RANDOM = new SecureRandom();
    private static final int STRING_LENGTH = 10;

    public static void main(String[] args) {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        Runnable task = () -> {
            CompletableFuture.runAsync(() -> {
                String randomString = generateRandomString(STRING_LENGTH);
                appendToFile(randomString);
            });
        };

        scheduler.scheduleAtFixedRate(task, 0, 10, TimeUnit.SECONDS);
    }

    private static String generateRandomString(int length) {
        StringBuilder sb = new StringBuilder(length);
        for (int i = 0; i < length; i++) {
            sb.append((char) ('a' + RANDOM.nextInt(26)));
        }
        return sb.toString();
    }

    private static void appendToFile(String text) {
        try (FileWriter writer = new FileWriter(FILE_PATH, true)) {
            writer.write(text + System.lineSeparator());
            System.out.println("Appended: " + text);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
import java.io.IOException;
import java.nio.file.*;
import java.security.SecureRandom;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class NonBlockingFileWriterNIO {

    private static final String FILE_PATH = "output.txt";
    private static final SecureRandom RANDOM = new SecureRandom();
    private static final int STRING_LENGTH = 10;

    public static void main(String[] args) {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        Runnable task = () -> {
            CompletableFuture.runAsync(() -> {
                String randomString = generateRandomString(STRING_LENGTH);
                appendToFile(randomString);
            });
        };

        scheduler.scheduleAtFixedRate(task, 0, 10, TimeUnit.SECONDS);
    }

    private static String generateRandomString(int length) {
        StringBuilder sb = new StringBuilder(length);
        for (int i = 0; i < length; i++) {
            sb.append((char) ('a' + RANDOM.nextInt(26)));
        }
        return sb.toString();
    }

    private static void appendToFile(String text) {
        Path path = Paths.get(FILE_PATH);
        try {
            Files.write(path, (text + System.lineSeparator()).getBytes(), StandardOpenOption.CREATE, StandardOpenOption.APPEND);
            System.out.println("Appended: " + text);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
                                       Demo - Phase 1
—------------------------------------------------------------------------------------------------------------------
what happens when we run a springboot application?

– searches main method - runs application - @SpringBootApplication has @Configuration@EnableAutoConfiguration, and @ComponentScan

The SpringApplication class sets up the default configuration, creates the Spring application context, and triggers the auto-configuration and component scanning.

SpringBoot creates an appropriate ApplicationContext instance (AnnotationConfigServletWebServerApplicationContext for web applications).
It registers all @Configuration classes and performs a component scan to identify Spring-managed components (@Component, @Service, @Repository, @Controller, etc.).

Prepares the environment like Properties, configuration
Any ApplicationListener and ApplicationContextInitializer beans are detected and invoked. These are used to customize the application context before the beans are loaded.
Spring Boot’s @EnableAutoConfiguration triggers auto-configuration classes, which automatically configure various components (like data sources, JPA, security, etc.) based on the available classpath dependencies and properties.
Bean Initialization:
The Spring container starts to create and initialize beans as defined by the configuration and component classes. This involves dependency injection, managing bean life cycles, and invoking any bean post-processors.Dependency Injection: Spring Boot manages dependencies using the IoC (Inversion of Control) container, injecting dependencies where needed, typically through constructor injection, field injection, or setter injection.
If it's a web application, an embedded web server (like Tomcat, Jetty, or Undertow) is started. The server listens for incoming HTTP requests and routes them to appropriate handlers (controllers).
Spring Boot detects any beans implementing ApplicationRunner or CommandLineRunner and executes their run methods. This is typically used for executing code after the application context has been initialized.
—-------------------------------------------------------------
Why watchService is best for monitoring file updates ?

The WatchService API in Java NIO (New I/O) is particularly well-suited for monitoring file updates and changes in a directory.

Efficiency:
Low CPU Usage: WatchService is event-driven and does not require constant polling, leading to lower CPU usage compared to traditional polling methods.
Resource Utilization: Since it relies on the operating system's native event notification mechanisms, it efficiently uses system resources.
Resource Efficiency:OS LEVEL NOTIFICATION
Unlike polling, which continuously checks the state of files and directories, WatchService waits for the operating system to notify it of changes. This results in lower CPU usage and more efficient resource management.
No Configuration required unlike library
Concurrency:
Non-Blocking Operations: The asynchronous nature allows other parts of the application to continue running without waiting for file events, improving the overall responsiveness and throughput.

The program enters a loop where it waits for events. The watchService.take() method blocks until an event is available.


Cons-Complexity:
Platform-Specific Limitations:
Inconsistent Support: While WatchService works on multiple platforms, the underlying file notification mechanisms may have limitations or bugs specific to certain operating systems or configurations.
Different Behaviors: The behavior of file system notifications can vary between operating systems, which might lead to inconsistent application behavior across platforms.
Scalability:Monitoring very large directories with many files can become resource-intensive and may result in a high volume of events, which can be challenging to manage efficiently.

—-------------------------------------------------------------------
Blocking Vs Non Blocking
Blocking and non-blocking operations are fundamental concepts in concurrent programming, particularly in the context of threads.

Blocking operations cause the calling thread to wait until the operation completes. During this waiting period, the thread is essentially idle and cannot perform any other tasks.
Thread Waits:
When a thread performs a blocking operation (e.g., reading from a file, waiting for network data, acquiring a lock), it is put into a waiting state until the operation finishes.
The thread is not available to execute other tasks during this time.
While waiting, the thread consumes system resources, particularly memory. This can be inefficient if many threads are blocked simultaneously.
In high-concurrency environments (like web servers), too many blocked threads can exhaust system resources (e.g., thread pool limits).

Non-blocking operations allow a thread to initiate an operation and immediately proceed with other tasks. The completion of the operation is typically handled through callbacks, futures, or polling mechanisms.
—------------------------------------------------------------

Concurrency and Scalability:
Non-blocking operations are essential for building highly concurrent and scalable applications. They allow a small number of threads to handle many tasks efficiently.
what will happen when both write and read operations are going on a file using 2 programs if we use blocking vs if we use nonBlocking?
Write Operation (Blocking):
Program 1 writes to the file using blocking I/O (e.g., BufferedWriter.write()). The program will block (wait) until the entire write operation is completed.
During this time, the file is often locked by the operating system to prevent concurrent modifications, depending on the file system and OS settings.
If the file is locked for writing by Program 1, Program 2 may be blocked from reading until the write operation is complete. This depends on the file system's locking policy.
Data Consistency: There might be inconsistencies or partial reads if Program 2 starts reading while Program 1 is in the middle of writing, especially if the write operation is not atomic.
Performance Impact: Both programs can be delayed due to blocking. Program 2 might wait for Program 1 to finish writing, and Program 1 might be blocked again if it tries to write while Program 2 is reading.
Non Blocking
Concurrency: Both programs can perform their tasks concurrently without waiting for the other to finish. This can significantly improve performance and responsiveness.

Atomicity and Locking:
Blocking I/O: File systems often lock files during blocking write operations to ensure atomicity, but this can lead to contention and delays.
Non-Blocking I/O: Locks are not typically used, allowing higher concurrency but requiring the application to manage potential data races and inconsistencies.
—------------------------------------------------------------------
How kafka starts?
—---------------------------------------------------------------
Use Cases:
Real-time Analytics: Stream consumers are used in applications where immediate insights or actions are required based on incoming data (e.g., fraud detection, real-time monitoring).
Event-driven Architectures: Stream processing is integral to event-driven architectures where events trigger responses or actions in real-time.
IoT Data Processing: Stream processing is common in IoT applications to handle high volumes of sensor data and make timely decisions based on sensor readings.
—---------------------------------------
Why kafka and not db
Concurrency and Scalability:
Locking: Traditional databases may face issues with locking and concurrency when handling high volumes of real-time data updates and queries simultaneously.
Scaling Challenges: Scaling traditional databases horizontally can be complex and expensive compared to horizontally scalable messaging systems like Kafka.
Latency and Performance:
Query Overhead: Databases may introduce overhead in query processing and transaction management, affecting real-time performance requirements.
Indexing: Real-time queries may require extensive indexing and optimization to achieve low-latency responses, which can be challenging to maintain at scale.

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.util.Collections;
import java.util.concurrent.ExecutionException;

@Service
public class KafkaTopicService {

    @Autowired
    private KafkaAdmin kafkaAdmin;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private KafkaConsumerService kafkaConsumerService;

    @Value(value = "${topic.name}")
    private String topicName;

    @PostConstruct
    public void resetTopic() throws ExecutionException, InterruptedException {
        try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfig())) {
            // Delete the topic
            DeleteTopicsResult deleteTopicsResult = adminClient.deleteTopics(Collections.singletonList(topicName));
            deleteTopicsResult.all().get();

            // Recreate the topic
            adminClient.createTopics(Collections.singleton(new NewTopic(topicName, 1, (short) 1))).all().get();

            // Start Kafka consumer
            kafkaConsumerService.startConsuming();
        }
    }
}
