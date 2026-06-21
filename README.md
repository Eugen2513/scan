# main.java
import com.jcraft.jsch.*;
import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.io.InputStream;
import java.net.InetSocketAddress;
import java.net.Socket;

public class Main extends JFrame {

    private JTextField ipField;
    private JTextField startPortField;
    private JTextField endPortField;

    private DefaultTableModel portModel;
    private JTextArea terminalArea;

    private JTextField sshHostField;
    private JTextField sshUserField;
    private JPasswordField sshPassField;
    private JTextField commandField;

    public Main() {

        setTitle("Network Scanner & SSH Client");
        setSize(1000, 700);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        JTabbedPane tabs = new JTabbedPane();

        tabs.add("Scanner", createScannerPanel());
        tabs.add("SSH", createSSHPanel());

        add(tabs);

        setVisible(true);
    }

    private JPanel createScannerPanel() {

        JPanel panel = new JPanel(new BorderLayout());

        JPanel top = new JPanel(new GridLayout(2, 4, 5, 5));

        ipField = new JTextField("127.0.0.1");
        startPortField = new JTextField("1");
        endPortField = new JTextField("1024");

        JButton scanButton = new JButton("Scan");

        top.add(new JLabel("IP Address"));
        top.add(ipField);

        top.add(new JLabel("Start Port"));
        top.add(startPortField);

        top.add(new JLabel("End Port"));
        top.add(endPortField);

        top.add(new JLabel(""));
        top.add(scanButton);

        panel.add(top, BorderLayout.NORTH);

        portModel = new DefaultTableModel(
                new Object[]{"Port", "Status"}, 0);

        JTable table = new JTable(portModel);

        panel.add(new JScrollPane(table), BorderLayout.CENTER);

        scanButton.addActionListener(e -> startScan());

        return panel;
    }

    private JPanel createSSHPanel() {

        JPanel panel = new JPanel(new BorderLayout());

        JPanel loginPanel = new JPanel(new GridLayout(5, 2, 5, 5));

        sshHostField = new JTextField("127.0.0.1");
        sshUserField = new JTextField();
        sshPassField = new JPasswordField();
        commandField = new JTextField("whoami");

        JButton runButton = new JButton("Run Command");

        loginPanel.add(new JLabel("Host"));
        loginPanel.add(sshHostField);

        loginPanel.add(new JLabel("Username"));
        loginPanel.add(sshUserField);

        loginPanel.add(new JLabel("Password"));
        loginPanel.add(sshPassField);

        loginPanel.add(new JLabel("Command"));
        loginPanel.add(commandField);

        loginPanel.add(new JLabel(""));
        loginPanel.add(runButton);

        panel.add(loginPanel, BorderLayout.NORTH);

        terminalArea = new JTextArea();
        terminalArea.setEditable(false);

        panel.add(new JScrollPane(terminalArea), BorderLayout.CENTER);

        runButton.addActionListener(e -> runSSHCommand());

        return panel;
    }

    private void startScan() {

        portModel.setRowCount(0);

        String ip = ipField.getText();

        int startPort;
        int endPort;

        try {
            startPort = Integer.parseInt(startPortField.getText());
            endPort = Integer.parseInt(endPortField.getText());
        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this,
                    "Invalid port range");
            return;
        }

        new Thread(() -> {

            for (int port = startPort; port <= endPort; port++) {

                try (Socket socket = new Socket()) {

                    socket.connect(
                            new InetSocketAddress(ip, port),
                            200
                    );

                    final int p = port;

                    SwingUtilities.invokeLater(() ->
                            portModel.addRow(
                                    new Object[]{p, "OPEN"}));

                } catch (Exception ignored) {
                }
            }

        }).start();
    }

    private void runSSHCommand() {

        new Thread(() -> {

            try {

                String host = sshHostField.getText();
                String user = sshUserField.getText();
                String pass =
                        new String(sshPassField.getPassword());

                String command = commandField.getText();

                JSch jsch = new JSch();

                Session session =
                        jsch.getSession(user, host, 22);

                session.setPassword(pass);

                session.setConfig(
                        "StrictHostKeyChecking",
                        "no");

                session.connect(5000);

                ChannelExec channel =
                        (ChannelExec)
                                session.openChannel("exec");

                channel.setCommand(command);

                InputStream in =
                        channel.getInputStream();

                channel.connect();

                StringBuilder output =
                        new StringBuilder();

                byte[] buffer = new byte[1024];

                while (true) {

                    while (in.available() > 0) {

                        int i =
                                in.read(buffer,
                                        0,
                                        buffer.length);

                        if (i < 0)
                            break;

                        output.append(
                                new String(buffer,
                                        0,
                                        i));
                    }

                    if (channel.isClosed())
                        break;

                    Thread.sleep(100);
                }

                channel.disconnect();
                session.disconnect();

                SwingUtilities.invokeLater(() ->
                        terminalArea.setText(
                                output.toString()));

            } catch (Exception ex) {

                SwingUtilities.invokeLater(() ->
                        terminalArea.setText(
                                "Error:\n" +
                                        ex.getMessage()));
            }

        }).start();
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(Main::new);
    }
}

# pom.xml
<dependency>
    <groupId>com.github.mwiede</groupId>
    <artifactId>jsch</artifactId>
    <version>0.2.21</version>
</dependency>
