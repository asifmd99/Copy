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
