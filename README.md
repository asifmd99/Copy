import java.io.BufferedWriter;
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
