import java.io.*;
import java.net.*;
import java.text.SimpleDateFormat;
import java.util.*;

public class ChatApp {
    private static final int PORT = 12345;

    public static void main(String[] args) throws IOException {
        Scanner sc = new Scanner(System.in);
        System.out.println("START AS (1) SERVER OR (2) CLIENT?");
        int choice = sc.nextInt();
        sc.nextLine();

        if (choice == 1) {
            new Thread(() -> {
                try {
                    startServer();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }).start();
        } else if (choice == 2) {
            startClient();
        } else {
            System.out.println("INVALID CHOICE.");
        }
    }

    // ===== SERVER CODE =====
    private static Set<ClientHandler> clientsList = Collections.synchronizedSet(new HashSet<>());

    public static void startServer() throws IOException {
        ServerSocket serverSocket = new ServerSocket(PORT);
        System.out.println("SERVER STARTED ON PORT " + PORT);

        while (true) {
            Socket socket = serverSocket.accept();
            ClientHandler handler = new ClientHandler(socket);
            clientsList.add(handler);
            new Thread(handler).start();
        }
    }

    static void broadcast(String message, ClientHandler exclude) {
        synchronized (clientsList) {
            for (ClientHandler client : clientsList) {
                if (client != exclude) {
                    client.sendMessage(message);
                }
            }
        }
    }

    static void sendPrivateMessage(String targetUser, String message, ClientHandler sender) {
        boolean found = false;
        synchronized (clientsList) {
            for (ClientHandler client : clientsList) {
                if (client.name.equalsIgnoreCase(targetUser)) {
                    client.sendMessage("[PRIVATE] " + sender.name + ": " + message);
                    found = true;
                    break;
                }
            }
        }
        if (!found) {
            sender.sendMessage("User '" + targetUser + "' not found.");
        }
    }

    static void removeClient(ClientHandler ch) {
        clientsList.remove(ch);
    }

    static class ClientHandler implements Runnable {
        private Socket socket;
        private BufferedReader br;
        private BufferedWriter bw;
        String name;

        public ClientHandler(Socket socket) {
            this.socket = socket;
            try {
                br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            } catch (IOException e) {
                closeAll();
            }
        }

        @Override
        public void run() {
            try {
                bw.write("Enter your name: ");
                bw.flush();
                name = br.readLine().trim();

                System.out.println((name + " CONNECTED.").toUpperCase());
                broadcast(name + " joined the chat.", this);

                bw.write("Welcome " + name + "! Type /users to see who's online. Type /msg username message to send private message.\n");
                bw.flush();

                String msg;
                while ((msg = br.readLine()) != null) {
                    if (msg.equalsIgnoreCase("exit")) break;

                    if (msg.equalsIgnoreCase("/users")) {
                        sendUserList();
                    } else if (msg.startsWith("/msg ")) {
                        String[] parts = msg.split(" ", 3);
                        if (parts.length >= 3) {
                            String target = parts[1];
                            String privateMsg = parts[2];
                            sendPrivateMessage(target, privateMsg, this);
                        } else {
                            sendMessage("Invalid format. Use /msg username message");
                        }
                    } else {
                        String fullMsg = getTimestamp() + " " + name + ": " + msg;
                        System.out.println(fullMsg.toUpperCase());
                        broadcast(fullMsg, this);
                    }
                }
            } catch (IOException e) {
                System.out.println((name + " DISCONNECTED.").toUpperCase());
            } finally {
                closeAll();
                removeClient(this);
                broadcast(name + " left the chat.", this);
            }
        }

        public void sendMessage(String msg) {
            try {
                bw.write(msg);
                bw.newLine();
                bw.flush();
            } catch (IOException e) {
                closeAll();
            }
        }

        public void sendUserList() {
            StringBuilder sb = new StringBuilder("Online users:\n");
            synchronized (clientsList) {
                for (ClientHandler client : clientsList) {
                    sb.append("• ").append(client.name).append("\n");
                }
            }
            sendMessage(sb.toString());
        }

        private String getTimestamp() {
            return "[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "]";
        }

        private void closeAll() {
            try {
                if (br != null) br.close();
                if (bw != null) bw.close();
                if (socket != null) socket.close();
            } catch (IOException e) {
                // ignore silently
            }
        }
    }

    // ===== CLIENT CODE =====
    public static void startClient() {
        try (Scanner scanner = new Scanner(System.in)) {
            Socket socket = new Socket("localhost", PORT);
            BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));

            // Listener thread for incoming messages
            new Thread(() -> {
                try {
                    String msg;
                    while ((msg = br.readLine()) != null) {
                        System.out.println(msg);
                    }
                } catch (IOException e) {
                    // silent close
                }
            }).start();

            // Send user messages to server
            while (true) {
                String input = scanner.nextLine();
                bw.write(input);
                bw.newLine();
                bw.flush();

                if (input.equalsIgnoreCase("exit")) {
                    socket.close();
                    break;
                }
            }

        } catch (IOException e) {
            System.out.println("CLIENT ERROR: " + e.getMessage().toUpperCase());
        }
    }
}
