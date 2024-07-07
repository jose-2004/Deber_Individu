## Codigo y Ejecucion

  

    package InterfazGrafica;

    import javax.swing.*;
    import javax.swing.table.DefaultTableCellRenderer;
    import javax.swing.table.DefaultTableModel;
    import javax.swing.table.TableRowSorter;
    import java.awt.*;
    import java.io.FileWriter;
    import java.io.IOException;
    import java.sql.*;
    import java.util.Vector;

    public class InterfazGrafica {
    private Connection conn;
    private JFrame frame;
    private JComboBox<String> comboBox;
    private JTable table;
    private DefaultTableModel tableModel;
    private TableRowSorter<DefaultTableModel> sorter;
    private JTextField filterField;

    public InterfazGrafica() {
        // Establecer conexión a la base de datos PostgreSQL
        connectDB();

        // Crear la interfaz gráfica
        createGUI();
    }

    private void connectDB() {
        try {
            String url = "jdbc:postgresql://localhost:5432/formula1";
            String user = "postgres";
            String password = "miguel";
            conn = DriverManager.getConnection(url, user, password);
            System.out.println("Conexión establecida con PostgreSQL.");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void createGUI() {
        frame = new JFrame("Tabla de Drivers por Año de Carrera");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(950, 500);

        // Panel superior para controles
        JPanel topPanel = new JPanel();
        topPanel.setLayout(new FlowLayout());

        // Etiqueta para el año
        JLabel yearLabel = new JLabel("Año:");
        topPanel.add(yearLabel);

        // Combo box para seleccionar el año de carrera
        comboBox = new JComboBox<>();
        populateComboBox();
        comboBox.addActionListener(e -> updateTable());
        topPanel.add(comboBox);

        // Botón para exportar los datos a un archivo CSV
        JButton exportButton = new JButton("Exportar a CSV");
        exportButton.addActionListener(e -> exportToCSV());
        topPanel.add(exportButton);

        // Campo de texto para filtrar la tabla
        filterField = new JTextField(15);
        filterField.setToolTipText("Filtrar por nombre o apellido");
        filterField.getDocument().addDocumentListener(new javax.swing.event.DocumentListener() {
            public void insertUpdate(javax.swing.event.DocumentEvent e) {
                filterTable();
            }

            public void removeUpdate(javax.swing.event.DocumentEvent e) {
                filterTable();
            }

            public void changedUpdate(javax.swing.event.DocumentEvent e) {
                filterTable();
            }
        });
        topPanel.add(filterField);

        // Tabla para mostrar los datos de corredores y carreras
        tableModel = new DefaultTableModel();
        table = new JTable(tableModel);
        sorter = new TableRowSorter<>(tableModel);
        table.setRowSorter(sorter);
        JScrollPane scrollPane = new JScrollPane(table);

        // Centrar el contenido de las celdas
        DefaultTableCellRenderer centerRenderer = new DefaultTableCellRenderer();
        centerRenderer.setHorizontalAlignment(JLabel.CENTER);
        table.setDefaultRenderer(Object.class, centerRenderer);

        frame.getContentPane().add(topPanel, BorderLayout.NORTH);
        frame.getContentPane().add(scrollPane, BorderLayout.CENTER);

        frame.setVisible(true);
    }

    private void populateComboBox() {
        try {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT DISTINCT year FROM races ORDER BY year DESC");
            while (rs.next()) {
                comboBox.addItem(rs.getString("year"));
            }
            rs.close();
            stmt.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void updateTable() {
        try {
            String selectedYear = (String) comboBox.getSelectedItem();
            if (selectedYear != null) {
                // Consulta para obtener los corredores que participaron en las carreras del año seleccionado
                String query = "SELECT d.driver_id, d.forename, d.surname, d.dob, " +
                        "(SELECT COUNT(*) FROM driver_standings ds " +
                        "JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE ds.driver_id = d.driver_id AND r.year = ? AND ds.position = 1) AS carreras_ganadas, " +
                        "(SELECT SUM(ds.points) FROM driver_standings ds " +
                        "JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE ds.driver_id = d.driver_id AND r.year = ?) AS total_points " +
                        "FROM drivers d " +
                        "JOIN driver_standings ds ON d.driver_id = ds.driver_id " +
                        "JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE r.year = ? " +
                        "GROUP BY d.driver_id " +
                        "ORDER BY total_points DESC";

                PreparedStatement pstmt = conn.prepareStatement(query);
                pstmt.setInt(1, Integer.parseInt(selectedYear));
                pstmt.setInt(2, Integer.parseInt(selectedYear));
                pstmt.setInt(3, Integer.parseInt(selectedYear));
                ResultSet rs = pstmt.executeQuery();

                // Obtener columnas
                Vector<String> columnNames = new Vector<>();
                columnNames.add("Driver Name");
                columnNames.add("Wins");
                columnNames.add("Total Points");
                columnNames.add("Date of Birth");

                // Obtener filas
                Vector<Vector<Object>> data = new Vector<>();
                while (rs.next()) {
                    Vector<Object> row = new Vector<>();
                    row.add(rs.getString("forename") + " " + rs.getString("surname"));
                    row.add(rs.getInt("carreras_ganadas"));
                    row.add(rs.getInt("total_points"));
                    row.add(rs.getDate("dob"));
                    data.add(row);
                }

                // Actualizar modelo de la tabla
                tableModel.setDataVector(data, columnNames);

                // Centrar el contenido de las celdas
                DefaultTableCellRenderer centerRenderer = new DefaultTableCellRenderer();
                centerRenderer.setHorizontalAlignment(JLabel.CENTER);
                table.setDefaultRenderer(Object.class, centerRenderer);

                rs.close();
                pstmt.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void filterTable() {
        String text = filterField.getText();
        if (text.trim().length() == 0) {
            sorter.setRowFilter(null);
        } else {
            sorter.setRowFilter(RowFilter.regexFilter("(?i)" + text, 0));
        }
    }

    private void exportToCSV() {
        JFileChooser fileChooser = new JFileChooser();
        int option = fileChooser.showSaveDialog(frame);
        if (option == JFileChooser.APPROVE_OPTION) {
            try (FileWriter writer = new FileWriter(fileChooser.getSelectedFile() + ".csv")) {
                for (int i = 0; i < tableModel.getColumnCount(); i++) {
                    writer.write(tableModel.getColumnName(i) + ",");
                }
                writer.write("\n");

                for (int i = 0; i < tableModel.getRowCount(); i++) {
                    for (int j = 0; j < tableModel.getColumnCount(); j++) {
                        writer.write(tableModel.getValueAt(i, j).toString() + ",");
                    }
                    writer.write("\n");
                }
                JOptionPane.showMessageDialog(frame, "Datos exportados con éxito.");
            } catch (IOException e) {
                e.printStackTrace();
                JOptionPane.showMessageDialog(frame, "Error al exportar los datos.");
            }
        }
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(InterfazGrafica::new);
    }
    }


## Folder Structure

The workspace contains two folders by default, where:

- `src`: the folder to maintain sources
- `lib`: the folder to maintain dependencies

Meanwhile, the compiled output files will be generated in the `bin` folder by default.

> If you want to customize the folder structure, open `.vscode/settings.json` and update the related settings there.

## Dependency Management

The `JAVA PROJECTS` view allows you to manage your dependencies. More details can be found [here](https://github.com/microsoft/vscode-java-dependency#manage-dependencies).
